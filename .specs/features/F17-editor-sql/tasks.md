# F17 — Editor SQL Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F17-editor-sql/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend serviço/controller | integration | Feliz + todos os erros da spec | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Execução de seleção, salvar como visão, histórico, explorador | `frontend/src/**/*.test.tsx` | `make test-frontend` |
| Ponta a ponta | e2e | Executar e salvar como visão | `frontend/e2e/*.spec.ts` | `make e2e` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-frontend` |
| Full | Integração/e2e | `make test-backend it-backend` (+ `make e2e`) |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Backend

```
T1 → T2 → T3
T1 → T4
```

### Phase 2: Frontend e E2E

```
T5 → T6 → T9
T5 → T7
T5 → T8
```

---

## Task Breakdown

### Phase 1: Backend

#### T1: Migração do console SQL

**What**: `V14__sql_console.sql` com `sql_history` e `sql_snippet`.
**Where**: `backend/src/main/resources/db/migration/V14__sql_console.sql`
**Depends on**: None
**Reuses**: —
**Requirement**: SQL-05
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SqlConsoleSchemaIT`: inserção e índice por usuário funcionam

**Tests**: integration
**Gate**: full
**Commit**: `feat(query): add sql console schema`

---

#### T2: `SqlConsoleService`

**What**: Execução com flag/ACL, tamanho, vazio, somente-SELECT, tokens, `QueryExecutor`, mapeamento de erros e gravação de histórico (limite 50).
**Where**: `backend/src/main/java/io/databoard/query/console/SqlConsoleService.java`
**Depends on**: T1
**Reuses**: `SqlSafetyValidator` (F07), `QueryExecutor` (F12), `TokenSqlRewriter` (F15)
**Requirement**: SQL-01, SQL-02, SQL-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SqlConsoleServiceIT`: SELECT ok; `DELETE` → `ONLY_SELECT_ALLOWED`; sem flag → 403; sem `VIEW` na conexão → 403; interna lendo `app.user_account` → `PERMISSION_DENIED_SOURCE`; truncamento; timeout; erro de sintaxe → `SQL_ERROR` e histórico com erro; 51ª execução mantém 50

**Tests**: integration
**Gate**: full
**Commit**: `feat(query): execute ad hoc select statements safely`

---

#### T3: `SqlConsoleController`

**What**: `POST /api/v1/sql/execute` e `GET /api/v1/sql/history`.
**Where**: `backend/src/main/java/io/databoard/query/console/SqlConsoleController.java`
**Depends on**: T2
**Reuses**: —
**Requirement**: SQL-01, SQL-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SqlConsoleControllerIT`: resposta com `columns/rows/truncated/durationMs`; corpo > 100 KB → 413; histórico só do usuário atual

**Tests**: integration
**Gate**: full
**Commit**: `feat(query): expose sql console endpoints`

---

#### T4: Snippets

**What**: `SnippetService` + `GET|POST /sql/snippets`, `PUT|DELETE /sql/snippets/{id}` restritos ao dono.
**Where**: `backend/src/main/java/io/databoard/query/console/SnippetController.java`
**Depends on**: T1
**Reuses**: —
**Requirement**: SQL-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SnippetControllerIT`: CRUD do dono; outro usuário → 404

**Tests**: integration
**Gate**: full
**Commit**: `feat(query): manage personal sql snippets`

---

### Phase 2: Frontend e E2E

#### T5: `SqlEditorPage`

**What**: Seletor de conexão, CodeMirror SQL com autocomplete do schema e de tokens, Ctrl+Enter executa seleção ou tudo, `SqlResultGrid` com aviso de truncamento.
**Where**: `frontend/src/features/sql/SqlEditorPage.tsx`
**Depends on**: None
**Reuses**: `SchemaExplorer` (F07), `tokenCompletion` (F15)
**Requirement**: SQL-01, SQL-03
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `frontend-design`

**Done when**:
- [ ] Testes: com seleção envia só o trecho; sem seleção envia tudo; erro exibido; `truncated` mostra aviso; autocomplete sugere tabela da conexão

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add sql editor page`

---

#### T6: Salvar como visão / criar objeto

**What**: Ações "Salvar como Visão" (modal de nome) e "Criar objeto a partir deste resultado".
**Where**: `frontend/src/features/sql/SaveAsViewAction.tsx`
**Depends on**: T5
**Reuses**: `POST /views` (F12)
**Requirement**: SQL-04

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: `POST /views` com `type=SQL`, SQL e conexão; navega para o editor da visão ou para `/telas/nova-com-objeto?viewId=`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): save sql as view`

---

#### T7: Painel de histórico

**What**: Lista das últimas execuções com status, duração e clique para reabrir no editor.
**Where**: `frontend/src/features/sql/SqlHistoryPanel.tsx`
**Depends on**: T5
**Reuses**: —
**Requirement**: SQL-05

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: clicar em item carrega o SQL e a conexão

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add sql history panel`

---

#### T8: Painel de snippets e inserção pelo explorador

**What**: Salvar/abrir/excluir snippets e inserir nome qualificado ao clicar no explorador.
**Where**: `frontend/src/features/sql/SnippetsPanel.tsx`
**Depends on**: T5
**Reuses**: `SchemaExplorer.onPick` (F07)
**Requirement**: SQL-05, SQL-06

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: salvar snippet envia nome e SQL; clicar em coluna insere `schema.tabela.coluna` no cursor

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add sql snippets and schema insert`

---

#### T9: E2E do Editor SQL

**What**: Analista com `SQL_EDITOR` executa um SELECT na conexão interna sobre uma UDT, tenta `DELETE` (erro) e salva o SELECT como visão.
**Where**: `frontend/e2e/sql-editor.spec.ts`
**Depends on**: T6
**Reuses**: Fixtures de UDT e usuários
**Requirement**: SQL-01, SQL-04

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): cover sql editor execution and save as view`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1 | None | início | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T3 | T2 | T2 → T3 | ✅ |
| T4 | T1 | T1 → T4 | ✅ |
| T5 | None | início da fase 2 | ✅ |
| T6, T7, T8 | T5 | T5 → T6/T7/T8 | ✅ |
| T9 | T6 | T6 → T9 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1–T4 | Backend persistência/serviço/controller | integration | integration | ✅ |
| T5–T8 | Frontend | unit | unit | ✅ |
| T9 | Ponta a ponta | e2e | e2e | ✅ |
