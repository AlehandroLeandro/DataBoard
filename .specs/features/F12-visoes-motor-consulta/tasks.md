# F12 — Visões e Motor de Consulta Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F12-visoes-motor-consulta/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio (planner, builder, merge, compiladores) | unit | Todos os ramos; 1:1 com ACs; injeção; 3 dialetos | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend persistência/controller/executor/cache/listeners | integration (Postgres interno + Postgres/MySQL/SQL Server externos) | Feliz + edge + erro | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Editor de visão, colunas, calculadas, impacto | `frontend/src/**/*.test.tsx` | `make test-frontend` |
| Ponta a ponta | e2e | Criar visão e consultar; visão de UDT disponível | `frontend/e2e/*.spec.ts` | `make e2e` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-backend` ou `make test-frontend` |
| Full | Integração/e2e | `make test-backend it-backend` (+ `make e2e`) |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Visões e colunas

```
T1 → T4
T2 → T4
T3 → T4 → T5
T4 → T6
```

### Phase 2: Colunas calculadas

```
T7 → T8 → T9
```

### Phase 3: Motor de consulta

```
T10 → T11 → T13
T12 → T13 → T14
T13 → T15
T13 → T16
```

### Phase 4: Frontend e E2E

```
T17 → T18 → T19
T18 → T20
T18 → T21
T21 → T22
```

---

## Task Breakdown

### Phase 1: Visões e colunas

#### T1: Migração semântica

**What**: `V12__semantic.sql` com `semantic_view` e `view_column`.
**Where**: `backend/src/main/resources/db/migration/V12__semantic.sql`
**Depends on**: None
**Reuses**: —
**Requirement**: VIEW-01, VIEW-05
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SemanticSchemaIT`: duas visões `SYSTEM_UDT` para a mesma UDT violam o índice

**Tests**: integration
**Gate**: full
**Commit**: `feat(semantic): add views and columns schema`

---

#### T2: `ColumnDiscovery`

**What**: Descoberta por `WHERE 1=0`, mapeamento JDBC → tipos da plataforma, `typeFallback`.
**Where**: `backend/src/main/java/io/databoard/semantic/view/ColumnDiscovery.java`
**Depends on**: None
**Reuses**: `DataSourceRegistry`, `SqlDialect` (F07)
**Requirement**: VIEW-02

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ColumnDiscoveryIT` nos 3 dialetos: `numeric` → DECIMAL, `int` → INTEGER, `date` → DATE, `timestamp` → DATETIME, `bit`/`boolean` → BOOLEAN, `json` → TEXT com fallback; SQL inválido → `VIEW_SOURCE_ERROR`

**Tests**: integration
**Gate**: full
**Commit**: `feat(semantic): discover view columns from source metadata`

---

#### T3: `ColumnMerge` e inferência de papéis

**What**: Mescla colunas existentes × descobertas (novas, `MISSING`, preservação de metadados) e inferência de papel/agregação.
**Where**: `backend/src/main/java/io/databoard/semantic/view/ColumnMerge.java`
**Depends on**: None
**Reuses**: —
**Requirement**: VIEW-02, VIEW-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: coluna nova numérica → MEASURE/SUM; `cliente_id` numérica → DIMENSION; coluna sumida → `MISSING` com metadado preservado; rótulo editado preservado após nova descoberta

**Tests**: unit
**Gate**: quick
**Commit**: `feat(semantic): merge discovered columns preserving metadata`

---

#### T4: `ViewService` (TABLE e SQL)

**What**: Criar/atualizar/excluir visões TABLE e SQL, validação somente-SELECT, flag `SQL_EDITOR`, `impact` no salvamento, `VIEW_IN_USE`, registro de recurso/categoria e arestas de dependência.
**Where**: `backend/src/main/java/io/databoard/semantic/view/ViewService.java`
**Depends on**: T1, T2, T3
**Reuses**: `SqlSafetyValidator` (F07), `AccessControl` (F04), `DependencyRegistry`, `CategoryAssignments` (F13)
**Requirement**: VIEW-01, VIEW-02, VIEW-03, VIEW-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ViewServiceIT`: TABLE e SQL criadas; `DELETE FROM x` → `ONLY_SELECT_ALLOWED` (CA-07.2); SQL sem flag → 403; remover coluna usada por 2 dependentes fictícios → `impact` com os 2 (CA-12.2); excluir com dependentes → `VIEW_IN_USE`; máscara registrada como dependência

**Tests**: integration
**Gate**: full
**Commit**: `feat(semantic): implement table and sql views`

---

#### T5: `ViewController` com prévia, refresh e impacto

**What**: `GET|POST /views`, `GET|PUT|DELETE /views/{id}`, `POST /views/{id}/refresh-columns`, `POST /views/{id}/preview` (100 linhas), `GET /views/{id}/impact`.
**Where**: `backend/src/main/java/io/databoard/semantic/view/ViewController.java`
**Depends on**: T4
**Reuses**: `PageResponse` (F00)
**Requirement**: VIEW-01, VIEW-11

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ViewControllerIT`: rotas felizes; prévia limitada a 100; listagem filtrada por `VIEW`; impacto por coluna

**Tests**: integration
**Gate**: full
**Commit**: `feat(semantic): expose view endpoints with preview and impact`

---

#### T6: `SystemUdtViewSync`

**What**: Criação e sincronização da visão `SYSTEM_UDT` em `UdtSchemaChanged` (após commit), recurso pai = UDT, `SYSTEM_VIEW_READONLY` e backfill no startup.
**Where**: `backend/src/main/java/io/databoard/semantic/system/SystemUdtViewSync.java`
**Depends on**: T4
**Reuses**: `UdtSchemaChanged`, metadados de UDT (F09)
**Requirement**: VIEW-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SystemUdtViewIT`: criar UDT → visão existe com rótulos e derivados (CA-09.6); novo campo → coluna adicionada mantendo rótulos editados; excluir a visão → 409; ACL herdada da UDT

**Tests**: integration
**Gate**: full
**Commit**: `feat(semantic): keep system views in sync with custom tables`

---

### Phase 2: Colunas calculadas

#### T7: Mapeamento de funções OBExpr por dialeto

**What**: `SqlCompiler` recebe `SqlDialect`; `FunctionMapping` para Postgres, MySQL e SQL Server (datas, texto, concatenação, divisão segura) e funções de agregação.
**Where**: `backend/src/main/java/io/databoard/udt/obexpr/FunctionMapping.java`
**Depends on**: None
**Reuses**: `SqlCompiler` (F09)
**Requirement**: VIEW-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Teste de tabela função × dialeto → SQL esperado; testes da F09 continuam verdes (Postgres inalterado)

**Tests**: unit
**Gate**: quick
**Commit**: `feat(udt): compile obexpr for mysql and sql server`

---

#### T8: `CalculatedColumnCompiler`

**What**: Escopo de colunas da visão, regra agregação × papel (`AGGREGATE_IN_DIMENSION`), expansão de calculadas aninhadas e ciclo (`VIEW_CIRCULAR_DEPENDENCY`).
**Where**: `backend/src/main/java/io/databoard/semantic/view/CalculatedColumnCompiler.java`
**Depends on**: T7
**Reuses**: `ObExprParser`, `TypeChecker` (F09)
**Requirement**: VIEW-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: `margem = SUM({receita}) - SUM({custo})` como medida; `FORMAT_DATE({data},'yyyy-MM')` como dimensão; `SUM` em dimensão → erro; ciclo entre duas calculadas → erro

**Tests**: unit
**Gate**: quick
**Commit**: `feat(semantic): compile calculated view columns`

---

#### T9: Calculadas no salvamento da visão

**What**: `ViewService` aceita colunas calculadas (OBEXPR; SQL com `SQL_EDITOR`), valida executando `WHERE 1=0` com a expressão e registra arestas.
**Where**: `backend/src/main/java/io/databoard/semantic/view/CalculatedColumnHandler.java`
**Depends on**: T8
**Reuses**: `ViewService` (T4), `ColumnDiscovery` (T2)
**Requirement**: VIEW-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `CalculatedColumnsIT` em Postgres e MySQL: as duas calculadas do RF-12.4 salvas e consultáveis; modo SQL sem flag → 403

**Tests**: integration
**Gate**: full
**Commit**: `feat(semantic): save calculated columns on views`

---

### Phase 3: Motor de consulta

#### T10: `QueryPlanner`

**What**: Validação do pedido (colunas visíveis, `MISSING`, operador × tipo, duplicatas, `dateGrain` só em datas) → `QueryPlan`.
**Where**: `backend/src/main/java/io/databoard/query/QueryPlanner.java`
**Depends on**: None
**Reuses**: Metadados de visão (T4)
**Requirement**: VIEW-08

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: coluna oculta → `VIEW_COLUMN_INVALID`; `MISSING` → `VIEW_COLUMN_MISSING`; `CONTAINS` em número → erro; dimensão duplicada → `DUPLICATE_DIMENSION`; `dateGrain` em texto → erro

**Tests**: unit
**Gate**: quick
**Commit**: `feat(query): validate query requests against view metadata`

---

#### T11: `SqlQueryBuilder`

**What**: SQL por dialeto com `GROUP BY`, agregações, calculadas, filtros (13 operadores) em binds, predicados extras, `ORDER BY`, `limit+1`, consulta de totais; `IN []` → `1=0`.
**Where**: `backend/src/main/java/io/databoard/query/SqlQueryBuilder.java`
**Depends on**: T10
**Reuses**: `SqlDialect` (F07), `CalculatedColumnCompiler` (T8)
**Requirement**: VIEW-07, VIEW-08, VIEW-09

**Tools**: MCP: NONE · Skill: `java-springboot`, `security-and-hardening`

**Done when**:
- [ ] Testes de snapshot nos 3 dialetos: CA-12.1 (`GROUP BY` regiao + `SUM(` valor); `dateGrain MONTH`; totais sem `GROUP BY`; valor `"x' OR 1=1 --"` vira bind; SQL Server usa `OFFSET/FETCH`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(query): build dialect-aware aggregate sql`

---

#### T12: `QueryExecutor`

**What**: Execução com timeout de `limits`, leitura tipada, detecção de truncamento e mapeamento de erros (`QUERY_TIMEOUT`, `DATASOURCE_UNAVAILABLE`).
**Where**: `backend/src/main/java/io/databoard/query/QueryExecutor.java`
**Depends on**: None
**Reuses**: `DataSourceRegistry` (F07), `LimitsProvider` (F02)
**Requirement**: VIEW-09

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `QueryExecutorIT`: `pg_sleep(3)` com timeout 1 s → `QUERY_TIMEOUT`; limite 1.000 com 5.000 linhas → 1.000 + `truncated=true` (CA-02.2); container parado → `DATASOURCE_UNAVAILABLE`

**Tests**: integration
**Gate**: full
**Commit**: `feat(query): execute queries with timeout and truncation`

---

#### T13: `QueryEngine` com interceptors e parâmetros

**What**: Orquestra planner → resolvers de parâmetro → interceptors → builder → executor → totais; verificação de `VIEW`; `stream` para exportação.
**Where**: `backend/src/main/java/io/databoard/query/QueryEngine.java`
**Depends on**: T11, T12
**Reuses**: `AccessControl` (F04)
**Requirement**: VIEW-08, VIEW-09, VIEW-10

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `QueryEngineIT`: interceptor de teste adiciona predicado e o resultado é filtrado; resolver de teste fornece parâmetro; sem `VIEW` → 403; sem dimensões → uma linha; `totals` corretos para `AVG` (não média de médias)

**Tests**: integration
**Gate**: full
**Commit**: `feat(query): orchestrate query engine with extension points`

---

#### T14: `QueryController`

**What**: `POST /api/v1/query`.
**Where**: `backend/src/main/java/io/databoard/query/QueryController.java`
**Depends on**: T13
**Reuses**: —
**Requirement**: VIEW-08

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `QueryControllerIT`: resposta `{columns,rows,totals,truncated,durationMs}` com `maskId` das colunas; pedido inválido → 400 com código

**Tests**: integration
**Gate**: full
**Commit**: `feat(query): expose query endpoint`

---

#### T15: `ResultCache` com invalidação

**What**: Cache Caffeine (peso, limite de 5 MB por resultado), chave SHA-256, índice por visão, TTL por visão, invalidação por `UdtRecordChanged`/`UdtSchemaChanged`.
**Where**: `backend/src/main/java/io/databoard/query/ResultCache.java`
**Depends on**: T13
**Reuses**: Eventos da F09/F06
**Requirement**: VIEW-12

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ResultCacheIT` (contador de consultas): segunda consulta idêntica sobre visão UDT não consulta; após `PATCH` de registro, consulta de novo; visão LIVE sem TTL nunca usa cache; binds diferentes → entradas diferentes

**Tests**: integration
**Gate**: full
**Commit**: `feat(query): cache results with event-based invalidation`

---

#### T16: `ViewBrowserSource`

**What**: Fonte VIEW para o Browser (F10), somente leitura, filtros da grade traduzidos para filtros do motor.
**Where**: `backend/src/main/java/io/databoard/semantic/browser/ViewBrowserSource.java`
**Depends on**: T13
**Reuses**: `BrowserSource` (F10)
**Requirement**: VIEW-13

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ViewBrowserIT`: Browser sobre visão SQL pagina e filtra; `meta.canInsert/Update/Delete=false`

**Tests**: integration
**Gate**: full
**Commit**: `feat(semantic): add read-only browser source for views`

---

### Phase 4: Frontend e E2E

#### T17: `ViewsListPage`

**What**: Lista de visões com tipo, conexão, categoria e busca; botão "Nova visão".
**Where**: `frontend/src/features/views/ViewsListPage.tsx`
**Depends on**: None
**Reuses**: `CategorySelect` (F13), `<Can>` (F04)
**Requirement**: VIEW-01
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: visões `SYSTEM_UDT` marcadas "Sistema"; filtro por tipo

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add views list page`

---

#### T18: `ViewEditorPage` — aba Fonte

**What**: Editor com escolha TABLE (conexão + `SchemaExplorer`) ou SQL (CodeMirror SQL, só com `SQL_EDITOR`), salvar e exibir erros do banco.
**Where**: `frontend/src/features/views/ViewEditorPage.tsx`
**Depends on**: T17
**Reuses**: `SchemaExplorer` (F07), CodeMirror
**Requirement**: VIEW-01, VIEW-02

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `frontend-design`

**Done when**:
- [ ] Testes: sem `SQL_EDITOR`, opção SQL oculta; `VIEW_SOURCE_ERROR` exibe a mensagem; `SYSTEM_UDT` com fonte bloqueada

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add view editor source tab`

---

#### T19: Grade de colunas

**What**: Edição de rótulo, descrição, papel, agregação, máscara (`useMasks`), visibilidade; colunas `MISSING` destacadas.
**Where**: `frontend/src/features/views/ColumnsGrid.tsx`
**Depends on**: T18
**Reuses**: `useMasks` (F13/F14)
**Requirement**: VIEW-05

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: mudar papel para DIMENSION força agregação `NONE`; `MISSING` com aviso

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add view columns grid`

---

#### T20: `CalculatedColumnDialog`

**What**: Diálogo de coluna calculada com `FormulaEditor` em escopo de visão (funções de agregação liberadas só para medida) e modo SQL com flag.
**Where**: `frontend/src/features/views/CalculatedColumnDialog.tsx`
**Depends on**: T18
**Reuses**: `FormulaEditor` (F09)
**Requirement**: VIEW-06

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: `AGGREGATE_IN_DIMENSION` exibido; autocomplete lista colunas da visão

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add calculated column dialog`

---

#### T21: Prévia, impacto e alerta de impacto

**What**: Painel de prévia (100 linhas formatadas), painel de impacto por coluna e `ImpactAlert` após salvar com `impact` não vazio.
**Where**: `frontend/src/features/views/ViewPreviewPanel.tsx`
**Depends on**: T18
**Reuses**: `formatValue` (F13/F14)
**Requirement**: VIEW-02, VIEW-11

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: `impact` com 2 objetos mostra alerta listando ambos; prévia aplica máscaras

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add view preview and impact panels`

---

#### T22: E2E de visões

**What**: Criar visão SQL sobre a conexão interna (lendo a view de uma UDT), editar rótulo e máscara, pré-visualizar; confirmar que a visão `SYSTEM_UDT` de uma UDT recém-criada aparece na lista.
**Where**: `frontend/e2e/views.spec.ts`
**Depends on**: T21
**Reuses**: Fixtures de UDT (F09)
**Requirement**: VIEW-01, VIEW-04, VIEW-11

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): cover view creation and system views`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T2, T3 | None | inícios da fase 1 | ✅ |
| T4 | T1, T2, T3 | T1/T2/T3 → T4 | ✅ |
| T5 | T4 | T4 → T5 | ✅ |
| T6 | T4 | T4 → T6 | ✅ |
| T7 | None | início da fase 2 | ✅ |
| T8 | T7 | T7 → T8 | ✅ |
| T9 | T8 | T8 → T9 | ✅ |
| T10, T12 | None | inícios da fase 3 | ✅ |
| T11 | T10 | T10 → T11 | ✅ |
| T13 | T11, T12 | T11 → T13, T12 → T13 | ✅ |
| T14, T15, T16 | T13 | T13 → T14/T15/T16 | ✅ |
| T17 | None | início da fase 4 | ✅ |
| T18 | T17 | T17 → T18 | ✅ |
| T19, T20, T21 | T18 | T18 → T19/T20/T21 | ✅ |
| T22 | T21 | T21 → T22 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T3, T7, T8, T10, T11 | Backend domínio | unit | unit | ✅ |
| T1, T2, T4–T6, T9, T12–T16 | Backend persistência/controller/executor/cache | integration | integration | ✅ |
| T17–T21 | Frontend | unit | unit | ✅ |
| T22 | Ponta a ponta | e2e | e2e | ✅ |
