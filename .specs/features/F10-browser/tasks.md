# F10 — Browser Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F10-browser/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio (compilador de filtros) | unit | Todos os operadores × tipos; injeção | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend controller/persistência/listener | integration | Feliz + edge + erro | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Edição, erros, conflito, permissões, lote | `frontend/src/**/*.test.tsx` | `make test-frontend` |
| Ponta a ponta | e2e | CA-10.1, CA-10.2, master-detail | `frontend/e2e/*.spec.ts` | `make e2e` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-backend` ou `make test-frontend` |
| Full | Integração/e2e | `make test-backend it-backend` (+ `make e2e`) |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Backend

```
T1 → T2 → T4 → T5
T3 → T4
T2 → T6
```

### Phase 2: Frontend — grade

```
T7 → T8 → T9
T8 → T10
T8 → T11
```

### Phase 3: Frontend — formulário, lote e configuração

```
T12 → T13
T12 → T14
T15
T16
T14 → T17
```

---

## Task Breakdown

### Phase 1: Backend

#### T1: Migração do Browser

**What**: `V9__browser.sql` com `browser` e `browser_user_pref`.
**Where**: `backend/src/main/resources/db/migration/V9__browser.sql`
**Depends on**: None
**Reuses**: —
**Requirement**: BRW-01, BRW-03, BRW-10
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `BrowserSchemaIT`: segundo Browser padrão para a mesma fonte viola o índice

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): add browser schema`

---

#### T2: `BrowserService` + listener de estrutura

**What**: CRUD, registro do recurso `BROWSER` (pai = UDT), criação do padrão em `UdtSchemaChanged`, limpeza de colunas removidas e bloqueio de exclusão do padrão.
**Where**: `backend/src/main/java/io/databoard/udt/browser/BrowserService.java`
**Depends on**: T1
**Reuses**: `AccessControl.registerResource` (F04), `UdtSchemaChanged` (F09)
**Requirement**: BRW-01, BRW-10

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `BrowserServiceIT`: criar UDT → Browser padrão existe; excluir campo → some da config; excluir padrão → `DEFAULT_BROWSER`

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): manage browsers and default browser`

---

#### T3: `GridFilterCompiler`

**What**: Compila filtros por tipo e busca global em SQL com binds, validando coluna e operador.
**Where**: `backend/src/main/java/io/databoard/udt/browser/GridFilterCompiler.java`
**Depends on**: None
**Reuses**: Metadados de UDT (F09)
**Requirement**: BRW-02

**Tools**: MCP: NONE · Skill: `java-springboot`, `security-and-hardening`

**Done when**:
- [ ] Testes: cada operador × tipo válido gera SQL esperado; `contains` em INTEGER → `INVALID_FILTER` com índice; `between` com um extremo → `gte`; valor com `'; DROP` vira bind; `q` só em colunas de texto

**Tests**: unit
**Gate**: quick
**Commit**: `feat(udt): compile grid filters to safe sql`

---

#### T4: `BrowserDataService` + endpoint de dados

**What**: `GET /browsers/{id}/data` com filtros padrão + usuário + busca, paginação (máx. 500), contagem separada, `meta` com colunas e permissões efetivas; `BrowserSource` com `UdtBrowserSource`.
**Where**: `backend/src/main/java/io/databoard/udt/browser/BrowserDataService.java`
**Depends on**: T2, T3
**Reuses**: `AccessControl` (F04)
**Requirement**: BRW-02, BRW-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `BrowserDataIT`: filtros combinados com E; `VIEW` no Browser sem `DATA_READ` → 403; `meta.canInsert=false` sem `DATA_INSERT`; UDT na lixeira → 410

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): serve browser data with filters`

---

#### T5: Endpoints CRUD e preferências

**What**: `GET|POST /browsers`, `GET|PUT|DELETE /browsers/{id}`, `PUT /browsers/{id}/preferences`.
**Where**: `backend/src/main/java/io/databoard/udt/browser/BrowserController.java`
**Depends on**: T4
**Reuses**: `PageResponse` (F00)
**Requirement**: BRW-03, BRW-10

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `BrowserControllerIT`: criar recorte 201 com `MANAGE` ao criador; sem `EDIT` na UDT → 403; preferências gravadas por usuário e devolvidas no `meta`

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): expose browser and preference endpoints`

---

#### T6: Teste de contrato CA-10.1 no backend

**What**: Teste de integração dedicado: VIEWER com `DATA_READ` + `DATA_UPDATE` → `PATCH` 200, `POST` 403, `meta.canInsert=false`.
**Where**: `backend/src/test/java/io/databoard/udt/browser/BrowserPermissionsIT.java`
**Depends on**: T2
**Reuses**: Fixtures de usuário/ACL (F04)
**Requirement**: BRW-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `BrowserPermissionsIT` verde

**Tests**: integration
**Gate**: full
**Commit**: `test(udt): cover browser data permissions`

---

### Phase 2: Frontend — grade

#### T7: `useBrowserData` e tipos

**What**: Hook TanStack Query para `data` (filtros, ordenação, página, busca na query key; `keepPreviousData`) e tipos `BrowserMeta`.
**Where**: `frontend/src/features/browser/useBrowserData.ts`
**Depends on**: None
**Reuses**: `http.ts` (F00)
**Requirement**: BRW-02
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: mudar filtro gera nova chamada com `filters` serializado; erro 400 `INVALID_FILTER` exposto

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add browser data hook`

---

#### T8: `BrowserPage` (grade somente leitura)

**What**: Grade com paginação, ordenação, preferências (ocultar, largura, congelar), ícone de somente leitura e `formatValue`.
**Where**: `frontend/src/features/browser/BrowserPage.tsx`
**Depends on**: T7
**Reuses**: TanStack Table/Virtual, `formatValue` (F13/F14)
**Requirement**: BRW-02, BRW-03, BRW-04

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `frontend-design`

**Done when**:
- [ ] Testes: ordenar altera `sort`; ocultar coluna chama `PUT preferences`; FORMULA com ícone de cadeado

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add browser grid`

---

#### T9: Filtros por coluna e busca global

**What**: `ColumnFilterPopover` por tipo (texto, número, data, seleção, booleano) e `GlobalSearch` (debounce 300 ms, mínimo 2 caracteres).
**Where**: `frontend/src/features/browser/ColumnFilterPopover.tsx`
**Depends on**: T8
**Reuses**: Mantine `Popover`, `DatePickerInput`
**Requirement**: BRW-02

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: intervalo de datas gera `between`; erro `INVALID_FILTER` marca o filtro

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add browser column filters and search`

---

#### T10: Editores de célula e edição inline

**What**: Editores por tipo (inclui `ReferenceCell` assíncrono), fila de salvamento por linha, erro inline, Esc cancela.
**Where**: `frontend/src/features/browser/cells/EditableCell.tsx`
**Depends on**: T8
**Reuses**: `reference-options` (F09)
**Requirement**: BRW-04, BRW-05, BRW-06

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: confirmar envia `PATCH` com `version`; 400 mostra erro na célula; Esc não chama API; sem `canUpdate` → não editável

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add inline cell editing`

---

#### T11: `ConflictDialog`

**What**: Diálogo para `RECORD_VERSION_CONFLICT` com "Recarregar" e "Sobrescrever".
**Where**: `frontend/src/features/browser/ConflictDialog.tsx`
**Depends on**: T8
**Reuses**: —
**Requirement**: BRW-05

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: "Sobrescrever" reenvia com o `version` do registro atual; "Recarregar" descarta a edição

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): handle record version conflicts`

---

### Phase 3: Frontend — formulário, lote e configuração

#### T12: `RecordFormDrawer`

**What**: Formulário do registro com seções, inclusão rápida/novo e abas "Dados" e "Histórico".
**Where**: `frontend/src/features/browser/RecordFormDrawer.tsx`
**Depends on**: None
**Reuses**: Editores da T10, `GET /history` (F09)
**Requirement**: BRW-04, BRW-08

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: seções renderizadas na ordem configurada; aba histórico lista campo/antigo/novo; sem `canInsert` → botão "Novo" oculto

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add record form drawer with history`

---

#### T13: `InverseRelationGrid` (master-detail)

**What**: Sub-grade por relação inversa (até 200 filhos) com inclusão inline já referenciando o pai e atualização do pai após salvar.
**Where**: `frontend/src/features/browser/InverseRelationGrid.tsx`
**Depends on**: T12
**Reuses**: `EditableCell` (T10)
**Requirement**: BRW-07

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: novo filho enviado com o campo de referência = id do pai; após salvar, refaz a leitura do pai

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add master-detail sub-grids`

---

#### T14: `BulkActionsBar`

**What**: Seleção múltipla + "Excluir" (com confirmação) e "Alterar campo", via `records/bulk`, com erros por linha.
**Where**: `frontend/src/features/browser/BulkActionsBar.tsx`
**Depends on**: T12
**Reuses**: —
**Requirement**: BRW-09

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: 3 linhas → um `bulk.delete` com 3 ids; alterar campo → `bulk.update` com versões; erro mantém seleção

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add bulk actions to browser`

---

#### T15: `BrowserConfigDrawer`

**What**: Configuração de colunas, ordem, filtros padrão, tamanho de página e seções do formulário; "Salvar como novo Browser".
**Where**: `frontend/src/features/browser/BrowserConfigDrawer.tsx`
**Depends on**: None
**Reuses**: `@dnd-kit/sortable`
**Requirement**: BRW-10

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: salvar recorte envia a config completa; filtros padrão aparecem fixos (não removíveis) na grade

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add browser configuration drawer`

---

#### T16: Rota e entrada pelo designer

**What**: Rota `/dados/:browserId` e botão "Abrir dados" no designer da UDT (abre o Browser padrão).
**Where**: `frontend/src/app/routes.tsx` (modify)
**Depends on**: None
**Reuses**: `TableDesignerPage` (F09)
**Requirement**: BRW-01

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Teste: botão navega para o Browser padrão retornado pela API

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): route to browser from table designer`

---

#### T17: E2E do Browser

**What**: VIEWER com `DATA_READ`+`DATA_UPDATE` edita célula e não vê "Novo" (CA-10.1); dois contextos editam o mesmo registro e o segundo vê o conflito (CA-10.2); pedido com itens via sub-grade atualiza o rollup.
**Where**: `frontend/e2e/browser.spec.ts`
**Depends on**: T14
**Reuses**: Fixtures de UDT (F09)
**Requirement**: BRW-05, BRW-06, BRW-07

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): cover browser editing, permissions and conflicts`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T3 | None | inícios da fase 1 | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T4 | T2, T3 | T2 → T4, T3 → T4 | ✅ |
| T5 | T4 | T4 → T5 | ✅ |
| T6 | T2 | T2 → T6 | ✅ |
| T7 | None | início da fase 2 | ✅ |
| T8 | T7 | T7 → T8 | ✅ |
| T9, T10, T11 | T8 | T8 → T9/T10/T11 | ✅ |
| T12, T15, T16 | None | inícios da fase 3 | ✅ |
| T13 | T12 | T12 → T13 | ✅ |
| T14 | T12 | T12 → T14 | ✅ |
| T17 | T14 | T14 → T17 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T3 | Backend domínio | unit | unit | ✅ |
| T1, T2, T4, T5, T6 | Backend persistência/controller | integration | integration | ✅ |
| T7–T16 | Frontend | unit | unit | ✅ |
| T17 | Ponta a ponta | e2e | e2e | ✅ |
