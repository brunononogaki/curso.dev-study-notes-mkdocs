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

Vamos começar fazendo os testes para updates com sucesso de `username` e `email`:

```javascript title="./api/v1/users/[username]/patch.test.js"
describe("PATCH to /api/v1/users/[username]", () => {
  describe("Anonymous user", () => {
    
    // Os demais testes foram ocultados

    test("With unique username", async () => {
      const userToBeCreated1 = {
        username: "UniqueEmail1",
        email: "uniqueemail1@email.com",
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

      const userToBeUpdated = {
        email: "uniqueemail2@email.com",
      };

      const responseUpdate = await fetch(
        "http://localhost:3000/api/v1/users/UniqueEmail1",
        {
          method: "PATCH",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify(userToBeUpdated),
        },
      );

      expect(responseUpdate.status).toBe(200);

      const responseUpdateBody = await responseUpdate.json();

      expect(responseUpdateBody).toEqual({
        id: responseUpdateBody.id,
        username: "UniqueEmail1",
        email: "uniqueemail2@email.com",
        password: responseUpdateBody.password,
        created_at: responseUpdateBody.created_at,
        updated_at: responseUpdateBody.updated_at,
      });

      expect(uuidVersion(responseUpdateBody.id)).toBe(4);
      expect(Date.parse(responseUpdateBody.created_at)).not.toBeNaN();
      expect(Date.parse(responseUpdateBody.created_at)).not.toBeNaN();
      expect(
        responseUpdateBody.updated_at > responseUpdateBody.created_at,
      ).toBe(true);
    });
    test("With unique email", async () => {
      const userToBeCreated1 = {
        username: "UniqueUser1",
        email: "uniqueuser1@email.com",
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

      const userToBeUpdated = {
        email: "uniqueuser2@email.com",
      };

      const responseUpdate = await fetch(
        "http://localhost:3000/api/v1/users/UniqueUser1",
        {
          method: "PATCH",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify(userToBeUpdated),
        },
      );

      expect(responseUpdate.status).toBe(200);

      const responseUpdateBody = await responseUpdate.json();

      expect(responseUpdateBody).toEqual({
        id: responseUpdateBody.id,
        username: "UniqueUser1",
        email: "uniqueuser2@email.com",
        password: responseUpdateBody.password,
        created_at: responseUpdateBody.created_at,
        updated_at: responseUpdateBody.updated_at,
      });

      expect(uuidVersion(responseUpdateBody.id)).toBe(4);
      expect(Date.parse(responseUpdateBody.created_at)).not.toBeNaN();
      expect(Date.parse(responseUpdateBody.created_at)).not.toBeNaN();
      expect(
        responseUpdateBody.updated_at > responseUpdateBody.created_at,
      ).toBe(true);
    });    
```

E agora vamos criar a query de `PATCH`:

```javascript title="./models/user.js"
async function update(username, userInputValues) {
  const currentUser = await findOneByUsername(username);

  if ("email" in userInputValues) {
    await validateUniqueEmail(userInputValues.email);
  }
  if ("username" in userInputValues) {
    await validateUniqueUsername(userInputValues.username);
  }

  const userWithNewValues = { ...currentUser, ...userInputValues };
  const updatedUser = await runUpdateQuery(userWithNewValues);
  return updatedUser;

  async function runUpdateQuery(userWithNewValues) {
    const users = await database.query({
      text: `
        UPDATE 
          users
        SET
          username = $2,
          email = $3,
          password = $4,
          updated_at = timezone('utc',now())
        WHERE 
          id = $1
        RETURNING *
      `,
      values: [
        userWithNewValues.id,
        userWithNewValues.username,
        userWithNewValues.email,
        userWithNewValues.password,
      ],
    });
    return users.rows[0];
  }
}
```

## Realizando a alteração da senha

Para fazer a alteração da senha, é a mesma coisa, com a diferença que precisaremos criar o hash dela, e nos testes comparar os hashes.
Vamos começar com os testes:

```javascript title="./api/v1/users/[username]/patch.test.js" hl_lines="3-4"
import orchestrator from "tests/orchestrator";
import { version as uuidVersion } from "uuid";
import user from "models/user.js";
import password from "models/password.js";


describe("PATCH to /api/v1/users/[username]", () => {
  describe("Anonymous user", () => {

    // Os demais testes foram ocultados

    test("With new password", async () => {
      const userToBeCreated1 = {
        username: "NewUserPassword1",
        email: "NewUserPassword1@email.com",
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

      const userToBeUpdated = {
        password: "NewPassword",
      };

      const responseUpdate = await fetch(
        "http://localhost:3000/api/v1/users/NewUserPassword1",
        {
          method: "PATCH",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify(userToBeUpdated),
        },
      );

      expect(responseUpdate.status).toBe(200);

      const responseUpdateBody = await responseUpdate.json();

      expect(responseUpdateBody).toEqual({
        id: responseUpdateBody.id,
        username: "NewUserPassword1",
        email: "NewUserPassword1@email.com",
        password: responseUpdateBody.password,
        created_at: responseUpdateBody.created_at,
        updated_at: responseUpdateBody.updated_at,
      });

      expect(uuidVersion(responseUpdateBody.id)).toBe(4);
      expect(Date.parse(responseUpdateBody.created_at)).not.toBeNaN();
      expect(Date.parse(responseUpdateBody.created_at)).not.toBeNaN();
      expect(
        responseUpdateBody.updated_at > responseUpdateBody.created_at,
      ).toBe(true);

      // Coleta dos dados do usuário na base e comparação dos hashes das senhas
      const userInDatabase = await user.findOneByUsername("NewUserPassword1");
      const correctPasswordMatch = await password.compare(
        "NewPassword",
        userInDatabase.password,
      );

      const incorrectPasswordMatch = await password.compare(
        "senha123",
        userInDatabase.password,
      );
      expect(correctPasswordMatch).toBe(true);
      expect(incorrectPasswordMatch).toBe(false);
    });    
```

E agora vamos configurar o model para receber a senha nova, criar o hash dela, e atualizar no banco. A gente já tinha a função `hashPasswordInObject` dentro de `create`, então vamos mover ela para fora, para que tenha um escopo global. E aí basta utilizá-la no `update`:

```javascript title="./models/user.js" hl_lines="10-12"
async function update(username, userInputValues) {
  const currentUser = await findOneByUsername(username);

  if ("email" in userInputValues) {
    await validateUniqueEmail(userInputValues.email);
  }
  if ("username" in userInputValues) {
    await validateUniqueUsername(userInputValues.username);
  }
  if ("password" in userInputValues) {
    await hashPasswordInObject(userInputValues)
  }

  const userWithNewValues = { ...currentUser, ...userInputValues };
  const updatedUser = await runUpdateQuery(userWithNewValues);
  return updatedUser;

  async function runUpdateQuery(userWithNewValues) {
    const users = await database.query({
      text: `
        UPDATE 
          users
        SET
          username = $2,
          email = $3,
          password = $4,
          updated_at = timezone('utc',now())
        WHERE 
          id = $1
        RETURNING *
      `,
      values: [
        userWithNewValues.id,
        userWithNewValues.username,
        userWithNewValues.email,
        userWithNewValues.password,
      ],
    });
    return users.rows[0];
  }
}

async function hashPasswordInObject(userInputValues) {
  const hashedPassword = await password.hash(userInputValues.password);
  userInputValues.password = hashedPassword;
}
```

!!! success

    Agora sim a nossa rota de `patch` está atualizando os dados do usuário com sucesso!

## Refatorando os testes

Por enquanto está tudo certo, mas tem como melhorarmos os testes de usuários. Nos testes dentro de `[username]` (o GET e o PATCH), em cada teste a gente tem que criar um novo usuário, e isso é uma tarefa repetitiva, que podemos delegar para o `orchestrator`. Nesses testes, a gente não está interessado na criação dele, essa criação é na verdade um pré-requisito para o teste.

Esse é o bloco de código que queremos eliminar:
```javascript
const userToBeCreated1 = {
  username: "UsuarioTeste",
  email: "usuario.teste@email.com",
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
```

Vamos então criar um novo método no `orchestrator`:
```javascript title="./tests/orchestrator.js"
import user from "models/user.js";

async function createUser(userObject) {
  await user.create(userObject);
}
```

E agora esses blocos podem ser substituídos simplesmente por isso (alterando apenas os dados do usuário):
```javascript
await orchestrator.createUser(
  {
    username: "UsuarioTeste",
    email: "usuario.teste@email.com",
    password: "senha123",
  }
)
```

Mas note que temos alguns testes em que não estamos interessados exatamente no email ou username. Por exemplo, no teste de username duplicado, podemos mandar qualquer e-mails, pois não interessa muito nesse teste. Para isso, podemos alterar o método `createUser` e passar a usar um módulo chamado `faker`, que cria randomicamente dados respeitando algumas regras, como ter um formato de e-mail válido. Vamos instalar o Faker como uma dependência de desenvolvimento:

```bash
npm i -E -D @faker-js/faker@9.7.0
```

E agora vamos alterar o Orchestrator:

```javascript title="./tests/orchestrator.js"
import { faker } from "@faker-js/faker"

async function createUser(userObject) {
  return await user.create({
    username:
      userObject?.username || faker.internet.username().replace(/[_.-]/g, ""),
    email: userObject?.email || faker.internet.email(),
    password: userObject?.password || "validpassword",
  });
}
```

E nos testes, podemos agora passar apenas os valores que nos interessam na criação do usuário, por exemplo:
```javascript
test("With duplicated email", async () => {
  // não precisamos mais passar o username e a senha do usuário 1
  await orchestrator.createUser({
    // username: "UsernameDuplicado3",
    email: "usernameduplicado3@email.com",
    // password: "senha123",
  });

  // não precisamos mais passar o username e a senha do usuário 2, mas como precisamos saber qual username foi criado pelo faker, anotamos o resultado na variável createdUser2
  const createdUser2 = await orchestrator.createUser({
    // username: "UsernameDuplicado4",
    email: "usernameduplicado4@email.com",
    // password: "senha123",
  });

  const userToBeUpdated = {
    email: "usernameduplicado3@email.com",
  };

  const responseUpdate = await fetch(
    `http://localhost:3000/api/v1/users/${createdUser2.username}`, // <= Utilizamos o usuário criado dinamicamente pelo faker
    {
      method: "PATCH",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(userToBeUpdated),
    },
  );
```

E assim, vamos refatorar os testes de `users/[username]`.