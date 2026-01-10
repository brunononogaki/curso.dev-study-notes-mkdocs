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

### Testando uma sessão inválida

Para testar uma sessão inválida é bastante simples. Primeiramente vamos criar o nosso cenário de teste que enviamos um Cookie que não existe na base de dados, e isso deveria nos retornar um erro `401 UNAUTHORIZED`, com uma mensagem informando que o usuário não possui nenhuma sessão válida:

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
    // testes anteriores ocultados

    test("With nonexistent session", async () => {
      const nonExistentToken =
        "ac59a711d8afd140910018a38adc9d9f7ba482663605f2dbab7412518d1360665216e2b54d6356b0da440afbcfaff6bd";
      const response = await fetch("http://localhost:3000/api/v1/user", {
        headers: {
          Cookie: `session_id=${nonExistentToken}`,
        },
      });

      expect(response.status).toBe(401);
      const responseBody = await response.json();
      expect(responseBody).toEqual({
        name: "UnauthorizedError",
        message: "Usuário não possui sessão ativa.",
        action: "Verifique se este usuário está logado e tente novamente.",
        status_code: 401,
      });
    });
  });
});
```

Agora vamos adicionar essa validação no model de `session`, exatamente da mesma forma que implementamos nos outros models como o de `users`:

```javascript title="./models/sessions.js" hl_lines="20-27"
//...
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
    if (results.rowCount === 0) {
      throw new UnauthorizedError({
        message: "Usuário não possui sessão ativa.",
        action: "Verifique se este usuário está logado e tente novamente.",
      });
    } else {
      return results.rows[0];
    }
  }
}
```

### Testando uma sessão expirada

Agora nesse teste, vamos validar a situação que a sessão existe no banco de dados, mas ela está expirada. Mas para não termos que esperar o tempo de ela expirar, vamos usar o `Fake Timers` do Jest para fingir que estamos em uma data do futuro! Veja como é simples:

```javascript title="./tests/integration/api/v1/user/get.test.js" hl_lines="15-18 25-56"
import { version as uuidVersion } from "uuid";
import orchestrator from "tests/orchestrator";
import session from "models/session.js";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
  await orchestrator.runPendingMigrations();
});

describe("GET /api/v1/user", () => {
  describe("Default user", () => {
    // testes anteriores ocultados
    test("With expired session", async () => {
      // Definindo o agora como sendo 30 dias no passado
      jest.useFakeTimers({
        now: new Date(Date.now() - session.EXPIRATION_IN_MILLISECONDS),
      });

      const createdUser = await orchestrator.createUser({
        username: "UserWithExpiredSession",
      });
      const sessionObject = await orchestrator.createSession(createdUser.id);

      // Definindo o agora como sendo agora de verdade
      jest.useRealTimers();     
       
      const response = await fetch("http://localhost:3000/api/v1/user", {
        headers: {
          Cookie: `session_id=${sessionObject.token}`,
        },
      });

      expect(response.status).toBe(401);
      const responseBody = await response.json();
      expect(responseBody).toEqual({
        name: "UnauthorizedError",
        message: "Usuário não possui sessão ativa.",
        action: "Verifique se este usuário está logado e tente novamente.",
        status_code: 401,
      });
    });
```

## Renovando sessões

Agora vamos fazer com que o usuário renove a sua sessão toda vez que ele encostar no endpoint `/user`. No teste de fizemos para "With valid session", ao invés de validar apenas o corpo da resposta com os dados do usuário, queremos também validar se a sessão foi renovada. Isso não está programado ainda, mas vamos implementar esse teste!



```javascript title="./tests/integration/api/v1/user/get.test.js" hl_lines="40-45"
import { version as uuidVersion } from "uuid";
import orchestrator from "tests/orchestrator";
import session from "models/session.js";
import setCookieParser from "set-cookie-parser";

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
      expect(responseBody).toEqual({
        id: createdUser.id,
        username: "UserWithValidSession",
        email: createdUser.email,
        password: createdUser.password,
        created_at: createdUser.created_at.toISOString(),
        updated_at: createdUser.updated_at.toISOString(),
      });
      expect(uuidVersion(responseBody.id)).toBe(4);
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();

      // Session renewal assertions
      const renewedSessionObject = await session.findOneValidByToken(
        sessionObject.token,
      );
      expect(renewedSessionObject.expires_at > sessionObject.expires_at).toBe(true);
      expect(renewedSessionObject.updated_at > sessionObject.updated_at).toBe(true);

    // Demais testes ocultados...
```      

Nesses testes, portanto, queremos que depois de validar o GET com sucesso no `/user`, retornando os dados do usuário, vamos ver direto na base de dados se o Token foi atualizado. Ou seja, se o `expires_at` e o `updated_at` estão com um valor futuro ao da criação do registro.

Agora, lá no controller do `/user`, vamos chamar um método de renew depois de consultarmos se a sessão existe:

```javascript title="./pages/api/v1/user/index.js" hl_lines="17"
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
  
  await session.renew(sessionObject.id);
  
  const userFound = await user.findOneById(sessionObject.user_id);
  return response.status(200).json(userFound);
}
```

E por fim, vamos implementar esse método dentro do model de `session`, que vai basicamente fazer um `UPDATE` na tabela de sessions:

```javascript title="./models/session.js"
async function renew(sessionId) {
  // Data atual somada a 30 dias para frente
  const expiresAt = new Date(Date.now() + EXPIRATION_IN_MILLISECONDS);

  const renewedSessionObject = await runUpdateQuery(sessionId, expiresAt);
  return renewedSessionObject;

  async function runUpdateQuery() {
    const results = await database.query({
      text: `
        UPDATE
          sessions
        SET
          expires_at=$2,
          updated_at=NOW()
        WHERE
          id=$1
        RETURNING *
        ;`,
      values: [sessionId, expiresAt],
    });
    return results.rows[0];
  }
}
```

!!! note

    Nesse momento, estamos fazendo o refresh do token no Banco de Dados. Como o nosso teste está apenas validando a informação no banco, os testes vão passar. Mas pensando no ponto de vista da aplicação, o browser do usuário não sabe que esse Token foi atualizado. Para notificá-lo disso, vamos devolver essa instrução com um `Set-Cookie` no Header da resposta do `/user`.

### Atualizando o Cookie no client

Atualmente já temos o código do `Set-Cookie` dentro do controller de `/sessions`, quando fizemos isso:

```javascript title="./pages/api/v1/sessions/index.js"
import * as cookie from "cookie";

async function postHandler(request, response) {
  // restante do código foi ocultado
  
  const setCookie = cookie.serialize("session_id", newSession.token, {
    path: "/",
    // expires: new Date(newSession.expires_at), <= Preferível usar maxAge
    maxAge: session.EXPIRATION_IN_MILLISECONDS / 1000,
    secure: process.env.NODE_ENV === "production",
    httpOnly: true, // previne ataque de XSS
  });
  response.setHeader("Set-Cookie", setCookie);


  return response.status(201).json(newSession);
}
```

Como precisaremos usar esse código em outros endpoints, vamos mover isso para o controller de `infra`, já que é um método que lida com a infraestrutura de Internet. Aqui poderíamos colocar também dentro do model de sessions, já que o cookie tem a ver com as sessões, mas enfim, são decisões do projeto. Não existe um lugar "certo" para abstrair esse código.

```javascript title="./infra/controller.js"
import * as cookie from "cookie";
import session from "models/session.js";

async function setSessionCookie(sessionToken, response) {
  const setCookie = cookie.serialize("session_id", sessionToken, {
    path: "/",
    // expires: new Date(newSession.expires_at),
    maxAge: session.EXPIRATION_IN_MILLISECONDS / 1000,
    secure: process.env.NODE_ENV === "production",
    httpOnly: true, // previne ataque de XSS
  });

  response.setHeader("Set-Cookie", setCookie);
}

const controller = {
  errorHandler: {
    onNoMatch: onNoMatchHandler,
    onError: onErrorHandler,
  },
  setSessionCookie, // <= Exportando esse novo método
};
export default controller;
```

E para manter o código atual funcionando, vamos alterar o que havíamos feito no controller de `/sessions`, apagar aquele bloco de código e substituir apenas para uma chamada da função `setSessionCookie()`.

```javascript title="./pages/api/v1/sessions/index.js" hl_lines="22"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import authentication from "models/authentication.js";
import session from "models/session.js";

const router = createRouter();

router.post(postHandler);

export default router.handler(controller.errorHandler);

async function postHandler(request, response) {
  const userInputValues = request.body;

  const authenticatedUser = await authentication.getAuthenticatedUser(
    userInputValues.email,
    userInputValues.password,
  );

  const newSession = await session.create(authenticatedUser.id);

  controller.setSessionCookie(newSession.token, response);

  return response.status(201).json(newSession);
}
```

Pronto, depois dessa pequena refatoração, agora é só usar esse novo método `setSessionCookie()` no nosso controller de `/user`!

```javascript title="./pages/api/v1/user/index.js" hl_lines="17"
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
  const renewSessionObject = await session.renew(sessionObject.id);
  controller.setSessionCookie(renewSessionObject.token, response);

  const userFound = await user.findOneById(sessionObject.user_id);
  return response.status(200).json(userFound);
}
```

Só faltou cobrir isso nos testes. Agora, depois de verificar se a sessão foi renovada no banco de dados, precisamos fazer o test do Set-Cookie, parseando header e verificando se `maxAge` está atualizado. A gente já tinha feito esse mesmo teste no `POST` do `/sessions`. Então é só copiar o mesmo código:

```javascript title="./tests/integration/api/v1/user/get.test.js" hl_lines="47-57"
import { version as uuidVersion } from "uuid";
import orchestrator from "tests/orchestrator";
import session from "models/session.js";
import setCookieParser from "set-cookie-parser";

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
      expect(responseBody).toEqual({
        id: createdUser.id,
        username: "UserWithValidSession",
        email: createdUser.email,
        password: createdUser.password,
        created_at: createdUser.created_at.toISOString(),
        updated_at: createdUser.updated_at.toISOString(),
      });
      expect(uuidVersion(responseBody.id)).toBe(4);
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();

      // Session renewal assertions
      const renewedSessionObject = await session.findOneValidByToken(
        sessionObject.token,
      );
      expect(renewedSessionObject.expires_at > sessionObject.expires_at).toBe(true);
      expect(renewedSessionObject.updated_at > sessionObject.updated_at).toBe(true);

      // Set-Cookie assertions
      const parsedSetCookie = setCookieParser(response, {
        map: true,
      });
      expect(parsedSetCookie.session_id).toEqual({
        name: "session_id",
        value: sessionObject.token,
        maxAge: session.EXPIRATION_IN_MILLISECONDS / 1000,
        path: "/",
        httpOnly: true,
      });    
      
      // Demais testes ocultados...
```    

!!! warning

    Tudo certo até agora, os testes estão passando e a sessão está sendo renovada. Mas vamos enfrentar aqui um problema de Cache! Veja só o que acontece:

    1) Na primeira requisição para o `/user`, vai dar tudo certo. A sessão será renovada, o usuário será buscado e o payload com os dados do usuário será retornado ao client. Nesse processo, o Next.js também calcula o hash do body de retorno, e envia esse hash em um Header chamado `Etag`.
    2) Na segunda requisição para o mesmo `/user`, o client faz o GET enviando esse hash em um cabeçalho chamado `If-None-Match`. Com isso, ele orienta o nosso servidor a retornar o body apenas se o hash mudar! Então o servidor recebe a solicitação, faz tudo o que ele tem que fazer (inclusive atualizar o expire_at da sessão no Banco de Dados), mas na hora de retornar ao client, ao invés de enviar um `200 OK` com o Body, ele manda um `304 Not Modified`! E nisso, o client não recebe o novo `Set-Cookie`, e não fica sabendo da renovação da sessão!

    Para resolver esse comportamento, podemos mandar o header `Cache-Control` no `/user` para "no-store, no-cache, max-age=0, must-revalidate", assim:

    ```javascript title="./pages/api/v1/user/index.js" hl_lines="17"
    async function getHandler(request, response) {
        // código ocultado...

        response.setHeader("Cache-Control", "no-store, no-cache, max-age=0, must-revalidate");
        return response.status(200).json(userFound);
    }
    ```

    E incluir essa validação nos testes:
    ```javascript title="./tests/integration/api/v1/user/get.test.js"
      const cacheControl = response.headers.get("Cache-Control");
      expect(cacheControl).toBe(
        "no-store, no-cache, max-age=0, must-revalidate",
      );
    ```

!!! success

    Sucesso! Agora sim, a nossa sessão está sendo novada toda vez que o client encostar na API `/user`!

