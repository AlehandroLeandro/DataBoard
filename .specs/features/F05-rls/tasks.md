# F05 — RLS Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F05-rls/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio (validador, builder) | unit | Todos os ramos da decisão; 1:1 com ACs; edge cases | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend integração (interceptor, UDT, controller, cache) | integration | Dois usuários por cenário; todos os caminhos de leitura | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Editor de regra, opções, ver como | `frontend/src/**/*.test.tsx` | `make test-frontend` |
| Ponta a ponta | e2e | CA-05.1, CA-05.2 | `frontend/e2e/*.spec.ts` | `make e2e` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-backend` ou `make test-frontend` |
| Full | Integração/e2e | `make test-backend it-backend` (+ `make e2e`) |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Regras

```
T1 → T3
T2 → T3 → T4
```

### Phase 2: Aplicação

```
T5 → T6 → T8
T5 → T7
T6 → T9
T10
```

### Phase 3: Frontend e E2E

```
T11 → T12
T11 → T13
T12 → T14
```

---

## Task Breakdown

### Phase 1: Regras

#### T1: Migração de RLS

**What**: `V13__rls.sql` com `rls_rule`, `rls_rule_target` e opções na `semantic_view`.
**Where**: `backend/src/main/resources/db/migration/V13__rls.sql`
**Depends on**: None
**Reuses**: —
**Requirement**: RLS-01, RLS-02
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `RlsSchemaIT`: cascata ao excluir a visão remove regras e alvos; padrão `DENY_ALL`

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): add row-level security schema`

---

#### T2: `PredicateValidator`

**What**: Validação do modo simples (colunas, operadores × tipo, tokens) e avançado (tokens, expressão booleana somente-SELECT, colunas da visão).
**Where**: `backend/src/main/java/io/databoard/security/rls/PredicateValidator.java`
**Depends on**: None
**Reuses**: `TokenService` (F15), `SqlSafetyValidator` (F07)
**Requirement**: RLS-01

**Tools**: MCP: NONE · Skill: `java-springboot`, `security-and-hardening`

**Done when**:
- [ ] Testes: coluna inexistente, token desconhecido, `GT` em texto, `1; DROP TABLE x`, `regiao = 'S' OR (SELECT ... INTO)` → `RLS_PREDICATE_INVALID`; predicados válidos dos exemplos do RF-05.1 aceitos

**Tests**: unit
**Gate**: quick
**Commit**: `feat(security): validate rls predicates`

---

#### T3: `RlsRuleService`

**What**: CRUD de regras (exige `MANAGE` na visão, `SQL_EDITOR` no avançado), opções da visão, arestas `RLS_RULE → VIEW_COLUMN` e `AclChanged`.
**Where**: `backend/src/main/java/io/databoard/security/rls/RlsRuleService.java`
**Depends on**: T1, T2
**Reuses**: `DependencyRegistry` (F13), `AccessControl` (F04)
**Requirement**: RLS-01, RLS-02

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `RlsRuleServiceIT`: criar/alterar/excluir; avançado sem flag → 403; arestas registradas; evento publicado com antes/depois

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): manage rls rules per view`

---

#### T4: `RlsRuleController`

**What**: `GET|POST /views/{viewId}/rls-rules`, `PUT|DELETE /views/{viewId}/rls-rules/{ruleId}`; opções via `PUT /views/{id}`.
**Where**: `backend/src/main/java/io/databoard/security/rls/RlsRuleController.java`
**Depends on**: T3
**Reuses**: —
**Requirement**: RLS-01, RLS-02

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `RlsRuleControllerIT`: rotas felizes; sem `MANAGE` → 403; predicado inválido → 400 com motivo

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): expose rls rule endpoints`

---

### Phase 2: Aplicação

#### T5: `RlsPredicateBuilder`

**What**: Decisão completa (sem regras, bypass, aplicáveis, padrão, OU entre regras, token ausente `1=0`, multivalor `EQ`→`IN`, `RLS_COLUMN_MISSING`).
**Where**: `backend/src/main/java/io/databoard/security/rls/RlsPredicateBuilder.java`
**Depends on**: None
**Reuses**: `TokenService`, `TokenSqlRewriter` (F15)
**Requirement**: RLS-03, RLS-04, RLS-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes 1:1 com os ACs da story Aplicação: duas regras → `(A) OR (B)`; Bruno sem atributo → `1=0` (CA-05.2); nenhuma aplicável + `DENY_ALL` → FALSE; `ALLOW_ALL` → NONE; ADMIN sem bypass filtrado; com bypass → NONE; nenhum valor literal no SQL gerado

**Tests**: unit
**Gate**: quick
**Commit**: `feat(security): build rls predicates`

---

#### T6: `RlsQueryInterceptor`

**What**: Registro no motor como `QueryInterceptor`.
**Where**: `backend/src/main/java/io/databoard/security/rls/RlsQueryInterceptor.java`
**Depends on**: T5
**Reuses**: `QueryEngine` (F12)
**Requirement**: RLS-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `RlsQueryIT`: CA-05.1 em `/query` e em `QueryEngine.stream` (caminho da exportação); CA-05.2; filtros do pedido combinados com E

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): enforce rls in query engine`

---

#### T7: `RlsRowFilter` em UDT

**What**: Predicado nas leituras de `UdtRecordService` e `UdtBrowserSource`, 404 em escrita sobre linha invisível, `visible` no 201.
**Where**: `backend/src/main/java/io/databoard/security/rls/RlsRowFilter.java`
**Depends on**: T5
**Reuses**: `UdtRecordService` (F09), `UdtBrowserSource` (F10)
**Requirement**: RLS-07

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `RlsUdtIT`: Ana e Bruno veem linhas diferentes em records e Browser; `PATCH` de Bruno em registro de Ana → 404; inserção fora da regra → 201 `visible=false`

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): enforce rls on custom table reads and writes`

---

#### T8: Teste de cache com RLS

**What**: Teste dedicado: mesma consulta por dois usuários com contextos diferentes sobre visão UDT com cache → resultados distintos; mesmo contexto → acerto de cache.
**Where**: `backend/src/test/java/io/databoard/security/rls/RlsCacheIsolationIT.java`
**Depends on**: T6
**Reuses**: `ResultCache` (F12)
**Requirement**: RLS-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `RlsCacheIsolationIT` verde

**Tests**: integration
**Gate**: full
**Commit**: `test(security): prove cache isolation under rls`

---

#### T9: Teste de arquitetura de caminhos de leitura

**What**: Regra ArchUnit/Modulith proibindo uso de `DataSourceRegistry` fora de `query`, `datasource`, `udt.data`, `udt.browser`, `udt.schema` e `semantic.view` (descoberta).
**Where**: `backend/src/test/java/io/databoard/architecture/ReadPathArchitectureTest.java`
**Depends on**: T6
**Reuses**: ArchUnit
**Requirement**: RLS-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Teste verde; violação introduzida num branch de teste faz o teste falhar

**Tests**: unit
**Gate**: quick
**Commit**: `test(architecture): restrict direct datasource access`

---

#### T10: Ver como usuário

**What**: `POST /views/{id}/preview?asUser=` (ADMIN), `CurrentUserProvider.loadFor`, `bypassCache`, evento `RLS_PREVIEW`.
**Where**: `backend/src/main/java/io/databoard/semantic/view/ViewPreviewAsUser.java`
**Depends on**: None
**Reuses**: `ViewController.preview` (F12)
**Requirement**: RLS-08

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `PreviewAsUserIT`: admin vê as linhas de Ana; BUILDER com `asUser` → 403; evento gravado

**Tests**: integration
**Gate**: full
**Commit**: `feat(semantic): preview views as another user`

---

### Phase 3: Frontend e E2E

#### T11: `RlsTab`

**What**: Aba RLS no editor de visão: lista de regras, `defaultBehavior`, `adminBypass`.
**Where**: `frontend/src/features/views/rls/RlsTab.tsx`
**Depends on**: None
**Reuses**: `ViewEditorPage` (F12)
**Requirement**: RLS-01, RLS-02
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: lista regras; alterar padrão envia `PUT /views/{id}`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add rls tab to view editor`

---

#### T12: `RlsRuleEditor`

**What**: Alvos (usuários/grupos), construtor de condições com grupos E/OU e valor literal ou token (autocomplete), modo avançado com CodeMirror (oculto sem `SQL_EDITOR`).
**Where**: `frontend/src/features/views/rls/RlsRuleEditor.tsx`
**Depends on**: T11
**Reuses**: `tokenCompletion` (F15)
**Requirement**: RLS-01

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `frontend-design`

**Done when**:
- [ ] Testes: árvore enviada no formato do design; `RLS_PREDICATE_INVALID` exibido; modo avançado oculto sem flag

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add rls rule editor`

---

#### T13: `PreviewAsUser`

**What**: Seletor de usuário + grade de prévia com o resultado do "ver como" (só ADMIN).
**Where**: `frontend/src/features/views/rls/PreviewAsUser.tsx`
**Depends on**: T11
**Reuses**: `ViewPreviewPanel` (F12)
**Requirement**: RLS-08

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: oculto para não-admin; chama `preview?asUser=`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add preview as user`

---

#### T14: E2E de RLS

**What**: Admin cria regra `vendedor_id = {{usuario.atributo.vendedor_id}}` na visão de uma UDT de vendas; Ana (7) vê só as suas linhas no Browser e em `/query`; Bruno (sem atributo) vê 0.
**Where**: `frontend/e2e/rls.spec.ts`
**Depends on**: T12
**Reuses**: Fixtures de UDT e usuários
**Requirement**: RLS-04, RLS-05, RLS-07

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde (CA-05.1, CA-05.2)

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): cover row-level security for two users`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T2 | None | inícios da fase 1 | ✅ |
| T3 | T1, T2 | T1 → T3, T2 → T3 | ✅ |
| T4 | T3 | T3 → T4 | ✅ |
| T5, T10 | None | inícios da fase 2 | ✅ |
| T6 | T5 | T5 → T6 | ✅ |
| T7 | T5 | T5 → T7 | ✅ |
| T8 | T6 | T6 → T8 | ✅ |
| T9 | T6 | T6 → T9 | ✅ |
| T11 | None | início da fase 3 | ✅ |
| T12, T13 | T11 | T11 → T12/T13 | ✅ |
| T14 | T12 | T12 → T14 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T2, T5, T9 | Backend domínio / arquitetura | unit | unit | ✅ |
| T1, T3, T4, T6–T8, T10 | Backend integração | integration | integration | ✅ |
| T11–T13 | Frontend | unit | unit | ✅ |
| T14 | Ponta a ponta | e2e | e2e | ✅ |
