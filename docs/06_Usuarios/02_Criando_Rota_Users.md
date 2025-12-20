# Criando a rota `/users/`

Com as migrations criadas, e com a base de usuários já no Postgres, podemos começar a criar a rota do `/users`, o que nos levará a criar o `Controller` e o `Model`. Já temos um teste automatizado que criamos para o POST no `/users`, e obviamente ele está falhando porque ainda não criamos nada.

Então vamos começar a criar as coisas!

## Criando a Rota e Controller

Inicialmente, vamos criar um novo arquivo em `./pages/api/v1/` chamados `users.js`, e utilizaremos a mesma estrutura que já temos para as APIs de `/status` e `/migrations`, utilizando o next-connect:

```javascript title="./pages/api/v1/users.js"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";

const router = createRouter();

router.post(postHandler);

export default router.handler(controller.errorHandler);

async function postHandler(request, response) {
  return response.status(201).json({});
}
```

Então até aqui, nada novo. Apenas criamos a rota de `POST` para o `/users`, que simplesmente retorna um 201, fazendo o nosso teste passar.

## Criando o teste de criação de usuário

Seguindo a metodologia do TDD, vamos criar um teste automatizado que valida a criação de um usuário com sucesso. Então a gente espera enviar um POST para essa rota com um determinado payload, e ela nos responder o 201, e com o mesmo payload retornado com os dados do usuário criado. Mas alguns campos não tem como validarmos porque eles são dinâmicos, como o `id`, `created_at` e `updated_at`. Nesses casos, vamos apenas validar se o dado que está lá é válido.

Para validar se uma string corresponde a um valor válido de UUID na versão 4, usaremos o módulo `uuid`:

```bash
npm i -E uuid@11.1.0
```

E agora sim, vamos criar o nosso teste:

```javascript title="./tests/integration/api/v1/users/post.test.js"
import orchestrator from "tests/orchestrator";
import { version as uuidVersion } from "uuid";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
  await orchestrator.runPendingMigrations();
});

describe("POST to /api/v1/users", () => {
  describe("Anonymous user", () => {
    test("With unique and valid data", async () => {
      const user_create = {
        username: "bruno.nonogaki",
        email: "brunono@gmail.com",
        password: "senha123",
      };

      const response = await fetch("http://localhost:3000/api/v1/users", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify(user_create),
      });
      expect(response.status).toBe(201); // Esperando o retorno 201

      const responseBody = await response.json();

      // Validando se o retorno da nossa API é os dados do nosso usuário recém-criado
      expect(responseBody).toEqual({
        id: responseBody.id,
        username: "bruno.nonogaki",
        email: "brunono@gmail.com",
        password: "senha123",
        created_at: responseBody.created_at,
        updated_at: responseBody.updated_at,
      });

      // Validação extra para o formato do UUID e validade da string de Data
      expect(uuidVersion(responseBody.id)).toBe(4);
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();
    });
  });
});
```

Claro que esse teste vai falhar, então vamos fazer a implementação!

## Fazendo o controller invocar um model

O `model` user ainda não existe, mas vamos abstrair o que esse model faz por enquanto, e implementar o nosso controller como se o model já existisse, e assim fica mais fácil entendermos o que vamos precisar no model:

```javascript title="./pages/api/v1/users.js" hl_lines="3 12-13"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import user from "models/user.js"; // <= Model não existe ainda, mas vamos criar

const router = createRouter();

router.post(postHandler);

export default router.handler(controller.errorHandler);

async function postHandler(request, response) {
  const userInputValues = request.body; // <= Pegando o input da request
  const newUser = await user.create(userInputValues); // <= Chamando a função create do model
  return response.status(201).json(newUser);
}
```

## Criando o model de `user`

Agora sim vamos criar o model `user`, que vai ter a lógica para inserir um usuário na base.

```javascript title="./models/user.js"
import database from "infra/database.js";

async function create(userInputValues) {
  const newUser = await runInsertQuery(userInputValues);
  return newUser;

  async function runInsertQuery(userInputValues) {
    const users = await database.query({
      text: `
        INSERT INTO 
          users (username, email, password)
        VALUES ($1, $2, $3)
        RETURNING *
      `, // RETURNING * para a query retornar o usuário criado
      values: [
        userInputValues.username,
        userInputValues.email,
        userInputValues.password,
      ],
    });
    return users.rows[0];
  }
}

const user = {
  create,
};

export default user;
```

!!! success

    Show! Nossa API já está prontinha para realizar cadastro de usuários na base. Mas ainda faltam muitas outras regras de negócio para deixarmos essa API mais robusta, e é isso que faremos a seguir!


