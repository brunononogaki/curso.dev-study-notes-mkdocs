# Configurando a rota para updates de usuários

O objetivo agora é termos uma rota `api/v1/users/[usuario]`, que aceite um PATCH para atualizar alguma informação do usuário.

## Criando o teste de updates de usuários

Dessa vez, vamos começar com os testes que falham (por exemplo, usuário inexistente, username duplicado, etc), para no final cobrirmos o caso que funciona.
Vamos criar o arquivo `patch.test.js` e criar o nosso primeiro teste de falha:

```javascript title="./api/v1/users/[username]/patch.test.js"
import orchestrator from "tests/orchestrator";
import { version as uuidVersion } from "uuid";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
  await orchestrator.runPendingMigrations();
});

describe("PATCH to /api/v1/users/[username]", () => {
  describe("Anonymous user", () => {
    test("With non existent username", async () => {
      const response = await fetch(
        "http://localhost:3000/api/v1/users/usuarionaoexiste",
          {
            method: "PATCH",
          },
      );
      expect(response.status).toBe(404);
      const responseBody = await response.json();
      expect(responseBody).toEqual({
        name: "NotFoundError",
        message: "O username informado não foi encontrado no sistema.",
        action: "Verifique se o username está digitado corretamente.",
        status_code: 404,
      });
    });
  });
});
```

Agora vamos configurar essa nova rota de patch:

```javascript title="./pages/api/v1/users/[username]/index.js" hl_lines="8 18-24"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import user from "models/user.js";

const router = createRouter();

router.get(getHandler);
router.patch(patchHandler);

export default router.handler(controller.errorHandler);

async function getHandler(request, response) {
  const username = request.query.username;
  const userFound = await user.findOneByUsername(username);
  return response.status(200).json(userFound);
}

async function patchHandler(request, response) {
  const username = request.query.username;
  const userInputValues = request.body;

  const updatedUser = await user.update(username, userInputValues);
  return response.status(200).json(updatedUser);
}
```

Assumimos que o model `user` tinha uma função `update`, mas ela ainda não existe. Vamos criá-la:

```javascript title="./models/user.js" hl_lines="7-9 14"
import database from "infra/database.js";
import password from "models/password.js";
import { ValidationError, NotFoundError } from "infra/errors.js";

// restante do código ocultado

async function update(username, userInputValues) {
  const currentUser = await findOneByUsername(username);
}

const user = {
  create,
  findOneByUsername,
  update
};

export default user;
```

!!! success

    Pronto, a nossa rota de `PATCH` já funciona, e no momento está passando o primeiro teste de usuário inexistente, pois essa condição já é tratada na função `findOneByUsername`.

## Teste de username e e-mails duplicados

Uma outra regra de negócio que precisamos implementar é se o usuário tenta mudar o username ou e-mail dele para um que já existe na base. Vamos cobrir isso com novos testes.
O teste vai basicamente criar dois novos usuários, e tentar alterar os dados do segundo conflitando com o do primeiro:

```javascript title="./api/v1/users/[username]/patch.test.js"
import orchestrator from "tests/orchestrator";
import { version as uuidVersion } from "uuid";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
  await orchestrator.runPendingMigrations();
});

describe("PATCH to /api/v1/users/[username]", () => {
  describe("Anonymous user", () => {
    test("With non existent username", async () => {
      const response = await fetch(
        "http://localhost:3000/api/v1/users/usuarionaoexiste",
        {
          method: "PATCH",
        },
      );
      expect(response.status).toBe(404);
      const responseBody = await response.json();
      expect(responseBody).toEqual({
        name: "NotFoundError",
        message: "O username informado não foi encontrado no sistema.",
        action: "Verifique se o username está digitado corretamente.",
        status_code: 404,
      });
    });
    test("With duplicated username", async () => {
      const userToBeCreated1 = {
        username: "UsernameDuplicado1",
        email: "usernameduplicado1@email.com",
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

      const userToBeCreated2 = {
        username: "UsernameDuplicado2",
        email: "usernameduplicado2@email.com",
        password: "senha123",
      };

      const response2 = await fetch("http://localhost:3000/api/v1/users", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify(userToBeCreated2),
      });
      expect(response2.status).toBe(201);

      const userToBeUpdated = {
        username: "UsernameDuplicado1",
      };

      const responseUpdate = await fetch(
        "http://localhost:3000/api/v1/users/UsernameDuplicado2",
        {
          method: "PATCH",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify(userToBeUpdated),
        },
      );

      expect(responseUpdate.status).toBe(400);

      const responseUpdateBody = await responseUpdate.json();

      expect(responseUpdateBody).toEqual({
        name: "ValidationError",
        message: "O username informado já está sendo utilizado.",
        action: "Utilize outro username para realizar esta operação.",
        status_code: 400,
      });
    });
    test("With duplicated email", async () => {
      const userToBeCreated1 = {
        username: "UsernameDuplicado3",
        email: "usernameduplicado3@email.com",
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

      const userToBeCreated2 = {
        username: "UsernameDuplicado4",
        email: "usernameduplicado4@email.com",
        password: "senha123",
      };

      const response2 = await fetch("http://localhost:3000/api/v1/users", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify(userToBeCreated2),
      });
      expect(response2.status).toBe(201);

      const userToBeUpdated = {
        email: "usernameduplicado3@email.com",
      };

      const responseUpdate = await fetch(
        "http://localhost:3000/api/v1/users/UsernameDuplicado4",
        {
          method: "PATCH",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify(userToBeUpdated),
        },
      );

      expect(responseUpdate.status).toBe(400);

      const responseUpdateBody = await responseUpdate.json();

      expect(responseUpdateBody).toEqual({
        name: "ValidationError",
        message: "O email informado já está sendo utilizado.",
        action: "Utilize outro email para realizar esta operação.",
        status_code: 400,
      });
    }); 
  });
});
```

E agora vamos cobrir esses casos no `model`. Repare que o model de users já tinha essas validações no `POST`, e haviamos implementado as funções `validateUniqueEmail` e `validateUniqueUsername` dentro da função `create`. Mas para podermos reaproveitar essas funções, vamos movê-la para fora do `created`, para terem um escopo global. Assim, a nossa função de `update` vai ficar dessa forma:

```javascript title="./models/user.js"
async function update(username, userInputValues) {
  const currentUser = await findOneByUsername(username);

  if ("username" in userInputValues) {
    await validateUniqueUsername(userInputValues.username);
  }

  if ("email" in userInputValues) {
    await validateUniqueEmail(userInputValues.email);
  }
}
```

## Realizando a alteração dos dados do usuário