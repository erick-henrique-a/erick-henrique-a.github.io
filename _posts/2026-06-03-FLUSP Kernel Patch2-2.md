---
categories: [Dev, Kernel_Dev]
date: 2026-06-03
tags: [kernel, mac5856]
title: Linux Kernel Patch 2-2 Kworkflow
---

## Contribuição na Issue #573

A segunda issue investigada foi relacionada a primeira:

>Description: explore_test fails if the option for signing git commits is set to true globally.
>How to Reproduce:

```bash
git config --global commit.gpgsign true
./run_tests.sh test explore_test
```
>Expected behavior: Successfully complete test

Disponível em: <https://github.com/kworkflow/kworkflow/issues/573>.

## Investigação

Os testes ``` ./run_tests.sh test explore_test ``` falham quando configuração do (git config --global commit.gpgsign) é verdadeira erro, para replicar o erro a issue instrui a tornar esta opção verdadeira, e caso os testes fosssem completaddos com exito, a issue não é mais um problema, caso algum dele falhe, o erro ainda existe, após o testo notamos que ainda existe a issue, ao colocar o seguinte comando: `./run_tests.sh test explore_test`. O erro se dá pois os testes usam as configurações globais, sendo que eles deviam usar as configurações locais. Notamos que o erro se dava em todos os comandos git. Então,em: `~/kworkflow/tests/unit/explore_test.sh`, adicionamos como prefixo `GIT_CONFIG_GLOBAL=/dev/null` e `GIT_CONFIG_NOSYSTEM=1` em todos os comandos git, para evitar que ele use as configurações locais:


```bash
# Setup git repository for test
 mk_fake_git

 for commit in {0..4}; do
   local file_name="${samples_names[$commit]}"
   git config --list --local
   echo "COMMIT ANTES"
   cp "$current_path/tests/unit/samples/$file_name" ./
   GIT_CONFIG_NOSYSTEM=1 GIT_CONFIG_GLOBAL=/dev/null git add "$file_name" &> /dev/null
   GIT_CONFIG_NOSYSTEM=1 GIT_CONFIG_GLOBAL=/dev/null git commit -m "Commit number $commit" &> /dev/null
   GIT_CONFIG_NOSYSTEM=1 GIT_CONFIG_GLOBAL=/dev/null git config --list --local

 done
```

E vimos que o erro da issue persistiu, os casos de teste falharam, porém o KW deu o retorno como se os testes tivessem dado certo. Rodamos os testes quando a configuração global gpgsign for igual a “false”, conseguimos o resultado esperado. Quando esta configuração está como true, precisamos ignorar as configurações globais para usar as config locais do kw. Primeiro isolamos o problema, descobrimos que os comandos `GIT_CONFIG_GLOBAL=/dev/null` e `GIT_CONFIG_NOSYSTEM=1` funcionam, após testar em um novo repositório de teste na nossa máquina. Vimos que o problema poderia estar em outro lugar. Após alguns testes, descobrimos que este problema ocorria exatamente no comando “git commit”. Então, fomos procurar outras instâncias do comando, supusemos que o `mk_fake_git` poderia ter mais instâncias do git commit já que, ele cria um repositório novo e estabelece configurações locais e descartáveis para o teste.

Dentro do arquivo /kworkflow/tests/unit/utils.sh, encontramos onde é instanciado a função mk_fake_git:

```bash
function mk_fake_git()
{
 local -r path="$PWD"

 git init -q "$path"

 touch "$path/first_file"
 printf 'This is the first file.\n' > "$path/first_file"

 git config --local user.name 'Xpto Lala'
 git config --local user.email 'test@email.com'
 git config --local test.config value

 git add first_file
 git commit -q -m 'Initial commit'

 printf 'Second change\n' >> "$path/first_file"
 git add --all
 git commit --allow-empty -q -m 'Second commit'

 printf 'Third change\n' >> "$path/first_file"
 git add --all
 git commit --allow-empty -q -m 'Third commit'
}
```

Após análise, confirmamos nossas suspeitas, e realizamos o teste, adicionando os comandos GIT_CONFIG_GLOBAL=/dev/null e GIT_CONFIG_NOSYSTEM=1 antes de todos os comandos “git commit ...” dentro dessa função. Conseguimos o resultado esperado com sucesso.

```bash
function mk_fake_git()
{
 local -r path="$PWD"


 git init -q "$path"


 touch "$path/first_file"
 printf 'This is the first file.\n' > "$path/first_file"


 git config --local user.name 'Xpto Lala'
 git config --local user.email 'test@email.com'
 git config --local test.config value


 git add first_file
 GIT_CONFIG_NOSYSTEM=1 GIT_CONFIG_GLOBAL=/dev/null git commit -q -m 'Initial commit'


 printf 'Second change\n' >> "$path/first_file"
 git add --all
 GIT_CONFIG_NOSYSTEM=1 GIT_CONFIG_GLOBAL=/dev/null git commit --allow-empty -q -m 'Second commit'


 printf 'Third change\n' >> "$path/first_file"
 git add --all
 GIT_CONFIG_NOSYSTEM=1 GIT_CONFIG_GLOBAL=/dev/null git commit --allow-empty -q -m 'Third commit'
}
```

Porém a solução ficou muito verbosa, e não muito eficiente, já que esses comandos teriam que ser executados antes de todos as instâncias de comandos git, ao observar as configurações locais, conseguimos reduzir a redûndancia adicionando mais uma variável no git.config local, commit.gpgsign false, que resolve diretamente o nosso problema, ao invés da antiga solução genérica que desconsiderava todas as configurações globais. 

```bash
function mk_fake_git()
{
 local -r path="$PWD"


 git init -q "$path"


 touch "$path/first_file"
 printf 'This is the first file.\n' > "$path/first_file"


 git config --local user.name 'Xpto Lala'
 git config --local user.email 'test@email.com'
 git config --local test.config value
 git config --local commit.gpgsign false


 git add first_file
 git commit -q -m 'Initial commit'


 printf 'Second change\n' >> "$path/first_file"
 git add --all
 git commit --allow-empty -q -m 'Second commit'


 printf 'Third change\n' >> "$path/first_file"
 git add --all
 git commit --allow-empty -q -m 'Third commit'
}
```

Seguimos os guias de contribuição na documentação do Kw, clonamos a branch unstable, então realizamos o commit na nossa branch clonada,foi realizado o seguinte commit: Add a local git configuration for commit.gpgsign=false e só depois o pull request. 


# Contribuições futuras
Após uma discussão com o mantenedor David, chegamos nos próximos passos dessa implementação, que seria a criação de um arquivo na pasta SAMPLE com um git config default incluindo essa variável “ git config --local commit.gpgsign false”. Ele mencionou que devemos nos atentar com a concatenação de strings no bash, e que é possível contribuir na refatoração de todas as outras variáveis strings para melhorar a consistência do projeto. 
