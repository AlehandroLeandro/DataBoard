# F28 — Filtros Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F28-filtros/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio (applier, datas relativas, param resolver) | unit | Todos os ramos; 1:1 com ACs | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend integração | integration | Aplicação real nos objetos, valores distintos com RLS, estado | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Precedência, URL, controles, indicador, editor | `frontend/src/**/*.test.ts(x)` | `make test-frontend` |
| Ponta a ponta | e2e | CA-28.1 e link compartilhado | `frontend/e2e/*.spec.ts` | `make e2e` |

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
T1 → T3
T2 → T3 → T4 → T5
T3 → T6
T7
```

### Phase 2: Frontend — consumo

```
T8 → T9 → T10
T9 → T11
```

### Phase 3: Frontend — edição e E2E

```
T12 → T13
T12 → T14
```

---

## Task Breakdown

### Phase 1: Backend

#### T1: `RelativeDateResolver`

**What**: Intervalos dos 7 atalhos no fuso da instância.
**Where**: `backend/src/main/java/io/databoard/dashboard/filter/RelativeDateResolver.java`
**Depends on**: None
**Reuses**: `GeneralSettings` (F02)
**Requirement**: FLT-04
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes (15/05/2026): mês atual 01–31/05 (CA-28.1); mês anterior 01–30/04; últimos 7 dias 09–15/05; ontem 14/05; ano atual 01/01–31/12/2026

**Tests**: unit
**Gate**: quick
**Commit**: `feat(dashboard): resolve relative date presets`

---

#### T2: `SqlParamResolver`

**What**: Troca de `:nome` por binds fora de literais e comentários, ignorando `::`; lista → múltiplos binds; ausente → `NULL`.
**Where**: `backend/src/main/java/io/databoard/semantic/view/SqlParamResolver.java`
**Depends on**: None
**Reuses**: `SqlTokenLexer` (F15)
**Requirement**: FLT-02

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: `:inicio` vira `?`; `x::date` intacto; `':inicio'` intacto; lista de 3 → `?, ?, ?`; ausente → `NULL`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(semantic): bind sql view parameters`

---

#### T3: `ScreenFilterApplier`

**What**: Resolução de filtros de tela (coluna homônima, mapeamento para coluna/parâmetro/ignorar), filtros de objeto, validação de tipo e `activeDateRange`.
**Where**: `backend/src/main/java/io/databoard/dashboard/filter/ScreenFilterApplier.java`
**Depends on**: T1, T2
**Reuses**: `TokenService` (F15)
**Requirement**: FLT-01, FLT-02, FLT-04, FLT-05, FLT-09

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: coluna homônima aplicada; mapeamento para outra coluna; `PARAMETER` com `rangePart`; `IGNORE`; múltipla vazia sem predicado; texto em numérico → `INVALID_FILTER_VALUE`; filtro de objeto somado com E; `activeDateRange` devolve o intervalo mapeado

**Tests**: unit
**Gate**: quick
**Commit**: `feat(dashboard): apply screen and object filters`

---

#### T4: Filtros no pipeline de dados e na validação

**What**: `ObjectDataService` passa `AppliedFilters` aos handlers; `PeriodResolver` (F20) usa `activeDateRange`; `ScreenDefinitionValidator` valida filtros (máx. 20, slugs, colunas, mapeamentos).
**Where**: `backend/src/main/java/io/databoard/dashboard/object/ObjectDataService.java` (modify)
**Depends on**: T3
**Reuses**: F19, F20
**Requirement**: FLT-01, FLT-05, FLT-09

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ScreenFiltersIT`: tela com 3 objetos e filtro de região → SQL dos dois homônimos e do mapeado contém o predicado, o `IGNORE` não; KPI usa o intervalo do filtro de período; 21 filtros → `SCREEN_TOO_MANY_FILTERS`; RLS e filtro combinados com E

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): wire filters into object data pipeline`

---

#### T5: Endpoint de valores distintos

**What**: `GET /screens/{id}/filters/{filterId}/values?q=&page=` via motor com RLS, 100 por página.
**Where**: `backend/src/main/java/io/databoard/dashboard/filter/FilterValuesController.java`
**Depends on**: T4
**Reuses**: `QueryEngine` (F12)
**Requirement**: FLT-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `FilterValuesIT`: valores distintos ordenados; `q` filtra; usuário com RLS só vê valores dos próprios dados; sem `VIEW` na tela → 403

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): serve distinct filter values`

---

#### T6: Parâmetros `:param` no motor

**What**: Registrar `SqlParamResolver` no `QueryEngine` com valores vindos de `AppliedFilters`.
**Where**: `backend/src/main/java/io/databoard/semantic/view/SqlParamIntegration.java`
**Depends on**: T3
**Reuses**: `QueryParameterResolver` (F12)
**Requirement**: FLT-02

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SqlParamIT`: visão SQL com `WHERE data >= :inicio_periodo` filtrada pelo filtro mapeado; sem valor → `NULL` (0 linhas)

**Tests**: integration
**Gate**: full
**Commit**: `feat(semantic): resolve view parameters from filters`

---

#### T7: Estado "lembrar última escolha"

**What**: `V16__filter_state.sql` e `GET|PUT /screens/{id}/filter-state`.
**Where**: `backend/src/main/java/io/databoard/dashboard/filter/FilterStateController.java`
**Depends on**: None
**Reuses**: —
**Requirement**: FLT-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `FilterStateIT`: grava e devolve por usuário; outro usuário não vê; excluir tela remove o estado

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): persist last used filter values`

---

### Phase 2: Frontend — consumo

#### T8: `filterUrlCodec`

**What**: (De)serialização de filtros na URL (`f.<slug>`, listas, intervalos ISO/`yyyy-MM`, atalhos relativos).
**Where**: `frontend/src/features/screens/filters/filterUrlCodec.ts`
**Depends on**: None
**Reuses**: —
**Requirement**: FLT-07
**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] Testes: ida e volta para cada tipo; `f.periodo=2026-01..2026-03`; `f.periodo=mes_atual`; slug desconhecido ignorado

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): encode screen filters in url`

---

#### T9: `useScreenFilters`

**What**: Estado dos filtros com precedência URL > LAST_USED > padrão, sincronização com a URL, `PUT filter-state` com debounce e `requiredMissing`.
**Where**: `frontend/src/features/screens/filters/useScreenFilters.ts`
**Depends on**: T8
**Reuses**: `useSearchParams`
**Requirement**: FLT-04, FLT-06, FLT-07

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: URL prevalece sobre LAST_USED; LAST_USED sobre padrão; obrigatório vazio → `requiredMissing` e objetos não consultam

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): manage screen filter state`

---

#### T10: `FilterBar` e controles

**What**: Barra superior com `SelectFilter` (async, busca, paginação), `TextFilter`, `NumberRangeFilter`, `DateRangeFilter` (atalhos + personalizado), `BooleanFilter`; "Aplicar" quando > 3 filtros.
**Where**: `frontend/src/features/screens/filters/FilterBar.tsx`
**Depends on**: T9
**Reuses**: Mantine `MultiSelect`, `DatePickerInput`
**Requirement**: FLT-01, FLT-03, FLT-04

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `frontend-design`

**Done when**:
- [ ] Testes: atalho "Mês atual" gera valor relativo; início > fim bloqueia; busca chama `values?q=`; com 4 filtros só aplica ao clicar

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add screen filter bar and controls`

---

#### T11: `ObjectFilterIndicator`

**What**: Ícone no `ObjectFrame` com tooltip dos filtros de tela e de objeto visíveis que afetam o objeto.
**Where**: `frontend/src/features/screens/filters/ObjectFilterIndicator.tsx`
**Depends on**: T9
**Reuses**: `ObjectFrame` (F19)
**Requirement**: FLT-08

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: objeto com mapeamento `IGNORE` não lista o filtro; filtro de objeto `visible=false` não aparece

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): show filters affecting each object`

---

### Phase 3: Frontend — edição e E2E

#### T12: `ScreenFiltersEditor`

**What**: Definir filtros de tela (rótulo, controle, coluna-fonte, padrão por tipo, obrigatório) e mapeamentos por objeto com sugestão automática por nome de coluna.
**Where**: `frontend/src/features/screens/filters/ScreenFiltersEditor.tsx`
**Depends on**: None
**Reuses**: `ScreenEditor` (F19), `tokenCompletion` (F15)
**Requirement**: FLT-01, FLT-02, FLT-04, FLT-06

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: objetos com coluna homônima aparecem como "automático"; mapear para parâmetro lista os `:param` da visão SQL; padrão `TOKEN` aceita autocomplete

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add screen filters editor`

---

#### T13: `ObjectFiltersEditor`

**What**: Filtros fixos por objeto (coluna, operador, valores, visível) dentro do `ObjectConfigDrawer`.
**Where**: `frontend/src/features/screens/filters/ObjectFiltersEditor.tsx`
**Depends on**: T12
**Reuses**: `ObjectConfigDrawer` (F19)
**Requirement**: FLT-05

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: operadores conforme o tipo da coluna; `visible` padrão desligado

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add object filters editor`

---

#### T14: E2E de filtros

**What**: Tela com filtro de período padrão "mês atual" (relógio do Playwright em 15/05/2026) mostra o intervalo 01–31/05 (CA-28.1); aplicar região, copiar a URL e abrir em outro contexto restaura o estado; filtro obrigatório vazio bloqueia os objetos.
**Where**: `frontend/e2e/filters.spec.ts`
**Depends on**: T12
**Reuses**: Fixtures de telas (F19–F22)
**Requirement**: FLT-04, FLT-06, FLT-07

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): cover screen filters, defaults and shared urls`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T2, T7 | None | inícios da fase 1 | ✅ |
| T3 | T1, T2 | T1 → T3, T2 → T3 | ✅ |
| T4 | T3 | T3 → T4 | ✅ |
| T5 | T4 | T4 → T5 | ✅ |
| T6 | T3 | T3 → T6 | ✅ |
| T8 | None | início da fase 2 | ✅ |
| T9 | T8 | T8 → T9 | ✅ |
| T10, T11 | T9 | T9 → T10/T11 | ✅ |
| T12 | None | início da fase 3 | ✅ |
| T13, T14 | T12 | T12 → T13/T14 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1–T3 | Backend domínio | unit | unit | ✅ |
| T4–T7 | Backend integração | integration | integration | ✅ |
| T8–T13 | Frontend | unit | unit | ✅ |
| T14 | Ponta a ponta | e2e | e2e | ✅ |
