# Linting de Commits

Vamos adotar algumas regras nos textos de commit daqui pra frente:
* Textos em inglês
* Verbos no imperativo
* Formato padrão (conventionalcommits.org):
  * \<type\>[optional scope]: description, onde type pode ser um desses:
    * fix
    * feat
    * build
    * chore
    * ci
    * docs
    * style
    * refactor
    * perf
    * revert
    * test
* Casos como uma feature nova desenvolvida junto com os testes, entrariam em um commit do tipo `feat`. Mas se futuramente descobrirmos que faltou algo nos testes e fizermos um novo commit, aí sim seria do tipo `test`.

Vamos usar algumas funcionalidades no projeto para nos ajudar no Linting dos commits, e na criação deles:
* [commitlint](https://commitlint.js.org)

## Commitlint

### Instalando o Commitlint localmente

```bash
npm i -D @commitlint/cli@19.3.0
npm i -D @commitlint/config-conventional@19.2.2
```

### Configurando o Commitlint com o Config Conventional
Crie um arquivo na raíz do projeto chamado `commitlint.config.js`:

```javascript title="/commitlint.config.js"
module.exports = {
  extends: ["@commitlint/config-conventional"]
}
```

### Testando o Commitlint

Podemos rodar o commitlint localmente com o `npx` para ele validar uma mensagem de commit:
```bash
echo "feat: teste de mensagem" | npx commitlint
```  

## Integrando no CI

No site do CommitLint já tem os scripts que você precisa adicionar para adicionar o Commitlint no CI. No nosso caso, vamos usar o do GitHub Actions:

```yaml
- name: Validate PR commits with commitlint
        if: github.event_name == 'pull_request'
        run: npx commitlint --from ${{ github.event.pull_request.base.sha }} --to ${{ github.event.pull_request.head.sha }} --verbose
```

O que esse comando faz é pegar todos os commits dentro de um `pull request`, e roda o commitlint um a um. 

Agora é basicamente colocar isso no nosso scrit de linting do CI. O problema é que a action `checkout@v4` que usamos no Worklow para baixar o código, por padrão só baixa o último commit, então não daria para validar todos os commits do PR. Então a gente vai ter que adicionar uma configuração `fetch-depth: 0` para baixarmos todo o histórico de commits.

Vai ficar assim:
```yaml title=".github/workflows/linting.yaml" hl_lines="6-7 15-18"
  commitlint:
    name: Commitlint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 

      - uses: actions/setup-node@v4
        with:
          node-version: "lts/hydrogen"

      - run: npm ci

      - name: Validate PR commits with commitlint
        if: github.event_name == 'pull_request'
        run: npx commitlint --from ${{ github.event.pull_request.base.sha }} --to ${{ github.event.pull_request.head.sha }} --verbose
```

!!! tip

    E aí, é só não esquecer de configurar a `Ruleset` no GitHub para ele não continuar o Merge caso falhe nessa verificação.

## Configurando um Hook  

Hook é um mecanismo do git para executar alguma coisa quando alguma ação do Git é disparada. No nosso caso, vamos usar o hoot `commit-msg` para rodar o commitlint no nosso ambiente local logo depois de inserirmos uma mensagem de commit, e já validarmos localmente antes de subirmos isso para o repositório remoto.

Para listar alguns exemplos de hooks, você pode entrar na pasta .git/hooks do seu projeto, e ver os arquivos que tem lá dentro:

```bash
-rwxr-xr-x  1 bruno.nonogaki  staff   478B Oct 30 16:51 applypatch-msg.sample
-rwxr-xr-x  1 bruno.nonogaki  staff   896B Oct 30 16:51 commit-msg.sample
-rwxr-xr-x  1 bruno.nonogaki  staff   4.6K Oct 30 16:51 fsmonitor-watchman.sample
-rwxr-xr-x  1 bruno.nonogaki  staff   189B Oct 30 16:51 post-update.sample
-rwxr-xr-x  1 bruno.nonogaki  staff   424B Oct 30 16:51 pre-applypatch.sample
-rwxr-xr-x  1 bruno.nonogaki  staff   1.6K Oct 30 16:51 pre-commit.sample
-rwxr-xr-x  1 bruno.nonogaki  staff   416B Oct 30 16:51 pre-merge-commit.sample
-rwxr-xr-x  1 bruno.nonogaki  staff   1.3K Oct 30 16:51 pre-push.sample
-rwxr-xr-x  1 bruno.nonogaki  staff   4.8K Oct 30 16:51 pre-rebase.sample
-rwxr-xr-x  1 bruno.nonogaki  staff   544B Oct 30 16:51 pre-receive.sample
-rwxr-xr-x  1 bruno.nonogaki  staff   1.5K Oct 30 16:51 prepare-commit-msg.sample
-rwxr-xr-x  1 bruno.nonogaki  staff   2.7K Oct 30 16:51 push-to-checkout.sample
-rwxr-xr-x  1 bruno.nonogaki  staff   3.6K Oct 30 16:51 update.sample
```

Então se renomearmos algum desses hooks removendo o `.sample` do nome, o Hook estará ativo. E ele nada mais é do que um script Shell. Mas por questões de segurança, nada do que você faça aqui vai ser adicionado no repositório! Então não é possível criar um Hook aqui e compartilhar com o time. Uma opção seria criar uma pasta hooks no projeto, e colocar esses scripts lá dentro, e criar um outro script para copiar esses arquivos para a pasta .git/hooks. Mas para facilitar a vida, foi criado um projeto opensource chamado `Husky`, que resolve esse problema.

### Configurando o Husky

Primeiramente vamos instalar a dependência de desenvolvimento:
```bash
npm i -D husky@9.1.4
```

E depois vamos rodar:
```bash
npx husky init
```

Esse script vai adicionar o comando `prepare` no nosso packages.json. Esse comando `prepare` é padrão do NPM e é uma das fases do ciclo de vida do `npm install`. Então depois de rodar o `npm install`, o NPM automaticamente roda o script de `prepare`, que vai rodar o binário do Husky.

Veja que agora o arquivo `.git/config` agora aponta o **hooksPath** para a pasta `.husky/_`:

```bash title=".git/config" hl_lines="8"
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
        ignorecase = true
        precomposeunicode = true
        hooksPath = .husky/_
[remote "origin"]
        url = https://github.com/brunononogaki/meubonsai-app-v2.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[remote "origin/main"]
        url = https://github.com/brunononogaki/meubonsai-app-v2.git
        fetch = +refs/heads/*:refs/remotes/origin/main/*
[branch "main"]
        remote = origin
        merge = refs/heads/main
[pull]
        rebase = true
[branch "hooks"]
        vscode-merge-base = origin/main
```

Iremos criar os nossos hooks na pasta .husky, e não dentro da sub-pasta _. Por padrão, já tem um arquivo chamado `pre-commit` lá dentro, que podemos remover. E vamos criar um novo arquivo chamado `commit-msg`.

```shell title=".husky/commit-msg"
npx commitlint --edit $1
```

O que esse hook vai fazer é que imediatamente após inserirmos a mensagem de commit, invocaremos o commitlint, passando a mensagem como parâmetro.

Agora se fizermos um commit com uma mensagem fora do padrão, o commitlint já vai barrar:
```bash
git commit -m "teste"                                                               ✔  10:14:54  
⧗   input: teste
✖   subject may not be empty [subject-empty]
✖   type may not be empty [type-empty]

✖   found 2 problems, 0 warnings
ⓘ   Get help: https://github.com/conventional-changelog/commitlint/#what-is-commitlint

husky - commit-msg script failed (code 1)
```

!!! tip

    Se estiver em uma situação de emergência, com pressa, e quiser subir um commit sem que seja verificado, basta fazer o commit com a opção `--no-verify` ou `-n`, que ele vai pular os hooks
  

## Configurando o Commitizen

Uma ferramenta que pode também nos auxiliar nos commits é o `Commitizen`, que abre uma espécie de formulário quando fazemos um commit, e já nos dá as opções válidas de types.
Vamos instalá-lo como uma dependência de desenvolvimento.

```bash
npm i -D commitizen@4.3.0
```

E para configuá-lo localmente, a documentação nos instrui a dar esse comando
```bash
npx commitizen init cz-conventional-changelog --save-dev --save-exact
```

Agora vamos criar um novo scritp no package.json para o comando `commit`:
```json title="package.json" hl_lines="16"
  "scripts": {
    "dev": "npm run services:up && npm run services:wait:database && npm run migrations:up && next dev",
    "test": "npm run services:up && npm run services:wait:database && concurrently --names next,jest --hide next --kill-others --success command-jest \"next dev\" \"jest --runInBand --verbose\"",
    "posttest": "npm run services:stop",
    "test:watch": "jest --watchAll --runInBand --verbose",
    "services:up": "docker compose -f infra/compose.yaml up -d",
    "services:down": "docker compose -f infra/compose.yaml down",
    "services:stop": "docker compose -f infra/compose.yaml stop",
    "services:wait:database": "node infra/wait-for-postgres.js",
    "migrations:create": "node-pg-migrate -m infra/migrations create",
    "migrations:up": "node-pg-migrate -m infra/migrations --envPath .env.development up",
    "lint:prettier:check": "prettier --check .",
    "lint:prettier:fix": "prettier --write .",
    "lint:eslint:check": "next lint --dir .",
    "prepare": "husky",
    "commit": "cz"
  },
```

Agora, quando rodarmos o comando `npm run commit`, ele vai chamar o `cz`:
```bash
meubonsai-app-v2@1.0.0 commit
> cz

cz-cli@4.3.0, cz-conventional-changelog@3.3.0

? Select the type of change that you're committing: (Use arrow keys)
❯ feat:     A new feature 
  fix:      A bug fix 
  docs:     Documentation only changes 
  style:    Changes that do not affect the meaning of the code (white-space, formatting, missing semi-colons, etc) 
  refactor: A code change that neither fixes a bug nor adds a feature 
  perf:     A code change that improves performance 
  test:     Adding missing tests or correcting existing tests 

? What is the scope of this change (e.g. component or file name): (press enter to skip) 
? Write a short, imperative tense description of the change (max 96 chars):
 (35) add commitzen and commit npm script
? Provide a longer description of the change: (press enter to skip)
 
? Are there any breaking changes? No
? Does this change affect any open issues? No
[hooks d897575] ci: add commitzen and commit npm script
 3 files changed, 909 insertions(+), 5 deletions(-)
```

!!! success

    Agora nossos commits daqui para frente estarão todos padronizados!

    Até aqui, temos o Next com uma API para `/status` e `/migrations`, banco de dados integrado, testes automatizados e CI/CD configurado.

    Ou seja, a fundação está montada, e daqui para frente iniciaremos o deploy da aplicação. O commit final com essa fundação montada está aqui:
    
    [Commit Final - Fundação](https://github.com/brunononogaki/meubonsai-app-v2/commit/d6fe95f5d7ce22d7d5c32e537642afe9f5e8c640)