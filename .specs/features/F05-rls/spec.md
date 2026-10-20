# F05 — Segurança em Nível de Linha (RLS) Specification

**Prioridade:** MVP · **Depende de:** F04, F12, F15 · **Origem:** F05 do `produto.specs.md`

## Problem Statement

A ACL (F04) decide quem vê uma visão, mas não quais linhas. Um vendedor deve ver só a própria carteira e um gerente só a sua filial, em qualquer tela, filtro, drill, exportação ou API, sem depender de o construtor lembrar de filtrar. A RLS precisa ser aplicada no backend, em todo caminho de saída, com valores sempre em bind parameters.

## Goals

- [ ] Uma regra `vendedor_id = {{usuario.atributo.vendedor_id}}` restringe Ana às suas linhas em telas, exportação e API (CA-05.1).
- [ ] Usuário sem o atributo vê 0 linhas, nunca todas (CA-05.2).
- [ ] Nenhum caminho de leitura de visão ou de UDT ignora a RLS.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Relatórios agendados e alertas no contexto do destinatário (CA-05.3) | V1 (F35/F37); o motor já aceita `QueryContext.user` = destinatário |
| Drill e API pública | F27 (V1) e F39 (V2) passam pelo motor e herdam a RLS sem trabalho extra |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Onde a RLS é aplicada | `RlsQueryInterceptor` (F12) para todo o motor; mais `RlsRowFilter` nas leituras e escritas diretas de UDT (records API da F09 e Browser da F10), usando as regras da visão `SYSTEM_UDT` | RN-05.2 e RF-09.20 cobrem também os caminhos que não usam o motor | y |
| Escrita em UDT com RLS | `PATCH`/`DELETE`/`bulk` só afetam linhas visíveis ao usuário; linha invisível → 404 | Evita alterar o que não se vê | y |
| Inserção em UDT com RLS | Permitida; se a nova linha ficar invisível ao próprio usuário, a resposta 201 traz `visible=false` | Não bloquear cadastro (ex.: lançar meta de outra filial) é decisão do construtor via ACL de dados | y |
| Modo simples | Árvore de até 2 níveis: grupo (`AND`/`OR`) de condições `{ column, op, value: TOKEN | LITERAL }`, ops `EQ, NEQ, IN, NOT_IN, GT, GTE, LT, LTE` | RF-05.2 | y |
| Modo avançado | Expressão SQL booleana sobre colunas da visão, validada por JSqlParser como expressão (embrulhada em `SELECT 1 FROM v WHERE <expr>` somente-SELECT); tokens via rewriter da F15; exige `SQL_EDITOR` | RF-05.2 | y |
| Aplicabilidade | Regra aplica-se se o usuário está entre os usuários-alvo ou em algum grupo-alvo | RF-05.1 | y |
| Padrão sem regra aplicável | Visão sem regras → sem filtro; visão com ≥ 1 regra → `defaultBehavior`, padrão `DENY_ALL` | RF-05.4 | y |
| ADMIN | Filtrado como qualquer usuário, exceto com `adminBypass=true` na visão | RN-05.4 | y |
| Cache | A chave do cache da F12 inclui o SQL final com o predicado e os binds; teste explícito garante que dois usuários não compartilham resultado | Risco registrado na F12 | y |
| "Ver como" | Só ADMIN; executa com o `CurrentUser` do alvo; auditado como evento `SECURITY/RLS_PREVIEW` | RF-05.5 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Definir regras de RLS por visão ⭐ MVP

**User Story**: Como construtor, quero definir regras que limitem as linhas de uma visão por usuário ou grupo.

**Why P1**: Requisito de segurança central do produto.

**Acceptance Criteria**:
1. WHEN quem tem `MANAGE` na visão chama `POST /api/v1/views/{viewId}/rls-rules` com nome, alvos (usuários/grupos) e predicado simples válido THEN o sistema SHALL criar a regra e responder 201.
2. IF o predicado avançado (SQL) é enviado por quem não tem `SQL_EDITOR` THEN o sistema SHALL responder 403 `FLAG_REQUIRED`.
3. IF o predicado referencia coluna inexistente na visão, token desconhecido ou SQL que não é expressão booleana THEN o sistema SHALL responder 400 com `code=RLS_PREDICATE_INVALID` e o motivo.
4. WHEN uma regra é salva THEN o sistema SHALL registrar dependências `RLS_RULE → VIEW_COLUMN` para as colunas usadas.
5. WHEN `PUT /api/v1/views/{id}` define `rls.defaultBehavior` (`ALLOW_ALL`/`DENY_ALL`) e `rls.adminBypass` THEN o sistema SHALL gravar as opções.
6. WHEN uma regra é criada, alterada ou excluída THEN o sistema SHALL publicar `AclChanged` (categoria SECURITY na F06) com antes/depois.

**Independent Test**: IT de CRUD de regras com validação.

---

### P1: Aplicação em todas as consultas ⭐ MVP

**User Story**: Como gestor, quero ter certeza de que cada pessoa só vê as linhas permitidas em qualquer lugar.

**Why P1**: RN-05.1/RN-05.2.

**Acceptance Criteria**:
1. WHEN Ana (`vendedor_id=7`) consulta qualquer objeto, exportação ou `/query` sobre a visão com a regra `vendedor_id = {{usuario.atributo.vendedor_id}}` THEN o sistema SHALL devolver apenas linhas com `vendedor_id=7`.
2. WHEN várias regras se aplicam ao mesmo usuário THEN o sistema SHALL combiná-las com OU entre si e aplicar o resultado com E sobre filtros e demais predicados.
3. IF o usuário não tem o atributo usado pela regra THEN o predicado SHALL ser falso (`1=0`) e o usuário SHALL ver 0 linhas.
4. WHILE a visão tem regras e nenhuma se aplica ao usuário the sistema SHALL aplicar `defaultBehavior` (`DENY_ALL` → 0 linhas; `ALLOW_ALL` → sem filtro).
5. WHILE o usuário é ADMIN e a visão não tem `adminBypass` the sistema SHALL aplicar as regras ao ADMIN como a qualquer usuário.
6. The predicado SHALL ser montado no backend, com valores apenas em bind parameters, embrulhando a fonte (`SELECT * FROM (<visão>) v WHERE <predicado>`).
7. WHEN dois usuários com contextos de RLS diferentes fazem a mesma consulta THEN o cache SHALL devolver a cada um apenas o próprio resultado.

**Independent Test**: CA-05.1 e CA-05.2 em IT com dois usuários.

---

### P1: RLS em tabelas personalizadas ⭐ MVP

**User Story**: Como construtor, quero que as regras da visão de uma UDT valham também na grade e na API de registros.

**Why P1**: RF-09.20 e RN-05.2.

**Acceptance Criteria**:
1. WHEN a visão `SYSTEM_UDT` de uma UDT tem regras THEN `GET /udt/tables/{t}/records`, `GET /records/{id}` e `GET /browsers/{id}/data` SHALL devolver apenas as linhas visíveis ao usuário.
2. IF o usuário tenta `PATCH`, `DELETE` ou `bulk` sobre um registro invisível a ele THEN o sistema SHALL responder 404.
3. WHEN o usuário insere um registro que fica fora das suas regras THEN o sistema SHALL responder 201 com `visible=false`.

**Independent Test**: IT com Browser e records API para dois usuários.

---

### P1: Ver como outro usuário ⭐ MVP

**User Story**: Como admin, quero pré-visualizar uma visão como se fosse outro usuário para validar as regras.

**Why P1**: RF-05.5.

**Acceptance Criteria**:
1. WHEN um ADMIN chama `POST /api/v1/views/{viewId}/preview?asUser={userId}` THEN o sistema SHALL devolver até 100 linhas aplicando as regras do usuário alvo.
2. IF um não-ADMIN usa `asUser` THEN o sistema SHALL responder 403.
3. WHEN o "ver como" é usado THEN o sistema SHALL gravar o evento `SECURITY/RLS_PREVIEW` com o usuário alvo.

**Independent Test**: IT com Ana e Bruno.

---

## Edge Cases

- IF a coluna usada por uma regra fica `MISSING` THEN toda consulta à visão SHALL falhar com `RLS_COLUMN_MISSING` (falha fechada), em vez de ignorar a regra.
- WHEN uma regra tem alvos vazios THEN ela SHALL não se aplicar a ninguém.
- IF um token multivalor é usado com `EQ` no modo simples e o usuário tem 2+ valores THEN o sistema SHALL tratá-lo como `IN`.
- WHEN um grupo-alvo é excluído THEN as regras SHALL perder esse alvo (sem erro).

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| RLS-01 | P1: Definir regras (simples/avançado) | RF-05.1, RF-05.2 | Tasks | Pending |
| RLS-02 | P1: Definir regras (opções da visão) | RF-05.4, RN-05.4 | Tasks | Pending |
| RLS-03 | P1: Aplicação (combinação OU/E) | RF-05.3 | Tasks | Pending |
| RLS-04 | P1: Aplicação (token ausente) | RN-05.3, CA-05.2 | Tasks | Pending |
| RLS-05 | P1: Aplicação (backend, binds, todos os caminhos) | RN-05.1, RN-05.2, RN-05.5, CA-05.1 | Tasks | Pending |
| RLS-06 | P1: Aplicação (cache seguro) | RF-12.8 × RN-05.2 | Tasks | Pending |
| RLS-07 | P1: RLS em UDT | RF-09.20 | Tasks | Pending |
| RLS-08 | P1: Ver como | RF-05.5 | Tasks | Pending |
| RLS-09 | (V1) Contexto do destinatário em alertas/relatórios | CA-05.3 | - | Pending |

**Coverage:** 9 total, 8 mapped to tasks, 1 diferido (V1 via F35/F37).

---

## Success Criteria

- [ ] CA-05.1 e CA-05.2 automatizados; CA-05.3 coberto quando F37 for implementada.
- [ ] Teste prova que nenhum endpoint de leitura de visão/UDT devolve linha fora das regras.
