# Implementando Autenticação e Autorização

## Autenticação JWT

Vamos implantar a autenticação por meio de tokens JWT. Para isso, vamos instalar o `PyJWT`

```bash
poetry add PyJWT
```

Agora vamos criar um arquivo em `myapi/core/auth.py`, para colocar as funções de criação do token e de autenticação, ambas usando a lib `jwt`

```python title="myapi/core/auth.py"
# myapi/core/auth.py
import jwt
from datetime import datetime, timedelta
from django.conf import settings
from django.contrib.auth import get_user_model
from ninja.security import HttpBearer

User = get_user_model()
ALGO = 'HS256'
ACCESS_LIFETIME = timedelta(minutes=15)
REFRESH_LIFETIME = timedelta(days=7)


def create_token(user):
    now = datetime.utcnow()
    access_token = jwt.encode(
        {'user_id': str(user.id), 'exp': now + ACCESS_LIFETIME, 'type': 'access'},
        settings.SECRET_KEY,
        algorithm=ALGO,
    )
    refresh_token = jwt.encode(
        {'user_id': str(user.id), 'exp': now + REFRESH_LIFETIME, 'type': 'refresh'},
        settings.SECRET_KEY,
        algorithm=ALGO,
    )
    return {'access_token': access_token, 'refresh_token': refresh_token}


class JWTAuth(HttpBearer):
    def authenticate(self, request, token):
        try:
            payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[ALGO])
            if payload.get('type') != 'access':
                return None
            user_id = payload.get('user_id')
            return User.objects.get(id=user_id)
        except jwt.ExpiredSignatureError:
            return None
        except Exception:
            return None
```

Agora precisamos criar a rota de geração de Token. Mas antes, vamos criar dois schemas para o retorno da rota de gerar token:

```python title="myapi/core/schemas.py"
class TokenResponse(Schema):
    access_token: str
    refresh_token: str | None = None
    token_type: str = 'bearer'

class ErrorSchema(Schema):
    detail: str

```

Agora sim vamos criar a rota de `login`:

```python title="myapi/core/api.py" hl_lines="10-11"
from django.contrib.auth import get_user_model, authenticate
from .auth import create_token, JWTAuth

from .schemas import (
    StatusSchema,
    UserSchema,
    UserWithGroupsSchema,
    UserCreateSchema,
    UserPatchSchema,
    TokenResponse,  #<= Adicione isso
    ErrorSchema,    #<= Adicione isso
)

@router.post('login', tags=['Auth'], response={200: TokenResponse, 401: ErrorSchema})
def login(request, username: str = Form(...), password: str = Form(...)):
    user = authenticate(username=username, password=password)
    if not user:
        return 401, {'detail': 'Invalid credentials'}
    tokens = create_token(user)
    return 200, {'access_token': tokens.get('access_token') or tokens.get('access'), 'token_type': 'bearer', **tokens}
```

E agora para proteger uma rota é só colocar um `auth=JWTAuth()`:

```python title="myapi/core/api.py" hl_lines="6"
@router.get(
    'users',
    response=list[UserWithGroupsSchema],
    summary='List users',
    description='List users',
    auth=JWTAuth() # <= Adicione isso
)
@paginate
def list_users(request):
    return User.objects.all()
```

## Testando o endpoint de Login

Agora vamos criar os testes para o endpoint de login:

```python title="myapi/core/tests/test_auth.py"
import json
from http import HTTPStatus

import pytest
from django.contrib.auth import get_user_model
from django.urls import reverse
from decouple import config


User = get_user_model()

@pytest.mark.django_db
def test_login_success(client):
    response = client.post(
        '/api/v1/login',
        data={'username': config('DJANGO_ADMIN_USER'), 'password': config('DJANGO_ADMIN_PASSWORD')},
    )
    data = response.json()
    assert response.status_code == HTTPStatus.OK
    assert 'access_token' in data
    assert data['token_type'] == 'bearer'
    assert 'refresh_token' in data

@pytest.mark.django_db
def test_login_invalid_credentials(client):
    response = client.post(
        '/api/v1/login',
        data={'username': config('DJANGO_ADMIN_USER'), 'password': 'wrongpassword'},
    )
    data = response.json()
    assert response.status_code == HTTPStatus.UNAUTHORIZED
    assert data['detail'] == 'Invalid credentials'

@pytest.mark.django_db
def test_login_missing_fields(client):
    response = client.post(
        '/api/v1/login',
        data={},
    )
    assert response.status_code == HTTPStatus.UNPROCESSABLE_ENTITY or response.status_code == 422
```

Certo, mas esse teste é basicamente para ver se estamos conseguindo gerar o Token. Mas veja que ao proteger as rotas de CRUD de Users, quebramos os testes, porque agora precisamos passar uma autenticação. Vamos corrigir isso criando uma fixture de geração de Token antes de rodar os testes que já tinhamos criado.

```python title="myapi/core/tests/test_users.py"
@pytest.fixture
def create_admin_access_token(client):
    response = client.post(
        '/api/v1/login',
        data={'username': config('DJANGO_ADMIN_USER'), 'password': config('DJANGO_ADMIN_PASSWORD')},
    )
    response_json = response.json()
    access_token = response_json['access_token']
    return access_token
```

E agora basta chamar essa fixture nos testes que precisam de autenticação, e enviar o header com o Token. Mas veja que o TestClient do Django não aceita passarmos um header={} como no requests, precisamos especificar os cabeçalhos como argumentos nomeados com o prefixo `HTTP_`, por exemplo `HTTP_AUTHORIZATION`:

```python title="myapi/core/tests/test_users.py"
@pytest.mark.django_db
def test_list_users(client, create_admin_access_token):
    response = client.get('/api/v1/users', HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}')
    data = response.json()

    assert response.status_code == HTTPStatus.OK
    assert data['count'] == 1
    assert data['items'][0]['username'] == 'admin'
```

E agora os testes todos de Users corrigidos para fazer a autenticação antes de rodar os requests:

```python title="myapi/core/tests/test_users.py"
import json
from http import HTTPStatus
from decouple import config
from django.contrib.auth import get_user_model

import pytest


@pytest.fixture
def create_admin_access_token(client):
    response = client.post(
        '/api/v1/login',
        data={'username': config('DJANGO_ADMIN_USER'), 'password': config('DJANGO_ADMIN_PASSWORD')},
    )
    response_json = response.json()
    access_token = response_json['access_token']
    return access_token


@pytest.mark.django_db
def test_list_users(client, create_admin_access_token):
    response = client.get('/api/v1/users', HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}')
    data = response.json()

    assert response.status_code == HTTPStatus.OK
    assert data['count'] == 1
    assert data['items'][0]['username'] == 'admin'


@pytest.mark.django_db
def test_get_user_detail(client, create_admin_access_token):
    User = get_user_model()
    admin = User.objects.get(username=config('DJANGO_ADMIN_USER'))

    response = client.get(f'/api/v1/users/{admin.id}', HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}')
    data = response.json()

    assert response.status_code == HTTPStatus.OK
    assert data['username'] == config('DJANGO_ADMIN_USER')


@pytest.mark.django_db
def test_create_users_success(client, create_admin_access_token):
    user_payload = {
        'username': 'admin_new',
        'first_name': 'New',
        'last_name': 'Admin',
        'email': 'admin_new@admin.com',
        'password': 'myadminpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )
    response_json = response.json()

    assert response.status_code == HTTPStatus.CREATED
    assert response_json['username'] == user_payload['username']


@pytest.mark.django_db
def test_create_users_duplicated_username(client, create_admin_access_token):
    user_payload = {
        'username': 'admin',
        'first_name': 'New',
        'last_name': 'Admin',
        'email': 'admin_new@admin.com',
        'password': 'myadminpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )
    assert response.status_code == HTTPStatus.CONFLICT


@pytest.mark.django_db
def test_create_users_duplicated_email(client, create_admin_access_token):
    user_payload = {
        'username': 'admin_new',
        'first_name': 'New',
        'last_name': 'Admin',
        'email': 'admin@admin.com',
        'password': 'myadminpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )
    assert response.status_code == HTTPStatus.CONFLICT


@pytest.mark.django_db
def test_delete_user(client, create_admin_access_token):
    user_payload = {
        'username': 'admin_new',
        'first_name': 'New',
        'last_name': 'Admin',
        'email': 'admin_new@admin.com',
        'password': 'myadminpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )
    user_id = response.json()['id']

    response = client.delete(f'/api/v1/users/{user_id}', HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}')
    assert response.status_code == HTTPStatus.NO_CONTENT


@pytest.mark.django_db
def test_patch_user(client, create_admin_access_token):
    user_payload = {
        'username': 'admin_new',
        'first_name': 'New',
        'last_name': 'Admin',
        'email': 'admin_new@admin.com',
        'password': 'myadminpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )
    user_id = response.json()['id']

    patch_data = {
        'first_name': 'NewName',
        'email': 'newemail@admin.com',
    }

    response = client.patch(
        f'/api/v1/users/{user_id}',
        data=json.dumps(patch_data),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )

    response_json = response.json()
    assert response.status_code == HTTPStatus.OK
    assert response_json['first_name'] == 'NewName'
    assert response_json['email'] == 'newemail@admin.com'
```

!!! success

    Agora temos a autenticação funcionando, assim com todos os testes que dependem dela! 🙌🙌🙌

## Autorização

Por enquanto, estamos apenas validando se o usuário está autenticado para poder executar essas operações, mas na verdade a gente precisa que apenas Usuários **ADMIN** possam fazer certas operações, como criar, listar, deletar e alterar usuários. Não queremos que um usuário qualquer logado posso fazer isso, né? Aí que entra a autorização.

Faremos o seguinte: o usuário admin pode fazer tudo isso, mas o usuário não admin poderá listar os detalhes dele mesmo, alterar as configurações dele mesmo, e apagar ele mesmo.

Vamos começar com as rotas que são permitidas apenas pelos admins. Uma forma fácil seria verificar dentro da própria rota com a propriedade `is_staff`, assim:

```python title="myapi/core/api.py"
@router.get('users', response=list[UserWithGroupsSchema], auth=JWTAuth())
@paginate
def list_users(request):
    user = request.auth
    if not user or not user.is_staff:
        return Response({'detail': 'Forbidden'}, status=403)
    return User.objects.all()
```

Mas se formos usar isso em muitas rotas, compensa criar uma classe que valida se o usuário é Admin, e colocamos ela no decorator de cada rota. Vamos implementar dessa forma. No arquivo `auth.py`, vamos adicionar a classe AdminAuth, herdando do `JWTAuth` que já tinhamos criado antes:

```python title="main/core/auth.py"
# Classe já existente
class JWTAuth(HttpBearer):
    def authenticate(self, request, token):
        try:
            payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[ALGO])
            if payload.get('type') != 'access':
                return None
            user_id = payload.get('user_id')
            return User.objects.get(id=user_id)
        except jwt.ExpiredSignatureError:
            return None
        except Exception:
            return None

# Criar essa classe nova
class AdminAuth(JWTAuth):
    def authenticate(self, request, token):
        user = super().authenticate(request, token)
        if not user:
            return None
        if not getattr(user, "is_staff", False):
            return None
        return user
```

E agora no `api.py` vamos importar essa classe e chamá-la no decorator das rotas que precisam de permissão de Admin:

```python title="myapi/core/api.py"
from .auth import create_token, JWTAuth, AdminAuth

@router.get(
    'users',
    response=list[UserWithGroupsSchema],
    summary='List users',
    description='List users',
    auth=AdminAuth() #<= Alterar aqui
)
@paginate
def list_users(request):
    return User.objects.all()
```

Agora quando um usuário normal (não admin) tentar chamar essa rota, ele vai tomar um erro 401. Mas vamos criar testes para isso. Antes, vou aplicar essa mudança nas rotas de get e post. O Patch e Delete vamos fazer diferente, porque um usuário poderia também chamá-las se o usuário destino for eles mesmos.

Para testar, criaremos uma nova Fixture que autentica um usuário não admin, e usamos esse access token nas rotas que precisam de admin, para confirmar se a API retorna 401:

```python title="myapi/core/tests/test_users.py"
@pytest.fixture
def create_non_admin_access_token(client, create_admin_access_token):
    # Create new non-admin user
    user_payload = {
        'username': 'new_user_non_admin',
        'first_name': 'New',
        'last_name': 'User',
        'email': 'user_new@admin.com',
        'password': 'myuserpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )

    # Get the auth token for the non-admin user
    response = client.post(
        '/api/v1/login',
        data={'username': 'new_user_non_admin', 'password': 'myuserpassword'},
    )
    response_json = response.json()
    access_token = response_json['access_token']
    return access_token

@pytest.mark.django_db
def test_list_users_unauthorized(client, create_non_admin_access_token):
    response = client.get('/api/v1/users', HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}')

    assert response.status_code == HTTPStatus.UNAUTHORIZED

@pytest.mark.django_db
def test_create_users_unauthorized(client, create_non_admin_access_token):
    user_payload = {
        'username': 'admin_new',
        'first_name': 'New',
        'last_name': 'Admin',
        'email': 'admin_new@admin.com',
        'password': 'myadminpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}',
    )
    assert response.status_code == HTTPStatus.UNAUTHORIZED
```

Ótimo, agora para as rotas de UPDATE, DELETE e GET USER DETAIL, vamos validar se o usuário logado é admin ou se é ele mesmo. Para todas essas rotas a gente passa o id no request, então precisamos comparar se o id é o mesmo do id do usuário logado. Também tem várias formas de implementar isso, mas para manter consistência, vamos fazer da mesma forma, criando uma classe OwnerOrAdminAuth, herdando de JWTAuth, e chamando no decorator das rotas.

```python title="myapi/core/auth.py"
class OwnerOrAdminAuth(JWTAuth):
    def authenticate(self, request, token):
        user = super().authenticate(request, token)
        if not user:
            return None

        target_id = None
        try:
            # Pega o ID do usuário fazendo o request
            target_id = request.path_params.get('id')
        except Exception:
            target_id = None

        # Se não há target_id, só admins tem acesso
        if not target_id:
            return user if getattr(user, 'is_staff', False) else None

        # permitir se for admin ou se for o próprio usuário
        if getattr(user, 'is_staff', False) or str(user.id) == str(target_id):
            return user

        return None
```

Agora na API é só importar a classe e chamar nos decorators:

```python title="myapi/core/api.py"
from .auth import create_token, JWTAuth, AdminAuth, OwnerOrAdminAuth

@router.get(
    'users/{id}',
    response=UserWithGroupsSchema,
    summary='Get user detail',
    description='Retrieve user details by ID',
    auth=OwnerOrAdminAuth(), #<= Alterar aqui
)
def get_user_detail(request, id: uuid.UUID):
    return get_object_or_404(User, id=id)
```

Vamos aplicar a mesma coisa nas rotas de PATCH e DELETE, e fazer os testes.

```python title="myapi/core/tests/test_users.py"

# Teste do admin fazendo o GET nele mesmo
@pytest.mark.django_db
def test_get_user_detail_admin(client, create_admin_access_token):
    User = get_user_model()
    admin = User.objects.get(username=config('DJANGO_ADMIN_USER'))

    response = client.get(f'/api/v1/users/{admin.id}', HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}')
    data = response.json()

    assert response.status_code == HTTPStatus.OK
    assert data['username'] == config('DJANGO_ADMIN_USER')

# Teste do admin fazendo o GET em outro usuário
@pytest.mark.django_db
def test_get_user_detail_admin_to_other_user(client, create_admin_access_token, create_non_admin_access_token):
    User = get_user_model()
    user = User.objects.get(username='new_user_non_admin')

    response = client.get(f'/api/v1/users/{user.id}', HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}')
    data = response.json()

    assert response.status_code == HTTPStatus.OK
    assert data['username'] == 'new_user_non_admin'


# Teste de um usuário não admin fazendo GET nele mesmo
@pytest.mark.django_db
def test_get_user_detail_user_to_himself(client, create_non_admin_access_token):
    User = get_user_model()
    user = User.objects.get(username='new_user_non_admin')

    response = client.get(f'/api/v1/users/{user.id}', HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}')
    data = response.json()

    assert response.status_code == HTTPStatus.OK
    assert data['username'] == 'new_user_non_admin'

# Teste de um usuário fazendo GET em outro usuário (Falha!)
@pytest.mark.django_db
def test_get_user_detail_user_to_other_user_fail(client, create_non_admin_access_token):
    User = get_user_model()
    user_admin = User.objects.get(username=config('DJANGO_ADMIN_USER'))

    response = client.get(f'/api/v1/users/{user_admin.id}', HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}')
    assert response.status_code == HTTPStatus.UNAUTHORIZED

# Teste de delete de um usuário pelo admin
@pytest.mark.django_db
def test_delete_user_admin_to_other_user(client, create_admin_access_token, create_non_admin_access_token):
    User = get_user_model()
    user = User.objects.get(username='new_user_non_admin')

    response = client.delete(f'/api/v1/users/{user.id}', HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}')
    assert response.status_code == HTTPStatus.NO_CONTENT

# Teste de delete de um usuário por ele mesmo
@pytest.mark.django_db
def test_delete_user_to_himself(client, create_non_admin_access_token):
    User = get_user_model()
    user = User.objects.get(username='new_user_non_admin')

    response = client.delete(f'/api/v1/users/{user.id}', HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}')
    assert response.status_code == HTTPStatus.NO_CONTENT

# Teste de delete de um usuário por outro usuário (Falha!)
@pytest.mark.django_db
def test_delete_user_to_other_user_fail(client, create_non_admin_access_token):
    User = get_user_model()
    user_admin = User.objects.get(username=config('DJANGO_ADMIN_USER'))

    response = client.delete(f'/api/v1/users/{user_admin.id}', HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}')
    assert response.status_code == HTTPStatus.UNAUTHORIZED

# Teste de patch de um usuário pelo admin
@pytest.mark.django_db
def test_patch_user_admin_to_other_user(client, create_admin_access_token, create_non_admin_access_token):
    User = get_user_model()
    user = User.objects.get(username='new_user_non_admin')

    patch_data = {
        'first_name': 'NewName',
        'email': 'newemail@admin.com',
    }

    response = client.patch(
        f'/api/v1/users/{user.id}',
        data=json.dumps(patch_data),
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )
    response_json = response.json()
    assert response.status_code == HTTPStatus.OK
    assert response_json['first_name'] == 'NewName'
    assert response_json['email'] == 'newemail@admin.com'

# Teste de patch de um usuário por ele mesmo
@pytest.mark.django_db
def test_patch_user_to_himself(client, create_non_admin_access_token):
    User = get_user_model()
    user = User.objects.get(username='new_user_non_admin')

    patch_data = {
        'first_name': 'NewName',
        'email': 'newemail@admin.com',
    }

    response = client.patch(
        f'/api/v1/users/{user.id}',
        data=json.dumps(patch_data),
        HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}',
    )
    response_json = response.json()
    assert response.status_code == HTTPStatus.OK
    assert response_json['first_name'] == 'NewName'
    assert response_json['email'] == 'newemail@admin.com'

# Teste de patch de um usuário por outro usuário (Falha!)
@pytest.mark.django_db
def test_patch_user_to_other_user_fail(client, create_non_admin_access_token):
    User = get_user_model()
    user_admin = User.objects.get(username=config('DJANGO_ADMIN_USER'))

    patch_data = {
        'first_name': 'NewName',
        'email': 'newemail@admin.com',
    }

    response = client.patch(
        f'/api/v1/users/{user_admin.id}',
        data=json.dumps(patch_data),
        HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}',
    )
    assert response.status_code == HTTPStatus.UNAUTHORIZED
```

## Resumo do código

Com isso a gente termina a Autenticação e Autorização do CRUD de Users, abaixo o resumo de como está o código

```python title="myapi/core/auth.py"
# myapi/core/auth.py
import jwt
from datetime import datetime, timedelta
from django.conf import settings
from django.contrib.auth import get_user_model
from ninja.security import HttpBearer

User = get_user_model()
ALGO = 'HS256'
ACCESS_LIFETIME = timedelta(minutes=15)
REFRESH_LIFETIME = timedelta(days=7)


def create_token(user):
    now = datetime.utcnow()
    access_token = jwt.encode(
        {'user_id': str(user.id), 'exp': now + ACCESS_LIFETIME, 'type': 'access'},
        settings.SECRET_KEY,
        algorithm=ALGO,
    )
    refresh_token = jwt.encode(
        {'user_id': str(user.id), 'exp': now + REFRESH_LIFETIME, 'type': 'refresh'},
        settings.SECRET_KEY,
        algorithm=ALGO,
    )
    return {'access_token': access_token, 'refresh_token': refresh_token}


class JWTAuth(HttpBearer):
    def authenticate(self, request, token):
        try:
            payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[ALGO])
            if payload.get('type') != 'access':
                return None
            user_id = payload.get('user_id')
            return User.objects.get(id=user_id)
        except jwt.ExpiredSignatureError:
            return None
        except Exception:
            return None


class AdminAuth(JWTAuth):
    def authenticate(self, request, token):
        user = super().authenticate(request, token)
        if not user:
            return None
        if not getattr(user, 'is_staff', False):
            return None
        return user


class OwnerOrAdminAuth(JWTAuth):
    def authenticate(self, request, token):
        user = super().authenticate(request, token)
        if not user:
            return None

        target_id = None
        try:
            # Pega o ID do usuário fazendo o request
            target_id = str(request.path).split('/')[-1]
        except Exception:
            target_id = None

        # Se não há target_id, só admins tem acesso
        if not target_id:
            return user if getattr(user, 'is_staff', False) else None

        # permitir se for admin ou se for o próprio usuário
        if getattr(user, 'is_staff', False) or str(user.id) == str(target_id):
            return user

        return None
```

```python title="myapi/core/api.py"
import uuid
from http import HTTPStatus
from django.db import connection
from ninja import Router, Form
from ninja.responses import Response
from django.contrib.auth import get_user_model, authenticate
from ninja.pagination import paginate
from django.shortcuts import get_object_or_404


from .schemas import (
    StatusSchema,
    UserSchema,
    UserWithGroupsSchema,
    UserCreateSchema,
    UserPatchSchema,
    TokenResponse,
    ErrorSchema,
)

from .auth import create_token, JWTAuth, AdminAuth, OwnerOrAdminAuth

router = Router(tags=['Admin'])

User = get_user_model()


@router.get(
    'status',
    response=StatusSchema,
    summary='Status Check',
    description='Status check endpoint to monitor the API health.',
)
def status(request):
    with connection.cursor() as cursor:
        # Database version
        cursor.execute('SELECT version()')
        db_version = cursor.fetchone()[0]

        # Maximum number of connections
        cursor.execute('SHOW max_connections')
        max_connections = int(cursor.fetchone()[0])

        # Active connections
        cursor.execute('SELECT count(*) FROM pg_stat_activity')
        active_connections = int(cursor.fetchone()[0])

    return HTTPStatus.OK, {
        'status': 'ok',
        'db_version': db_version,
        'max_connections': max_connections,
        'active_connections': active_connections,
    }


##############
# Users
##############
@router.get(
    'users', response=list[UserWithGroupsSchema], summary='List users', description='List users', auth=AdminAuth()
)
@paginate
def list_users(request):
    return User.objects.all()


@router.get(
    'users/{id}',
    response=UserWithGroupsSchema,
    summary='Get user detail',
    description='Retrieve user details by ID',
    auth=OwnerOrAdminAuth(),
)
def get_user_detail(request, id: uuid.UUID):
    return get_object_or_404(User, id=id)


@router.post(
    'users', response=UserWithGroupsSchema, summary='Create user', description='Create a new user', auth=AdminAuth()
)
def create_users(request, data: UserCreateSchema):
    # Pre-create validation: check username and email uniqueness
    if User.objects.filter(username=data.username).exists():
        return Response({'detail': 'Username or email already exist!'}, status=409)
    if User.objects.filter(email=data.email).exists():
        return Response({'detail': 'Username or email already exist!'}, status=409)

    try:
        user = User.objects.create_user(
            username=data.username,
            first_name=data.first_name,
            last_name=data.last_name,
            email=data.email,
            password=data.password,
        )
    except:
        return Response({'detail': 'Unable to create user'}, status=500)

    return Response(UserWithGroupsSchema.from_orm(user), status=201)


@router.delete(
    'users/{id}', summary='Delete user', response={204: None}, description='Delete an user', auth=OwnerOrAdminAuth()
)
def delete_user(request, id: uuid.UUID):
    user = get_object_or_404(User, id=id)
    user.delete()
    return Response(None, status=204)


@router.patch(
    'users/{id}',
    response=UserWithGroupsSchema,
    summary='Update user partially',
    description='Update only specified user fields',
    auth=OwnerOrAdminAuth(),
)
def patch_user(request, id: uuid.UUID, payload: UserPatchSchema):
    user = get_object_or_404(User, id=id)

    for field, value in payload.dict(exclude_unset=True).items():
        setattr(user, field, value)

    user.save()
    return Response(UserWithGroupsSchema.from_orm(user), status=200)


########
# AUTH
########
@router.post('login', tags=['Auth'], response={200: TokenResponse, 401: ErrorSchema})
def login(request, username: str = Form(...), password: str = Form(...)):
    user = authenticate(username=username, password=password)
    if not user:
        return 401, {'detail': 'Invalid credentials'}
    tokens = create_token(user)
    return 200, {'access_token': tokens.get('access_token') or tokens.get('access'), 'token_type': 'bearer', **tokens}
```

```python title="myapi/core/schemas.py"
from django.contrib.auth import get_user_model
from ninja import Field, ModelSchema, Schema
from ninja.orm import create_schema

# Para criar um novo Schema de User baseado no Model User
User = get_user_model()


class StatusSchema(Schema):
    status: str
    db_version: str
    max_connections: int
    active_connections: int


class UserSchema(ModelSchema):  # <= Não está sendo usado, é apenas para referência
    class Meta:
        model = User
        exclude = ['password', 'last_login', 'date_joined', 'user_permissions', 'groups']


UserWithGroupsSchema = create_schema(
    User,
    depth=1,
    fields=['id', 'username', 'first_name', 'last_name', 'email', 'groups'],
    custom_fields=[('get_full_name', str, None)],
)


class UserCreateSchema(Schema):
    username: str = Field(..., example='newuser')
    first_name: str = Field(..., example='Firstname')
    last_name: str = Field(..., example='Lastname')
    email: str = Field(..., example='newuser@email.com')
    password: str = Field(..., example='strongpassword')


class UserPatchSchema(Schema):
    username: str | None = None
    first_name: str | None = None
    last_name: str | None = None
    email: str | None = None


class TokenResponse(Schema):
    access_token: str
    refresh_token: str | None = None
    token_type: str = 'bearer'


class ErrorSchema(Schema):
    detail: str
```

```python title="myapi/core/tests/test_users.py"
import json
from http import HTTPStatus
from decouple import config
from django.contrib.auth import get_user_model
import pytest


@pytest.fixture
def create_admin_access_token(client):
    response = client.post(
        '/api/v1/login',
        data={'username': config('DJANGO_ADMIN_USER'), 'password': config('DJANGO_ADMIN_PASSWORD')},
    )
    response_json = response.json()
    access_token = response_json['access_token']
    return access_token


@pytest.fixture
def create_non_admin_access_token(client, create_admin_access_token):
    # Create new non-admin user
    user_payload = {
        'username': 'new_user_non_admin',
        'first_name': 'New',
        'last_name': 'User',
        'email': 'user_new@admin.com',
        'password': 'myuserpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )

    # Get the auth token for the non-admin user
    response = client.post(
        '/api/v1/login',
        data={'username': 'new_user_non_admin', 'password': 'myuserpassword'},
    )
    response_json = response.json()
    access_token = response_json['access_token']
    return access_token


@pytest.mark.django_db
def test_list_users(client, create_admin_access_token):
    response = client.get('/api/v1/users', HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}')
    data = response.json()

    assert response.status_code == HTTPStatus.OK
    assert data['count'] == 1
    assert data['items'][0]['username'] == 'admin'


@pytest.mark.django_db
def test_list_users_unauthorized(client, create_non_admin_access_token):
    response = client.get('/api/v1/users', HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}')

    assert response.status_code == HTTPStatus.UNAUTHORIZED


@pytest.mark.django_db
def test_get_user_detail_admin(client, create_admin_access_token):
    User = get_user_model()
    admin = User.objects.get(username=config('DJANGO_ADMIN_USER'))

    response = client.get(f'/api/v1/users/{admin.id}', HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}')
    data = response.json()

    assert response.status_code == HTTPStatus.OK
    assert data['username'] == config('DJANGO_ADMIN_USER')


@pytest.mark.django_db
def test_get_user_detail_admin_to_other_user(client, create_admin_access_token, create_non_admin_access_token):
    User = get_user_model()
    user = User.objects.get(username='new_user_non_admin')

    response = client.get(f'/api/v1/users/{user.id}', HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}')
    data = response.json()

    assert response.status_code == HTTPStatus.OK
    assert data['username'] == 'new_user_non_admin'


@pytest.mark.django_db
def test_get_user_detail_user_to_himself(client, create_non_admin_access_token):
    User = get_user_model()
    user = User.objects.get(username='new_user_non_admin')

    response = client.get(f'/api/v1/users/{user.id}', HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}')
    data = response.json()

    assert response.status_code == HTTPStatus.OK
    assert data['username'] == 'new_user_non_admin'


@pytest.mark.django_db
def test_get_user_detail_user_to_other_user_fail(client, create_non_admin_access_token):
    User = get_user_model()
    user_admin = User.objects.get(username=config('DJANGO_ADMIN_USER'))

    response = client.get(
        f'/api/v1/users/{user_admin.id}', HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}'
    )
    assert response.status_code == HTTPStatus.UNAUTHORIZED


@pytest.mark.django_db
def test_create_users_success(client, create_admin_access_token):
    user_payload = {
        'username': 'admin_new',
        'first_name': 'New',
        'last_name': 'Admin',
        'email': 'admin_new@admin.com',
        'password': 'myadminpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )
    response_json = response.json()

    assert response.status_code == HTTPStatus.CREATED
    assert response_json['username'] == user_payload['username']


@pytest.mark.django_db
def test_create_users_unauthorized(client, create_non_admin_access_token):
    user_payload = {
        'username': 'admin_new',
        'first_name': 'New',
        'last_name': 'Admin',
        'email': 'admin_new@admin.com',
        'password': 'myadminpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}',
    )
    assert response.status_code == HTTPStatus.UNAUTHORIZED


@pytest.mark.django_db
def test_create_users_duplicated_username(client, create_admin_access_token):
    user_payload = {
        'username': 'admin',
        'first_name': 'New',
        'last_name': 'Admin',
        'email': 'admin_new@admin.com',
        'password': 'myadminpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )
    assert response.status_code == HTTPStatus.CONFLICT


@pytest.mark.django_db
def test_create_users_duplicated_email(client, create_admin_access_token):
    user_payload = {
        'username': 'admin_new',
        'first_name': 'New',
        'last_name': 'Admin',
        'email': 'admin@admin.com',
        'password': 'myadminpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )
    assert response.status_code == HTTPStatus.CONFLICT


@pytest.mark.django_db
def test_delete_user(client, create_admin_access_token):
    user_payload = {
        'username': 'admin_new',
        'first_name': 'New',
        'last_name': 'Admin',
        'email': 'admin_new@admin.com',
        'password': 'myadminpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )
    user_id = response.json()['id']

    response = client.delete(f'/api/v1/users/{user_id}', HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}')
    assert response.status_code == HTTPStatus.NO_CONTENT


@pytest.mark.django_db
def test_delete_user_admin_to_other_user(client, create_admin_access_token, create_non_admin_access_token):
    User = get_user_model()
    user = User.objects.get(username='new_user_non_admin')

    response = client.delete(f'/api/v1/users/{user.id}', HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}')
    assert response.status_code == HTTPStatus.NO_CONTENT


@pytest.mark.django_db
def test_delete_user_to_himself(client, create_non_admin_access_token):
    User = get_user_model()
    user = User.objects.get(username='new_user_non_admin')

    response = client.delete(f'/api/v1/users/{user.id}', HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}')
    assert response.status_code == HTTPStatus.NO_CONTENT


@pytest.mark.django_db
def test_delete_user_to_other_user_fail(client, create_non_admin_access_token):
    User = get_user_model()
    user_admin = User.objects.get(username=config('DJANGO_ADMIN_USER'))

    response = client.delete(
        f'/api/v1/users/{user_admin.id}', HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}'
    )
    assert response.status_code == HTTPStatus.UNAUTHORIZED


@pytest.mark.django_db
def test_patch_user(client, create_admin_access_token):
    user_payload = {
        'username': 'admin_new',
        'first_name': 'New',
        'last_name': 'Admin',
        'email': 'admin_new@admin.com',
        'password': 'myadminpassword',
    }
    response = client.post(
        '/api/v1/users',
        data=json.dumps(user_payload),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )
    user_id = response.json()['id']

    patch_data = {
        'first_name': 'NewName',
        'email': 'newemail@admin.com',
    }

    response = client.patch(
        f'/api/v1/users/{user_id}',
        data=json.dumps(patch_data),
        content_type='application/json',
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )

    response_json = response.json()
    assert response.status_code == HTTPStatus.OK
    assert response_json['first_name'] == 'NewName'
    assert response_json['email'] == 'newemail@admin.com'


@pytest.mark.django_db
def test_patch_user_admin_to_other_user(client, create_admin_access_token, create_non_admin_access_token):
    User = get_user_model()
    user = User.objects.get(username='new_user_non_admin')

    patch_data = {
        'first_name': 'NewName',
        'email': 'newemail@admin.com',
    }

    response = client.patch(
        f'/api/v1/users/{user.id}',
        data=json.dumps(patch_data),
        HTTP_AUTHORIZATION=f'Bearer {create_admin_access_token}',
    )
    response_json = response.json()
    assert response.status_code == HTTPStatus.OK
    assert response_json['first_name'] == 'NewName'
    assert response_json['email'] == 'newemail@admin.com'


@pytest.mark.django_db
def test_patch_user_to_himself(client, create_non_admin_access_token):
    User = get_user_model()
    user = User.objects.get(username='new_user_non_admin')

    patch_data = {
        'first_name': 'NewName',
        'email': 'newemail@admin.com',
    }

    response = client.patch(
        f'/api/v1/users/{user.id}',
        data=json.dumps(patch_data),
        HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}',
    )
    response_json = response.json()
    assert response.status_code == HTTPStatus.OK
    assert response_json['first_name'] == 'NewName'
    assert response_json['email'] == 'newemail@admin.com'



@pytest.mark.django_db
def test_patch_user_to_other_user_fail(client, create_non_admin_access_token):
    User = get_user_model()
    user_admin = User.objects.get(username=config('DJANGO_ADMIN_USER'))

    patch_data = {
        'first_name': 'NewName',
        'email': 'newemail@admin.com',
    }

    response = client.patch(
        f'/api/v1/users/{user_admin.id}',
        data=json.dumps(patch_data),
        HTTP_AUTHORIZATION=f'Bearer {create_non_admin_access_token}',
    )
    assert response.status_code == HTTPStatus.UNAUTHORIZED
```
