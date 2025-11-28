# Criando Boilerplate Django Ninja

Esse guia será usado para documentar a criação de uma API com Django Ninja. Não tem a ver exatamente com o curso.dev, mas tem, porque a ideia é ir seguindo mais ou menos os mesmos passos, mas numa linguagem que eu estou mais familiarizado (Python). Para esse deployment utilizarei:

- Poetry (versão 2.1.4)
- Taskipy
- Django + Django Ninja
- PostreSQL rodando com Docker

Repositório do Projeto: [GitHub - django-ninja-boilerplate](https://github.com/brunononogaki/django-ninja-boilerplate)

## Criando um ambiente de desenvolvimento com Poetry

- Primeiramente, vamos criar um projeto novo com o Poetry:

```bash
poetry new django-ninja-boilerplate
```

- Agora vamos instalar algumas dependências para começar:

```
cd django-ninja-boilerplate

poetry add django-ninja
poetry add django-extensions
poetry add python-decouple
poetry add psycopg
poetry add uvicorn
```

- Agora temos que editar o arquivo `pyproject.toml`:

Substitua esse bloco:

```toml
[tool.poetry]
packages = [{include = "django_ninja_boilerplate", from = "src"}]
```

Por esse:

```toml
[tool.poetry]
package-mode = false
```

E altere o requires-python adicionando um `,<4.0` (é necessário para instar o Taskipy mais pra frente):

```
requires-python = ">=3.13,<4.0"
```

- Para ficar mais fácil, vamos habilitar o `poetry shell`:

```bash
poetry shell
```

- Agora vamos criar um projeto Django:

```bash
django-admin startproject myapi .

Nota: Pode dar o nome que quiser, no meu caso, chamarei o projeto de "myapi"
Nota2: Se o comando poetry shell não funcionar, rode esses comandos como poetry run django-admin startproject myapi .
```

- Crie um arquivo `.env.development` no diretório raíz. Já vamos deixar criado umas coisas que usaremos mais pra frente.

```bash title=".env.development"
DATABASE_HOST=localhost
DATABASE_PORT=5432
POSTGRES_USER=devuser
POSTGRES_PASSWORD=devpassword
POSTGRES_DB=postgres

SECRET_KEY='mysecretkey-dev'
ALLOWED_HOSTS=localhost,127.0.0.1,.brunononogaki.com
```

## Instalando ferramentas de Dev

- Adicionando algumas dependências de `dev`:

Vamos instalar as seguintes ferramentas como dependência de Dev:

```bash
poetry add --dev pytest-django      # Framework de teste
poetry add --dev pytest-cov
poetry add --dev pytest-watch
poetry add --dev ruff               # Linter
poetry add --dev taskipy            # Atalho para comandos
poetry add --dev honcho             # Rodar rotinas em paralelo, como o concurrently, que se usa no curso.dev
```

- Configurando o Taskipy:

Utilizaremos o `Taskipy` para criar atalhos de comandos, igual a gente faz `npm run`, `npm test` no Node, mas vai ser `task run`, `task test` no Python.

Vamos adicionar as seguintes configurações `pyproject.toml` para o `Taskipy`. Já vamos colocar alguns comandos que a gente só vai usar mais pra frente, mas já fica no jeito para funcionar quando tivermos tudo pronto:

```toml title="pyproject.toml"
[tool.taskipy.tasks]
services-up = "docker compose -f infra/compose-dev.yaml up -d"
services-stop = "docker compose -f infra/compose-dev.yaml stop"
services-down = "docker compose -f infra/compose-dev.yaml down"
create-env-dev = "ln -sf .env.development .env"
create-env-prod = "ln -sf .env.production .env"
run = 'task create-env-dev && task services-up && python infra/wait-for-postgres.py && python manage.py migrate && python manage.py runserver'
down = "pkill -f 'manage.py runserver'; task services-down"
test = 'task create-env-dev && task services-up && python infra/wait-for-postgres.py && honcho start web test && task down'
test-watch = 'pytest-watch'
lint = 'ruff check'
format = 'ruff format'
```

- Configurando o Ruff:

Vamos adicionar as seguintes configurações no `pyproject.toml` para o `Ruff` (pode alterar como preferir):

```toml title="pyproject.toml"
[tool.ruff]
line-length = 119
extend-exclude = ['migrations', 'manage.py']

[tool.ruff.lint]
preview = true
select = ['I', 'F', 'E', 'W', 'PL', 'PT', 'FAST']

[tool.ruff.format]
preview = true
quote-style = 'single'
```

Como já temos os comandos do Taskipy configurados, podemos usar o `task format` para rodar o Ruff e corrigir o código, ou o `task lint` apenas para visualizar os erros existentes.

- Configurando o Pytest:

Vamos adicionar as seguintes configurações `pyproject.toml` para o `Pytest` (pode alterar como preferir):

```toml title="pyproject.toml"
[tool.pytest.ini_options]
pythonpath = "."
addopts = '-p no:warnings'
```

E criar um novo arquivo na raíz chamado `pytest.ini`:

```toml title="pytest.ini"
[pytest]
DJANGO_SETTINGS_MODULE = myapi.settings
python_files = tests.py test_*.py *_tests.py
addopts = -p no:warnings
```

Como já temos os comandos do Taskipy configurados, podemos usar o `task test` ou o `task test-watch` (modo real-time) para rodar os testes. Voltaremos a essas configurações mais pra frente.

## Subindo um Banco de Dados Local (Dev)

Para esse desenvolvimento, ao invés de usar o SQLite padrão do Django, vamos subir um banco Postgres local com Docker. E aí quando subirmos na produção, apenas mudamos os valores no .env para apontar para o Banco de Prod. Então vamos criar uma pasta chamada `infra`, e nela criar os arquivos de compose do `docker-compose`.

```yaml title="compose-dev.yaml"
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
```

Agora você pode subir e parar o banco com os comandos `task services-up`, `task services-stop` e `task services-down` que definimos nas configurações do Taskipy. Lembre-se que para os comandos funcionarem, você deve estar dentro do `poetry shell`. Caso contrário, você pode executar a mesma coisa, só que com os comandos `poetry run task services-up`, por exemplo.

## Subindo a aplicação

Para subir a aplicação Django, precisamos antes criar um arquivo `.env`, porque o Django não lê automaticamente o `.env.development`, `.env.production`, etc, assim como faz o Next. Uma das formas de fazer isso é criar um link simbólico apontando o `.env.development` -> `.env`. Assim:

```bash
ln -sf .env.development .env
```

No Windows (PowerShell) pode ser feito assim:

```powershell
New-Item -ItemType SymbolicLink -Path ".env" -Target ".env.development" -Force
```

Outra coisa que precisamos fazer é rodar os scripts de Migration pela primeira vez para configurar o banco:

```bash
python manage.py migrate
```

E finalmente subir o servidor web:

```bash
python manage.py runserver
```

E pronto, com isso a nossa a aplicação já estaria executando na porta 8000:

```bash
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
November 27, 2025 - 20:34:28
Django version 5.2.8, using settings 'myapi.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.

WARNING: This is a development server. Do not use it in a production setting. Use a production WSGI or ASGI server instead.
For more information on production servers see: https://docs.djangoproject.com/en/5.2/howto/deployment/
```

Podemos colocar tudo isso dentro do comando `task run` para ficar mais fácil, e já aproveitar e nele rodar o `task services-up` para subir o banco. Mas, assim como vimos no curso.dev, seria legal subir a Web somente depois de o banco estar disponível. Para isso, vamos criar um script chamado `wait-for-postgres.py` e colocá-lo na pasta `infra`:

```python title="wait-for-postgres.py"
import subprocess
import sys
import time


def check_postgres():
    result = subprocess.run(
        ['docker', 'exec', 'postgres-dev', 'pg_isready', '--host', 'localhost'],
        check=False,
        capture_output=True,
        text=True,
    )
    if 'accepting connections' not in result.stdout:
        sys.stdout.write('.')
        sys.stdout.flush()
        time.sleep(1)
        check_postgres()
    else:
        print('\n🟢 Postgres is ready!')


if __name__ == '__main__':
    print('\n\n🔴 Waiting for Postgres to accept connections...')
    check_postgres()
```

E agora, voltando ao que já temos configurado no Taskipy:

```toml
[tool.taskipy.tasks]
services-up = "docker compose -f infra/compose-dev.yaml up -d"
services-stop = "docker compose -f infra/compose-dev.yaml stop"
services-down = "docker compose -f infra/compose-dev.yaml down"
create-env-dev = "ln -sf .env.development .env"
create-env-prod = "ln -sf .env.production .env"
run = 'task create-env-dev && task services-up && python infra/wait-for-postgres.py && python manage.py migrate && python manage.py runserver'
down = "pkill -f 'manage.py runserver'; task services-down"
test = 'task create-env-dev && task services-up && python infra/wait-for-postgres.py && honcho start web test && task down'
test-watch = 'pytest-watch'
lint = 'ruff check'
format = 'ruff format'
```

Veja que o `task run` vai fazer o seguinte:

- Rodar o `create-env-dev` para criar o link simbólico do `.env.development` para `.env` (se for Windows, você vai precisar alterar o `create-env-dev`)
- Rodar o `task services-up` para subir o Banco Postgres local
- Rodar o script `python infra/wait-for-postgres.py` para esperar o Banco ficar disponível
- Rodar a Migration com o `python manage.py migrate`
- Iniciar o servidor com o `python manage.py runserver`

O `task down`, por sua vez, vai matar o processo do Web Server, e matar o container do Postgres.

## Executando os testes

Já temos o Pytest configurados e os comandos de `task test` e `task test-watch` no Taskipy. Mas tem uma coisa que precisamos fazer antes de rodá-los. Veja que para rodar o task test, teríamos que ter o servidor rodando antes, certo? Porque o teste vai literalmente enviar requisições ao nosso web server, como se fosse um client. Então, antes de rodar o task test, vamos subir o banco, aguardar ele ficar disponível, rodar a migração, subir o Web Server e rodar os testes. Que tal?

Só que, assim como vimos no curso.dev, tem uma complexidade aí. O banco com o Docker roda em modo `detached` com o parâmetros -d do docker-compose, então tranquilo, porque ele vai subir e não vai travar o terminar. Mas o Django não tem essa função. No curso.dev utilizamos uma ferramenta do NPM chamada `concurrently`. A gente aqui no Python vai usar uma chamada `Honchu`.

Já instalamos o Honchu como dependência de dev mais pra cima, então só falta configurá-lo. Para isso, vamos criar na raiz do projeto um arquivo chamado `Procfile`. E nesse arquivo, vamos definir dois processos: um chamado web, que vai subir o Django e esconder os logs dele no terminal; e um chamado test, que vai rodar o Pytest:

```title="Procfile"
web: python manage.py runserver 0.0.0.0:8000 > /dev/null 2>&1
test: pytest -vv
```

Agora vamos entender o comando que colocamos no Taskipy

```toml
test = 'task create-env-dev && task services-up && python infra/wait-for-postgres.py && honcho start web test && task down'
```

Esse comando vai criar o link simbólico do .env, subir o banco Postgres, esperar ele subir, aí em paralelo, usando o Honchu, ele vai iniciar o web server, e rodar os testes. E depois vai matar o container do banco.

## pyproject.toml até agora

Então até agora, o nosso `pyproject.toml` tá assim:

```toml title="pyproject.toml"
[project]
name = "django-ninja-boilerplate"
version = "0.1.0"
description = ""
authors = [
    {name = "Bruno Nonogaki",email = "brunono@gmail.com"}
]
readme = "README.md"
requires-python = ">=3.13,<4.0"
dependencies = [
    "django-ninja (>=1.5.0,<2.0.0)",
    "django-extensions (>=4.1,<5.0)",
    "python-decouple (>=3.8,<4.0)",
    "psycopg (>=3.2.13,<4.0.0)",
    "uvicorn (>=0.38.0,<0.39.0)"
]

[tool.poetry]
package-mode = false


[tool.poetry.group.dev.dependencies]
pytest-django = "^4.11.1"
pytest-cov = "^7.0.0"
ruff = "^0.12.11"
taskipy = "^1.14.1"
pytest-watch = "^4.2.0"
honcho = "^2.0.0"

[build-system]
requires = ["poetry-core>=2.0.0,<3.0.0"]
build-backend = "poetry.core.masonry.api"

[tool.ruff]
line-length = 119
extend-exclude = ['migrations', 'manage.py']

[tool.ruff.lint]
preview = true
select = ['I', 'F', 'E', 'W', 'PL', 'PT', 'FAST']

[tool.ruff.format]
preview = true
quote-style = 'single'

[tool.pytest.ini_options]
pythonpath = "."
addopts = '-p no:warnings'

[tool.taskipy.tasks]
services-up = "docker compose -f infra/compose-dev.yaml up -d"
services-stop = "docker compose -f infra/compose-dev.yaml stop"
services-down = "docker compose -f infra/compose-dev.yaml down"
create-env-dev = "ln -sf .env.development .env"
create-env-prod = "ln -sf .env.production .env"
run = 'task create-env-dev && task services-up && python infra/wait-for-postgres.py && python manage.py migrate && python manage.py runserver'
down = "pkill -f 'manage.py runserver'; task services-down"
test = 'task create-env-dev && task services-up && python infra/wait-for-postgres.py && honcho start web test && task down'
test-watch = 'pytest-watch'
lint = 'ruff check'
format = 'ruff format'
```

Boa! Agora a seguir a gente vai configurar o Django e criar nossa primeira aplicação! 😎
