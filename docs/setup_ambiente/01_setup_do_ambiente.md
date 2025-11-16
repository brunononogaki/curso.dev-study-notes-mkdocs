# Iniciando o setup do ambiente

Vamos começar o setup do NextJS no nosso ambiente local

```bash
nvm ls

# Instalando a versão LTS Hydrogen
nvm install lts/hydrogen

# Definindo essa versão com o a default
nvm alias default lts/hydrogen
```

Para definir essa versão padrão dentro do projeto:
<div><strong>.nvmrc</strong></div>

```bash
lts/hydrogen
```

## Criando o projeto e instalando dependências
Para criar o projeto, faremos:
```bash
npm init
# Defina um nome de projeto, author, description, ou deixe tudo default

npm install next@13.1.6
npm install react@18.2.0
npm install react-dom@18.2.0
```

Isso vai criar um arquivo ``package.json`` assim:
```javascript
{
  "name": "teste_next",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "description": "",
  "dependencies": {
    "next": "^13.1.6",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  }
}
```

Vamos adicionar o nosso primeiro script e apagar esse de "test" que ele criou automaticamente
```javascript
"scripts": {
  "dev": "next dev",
}

```

Agora vamos criar um arquivo ``index.js`` em uma nova pasta chamada /pages/:
<div><strong>/pages/index.js</strong></div>

```javascript
function Home() {
    return <h1>Teste</h1>
}
export default Home
```

Agora vamos iniciar o servidor
```bash
npm run dev
> teste_next@1.0.0 dev
> next dev

ready - started server on 0.0.0.0:3000, url: http://localhost:3000
event - compiled client and server successfully in 589 ms (149 modules)
```

Certo! Nosso servidor já está no ar! 😎


## Definindo estrutura de pastas

Agora vamos adotar uma estrutura de pastas para refletir o padrão de projetos MVC (Model / View / Controller)

```bash
.
├── infra
│   ├── migrations
│   └── provisioning
│       ├── production
│       └── staging
├── models
├── package-lock.json
├── package.json
├── pages
│   └── index.js
└── tests
```

Aos poucos vamos populando essas pastas e acertando a estrutura.
