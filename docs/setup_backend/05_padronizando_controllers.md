# Padronizando os Controllers

Atualmente o nosso controller do `/status` está sem nenhuma abstração, e toda a lógica dele está implementada dentro dele, como as funções que fazem as queries no banco de dados, um try/catch global para tratar os erros, o tratamento do erro, etc. Além disso, do jeito que está agora, essa função status está aceitando `POST`, `PUT`, qualquer coisa além do `GET`:

```javascript title="./pages/api/v1/status.js"
import database from "infra/database.js";
import { InternalServerError } from "infra/errors";

async function get_postgres_version() {
  const result = await database.query("SHOW server_version");
  return result.rows[0].server_version;
}

async function get_postgres_max_connections() {
  const result = await database.query("SHOW max_connections");
  return parseInt(result.rows[0].max_connections);
}

async function get_postgres_used_connections() {
  const result = await database.query({
    text: "SELECT COUNT(*)::int FROM pg_stat_activity WHERE datname = $1;",
    values: [process.env.POSTGRES_DB],
  });
  return result.rows[0].count;
}

export default async function status(request, response) {
  try {
    const updatedAt = new Date().toISOString();
    response.status(200).json({
      updated_at: updatedAt,
      dependencies: {
        database: {
          version: await get_postgres_version(),
          max_connections: await get_postgres_max_connections(),
          opened_connections: await get_postgres_used_connections(),
        },
      },
    });
  } catch (err) {
    const publicErrorObject = new InternalServerError({
      cause: err,
    });
    console.log("\n Erro dentro do catch do controller:");
    console.error(publicErrorObject);

    response.status(500).json(publicErrorObject);
  }
}
```

Então o nosso objetivo é refatorar isso, e vamos começar instalando um módulo do NPM chamado `next-connect`:

```bash
npm install -E next-connect@1.0.0
```

## Utilização do next-connect

Para utilizar o `next-connect`, vamos importar a função `createRouter` do objeto nextConnect, criar uma variável chamada router chamando essa função, e aí começar a definir as rotas de `GET`, `POST`, `PUT`, etc. Por exemplo:

```javascript
import { createRouter } from "next-connect"

const router = createRouter()

router.get(...);
router.post(...);
router.put(...);

export default router.handler();
```

O nosso primeiro objetivo é fazer com que o nosso `/status` aceite apenas `GET`. Vamos implementar essa estrutura acima, e dentro do `router.get`, vamos chamar a função `status`. Ou melhor, vamos renomear a função `status` para `getHandler`, e deixar o código mais genérico. Assim, todos os `router.get` vão sempre invocar uma função chamada `getHandler`. E depois, temos que exportar não mais a função status, mas sim o `router.handler()`. Vai ficar assim:

```javascript title="./pages/api/v1/status.js" hl_lines="1 5 7 9 29"
import { createRouter } from "next-connect";
import database from "infra/database.js";
import { InternalServerError } from "infra/errors";

const router = createRouter();

router.get(getHandler);

export default router.handler();

async function get_postgres_version() {
  const result = await database.query("SHOW server_version");
  return result.rows[0].server_version;
}

async function get_postgres_max_connections() {
  const result = await database.query("SHOW max_connections");
  return parseInt(result.rows[0].max_connections);
}

async function get_postgres_used_connections() {
  const result = await database.query({
    text: "SELECT COUNT(*)::int FROM pg_stat_activity WHERE datname = $1;",
    values: [process.env.POSTGRES_DB],
  });
  return result.rows[0].count;
}

async function getHandler(request, response) {
  try {
    const updatedAt = new Date().toISOString();
    response.status(200).json({
      updated_at: updatedAt,
      dependencies: {
        database: {
          version: await get_postgres_version(),
          max_connections: await get_postgres_max_connections(),
          opened_connections: await get_postgres_used_connections(),
        },
      },
    });
  } catch (err) {
    const publicErrorObject = new InternalServerError({
      cause: err,
    });
    console.log("\n Erro dentro do catch do controller:");
    console.error(publicErrorObject);

    response.status(500).json(publicErrorObject);
  }
}
```

### next-connect: `onNoMatchHandler`

E se quisermos fazer com que os demais métodos não declarados retornem um erro `405: Method Not Allowed` ao invés do padrão que é `404: Not Found`, podemos incrementar isso no `router.handler()`:

```javascript
export default router.handler({
  onNoMatch: onNoMatchHandler,
});

function onNoMatchHandler(request, response) {
  response.status(405).end();
}
```

Vamos inclusive criar um erro customizado para isso? Vamos!

Mas podemos usar a abordagem de TDD para praticar um pouco. Vamos criar o seguinte teste, esperando uma resposta padrão, como definimos em [Padronizando Erros](04_padronizando_erros.md):

```javascript title="./tests/integration/api/v1/status/post.test.js"
import orchestrator from "tests/orchestrator";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
});

describe("POST to /api/v1/status", () => {
  describe("Anonymous user", () => {
    test("Retrieving current system status", async () => {
      const response = await fetch("http://localhost:3000/api/v1/status", {
        method: "POST",
      });
      expect(response.status).toBe(405);
      const responseBody = await response.json();
      expect(responseBody).toEqual({
        name: "MethodNotAllowedError",
        message: "Método não permitido para este endpoint.",
        action:
          "Verifique se o método HTTP enviado é válido para este endpoint.",
        status_code: 405,
      });
    });
  });
});
```

Agora se rodarmos o teste, ele vai falhar! Então vamos criar uma classe de Erro customizada no arquivo `./infra/errors.js`, definindo esse retorno:

```javascript title="./infra/tests.js"
export class MethodNotAllowedError extends Error {
  constructor() {
    super("Método não permitido para este endpoint.");
    this.name = "MethodNotAllowedError";
    this.action =
      "Verifique se o método HTTP enviado é válido para este endpoint.";
    this.statusCode = 405;
  }

  toJSON() {
    return {
      name: this.name,
      message: this.message,
      action: this.action,
      status_code: this.statusCode,
    };
  }
}
```

E agora basta importarmos essa classe e chamarmos o erro na função `onNoMatchHandler`:

```javascript title="./pages/api/v1/status.js"
import { InternalServerError, MethodNotAllowedError } from "infra/errors";

export default router.handler({
  onNoMatch: onNoMatchHandler,
});

function onNoMatchHandler(request, response) {
  const publicErrorObject = new MethodNotAllowedError();
  response.status(publicErrorObject.statusCode).json(publicErrorObject);
}
```

### next-connect: `onError`

O `next-connect` também já nos fornece uma forma de tratar erros genéricos, da mesma forma que fizemos com o `onNoMatchHandler`, podemos usar o `onError`. A única diferença é que o `onError` recebe um argumento `error` na assinatura da função. O que vamos fazer com isso é remover o try/catch da função `getHandler`, e implementar o tratamento de erro nessa função `onErrorHandler`, assim:

```javascript title="./pages/api/v1/status.js" hl_lines="11 19-28 49-50 62-71"
import { createRouter } from "next-connect";
import database from "infra/database.js";
import { InternalServerError, MethodNotAllowedError } from "infra/errors";

const router = createRouter();

router.get(getHandler);

export default router.handler({
  onNoMatch: onNoMatchHandler,
  onError: onErrorHandler,
});

function onNoMatchHandler(request, response) {
  const publicErrorObject = new MethodNotAllowedError();
  response.status(publicErrorObject.statusCode).json(publicErrorObject);
}

// Implementando o novo onErrorHandler:
function onErrorHandler(error, request, response) {
  const publicErrorObject = new InternalServerError({
    cause: error,
  });
  console.log("\n Erro dentro do catch do next-connect:");
  console.error(publicErrorObject);

  response.status(500).json(publicErrorObject);
}

async function get_postgres_version() {
  const result = await database.query("SHOW server_version");
  return result.rows[0].server_version;
}

async function get_postgres_max_connections() {
  const result = await database.query("SHOW max_connections");
  return parseInt(result.rows[0].max_connections);
}

async function get_postgres_used_connections() {
  const result = await database.query({
    text: "SELECT COUNT(*)::int FROM pg_stat_activity WHERE datname = $1;",
    values: [process.env.POSTGRES_DB],
  });
  return result.rows[0].count;
}

async function getHandler(request, response) {
  // Removendo completamente o try-catch
  // try {
    const updatedAt = new Date().toISOString();
    response.status(200).json({
      updated_at: updatedAt,
      dependencies: {
        database: {
          version: await get_postgres_version(),
          max_connections: await get_postgres_max_connections(),
          opened_connections: await get_postgres_used_connections(),
        },
      },
    });
  // } catch (err) {
  //   const publicErrorObject = new InternalServerError({
  //     cause: err,
  //   });
  //   console.log("\n Erro dentro do catch do controller:");
  //   console.error(publicErrorObject);

  //   response.status(500).json(publicErrorObject);
  // }
// }
```

## Abstraindo o Controller dos erros

Note que essa parte deo código vai ficar repetida conforme vamos implementando o controle de erros em outras APIs:

```javascript title="./pages/api/v1/status"
import { InternalServerError, MethodNotAllowedError } from "infra/errors";

function onNoMatchHandler(request, response) {
  const publicErrorObject = new MethodNotAllowedError();
  response.status(publicErrorObject.statusCode).json(publicErrorObject);
}

function onErrorHandler(error, request, response) {
  const publicErrorObject = new InternalServerError({
    cause: error,
    statusCode: error.statusCode,
  });
  console.log("\n Erro dentro do catch do next-connect:");
  console.error(publicErrorObject);

  response.status(publicErrorObject.statusCode).json(publicErrorObject);
}
```

Então vamos criar um arquivo `controller.js` dentro da pasta `./infra`, e jogar essa implementação lá pra dentro, e vamos exportar o controller com uma propriedade chamada `errorHandler`, que já vai dar pra gente um handler para usarmos no nosso router.

```javascript title="./infra/controller.js"
import { InternalServerError, MethodNotAllowedError } from "infra/errors";

function onNoMatchHandler(request, response) {
  const publicErrorObject = new MethodNotAllowedError();
  response.status(publicErrorObject.statusCode).json(publicErrorObject);
}

function onErrorHandler(error, request, response) {
  const publicErrorObject = new InternalServerError({
    cause: error,
    statusCode: error.statusCode,
  });

  console.error(publicErrorObject);

  response.status(publicErrorObject.statusCode).json(publicErrorObject);
}

const controller = {
  errorHandler: {
    onNoMatch: onNoMatchHandler,
    onError: onErrorHandler,
  },
};
export default controller;
```

!!! note

    Esse será o nosso arquivo de `controller` de infra. Por enquanto só temos o tratamento de erros aqui dentro, mas futuramente pode ser que tenhamos que adicionar mais coisas.

Bom, agora é só a gente importar esse controller no nosso arquivo de `status.js`, e simplificar essa implementação, assim:

```javascript title="./pages/api/v1/status.js" hl_lines="3 9"
import { createRouter } from "next-connect";
import database from "infra/database.js";
import controller from "infra/controller.js";

const router = createRouter();

router.get(getHandler);

export default router.handler(controller.errorHandler);

async function get_postgres_version() {
  const result = await database.query("SHOW server_version");
  return result.rows[0].server_version;
}

async function get_postgres_max_connections() {
  const result = await database.query("SHOW max_connections");
  return parseInt(result.rows[0].max_connections);
}

async function get_postgres_used_connections() {
  const result = await database.query({
    text: "SELECT COUNT(*)::int FROM pg_stat_activity WHERE datname = $1;",
    values: [process.env.POSTGRES_DB],
  });
  return result.rows[0].count;
}

async function getHandler(request, response) {
  const updatedAt = new Date().toISOString();
  response.status(200).json({
    updated_at: updatedAt,
    dependencies: {
      database: {
        version: await get_postgres_version(),
        max_connections: await get_postgres_max_connections(),
        opened_connections: await get_postgres_used_connections(),
      },
    },
  });
}
```

## Criando `ServiceError`

Lá no `database.js` ainda temos um erro genérico sendo gerado aqui:

```javascript title="database.js" hl_lines="7-10"
async function query(queryObject) {
  let client;
  try {
    client = await getNewClient();
    const result = await client.query(queryObject);
    return result;
  } catch (err) {
    console.log("\n Erro dentro do catch do database.js:");
    console.error(err);
    throw err;
  } finally {
    await client?.end();
  }
}
```

Vamos criar um outro erro no nosso `errors.js` chamado `ServiceError`:

```javascript title="./infra/errors.js"
export class ServiceError extends Error {
  constructor({ cause, message }) {
    super(message || "Serviço indisponível no momento.", {
      cause,
    });
    this.name = "ServiceError";
    this.action = "Verifique se o serviço está disponível.";
    this.statusCode = 503;
  }

  toJSON() {
    return {
      name: this.name,
      message: this.message,
      action: this.action,
      status_code: this.statusCode,
    };
  }
}
```

!!! tip

    Veja que nesse erro a gente consegue passar um `message` customizado, informando que o problema é na conexão com o banco. Isso porque futuramente se precisarmos usar esse erro para um outro serviço, podemos apenas alterar a mensagem que vai aparecer na console. Mas caso nenhuma mensagem seja passada, exibiremos apenas um "Serviço indiponível no momento".

Bom, agora vamos importá-lo no `database.js`, passando uma mensagem informando que houve um problema na conexão com o Banco ou com a query:

```javascript title="./infra/database.js" hl_lines="2 11-15"
import { Client } from "pg";
import { ServiceError } from "./errors.js";

async function query(queryObject) {
  let client;
  try {
    client = await getNewClient();
    const result = await client.query(queryObject);
    return result;
  } catch (err) {
    const serviceErrorObject = new ServiceError({
      message: "Erro na conexão com o Banco ou na Query.",
      cause: err,
    });
    throw serviceErrorObject;
  } finally {
    await client?.end();
  }
}
```

Note que nesse caso, o erro que retornamos é um `503: Service Unavailable`. Mas o nosso erro `InternalServerError` está retornando na API um erro `500: Internal Server Error`. Então vamos alterar essa classe para permitir receber um `statusCode` diferente, e usá-lo caso receba essa informação. Caso contrário, continua usando o erro 500 mesmo:

```javascript title="./infra/errors.js" hl_lines="2 8"
export class InternalServerError extends Error {
  constructor({ cause, statusCode }) {
    super("Um erro interno não esperado aconteceu.", {
      cause: cause,
    });
    this.name = "InternalServerError";
    this.action = "Entre em contato com o suporte.";
    this.statusCode = statusCode || 500;
  }

  toJSON() {
    return {
      name: this.name,
      message: this.message,
      action: this.action,
      status_code: this.statusCode,
    };
  }
}
```

Agora vamos no `controller.js`, que é quem utiliza essa classe, e vamos configurár o `onErrorHandler` para passar o statusCode:

```javascript title="./infra/controller.js" hl_lines="4"
function onErrorHandler(error, request, response) {
  const publicErrorObject = new InternalServerError({
    cause: error,
    statusCode: error.statusCode,
  });

  console.error(publicErrorObject);

  response.status(publicErrorObject.statusCode).json(publicErrorObject);
}
```

Então agora lá no nosso `status.js`, quando a função `getHandler` der algum erro de conexão com o banco de dados, o `database.js` vai gerar lançar um erro que definimos como 503 (que vem o `ServiceError` que criamos). Esse erro será passado para o tratamento do `onErrorHandler`, que recebe o erro como parâmetro, e com isso temos o `error.statusCode`, que será o 503.

!!! success

    Com isso, terminamos a primeira etapa da nossa refatoração!
