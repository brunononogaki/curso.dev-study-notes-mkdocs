# Implementando a Ativação de Conta do Usuário

Com o sistema de e-mail configurado e pronto para testes, vamos implementar a ativação de conta do usuário. Esse é o fluxo que queremos implementar e testar:

1. Usuário cria a conta
2. Usuário recebe um e-mail para ativar esta conta
3. Usuário clica no link dentro do e-mail, ativa a conta e recebe as credenciais base
4. Usuário consegue criar uma nova sessão no sistma
5. Após a sessão criada, ele conssegue executar ações contra a API nos endpoints que precisam de credencial

A ideia é criarmos um teste automatizado que cobre tudo isso, e vamos implementando teste a teste até tudo funcionar!

## Lidando com as permissões do usuário

Mas a primeira pergunta que precisamos nos fazer é: "ok, depois que o usuário ativar a conta, o que acontece? Como eu vou saber que ele é um usuário ativado?"

Uma possibilidade de implementação é criarmos uma coluna chamada `isActive` na tabela de Users, por exemplo, e guardar um valor booleano. Mas ao invés disso, vamos já começar a mesclar essa ativação com o sistema de Autorização (que iremos implementar mais pra frente).

No sistema de autorização, planejamos ter uma coluna chamada `features` na tabela Users, e essa coluna será do tipo Array de strings. Nesse array, vamos armazenar todas as capacidades que o usuário vai ter. Definiremos um padrão para essa string, e a primeira permissão vai ser justamente para ver se o usuário está ou não ativo.

Então de cara, vamos criar essa coluna nova a partir de uma migration:

```bash
npm run migrations:create add features to users
```

E no arquivo de migrations que foi criado, vamos adicionar essa coluna:

```javascript title="./infra/migrations/1768438712879_add-features-to-users.js"
exports.up = (pgm) => {
  pgm.addColumn("users", {
    features: {
      type: "varchar[]",
      notNull: true,
      default: "{}",
    },
  });
};

exports.down = false;
```

Bom, mas todos os usuários precisam iniciar no sistema com uma feature padrão, que chamaremos de `read:activation_token`. Essa "feature" vai nos indicar que o usuário possui permissão de leitura na rota de `activation_token`, porque ele ainda não está ativado. Se o usuário possuir essa feature, sabemos que é um usuário novo que ainda não fez a ativação por e-mail.

Vamos configurar lá no POST de criação de usuário para ele já ser criado com essa feature por padrão:

```javascript title="./models/users.js" hl_lines="5 15 22 27-29"
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

!!! warning

    A criação desse novo campo vai quebrar todos os nossos testes que fazem o assertion do retorno de Users, porque agora a API retornará também esse novo valor. Para corrigir, precisamos incluir esse assertion nos testes, por exemplo:
    ```javascript hl_lines="5"
    expect(responseBody).toEqual({
      id: responseBody.id,
      username: "bruno.nonogaki",
      email: "brunono@email.com",
      features: ["read:activation_token"],
      password: responseBody.password,
      created_at: responseBody.created_at,
      updated_at: responseBody.updated_at,
    });
    ```

!!! success

    Pronto, a base do nosso sistema de autorização está pronta. Agora vamos começar a fazer o teste completo de registro de um usuário, e a sua ativação.

## Criando a estrutura do teste

Vamos iniciar criando um teste chamado `registration-flow.test.js`. Nesse teste, vamos inicialmente criar um usuário (copiando dos testes que já fizemos no endpoint `/users`), e depois vamos criando os demais testes, que por hora vamos deixar em branco:

```javascript title="./tests/integration/_use-cases/registration-flow.test.js"
import orchestrator from "tests/orchestrator.js";

beforeAll(async () => {
  await orchestrator.waitForAllServices();
  await orchestrator.clearDatabase();
  await orchestrator.runPendingMigrations();
  await orchestrator.deleteAllEmails();
});

describe("Use case: Registration Flow (all successful)", () => {
  test("Create user account", async () => {
    const createUserResponse = await fetch(
      "http://localhost:3000/api/v1/users",
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          username: "RegistrationFlow",
          email: "registration.flow@email.com",
          password: "senha123",
        }),
      }
    );
    expect(createUserResponse.status).toBe(201);

    const createUserResponseBody = await createUserResponse.json();

    expect(createUserResponseBody).toEqual({
      id: createUserResponseBody.id,
      username: "RegistrationFlow",
      email: "registration.flow@email.com",
      password: createUserResponseBody.password,
      created_at: createUserResponseBody.created_at,
      updated_at: createUserResponseBody.updated_at,
    });
  });

  test("Receive activation email", async () => {});
  test("Activation account", async () => {});
  test("Login", async () => {});
  test("Get user information", async () => {});
});
```

## Enviando o e-mail de ativação

Agora vamos nos focar no teste `Receive activation email`, que ainda está em branco. Aqui a gente quer validar que depois do registro, o usuário vai receber um e-mail de ativação. A gente já sabe testar isso, pegando o último e-mail da caixa lá no `Mailcatcher`, então bora la:

```javascript title="./tests/integration/_use-cases/registration-flow.test.js"
// ...
  test("Receive activation email", async () => {
    const lastEmail = await orchestrator.getLastEmail();
    expect(lastEmail.sender).toBe("<contato@meubonsai.app>");
    expect(lastEmail.recipients[0]).toBe("<registration.flow@email.com>");
    expect(lastEmail.subject).toBe("Ative seu cadastro no MeuBonsai.App");
    expect(lastEmail.text).toContain("RegistrationFlow")
  });
```

Certamente esse teste vai falhar, porque ainda não estamos enviando e-mail nenhum! Vamos programar isso! Mas... onde podemos colocar essa lógica de envio de e-mail após a criação de um usuário. Poderíamos colocar dentro do `model` user, fazendo que sempre que eu crie um usuário na base o sistema envie um e-mail; ou daria para colocar dentro do controller `/users`, após a chamada do método do `create()`. Como temos casos de testes automatizados chamando direto o model para criar um usuário, e como nesses casos a gente não precisa enviaar e-mail nenhum, pois queremos simplesmete que um usuário seja criado na base, optaremos por criar essa chamada dentro do controller `/users`.

```javascript title="./pages/api/v1/users/index.js" hl_lines="4 15"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import user from "models/user.js";
import activation from "models/activation";

const router = createRouter();

router.post(postHandler);

export default router.handler(controller.errorHandler);

async function postHandler(request, response) {
  const userInputValues = request.body;
  const newUser = await user.create(userInputValues);
  await activation.sendEmailToUser(newUser);
  return response.status(201).json(newUser);
}
```

Aqui a gente especulou um novo model chamado `activation`, que vai ter essa lógica de gerar um token, enviar e-mail, etc. Vamos criá-lo e já criar esse método `sendEmailToUser`:

```javascript title="./models/activation.js"
import email from "infra/email.js"

async function sendEmailToUser(user) { 
  await email.send({
    from: "Contato <contato@meubonsai.app>",
    to: user.email,
    subject: "Ative seu cadastro no MeuBonsai.App",
    text: `${user.username}, clique no link abaixo para ativar seu cadastro no MeuBonsai.App

https://link

Atenciosamente,

Equipe MeuBonsai.App  
    `,
  })
}

const activation = {
  sendEmailToUser
}

export default activation
```

!!! success

    Show, já estamos enviando o e-mail de ativação, e os testes estão passando! Mas ainda falta gerar um token e um link de verdade para o usuário poder ativar sua conta. Faremos isso em seguida!


## Criando o Token de Ativação

Agora vamos começar a gerar dinamicamente o Token de ativação para ser enviado para o usuário.

### Criando a tabela no banco de dados
Pecisaremos persistir esse token em algum lugar, e ter o controle se ele já foi usado ou não. Por isso, vamos criar uma nova tabela na base de dados através de uma migration

```bash
npm run migrations:create create user activation tokens 
```

E definir esse arquivo de migrations:
```javascript title="./infra/migration/1768559591990_create-user-activation-tokens.js"
exports.up = (pgm) => {
  pgm.createTable("user_activation_tokens", {
    id: {
      type: "uuid",
      primaryKey: true,
      default: pgm.func("gen_random_uuid()"),
    },
    user_id: {
      type: "uuid",
      notNull: true,
    },
    used_at: {
      type: "timestamptz",
      notNull: false,
    },   
    expires_at: {
      type: "timestamptz",
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

### Criando o método de criação do token e enviando o token por e-mail
Agora que já temos a tabela criada, podemos criar o token exatamente da mesma forma que no passado criamos as sessões, com a diferença que ele terá uma validade de 15 minutos ao invés de 30 dias. Além disso, como definimos que o token será o próprio ID da coluna, que é gerado dinamicamente pelo Postgres, nem precisamos nos preocupar em gerar um ID randômico pelo código. De resto, é meio que cópia do que temos no método de `create()` de `sessions.js`:

```javascript title="./models/activation.js"
import email from "infra/email.js";
import database from "infra/database.js";

const EXPIRATION_IN_MILLISECONDS = 60 * 15 * 1000 // 15 minutes

async function create(userId) {
  const expiresAt = new Date(Date.now() + EXPIRATION_IN_MILLISECONDS);

  const newToken = await runInsertQuery(userId, expiresAt);
  return newToken;

  async function runInsertQuery(userId, expiresAt) {
    const results = await database.query({
      text: `
        INSERT INTO
          user_activation_tokens (user_id, expires_at)
        VALUES
          ($1, $2)
        RETURNING *
      ;`,
      values: [userId, expiresAt],
    });
    return results.rows[0];
  }
}

const activation = {
  sendEmailToUser,
  create
};

export default activation;
```

E agora lá no controller de `/users`, podemos chamar esse método, salvando o token na variável `activationToken`, e depois passá-la como parâmetro no método `sendEmailToUser()`, para podermos colocar esse valor no link que chega no e-mail:

```javascript title="./pages/api/v1/users/index.js" hl_lines="16-17"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import user from "models/user.js";
import activation from "models/activation";

const router = createRouter();

router.post(postHandler);

export default router.handler(controller.errorHandler);

async function postHandler(request, response) {
  const userInputValues = request.body;
  const newUser = await user.create(userInputValues);

  const activationToken = await activation.create(newUser.id);
  await activation.sendEmailToUser(newUser, activationToken);
  return response.status(201).json(newUser);
}
```

Agora vamos adicionar o token no link:

```javascript title="./models/activation.js"
import email from "infra/email.js"

async function sendEmailToUser(user, activationToken) { 
  await email.send({
    from: "Contato <contato@meubonsai.app>",
    to: user.email,
    subject: "Ative seu cadastro no MeuBonsai.App",
    text: `${user.username}, clique no link abaixo para ativar seu cadastro no MeuBonsai.App

https://meubonsai.app/cadastro/ativar/${activationToken.id}

Atenciosamente,

Equipe MeuBonsai.App  
    `,
  })
}

const activation = {
  sendEmailToUser
}

export default activation
```

!!! tip

    Para deixarmos esse domínio da URL dinânico, dependendo se estamos no ambiente de Produção, Homologação ou Desenvolvimento, podemos criar um arquivo chamado webserver.js dentro de `infra`, que vai ter um método `getOrigin`, que nos retornará esse valor:

    ```javascript title="./infra/webserver.js"
    function getOrigin() {
      if (["test", "development"].includes(process.env.NODE_ENV)) {
        return "http://localhost:3000";
      }

      if (process.env.VERCEL_ENV === "preview") {
        return `https://${process.env.VERCEL_URL}`;
      }

      return "https://naquelesdias.com.br";
    }

    const webserver = {
      getOrigin,
    };

    export default webserver;
    ```

    Aí basta importarmos esse módulo e substituir a URL hard-coded por `webserver.getOrigin()`


### Fazendo a validação do Token nos testes

Agora precisamos testar esse fluxo até agora. A ideia desse teste é:
* Extrair o token que vem no link do e-mail
* Procura esse token no banco de dados para ver se ele é válido (não expirado e nào utilizado)
* Ver se o user id atrelado a esse token é o mesmo id que fez o cadastro

Primeiramente, vamos criar um método no orchestrator que consegue extrair um UUID de um texto:

```javascript title="./tests/orchestrator.js"
function extractUUID(text) {
  const match = text.match(/[0-9a-fA-F-]{36}/);
  return match ? match[0] : null;
}
```

E agora vou criar um método findOneValidById no `activation.js`, que será capaz de procurar o token na base de dados e encontrar um que esteja válido.

```javascript title="./models/activation.js"
async function findOneValidById(activationToken) {
  const tokenFound = await runSelectQuery(activationToken);
  return tokenFound;

  async function runSelectQuery(activationToken) {
    const results = await database.query({
      text: `
        SELECT 
          *
        FROM
          user_activation_tokens
        WHERE
          id = $1
          AND expires_at > NOW()
          AND used_at is NULL
        LIMIT 1
      `,
      values: [activationToken],
    });
    if (results.rowCount === 0) {
      throw new NotFoundError({
        message: "Token de ativação não encontrado.",
        action:
          "Verifique se este token de ativação não está expirado ou não foi utilizado.",
      });
    } else {
      return results.rows[0];
    }
  }
}
```

E por fim, vamos incrementar nos testes:

```javascript title="./tests/integration/_use-cases/registration-flow.test.js"
describe("Use case: Registration Flow (all successful)", () => {
  let createUserResponseBody; //<= Precisamos mover a declaração da variável para fora dos testes, pois ela será usada nos dois testes criados.
  // ...

  test("Receive activation email", async () => {
    const lastEmail = await orchestrator.getLastEmail();
    expect(lastEmail.sender).toBe("<contato@meubonsai.app>");
    expect(lastEmail.recipients[0]).toBe("<registration.flow@email.com>");
    expect(lastEmail.subject).toBe("Ative seu cadastro no MeuBonsai.App");
    expect(lastEmail.text).toContain("RegistrationFlow");

    const activationTokenId = orchestrator.extractUUID(lastEmail.text);
    expect(lastEmail.text).toContain(
      `${webserver.getOrigin()}/cadastro/ativar/${activationTokenId}`,
    );

    const activationTokenObject =
      await activation.findOneValidById(activationTokenId);
    expect(activationTokenObject.user_id).toBe(createUserResponseBody.id);
    expect(activationTokenObject.used_at).toBe(null);
  });
```

## Ativando a conta

Agora, para concluir o fluxo de ativação, o usuário deverá enviar um `PATCH` para o endpoint `/api/v1/activations/[token_id]`, que fará a alteração do campo `used_at` com a data atual. Além disso, vamos remover a feature `[read:activation_token]` do usuário, e adicionar a feature `[create:session]`. A página pública que a gente retorna no e-mail ainda não existe e será criada depois, mas podemos deixar o endpoint da API pronto.

### Criando os testes

Agora vamos atacar os testes de Activation account que vai fazer tudo isso que comentamos acima:
```javascript title="./tests/integration/_use-cases/registration-flow.test.js"
describe("Use case: Registration Flow (all successful)", () => {
  let createUserResponseBody;
  let activationTokenObject; // <= Precisamos mover também a declaração da variável activationTokenObject para fora dos testes para podermos reaproveitá-la nesse
  // ...

  test("Activation account", async () => {
    const activationResponse = await fetch(
      `http://localhost:3000/api/v1/activations/${activationTokenObject.id}`,
      {
        method: "PATCH",
      },
    );
    expect(activationResponse.status).toBe(200);

    const activationResposeBody = await activationResponse.json();
    expect(Date.parse(activationResposeBody.used_at)).not.toBeNull();

    const activatedUser = await user.findOneByUsername("RegistrationFlow");
    expect(activatedUser.features).toEqual(["create:session"]);
  });
```

Então estamos testando se conseguimos rodar o `PATCH`, se ele está preenchendo o campo `used_at`, e se o usuário possui somente a feature `create:session`

### Criando o controller `/activations/[token_id]`

Agora vamos criar a pasta `activations/[token_id]` dentro de `api/v1` para criar a nossa rota, e criar o index.js dentro:

```javascript title="./pages/api/v1/activations/[token_id]/index.js"
import { createRouter } from "next-connect";
import controller from "infra/controller.js";
import activation from "models/activation";

const router = createRouter();

router.patch(patchHandler);

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

De forma similar ao que fizemos na rota `api/v1/users/[username]`, usamos esse placeholder `[token_id]` para passarmos o token que queremos ativar.
O controller então vai validar se o token existe e é válido, marcá-lo como usado (alterando o campo used_at), e depois ativar o usuário (atribuindo a feature `create:session`). O retorno no caso de sucesso será um `200 OK`, com os dados do token ativdado (com o `used_at` preenchido)

### Criando os métodos nos models

Agora no model `activation.js`, vamos criar o método `markTokenAsUsed()`, que recebe o ID do token e altera o campo used_at dele:
```javascript title="./models/activation.js"
async function markTokenAsUsed(activationTokenId) {
  const updatedToken = await runUpdateQuery(activationTokenId);
  return updatedToken;

  async function runUpdateQuery(activationTokenId) {
    const results = await database.query({
      text: `
        UPDATE 
          user_activation_tokens
        SET
          used_at = timezone('utc', now()), 
          updated_at = timezone('utc', now())
        WHERE
          id = $1
        RETURNING *
      `,
      values: [activationTokenId],
    });
    if (results.rowCount === 0) {
      throw new NotFoundError({
        message: "Token de ativação não encontrado.",
        action:
          "Verifique se este token de ativação não está expirado ou não foi utilizado.",
      });
    } else {
      return results.rows[0];
    }
  }
}
```

E vamos criar o método `activateUserByUserId()`, que recebe o ID de um usuário e a feature desejada, e altera esse dado no banco:

```javascript title="./models/activation.js"
async function activateUserByUserId(userId) {
  const activatedUser = await user.setFeatures(userId, ["create:session"]);
  return activatedUser;
}
```

E aqui um pequeno detalhe. Como isso afeta o usuário, vamos abstrair isso dentro de um novo método chamado `setFeatures`, que faz parte do model de `user`:

```javascript title="./models/user.js"
async function setFeatures(userId, features) {
  const updatedUser = await runUpdateQuery(userId, features);
  return updatedUser;

  async function runUpdateQuery(userId, features) {
    const results = await database.query({
      text: `
        UPDATE 
          users
        SET
          features = $2, 
          updated_at = timezone('utc', now())
        WHERE
          id = $1
        RETURNING *
      `,
      values: [userId, features],
    });

    return results.rows[0];
  }
}
```

Tudo sem segredo nenhum, é tudo coisa que já fizemos em outros endpoints! 

!!! success

    Sucesso, o nosso sistema de ativação de contas já está funcionando. Agora podemos seguir para o usuário conseguir fazer o login, e acessar algum endpoint protegido na API!
