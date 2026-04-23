---
title: Tutorial 5 Sending patches with git and a USP email
date: 2026-04-08
categories: [Dev, Kernel_Dev]
tags: [kernel]
---
<em>disponível em(https://flusp.ime.usp.br/git/sending-patches-with-git-and-a-usp-email/)</em>

Este tutorial remete a ultima, porém não mais importante, etapa do desenvolvimento de Kernel, o envio da contribuição.

Diferente da maioria dos projetos onde costumo mandar meus commits para um branch separada do github para depois pedir um pull request, contribuições de projetos baseados em Linux são feitos enviando um PATCH por e-mail usando o *Git*

Este tutorial é mais específico para alunos da USP já que ele ensina especificamente como mandar o PATCH usando um e-mail USP. Para configurar o e-mail USP usamos o git-credential helper.

Com as credenciais configuradas, realizei um envio simulado com ``git send-email`` para testar se não houve nenhum erro. Primeiramente, criei um repositório git novo, e criei um commit simples, e adicionei essas configurações de e-mail no arquivo config utilizando o comando ``git config --local --edit`` 
```bash
...
[credential "smtp://smtp.gmail.com:465"]
helper = 
helper = gmail
[sendemail]
smtpEncryption = ssl
smtpServer = smtp.gmail.com
smtpUser = erick.am@usp.br
smtpServerPort = 465
smtpAuth = OAUTHBEARER
from = Erick Henrique
```

Com o repositório git configurado, restava apenas o teste em si. O *kw* também nos ajuda nesta etapa do desenvolvimento com o comando ``kw send-patch`` 
```bash
kw send-patch --send -1 --private --simulate --to='<EMAIL-ADDRESS-1>','<EMAIL-ADDRESS-2>' ...
```
Rodando este comando com a variável ``--simulate`` garante que o e-mail não é enviado, assim é possível ver o corpo do e-mail e se todas as informações do PATCH estão colocadas corretamente.

Quando for a hora de enviar o PATCH aos mantenedores, basta remover o comando ``--simulate`` para fazer o envio real.