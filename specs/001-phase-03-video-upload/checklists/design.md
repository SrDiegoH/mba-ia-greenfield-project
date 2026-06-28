# Design Checklist: Phase 03 — Upload e Processamento de Vídeos

**Purpose**: Auditoria formal de qualidade dos artefatos de design (contratos, modelo de dados, plano, research) — usada pelo autor antes de iniciar a implementação para identificar lacunas, ambiguidades e inconsistências nos requisitos escritos.
**Created**: 2026-06-27
**Feature**: [spec.md](../spec.md) | [plan.md](../plan.md) | [data-model.md](../data-model.md) | [contracts/videos-api.md](../contracts/videos-api.md)

> Este checklist é um "unit test para requisitos escritos": cada item pergunta se os requisitos estão **completos, claros, consistentes e mensuráveis** — não se a implementação funciona.

---

## Completude dos Contratos da API

- [ ] CHK001 — O contrato de `POST /videos` especifica o comportamento quando o usuário autenticado não possui canal (cenário US1 Scenario 4)? A distinção entre "sem canal" (403) e "sem autenticação" (401) está documentada? [Completeness, Contracts §1, Spec §US1 Scenario 4]
- [ ] CHK002 — A validade da URL pré-assinada (15 minutos, conforme research.md §2) está documentada no contrato de `POST /videos` como parte da resposta ou das notas do endpoint? [Completeness, Contracts §1, Research §2, Gap]
- [ ] CHK003 — O contrato de `POST /videos/:id/upload-complete` especifica o comportamento para chamadas idempotentes (endpoint chamado duas vezes para o mesmo vídeo)? [Completeness, Contracts §2, Spec §Edge Cases]
- [ ] CHK004 — O contrato de `GET /videos/:id/stream` especifica como o `Content-Type` da resposta é determinado pelo servidor (qual mime-type é servido, dado que o formato não é conhecido em tempo de design)? [Completeness, Contracts §4, Gap]
- [ ] CHK005 — O contrato de `GET /videos/:id/stream` especifica o comportamento quando o vídeo tem status `error` (além de `draft` e `processing`)? [Completeness, Contracts §4]
- [ ] CHK006 — O contrato de `GET /videos/:id/download` especifica a extensão de arquivo usada no `Content-Disposition` (e.g., `video.mp4`) — como o servidor determina a extensão se ela não está explicitamente armazenada? [Clarity, Contracts §5, Gap]
- [ ] CHK007 — O contrato de `GET /channels/:channelId/videos` especifica o comportamento quando um usuário autenticado (não-dono) tenta acessar a listagem de outro canal? [Completeness, Contracts §6, Spec §FR-017]
- [ ] CHK008 — Os parâmetros de paginação (`page`, `limit`) têm valores mínimos e máximos documentados no contrato? O comportamento com `limit=0` ou `page=0` está definido? [Completeness, Contracts §6]
- [ ] CHK009 — O contrato de `DELETE /videos/:id` especifica o comportamento quando o `storage_key` é nulo (vídeo `draft` cujo upload nunca foi feito)? A exclusão de arquivos inexistentes é tratada graciosamente? [Completeness, Contracts §8, Gap]
- [ ] CHK010 — O código de resposta `200 OK` de `POST /videos/:id/upload-complete` é adequado para indicar processamento assíncrono, ou deveria ser `202 Accepted`? A escolha está justificada nos artefatos? [Clarity, Contracts §2]
- [ ] CHK011 — São os cenários de falha de infraestrutura (MinIO indisponível, Redis indisponível) durante chamadas de API especificados em algum artefato? [Completeness, Gap]

---

## Clareza dos Contratos da API

- [x] CHK012 — O campo `thumbnail_url` retornado por `GET /videos/:id` é uma URL completa (http://...) ou uma chave de objeto? Como o servidor monta essa URL? Está especificado? [Clarity, Contracts §3, Ambiguity] → **Resolvido**: definido como URL HTTP completa gerada pelo servidor a partir de thumbnail_key.
- [x] CHK013 — O campo `processing_metadata` (jsonb) em `GET /videos/:id` tem o schema dos campos retornados especificado (e.g., resolução, codec, bitrate)? Ou é opaco para o cliente? [Clarity, Contracts §3, Data-Model §Video, Gap] → **Resolvido**: schema mínimo `{ width, height, codec, bitrate_kbps }` documentado em data-model.md e contracts.
- [ ] CHK014 — O campo `upload_key` retornado por `POST /videos` está claramente diferenciado do `id` do vídeo? A semântica (caminho no storage vs. identificador interno) está documentada para o cliente? [Clarity, Contracts §1]
- [ ] CHK015 — A definição de "OWNER" usada nos contratos (AUTH + dono do canal) está formalizada em uma seção de autenticação no próprio documento de contratos, ou apenas implícita? [Clarity, Contracts §Auth]
- [ ] CHK016 — O contrato de `PATCH /videos/:id` especifica se todos os campos do body são opcionais (PATCH semântico) ou obrigatórios? A ausência do campo `title` resulta em 400 ou é ignorada? [Clarity, Contracts §7]

---

## Completude do Modelo de Dados

- [x] CHK017 — O campo `storage_key` na entidade Video é declarado como nullable no data-model.md, mas o contrato de `POST /videos` retorna `upload_key` — o que indica que a chave é pré-determinada pela API no momento do pré-registro. Essa inconsistência (nullable vs. preenchido imediatamente) está resolvida? [Conflict, Data-Model §storage_key, Contracts §1] → **Resolvido**: storage_key agora é NOT NULL, preenchido no pré-registro.
- [ ] CHK018 — O schema esperado de `processing_metadata` (jsonb) — quais campos o worker deve persistir e em qual formato — está especificado no data-model.md ou em algum artefato? [Completeness, Data-Model §Video, Gap]
- [x] CHK019 — O campo `error_cause` tem comprimento máximo definido? É um campo interno (não exposto ao usuário final) ou retornado em `GET /videos/:id`? [Clarity, Data-Model §Video, Ambiguity] → **Resolvido**: varchar(2048), campo interno, não exposto ao usuário final.
- [ ] CHK020 — A convenção de nomenclatura dos objetos no storage (`videos/{id}/original.mp4`, `videos/{id}/thumbnail.jpg`) está especificada como requisito de forma que o worker e a API usem a mesma convenção? [Completeness, Data-Model §Video, Gap]
- [ ] CHK021 — A duração em segundos é armazenada como `integer` — é especificado o arredondamento para vídeos com duração fracionada (e.g., 90.5s → 90 ou 91)? [Clarity, Data-Model §duration_seconds]
- [x] CHK022 — Os índices planejados (`idx_videos_channel_id_created_at`) cobrem as queries de paginação com `ORDER BY created_at DESC` e `WHERE channel_id = ?`? A ordem dos campos no índice composto está especificada? [Completeness, Data-Model §Índices] → **Resolvido**: índice declarado como `(channel_id, created_at DESC)` com nota sobre ordem relevante.
- [ ] CHK023 — A constraint de FK `channel_id → channels.id` com `ON DELETE RESTRICT` é consistente com a possibilidade de um criador ter vídeos e querer excluir o canal? Existe restrição de negócio para esse cenário? [Consistency, Data-Model §Constraints, Gap]

---

## Clareza da Máquina de Estados

- [ ] CHK024 — A transição `draft → processing` ocorre via `POST /videos/:id/upload-complete`. O que acontece se essa chamada for feita quando o status já é `processing` ou `ready`? A máquina de estados trata esse caso? [Completeness, Data-Model §Máquina de Estados, Spec §Edge Cases]
- [ ] CHK025 — O cancelamento por DELETE está representado como transição válida a partir de todos os estados (`draft`, `processing`, `ready`, `error`)? A máquina de estados e o contrato de DELETE estão alinhados? [Consistency, Data-Model §Máquina de Estados, Contracts §8]
- [ ] CHK026 — A regra de idempotência do worker ("verificar se o status ainda é `processing` antes de iniciar") está documentada como requisito funcional (FR) ou apenas como nota de implementação no data-model? [Completeness, Data-Model §Idempotência, Spec §Edge Cases]
- [ ] CHK027 — As transições proibidas da máquina de estados (`ready → qualquer`, `error → qualquer`) estão listadas de forma completa? O que acontece se uma chamada externa tentar forçar uma transição proibida? [Completeness, Data-Model §Máquina de Estados]

---

## Consistência de Autorização

- [ ] CHK028 — A distinção entre `401 Unauthorized` e `403 Forbidden` está aplicada consistentemente em todos os 8 endpoints? A regra (401 = sem token, 403 = token válido mas sem permissão) está documentada? [Consistency, Contracts §Auth]
- [ ] CHK029 — FR-017 proíbe edição/exclusão de vídeos de outros canais → os endpoints `PATCH /videos/:id` e `DELETE /videos/:id` têm 403 documentado. A listagem `GET /channels/:channelId/videos` também proíbe acesso não-dono? [Consistency, Spec §FR-017, Contracts §6]
- [ ] CHK030 — US1 Scenario 4 menciona "usuário autenticado sem canal" → o que define um canal como ativo ou necessário para upload? A condição de "canal necessário" está especificada na entidade Channel? [Clarity, Spec §US1 Scenario 4, Ambiguity]
- [ ] CHK031 — O endpoint `GET /videos/:id/download` requer qualquer usuário autenticado (AUTH) — há restrição de que o usuário deve ser dono do canal? Ou qualquer usuário autenticado pode baixar vídeos `ready` de qualquer canal? [Clarity, Spec §FR-010, Contracts §5]
- [ ] CHK032 — O cenário de race condition de autorização (usuário cria token, canal é deletado, usuário tenta upload) está documentado como edge case ou assumption? [Completeness, Spec §Assumptions, Gap]

---

## Requisitos do Worker e Processamento Assíncrono

- [ ] CHK033 — O intervalo de 5 segundos entre tentativas (research.md §1) está registrado como requisito funcional em algum artefato, ou permanece apenas como decisão de configuração de pesquisa? [Completeness, Spec §FR-008, Research §1]
- [ ] CHK034 — O comportamento de cancelamento de job BullMQ para DELETE de vídeo em `processing` — "cancela a tarefa se ainda possível" — especifica o que acontece quando o job já está sendo executado pelo worker (não apenas enfileirado)? [Clarity, Spec §US6 Scenario 4, Spec §Edge Cases]
- [ ] CHK035 — O destino dos arquivos de vídeo no storage após falha definitiva (status `error`) está especificado? O arquivo bruto enviado pelo cliente é mantido ou removido? [Completeness, Spec §FR-008, Gap]
- [ ] CHK036 — O timeout máximo de um job de processamento no BullMQ está especificado? O que acontece se o processamento durar mais do que o lock timeout do BullMQ? [Completeness, Gap]
- [ ] CHK037 — O cenário em que o arquivo de vídeo foi removido do MinIO entre o upload e o início do processamento pelo worker está coberto como falha tratável? [Completeness, Spec §Edge Cases, Gap]
- [ ] CHK038 — O que acontece quando o worker gera a thumbnail com sucesso mas falha ao fazer upload dessa thumbnail para o MinIO? Esse sub-cenário está coberto como erro tratável dentro das 3 tentativas? [Completeness, Spec §Edge Cases]
- [ ] CHK039 — A mensagem de fila `VideoProcessingJob` inclui `bucketName` como campo explícito. A justificativa para não obtê-lo da configuração está documentada? [Clarity, Data-Model §Mensagem de Fila]

---

## Requisitos Não-Funcionais e Infraestrutura

- [ ] CHK040 — SC-001 define "sem timeout" para uploads de 10 GB, mas não quantifica tempo máximo aceitável de processamento do vídeo pelo worker — há uma métrica de SLA de processamento definida? [Measurability, Spec §SC-001, Gap]
- [ ] CHK041 — SC-004 menciona streaming "com suporte a range requests" mas não quantifica latência de início de reprodução — essa métrica está intencionalmente ausente ou é uma lacuna? [Measurability, Spec §SC-004]
- [ ] CHK042 — O criador do bucket MinIO está especificado — a aplicação cria o bucket automaticamente no startup, ou ele deve ser pré-criado? Há um requisito de bootstrap documentado? [Completeness, Research §2, Gap]
- [ ] CHK043 — As variáveis de ambiente necessárias para os novos serviços Docker (Redis host/port, MinIO endpoint/credenciais/bucket) estão listadas explicitamente em algum artefato de configuração? [Completeness, Plan §Source Code, Spec §FR-012, Gap]
- [ ] CHK044 — SC-006 exige "um único comando de inicialização" — os health checks dos novos serviços (Redis, MinIO, worker) estão especificados para garantir ordem de startup correta? [Completeness, Spec §SC-006, Gap]
- [ ] CHK045 — O espaço em disco temporário necessário no container do worker para processar um vídeo de 10 GB com FFmpeg está estimado ou mencionado como requisito de infraestrutura? [Completeness, Gap]
- [ ] CHK046 — O `Dockerfile.worker` está planejado com FFmpeg — há requisito de versão mínima do FFmpeg ou apenas "qualquer versão disponível no base image"? [Clarity, Plan §Source Code]

---

## Cobertura de Edge Cases e Fluxos de Falha

- [ ] CHK047 — O edge case de URL pré-assinada expirada (cliente demora mais de 15 min para iniciar o upload) está documentado? O que o cliente deve fazer nesse cenário? [Completeness, Research §2, Gap]
- [ ] CHK048 — O edge case de MinIO indisponível no momento de `POST /videos` (geração da URL pré-assinada) está coberto como falha tratável com resposta de erro? [Completeness, Gap]
- [x] CHK049 — O edge case de Redis/fila indisponível no momento de `POST /videos/:id/upload-complete` está especificado? O status do vídeo muda para `processing` antes ou depois do enfileiramento? [Completeness, Spec §FR-004, Gap] → **Resolvido**: enfileiramento ocorre primeiro; falha retorna 500 e status permanece `draft`. Documentado em data-model.md e contracts §2.
- [ ] CHK050 — A política de out-of-scope para limpeza de drafts abandonados (spec §Edge Cases item 1) está documentada com clareza suficiente para que o desenvolvedor saiba que NÃO deve implementar esse cleanup? [Clarity, Spec §Edge Cases, Spec §Assumptions]
- [ ] CHK051 — O edge case de arquivo vazio ou corrompido especifica em qual etapa do worker a detecção ocorre (ffprobe, ffmpeg, ou upload) para que o teste de integração possa simulá-la corretamente? [Clarity, Spec §Edge Cases]
- [ ] CHK052 — O comportamento de DELETE de vídeo com `thumbnail_key` nulo (processamento falhou antes de gerar thumbnail) está especificado — a exclusão de arquivos inexistentes no storage é tolerada? [Completeness, Contracts §8, Gap]

---

## Consistência Entre Artefatos

- [ ] CHK053 — O tamanho de página padrão definido em research.md (20 itens) está refletido no contrato de `GET /channels/:channelId/videos` como valor padrão explícito do parâmetro `limit`? [Consistency, Contracts §6, Research §6]
- [ ] CHK054 — O requisito FR-013 (migration versionada) está alinhado entre spec.md, data-model.md (seção Migration) e plan.md (Source Code)? [Consistency, Spec §FR-013, Data-Model §Migration, Plan §Source Code]
- [ ] CHK055 — O limite de 255 caracteres para `title` está consistente entre data-model.md (`varchar(255)`) e contracts/videos-api.md (`1-255 chars` no request body)? [Consistency, Data-Model §title, Contracts §1 e §7]
- [x] CHK056 — A decisão de usar o UUID `id` como identificador de URL (research.md §5) elimina a ambiguidade do spec que fala em "identificador único da URL" como conceito independente do PK? O spec foi atualizado para refletir essa decisão? [Consistency, Spec §US5, Research §5] → **Resolvido**: adicionada assumption em spec.md: UUID `id` É o identificador da URL; não existe campo slug separado.
- [ ] CHK057 — FR-012 exige que storage, fila e worker "subam com a stack existente" — o plan.md lista os novos serviços Docker mas os `env vars` necessários estão explicitados em algum artefato (e.g., `compose.yaml` documentado ou `.env.example`)? [Completeness, Spec §FR-012, Plan §Source Code, Gap]
- [x] CHK058 — A propriedade `storage_key` na entidade Video e o campo `upload_key` retornado no contrato de `POST /videos` referem-se ao mesmo valor? A terminologia está unificada entre os artefatos? [Consistency, Data-Model §storage_key, Contracts §1, Ambiguity] → **Resolvido**: campo renomeado para `storage_key` no contrato; terminologia unificada.

---

## Notes

- Itens marcados `[Gap]` indicam que o requisito está ausente nos artefatos atuais — decisão de preenchê-los ou aceitar o risco fica com o autor.
- Itens marcados `[Ambiguity]` indicam texto que pode ser interpretado de múltiplas formas durante implementação.
- Itens marcados `[Conflict]` indicam contradições entre dois ou mais artefatos — devem ser resolvidos antes de implementar.
- Itens marcados `[Consistency]` indicam que múltiplos artefatos cobrem o mesmo tópico — verificar alinhamento.
- Marcar como concluído: `[x]`. Adicionar observações inline quando relevante.
