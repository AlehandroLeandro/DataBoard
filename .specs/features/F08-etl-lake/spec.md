# F08 — ETL e Data Lake Interno Specification

**Prioridade:** V1 · **Depende de:** F07, F33 · **Origem:** F08 do `produto.specs.md`
**Artefatos:** só spec (V1). Design e tasks entram quando a feature for para execução.

## Problem Statement

Consultar o ERP ao vivo sobrecarrega o sistema do cliente e impede cruzar dados de bancos diferentes. Copiar dados das conexões externas para o schema `lake` do Postgres interno tira carga do ERP, permite joins entre fontes e permite referenciar esses dados nas Tabelas Personalizadas.

## Goals

- [ ] Uma carga FULL que falha no meio nunca deixa as telas sem dados (CA-08.1).
- [ ] Uma carga INCREMENTAL sem mudanças na origem grava 0 linhas (CA-08.2).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Transformações SQL pós-carga | F34 (Processos) |
| CDC/streaming (log-based) | Fora do produto; cargas são em lote |
| Escrita de volta nas origens | Plataforma só lê bancos externos (RN-07.1) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Motor de execução | Jobs Quartz da F33, lock por carga (`@DisallowConcurrentExecution` por chave) | RN-08.1 | y |
| Escrita no lake | Spring via JDBC em lote (`COPY` do Postgres quando possível) | AD-002: só o Spring acessa bancos | y |
| Swap atômico do FULL | Carrega em `lake.<slug>__staging` e faz `ALTER TABLE ... RENAME` numa transação | RF-08.3, RN-08.2 | y |
| Prévia de mapeamento | `POST /etl/jobs/preview` usa o motor de leitura da conexão (100 linhas) | RF-08.1 | y |
| Referência a lake em UDT | Entra junto: referência suave validada na gravação e sinalização de órfãos (F09 RF-09.1b, RN-09.2, CA-09.7) | Pendência registrada na F09 | y |
| Invalidação de cache | Fim de carga publica `LakeTableRefreshed`, e o `ResultCache` (F12) invalida visões dependentes | RF-12.8 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Cargas FULL e INCREMENTAL [V1]

**User Story**: Como admin, quero copiar tabelas ou consultas de uma conexão para o lake em modo completo ou incremental.

**Why P1**: Núcleo da feature.

**Acceptance Criteria**:
1. WHEN uma carga FULL termina com sucesso THEN o sistema SHALL substituir `lake.<slug>` pelos novos dados por swap atômico, sem janela de tabela vazia.
2. IF uma carga FULL falha no meio THEN o sistema SHALL manter a versão anterior da tabela intacta e registrar a execução como `FAILED`.
3. WHEN uma carga INCREMENTAL por `updated_at` roda duas vezes sem mudanças na origem THEN a segunda execução SHALL gravar 0 linhas.
4. WHEN a carga INCREMENTAL encontra registros já existentes pela chave de upsert THEN o sistema SHALL atualizá-los, sem duplicar.
5. WHEN uma coluna nova aparece na origem THEN o sistema SHALL executar `ALTER TABLE ADD COLUMN` na tabela destino; colunas removidas na origem SHALL ficar nulas nas novas linhas.
6. IF a mesma carga é disparada enquanto já executa THEN o sistema SHALL recusar a segunda execução com `ETL_ALREADY_RUNNING`.

**Independent Test**: CA-08.1 e CA-08.2 com Testcontainers (origem Postgres/MySQL).

---

### P1: Mapeamento, prévia e histórico [V1]

**User Story**: Como admin, quero mapear colunas, ver uma prévia e acompanhar o histórico das execuções.

**Why P1**: RF-08.2, RF-08.4, RF-08.5.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/etl/jobs/preview` recebe fonte e mapeamento THEN o sistema SHALL devolver as 100 primeiras linhas já mapeadas e convertidas.
2. WHEN uma execução termina THEN o sistema SHALL registrar início, fim, linhas lidas e gravadas, status e erro, visíveis em "Tarefas Executadas" (F33).
3. WHEN `POST /api/v1/etl/jobs/{id}/run` recebe `fromValue` THEN o sistema SHALL reprocessar o incremental a partir desse valor.

**Independent Test**: IT de prévia e de reprocessamento.

---

### P2: Dependências entre cargas e integração [V1]

**User Story**: Como admin, quero encadear cargas e disparar processos ao fim, e usar o lake em visões e tabelas personalizadas.

**Why P2**: RF-08.6, RF-08.7.

**Acceptance Criteria**:
1. WHEN a carga A conclui com sucesso e B depende de A THEN o sistema SHALL disparar B.
2. The tabelas do lake SHALL aparecer como fonte de visões `TABLE` (conexão interna) e como alvo de `REFERENCE` em UDT.
3. WHEN uma carga FULL remove a chave 42 referenciada por registros de UDT THEN o Browser SHALL sinalizar esses registros como "referência quebrada" (CA-09.7).

**Independent Test**: IT de encadeamento + CA-09.7.

---

## Edge Cases

- IF a conversão de tipo de uma coluna falha para uma linha THEN a carga SHALL falhar com a linha e a coluna no erro (sem gravar parcialmente no FULL).
- WHEN o usuário tenta alterar a estrutura de uma tabela do lake manualmente THEN o sistema SHALL impedir (RN-08.3: só o role da plataforma tem DDL no `lake`).
- IF a conexão de origem está fora do ar THEN a execução SHALL falhar com `DATASOURCE_UNAVAILABLE` e seguir a política de retentativa da F33.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| ETL-01 | P1: FULL com swap atômico | RF-08.1, RF-08.3, RN-08.2, CA-08.1 | - | Pending |
| ETL-02 | P1: INCREMENTAL com upsert | RF-08.1, CA-08.2 | - | Pending |
| ETL-03 | P1: Evolução de schema | RF-08.2 | - | Pending |
| ETL-04 | P1: Lock por carga | RN-08.1 | - | Pending |
| ETL-05 | P1: Prévia, histórico, reprocessamento | RF-08.4, RF-08.5 | - | Pending |
| ETL-06 | P2: Encadeamento e processos | RF-08.6 | - | Pending |
| ETL-07 | P2: Lake em visões e UDT | RF-08.7, RF-09.1b, RN-09.2, CA-09.7 | - | Pending |

**Coverage:** 7 total, 0 mapped to tasks (V1 — tasks a criar na execução).

---

## Success Criteria

- [ ] CA-08.1, CA-08.2 e CA-09.7 automatizados quando a feature for executada.
