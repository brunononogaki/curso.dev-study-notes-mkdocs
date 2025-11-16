# Configurando o node-pg-migrate

O Node possui vários frameworks de migrations, e um dos mais famosos é o Sequelize. Mas no projeto, usarmeos o node-pg-migrate por questões de simplicidade.

Primeiramente, precisamos instalar essas dependências

```bash
npm install node-pg-migrate@6.2.2
npm install dotenv@16.4.4
```

E vamos adicionar novos scripts no ``package.json``

<div><strong>package.json</strong></div>

```javascript
  "scripts": {
     ...
    "migration:create": "node-pg-migrate -m infra/migrations create",
    "migration:up": "node-pg-migrate -m infra/migrations --envPath .env.development up"
  },
```

## Criando a primeira migração

Para rodar a migração, primeiro precisamos rodar um "create":

```bash
npm run migration:create first migration
> meu-projeto@1.0.0 migration:create
> node-pg-migrate create first migration test
Created migration -- .../meu-projeto/migrations/1762387227077_first-migration-test.js
```

Como não criamos nada ainda, esse arquivo de migration não vai ter nada em "up" ou "down":

```javascript
/* eslint-disable camelcase */
exports.shorthands = undefined;
exports.up = (pgm) => {};
exports.down = (pgm) => {};
```

## Definindo uma connection string no .env

Para a migration funcionar, precisamos definir uma variável de ambiente **DATABASE_URL** contendo a connection string para o nosso banco de dados:

<div><strong>.env.development</strong></div>

```bash
DATABASE_URL=postgres://devuser:devpassword@localhost:5432/postgres
```

Mas como esses valores de usuário, senha, host, porta e nome da database já estão definidos antes no mesmo .env, podemos reutilizar esses valores. Mas para isso, precisamos instalar um módulo chamado **dotenv-expand**.

```bash
npm install dotenv-expand
```

Agora vamos deixar o nosso **.env.development** assim:

<div><strong>.env.development</strong></div>

```bash
DATABASE_HOST=localhost
DATABASE_PORT=5432
POSTGRES_USER=devuser
POSTGRES_PASSWORD=devpassword
POSTGRES_DB=postgres
DATABASE_URL=postgres://$POSTGRES_USER:$POSTGRES_PASSWORD@$DATABASE_HOST:$DATABASE_PORT/$POSTGRES_DB
```

## Executando a migração

Agora estamos prontos para rodar a migration:

```bash
npm run migration:up
```

Como ainda não temos nada para subir no banco, não vai acontecer nada, mas ao rodar isso pela primeira vez, será criada uma tabela no banco chamada **pgmigrations**, que armazenará o versionamento das migrações daqui pra frente.
