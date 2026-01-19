# Implementando o Login do Usuário

Agora já temos o nosso usuário conseguindo fazer a ativação da sua conta, é hora de tratarmos o login. A ideia é já linkarmos isso com o futuro sistema de autorização, e na criação da sessão, vamos injetar um usuário, seja ele um usuário Anônimo ou um usuário Autenticado. O objetivo é que no request de criação da sessão, tenhamos dentro dela essa informação.

A forma como implementaremos isso é através de `middlewares`. O middleware vai interceptar a request e incluir essa informação.

## Funcionamento do `middleware`

Vamos primeiramente entender como um middleware funciona. Lá no controller de sessions, vamos adicionar o seguinte teste:

```javascript title="./pages/api/v1/sessions/index.js"
router.use(testeDeLog); //<= o método use() é quem vai invocar o middleware
router.post(postHandler);
router.delete(deleteHandler);

function testeDeLog(request, response, next) {
    console.log("Hora", new Date().toISOString());
    console.log("Path:", request.method, request.url);
    return next();
}
```

O middleware é invocado através do método `use()`, e como argumento a gente vai passar uma função. Essa função precisa ter como assinatura o request, response e next. O next serve para o código continuar depois que a função do middleware for executada. Nesse código, por exemplo, iremos exibir dos dois `console.log`, e em seguida ele continua com o restante (router.post e router.delete). O middleware, portanto, intercepta a execução do código, e pode inclusive modificar valores no request. E é exatamente isso que faremos para injetar o usuário anonimo ou autenticado na sessão!

## Injetando o usuário anonimo ou autenticado

Agora vamos apagar esse exemplo e fazer a implementação real:

```javascript title="./pages/api/v1/sessions/index.js"
router.use(controller.injectAnonymousOrUser);
router.post(postHandler);
router.delete(deleteHandler);
```

!!! tip

    Nesse caso, vamos inserir esse método de injetar o usuário dentro do `controller`. Mas poderia estar também dentro do model de `authentications`. Isso fica meio que a critério de como cada um entende a arquitetura do seu sistema, não tem uma resposta certa para isso.

E agora vamos criar esse método no controller. A lógica será a seguinte:
1. Se o cookie `session_id` existe, injetar o usuário
2. Se o cookie não existir, injetar usuário anomimo

```javascript title="./infra/controller.js"
async function injectAnonymousOrUser(request, response, next) {
  if (request.cookies?.session_id) {
    await injectAuthenticatedUser(request);
    return next();
  }
  injectAnonymousUser(request);
  return next();
}
```

Agora para injetar o usuário autenticado, a lógica é:
1. Buscar a sessão válida
2. Buscar o usuário
3. Injetar ele no request, em uma nova propriedade chamada `context`

```javascript title="./infra/controller.js"
async function injectAuthenticatedUser(request) {
  const sessionToken = request.cookies.session_id;
  const sessionObject = await session.findOneValidByToken(sessionToken);
  const userObject = await user.findOneById(sessionObject.user_id);

  request.context = {
    ...request.context, // <= Para evitar sobrescrever o contexto, e sim adicionar a propriedade
    user: userObject
  }
}
```

Agora para injetar um usuário anônimo é um pouco mais simples. Não precisamos pegar detalhes do usuário, mas sim definir as features que ele terá acesso. Um usuário anônimo (não logado), possui as permissões de:
1. Criar uma conta
2. Ativar uma conta com o activation token que vem no e-mail
3. Criar uma sessão logada

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

Agora podemos escrever um teste simples de login dentro do `registration-flow.test.js`:

```javascript title="./tests/integration/_use-cases/registration-flow.test.js"
  test("Login", async () => {
    const createSessionResponse = await fetch(
      "http://localhost:3000/api/v1/sessions",
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          email: createUserResponseBody.email,
          password: "senha123",
        }),
      },
    );

    expect(createSessionResponse.status).toBe(201);
      const responseBody = await createSessionResponse.json();
      expect(responseBody).toEqual({
        id: responseBody.id,
        token: responseBody.token,
        user_id: createUserResponseBody.id,
        created_at: responseBody.created_at,
        updated_at: responseBody.updated_at,
        expires_at: responseBody.expires_at,
      });
  });
```

!!! note
    
    Então nesse teste, temos um usuário deslogado (anônimo) fazendo um POST request para o `/sessions`. Esse request será interceptado pelo `middlware`, que vai chamar a função `injectAnonymousUser` (já que não temos nenhum cookie de sessão ainda), que por sua vez vai injetar uma casca de um usuário no `context`, contendo apenas as `features` que um usuário anônimo tem acesso. Sendo assim, se colocarmos um `console.log(request.context)` no `postHandler()`, teremos isso:
    ```bash
    {
    user: {
        features: [ 'read:activation_token', 'create:session', 'create:user' ]
    }
    }    
    ```

## Fazendo a verificação de permissão

Agora que estamos injetando o usuário, seja ele anonimo ou autenticado, já temos acesso à lista de feature que ele possui. O próximo passo é saber se esse usuário possui permissão sobre a request que ele está tentando fazer.

Para programar isso de uma forma que fique reutilizável em todos os requests, vamos criar um middlware chamado `canRequest()` dentro do `controller.js`. E a forma de usarmos esse middleware, é passando ele como parâmetro no router.post, router.delete, etc, assim:

```javascript title="./pages/api/v1/sessions/index.js" hl_lines="2"
router.use(controller.injectAnonymousOrUser);
router.post(controller.canRequest, postHandler);
router.delete(deleteHandler);
```

Mas a gente precisa passsar como parâmetro pra esse método qual é a permissão que ele necessita, que no caso seria a "create:session":
```javascript title="./pages/api/v1/sessions/index.js" hl_lines="2"
router.use(controller.injectAnonymousOrUser);
router.post(controller.canRequest("create:session"), postHandler);
router.delete(deleteHandler);
```

Mas veja que se eu chamar assim, não estamos passando apenas a função como parâmetro, estamos executando ela! E o que se espera é apenas a função, sem a execução dela. Então o que precisaremos criar aqui é uma `Função de alta ordem`, ou `higher-order function`, que é uma função que retorna uma outra função.

```javascript title="./infra/controller.js"
function canRequest(feature) {
  return function canRequestMiddleware(request, response, next) {
    const userTryingToRequest = request.context.user;

    if (userTryingToRequest.features.includes(feature)) {
      return next();
    }

    throw new ForbiddenError({
      message: "Você não possui permissão para executar esta ação.",
      action: `Verifique se o seu usuário possui a feature: "${feature}"`,
    });
  };
}
```

!!! note

    Olha que interessante o funcionamento do `canRequest`. Ela é uma função que recebe a feature que queremos validar como parâmetro, e ela retorna uma outra função chamada `canRequestMiddleware`, que é um `middleware`. Ou seja, ela recebe o request, response e o next, como vimos na função `injectAnonymousOrUser`. Com isso, essa função canRequestMiddleware tem acesso ao que foi interceptado da request (e por consequência tem o atributo `context` que injetamos logo antes com o middleware `injectAnonymousOrUser`), e com isso temos acesso às features que o usuário injetado tem permissão. Agora basta checarmos se dentro da lista de permissão do usuário injetado temos a feature que queremos validar, e caso não tenha, lançamos o erro `403 Forbidden`.

O erro `403 Forbidden` ainda não existe, mas é só criá-lo dentro do `errors.js`:

```javascript title="./infra/errors.js"
export class ForbiddenError extends Error {
  constructor({ cause, message, action }) {
    super(message || "Acesso negado.", {
      cause: cause,
    });
    this.name = "ForbiddenError";
    this.action =
      action || "Verifique as features necessárias antes de continuar.";
    this.statusCode = 403;
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

E incluímos ele no OR do onErrorHandler do `controller.js`, assim como sempre fazemos com os demais erros que criamos para ele acabar não caindo no InternalServerError genérico. Isso não é novidade:

```javascript title="./infra/controller.js" hl_lines="2"
function onErrorHandler(error, request, response) {
  if (error instanceof ValidationError || error instanceof NotFoundError || error instanceof ForbiddenError) {
    return response.status(error.statusCode).json(error);
  }
  if (error instanceof UnauthorizedError) {
    clearSessionCookie(response);
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

!!! success

    Agora temos o nosso usuário sendo capaz de efetuar o login, e com isso temos também a estrutura do sistema de autenticação montado! Para testar, você pode tentar remover a feature `create:session` da lista criada na função `injectAnonymousUser`. Assim, o usuário anônimo injetado não vai ter a permissão de criar sessão, e esse POST do Login vai falhar com o erro 403.