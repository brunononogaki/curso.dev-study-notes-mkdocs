# Configurando a rota para os detalhes de usuários

O objetivo agora é termos uma rota `api/v1/users/[usuario]`, que retorne os detalhes desse usuário que passamos na URL.

!!! warning

    Antes de seguirmos com essa implementação, vamos fazer uma pequena mudança na estrutura de pastas e arquivos em pages. Até o momento, tinhamos isso:
    ```bash title="./pages"
    .
    ├── api
    │   └── v1
    │       ├── migrations.js
    │       ├── status.js
    │       └── users.js
    ├── index.js
    └── status
        └── index.js
    ```

    Vamos criar uma pasta para cada rota, e renomear os arquivos para `index.js`:
    ```bash title="./pages"
    .
    ├── api
    │   └── v1
    │       ├── migrations
    │       │   └── index.js
    │       ├── status
    │       │   └── index.js
    │       └── users
    │           └── index.js
    ├── index.js
    └── status
        └── index.js
    ```

    Isso porque para essa nova rota, vamos criar uma outra pasta dentro de `users` chamada `[username]`, que o Next por padrão já utiliza como uma URL dinâmica. Veremos mais pra frente!

## Criando o teste de detalhes de usuários

Como sempre, começaremos pelo teste. Dentro da pasta ./tests/integration/api/v1/users, vamos criar uma pasta chamada `[username]`, e dentro criar uma rquivo `get.test.js`.

!!! note

    Não é obrigatório que a pasta em `tests` se chame `[username]`, mas faremos assim para ficar igual às nossas rotas.

```javascript title="./api/v1/users/[username]/get.test.js"
import orchestrator from "tests/orchestrator";
import { version as uuidVersion } from "uuid";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
  await orchestrator.runPendingMigrations();
});

describe("GET to /api/v1/users/[username]", () => {
  describe("Anonymous user", () => {
    test("With exact case match", async () => {
      const userToBeCreated1 = {
        username: "MesmoCase",
        email: "mesmo.case@email.com",
        password: "senha123",
      };

      const response1 = await fetch("http://localhost:3000/api/v1/users", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify(userToBeCreated1),
      });
      expect(response1.status).toBe(201);

      const response2 = await fetch("http://localhost:3000/api/v1/users/MesmoCase")
      expect(response2.status).toBe(200);
      const response2Body = await response2.json();

      expect(response2Body).toEqual({
        id: response2Body.id,
        username: "MesmoCase",
        email: "mesmo.case@email.com",
        password: responseBody.password,
        created_at: response2Body.created_at,
        updated_at: response2Body.updated_at,
      });

      expect(uuidVersion(response2Body.id)).toBe(4);
      expect(Date.parse(response2Body.created_at)).not.toBeNaN();
      expect(Date.parse(response2Body.created_at)).not.toBeNaN();     
    });

    test("With exact case mismatch", async () => {
      const userToBeCreated1 = {
        username: "CaseDiferente",
        email: "case.diferente@email.com",
        password: "senha123",
      };

      const response1 = await fetch("http://localhost:3000/api/v1/users", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify(userToBeCreated1),
      });
      expect(response1.status).toBe(201);

      const response2 = await fetch("http://localhost:3000/api/v1/users/casediferente")
      expect(response2.status).toBe(200);
      const response2Body = await response2.json();

      expect(response2Body).toEqual({
        id: response2Body.id,
        username: "CaseDiferente",
        email: "case.diferente@email.com",
        password: responseBody.password,
        created_at: response2Body.created_at,
        updated_at: response2Body.updated_at,
      });

      expect(uuidVersion(response2Body.id)).toBe(4);
      expect(Date.parse(response2Body.created_at)).not.toBeNaN();
      expect(Date.parse(response2Body.created_at)).not.toBeNaN();     
    });    
  });
});
```

!!! tip

    Nesse teste, já estamos fazendo as consultas independente das letras maiúsculas e minúsculas, por isso são dois testes!

!!! note

    Note que para esse teste funcionar, temos que primeiramente criar um usuário, para depois buscá-lo. Futuramente vamos refatorar isso e deixar esse teste mais focado no que ele precisa de fato testar, mas por enquanto vamos fazer assim para ficar mais claro. Então a ideia é criarmos um usuário chamado `MesmoCase` e em seguida buscá-lo na URL `/api/v1/users/MesmoCase`. O resultado deve ser os detalhes do usuário que criamos.

## Criando a Rota de `[username]`

Para o Next criar rotas dinâmicas, basta criarmos uma pasta com o nome da variável, que no nosso caso será `[username]`.

```bash
mkdir -p ./pages/api/v1/users/[username]
```

E dentro da pasta `[username]`, vamos criar o arquivo `index.js`:

```javascript title="./pages/api/v1/users/[username]/index.js"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import user from "models/user.js";

const router = createRouter();

router.get(getHandler);

export default router.handler(controller.errorHandler);

async function getHandler(request, response) {
  const username = request.query.username;
  const userFound = await user.findOneByUsername(username)
  return response.status(200).json(userFound);
}
```

!!! note

    Nesse arquivo, assim como nos outros, vamos usar o `next-connect` para o nosso Controller, abstraindo já toda a parte de tratamento de erros. O único detalhe a se notar aqui é que para obter o `username` passado na URL, basta pegarmos o `request.query.username`.

Escrevemos esse arquivo, mas o nosso `model` ainda não tem a função `findOneByUsername`. Do ponto de vista do controller, não estamos interessados nessa complexidade, apenas assumimos que existe uma função que faz isso, mas agora é hora de criá-la!

## Criando a função `findOneByUsername` no Model

Vamos abrir o nosso Model de `User` e criar a função `findOneByUsername`. E ela é super simples. Vamos simplesmente fazer uma query no Banco de Dados buscando o usuário por username:

```javascript title="./models/user.js"
import database from "infra/database.js";
import { ValidationError, NotFoundError } from "infra/errors.js";

async function findOneByUsername(username) {
  const userFound = await runSelectQuery(username);
  return userFound;

  async function runSelectQuery(username) {
    const results = await database.query({
      text: `
        SELECT 
          *
        FROM
          users
        WHERE
          LOWER(username) = LOWER($1)
        LIMIT 1
      `,
      values: [username],
    });
    return results.rows[0];
    
  }
}

async function create(userInputValues) {
  // implementação ocultada
}

const user = {
  create,
  findOneByUsername,
};

export default user;
```

## E se não encontrarmos o usuário?

Agora precisamos cobrir o caso de quando o usuário não é encontrado. Vamos incrementar com mais um teste:

```javascript title="./api/v1/users/[username]/get.test.js"
import orchestrator from "tests/orchestrator";
import { version as uuidVersion } from "uuid";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
  await orchestrator.runPendingMigrations();
});

describe("GET to /api/v1/users/[username]", () => {
  describe("Anonymous user", () => {
    test("With exact case match", async () => {
      // Implementação ocultada
    });

    test("With exact case mismatch", async () => {
      // Implementação ocultada
    });    
    test("With non existent username", async () => {
      const response = await fetch("http://localhost:3000/api/v1/users/usuarionaoexiste")
      expect(response.status).toBe(404);
      const responseBody = await response.json();
      expect(responseBody).toEqual({
        name: "NotFoundError",
        message: "O username informado não foi encontrado no sistema.",
        action: "Verifique se o username está digitado corretamente.",
        status_code: 404
      })
    });      
  });
});
```

Agora vamos criar lançar esse erro no Model, caso o usuário não seja encontrado:

```javascript title="./model/user.js"
async function findOneByUsername(username) {
  const userFound = await runSelectQuery(username);
  return userFound;

  async function runSelectQuery(username) {
    const results = await database.query({
      text: `
        SELECT 
          *
        FROM
          users
        WHERE
          LOWER(username) = LOWER($1)
        LIMIT 1
      `,
      values: [username],
    });
    if (results.rowCount === 0) {
      throw new NotFoundError({
        message: "O username informado não foi encontrado no sistema.",
        action: "Verifique se o username está digitado corretamente.",
      });
    } else {
      return results.rows[0];
    }
  }
}
```

E criar o erro `NotFoundError` no nosso arquivo `./infra/errors.js`:
```javascript title="./infra/errors.js"
export class NotFoundError extends Error {
  constructor({ cause, message, action }) {
    super("Não foi possível encontrar esse recurso no sistema", {
      cause: cause,
    });
    this.name = "NotFoundError";
    this.action = action || "Verifique se os parâmetros enviados na consulta estão certos.";
    this.statusCode = 404;
    this.message = message;
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

Agora, precisamos alterar o nosso controller para lançar esse erro mais específico, e não o erro 500 padrão:
```javascript title="./infra/controller.js" hl_lines="5 18-20"
import {
  InternalServerError,
  MethodNotAllowedError,
  ValidationError,
  NotFoundError,
} from "infra/errors";

function onNoMatchHandler(request, response) {
  const publicErrorObject = new MethodNotAllowedError();
  response.status(publicErrorObject.statusCode).json(publicErrorObject);
}

function onErrorHandler(error, request, response) {
  if (error instanceof ValidationError) {
    return response.status(error.statusCode).json(error);
  }

  if (error instanceof NotFoundError) {
    return response.status(error.statusCode).json(error);
  }

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

!!! success

    Boa!! Agora o nosso endpoint de buscar detalhes do usuário está funcionando!