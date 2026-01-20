# Implementando o sistema de Autorização

Agora vamos criar o model `authorization`, que será responsável por lidar com o sistema de autorização do sistema. A boa notícia é que já está quase tudo feito, e só precisaremos refatorar algumas coisas!

## Model `authorization`

Vamos começar o nosso novo model bem simples, apenas abstraindo a lógica de validar se um usuário tem acesso a uma determinada feature, que a gente resolveu por enquanto dentro do `controller.js`, mais especificamente nessa linha:
```javascript title="./infra/controller.js"
if (userTryingToRequest.features.includes(feature)) {
  ...
```

Vamos jogar isso para dentro do model `authorization`, em um método chamado `can()`:

```javascript title="./models/authorization.js"
function can(user, feature) {
  let authorized = false;
  if (user.features.includes(feature)) {
    authorized = true;
  }
  return authorized;
}

const authorization = {
  can,
};

export default authorization;
```

E agora importar o módulo no `controller.js` e começar a usá-lo:

```javascript title="./infra/controller.js"
import authorization from "models/authorization";

function canRequest(feature) {
  return function canRequestMiddleware(request, response, next) {
    const userTryingToRequest = request.context.user;

    if (authorization.can(userTryingToRequest, feature)) {
      return next();
    }

    throw new ForbiddenError({
      message: "Você não possui permissão para executar esta ação.",
      action: `Verifique se o seu usuário possui a feature: "${feature}"`,
    });
  };
}
```

Show! Por enquanto não mudou muita coisa, mas mais pra frente vamos adicionar outra coisa nesse `can`, que é receber o resource que o usuário está querendo acessar. Mas vamos começar assim simples.

## Trancando o endpoint `/sessions`

Pensando no nosso fluxo até agora, temos o seguinte:

1. Usuário faz o cadastro
2. Usuário recebe o e-mail de ativação
3. Usuário faz a **ativação** da conta
   1. Client envia um PATCH para o endpoint `/activations/[token_id]`, 
   2. Token é validado com sucesso
   3. Chama o método `activation.activateUserByUserId()`, que vai setar a feature `create:session` para o usuário, dando a ele a permissão para fazer o login
4. Usuário tenta fazer o **login** enviando um POST para o endpoint `/sessions`
5. O POST é interceptado pelo primeiro middleware `injectAnonymousOrUser`, que vai injetar no contexto um usuário Anônimo (já que ele ainda não está logado)
6. Validamos se o usuário injetado no contexto tem a permissão `create:session`
7. Como o usuário anônimo por padrão tem já a feature `create:session`, ele vai conseguir entrar na função postHandler, que vai executar o POST
8. Dentro do POST, criamos uma variável chamada authenticatedUser, através do método `authentication.getAuthenticatedUser`, caso o usuário e senha esteja correto
9. Criamos uma sessão com com esse usuário e retornamos o `Set-Cookie` para o cliente

O detalhe é que no passo 9, a sessão é criada para o usuário porque ele recebeu a feature `create:session` depois de fazer a ativação, correto? Mas repare que se a gente remover essa feature no método `activateUserByUserId()`, o login continua funcionando normalmente!

```javascript title="./models/activation.js"
async function activateUserByUserId(userId) {
  const activatedUser = await user.setFeatures(userId, []); //<= Array deixando em branco de propósito para teste
  return activatedUser;
}
```

Isso acontece porque não estamos validando se o `authenticatedUser` possui a feature `create:session` antes de invocarmos o `session.create()`. Vamos implementar esse bloqueio

```javascript title="./pages/api/v1/sessions/index.js" hl_lines="4 7 25-30"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import authentication from "models/authentication.js";
import authorization from "models/authorization.js";
import session from "models/session.js";

import { ForbiddenError } from "infra/errors.js";

const router = createRouter();

router.use(controller.injectAnonymousOrUser); // middleware
router.post(controller.canRequest("create:session"), postHandler);
router.delete(deleteHandler);

export default router.handler(controller.errorHandler);

async function postHandler(request, response) {
  const userInputValues = request.body;

  const authenticatedUser = await authentication.getAuthenticatedUser(
    userInputValues.email,
    userInputValues.password,
  );
  
  if (!authorization.can(authenticatedUser, "create:session")) {
    throw new ForbiddenError({
      message: "Você não possui permissão para fazer login.",
      action: "Contate o suporte caso você acredite que isto seja um erro.",
    });
  }

  const newSession = await session.create(authenticatedUser.id);

  controller.setSessionCookie(newSession.token, response);

  return response.status(201).json(newSession);
}
```

!!! success

    Olha só, agora caso a gente teste novamente o fluxo atribuindo o array de features em branco no `activateUserByUserId`, o teste de login vai falhar com um erro `403 Forbidden`! Mas vamos voltar a feature `create:session` nesse array porque isso era só um teste!

## Corrigindo o teste de login

Após essa implementação, o teste que tinhamos criado para o login vai falhar:

```javascript title="./tests/integration/sessions/post.test.js"
    test("With correct email and correct password", async () => {
      const createdUser = await orchestrator.createUser({
        email: "correct.email@email.com",
        password: "senha-correta",
      });

      const response = await fetch("http://localhost:3000/api/v1/sessions", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          email: activatedUser.email,
          password: "senha-correta",
        }),
      });

      expect(response.status).toBe(201);
```

Isso porque no teste a gente criava o usuário e logo em seguida tentava fazer o login. Precisamos incluir uma etapa intermediária que é ativar o usuário. Vamos fazer isso com mais um método no `orchestrator`:

```javascript title="./tests/integration/sessions/post.test.js" hl_lines="7"
    test("With correct email and correct password", async () => {
      const createdUser = await orchestrator.createUser({
        email: "correct.email@email.com",
        password: "senha-correta",
      });

      const activatedUser = await orchestrator.activateUser(createdUser);

      const response = await fetch("http://localhost:3000/api/v1/sessions", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          email: activatedUser.email,
          password: "senha-correta",
        }),
      });

      expect(response.status).toBe(201);
```

E o método no orchestrator ficará assim:

```javascript title="./tests/orchestrator.js"
async function activateUser(userObject) {
  return await activation.activateUserByUserId(userObject.id);
}
```

## Implementando o teste `Get user information`

Agora só está faltando implementar o teste `Get user information` do `registration-flow.test.js`. Esse teste vai rodar depois do login do usuário, e vai ser bem simples! Precisamos apenas fazer um GET em `/user` e validar o retorno.

Mas antes disso, vamos injetar o usuário no controller de `/user` com os middlewares e validar se o usuário tem a feature `read:session`:

```javascript title="./pages/api/v1/user/index.js"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import user from "models/user.js";
import session from "models/session.js";

const router = createRouter();
router.use(controller.injectAnonymousOrUser); // middleware
router.get(controller.canRequest("read:session"), getHandler);

...
```

E agora na ativação do usuário, vamos adicionar essa feature, além da `create:session` que ele já tinha:

```javascript title="./models/activation.js" hl_lines="4"
async function activateUserByUserId(userId) {
  const activatedUser = await user.setFeatures(userId, [
    "create:session",
    "read:session", // <= Permitindo que um usuário ativado possa ler os dados de sua sessão
  ]);
  return activatedUser;
}
```

Por fim, vamos criar os testes no registration-flow:

```javascript title="./tests/integration/_use-cases/registration-flow.test.js"
  test("Get user information", async () => {
    const responseUserInformation = await fetch(
      "http://localhost:3000/api/v1/user",
      {
        headers: {
          Cookie: `session_id=${createSessionsResponseBody.token}`,
        },
      },
    );
    expect(responseUserInformation.status).toBe(200);
    const responseUserInformationBody = await responseUserInformation.json();
    expect(responseUserInformationBody).toEqual({
      id: createUserResponseBody.id,
      username: "RegistrationFlow",
      email: createUserResponseBody.email,
      features: ["create:session", "read:session"],
      password: createUserResponseBody.password,
      created_at: createUserResponseBody.created_at,
      updated_at: responseUserInformationBody.updated_at,
    });
    expect(uuidVersion(responseUserInformationBody.id)).toBe(4);
    expect(Date.parse(responseUserInformationBody.created_at)).not.toBeNaN();
    expect(Date.parse(responseUserInformationBody.created_at)).not.toBeNaN();
  });
```

!!! warning

    Alguns testes antigos vão começar a falhar. Primeiro que no teste de GET user precisaremos adicionar a ativação do usuário:

    ```javascript
    const activatedUser = await orchestrator.activateUser(createdUser);
    ```

    Além disso, a validação do retorno do usuário estava sendo `features: ["create:session"]`, e agora deverá ser `features: ["create:session", "read:session"]`