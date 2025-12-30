# Configurando a rota para criação de usuários

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

## Criptografando a senha

Agora o nosso backend tem um problema sério. Estamos armazenando a senha em `texto puro` no Banco de Dados. Vamos resolver isso com o uso de `Bcrypt`.
Primeiramente, vamos instalar a dependência:

```bash
npm i -E bcryptjs@3.0.2
```

Agora vamos atacar o pedaço do código que gera o usuário, que é no model `users`, mais especificamente na função `create`. O que precisamos fazer é alterar a senha no payload `userInputValues`, fazendo um Hash dela.
Vamos chamar uma função chamada `hashPasswordInObject`, que vai receber como argumento o payload, e fará a manipulação dele:

```javascript title="./models/users.js" hl_lines="6 32-35"
import password from "models/password.js"; // <= Vamos criar esse model, que terá as funções que lidarão com passwords

async function create(userInputValues) {
  await validateUniqueEmail(userInputValues.email);
  await validateUniqueUsername(userInputValues.username);
  await hashPasswordInObject(userInputValues); // <= Vamos criar essa função mais pra baixo

  const newUser = await runInsertQuery(userInputValues);
  return newUser;

  async function validateUniqueEmail(email) {
    const results = await database.query({
      text: `
        SELECT
          email
        FROM
          users
        WHERE
          LOWER(email) = LOWER($1)
        LIMIT 1
      `,
      values: [email],
    });
    if (results.rowCount > 0) {
      throw new ValidationError({
        message: "O email informado já está sendo utilizado.",
        action: "Utilize outro email para realizar o cadastro.",
      });
    }
  }

  async function hashPasswordInObject(userInputValues) {
    const hashedPassword = await password.hash(userInputValues.password); // <= Invocando a função hash, que ainda não criamos
    userInputValues.password = hashedPassword;
  }
```

E agora vamos criar o model `password`, que vai ter essa função de hash usando o módulo `Bcryptjs`:

```javascript title="./models/passwords.js"
import bcryptjs from "bcryptjs";

async function hash(password) {
  const rounds = getNumberOfRounds();
  return await bcryptjs.hash(password, rounds);
}

function getNumberOfRounds() {
  return process.env.NODE_ENV === "production" ? 14 : 1;
}

const password = {
  hash,
};

export default password;
```

!!! tip

    Nessa função, estamos utilizando a `getNumberOfRounds` para determinar quantos rounds o Bcrypt rodará para criar o hash. Em ambiente de dev, vamos rodar 1 só para ficar mais rápido na hora de rodar os testes, e apenas em produção rodaremos 14 rounds.

Ao terminar essa implementação, os nossos testes estarão falhando, porque há testes que esperamos que colocamos a validação da senha em clear text, por exemplo:

```javascript
expect(responseBody).toEqual({
  id: responseBody.id,
  username: "bruno.nonogaki",
  email: "brunono@gmail.com",
  password: "senha123",
  created_at: responseBody.created_at,
  updated_at: responseBody.updated_at,
});
```

Mas agora o resultado é um hash:

```bash
  ● POST to /api/v1/users › Anonymous user › With unique and valid data

    expect(received).toEqual(expected) // deep equality

    - Expected  - 1
    + Received  + 1

      Object {
        "created_at": "2025-12-30T00:15:20.928Z",
        "email": "brunono@gmail.com",
        "id": "4d29aa5e-8818-4e80-b83a-2749cb40b932",
    -   "password": "senha123",
    +   "password": "$2b$04$Xhwl/YkBvMVHa.0vUUeo9enC0v.3am.pePZ2PlB716LA3VmdoaZVq",
        "updated_at": "2025-12-30T00:15:20.928Z",
        "username": "bruno.nonogaki",
      }
```

Por hora, vamos resolver isso "tunelando" a saída, como fizemos com o `created_at` e `updated_at`, que também são dados dinâmicos:

```javascript
expect(responseBody).toEqual({
  id: responseBody.id,
  username: "bruno.nonogaki",
  email: "brunono@gmail.com",
  password: responseBody.password,
  created_at: responseBody.created_at,
  updated_at: responseBody.updated_at,
});
```

## Comparando senhas criptografadas

Agora nos testes, como podemos validar se o valor que está na base é um Hash válido? Uma alternativa é pegarmos as informações do usuário lá no banco de dados e verificar o que tem no campo `password` dele.
Para isso, vamos importar o model de users nos testes, que é esse model que sabe fazer isso:

```javascript title="./tests/integration/api/v1/users/post.test.js" hl_lines="3-4 39-51"
import orchestrator from "tests/orchestrator";
import { version as uuidVersion } from "uuid";
import user from "models/user.js";         // <= import novo
import password from "models/password.js"; // <= import novo

describe("POST to /api/v1/users", () => {
  describe("Anonymous user", () => {
    test("With unique and valid data", async () => {
      const userToBeCreated = {
        username: "bruno.nonogaki",
        email: "brunono@gmail.com",
        password: "senha123",
      };

      const response = await fetch("http://localhost:3000/api/v1/users", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify(userToBeCreated),
      });
      expect(response.status).toBe(201);

      const responseBody = await response.json();

      expect(responseBody).toEqual({
        id: responseBody.id,
        username: "bruno.nonogaki",
        email: "brunono@gmail.com",
        password: responseBody.password,
        created_at: responseBody.created_at,
        updated_at: responseBody.updated_at,
      });

      expect(uuidVersion(responseBody.id)).toBe(4);
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();

      // Coleta dos dados do usuário na base e comparação dos hashes das senhas
      const userInDatabase = await user.findOneByUsername("bruno.nonogaki"); // <= Essa função será criada no capítulo 4 (Detalhes de Usuários)
      const correctPasswordMatch = await password.compare(
        "senha123",
        userInDatabase.password,
      );

      const incorrectPasswordMatch = await password.compare(
        "SenhaErrada",
        userInDatabase.password,
      );
      expect(correctPasswordMatch).toBe(true);
      expect(incorrectPasswordMatch).toBe(false);
    });
```

E agora vamos criar essa função `compare` no model `password`.

```javascript title="./models/passwords.js" hl_lines="12-14 18"
import bcryptjs from "bcryptjs";

async function hash(password) {
  const rounds = getNumberOfRounds();
  return await bcryptjs.hash(password, rounds);
}

function getNumberOfRounds() {
  return process.env.NODE_ENV === "production" ? 14 : 1;
}

async function compare(providedPassword, storedPassword) {
  return await bcryptjs.compare(providedPassword, storedPassword);
}

const password = {
  hash,
  compare,
};

export default password;
```

!!! success

    Agora sim as senhas estão sendo armazenadas de forma segura no nosso Banco de Dados!