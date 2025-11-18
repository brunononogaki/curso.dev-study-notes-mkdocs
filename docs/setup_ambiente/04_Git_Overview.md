# Git Overview

Ao contrário de sistemas antigos como o CVS, que utilizavam _Delta Encoding_ (mantendo o histórico de alterações com base nos **diffs**), o Git trabalha com "fotos" completas do repositório.

Quando fazemos um commit, o Git tira uma “foto” do estado atual do projeto. Para cada arquivo, ele calcula um identificador único com base no conteúdo e salva esse conteúdo na pasta `.git` como um objeto chamado Blob (Binary Large Object). Ao criar um novo commit, o Git verifica quais arquivos foram alterados:

- Arquivos modificados geram novos blobs com o conteúdo atualizado, mas os blobs antigos continuam armazenados.
- Arquivos que não mudaram não geram novos blobs; o commit apenas aponta para os blobs já existentes.

Assim, um commit é basicamente um conjunto de referências para blobs (e para a árvore que organiza esses blobs). Por isso o Git alterna rapidamente entre diferentes “fotos” do repositório. Para visualizar essas "fotos" (commits), usamos o comando `git log`:

```bash title="git log"
commit 4cd6741adf5b1033ea47e8d622b0262113366130 (HEAD -> main, origin/main, origin/fix-migrations-endpoint, fix-migrations-endpoint)
Author: Bruno Nonogaki <brunono@gmail.com>
Date:   Mon Nov 17 17:56:14 2025 -0300

    Implementing same fix as in class

commit 4531c8a9ff6e5d977cd5d8b5429ebba435ecbc17
Author: Bruno Nonogaki <brunono@gmail.com>
Date:   Mon Nov 17 17:36:39 2025 -0300

    adding try and catch

commit d06fde4f85af929674503b9261b8eecdb5e4214e
Author: Bruno Nonogaki <brunono@gmail.com>
Date:   Sun Nov 16 22:07:44 2025 -0300

    Change header for preview test

commit ecdfac3dcd6d753bdc8716eaae579fc6e8d5e315
Author: Bruno Nonogaki <brunono@gmail.com>
Date:   Wed Nov 12 21:38:44 2025 -0300

    adds api/v1/migrations endpoint
...
```

Ou use `git log --stat` para ter uma visão mais detalhada do que foi alterado em cada commit:

```bash
commit 4cd6741adf5b1033ea47e8d622b0262113366130 (HEAD -> main, origin/main, origin/fix-migrations-endpoint, fix-migrations-endpoint)
Author: Bruno Nonogaki <brunono@gmail.com>
Date:   Mon Nov 17 17:56:14 2025 -0300

    Implementing same fix as in class

 pages/api/v1/migrations.js | 35 ++++++++++++++++++++---------------
 1 file changed, 20 insertions(+), 15 deletions(-)

...
```

Ou ainda `git log --oneline` para uma visão mais resumida:

```bash
4cd6741 (HEAD -> main, origin/main, origin/fix-migrations-endpoint, fix-migrations-endpoint) Implementing same fix as in class
4531c8a adding try and catch
d06fde4 Change header for preview test
ecdfac3 adds api/v1/migrations endpoint
b6de408 Add migratios scripts
247d618 Add SSL in Database connection
93101ad Add throw error in database.js
6a584a3 finishing  status route
08c5c7d deploy script
f151262 Adicionando integração com banco de dados
708fdc9 add  endpoint
47cd4f5 Merge branch 'main' of https://github.com/brunononogaki/meubonsai-app-v2
252f63b Setting up dev environment
24a928e first commit
```

## Os ~~três~~ quatro estágios do arquivo

No Git, os arquivos podem estar em quatro estágios:

- `untracked`: Arquivo que não está sob controle do Git (na verdade, para o Git, esse arquivo nem existe, então não seria propriamente um estágio)
- `modified`: Arquivo já conhecido pelo Git, que foi alterado
- `staged`: Dos arquivos alterados, aqueles que estão prontos para serem “fotografados”
- `commit`: Arquivos já “fotografados”

Para acompanhar o estado dos arquivos, usamos o comando `git status`:

```bash
On branch main
Your branch is up to date with 'origin/main'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   mkdocs.yml

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        docs/setup_ambiente/04_Git_Overview.md
```

Para incluir arquivos modificados na área de stage, usamos `git add`.

Para salvar as alterações, fazemos o commit com `git commit -m "descrição do commit"`.

Depois do commit, teremos uma nova “foto” desses arquivos, que pode ser visualizada com `git log`:

```bash
commit f0203262c216b5fd4e311a22c2eac311224ea53a (HEAD -> main)
Author: Bruno Nonogaki <brunono@gmail.com>
Date:   Tue Nov 18 14:54:23 2025 -0300

    Adding git documentation

commit 9235c2ae3e23db0ae014c7090b4acc1757fbfb3b (origin/main)
Author: Bruno Nonogaki <brunono@gmail.com>
Date:   Mon Nov 17 18:13:02 2025 -0300

    Criando a sessão de banco de dados de prod e staging
```

No exemplo acima, criamos um novo commit chamado "Adding git documentation".

## git commit --amend

Depois de fazer o commit, a “foto” foi registrada. Mas se você quiser incluir uma nova alteração nesse mesmo commit, sem criar outro (evitando “sujar” o histórico), é possível usar:

```bash
git commit --amend
```

Veja que antes de rodar esse comando, o último commit tinha o ID `f0203262c216b5fd4e311a22c2eac311224ea53a`:

```
commit f0203262c216b5fd4e311a22c2eac311224ea53a (HEAD -> main)
Author: Bruno Nonogaki <brunono@gmail.com>
Date:   Tue Nov 18 14:54:23 2025 -0300

    Adding git documentation
```

Após rodar o `git commit --amend`, fica assim:

```
commit 8178ecd5d0c757b2c99364c990fbeb1ee2c5a334 (HEAD -> main)
Author: Bruno Nonogaki <brunono@gmail.com>
Date:   Tue Nov 18 14:54:23 2025 -0300

    Adding git documentation
```

O ID mudou para `8178ecd5d0c757b2c99364c990fbeb1ee2c5a334`! Isso acontece porque commits são imutáveis, então o Git cria uma nova “foto” e sobrescreve a anterior.

## Repositório remoto (origin)

Para enviar as alterações para o repositório remoto (fazer upload):

```bash
git push
```

Se você fez um amend, o ID do commit muda e o Git pode recusar o push. Nesse caso, use:

```bash
git push -f
```

Para sincronizar o repositório local com o remoto (fazer download):

```bash
git pull
```

## Criando uma branch

Na prática, uma branch é apenas um apontamento para um commit específico, é como um apelido. E quando estamos em uma branch, o HEAD está apontando para ela. No exemplo abaixo, a branch main aponta para o commit 02db, enquanto a branch tamanho-do-cabelo aponta para o commit 5a0d. O HEAD está atualmente na branch tamanho-do-cabelo, mas mudar de branch é só mudar esse apontamento.

![alt text](static/branches_overview.png)

Para saber em qual branch você está, é possível usar o comado `git branch`

> [!NOTE]
> Se o comando `git branch` estiver exibindo os dados em um editor de texto, você pode mudar esse comportamento com:
>
> git config --global core.pager cat

Para criar uma branch, o comando é `git branch nova-branch`.

Para entrar na branch: `git checkout nova-branch`

Também é possível fazer checkout em qualquer outro commit anterior, por exemplo: `git checkout 578b`
![alt text](static/checkout_commit_antigo.png)

E para criar uma nova branch e já fazer o checkout nela, fazemos: `git checkout -b nova-branch`

## Apagando uma branch

Para apagar uma branch, usamos a flag -d: `git branch -d nova-branch`

Mas caso a branch não tenha sido mesclada ainda, o Git mostra um aviso. Para apagar mesmo assim, use o `-D` maiúsculo: `git branch -D nova-branch`.

Se você fez checkout de volta para a branch main e resolveu apagar sem querer a nova-branch, é só o ponteiro para o commit foi removido! Ou seja, os dados ainda estão preservados! Basta rodar `git reflog` para ver o histórico do repositório, descobrir qual era o commit que tem uma sitação que o seu código esteja funcionando e fazer checkout nele.

> [!NOTE]
> Atenção: ao fazer isso, o HEAD estará apontando para um commit sem branch, chamado de "dangling".
> O garbage collection do Git vai apagar esse commit depois de 14 dias (por padrão)!

A partir desse commit, você pode criar uma nova branch com:

```
git checkout -b nova-branch c5cc524

Onde c5cc524 é o nome do commit
```

## Merge

Existem dois tipos principais de merge:

- `Fast-Forward`: Ocorre quando a branch de destino não tem commits novos em relação à branch que está sendo mesclada. Nesse caso, o Git simplesmente "avança" o ponteiro da branch de destino para o último commit da branch de origem, sem criar um novo commit de merge. É como se a história fosse linear.

Exemplo:

```
main:    A---B
feature:     \
              C---D
```

Se não houve novos commits em `main` após o ponto de divergência, ao fazer o merge:

```
git checkout main
git merge feature
```

O ponteiro de `main` vai direto para o commit D.

- `3-Way Merge`: Acontece quando as duas branches tiveram commits diferentes após o ponto de divergência. O Git então cria um novo commit de merge, combinando as alterações das duas branches.

Exemplo:

```
main:    A---B---E
feature:     \
              C---D
```

Aqui, `main` teve o commit E, enquanto `feature` teve C e D. Ao fazer o merge:

```
git checkout main
git merge feature
```

O Git cria um novo commit de merge (por exemplo, F), que une as alterações de E, C e D:

```
main:    A---B---E---F
feature:     \     /
              C---D
```

Se houver conflitos (alterações em linhas iguais nos dois lados), o Git vai pedir para você resolver manualmente antes de concluir o merge.


Esses são os dois tipos mais comuns de merge no Git. O Fast-Forward é mais simples e não deixa "marcas" na história, enquanto o 3-Way Merge preserva o histórico de desenvolvimento paralelo.
