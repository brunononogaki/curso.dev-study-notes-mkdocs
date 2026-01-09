# Criando o endpoint `/user/`

O nosso sistema de autenticação já está funcional. Ao criar uma sessão no endpoint `api/v1/sessions`, o servidor retorna no header o parâmetro `Set-Cookie`, e a partir disso o client (navegador) passa a enviar o Cookie nas próximas requests. Mas por enquanto não estamos usando isso para nada ainda.

O objetivo agora é criar um endpoint que de fato utiliza esse Cookie, e esse endpoint será o `api/v1/user`. Já temos os endpoints do /users, que serve para criar e atualizar usuários, mas o /user (no singular) será para trazer as informações do próprio usuário logado. Ou seja, com o cookie iremos identificar quem é o usuário que está solicitando a informação, e retornar os dados.

## Setup do Teste

Aqui o setup do teste é meio que o mais do mesmo: vamos criar um usuário, criar uma sessão desse usuário, e fazer o GET no novo endpoint `api/v1/user` para receber os dados do usuário. O único detalhe por enquanto é que a criação da sessão não é uma coisa que estamos interessados na validação do teste, ela é apenas um setup para o teste que realmente queremos fazer. Então vamos abstrair isso no `orchestrator`. 

```javascript title="./tests/orchestrator.js"
import session from "models/session.js"

async function createSession(userId) {
  return await session.create(userId);
} 

const orchestrator = {
  waitForAllServices,
  clearDatabase,
  runPendingMigrations,
  createUser,
  createSession, // <= Exportando o novo método
};
```

Agora vamos criar um teste bem simples, cobrindo o caso de sucesso (sessão válida). Nesse teste, vamos criar o usuário, criar a sessão dele, e fazer um GET para o endpoint `api/v1/user`, passando o Cookie no cabeçalho, exatamente como um navegador faria. Por hora, vamos apenas validar se o retorno é um `200 OK`, e em seguida vamos começar a validar melhor esse endpoint.

```javascript title="./tests/integration/api/v1/user/get.test.js"
import orchestrator from "tests/orchestrator";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
  await orchestrator.runPendingMigrations();
});

describe("GET /api/v1/user", () => {
  describe("Default user", () => {
    test("With valid session", async () => {
      const createdUser = await orchestrator.createUser({
        username: "UserWithValidSession",
      });
      const sessionObject = await orchestrator.createSession(createdUser.id);

      const response = await fetch("http://localhost:3000/api/v1/user", {
        headers: {
          Cookie: `session_id=${sessionObject.token}`,
        },
      });

      expect(response.status).toBe(200);
    });
  });
});

```

Show! Claro que o teste vai falhar, porque ainda não temos o controller do `/user` criado. Então bora criá-lo:

```javascript title="./pages/api/v1/user/index.js"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import user from "models/user.js";

const router = createRouter();

router.get(getHandler);

export default router.handler(controller.errorHandler);

async function getHandler(request, response) {
  return response.status(200).json({});
}
```

Ok, sem novidades até aqui!
 
## Validando o usuário

Agora a nossa aplicação está recebendo um Cookie no cabeçalho da request. O que ela vai precisar fazer é verificar se esse cookie está no banco de dados, e se não está expirado. Assim, poderemos ver se essa é uma sessão válida, e saberemos quem é o usuário dono da sessão! Vamos começar a especular como será esse código no controller, mesmo não tendo ainda nada implementado nos nossos models:

```javascript title="./pages/api/v1/user/index.js" hl_lines="4 13-16"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import user from "models/user.js";
import session from "models/session.js";

const router = createRouter();

router.get(getHandler);

export default router.handler(controller.errorHandler);

async function getHandler(request, response) {
  const sessionToken = request.cookies.session_id;

  const sessionObject = await session.findOneValidByToken(sessionToken);
  const userFound = await user.findOneById(sessionObject.user_id);
  return response.status(200).json(userFound);
}

```

Ou seja, precisamos de um método no model `session` que recebe um Token e consulta na base de dados se o token existe, e se o expires_at dele está na frente da data atual. Vamos escrevê-lo:

```javascript title="./models/session.js"
async function findOneValidByToken(sessionToken) {
  const sessionFound = await runSelectQuery(sessionToken);
  return sessionFound;

  async function runSelectQuery(sessionToken) {
    const results = await database.query({
      text: `
        SELECT 
          *
        FROM
          sessions
        WHERE
          token = $1
          AND expires_at > NOW()
        LIMIT 1
      `,
      values: [sessionToken],
    });
    return results.rows[0];
  }
}
```

Esse método vai retornar em `sessionObject` os valores que estão na tabela session. Com isso, temos o ID do usuário na coluna `user_id`. Então precisamos escrever um método no nosso model `user` para buscar um usuário por ID. Esse será o método `findOneById`:

```javascript title="./models/user.js"
async function findOneById(id) {
  const userFound = await runSelectQuery(id);
  return userFound;

  async function runSelectQuery(id) {
    const results = await database.query({
      text: `
        SELECT 
          *
        FROM
          users
        WHERE
          id = $1
        LIMIT 1
      `,
      values: [id],
    });
    if (results.rowCount === 0) {
      throw new NotFoundError({
        message: "O id informado não foi encontrado no sistema.",
        action: "Verifique se o id está digitado corretamente.",
      });
    } else {
      return results.rows[0];
    }
  }
}
```

Pronto, o nosso endpoint já deve estar 100% funcional. Vamos incrementar os testes para validar a response:

```javascript title="./tests/integration/api/v1/user/get.test.js"
import { version as uuidVersion } from "uuid";
import orchestrator from "tests/orchestrator";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
  await orchestrator.runPendingMigrations();
});

describe("GET /api/v1/user", () => {
  describe("Default user", () => {
    test("With valid session", async () => {
      const createdUser = await orchestrator.createUser({
        username: "UserWithValidSession",
      });
      const sessionObject = await orchestrator.createSession(createdUser.id);

      const response = await fetch("http://localhost:3000/api/v1/user", {
        headers: {
          Cookie: `session_id=${sessionObject.token}`,
        },
      });

      expect(response.status).toBe(200);
      const responseBody = await response.json();
      expect(responseBody).toEqual(
        {
          id: createdUser.id,
          username: "UserWithValidSession",
          email: createdUser.email,
          password: createdUser.password,
          // conversão para toISOString, porque o que retornamos do orchestrator.createUser é um objeto Date nativo do JavaScript
          // e o que retornamos da API é uma string, e não um objeto do tipo Date
          // Portanto, precisamos converter o que retornamos do orchestrtor para uma string, para bater com o tipo que volta da response da API
          created_at: createdUser.created_at.toISOString(),
          updated_at: createdUser.updated_at.toISOString()
        }
      )
      expect(uuidVersion(responseBody.id)).toBe(4);
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();      
    });
  });
});
```

!!! success

    Sucesso, o nosso endpoint `/user` está funcional! A seguir, vamos implementar a cobertura de testes nas situações de falha!

## Usando `Fake Timers` para testes de sessões expiradas