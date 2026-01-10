# Implementando o Logout

Depois de tudo que já fizemos, o logout está muito fácil. Basicamente o que vamos precisar é criar um endpoint de `DELETE` no controller de `sessions` que vai basicamente setar a data de expiração para uma data no passado, e assim a gente força a expiração da sessão. Existem outras formas de implementar isso, como criar um campo novo na tabela indicando se o token está expirado por exemplo. Mas para esse projeto, vamos simplesmente alterar a data do `expires_at`.

E veja que não tem nada de muito novo aqui. Vamos começar implementando os testes de falha, e depois o teste de sucesso.

## Cobertura dos testes de falha

Já vamos de cara criar os testes de nonexistent session e de expired session. Ambos são baseados nos testes que fizemos no endpoint de `/user`:

```javascript title="./tests/integration/api/v1/sessions/delete.test.js"
import { version as uuidVersion } from "uuid";
import orchestrator from "tests/orchestrator";
import session from "models/session.js";
import setCookieParser from "set-cookie-parser";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
  await orchestrator.runPendingMigrations();
});

describe("DELETE /api/v1/sessions", () => {
  describe("Default user", () => {
    test("With nonexistent session", async () => {
      const nonExistentToken =
        "ac59a711d8afd140910018a38adc9d9f7ba482663605f2dbab7412518d1360665216e2b54d6356b0da440afbcfaff6bd";
      const response = await fetch("http://localhost:3000/api/v1/sessions", {
        method: "DELETE",
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

      const response = await fetch("http://localhost:3000/api/v1/sessions", {
        method: "DELETE",
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
  });
});
```

Como o nosso endpoint de `DELETE` não existe ainda, esses testes falharão com o erro `405 Method Not Allowed`. Vamos criar a rota no controller:

```javascript title="./pages/api/v1/sessions/index.js"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import authentication from "models/authentication.js";
import session from "models/session.js";

const router = createRouter();

router.post(postHandler);
router.delete(deleteHandler); // <= Nova rota de DELETE

export default router.handler(controller.errorHandler);

async function deleteHandler(request, response) {
  const sessionToken = request.cookies.session_id;

  const sessionObject = await session.findOneValidByToken(sessionToken);
  
  return response.status(200).json(expiredSession);
}
```

Só isso já basta, porque o método `session.findOneValidByToken()` já buscava a sessão no banco de dados, e caso não encontrasse, retornava o 401. Isso já estava implementado, sucesso!

## Forçando a expiração do Token

Agora vamos adicionar o teste de sucesso. Nesse teste a gente espera que o retorno do DELETE seja um `200 OK`, mas mais que isso, queremos que o `expires_at` esteja numa data inferior a da criação da sessão, mas que o `updated_at` esteja em uma data mais atual (indicando que o dado foi alterado na base):

```javascript title="./tests/integration/api/v1/sessions/delete.test.js"
import { version as uuidVersion } from "uuid";
import orchestrator from "tests/orchestrator";
import session from "models/session.js";
import setCookieParser from "set-cookie-parser";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
  await orchestrator.runPendingMigrations();
});

describe("DELETE /api/v1/sessions", () => {
  describe("Default user", () => {
    // Demais testes foram ocultados...

    test("With valid session", async () => {
      const createdUser = await orchestrator.createUser({
        username: "UserWithValidSession",
      });
      const sessionObject = await orchestrator.createSession(createdUser.id);

      const response = await fetch("http://localhost:3000/api/v1/sessions", {
        method: "DELETE",
        headers: {
          Cookie: `session_id=${sessionObject.token}`,
        },
      });

      expect(response.status).toBe(200);
      const responseBody = await response.json();

      expect(responseBody).toEqual({
        id: sessionObject.id,
        token: sessionObject.token,
        user_id: sessionObject.user_id,
        created_at: responseBody.created_at,
        updated_at: responseBody.updated_at,
        expires_at: responseBody.expires_at,
      });
      expect(uuidVersion(responseBody.id)).toBe(4);
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();
      expect(Date.parse(responseBody.updated_at)).not.toBeNaN();
      expect(Date.parse(responseBody.expires_at)).not.toBeNaN();

      // Session delete assertions
      expect(
        responseBody.expires_at < sessionObject.expires_at.toISOString(),
      ).toBe(true);
      expect(
        responseBody.updated_at > sessionObject.updated_at.toISOString(),
      ).toBe(true);
    });
  });
});
```

E implementar isso também é mais do mesmo, só vamos ter que criar um método novo no model de `sessions` que faz essa alteração. Para isso, podemos simplesmente alterar o campo `expires_at` para a mesma data 1 ano para trás, forçando assim a expiração do token.

No controller de sessions, vamos fazer uma chamada para esse método que não existe ainda, e depois programar o método:

```javascript title="./pages/api/v1/sessions/index.js" hl_lines="6"
async function deleteHandler(request, response) {
  const sessionToken = request.cookies.session_id;

  const sessionObject = await session.findOneValidByToken(sessionToken);
  
  const expiredSession = await session.expireById(sessionObject.id);
  
  return response.status(200).json(expiredSession);
}
```

E o método expiredById será assim:

```javascript title="./models/session.js"
async function expireById(sessionId) {
  const deletedSessionObject = await runUpdateQuery(sessionId);
  return deletedSessionObject;

  async function runUpdateQuery(sessionId) {
    const results = await database.query({
      text: `
        UPDATE
          sessions
        SET
          expires_at = expires_at - interval '1 year',
          updated_at = NOW()
        WHERE
          id=$1
        RETURNING *
        ;`,
      values: [sessionId],
    });
    return results.rows[0];
  }
}
```

## Informando o client sobre o Cookie expirado

Nesse momento a gente está somente forçando a expiração da sessão no lado da aplicação (no banco de dados), mas o navegador do cliente não sabe disso! Para fazer com que o navegador apague esse cookie do seu Cookie Jar, precisamos mandar para ele uma instrução de `Set-Cookie`, mas com o maxAge `-1`. Com isso o client sabe que ele deve remover o Cookie.

Então vamos testar mais essa situação dentro do caso de sucesso:

```javascript title="./tests/integration/api/v1/sessions/delete.test.js"
// ...
      // Set-Cookie assertions
      const parsedSetCookie = setCookieParser(response, {
        map: true,
      });
      expect(parsedSetCookie.session_id).toEqual({
        name: "session_id",
        value: "invalid",
        maxAge: -1,
        path: "/",
        httpOnly: true,
      });
```

E bora implementar isso:
```javascript title="./pages/api/v1/sessions/index.js" hl_lines="7"
async function deleteHandler(request, response) {
  const sessionToken = request.cookies.session_id;

  const sessionObject = await session.findOneValidByToken(sessionToken);
  
  const expiredSession = await session.expireById(sessionObject.id);
  controller.clearSessionCookie(response);
  
  return response.status(200).json(expiredSession);
}
```

O método `clearSessionCookie` será muito parecido com o `setSessionCookie`, mas nesse caso precisamos passar apenas a response como parâmetro, porque o valor do Token pode ser qualquer coisa. Nesse caso, vamos setar apenas como `invalid`. O mais importante aqui é o `maxAge: -1`:

```javascript title="./infra/controller.js"
async function clearSessionCookie(response) {
  const setCookie = cookie.serialize("session_id", "invalid", {
    path: "/",
    maxAge: -1,
    secure: process.env.NODE_ENV === "production",
    httpOnly: true,
  });

  response.setHeader("Set-Cookie", setCookie);
}
```

## Testando o logout

Agora podemos incluir mais um teste. Depois de enviarmos o `DELETE`, e validarmos que o `expires_at` no banco foi setado como uma data no passado, e que o cookie retorno contem o `maxAge = -1`, vamos enviar um request para o endpoint `/user`, e garantir que estamos recebendo um erro `401 Unauthorized`, porque seria o mesmo que fazer essa consulta depois de deslogar do sistema:

```javascript title="./tests/integration/api/v1/sessions/delete.test.js"
// ...
      // Double Check
      const doubleCheckResponse = await fetch("http://localhost:3000/api/v1/user", {
        headers: {
          Cookie: `session_id=${sessionObject.token}`,
        },
      });      
      expect(doubleCheckResponse.status).toBe(401);
      const doubleCheckResponseBody = await doubleCheckResponse.json();
      expect(doubleCheckResponseBody).toEqual({
        name: "UnauthorizedError",
        message: "Usuário não possui sessão ativa.",
        action: "Verifique se este usuário está logado e tente novamente.",
        status_code: 401,
      });     
```

!!! success

    Sucesso, o endpoint de DELETE de sessions já está funcionando. Agora temos como fazer um usuário fazer o logout e invalidar a sua sessão.