# Criando uma página de status

Esse vai ser o nosso primeiro endpoint de API do app, que vai responder por ``/api/v1/status``.

O NextJS adota o file-based routing, o que facilita muito na hora de criarmos as nossas rotas!

Vamos criar a pasta ``/pages/api/v1``, e criar um arquivo lá que nos retorna um JSON:

```javascript title="/pages/api/v1/status.js"
export default function status(request, response){
  response.status(200).json({status:"up"}) 
}
```

Com isso, a nossa página http://localhost:3000/api/v1/status já vai responder um 200 OK. Fácil assim!

## Criando os testes

Essa página ainda não faz muita coisa, mas vamos já criar um teste para ver se ela está ao menos retornando 200.
Vamos criar esses testes já na pasta certa na nossa arquitetura:

```javascript title="/tests/integration/api/v1/status/get.test.js"
test("GET to /api/v1/status should return 200", async () => {
  const response = await fetch("http://localhost:3000/api/v1/status");
  expect(response.status).toBe(200);
});
```

Esse teste vai acessar a nossa API e validar se o retorno é 200.