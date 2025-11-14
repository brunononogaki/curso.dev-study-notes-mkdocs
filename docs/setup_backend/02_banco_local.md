# Integrando um Banco de Dados Local

Agora vamos subir um Banco de Dados Postgres para integrar com o nosso back-end. 

## Criando um .env
Vamos criar um .env para definir o acesso ao banco. O Next.JS já carrega os .env automaticamente nas variáveis de ambiente, que podem ser lidas através do **process.env.NOME_DA_VARIAVEL**. Como estamos no ambiente de desenvolvimento, vamos criar o arquivo chamado **.env.development**.

Quando for para produção na Vercel, esses valores serão definidos nas envs da própria Vercel

<div>
    <strong>.env.development</strong>
</div>

```bash
DATABASE_HOST=localhost
DATABASE_PORT=5432
POSTGRES_USER=dbadmin
POSTGRES_PASSWORD=mypassword
POSTGRES_DB=postgres
```

## Criando um docker-compose
Agora vamos criar um arquivo compose.yml para subir o nosso PostgreSQL com Docker Compose. Já vamos aproveitar e criar esse arquivo na pasta /infra/

<div>
    <strong>/infra/compose.yaml</strong>
</div>

```yaml
services:
  database:
    container_name: database
    image: postgres:16.0-alpine3.18
    env_file:
      - ../.env
    ports:
      - "5432:5432"
    restart: unless-stopped  
```

Note que como é um Database de Desenvolvimento, eu não preciso salvar o volume dele, porque toda vez que eu for testar, eu quero ter um banco de dados zerado!

## Subir o container do PostgreSQL
Para subir o container com docker compose:
```bash
docker compose --file infra/compose.yaml up -d --force-recreate
```

Para testar a conexão local, podemos usar o psql
```bash
psql -h localhost -p 5432 -U dbadmin -d postgres
```

## Integrando o Back com o Banco de Dados

Para integrar com o banco de dados, vamos usar o pg do npm
```bash
npm install pg@8.11.3
```

E criaremos um arquivo infra/database.js
<div>
    <strong>infra/database.js</strong>
</div>

```javascript
import {Client} from "pg";

async function query(queryObject) {
    const client = new Client({
        host: process.env.DATABASE_HOST,
        port: process.env.DATABASE_PORT,
        user: process.env.POSTGRES_USER,
        password: process.env.POSTGRES_PASSWORD,
        database: process.env.POSTGRES_DB,
        // Precisamos definir o ssl como True no ambiente de produção,
        // pois usaremos um serviço de DB online        
        ssl: process.env.NODE_ENV === 'production' ? true : false     
    });
    await client.connect();
    try {
        const result = await client.query(queryObject);
        return result;
    } catch (err) {
        console.log(err)
    } finally {
        await client.end();
    }
}

export default {
    query: query
}
```

Uma forma mais elegante de fazer é separar a conexão, para termos uma função que retorna uma instância conectada ao banco, que podemos utilizar mais pra frente em outros endpoints.
<div>
    <strong>infra/database.js (refatorada)</strong>
</div>

```javascript
import { Client } from "pg";

async function query(queryObject) {
  // precisamos definir client fora do try para que possa ser fechado no finally
  let client; 
    
  try {
    client = await getNewClient();
    const result = await client.query(queryObject);
    return result;
  } catch (err) {
    console.log(err);
    throw error;
  } finally {
    await client.end();
  }
}

async function getNewClient() {
  const client = new Client({
    host: process.env.DATABASE_HOST,
    port: process.env.DATABASE_PORT,
    user: process.env.POSTGRES_USER,
    password: process.env.POSTGRES_PASSWORD,
    database: process.env.POSTGRES_DB,
    // Precisamos definir o ssl como True no ambiente de produção,
    // pois usaremos um serviço de DB online
    ssl: process.env.NODE_ENV === "production" ? true : false,
  });
  await client.connect();
  return client;
}

// Exportando as funções para poderem ser usadas de fora
export default {
  query,
  getNewClient,  
};

```

## Usando a conexão ao Banco de Dados na tela de status
Agora vamos fazer a tela de status se conectar ao banco e rodar uma query genérica:

```javascript
import database from "infra/database.js";

export default async function status(request, response) {
  const result = await database.query("SELECT 1+1 AS sum;");
  response.status(200).json(result);
}
```

E confirme se a página continua retornando 200OK e se os testes estão passando.


## Criando scripts de inicalização
Vamos agora alterar o package.json e criar scripts que sobem a nossa infra toda:

<div>
    <strong>package.json</strong>
</div>

```javascript
  "scripts": {
    "dev": "npm run services:up && next dev",
    "services:up": "docker compose -f infra/compose.yaml up -d",
    "services:stop": "docker compose -f infra/compose.yaml stop",
    "services:down": "docker compose -f infra/compose.yaml down",
    "lint:check": "prettier --check .",
    "lint:fix": "prettier --write .",
    "test": "jest",
    "test:watch": "jest --watch-all"
  },
```

Show! Agora é possível subir todo o nosso ambiente (container do Postgres + aplicação) com o comando **npm run dev**! Coisa linda! 😎

## Retornando dados do banco
Agora vamos retornar dados do Banco, como a versão e a quantidade de conexão que ele suporta:

```javascript
import database from "infra/database.js";

async function get_postgres_version() {
  const result = await database.query("SHOW server_version");
  return result.rows[0].server_version;
}

async function get_postgres_max_connections() {
  const result = await database.query(
    "SHOW max_connections",
  );
  return parseInt(result.rows[0].max_connections);
}

async function get_postgres_used_connections() {
  // Sem proteção de SQL Injection:
  // `SELECT COUNT(*)::int FROM pg_stat_activity WHERE datname = '${process.env.POSTGRES_DB}';`
  const result = await database.query({
    text: "SELECT COUNT(*)::int FROM pg_stat_activity WHERE datname = $1;",
    values: [process.env.POSTGRES_DB]
  });
  return result.rows[0].count;
}

export default async function status(request, response) {
  const updatedAt = new Date().toISOString();
  response.status(200).json({
    updated_at: updatedAt,
    dependencies: {
      database : {
        version: await get_postgres_version(),
        max_connections: await get_postgres_max_connections(),
        opened_connections: await get_postgres_used_connections(),
      }
    }
  });
}

```

E vamos rodar os testes para confirmar se está tudo certo:
```javascript
test("GET to /api/v1/status should return 200", async () => {
  const response = await fetch("http://localhost:3000/api/v1/status");
  expect(response.status).toBe(200);
  const responseBody = await response.json();
  expect(responseBody.updated_at).toBeDefined();
  const parsedUpdatedAt = new Date(responseBody.updated_at).toISOString();
  expect(responseBody.updated_at).toEqual(parsedUpdatedAt)
  expect(responseBody.dependencies.database.version).toEqual('16.0')
  expect(Number.isInteger(responseBody.dependencies.database.max_connections)).toEqual(true)
  expect(responseBody.dependencies.database.opened_connections).toEqual(1)
});
```

## Proteção conta SQL Injection

Se eu implementar a query dessa forma:

```javascript
const result = await database.query(`SELECT COUNT(*)::int FROM pg_stat_activity WHERE datname = '${dbname}';`);
```

E chamar a API assim:
```javascript
fetch("http://localhost:3000/api/v1/status?dbname='; SELECT pg_sleep(5); --");
```

A query ficaria assim:
```sql
SELECT COUNT(*)::int FROM pg_stat_activity WHERE datname = ';
SELECT SELECT pg_sleep(5);
---
```

Ou seja, ele vai passar no parâmetro dbame uma outra query, que fecha as aspas simples do dbname, e depois faz um sleep, mas poderia ser qualquer outra coisa. Isso é ataque de SQL Injection.

Para resolver isso, basta fazer a query com parametros, assim:
```javascript
  const result = await database.query({
    text: "SELECT COUNT(*)::int FROM pg_stat_activity WHERE datname = $1;",
    values: [process.env.POSTGRES_DB]
  });
```

## Subindo um banco em produção

Para o Banco de Dados em produção, podemos usar um serviço como o neon.tech. Basta criar uma conta gratuita e iniciar um projeto novo. De lá, podemos pegar a Connection String, com os dados de conexão:

![image](static/neon_db.png)

E vamos editar os valores do nosso .env, mas dentro das variáveis de ambiente da Vercel:

<div>
    <strong>.env.develop (ambiente de desenvolvimento local)</strong>
</div>

```bash
DATABASE_HOST=localhost
DATABASE_PORT=5432
POSTGRES_USER=dbadmin
POSTGRES_PASSWORD=!lab@DD123
POSTGRES_DB=postgres
```

Configuração na Vercel:
![image](static/vercel_envs.png)

Dessa forma, quando rodarmos o código no ambiente de produção, o retorno da API Status será assim:
```json
{
  "updated_at": "2025-11-14T02:00:39.119Z",
  "dependencies": {
    "database": {
      "version": "17.5 (aa1f746)",
      "max_connections": 112,
      "opened_connections": 1
    }
  }
}
```