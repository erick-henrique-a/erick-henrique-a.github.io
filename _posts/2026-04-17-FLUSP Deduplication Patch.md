---
title: PATCH drm/amdgpu":" Deduplicate eop_irq v11/v12 functions to use helper
date: 2026-04-17
categories: [Dev, Kernel_Dev, AMD_Drivers, Deduplication]
tags: [kernel]
---

### 1 - Introdução

Após ter acompanhado todos os tutoriais do <em>FLUSP</em>, estava pronto para ativamente contribuir com o desenvolvimento do <em>Kernel</em> Linux.

Havia muitas possibilidades diferentes de contribuir com o <em>Kernel</em> do Linux, acabei optando por contribuir com a parte de drivers da Amd, refatorando partes idênticas do código com a ajuda de um <em>Helper</em> para evitar repetição de código e melhorar a mantenabilidade do projeto.

Usando a ferramenta **Arkanjo** <em>(disponível em: https://github.com/arkanjo-tool/arkanjo)</em> foram descobertas várias duplicações de códigos, eu escolhi lidar com o caso de 100% de similaridade entre as funções  ``gfx_v11_0_eop_irq()`` e ``gfx_v12_0_eop_irq()``, dentro dos arquivos ``gfx_v11_0.c`` e ``gfx_v12_0.c`` respectivamente. Ambas estão na pasta<em> <strong>drivers/gpu/drm/amd/amdgpu/</strong></em> do projeto Linux.

### 2 - Identificação

A primeira etapa era análisar a função duplicada para descobrir sua estrutura e similaridades:

```c
static int gfx_v11_0_eop_irq(struct amdgpu_device *adev,
			     struct amdgpu_irq_src *source,
			     struct amdgpu_iv_entry *entry)
{
	u32 doorbell_offset = entry->src_data[0];
	u8 me_id, pipe_id, queue_id;
	struct amdgpu_ring *ring;
	int i;

	DRM_DEBUG("IH: CP EOP\n");

	if (adev->enable_mes && doorbell_offset) {
		struct amdgpu_userq_fence_driver *fence_drv = NULL;
		struct xarray *xa = &adev->userq_xa;
		unsigned long flags;

		xa_lock_irqsave(xa, flags);
		fence_drv = xa_load(xa, doorbell_offset);
		if (fence_drv)
			amdgpu_userq_fence_driver_process(fence_drv);
		xa_unlock_irqrestore(xa, flags);
	} else {
		me_id = (entry->ring_id & 0x0c) >> 2;
		pipe_id = (entry->ring_id & 0x03) >> 0;
		queue_id = (entry->ring_id & 0x70) >> 4;

		switch (me_id) {
		case 0:
			if (pipe_id == 0)
				amdgpu_fence_process(&adev->gfx.gfx_ring[0]);
			else
				amdgpu_fence_process(&adev->gfx.gfx_ring[1]);
			break;
		case 1:
		case 2:
			for (i = 0; i < adev->gfx.num_compute_rings; i++) {
				ring = &adev->gfx.compute_ring[i];
				/* Per-queue interrupt is supported for MEC starting from VI.
				 * The interrupt can only be enabled/disabled per pipe instead
				 * of per queue.
				 */
				if ((ring->me == me_id) &&
				    (ring->pipe == pipe_id) &&
				    (ring->queue == queue_id))
					amdgpu_fence_process(ring);
			}
			break;
		}
	}

	return 0;
}
```
A função gfx_v11_0_eop_irq é a rotina de tratamento de interrupção (Interrupt Service Routine - ISR) para o sinal EOP (End Of Pipe/Packet) das engines gráficas e de computação das GPUs AMD RDNA 3 (gfx_v11_0).

Ela é chamada pelo subsistema de interrupções do driver quando a GPU termina
de processar um pacote de comandos. O objetivo é sinalizar para o software (driver) que o trabalho foi concluído e, assim, liberar fences e acordar processos que estavam esperando.

A ferramenta **Arkanjo** apontava que esta função foi implementada na V11 e na V12 de forma idêntica, mas a primeira coisa que pensei em fazer foi verificar outras versões e ver se alguma parte delas também poderia ser reaproveitada com o Helper.

Ao comparar com o código dos arquivos que tinham uma função eop: gfx_v11_0_3.c, gfx_v12_1.c e gfx_v9_4.c, concluí que nenhum deles poderia ser utilizado com o Helper, por pequenas discrepâncias entre eles.

No início tentei transformar um switch que aparece dentro de todos esses arquivos em uma função do helper porém todos eles tem uma pequena discrepância uns cons os outros:

```c
switch (me_id) {
		case 1:
		case 2:
			for (i = 0; i < adev->gfx.num_compute_rings; i++) {
				ring = &adev->gfx.compute_ring
						[i +
						 xcc_id * adev->gfx.num_compute_rings];
				/* Per-queue interrupt is supported for MEC starting from VI.
				 * The interrupt can only be enabled/disabled per pipe instead
				 * of per queue.
				 */
				if ((ring->me == me_id) &&
				    (ring->pipe == pipe_id) &&
				    (ring->queue == queue_id))
					amdgpu_fence_process(ring);
			}
			break;
		default:
			dev_dbg(adev->dev, "Unexpected me %d in eop_irq\n", me_id);
			break;
		}
```

```c
switch (me_id) {
		case 0:
			if (pipe_id == 0)
				amdgpu_fence_process(&adev->gfx.gfx_ring[0]);
			else
				amdgpu_fence_process(&adev->gfx.gfx_ring[1]);
			break;
		case 1:
		case 2:
			for (i = 0; i < adev->gfx.num_compute_rings; i++) {
				ring = &adev->gfx.compute_ring[i];
				/* Per-queue interrupt is supported for MEC starting from VI.
				 * The interrupt can only be enabled/disabled per pipe instead
				 * of per queue.
				 */
				if ((ring->me == me_id) &&
				    (ring->pipe == pipe_id) &&
				    (ring->queue == queue_id))
					amdgpu_fence_process(ring);
			}
			break;
		}
```
Por fim, ciente de que apenas os dois arquivos (v11 e v12) tinham funções idênticas, meu próximo passo era encontrar um helper pré-estabelecido para mover esta função. Ao analisar o início do arquivo *.c* encontrei o header ``amdgpu_gfx.h`` que era exatamente o helper que eu estava procurando.

### 3 - Solução implementada
Agora para implementar esta função no helper, bastou copiar toda a função do arquivo original para o arquivo ``amdgpu_gfx.c``:

```c

int amdgpu_gfx_eop_irq(struct amdgpu_device *adev,
                            struct amdgpu_irq_src *source,
                            struct amdgpu_iv_entry *entry)
{
       u32 doorbell_offset = entry->src_data[0];
               u8 me_id, pipe_id, queue_id;
               struct amdgpu_ring *ring;
               int i;

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
                       me_id = (entry->ring_id & 0x0c) >> 2;
                       pipe_id = (entry->ring_id & 0x03) >> 0;
                       queue_id = (entry->ring_id & 0x70) >> 4;

                       switch (me_id) {
                       case 0:
                               if (pipe_id == 0)
                                       amdgpu_fence_process(&adev->gfx.gfx_ring[0]);
                               else
                                       amdgpu_fence_process(&adev->gfx.gfx_ring[1]);
                               break;
                       case 1:
                       case 2:
                               for (i = 0; i < adev->gfx.num_compute_rings; i) {
                                       ring = &adev->gfx.compute_ring[i];
                                       /* Per-queue interrupt is supported for MEC starting from VI.
                                       * The interrupt can only be enabled/disabled per pipe instead
                                       * of per queue.
                                       */
                                       if ((ring->me == me_id) &&
                                               (ring->pipe == pipe_id) &&
                                               (ring->queue == queue_id))
                                               amdgpu_fence_process(ring);
                               }
                               break;
                       }
               }

               return 0;
}
```
E adicionar o cabeçalho da função no header ``amdgpu_gfx.h``
```c
int amdgpu_gfx_eop_irq(struct amdgpu_device *adev, struct amdgpu_irq_src *source, struct amdgpu_iv_entry *entry);
```
Uma das coisas que tive que me manter atento durante esse processo é manter todas as convenções de código pré estabelecida nos arquivos, como por exemplo, se a linha chegar a 80 caracteres, obrigatóriamente, devo usar uma quebra de linha. Além disso também mantive o estilo de nomeclatura das funções no helper, usando o prefixo ``amdgpu_gfx_``. 

### 4 - Enviando o Patch
Com minha solução implantada, o que restava era enviar minha contribuição para o Kernel do linux, mas antes, precisava enviar para o Pipeline do IME-USP para realizar os testes necessários para garantir o funcionamento do código antes do envio final.

Seguindo o tutorial de envio de Patches por e-mail <em>(disponível em: https://flusp.ime.usp.br/git/sending-patches-with-git-and-a-usp-email/)</em>, enviei o meu patch de contribuição por volta das 6 horas da manhã, para melhor se adequar ao fuso-horário do mantenedor. Pouco depois, recebi a resposta do mantenedor dizendo o seguinte:

```
The separation was intentional.

> +
> +               DRM_DEBUG("IH: CP EOP\n");
> +
> +               if (adev->enable_mes && doorbell_offset) {


> +                       struct xarray *xa = &adev->userq_doorbell_xa;
> +                       struct amdgpu_usermode_queue *queue;
> +                       unsigned long flags;
> +
> +                       xa_lock_irqsave(xa, flags);
> +                       queue = xa_load(xa, doorbell_offset);
> +                       if (queue)
> +                               amdgpu_userq_fence_driver_process(queue->fence_drv);
> +                       xa_unlock_irqrestore(xa, flags);

What we could do is to move this chunk into a common function, there should be multiple copies of it in the SDMA code as well.

Regards,
Christian.
```
O retorno que ele me deu me indicou que na verdade esta duplicação de código era intencional, o que significa que minha tentativa de refatoração pode ter se encaixado em uma das seguintes categorias de desenvolvimento de kernel:

-  **Driver Forking (T1)** 
A duplicação de código às vezes é intencional. Drivers inteiros são clonados para servirem como bases de referência (baselines) independentes.

- **Readability over Deduplication (T2)**
Os mantenedores geralmente preferem código duplicado se isso melhorar a clareza e reduzir a carga cognitiva.

Isso se dá possívelmente possívelmente pois cada versão do Driver tem que ter sua independência, então para facilitar a legibilidade, considera que cada versão do driver é única e não reutiliza código de outras versões.

Os próximos passos segundo as orientações do mantenedor é tornar este bloco de código mencionado, em sua própria função, a ser implementado no helper. Que vai se tornar uma nova submissão ao Kernel, em forma de um **Patch V2**.