# MailCatcher e envio de e-mails

Agora chegou a hora de implementarmos um sistema de envio de e-mails, e mais do que isso, subir uma infraestrutura interna de e-mails, com um servidor chamado `Mailcatcher`, que consegue falar o protocolo SMTP, e disponibiliza uma "mailbox" online para consultarmos esses e-mails.

## Instalando o `Mailcatcher`

A instalação dele é muito simples. Basta adicionarmos a declaração desse container no nosso arquivo de `compose.yaml`, para subir junto com o Banco de Dados:

```yaml title="./infra/compose.yaml" hl_lines="16-27"
services:
  database:
    container_name: postgres-dev
    image: postgres:17.0
    env_file:
      - ../.env.development
    ports:
      - "5432:5432"
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  mailcacher:
    container_name: mailcacher-dev
    image: sj26/mailcatcher
    ports:
      - "1025:1025"
      - "1080:1080"
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

!!! success

    E pronto, agora ao dar o comando `npm run dev`, esse container já será inicializado! É possível acessar a "mailbox" no endereço `http://localhost:1080`

!!! tip

    Para testar o envio de e-mails, é possível fazer um telnet na porta 1025 do Mailcatcher:
    ```bash
    telnet localhost 1025

    Trying ::1...
    Connected to localhost.
    Escape character is '^]'.
    220 EventMachine SMTP Server
    ```

    E agora esses comandos farão nos comunicarmos via SMTP:
    ```bash hl_lines="1 3 5 7 9-12 14"
    HELO
    250 Ok EventMachine SMTP Server
    MAIL FROM:<bruno.nonogaki@gmail.com>
    250 Ok
    RCPT TO:<brunono@gmail.com>
    250 Ok
    DATA
    354 Send it
    Subject: Teste por Telnet

    Corpo do email.
    .
    250 Message accepted
    quit
    221 Ok
    Connection closed by foreign host.
    ```

    Agora, ao abrir a nossa Mailbox, o e-mail estará lá. Ele não vai enviar o e-mail de verdade, ele só está capturando o e-mail para podermos usar isso nos nossos testes!

## Instalando o `nodemailer` e criando o módulo `email.js`

O `nodemailer` é um módulo do NPM para abstrair o envio de e-mails, que é basicamente a comunicação que simulamos agora via telnet. Vamos adicioná-lo como dependência do projeto:

```bash
npm install -E nodemailer@7.0.5
```

Agora vamos construir o módulo `email.js` na pasta `infra`, similar ao que fizemos por exemplo com o módulo do `database.js`:

```javascript title="./infra/email.js"
import nodemailer from "nodemailer";

async function send() {}

const email = {
  send,
};

export default email;
```

## Testando envio de e-mails

Por enquanto nossa função `send` está vazia, e vamos começar a codificá-la. Mas antes, vamos entender como vamos fazer para testar isso.
A real utilização do módulo `email.js` virá mais pra frente, quando o usuário tiver que confirmar o seu cadastro, por exemplo. Mas ainda não temos isso implementado. Uma solução seria escrever um código temporário para ficar chamando esse módulo, mas como já temos os testes automatizados meio que fazendo isso, podemos já começar a escrever direto em um teste! Para isso, dentro da pasta `./tests/integration`, vamos criar uma pasta nova chamada `infra`, e dentro dela o arquivo `email.test.js`:

```javascript title="./tests/integration/email.test.js"
import email from "infra/email.js";

describe("Test infra/email.js", () => {
  test("send()", async () => {
    await email.send();
  });
});
```

Pronto, a estrutura está montada. Ao fazer um `npm run test:watch --email`, ficaremos executando o método `send()` do módulo que criamos. Por enquanto sem assertions e nem nada, só queremos chamar o método e ver se conseguimos enviar um e-mail pelos testes. Agora vamos programar isso no método. É só criar um `transporter`, usando as variáveis do .env, e depois invocar o método `sendMail`, passando um objeto com as configurações do e-mail:

```javascript title="./infra/email.js"
import nodemailer from "nodemailer";

const transporter = nodemailer.createTransport({
  host: process.env.EMAIL_SMTP_HOST,
  port: process.env.EMAIL_SMTP_PORT,
  auth: {
    user: process.env.EMAIL_SMTP_USER,
    pass: process.env.EMAIL_SMTP_PASSWORD,
    secure: process.env.NODE_ENV === "production" ? true : false,
  },
});

async function send(mailOptions) {
  await transporter.sendMail(mailOptions);
}

const email = {
  send,
};

export default email;
```

No nosso `.env.development`, adicionaremos isso:

```bash title="./.env.development"
EMAIL_SMTP_HOST=0.0.0.0
EMAIL_SMTP_PORT=1025
EMAIL_SMTP_USER=
EMAIL_SMTP_PASSWORD=
```

E por fim, vamos passar o objeto de configuração do email para o método `send()`:

```javascript title="./tests/integration/email.test.js"
import email from "infra/email.js";

describe("Test infra/email.js", () => {
  test("send()", async () => {
    await email.send({
      from: "MeuBonsai <contato@meubonsai.app>",
      to: "contato@brunononogaki.com",
      subject: "Teste de email",
      text: "Text de corpo",
    });
  });
});
```

!!! success

    Sucesso, agora já estamos conseguindo fazer o envio do e-mail pelo `mailcatcher`!

## Finalizando o teste de integração

Agora vamos acrescentar algumas coisas no nosso teste, como limpar a caixa de entrada a cada início de teste (como fazemos com a Database), e uma função para pegar o ultimo e-mail da caixa, e assim podermos fazer os assertions. O `mailcatcher` disponibiliza uma interface via API para executarmos essas operações, então vamos adicionar mais duas variáveis no nosso `.env`:

```bash title="./.env.development"
EMAIL_HTTP_HOST=0.0.0.0
EMAIL_HTTP_PORT=1080
```

Agora vamos criar a função `deleteAllEmails()` no `orchestrator.js`:

```javascript title="./tests/orchestrator.js"
async function deleteAllEmails() {
  await fetch(
    `http://${process.env.EMAIL_HTTP_HOST}:${process.env.EMAIL_HTTP_PORT}/messages`,
    {
      method: "DELETE",
    }
  );
}
```

E agora chamar essa função no `beforeAll` dos testes de e-mail:

```javascript title="./tests/integration/email.test.js" hl_lines="2 4-6"
import email from "infra/email.js";
import orchestrator from "tests/orchestrator";

beforeAll(async () => {
  await orchestrator.deleteAllEmails();
});

describe("Test infra/email.js", () => {
  test("send()", async () => {
    await email.send({
      from: "MeuBonsai <contato@meubonsai.app>",
      to: "contato@brunononogaki.com",
      subject: "Teste de email",
      text: "Text de corpo",
      // html:
    });
  });
});
```

!!! success

    Agora toda vez que iniciarmos os testes, a caixa de entrada será limpada!

Vamos agora implementar a função `getLastEmail()`:

```javascript title="./tests/orchestrator.js"
async function getLastEmail() {
  // Collect all messages in the mailbox
  const emailListResponse = await fetch(
    `http://${process.env.EMAIL_HTTP_HOST}:${process.env.EMAIL_HTTP_PORT}/messages`
  );
  const emailListBody = await emailListResponse.json();
  // Get the last item
  const lastEmailItem = emailListBody.pop();

  // Get the text of this email
  const emailTextResponse = await fetch(
    `http://${process.env.EMAIL_HTTP_HOST}:${process.env.EMAIL_HTTP_PORT}/messages/${lastEmailItem.id}.plain`
  );

  // Add the email text in the response payload
  const emailTextBody = await emailTextResponse.text();
  lastEmailItem.text = emailTextBody;

  return lastEmailItem;
}
```

!!! tip

    Pela API do `mailcatcher`, precisamos primeiramente coletar a lista de emails da fila, mas o retorno dessa API não vai trazer o texto do e-mail. Então depois de identificarmos o Id o último e-mail (mais recente), enviamos um outro GET, mas adicionando o ID do email na URL. Essa request sim nos trará o corpo do email. E outro detalhe é que a lista de e-mails retornada na nossa primeira requisição é ordenada do mais velho para o mais recente. Por isso utilizamos o método `pop()`, para trazer o último elemento da lista

Agora sim podemos invocar esse método no nosso teste, e fazer um assertion do conteúdo do e-mail. Nesse teste, vamos fazer o envio de dois e-mails, para garantir que estamos pegando sempre o último (mais recente):

```javascript title="./tests/integration/email.test.js"
import email from "infra/email.js";
import orchestrator from "tests/orchestrator";

beforeAll(async () => {
  await orchestrator.deleteAllEmails();
});

describe("Test infra/email.js", () => {
  test("send()", async () => {
    await email.send({
      from: "MeuBonsai <contato@meubonsai.app>",
      to: "contato@brunononogaki.com",
      subject: "Teste de email",
      text: "Text de corpo",
    });

    await email.send({
      from: "MeuBonsai <contato@meubonsai.app>",
      to: "contato@brunononogaki.com",
      subject: "Último email enviado",
      text: "Text de corpo",
    });

    const lastEmail = await orchestrator.getLastEmail();
    expect(lastEmail.sender).toBe("<contato@meubonsai.app>");
    expect(lastEmail.recipients[0]).toBe("<contato@brunononogaki.com>");
    expect(lastEmail.subject).toBe("Último email enviado");
    expect(lastEmail.text).toBe("Text de corpo\n");
  });
});
```

## Esperando o serviço de e-mail subir

Por fim, da mesma forma que fizemos o `waitForWebServer` no `orchestrator`, garantindo que os testes iniciariam somente depois de o webserver estar de pé, vamos fazer com o serviço de e-mail. Vamos adicionar o método `waitForEmailServer()`:

```javascript title="./tests/orchestrator.js" hl_lines="3 19-31"
async function waitForAllServices() {
  await waitForWebServer();
  await waitForEmailServer();

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

  async function waitForEmailServer() {
    return retry(fetchStatusPage, {
      retries: 100,
      maxTimeout: 1000,
    });

    async function fetchStatusPage() {
      const response = await fetch(
        `http://${process.env.EMAIL_HTTP_HOST}:${process.env.EMAIL_HTTP_PORT}`
      );
      if (response.status !== 200) {
        throw Error();
      }
    }
  }
}
```

E agora vamos adicionar isso no `beforeAll` dos testes:

```javascript title="./tests/integration/email.test.js" hl_lines="6"
import email from "infra/email.js";
import orchestrator from "tests/orchestrator";

beforeAll(async () => {
  await orchestrator.deleteAllEmails();
  await orchestrator.waitForAllServices();
});
```

!!! success

    Pronto, temos agora a infraestrutura de e-mails pronta, com os métodos sendo testados!
