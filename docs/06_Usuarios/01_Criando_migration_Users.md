# Criando migration de Users

Nos próximos capítulos vamos começar a criar os Users. Então antes de mais nada, precisamos criar as migrations para criar essa tabela nova na Base de Dados. Para começar, vamos primeiramente remover a migration de teste que havíamos criado, e criar uma nova.

```bash
git rm ./infra/migrations/*.js
```

!!! warning

    Apagar migrations não é algo que deve ser feito. Faremos apenas agora em caráter de exceção, porque a migration que havíamos criado era só para testar, e não fazia nada de fato. Daqui pra frente, nunca mais apagaremos nenhuma migration manualmente!

Agora vamos criar uma migration nova:

```bash
npm run migrations:create create users
```

Com isso, ele vai criar um novo arquivo na pasta `./infra/migrations`. Vamos abrir esse arquivo e declarar a nossa tabela `users`, dessa forma:

```javascript title="./infra/migrations/1765930466222_create-users.js"
exports.up = (pgm) => {
  pgm.createTable("users", {
    id: {
      type: "uuid",
      primaryKey: true,
      default: pgm.func("gen_random_uuid()"),
    },
    username: {
      type: "varchar(30)",
      notNull: true,
      unique: true,
    },
    email: {
      type: "varchar(254)",
      notNull: true,
      unique: true,
    },
    password: {
      type: "varchar(60)",
      notNull: true,
    },
    created_at: {
      type: "timestamptz",
      default: pgm.func("timezone('utc', now())"),
      notNull: true,
    },
    updated_at: {
      type: "timestamptz",
      default: pgm.func("timezone('utc', now())"),
      notNull: true,
    },
  });
};

exports.down = false;
```

!!! note

    Nesse arquiv, podemos excluir tudo que é comentário, e remover a parte do `down`, porque vamos sempre fazer apenas migrações do tipo `up`.

Com isso, se subirmos o nosso servidor web com o `npm run dev`, ele vai automaticamente rodar essa migração e inserir essa nova tabela:

```bash
> node-pg-migrate -m infra/migrations --envPath .env.development up

> Migrating files:
> - 1765930466222_create-users
### MIGRATION 1765930466222_create-users (UP) ###
CREATE TABLE "users" (
  "id" uuid PRIMARY KEY
);
INSERT INTO "public"."pgmigrations" (name, run_on) VALUES ('1765930466222_create-users', NOW());


Migrations complete!
```

## Criando testes de Users

Vamos criar um teste simples de POST na rota `/api/v1/users`, que ainda nem existe, mas estamos usando a abordagem de TDD:

```javascript title="./tests/integration/api/v1/users/post.test.js"
import orchestrator from "tests/orchestrator";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
});

describe("POST to /api/v1/users", () => {
  describe("Anonymous user", () => {
    test("With unique and valid data", async () => {
      const response = await fetch("http://localhost:3000/api/v1/users", {
        method: "POST",
      });
      expect(response.status).toBe(201);
    });
  });
});
```

Esse teste vai rodar com erro porque a rota da API ainda não foi criada, mas tem um detalhe antes de resolvermos isso. Como no `beforeAll` a gente faz um clearDatabase, que zera o banco de dados antes de rodar a bateria de testes, mesmo que a migration tenha rodado no inicio do nosso servidor web, quando rodarmos o teste ela será apagada. Então o que precisamos fazer é rodar as migrações depois do `orchestrator.clearDatabase`. Para isso, lá no `orchestrator` precisamos criar um novo método chamado `runPendingMigrations`, que será reaproveitado do model `migrator.js` que criamos anteriormente:

```javascript title="./tests/integration/orchestrator.js" hl_lines="3 27-29 34"
import retry from "async-retry";
import database from "infra/database";
import migrator from "models/migrator.js";

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
}

async function clearDatabase() {
  await database.query("drop schema public cascade; create schema public");
}

async function runPendingMigrations() {
  await migrator.runPendingMigrations();
}

const orchestrator = {
  waitForAllServices,
  clearDatabase,
  runPendingMigrations,
};

export default orchestrator;
```

E agora chamar no `beforeAll` do nosso teste:

```javascript title="./tests/integration/api/v1/users/post.test.js" hl_lines="6"
import orchestrator from "tests/orchestrator";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
  await orchestrator.runPendingMigrations();
});

describe("POST to /api/v1/users", () => {
  describe("Anonymous user", () => {
    test("With unique and valid data", async () => {
      const response = await fetch("http://localhost:3000/api/v1/users", {
        method: "POST",
      });
      expect(response.status).toBe(201);
    });
  });
});
```

!!! success

    Nossa migration de usuários foi criada, e já configuramos o nosso teste para rodar as migrations depois de limpar a base!