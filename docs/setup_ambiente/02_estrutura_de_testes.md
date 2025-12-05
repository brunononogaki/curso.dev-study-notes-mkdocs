# Iniciando o setup do Test Runner (Jest)

Agora vamos começar a definir como faremos os testes. Adoraremos o test runner ``Jest``.

## Instalando o Jest

Para instalar o Jest como dependência do projeto, é só dar um ``npm install``. Mas instalaremos ele como dependência de dev, e não de produção, através da flag ``--save-dev``

```bash
npm install --save-dev jest@29.6.2
```

Agora vamos editar o ``package.json`` para incluirmos atalhos para os testes. Já vamos definir a flag ``--runInBand``, que faz com que ele rode os testes sequenciais e não paralelizados, o que pra gente facilita a vida principalmente quando formos testar integração com o banco de dados.

```javascript title="package.json"
  "scripts": {
    "dev": "next dev",
    "test": "jest --runInBand --verbose",
    "test:watch": "jest --watchAll --runInBand --verbose"
  },
```

## Criando um teste de teste

Vamos criar um teste simples, que não testa nada na verdade, mas é só pra ver o Jest em ação.
Dentro da pasta tests, vamos criar um arquivo chamado ``exemplo.test.js``:

```javascript title="/tests/exemplo.test.js"
test("teste 1", () => {
  expect(1).toBe(1);
});
```


Esse teste vai simplesmente testar que 1 == 1. Bem útil!

Para rodar o teste:

```bash
npm test

> teste_next@1.0.0 test
> jest --runInBand

 PASS  tests/exemplo.test.js
  ✓ teste 1 (1 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.165 s
Ran all test suites.
```

Ou podemos rodar no modo watch, que vai ficar rodando o código sempre que tiver uma alteração:

```bash
 PASS  tests/exemplo.test.js
  ✓ teste 1 (3 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.241 s, estimated 1 s
Ran all test suites related to changed files.

Watch Usage
 › Press a to run all tests.
 › Press f to run only failed tests.
 › Press p to filter by a filename regex pattern.
 › Press t to filter by a test name regex pattern.
 › Press q to quit watch mode.
 › Press Enter to trigger a test run.

```

## Usando `describe` nos testes
Uma funcionalidade do Jest que nos ajuda a descrever melhor os testes é a função `describe`, que pode ser usada assim:
```javascript
describe("GET /api/v1/migrations", () => {
  describe("Anonymous user", () => {
    test("Getting pending migrations", async () => {
      const response = await fetch("http://localhost:3000/api/v1/migrations");
      expect(response.status).toBe(200);

      const responseBody = await response.json();
      // console.log(responseBody);
      expect(Array.isArray(responseBody)).toBe(true);
      expect(responseBody.length).toBeGreaterThan(0);
    });
  });
});
```

Se rodarmos o Jest com o modo `--verbose`, o resultado será assim:

```bash
 PASS  tests/integration/api/v1/migrations/get.test.js
  GET /api/v1/migrations
    Anonymous user
      ✓ Getting pending migrations (19 ms)
```

Agora mais pra frente vamos começar a criar muitos testes. Optaremos pelos testes de integração, e não tanto testes unitários.
