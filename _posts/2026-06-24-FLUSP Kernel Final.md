---
categories: [Dev, Kernel_Dev]
date: 2026-06-24
tags: [kernel, mac5856]
title: Apresentação Final - Proposta e Contribuições
---

# Proposta de Pesquisa

Como uma das etapas finais da disciplina, foi designada a tarefa de estudar o dataset do kernel. O nosso conjunto de dados está disponível em: 
<https://docs.google.com/spreadsheets/d/1i_GRbsFhhvW00G9Pl36JfrToa-q-JmOxgRPSkbe1fHg/edit?usp=sharing>. 

Escolhemos coletar dados gerais do todo, extraídos do repositório oficial da árvore de desenvolvimento do Kernel Linux no GitHub, sob a gerência de Linus Torvalds. Por se tratar do repositório centralizado de produção, ele indexa exclusivamente os patches (ajustes de código) que foram revisados, aprovados e integrados à branch principal. Estruturamos em uma planilha do Excel as variáveis coletadas para as colunas: Nome do Autor, E-mail, Data com Fuso Horário Original e Dia da Semana. Ao todo, as linhas totalizaram 1.431.269 commits com dados abrangendo de 2007 a 2026 (um período de 19 anos). 

No link indicado, estão disponíveis mais informações pertinentes. Nossas perguntas iniciais foram: de onde vêm os commits no globo? Esses commits são realizados em horário comercial (workhour definido de segunda a sexta, das 8h às 18h)? Para realizar a filtragem, usamos a função Query do Excel: primeiro separamos os dados por todos os fusos horários do globo e, para cada um, contabilizamos a quantidade total de commits e a quantidade daqueles que foram realizados nesse intervalo comercial. Organizamos tudo em uma tabela no Excel.

Diante desse cenário corporativo, a hipótese natural seria encontrar uma atividade concentrada quase na totalidade dentro do horário comercial padrão (Workhour: segunda a sexta, das 08h às 18h) das respectivas regiões geográficas dos contribuidores. Contudo, para fundamentar nossa proposta de pesquisa, observamos que 39,43% de todo o trabalho técnico em um dos softwares mais críticos do planeta está ocorrendo fora desse intervalo, ou seja, em zonas de hora extra, madrugadas ou finais de semana. Olhando para as regiões e desconsiderando os dados isolados (*), a diferença é bem clara: nas Américas, Europa e África, de 50% a 63% dos commits acontecem no horário comercial. Já no Oriente Médio, Ásia Central e Sul da Ásia (especialmente nos fusos UTC+4, +5 e +6, sem contar a Índia), esse número cai para a faixa de 33% a 34%. Diante disso, propõe-se investigar se esse fenômeno decorre de:

(a) Diferença nos calendários: a nossa definição de horário comercial (segunda a sexta, das 8h às 18h) pode não fazer sentido nesses países por causa de rotinas culturais ou religiosas diferentes; ou
(b) Trabalho para o exterior: esses desenvolvedores podem trabalhar para empresas dos EUA ou da Europa e precisam mudar seus horários para se adaptar ao fuso horário dos clientes lá fora.

O que explica a barreira dos ~60% de Workhour nas regiões com muitos commits? Trata-se de uma jornada exploratória/invisível ou de flexibilidade de tempo? Propomos uma pesquisa que investigue se os desenvolvedores estão sofrendo sobrecarga (burnout/trabalho assíncrono estendido) ou se a cultura de engenharia de software permite uma alta fragmentação de horários. Além disso, entender se essas conclusões condiz com a lista de discussões e qual é o perfil dos contribuidores que operam estritamente fora do horário comercial (não-workhour).


## Contribuição Patch 1: Deduplicação

Para recordar a nossa contribuição, enviamos nosso patch às 6h da manhã para alinhar com seus fusos horários. Como resultado, recebemos uma resposta no mesmo dia de Christian König, que forneceu o seguinte feedback:

>"The coding style here looks completely broken. Additional to that the separation was intentional [...] What we could do is to move this chunk into a common function, there should be multiple copies of it in the SDMA code as well."

A recomendação do mantenedor Christian König foi implementada em todas as instâncias onde o código aparece, restando apenas realizar o commit.
Como era o trecho original da função:


```bash
DRM_DEBUG("IH: CP EOP\n");

    if (adev->enable_mes && doorbell_offset) {
        struct xarray *xa = &adev->userq_doorbell_xa;
        struct amdgpu_usermode_queue *queue;
        unsigned long flags;

        xa_lock_irqsave(xa, flags);
        queue = xa_load(xa, doorbell_offset);
        if (queue)
            amdgpu_userq_fence_driver_process(queue->fence_drv);

        xa_unlock_irqrestore(xa, flags);
    } else {
```

E como ficou com a nossa contribuição:

```bash
DRM_DEBUG("IH: CP EOP\n");

    if (adev->enable_mes && doorbell_offset) {
        amdgpu_userq_find_by_doorbell(adev, doorbell_offset);
    } else {
```
Assim ficou instanciada a função `amdgpu_userq_find_by_doorbell()` no helper `amdgpu_gfx.c`:

```bash
static void amdgpu_userq_find_by_doorbell(struct amdgpu_device *adev,
                 u32 doorbell_offset)
{
        struct xarray *xa = &adev->userq_doorbell_xa;
        struct amdgpu_usermode_queue *queue;
        unsigned long flags;

        xa_lock_irqsave(xa, flags);
        queue = xa_load(xa, doorbell_offset);
        if (queue)
            amdgpu_userq_fence_driver_process(queue->fence_drv);
        xa_unlock_irqrestore(xa, flags);
}
```

Observação: Planejamos realizar o próximo envio em um horário de workhour dos fusos da América do Norte, Europa ou África, assim como fizemos na primeira versão às 6h da manhã, quando obtivemos resposta em pouco tempo.

## Contribuição Patch 2: Kernel Workflow

Nossa conclusão foi que a issue 305 já havia sido resolvida em versões mais recentes do Kworkflow, mesmo que ela permanecesse aberta no GitHub. Dessa forma, registramos nossas observações para auxiliar os mantenedores do projeto. Agora cabe à equipe do Kworkflow avaliar a reprodução realizada e decidir pelo encerramento oficial da issue. Eles ainda não fecharam a issue, que está disponível em: <https://github.com/kworkflow/kworkflow/issues/305>

Para a segunda issue 573, disponível em: <https://github.com/kworkflow/kworkflow/issues/573>, nós resolvemos o problema e recebemos um comentário do Rodrigo Siqueira no nosso pull request:

>"Could you add a commit message that describes the issue you are addressing? Please check the git history for more references."

Então alteramos o corpo do nosso commit conforme recomendado, usando `git commit --amend`. Eles ainda não fecharam essa issue também.

Para finalizar, estamos criando um `sample` com as configurações do `git test` para ajudar a padronizar o projeto. Porém, para integrar essa mudança é necessário que a issue 573 seja fechada com o merge do nosso pull request, já que dependemos dessa continuidade para abrir a nova issue.

