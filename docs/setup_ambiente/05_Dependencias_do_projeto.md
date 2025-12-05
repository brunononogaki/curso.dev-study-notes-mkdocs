# Trabalhando com as dependências do projeto

Para não causar problemas de versões das dependências do projeto, vamos primeiramente editar o arquivo `package.json`, e tirar o `^` na frente das versões, forçando o npm a usar as exatas versões que declaramos:

```json title="/package.json"
...
  "dependencies": {
    "async-retry": "1.3.3",
    "dotenv": "16.4.4",
    "dotenv-expand": "12.0.3",
    "eslint-plugin-react": "7.37.5",
    "next": "13.1.6",
    "node-pg-migrate": "6.2.2",
    "pg": "8.11.3",
    "react": "18.2.0",
    "react-dom": "18.2.0"
  },
  "devDependencies": {
    "concurrently": "8.2.2",
    "eslint": "8.57.0",
    "eslint-config-next": "14.2.4",
    "eslint-config-prettier": "9.1.0",
    "eslint-plugin-jest": "28.6.0",
    "jest": "29.6.2",
    "prettier": "3.6.2"
  }
```

Mas agora, vamos começar a ver as versões mais atualizadas de cada dependência e começar a atualizá-las. Dois comandos bons de verificar isso são o `npm outdated` e o `npm audit`.
```bash title="npm outdated"
npm outdated

Package                 Current  Wanted   Latest  Location                             Depended by
concurrently              8.2.2   8.2.2    9.2.1  node_modules/concurrently            @DEV/meubonsai-app-v2
dotenv                   16.4.4  16.4.4   17.2.3  node_modules/dotenv                  @DEV/meubonsai-app-v2
eslint                   8.57.0  8.57.0   9.39.1  node_modules/eslint                  @DEV/meubonsai-app-v2
eslint-config-next       14.2.4  14.2.4   16.0.7  node_modules/eslint-config-next      @DEV/meubonsai-app-v2
eslint-config-prettier    9.1.0   9.1.0   10.1.8  node_modules/eslint-config-prettier  @DEV/meubonsai-app-v2
eslint-plugin-jest       28.6.0  28.6.0  28.14.0  node_modules/eslint-plugin-jest      @DEV/meubonsai-app-v2
jest                     29.6.2  29.6.2   30.2.0  node_modules/jest                    @DEV/meubonsai-app-v2
next                     13.1.6  13.1.6   15.5.7  node_modules/next                    @DEV/meubonsai-app-v2
node-pg-migrate           6.2.2   6.2.2    7.9.1  node_modules/node-pg-migrate         @DEV/meubonsai-app-v2
react                    18.2.0  18.2.0   19.2.1  node_modules/react                   @DEV/meubonsai-app-v2
react-dom                18.2.0  18.2.0   19.2.1  node_modules/react-dom               @DEV/meubonsai-app-v2
```

```bash title="npm audit"
npm audit

glob  10.2.0 - 10.4.5
Severity: high
glob CLI: Command injection via -c/--cmd executes matches with shell:true - https://github.com/advisories/GHSA-5j98-mcp5-4vw2
fix available via `npm audit fix --force`
Will install eslint-config-next@16.0.7, which is a breaking change
node_modules/@next/eslint-plugin-next/node_modules/glob
  @next/eslint-plugin-next  14.0.5-canary.0 - 15.0.0-rc.1
  Depends on vulnerable versions of glob
  node_modules/@next/eslint-plugin-next
    eslint-config-next  14.0.5-canary.0 - 15.0.0-rc.1
    Depends on vulnerable versions of @next/eslint-plugin-next
    node_modules/eslint-config-next

js-yaml  <3.14.2
Severity: moderate
js-yaml has prototype pollution in merge (<<) - https://github.com/advisories/GHSA-mh29-5h37-fv8m
fix available via `npm audit fix`
node_modules/js-yaml

next  0.9.9 - 14.2.31
Severity: critical
Next.js missing cache-control header may lead to CDN caching empty reply - https://github.com/advisories/GHSA-c59h-r6p8-q9wc
Denial of Service condition in Next.js image optimization - https://github.com/advisories/GHSA-g77x-44xx-532m
Next.js Allows a Denial of Service (DoS) with Server Actions - https://github.com/advisories/GHSA-7m27-7ghc-44w9
Information exposure in Next.js dev server due to lack of origin verification - https://github.com/advisories/GHSA-3h52-269p-cp9r
Next.js Affected by Cache Key Confusion for Image Optimization API Routes - https://github.com/advisories/GHSA-g5qg-72qw-gw5v
Next.js authorization bypass vulnerability - https://github.com/advisories/GHSA-7gfc-8cq8-jh5f
Next.js Improper Middleware Redirect Handling Leads to SSRF - https://github.com/advisories/GHSA-4342-x723-ch2f
Next.js Content Injection Vulnerability for Image Optimization - https://github.com/advisories/GHSA-xv57-4mr9-wg8v
Next.js Race Condition to Cache Poisoning - https://github.com/advisories/GHSA-qpjv-v59x-3qc4
Authorization Bypass in Next.js Middleware - https://github.com/advisories/GHSA-f82v-jwr5-mffw
Depends on vulnerable versions of postcss
fix available via `npm audit fix --force`
Will install next@15.5.7, which is a breaking change
node_modules/next

postcss  <8.4.31
Severity: moderate
PostCSS line return parsing error - https://github.com/advisories/GHSA-7fh5-64p2-3v2j
fix available via `npm audit fix --force`
Will install next@15.5.7, which is a breaking change
node_modules/postcss

6 vulnerabilities (2 moderate, 3 high, 1 critical)

To address issues that do not require attention, run:
  npm audit fix

To address all issues (including breaking changes), run:
  npm audit fix --force
```

Para começar a atualizar as dependêcias, vamos usar o comando `npx npm-check-updated -i`. Esse comando vai instalar o npm-check-updated, e dar uma interface interativa para irmos atualizando os módulos sugeridos um a um.

```bash title="npx npm-check-updated -i"
npx npm-check-updates -i
Upgrading /Users/bruno.nonogaki/Documents/@DEV/meubonsai-app-v2/package.json
[====================] 16/16 100%

? Choose which packages to update › 
  ↑/↓: Select a package
  Space: Toggle selection
  a: Toggle all
  Enter: Upgrade
 
❯ ◉ concurrently             8.2.2  →   9.2.1
  ◉ dotenv                  16.4.4  →  17.2.3
  ◉ eslint                  8.57.0  →  9.39.1
  ◉ eslint-config-next      14.2.4  →  16.0.7
  ◉ eslint-config-prettier   9.1.0  →  10.1.8
  ◉ eslint-plugin-jest      28.6.0  →  29.2.1
  ◉ jest                    29.6.2  →  30.2.0
  ◉ next                    13.1.6  →  16.0.7
  ◉ node-pg-migrate          6.2.2  →   8.0.3
  ◉ react                   18.2.0  →  19.2.1
  ◉ react-dom               18.2.0  →  19.2.1
```



!!! tip

    A forma mais fácil de resolver `peer dependencies` é remover o arquivo `package-lock.json` e a pasta `node_modules`, e rodar o `npm install` de novo.

