# Definindo as Regras de Negócio de Users

Agora temos uma API básica para criar usuários na base, e é hora de definirmos algumas validações e regras de negócio. A princípio, vamos considerar o seguinte

- Dois usuários não podem ter o mesmo e-mail, ele deve ser único
- O e-mail deve ser case insensitive, ou seja, meu.usuario@email.com e Meu.usuario@email.com devem ser a mesma coisa

## Criando testes para cobrir esses casos

Vamos usar TDD para definir testes e cobrir esses casos. Então dentro dos tests do POST, vamos adicionar um novo teste, que vai primeiramente criar um usuário com sucesso, mas o segundo usuário se cadastrando com o mesmo e-mail (mas com a primeira letra maiúscula) deveria retornar um erro 400:

```javascript title="./tests/integration/api/v1/users/post.test.js" hl_lines="44-74"
import database from "infra/database.js";
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
      });**

      expect(uuidVersion(responseBody.id)).toBe(4);
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();
    });
    test("With duplicated e-mail address", async () => {
      const userToBeCreated1 = {
        username: "emailduplicado1",
        email: "emailduplicado@email.com",
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
        username: "emailduplicado2",
        email: "Emailduplicado@email.com",
        password: "senha123",
      };

      const response2 = await fetch("http://localhost:3000/api/v1/users", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify(userToBeCreated2),
      });
      expect(response2.status).toBe(400);
    });
  });
});
```

Esse teste vai falhar, porque como não temos nenhuma proteção no nosso model, como o e-mail é "diferente" (por causa da letra maiuscula), ele vai adicionar o usuário com sucesso e retornar um 201, e não o erro 400 que estamos esperando.

## Criando a validação de e-mails duplicados

Para essa validação, no começo da função create, vamos verificar se o email já existe na base (usando o comando LOWER para normalizar os dados para a comparação), e se existir vamos lançar um erro chamado `ValidationError`. Como já temos a nossa implementação de tratamento de erros feita, vai bastar criar esse novo erro customizado. Primeiramente, vamos fazer então a validação no Model:

```javascript title="./models/user.js" hl_lines="2 5 10-29"
import database from "infra/database.js";
import { ValidationError } from "infra/errors.js";

async function create(userInputValues) {
  await validateUniqueEmail(userInputValues.email);

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

  async function runInsertQuery(userInputValues) {
    const users = await database.query({
      text: `
        INSERT INTO 
          users (username, email, password)
        VALUES ($1, $2, $3)
        RETURNING *
      `,
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

E agora vamos criar o `ValidationError` no arquivo `./infra/errors.js`

```javascript title="./infra/errors.js"
export class ValidationError extends Error {
  constructor({ cause, message, action }) {
    super("Um erro de validação aconteceu.", {
      cause: cause,
    });
    this.name = "ValidationError";
    this.action = action || "Ajuste os dados enviados e tente novamente";
    this.statusCode = 400;
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

Mas repare que quem lida hoje com os nossos erros é o `controller`, que criamos na [Padronização dos Controllers](../02_setup_backend/05_padronizando_controllers.md). Nele, a gente tem definida uma função chamada onErrorHandler que está assim:

```javascript title="./infra/controller.js"
function onErrorHandler(error, request, response) {
  const publicErrorObject = new InternalServerError({
    cause: error,
    statusCode: error.statusCode,
  });

  console.error(publicErrorObject);

  response.status(publicErrorObject.statusCode).json(publicErrorObject);
}
```

Então ela sempre retorna para a API um erro do tipo `InternalServerError`, só que esse erro deveria ser o último recurso utilizado pela nossa API, caso nenhum outro erro mais específico aconteça. Para tratar isso, vamos colocar um if antes:

```javascript title="./infra/controller.js" hl_lines="2-4"
function onErrorHandler(error, request, response) {
  if (error instanceof ValidationError) {
    return response.status(error.statusCode).json(error);
  }

  const publicErrorObject = new InternalServerError({
    cause: error,
    statusCode: error.statusCode,
  });

  console.error(publicErrorObject);

  response.status(publicErrorObject.statusCode).json(publicErrorObject);
}
```

Agora vamos voltar para o nosso teste, e colocar umas validações adicionais, para garantir que estamos recebendo o erro que definimos, do tipo `ValidationError`:

```javascript title="./tests/integration/api/v1/users/post.test.js" hl_lines="34-39"
    ...
    test("With duplicated e-mail address", async () => {
      const userToBeCreated1 = {
        username: "emailduplicado1",
        email: "emailduplicado@email.com",
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
        username: "emailduplicado2",
        email: "Emailduplicado@email.com",
        password: "senha123",
      };

      const response2 = await fetch("http://localhost:3000/api/v1/users", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify(userToBeCreated2),
      });
      expect(response2.status).toBe(400);

      const response2Body = await response2.json();
      expect(response2Body).toEqual({
        name: "ValidationError",
        message: "O email informado já está sendo utilizado.",
        action: "Utilize outro email para realizar o cadastro.",
        status_code: 400
      })
    });
```

!!! success

    Nossa primeira regra foi implementada, e o teste já está passando com sucesso! Agora, da mesma forma, podemos fazer a validação de usuários duplicados, que vai seguir exatamente a mesma lógica.

