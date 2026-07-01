---
categories: [Dev, Kernel_Dev]
date: 2026-05-20
tags: [kernel, mac5856]
title: Linux Kernel Patch 2-1 Kworkflow
---

## Contexto

Como parte da disciplina, minha dupla e eu precisávamos contribuir para um grande projeto de software livre. Escolhemos o **Kworkflow (kw)**, uma ferramenta cuja missão é reduzir a complexidade de configuração e do ambiente de desenvolvimento para o Linux Kernel. O projeto reúne diversos componentes de software em uma única interface, disponibilizando comandos que podem ser utilizados diretamente pela linha de comando após a instalação.

Durante a disciplina, os comandos do **kw** foram extremamente úteis para a realização dos tutoriais e atividades práticas. No entanto, enfrentei um problema específico em minha máquina: os comandos do Kworkflow estavam acessando um diretório diferente daquele utilizado nos tutoriais, causando comportamentos inesperados e erros durante a execução. Como solução temporária, era necessário informar explicitamente o caminho do projeto antes de executar os comandos do kw.

Após discutir o problema com um dos mantenedores do projeto, **David Tadokoro**, conseguimos identificar uma solução. Foi necessário remover completamente a instalação existente utilizando o comando:

```bash
sudo apt remove --purge kworkflow
```

Esse comando remove o programa e também apaga seus arquivos de configuração do sistema. Em seguida, realizei uma nova instalação a partir da versão mais recente do repositório utilizando `git clone`.

Durante essa investigação, descobrimos que o comportamento observado não correspondia exatamente a um bug do projeto. Na verdade, tratava-se de uma fragilidade relacionada ao tratamento de configurações locais, um cenário que aparentemente não havia sido considerado durante a implementação original. Como esse problema não se encaixava em uma issue já existente e exigiria uma análise mais aprofundada, optamos por direcionar nossos esforços para as issues abertas do projeto que já estavam catalogadas pelos mantenedores.

## Issues em aberto e documentação

Após nos familiarizarmos com o projeto, analisamos as issues abertas do Kworkflow e decidimos focar naquelas marcadas com a tag **`try to reproduce`**, disponíveis em: <https://github.com/kworkflow/kworkflow/issues?q=state%3Aopen%20label%3A%22try%20to%20reproduce%22>

Observamos que essas issues eram relativamente antigas, datadas de 2021 e 2022. Das quatro issues abertas com essa marcação, todas estão relacionadas a 'bugs' e três estavam relacionadas a 'tests' do projeto.

Antes de iniciar qualquer investigação, consultamos a documentação do Kworkflow sobre testes, disponível na seção **About Tests**: <https://kworkflow.org/content/tests.html>

Para executar os testes localmente, foi necessário instalar o **shunit2**, pois a infraestrutura de testes do Kworkflow depende dessa ferramenta. O script `run_tests.sh` detecta automaticamente sua presença no sistema através da variável de ambiente `$PATH`.

Nosso objetivo era verificar se os bugs reportados continuavam presentes nas versões atuais do projeto ou se já haviam sido corrigidos ao longo do tempo sem que as respectivas issues fossem atualizadas ou encerradas.

## Contribuição na Issue #305

A primeira issue investigada foi:

**"Test for tests/vm_test.sh uses user configuration data"**

Disponível em: <https://github.com/kworkflow/kworkflow/issues/305>

Segundo o relato original:

> Testing the vm module of kw (./run_tests.sh test vm_test) will use the installed configuration, leading to undefined behavior.

Os passos para reproduzir o problema eram:

```bash
export SHELLOPTS
bash -x ./run_tests.sh test vm_test 2>&1 | grep configurations
```

De acordo com a descrição da issue, o teste estava utilizando configurações definidas pelo usuário em vez de utilizar exclusivamente as configurações específicas do ambiente de testes.

### Investigação

O parâmetro `-x` do Bash ativa um modo de depuração que exibe cada comando executado e suas expansões. Isso permite acompanhar detalhadamente a atribuição de variáveis e o fluxo de execução do script. Ao executar o procedimento descrito na issue, observamos que o comportamento reportado originalmente não ocorria mais. O resultado obtido foi:

```bash
++ declare -gA configurations
++ declare -gA configurations_global
++ declare -gA configurations_local
++ local target_array=configurations
+++ local target_array=configurations
+++ target_array_global=configurations_global
+++ target_array_local=configurations_local
+++ parse_configuration //home/lais/Codigos/kernel_workflow/kworkflow/tests/unit/samples/kworkflow.config.config configurations configurations_global
+++ local config_array=configurations
+++ local config_array_scope=configurations_global
+++ parse_configuration /etc/xdg/.kw//home/lais/Codigos/kernel_workflow/kworkflow/tests/unit/samples/kworkflow.config.config configurations configurations_global
+++ local config_array=configurations
+++ local config_array_scope=configurations_global
+++ parse_configuration /etc/xdg/xdg-ubuntu/.kw//home/lais/Codigos/kernel_workflow/kworkflow/tests/unit/samples/kworkflow.config.config configurations configurations_global
+++ local config_array=configurations
+++ local config_array_scope=configurations_global
+++ parse_configuration /home/lais/.config/.kw//home/lais/Codigos/kernel_workflow/kworkflow/tests/unit/samples/kworkflow.config.config configurations configurations_global
+++ local config_array=configurations
+++ local config_array_scope=configurations_global
+++ local target_array=configurations
+++ local target_array=configurations
```

A saída indica que o teste está utilizando o arquivo de configuração localizado dentro do diretório de testes do próprio projeto, e não herdando configurações personalizadas do ambiente do usuário.

### Comentário enviado na issue

Com base na análise realizada, adicionamos o seguinte comentário na issue:

> While attempting to reproduce this issue, we observed that the original problem, where the configuration included user-defined settings, no longer occurs. The test-specific local configurations are now being correctly applied. This can be verified by the following output line:
>
> ```bash
> parse_configuration /home/lais/Codigos/kernel_workflow/kworkflow/tests/unit/samples/kworkflow.config.config configurations configurations_global
> ```
>
> This indicates that the configuration parser is prioritizing the test configuration file instead of inheriting settings from the user's environment, which was the behavior reported in the original issue.

Nossa conclusão foi que a issue já havia sido resolvida em versões mais recentes do Kworkflow, mesmo que ela permanecesse aberta no GitHub. Dessa forma, registramos nossas observações para auxiliar os mantenedores do projeto. Agora cabe à equipe do Kworkflow avaliar a reprodução realizada e decidir pelo encerramento oficial da issue.