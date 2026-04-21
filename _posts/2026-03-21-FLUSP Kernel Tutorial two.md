---
title: Tutorial 2 Building and booting a custom Linux kernel for ARM using kw
date: 2026-03-21
categories: [Dev, Kernel_Dev]
tags: [kernel]
---

<em>(disponível em <https://flusp.ime.usp.br/kernel/build-linux-for-arm-kw/>)</em>

Prosseguindo com meu treinamento em desenvolvimento Kernel usando os tutoriais Flusp, cheguei na primeira etapa em que usarei a ferramenta <strong>KW</strong> para me auxiliar no desenvolvimento e testes do Kernel Linux, ferramenta esta que será muito recorrente no meu processo de desenvolvimento.


O <strong>KW</strong> (<em>Kernel Workflow</em>) é um sistema de automação de código aberto feito por membros da **USP** voltado para desenvolvedores, que visa simplificar a configuração do ambiente de desenvolvimento Linux. Assim como eu utilizei o libvirt para otimizar o manejo de máquinas virtuais, o kw automatiza processos críticos e repetitivos, como a compilação e o deploy de kernels customizados. Ele atua desde a busca de configurações otimizadas até a gestão de dependências e instalação de módulos, permitindo que o foco permaneça na implementação e nos testes, reduzindo significativamente a carga operacional do fluxo de trabalho.

No tutorial fui apresentado aos repositórios de desenvolvimento <strong>Linux</strong>, conhecidos como <strong><em>Linux Kernel Trees</em></strong>, onde ocorre todo o desenvolvimento do Kernel, com cada repositório seguindo regras e padrões estabelecidos pela própria comunidade.

Neste tutorial utilizei o repositório do <em>Industrial I/O (IIO) subsystem</em>, comecei clonando o repositório utilizando o git:

```bash
git clone git://git.kernel.org/pub/scm/linux/kernel/git/jic23/iio.git "${IIO_TREE}" --branch testing --single-branch --depth 10
```

Variaveis importantes de se notar neste comando git, ```--branch testing --single-branch``` escolhe especificamente a branch teste deste repositório, e o comando ```--depth 10``` clona o repositório somente com os últimos 10 commits, isto é importante nesse caso pois se eu fizesse o download do repositório <em>Kernel</em> com todo o histórico de desenvolvimento, eu ocuparia um enorme espaço de memória.

Ao longo do tutorial fui apresentado varios comandos do <em>Kw</em> para entender como funciona sua estrutura.
