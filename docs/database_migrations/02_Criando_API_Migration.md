# Criando a API de Migrations

Agora a ideia é criamos uma rota de GET e POST para o endpoint **api/v1/migrations**.

O GET vai rodar em modo "Dry-Run", ou seja, ele não vai de fato aplicar nada no Banco. Já o POST vai rodar sem o modo Dry-Run, então vai executar pra valer!

## Criando os endpoints

Para criar um endpoint de API, vamos começar com a estrutura padrão:

```javascript
export default async function migration(request, response) {
  // aqui vai a função
}
```

Através do **request.method**, podemos ver se o método que chamou a função é um GET, POST, PUT, etc. E através da documentação do node-pg-migrate, vemos que esse módulo tem um **migrationRunner**, que podemos chamar para rodar as migrações. Então código podería ser assim:

```javascript title="pages/api/v1/migration.js"
import migrationRunner from "node-pg-migrate";
import { join } from "node:path"; // serve para apontarmos um diretório no projeto, sem tem que usar "/" ou "\"

export default async function status(request, response) {
  if (request.method === "GET") {
    const migrations = await migrationRunner({
      databaseUrl: process.env.DATABASE_URL,
      dryRun: true,
      dir: join("infra", "migrations"),
      verbose: true,
      direction: "up",
      migrationsTable: "pgmigrations",
    });
    return response.status(200).json(migrations);
  }
  if (request.method === "POST") {
    const migrations = await migrationRunner({
      databaseUrl: process.env.DATABASE_URL,
      dryRun: false,
      dir: join("infra", "migrations"),
      verbose: true,
      direction: "up",
      migrationsTable: "pgmigrations",
    });
    return response.status(200).json(migrations);
  }
  return response.status(405);
}
```

## Criando os testes do GET

Primeiramente, vamos criar os testes do GET:

```javascript title="/tests/integration/v1/migrations.get.test.js"
describe("GET /api/v1/migrations", () => {
  describe("Anonymous user", () => {
    test("Getting pending migrations", async () => {
      const response = await fetch("http://localhost:3000/api/v1/migrations");
      expect(response.status).toBe(200);

      const responseBody = await response.json();
      // console.log(responseBody);
      expect(Array.isArray(responseBody)).toBe(true);
      expect(responseBody.length).toBeGreaterThan(0);
    });
  });
});
```

Mas ao rodar esse teste, vemos que ele vai falhar, porque a gente espera que ele retore um array com migrações para rodar (toBeGreaterThan(0)), mas ele retorna um Array zerado, pois o banco já está "usado", e não tem migrações para rodar. A gente precisa iniciar o teste com um banco limpo!

Vamos então rodar uma limpeza no banco antes de rodar os testes. Para isso, vamos ter que importar o nosso **infra/database.js** para poder rodar uma query no banco. Mas se importarmos assim, vai dar erro:

```javascript
import database from "infra/database.js";
```

Motivo: o Jest não trabalha com imports ECMAScript como o Next, e não tem as ferramentas do Next.JS para importar os .env para o process.env, não em os absolute imports, etc. Então vamos criar um arquivo de configuração do jest para ele usar os "poderes" do Next:

```javascript title="/jest.config.js"
// para esse import no arquivo de config, temos que usar esse método "antigo"
const nextJest = require("next/jest");

// o dotenv será importado para falarmos para o Jest que o arquivo .env que ele precisa usar é o .env.development
// Isso é necessário porque o Jest configura a variável de ambiente NODE_ENV como "test", e não como "development". Logo, ele não importa as variáveis do arquivo .env.development. Ou seja, o Jest está rodando no "test", enquanto o Next está rodando no "development"
const dotenv = require("dotenv");
dotenv.config({
  path: ".env.development",
});

const createjestConfig = nextJest({
  dir: ".",
});

const jestConfig = createjestConfig({
  moduleDirectories: ["node_modules", "<rootDir>"],
});
module.exports = jestConfig;
```

Agora sim, Jest configurado, podemos voltar no nosso arquivo de teste e limpar o banco. Para isso, podemos usar o **beforeAll** do Jest, que executa comandos antes de começar os testes.

```javascript title="/tests/integration/v1/migrations.get.test.js"
beforeAll(clearDatabase);
async function clearDatabase() {
  await database.query("drop schema public cascade; create schema public");
}

describe("GET /api/v1/migrations", () => {
  describe("Anonymous user", () => {
    test("Getting pending migrations", async () => {
      const response = await fetch("http://localhost:3000/api/v1/migrations");
      expect(response.status).toBe(200);

      const responseBody = await response.json();
      // console.log(responseBody);
      expect(Array.isArray(responseBody)).toBe(true);
      expect(responseBody.length).toBeGreaterThan(0);
    });
  });
});
```

Sucesso! Testes do GET funcionando!

## Refatorando o migrations.js (primeira parte)

Vamos dar uma melhorada no código do migrations.js, que tem muita repetição de definição do banco. E podemos reaproveitar a função getNewClient() que exportamos do database.js, que retorna uma instância conectada do banco.

```javascript title="/pages/api/v1/migration.js"
import migrationRunner from "node-pg-migrate";
import { join } from "node:path";
import database from "infra/database.js";

export default async function migrations(request, response) {
  // Limitando os métodos permitidos para GET e POST apenas
  const allowedMethods = ["GET", "POST"];
  if (!allowedMethods.includes(request.method)) {
    return response
      .status(405)
      .json({ error: `Method "${request.method}" not allowed` });
  }

  let dbClient;

  try {
    dbClient = await database.getNewClient();

    // Criando a variável defaultMigrationOptions com os dados do migrationRunner
    const defaultMigrationOptions = {
      dbClient: dbClient,
      dryRun: true,
      dir: join("infra", "migrations"),
      verbose: true,
      direction: "up",
      migrationsTable: "pgmigrations",
    };

    // Se for um GET, rodamos como Dry Run mesmo
    if (request.method === "GET") {
      const pendingMigrations = await migrationRunner(defaultMigrationOptions);
      return response.status(200).json(pendingMigrations);
    }

    // Se for POST, temos que mudar o dryRun para false no JSON
    if (request.method === "POST") {
      const migratedMigrations = await migrationRunner({
        ...defaultMigrationOptions,
        dryRun: false,
      });
      if (migratedMigrations.length > 0) {
        // retorno 201 se tinha alguma migração para fazer
        return response.status(201).json(migratedMigrations);
      } else {
        // retorno 200 se não tinha nenhuma migração pendente
        return response.status(200).json(migratedMigrations);
      }
    }
  } catch (err) {
    console.error(err);
    throw error;
  } finally {
    await dbClient.end();
  }
}
```

E agora é só criar os testes do post. Nesse teste, rodaremos o POST duas vezes. Na primeira ele tem que detectar uma migração e rodar ela, e na segunda, como a migração já vai ter acontecido, ele deve retornar 0 (nenhuma migração pendente):

```javascript title="/tests/integration/api/v1/migrations/post.test.js"
import database from "infra/database.js";

beforeAll(clearDatabase);

async function clearDatabase() {
  await database.query("drop schema public cascade; create schema public");
}

describe("POST to /api/v1/migrations", () => {
  describe("Running pending migrations", () => {
    test("First run", async () => {
      const response1 = await fetch("http://localhost:3000/api/v1/migrations", {
        method: "POST",
      });
      expect(response1.status).toBe(201);

      const responseBody1 = await response1.json();
      expect(Array.isArray(responseBody1)).toBe(true);
      expect(responseBody1.length).toBeGreaterThan(0);
    });
    test("Second run", async () => {
      const response2 = await fetch("http://localhost:3000/api/v1/migrations", {
        method: "POST",
      });
      expect(response2.status).toBe(200);

      // Segundo POST
      const responseBody2 = await response2.json();
      expect(Array.isArray(responseBody2)).toBe(true);
      expect(responseBody2.length).toBe(0);
    });
  });
});
```

## Segunda refatoração do `migrations.js` com a Padronização dos Controllers

Como na [Padronização dos Controllers](../setup_backend/05_padronizando_controllers.md) optamos por usar um módulo chamado next-connect, e já implementamos tratamentos de erros, vamos implementar da mesma forma que fizemos com a API `status.js`.

```javascript title="pages/api/v1/migration.js"
import { createRouter } from "next-connect";
import migrationRunner from "node-pg-migrate";
import { resolve } from "node:path";
import database from "infra/database.js";
import controller from "infra/controller.js";

const router = createRouter();

router.get(getHandler);
router.post(postHandler);

export default router.handler(controller.errorHandler);

const defaultMigrationOptions = {
  dryRun: true,
  dir: resolve("infra", "migrations"),
  verbose: true,
  direction: "up",
  migrationsTable: "pgmigrations",
};

async function getHandler(request, response) {
  let dbClient;
  try {
    dbClient = await database.getNewClient();
    const pendingMigrations = await migrationRunner({
      ...defaultMigrationOptions,
      dbClient,
    });

    return response.status(200).json(pendingMigrations);
  } finally {
    await dbClient?.end();
  }
}

async function postHandler(request, response) {
  let dbClient;
  try {
    dbClient = await database.getNewClient();
    const migratedMigrations = await migrationRunner({
      ...defaultMigrationOptions,
      dryRun: false,
      dbClient,
    });

    if (migratedMigrations.length > 0) {
      return response.status(201).json(migratedMigrations);
    } else {
      return response.status(200).json(migratedMigrations);
    }
  } finally {
    await dbClient?.end();
  }
}
```

!!! success

    Agora já podemos subir esse código para o git, e rodar as migrações nos nossos bancos de Produção e Homologação, chamando as nossas APIs públicas! 🙌🙌🙌
