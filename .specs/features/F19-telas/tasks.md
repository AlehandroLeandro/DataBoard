# F19 — Telas Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F19-telas/design.md`
**Status**: Draft

> O E2E de tela completa (CA-19.1 e CA-19.2 pela UI) fica na F22 (T-final), quando existirem tipos de objeto reais. Aqui os dois CAs são cobertos por IT e testes de componente.

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio (validador, registry, coalescer) | unit | Todos os ramos; edge cases | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend persistência/controller/serviço de dados/job | integration (com analytics real) | Feliz + edge + erro; isolamento de erro | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| analytics rotas de transform | integration | Kind conhecido/desconhecido, token | `analytics/tests/api/test_*.py` | `make test-analytics` |
| Frontend | unit | Grade, frames, refresh (timers falsos), editor | `frontend/src/**/*.test.tsx` | `make test-frontend` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-backend` ou `make test-frontend` |
| Full | Integração | `make test-backend it-backend test-analytics` |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Telas no backend

```
T1 → T4
T2 → T4
T3 → T4 → T5
T4 → T6
```

### Phase 2: Pipeline de dados dos objetos

```
T7 → T10
T8 → T9 → T10 → T11
```

### Phase 3: Frontend — visualização

```
T12 → T13 → T14
T13 → T15
T16
```

### Phase 4: Frontend — edição

```
T17 → T18 → T21
T17 → T19
T17 → T20
```

---

## Task Breakdown

### Phase 1: Telas no backend

#### T1: Migração de telas

**What**: `V15__screens.sql` com `app.screen`.
**Where**: `backend/src/main/resources/db/migration/V15__screens.sql`
**Depends on**: None
**Reuses**: —
**Requirement**: SCR-01
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ScreenSchemaIT`: inserção com `definition` JSON e `version` padrão 1

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): add screens schema`

---

#### T2: Framework de tipos de objeto

**What**: Interface `ObjectTypeHandler`, `ObjectDataPlan`, `ObjectTypeHandlerRegistry` (descoberta por beans, tipo duplicado → erro no startup) e `TestObjectHandler` no escopo de teste.
**Where**: `backend/src/main/java/io/databoard/dashboard/object/ObjectTypeHandlerRegistry.java`
**Depends on**: None
**Reuses**: —
**Requirement**: SCR-02

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: tipo registrado encontrado; desconhecido → `Optional.empty`; dois handlers com mesmo tipo → falha de inicialização

**Tests**: unit
**Gate**: quick
**Commit**: `feat(dashboard): add pluggable object type framework`

---

#### T3: `ScreenDefinitionValidator`

**What**: Validação estrutural (máx. 100 objetos, ids únicos, posições na grade de 12, `refreshIntervalSec` 0 ou ≥ 60) e delegação ao handler do tipo.
**Where**: `backend/src/main/java/io/databoard/dashboard/screen/ScreenDefinitionValidator.java`
**Depends on**: None
**Reuses**: `ObjectTypeHandlerRegistry` (T2)
**Requirement**: SCR-01, SCR-03, SCR-08

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: 101 objetos → `SCREEN_TOO_MANY_OBJECTS`; id repetido → `DUPLICATE_OBJECT_ID`; `x+w > 12` → erro; `refreshIntervalSec=30` → erro; tipo desconhecido → erro em `objects[i].type`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(dashboard): validate screen definitions`

---

#### T4: `ScreenService`

**What**: Criar, obter (tokens interpolados, `ScreenOpened`), atualizar com versão e dependências por objeto, listar com `filterVisible`/categoria/busca.
**Where**: `backend/src/main/java/io/databoard/dashboard/screen/ScreenService.java`
**Depends on**: T1, T2, T3
**Reuses**: `AccessControl`, `CategoryAssignments`, `DependencyRegistry`, `TokenService`, `ScreenOpened`
**Requirement**: SCR-01, SCR-02, SCR-04, SCR-07, SCR-10

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ScreenServiceIT`: criação com `MANAGE`; update com versão errada → `SCREEN_VERSION_CONFLICT`; arestas `SCREEN_OBJECT → VIEW` criadas e removidas ao tirar o objeto; título "Vendas – {{inicio_mes}}" interpolado; `ScreenOpened` publicado no `get`; VIEWER com `EDIT` concedido → 403 ao salvar

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): implement screen service`

---

#### T5: `ScreenController`

**What**: `GET|POST /screens`, `GET|PUT|DELETE /screens/{id}`.
**Where**: `backend/src/main/java/io/databoard/dashboard/screen/ScreenController.java`
**Depends on**: T4
**Reuses**: `PageResponse` (F00)
**Requirement**: SCR-01, SCR-10

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ScreenControllerIT`: rotas felizes; listagem só com `VIEW`; erro de objeto inválido com caminho `objects[0].config.viewId`

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): expose screen endpoints`

---

#### T6: Duplicar, lixeira e purga

**What**: `POST /screens/{id}/duplicate`, exclusão lógica, `GET /screens/trash`, `POST /screens/trash/{id}/restore` e `ScreenTrashPurgeJob`.
**Where**: `backend/src/main/java/io/databoard/dashboard/screen/ScreenLifecycleService.java`
**Depends on**: T4
**Reuses**: `JobRunRecorder`
**Requirement**: SCR-09

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ScreenLifecycleIT`: cópia com novos ids e "(cópia)"; excluída some da lista; restaurar volta; 31 dias → purgada com arestas removidas; `GET` na lixeira → 410 para `MANAGE`, 404 para outros

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): duplicate, trash and restore screens`

---

### Phase 2: Pipeline de dados dos objetos

#### T7: Registry de transforms no analytics

**What**: `models.py` (`TransformRequest`), `transforms/registry.py`, router `POST /v1/transform/{kind}` e kind `passthrough`; OpenAPI atualizado.
**Where**: `analytics/src/databoard_analytics/api/v1/transform.py`
**Depends on**: None
**Reuses**: `InternalTokenMiddleware` (F00)
**Requirement**: SCR-05
**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `tests/api/test_transform.py`: `passthrough` devolve o resultado; kind desconhecido → 404 `UNKNOWN_TRANSFORM`; sem token → 401; `make contract` atualizado

**Tests**: integration
**Gate**: full
**Commit**: `feat(analytics): add transform registry endpoint`

---

#### T8: `InFlightQueryCoalescer`

**What**: Coalescência de consultas idênticas em andamento por chave (datasource + SQL final + binds).
**Where**: `backend/src/main/java/io/databoard/query/InFlightQueryCoalescer.java`
**Depends on**: None
**Reuses**: —
**Requirement**: SCR-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: 5 chamadas concorrentes com a mesma chave executam o fornecedor 1 vez; chaves diferentes executam separadamente; exceção propaga para todos os aguardando e libera a chave

**Tests**: unit
**Gate**: quick
**Commit**: `feat(query): coalesce identical in-flight queries`

---

#### T9: `QueryEngine` com coalescer e autorização via tela

**What**: Integrar o coalescer e `QueryContext.authorizedVia=SCREEN` (dispensa `VIEW` na visão quando a autorização veio da tela; RLS mantida).
**Where**: `backend/src/main/java/io/databoard/query/QueryEngine.java` (modify)
**Depends on**: T8
**Reuses**: `QueryEngine` (F12)
**Requirement**: SCR-05, SCR-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `QueryEngineScreenIT`: usuário sem `VIEW` na visão + `authorizedVia=SCREEN` → executa com RLS aplicada; sem `authorizedVia` → 403; duas consultas idênticas simultâneas → 1 execução (contador)

**Tests**: integration
**Gate**: full
**Commit**: `feat(query): authorize via screen and deduplicate queries`

---

#### T10: `ObjectDataService`

**What**: Carrega a definição salva, chama `handler.plan`, executa consultas, chama o transform no analytics e monta `{ data, meta }`, mapeando erros por objeto.
**Where**: `backend/src/main/java/io/databoard/dashboard/object/ObjectDataService.java`
**Depends on**: T7, T9
**Reuses**: `AnalyticsClient` (F00), `QueryEngine` (F12)
**Requirement**: SCR-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ObjectDataServiceIT` (analytics real, `TestObjectHandler`): resultado passa pelo `passthrough`; visão quebrada → erro com `code` do motor; visão excluída → `VIEW_DELETED`; analytics parado → `ANALYTICS_UNAVAILABLE`; objeto estático não consulta nada

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): load object data through engine and analytics`

---

#### T11: Endpoint de dados do objeto

**What**: `POST /screens/{id}/objects/{objectId}/data` com `filterValues` e `bypassCache`, exige `VIEW` na tela.
**Where**: `backend/src/main/java/io/databoard/dashboard/object/ScreenObjectDataController.java`
**Depends on**: T10
**Reuses**: —
**Requirement**: SCR-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ScreenObjectDataIT`: CA-19.1 (6 objetos, 1 com visão quebrada → 5 respostas 200 e 1 erro); sem `VIEW` na tela → 403 sem consulta executada; objeto inexistente → 404

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): expose object data endpoint`

---

### Phase 3: Frontend — visualização

#### T12: `objectRegistry`, `ObjectFrame` e `useObjectData`

**What**: Registro de tipos no front, frame com skeleton/erro/"Tentar novamente"/menu "Atualizar" e hook por objeto.
**Where**: `frontend/src/features/screens/objects/ObjectFrame.tsx`
**Depends on**: None
**Reuses**: TanStack Query
**Requirement**: SCR-05
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: carregando → skeleton; erro → mensagem só no frame; "Tentar novamente" refaz só aquela query; tipo desconhecido → placeholder

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add object frame and registry`

---

#### T13: `ScreenPage` (visualização)

**What**: Cabeçalho, rodapé, grade responsiva somente leitura com `react-grid-layout` e botão "Atualizar tela".
**Where**: `frontend/src/features/screens/ScreenPage.tsx`
**Depends on**: T12
**Reuses**: `react-grid-layout`
**Requirement**: SCR-03, SCR-04, SCR-08

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `frontend-design`

**Done when**:
- [ ] Testes: 6 objetos disparam 6 requisições em paralelo; "Atualizar tela" envia `bypassCache=true` para todos; layout `sm` derivado quando ausente

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): render screens in view mode`

---

#### T14: Atualização automática

**What**: Hook `useAutoRefresh(intervalSec)` com pausa em aba oculta e hora da última atualização.
**Where**: `frontend/src/features/screens/useAutoRefresh.ts`
**Depends on**: T13
**Reuses**: —
**Requirement**: SCR-08

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes com timers falsos: intervalo 300 s e 10 min → 2 recargas (CA-19.2); aba oculta não recarrega; ao voltar, recarrega

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): auto-refresh screens`

---

#### T15: Tela cheia

**What**: Modo apresentação com Fullscreen API, sem menus, mantendo o auto-refresh.
**Where**: `frontend/src/features/screens/FullscreenToggle.tsx`
**Depends on**: T13
**Reuses**: —
**Requirement**: SCR-10

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: chama `requestFullscreen`; layout sem menu lateral; Esc sai

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add screen fullscreen mode`

---

#### T16: `ScreensListPage` e lixeira

**What**: Lista de telas com categoria e busca, botão "Nova tela" (BUILDER), aba "Lixeira" com restaurar.
**Where**: `frontend/src/features/screens/ScreensListPage.tsx`
**Depends on**: None
**Reuses**: `CategorySelect` (F13), `<Can>` (F04)
**Requirement**: SCR-09, SCR-10

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: VIEWER não vê "Nova tela"; restaurar remove da lixeira

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add screens list and trash`

---

### Phase 4: Frontend — edição

#### T17: `ScreenEditor` (grade editável)

**What**: Modo edição com drag/resize por breakpoint, salvar com `version`, conflito 409 tratado e erros de objeto destacados.
**Where**: `frontend/src/features/screens/editor/ScreenEditor.tsx`
**Depends on**: None
**Reuses**: `ScreenPage` (T13), `react-grid-layout`
**Requirement**: SCR-01, SCR-03

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `frontend-design`

**Done when**:
- [ ] Testes: arrastar atualiza `position.lg`; salvar envia `version`; 409 mostra aviso; erro `objects[2].config` destaca o 3º objeto

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add screen editor grid`

---

#### T18: Menu de objetos e ações

**What**: `ObjectMenu` (inserir por tipo com tamanho padrão), `ObjectConfigDrawer` (editor do tipo via registry), duplicar, copiar/colar entre telas e excluir objeto.
**Where**: `frontend/src/features/screens/editor/ObjectMenu.tsx`
**Depends on**: T17
**Reuses**: `objectRegistry` (T12)
**Requirement**: SCR-01

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: inserir cria objeto com id novo e `defaultSize`; duplicar gera outro id; colar em outra tela gera id novo; excluir remove

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add object menu and actions`

---

#### T19: `HeaderFooterEditor`

**What**: Edição de título (com autocomplete de tokens), logo (upload via `/files`), "mostrar última atualização" e rodapé.
**Where**: `frontend/src/features/screens/editor/HeaderFooterEditor.tsx`
**Depends on**: T17
**Reuses**: `tokenCompletion` (F15), `POST /files` (F11)
**Requirement**: SCR-04

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: título com `{{` sugere tokens; upload de logo grava `logoFileId`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): edit screen header and footer`

---

#### T20: `ScreenSettingsDrawer`

**What**: Nome, descrição, ícone, categoria, intervalo de atualização (0 ou ≥ 60 s) e ACL.
**Where**: `frontend/src/features/screens/editor/ScreenSettingsDrawer.tsx`
**Depends on**: T17
**Reuses**: `AclEditor` (F04), `CategorySelect` (F13)
**Requirement**: SCR-01, SCR-08

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: intervalo 30 s bloqueado no formulário; ACL abre o editor de acessos

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add screen settings drawer`

---

#### T21: Rota "nova tela com objeto"

**What**: `/telas/nova-com-objeto?viewId=` cria uma tela nova em modo edição com um objeto Consulta pré-configurado para a visão (usado pela F17).
**Where**: `frontend/src/features/screens/NewScreenWithObjectRoute.tsx`
**Depends on**: T18
**Reuses**: `ScreenEditor` (T17)
**Requirement**: SCR-01

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Teste: abre o editor com um objeto `QUERY` com `viewId` da URL

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): create screen with object from view`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T2, T3 | None | inícios da fase 1 | ✅ |
| T4 | T1, T2, T3 | T1/T2/T3 → T4 | ✅ |
| T5 | T4 | T4 → T5 | ✅ |
| T6 | T4 | T4 → T6 | ✅ |
| T7, T8 | None | inícios da fase 2 | ✅ |
| T9 | T8 | T8 → T9 | ✅ |
| T10 | T7, T9 | T7 → T10, T9 → T10 | ✅ |
| T11 | T10 | T10 → T11 | ✅ |
| T12, T16 | None | inícios da fase 3 | ✅ |
| T13 | T12 | T12 → T13 | ✅ |
| T14, T15 | T13 | T13 → T14/T15 | ✅ |
| T17 | None | início da fase 4 | ✅ |
| T18, T19, T20 | T17 | T17 → T18/T19/T20 | ✅ |
| T21 | T18 | T18 → T21 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T2, T3, T8 | Backend domínio | unit | unit | ✅ |
| T1, T4–T6, T9–T11 | Backend persistência/serviço/controller | integration | integration | ✅ |
| T7 | analytics rotas | integration | integration | ✅ |
| T12–T21 | Frontend | unit | unit | ✅ |
