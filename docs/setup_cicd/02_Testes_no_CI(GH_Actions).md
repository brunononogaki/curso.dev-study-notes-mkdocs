# Criando os testes no CI com GitHub Actions

## Criando o Workflow de Teste
Agora que já temos os testes estabilizados no nosso ambiente local, já estamos prontos para rodá-los em um servidor remoto, através de um Workflow de CI no GitHub Actions.

Na raíz do nosso projeto, vamos criar uma pasta especial chamada `.github`, e dentro dela uma outra chamada `workflows`. E nessa pasta `workflows`, vamos criar o nosso primeiro fluxo, que chamaremos de `tests.yaml`:

```yaml title=".github/workflows/tests.yaml"
name: Automated tests

on: pull_request

jobs:
  jest:
    name: Jest Ubuntu
    runs-on: ubuntu-latest
    steps:
      # Faz o download do código, através de uma action chamada checkout
      - uses: actions/checkout@v4

      # Faz o setup de um ambiente Node, através de uma action chamada setup-node
      - uses: actions/setup-node@v4
        with:
          node-version: "lts/hydrogen"

      # O npm ci usa estritamente o que está no package-lock.json, diferente do npm install que pode pegar pacotes atualizados, e aí os ambiente de teste poderia mudar com o tempo
      - run: npm ci 

      # Executa o npm test
      - run: npm test
```

Agora, vamos criar uma branch nova chamada `actions`, e fazer o push para ela:
```bash
git checkout -b actions
git add .
git commit -m "Adding tests workflow"
git push --set-upstream origin actions
```

E lá no GitHub, vamos abrir um Pull Request da branch `action` para a branch `main`. Ao fazer isso, esse fluxo de teste é chamado.

## Protegendo a Branch main
Caso os testes falhem, ainda assim o GitHub vai permitir fazermos o merge para a main. O ideal seria isso não ser possível, pois não queremos subir um código quebrado. Para isso, vamos nos Settings do nosso projeto, e navegar nas seguintes opções (pode ser que a interface mude no futuro, mas a ideia é essa):

* Settings --> Branches --> Add Branch ruleset
* Crie um nome para o ruleset, por exemplo: branch-main-protection
* Mude para `Enabled`
* Em `Target branches`, clique em Add target e selecione `Include default branch`
* Marque as opções:
  * Restrict deletions
  * Require a pull request before merging
  * Require status checks to pass
    * Adicione o check Jest Ubuntu
  * Block force pushes

## Criando o Workflow de Lint com o Prettier (estilização)

Aqui não tem muito segredo agora, vamos criar um novo workflow chamado `linting.yaml` assim:

```yaml title=".github/workflows/linting.yaml"
name: Linting

on: pull_request

jobs:
  prettier:
    name: Prettier
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "lts/hydrogen"

      - run: npm ci

      - run: npm run lint:prettier:check
```

E lá no Ruleset no GitHub, edite o que criamos acima, e em Require status check to pass, adicione o `Prettier`.

## Criando o Workflow de Lint com o ESLinter (qualidade)

### Instalando e configurando o ESLint

Vamos instalar o ESLinter no projeto, junto com alguns plugins para next, jest, e para evitar conflitos com o prettier:

```bash
npm install --save-dev eslint@8.57.0 eslint-config-next@14.2.4
npm i -D eslint-plugin-jest@28.6.0
npm i -D eslint-config-prettier@9.1.0
```

Ele vai criar um arquivo chamado `.eslintrc.json` na raíz do projeto. Vamos editá-lo para adicionar esses plugins, e adicionar também um conjunto de regras chamado `eslint:recommended`:
```json title=".eslintrc.json"
{
  "extends": [
    "eslint:recommended",
    "plugin:jest/recommended",
    "next/core-web-vitals",
    "prettier"
  ]
}
```

E agora vamos adicionar um atalho no nosso `package.json` para rodar o ESLint:
```json title="package.json" hl_lines="8"
  "scripts": {
    "dev": "npm run services:up && npm run wait-for-postgres && npm run migration:up && next dev",
    "services:up": "docker compose -f infra/compose.yaml up -d",
    "services:stop": "docker compose -f infra/compose.yaml stop",
    "services:down": "docker compose -f infra/compose.yaml down",
    "lint:prettier:check": "prettier --check .",
    "lint:prettier:fix": "prettier --write .",
    "lint:eslint:check": "next lint --dir .",
    "test": "npm run services:up && npm run wait-for-postgres && concurrently --names next,jest --hide next --kill-others --success command-jest \"next dev\" \"jest --runInBand --verbose\"",
    "test:watch": "jest --watchAll --runInBand --verbose",
    "migration:create": "node-pg-migrate -m infra/migrations create",
    "migration:up": "node-pg-migrate -m infra/migrations --envPath .env.development up",
    "wait-for-postgres": "node infra/wait-for-postgres.js"
  },
```

Agora se rodarmos o comando `npm run lint:eslint:check`, ele vai nos apontar alguns erros:

```bash
> meubonsai-app-v2@1.0.0 lint:eslint:check
> next lint --dir .


./infra/database.js
11:11  Error: 'error' is not defined.  no-undef
30:1  Warning: Assign object to a variable before exporting as module default  import/no-anonymous-default-export

./infra/migrations/1762550294138_test-migration.js
5:15  Error: 'pgm' is defined but never used.  no-unused-vars
7:17  Error: 'pgm' is defined but never used.  no-unused-vars

./pages/api/v1/migrations.js
43:11  Error: 'error' is not defined.  no-undef

./tests/orchestrator.js
21:1  Warning: Assign object to a variable before exporting as module default  import/no-anonymous-default-export

info  - Need to disable some ESLint rules? Learn more here: https://nextjs.org/docs/basic-features/eslint#disabling-rules
```

### Criando workflow no Github Actions

Para criar esse check no GitHub Actions, é a mesma coisa que fizemos nos outros workflows. Vamos editar o workflow de linting, e adicionar o job do eslint:

```yaml title=".github/workflows/linting.yaml" hl_lines="20-32"
name: Linting

on: pull_request

jobs:
  prettier:
    name: Prettier
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "lts/hydrogen"

      - run: npm ci

      - run: npm run lint:prettier:check

  eslint:
    name: Eslint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "lts/hydrogen"

      - run: npm ci

      - run: npm run lint:eslint:check
```

E agora basta adicionar o ESLint na lista de Checks do ruleset do Github, como fizemos com os demais.

!!! note

    Os erros levantados pelo ESLint foram resolvidas manualmente, seja ignorando o erro, ou corrigindo ele. Não entrarei em detalhes aqui no documento, mas o proprio VSCode sugere como corrigir, e o Copilot também é excelente nesses casos


!!! note

    Nos próximos documentos iremos avançar com a parte de Usuários e Autenticação do projeto. O commit final com as alterações até agora estão aqui:
    [Commit atual](https://github.com/brunononogaki/meubonsai-app-v2/commit/6d16ad2a12eb752c31bb0604d08c651747af2263)