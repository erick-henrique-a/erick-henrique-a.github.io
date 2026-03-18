---
title: Tutorial 1 Setting up a test environment for Linux Kernel Dev using QEMU and libvirt
date: 2026-03-14
categories: [Dev, Kernel_Dev]
tags: [kernel]     # TAG names should always be lowercase
---
<em>(disponível em <https://flusp.ime.usp.br/kernel/qemu-libvirt-setup/>)</em>

Como aluno de mestrado na matéria de Desenvolvimento de software livre do Instituto de Matemática e Estatística da USP, tenho como avaliação, o desenvolvimento de um patch para o Kernel do <strong>Linux</strong>.

Como bacharel em Ciência da Computação, já tive muitas experiências com ambientes de desenvolvimento coletivo e com a linguagem C, porém esta é a primeira vez em que estou colaborando com um projeto open-source da magnitude do Linux, então decidi criar um blog para acompanhar o meu progresso e minha jornada do **0** ao **patch**

Eu sou um usuário de <strong><em>Windows</em></strong> e estou muito acostumado a navegar por abas e com o auxílio do mouse, então no início do tutorial houve uma dificuldade em relação aos comandos do Linux, apesar que eu já tinha uma familiaridade com Linux pois tenho um servidor com <strong><em>Ubuntu</em></strong>, mas por usar frameworks que facilitam o deploy como <strong><em>Coolify</em></strong>, acabei não aprendendo muito, este projeto é a oportunidade perfeita para desenvolver minhas habilidades com Linux de uma vez.

Todos os códigos que irei desenvolver para o meu projeto será testado em uma <strong>VM</strong> (<em>Virtual Machine</em>), isso é essencial pois códigos Kernel são de muito baixo nível, caso eu faça algo errado, corro o perigo de destruir minha própria maquina, por isso usarei uma VM para garantir minha paz. No tutorial, utilizamos <strong>QEMU </strong> para criar a VM e o <strong>libvirt</strong> para simplificar o gerenciamento dessas VMs e fazer automações. 

Fui seguindo o tutorial sem problemas até chegar na seção em que eu tinha de instalar uma imagem do Debian na VM em que eu estava configurando, após análises e ajuda do professor, descobri que 
aquele link na verdade estava deprecado, porém a solução era simples, bastava seguir o caminho do link de download e seguir a fonte até encontrar um download mais recente.

Para instalar as dependências do tutorial, utilizei o seguinte comando (já que o Ubuntu é baseado em Debian):

```bash
# Link que está deprecado
http://cdimage.debian.org/cdimage/cloud/bookworm/daily/20250217-2026/debian-12-nocloud-arm64-daily-20250217-2026.qcow2

#Link que eu usei no lugar
https://cdimage.debian.org/cdimage/cloud/bookworm/daily/20260318-2420/debian-12-nocloud-amd64-daily-20260318-2420.qcow2
```

Prosseguindo com o tutorial, tudo parecia correr bem, porém a etapa de criar uma cópia da minha imagem com uma alocação de memória maior tinha algumas divergências da minha colega Lais, que estava usando a mesma versão do Debian que eu, divergências como tamanho da imagem e quantidade de partições da memória, mas relevei e segui com o tutorial.

Posteriormente ao tentar rodar a função <em><strong>create_vm_virsh</strong></em>, meu terminal travava e não carregava a função. Os monitores que me ajudaram não estavam conseguindo identificar a razão do
travamento, então decidi tentar fazer uma depuração básica para descobrir a origem do meu problema, primeira coisa que fiz foi comentar todo código dentro da função <em><strong>create_vm_virsh</strong></em>
e colocar um <em>"echo"</em> com o texto "a funcao foi chamada" para testar se ele sequer estava rodando a funcao, ao testar o texto foi impresso na minha tela, logo deduzi que o problema 
estava em alguma linha da função, talvez algo que eu tenha copiado errado do tutorial e gerou uma possível recursão infinita.

Comecei a comparar meu arquivo <strong><em>activate.sh</em></strong> com a da minha colega Lais, que estava operando corretamente, após uma análise minuciosa linha por linha notei uma discrepância, o arquivo
Kernel que ela atribuia na função terminava com **arm64** enquanto o meu terminava com **amd64**, um erro que cometi a várias etapas atrás quando fui baixar a imagem do Debian, havia tantas
versões e com nomes de arquivos tão grandes acabei não me atentando e baixei a versão amd64 ao invés de arm64, isso acabou explicando também o motivo da discriepância entre a quantidade
de partições e o tamanho da imagem.

Isso me serviu de lição para estar sempre muito atento aos detalhes.

Mas ao final, consegui acessar a máquina virtual via SSH, e completei o tutorial sem maiores dificuldades.

