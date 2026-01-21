# Bloqueando endpoints

Agora que o sistema de autorização está pronto, vamos começar a bloquear alguns endpoints

## Bloqueando o endpoint `/activations/[token_id]`

Começando com o endpoint de fazer a ativação, vamos configurará-lo para permitir apenas usuários com a feature `read:activation_token`, que é a feature padrão que atribuímos ao usuário no momento que ele faz o cadastro. Isso é feito no método `create()` do model `user.js`:

```javascript title="./models/user.js" hl_lines="5 28-30"
async function create(userInputValues) {
  await validateUniqueEmail(userInputValues.email);
  await validateUniqueUsername(userInputValues.username);
  await hashPasswordInObject(userInputValues);
  injectDefaultFeaturesInObject(userInputValues);

  const newUser = await runInsertQuery(userInputValues);
  return newUser;

  async function runInsertQuery(userInputValues) {
    const users = await database.query({
      text: `
        INSERT INTO 
          users (username, email, password, features)
        VALUES ($1, $2, $3, $4)
        RETURNING *
      `,
      values: [
        userInputValues.username,
        userInputValues.email,
        userInputValues.password,
        userInputValues.features,
      ],
    });
    return users.rows[0];
  }

  function injectDefaultFeaturesInObject(userInputValues) {
    userInputValues.features = ["read:activation_token"];
  }
}
```

E após a ativação, o usuário perde essa feature para ganhar as features `create:session` e `read:session`.

Então vamos bloquear o endpoint `/activation/[token_id]`, injetando um usuário no contexto e validando se ele tem a feature `read:activation_token`.

```javascript title="./pages/api/v1/activations/[token_id]/index.js" hl_lines="6-7"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import activation from "models/activation";

const router = createRouter();
router.use(controller.injectAnonymousOrUser);
router.patch(controller.canRequest("read:activation_token"), patchHandler);

export default router.handler(controller.errorHandler);

async function patchHandler(request, response) {
  const activationTokenId = request.query.token_id;

  const validActivationToken =
    await activation.findOneValidById(activationTokenId);
  const usedActivationToken =
    await activation.markTokenAsUsed(activationTokenId);

  await activation.activateUserByUserId(validActivationToken.user_id);

  return response.status(200).json(usedActivationToken);
}
```

Só isso já basta para bloquearmos o endpoint. Mas vamos fazer algo além... veja que dessa forma estamos validando se o usuário injetado no contexto (que vai ser um usuário anônimo) possui a feature `read:activation_token`, mas não validamos se o usuário alvo (o usuário que queremos ativar) tem essa feature.

Podemos incluir essa validação dentro do método `activateUserByUserId()`, assim:

```javascript title="./models/activation.js"
sync function activateUserByUserId(userId) {

  const userToActivate = await user.findOneById(userId);

  // Verifica se o usuário que está sendo ativado possui e feature read:activation_token
  if (!authorization.can(userToActivate, "read:activation_token")) {
    throw new ForbiddenError({
      message: "Você não pode mais utilizar tokens de ativação",
      action: "Entre em contato com o suporte.",
    });
  }

  const activatedUser = await user.setFeatures(userId, [
    "create:session",
    "read:session",
  ]);
  return activatedUser;
}
```

!!! note

    Veja que agora ao invés desse método simplesmente fazer o setFeatures sem nenhuma validação, antes a gente verifica se o usuário que está querendo se ativar de fato possui essa feature, e se por acaso não é um usuário que já se ativou, por exemplo.

Mas para isso funcionar, lá no controller a gente tem que inverter a chamada dos métodos, fazendo o `activateUserByUserId` primeiro, e depois o `markTokenAsUsed`, que será chamado apenas se o usuário conseguir ser ativado:

```javascript title="./pages/api/v1/activations/[token_id]/index.js" hl_lines="7-8 10-12"
async function patchHandler(request, response) {
  const activationTokenId = request.query.token_id;

  const validActivationToken =
    await activation.findOneValidById(activationTokenId);

  // Primeiro ativamos
  await activation.activateUserByUserId(validActivationToken.user_id);

  // E depois marcamos o token como usado
  const usedActivationToken =
    await activation.markTokenAsUsed(activationTokenId);

  return response.status(200).json(usedActivationToken);
}
```

## Testando o endpoint `activations`

Atualmente o nosso endpoint de ativação só está sendo testado no fluxo do registration-flow. Vamos criar alguns testes para cobrir esse endpoint (nada diferente do que já fizemos até agora):

```javascript title="./tests/integration/api/v1/activations/[token_id]/patch.test.js"
import orchestrator from "tests/orchestrator";
import { version as uuidVersion } from "uuid";
import user from "models/user.js";
import activation from "models/activation.js";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
  await orchestrator.runPendingMigrations();
});

describe("PATCH to /api/v1/activations/[token_id]", () => {
  describe("Anonymous user", () => {
    test("With non existent token", async () => {
      const response = await fetch(
        "http://localhost:3000/api/v1/activations/e12b6b5b-33ee-4ab2-aa18-53047cb254f8",
        {
          method: "PATCH",
        },
      );
      expect(response.status).toBe(404);

      const responseBody = await response.json();

      expect(responseBody).toEqual({
        name: "NotFoundError",
        message: "Token de ativação não encontrado.",
        action:
          "Verifique se este token de ativação não está expirado ou não foi utilizado.",
        status_code: 404,
      });
    });
    test("With expired token", async () => {
      jest.useFakeTimers({
        now: new Date(Date.now() - activation.EXPIRATION_IN_MILLISECONDS),
      });

      const createdUser = await orchestrator.createUser();
      const expiredActivationToken = await activation.create(createdUser.id);

      jest.useRealTimers();

      const response = await fetch(
        `http://localhost:3000/api/v1/activations/${expiredActivationToken.id}`,
        {
          method: "PATCH",
        },
      );
      expect(response.status).toBe(404);

      const responseBody = await response.json();

      expect(responseBody).toEqual({
        name: "NotFoundError",
        message: "Token de ativação não encontrado.",
        action:
          "Verifique se este token de ativação não está expirado ou não foi utilizado.",
        status_code: 404,
      });
    });
    test("With already used token", async () => {
      const createdUser = await orchestrator.createUser();
      const activationToken = await activation.create(createdUser.id);

      const response1 = await fetch(
        `http://localhost:3000/api/v1/activations/${activationToken.id}`,
        {
          method: "PATCH",
        },
      );
      expect(response1.status).toBe(200);

      const response2 = await fetch(
        `http://localhost:3000/api/v1/activations/${activationToken.id}`,
        {
          method: "PATCH",
        },
      );
      const responseBody = await response2.json();

      expect(responseBody).toEqual({
        name: "NotFoundError",
        message: "Token de ativação não encontrado.",
        action:
          "Verifique se este token de ativação não está expirado ou não foi utilizado.",
        status_code: 404,
      });
    });
    test("With valid token", async () => {
      const createdUser = await orchestrator.createUser();
      const activationToken = await activation.create(createdUser.id);

      const response = await fetch(
        `http://localhost:3000/api/v1/activations/${activationToken.id}`,
        {
          method: "PATCH",
        },
      );
      expect(response.status).toBe(200);
      const responseBody = await response.json();

      expect(responseBody).toEqual({
        id: activationToken.id,
        user_id: activationToken.user_id,
        used_at: responseBody.used_at,
        expires_at: activationToken.expires_at.toISOString(),
        created_at: activationToken.created_at.toISOString(),
        updated_at: responseBody.updated_at,
      });

      expect(uuidVersion(responseBody.id)).toBe(4);
      expect(uuidVersion(responseBody.user_id)).toBe(4);

      expect(Date.parse(responseBody.expires_at)).not.toBeNaN();
      expect(Date.parse(responseBody.created_at)).not.toBeNaN();
      expect(Date.parse(responseBody.updated_at)).not.toBeNaN();
      expect(responseBody.updated_at > responseBody.created_at).toBe(true);

      // Validando se a expiração é de 15 minutos
      const createdAt = new Date(responseBody.created_at);
      const expiresAt = new Date(responseBody.expires_at);
      expiresAt.setMilliseconds(0);
      createdAt.setMilliseconds(0);
      expect(expiresAt - createdAt).toBe(activation.EXPIRATION_IN_MILLISECONDS);

      // Validando se o usuário autenticado possui as features corretas
      const activatedUser = await user.findOneById(responseBody.user_id);
      expect(activatedUser.features).toEqual([
        "create:session",
        "read:session",
      ]);
    });
    test("With valid but already activated user", async () => {
      const createdUser = await orchestrator.createUser();
      await orchestrator.activateUser(createdUser);
      const activationToken = await activation.create(createdUser.id);

      const response = await fetch(
        `http://localhost:3000/api/v1/activations/${activationToken.id}`,
        {
          method: "PATCH",
        },
      );
      expect(response.status).toBe(403);
      const responseBody = await response.json();
      expect(responseBody).toEqual({
        name: "ForbiddenError",
        message: "Você não pode mais utilizar tokens de ativação",
        action: "Entre em contato com o suporte.",
        status_code: 403,
      });
    });
  });
  describe("Default user", () => {
    test("With valid token, but already logged in user", async () => {
      const user1 = await orchestrator.createUser();
      await orchestrator.activateUser(user1);
      const user1SessionObject = await orchestrator.createSession(user1.id);

      const user2 = await orchestrator.createUser();
      const user2ActivationToken = await activation.create(user2.id);

      const response = await fetch(
        `http://localhost:3000/api/v1/activations/${user2ActivationToken.id}`,
        {
          method: "PATCH",
          headers: {
            Cookie: `session_id=${user1SessionObject.token}`,
          },
        },
      );

      expect(response.status).toBe(403);
      const responseBody = await response.json();
      console.log(responseBody);
      expect(responseBody).toEqual({
        name: "ForbiddenError",
        message: "Você não possui permissão para executar esta ação.",
        action:
          'Verifique se o seu usuário possui a feature: "read:activation_token"',
        status_code: 403,
      });
    });
  });
});
```

## Bloqueando o endpoint `/users`

Para bloquear o endpoint de criação de usuários, limitando apenas para quem tiver a feature `create:user`, não tem segredo nenhum:

```javascript title="./pages/api/v1/users/index.js" hl_lines="7-8"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import user from "models/user.js";
import activation from "models/activation";

const router = createRouter();
router.use(controller.injectAnonymousOrUser);
router.post(controller.canRequest("create:user"), postHandler);

export default router.handler(controller.errorHandler);

async function postHandler(request, response) {
  const userInputValues = request.body;
  const newUser = await user.create(userInputValues);

  const activationToken = await activation.create(newUser.id);
  await activation.sendEmailToUser(newUser, activationToken);
  return response.status(201).json(newUser);
}
```

Só isso já basta, porque o nosso código já estava atribuindo essa permissão para os usuários anônimos:
```javascript title="./infra/controller.js"
async function injectAnonymousUser(request) {
  const anonymousUserObject = {
    features: ["read:activation_token", "create:session", "create:user"],
  };

  request.context = {
    ...request.context,
    user: anonymousUserObject,
  };
}
```

Mas depois que o usuário faz a ativação, ele perde essa feature:
```javascript title="./models/activation.js" hl_lines="11-14"
async function activateUserByUserId(userId) {
  const userToActivate = await user.findOneById(userId);

  if (!authorization.can(userToActivate, "read:activation_token")) {
    throw new ForbiddenError({
      message: "Você não pode mais utilizar tokens de ativação",
      action: "Entre em contato com o suporte.",
    });
  }

  const activatedUser = await user.setFeatures(userId, [
    "create:session",
    "read:session",
  ]);
  return activatedUser;
}
```

Portanto, podemos criar mais um test no `users/post.test.js` cobrindo esse caso de um usuário logado tentando criar um outro usuário, situação essa que ele deveria receber um `403 Forbidden`:

```javascript title="./tests/integration/api/v1/users/post.test.js"
...
  describe("Default user", () => {
    test("With unique and valid data", async () => {
      const user1 = await orchestrator.createUser();
      await orchestrator.activateUser(user1);
      const user1SessionObject = await orchestrator.createSession(user1.id);

      const user2Response = await fetch("http://localhost:3000/api/v1/users", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Cookie: `session_id=${user1SessionObject.token}`,
        },
        body: JSON.stringify({
          username: "usuariologado",
          password: "senha123",
        }),
      });
      expect(user2Response.status).toBe(403);
      const response2Body = await user2Response.json();
      expect(response2Body).toEqual({
        name: "ForbiddenError",
        message: "Você não possui permissão para executar esta ação.",
        action:
          'Verifique se o seu usuário possui a feature: "create:user"',
        status_code: 403,
      });
    });
  });
```

!!! success

    Pronto, agora os nossos endpoints de `/activations` e `/users` já estão sendo bloqueados para quem não possui o devido acesso!