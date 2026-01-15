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
