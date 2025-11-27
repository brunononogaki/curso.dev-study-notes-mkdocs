# Estabilizando o Ambiente Local e Testes

## Criando um script unico de inicialização

Atualmente já temos isso configurado no nosso `package.json`

```bash title="package.json"
  "scripts": {
    "dev": "npm run services:up && next dev",
    "services:up": "docker compose -f infra/compose.yaml up -d",
    "services:stop": "docker compose -f infra/compose.yaml stop",
    "services:down": "docker compose -f infra/compose.yaml down",
    "lint:check": "prettier --check .",
    "lint:fix": "prettier --write .",
    "test": "jest --runInBand",
    "test:watch": "jest --watch-all --runInBand",
    "migration:create": "node-pg-migrate -m infra/migrations create",
    "migration:up": "node-pg-migrate -m infra/migrations --envPath .env.development up"
  },
```

Mas o ideal seria ao subir o `npm run dev`, ele já subisse as migrations. Fácil:

```bash title="package.json"
  "scripts": {
    "dev": "npm run services:up && next dev && npm run migration:up",
    ...
  },
```

Mas tem um porém, se a gente remover o banco de dados e der um novo `npm run dev`, vai falhar, porque quando rodamos a migration, o Banco ainda não estava pronto! Então a ideia é criar um script node para verificar se o Postgres está aceitando conexões, e rodar ele antes de executar as migrations.

```javascript title="/infra/wait-for-postgres.js"
// Import sendo feito com o require, porque como aqui o Next não vai transpilar, fazemos assim para manter o máximo de compatibilidade
const { exec } = require("node:child_process");

function checkPostgres() {
  // Comando para verificar se o Postgres está pronto e respondendo no localhost
  exec("docker exec postgres-dev pg_isready --host localhost", handleReturn);

  function handleReturn(error, stdout) {
    if (stdout.search("accepting connections") === -1) {
      process.stdout.write(".");
      // Caso não esteja pronto ainda, vamos chamar a função recursivamente
      checkPostgres();
      return;
    }
    console.log("\n🟢 Postgres está pronto!");
  }
}

console.log("\n\n🔴 Aguardando Postgres aceitar conexões...");
checkPostgres();
```

Agora criamos um script para rodar esse código:

```bash title="package.json"
  "scripts": {
    ...
    "wait-for-postgres": "node infra/wait-for-postgres.js"
  },
```

E agora vamos chamar ele no `npm run dev`, antes de rodar a migration:

```bash title="package.json"
  "scripts": {
    "dev": "npm run services:up && npm run wait-for-postgres && npm run migration:up && next dev",
  },
```

Agora sim, quando rodarmos o `npm run dev`, vamos subir os containers, esperar o Postgres ficar disponível, rodar as migrations e depois subir o Next! Show de bola!!!

## Consertando o script de testes

Nas aulas anteriores tinhamos criado também o script de testes:

```bash title="package.json"
  "scripts": {
    "test": "jest --runInBand",
    ...
  },
```

Mas se a gente rodar eles com o ambiente fora, vai quebrar, porque esse script não está subindo o ambiente. Poderíamos subir o `npm run dev` antes do Jest, mas a dificuldade aqui é que o `next dev` não tem um modo "detached", como o docker compose. Temos que dar um jeito de executar o next e o jest de forma **concorrente**, e para isso, vamos usar um módulo do npm chamado `concurrently`.

```bash
npm install --save-dev concurrently@8.2.2
```

Agora sim, podemos voltar nos scripts e rodar o jest e o next de forma concorrente, assim:

```bash title="package.json"
  "scripts": {
    "test": "npm run services:up && npm run wait-for-postgres && concurrently --names next,jest --hide next \"next dev\" \"jest --runInBand\"",
    ...
  },
```

Dessa forma, estamos nomeando os processos do Next como "next", e os processos do Jest como "jest", e assim os logs não ficam confusos. Além disso, estamos escondendo os logs do next, que não importa pra gente nesse momento. Já está funcionando!

Mas temos dois problemas:

1. Esse processo não tem um fim. Ou seja, depois que os testes acabam, os processos ficam em aberto;
2. Existe um risco de os testes rodarem antes de o Next subir, pois não estamos definindo nenhuma ordem. Por exemplo, se colocar mos um `sleep 1;` antes do `next dev`, já vai quebrar tudo!

Para o primeiro problema, podemos resolver com algumas parametrizações do `concurrently`:

```bash title="package.json"
  "scripts": {
    "test": "npm run services:up && npm run wait-for-postgres && concurrently --names next,jest --hide next --kill-others --success command-jest \"next dev\" \"jest --runInBand\"",
    ...
  },
```

Isso vai fazer com que quando um dos processos terminarem, ele mate os demais. E vai fazer também com que o processo do concurrently finalize com o mesmo exit code do processo do jest. Ou seja, se o jest terminar com sucesso, o concurrently também vai terminar com sucesso. Uma forma de ver o exit code do processo, basta dar um `echo $?`. Se for 0 é sucesso, se for 1 é erro.

Certo, agora vamos para o segundo problema. Isso será resolvido com um `orchestrator`, que vai fazer com que o jest fique olhando se o next está rodando para ele rodar.

Já vamos construir esse orchestrator, mas antes vamos instalar uma dependência que vamos utilizar nesse projeto: o `async-retry`:

```bash
npm install async-retry@1.3.3
```

Agora a ideia é criarmos uma função que fica aguardando os serviços todos estarem disponíveis, e a gente executa ela antes de executar os testes. Então lá na pasta de tests, vamos criar um arquivo chamado `orchestrator.js`:

```javascript title="/tests/orchestrator.js"
import retry from "async-retry";

async function waitForAllServices() {
  await waitForWebServer();

  async function waitForWebServer() {
    return retry(fetchStatusPage, {
      retries: 100,
      maxTimeout: 1000,      
    });

    async function fetchStatusPage() {
      const response = await fetch("http://localhost:3000/api/v1/status");
      if (response.status !== 200) {
        throw Error();
      }
  }
}

export default {
  waitForAllServices,
};
```

E agora vamos adicionar esse hook nos arquivos de teste:

```javascript title="/tests/integration/api/v1/status/get.test.js"
import orchestrator from "tests/orchestrator";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
});
```

```javascript title="/tests/integration/api/v1/migration/get.test.js"
import orchestrator from "tests/orchestrator";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await database.query("drop schema public cascade; create schema public");
});
```

```javascript title="/tests/integration/api/v1/migration/post.test.js"
import orchestrator from "tests/orchestrator";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await database.query("drop schema public cascade; create schema public");
});
```

Agora sim, mesmo que a gente adicione um atraso de 1s no next dev, os testes vão passar porque eles vão esperar o serviço responder:

```bash title="package.json"
  "scripts": {
    "test": "npm run services:up && npm run wait-for-postgres && concurrently --names next,jest --hide next --kill-others --success command-jest \"sleep 1; next dev\" \"jest --runInBand\"",
    ...
  },
```

Maassss, se adicionarmos um sleep de 5 segundos, aí sim os testes vão falhar:

```bash
[jest]  FAIL  tests/integration/api/v1/migrations/post.test.js (5.107 s)
[jest]   ● POST to /api/v1/migrations should return 200
[jest]
[jest]     thrown: "Exceeded timeout of 5000 ms for a hook.
[jest]     Add a timeout value to this test to increase the timeout, if this is a long-running test. See https://jestjs.io/docs/api#testname-fn-timeout."
```

Isso porque o hook aguarda apenas 5s. Como waitForAllServices demorou mais que 5s, ele aborta.
Para resolver isso, lá no `jest.config.js`, podemos aumentar esse timeout para 60s:

```javascript title="/jest.config.js"
const nextJest = require("next/jest");
const dotenv = require("dotenv");
dotenv.config({
  path: ".env.development",
});

const createjestConfig = nextJest({
  dir: ".",
});

const jestConfig = createjestConfig({
  moduleDirectories: ["node_modules", "<rootDir>"],
  testTimeout: 60000,
});

module.exports = jestConfig;
```

Maravilha, agora os testes estão esperando o ambiente levantar. Vamos remover o timer que forçamos antes do `next dev` e concluir essa etapa.