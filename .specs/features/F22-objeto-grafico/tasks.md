# F22 — Objeto Gráfico Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F22-objeto-grafico/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| analytics charts (prepare, builders, options) | unit (snapshot) | Todos os tipos e variações; limites de categoria; log; meta | `analytics/tests/unit/charts/test_*.py` | `make test-analytics` |
| Backend handler | unit | Restrições por tipo | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend integração | integration (analytics real) | Gráfico com meta por medida | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Formatters, troca de tipo, avisos, vazio | `frontend/src/**/*.test.tsx` | `make test-frontend` |
| Ponta a ponta | e2e | Tela completa (CA-19.1, CA-19.2) | `frontend/e2e/*.spec.ts` | `make e2e` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-analytics` / `make test-backend` / `make test-frontend` |
| Full | Integração/e2e | `make test-backend it-backend test-analytics` (+ `make e2e`) |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: analytics

```
T1 → T2 → T3 → T6
T2 → T4 → T6
T2 → T5 → T6
```

### Phase 2: Backend

```
T7 → T8
```

### Phase 3: Frontend e E2E

```
T9 → T10 → T11
T11 → T12
```

---

## Task Breakdown

### Phase 1: analytics

#### T1: Paleta e utilitários de opções

**What**: Paleta categórica da plataforma (8 cores validadas) e helpers de eixo, legenda, rótulos e `markLine`.
**Where**: `analytics/src/databoard_analytics/transforms/charts/options.py`
**Depends on**: None
**Reuses**: —
**Requirement**: CHT-03, CHT-04
**Tools**: MCP: NONE · Skill: `dataviz`

**Done when**:
- [ ] Testes: legenda `hidden` → `show: False`; `min/max` aplicados; cor fixa por série prevalece sobre a paleta; `markLine` no valor da meta

**Tests**: unit
**Gate**: quick
**Commit**: `feat(analytics): add chart palette and option helpers`

---

#### T2: `prepare` — pivot, ordem, limites e 100%

**What**: Long→wide por série, ordenação de categorias, limite de categorias/séries com "Outros" (aditivas) ou aviso (não aditivas), normalização 100%.
**Where**: `analytics/src/databoard_analytics/transforms/charts/prepare.py`
**Depends on**: T1
**Reuses**: polars
**Requirement**: CHT-01, CHT-07

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] Testes: 80 categorias SUM → 50 com "Outros" = soma das 31 menores; 80 AVG → 50 primeiras + aviso; 25 séries → "Outras séries"; 100% soma 1 por categoria; datas em ordem cronológica; `null` preservado

**Tests**: unit
**Gate**: quick
**Commit**: `feat(analytics): prepare chart data with category limits`

---

#### T3: Builder cartesiano (BAR, LINE, AREA, COMBO)

**What**: Séries, orientação, empilhamento, `areaStyle`, eixo secundário no COMBO, escala log com fallback.
**Where**: `analytics/src/databoard_analytics/transforms/charts/builders/cartesian.py`
**Depends on**: T2
**Reuses**: `options.py` (T1)
**Requirement**: CHT-01, CHT-03

**Tools**: MCP: NONE · Skill: `dataviz`

**Done when**:
- [ ] Snapshots: barra vertical/horizontal, agrupada/empilhada/100%, linha, área empilhada, combo com `yAxisIndex: 1`; log com valor 0 → linear + `LOG_SCALE_INVALID`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(analytics): build cartesian chart options`

---

#### T4: Builder de pizza/rosca

**What**: Série `pie` com rosca (raio interno 50%) e cores por valor.
**Where**: `analytics/src/databoard_analytics/transforms/charts/builders/pie.py`
**Depends on**: T2
**Reuses**: `options.py` (T1)
**Requirement**: CHT-01

**Tools**: MCP: NONE · Skill: `dataviz`

**Done when**:
- [ ] Snapshots de pizza e rosca; cor fixa por valor aplicada

**Tests**: unit
**Gate**: quick
**Commit**: `feat(analytics): build pie and donut chart options`

---

#### T5: Builders de dispersão e funil

**What**: `scatter` (medida 1 em X, medida 2 em Y, dimensão como série/rótulo) e `funnel` (decrescente).
**Where**: `analytics/src/databoard_analytics/transforms/charts/builders/scatter.py`
**Depends on**: T2
**Reuses**: `options.py` (T1)
**Requirement**: CHT-01

**Tools**: MCP: NONE · Skill: `dataviz`

**Done when**:
- [ ] Snapshots de dispersão e funil; funil ordenado decrescente

**Tests**: unit
**Gate**: quick
**Commit**: `feat(analytics): build scatter and funnel chart options`

---

#### T6: Kind `chart` e `meta`

**What**: Dispatch por tipo, `meta` com máscaras por série/eixo e avisos, registro no registry; OpenAPI atualizado.
**Where**: `analytics/src/databoard_analytics/transforms/charts/__init__.py`
**Depends on**: T3, T4, T5
**Reuses**: Builders (T3–T5), registry (F19)
**Requirement**: CHT-01, CHT-05, CHT-07

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] Testes: cada tipo despacha para o builder certo; `meta.valueMasks` por série; avisos agregados; resultado vazio → `option` vazio + `empty: true`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(analytics): register chart transform`

---

### Phase 2: Backend

#### T7: `ChartObjectHandler`

**What**: Validação das restrições por tipo, dependências e plano (consulta principal + meta opcional).
**Where**: `backend/src/main/java/io/databoard/dashboard/object/chart/ChartObjectHandler.java`
**Depends on**: None
**Reuses**: `ObjectTypeHandler` (F19), consultas opcionais (F20)
**Requirement**: CHT-02, CHT-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: `PIE` com 2 medidas → erro; `SCATTER` com 1 medida → erro; `COMBO` sem linha → erro; cor inválida → erro; plano com dimensões `[x, series]`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(dashboard): add chart object handler`

---

#### T8: Integração do gráfico

**What**: IT ponta a ponta de um `BAR` empilhado com série e meta por medida de outra visão.
**Where**: `backend/src/test/java/io/databoard/dashboard/object/chart/ChartObjectIT.java`
**Depends on**: T7
**Reuses**: `ObjectDataService` (F19)
**Requirement**: CHT-01, CHT-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ChartObjectIT` verde: `option.series` por canal, `stack` comum, `markLine` com o valor da meta

**Tests**: integration
**Gate**: full
**Commit**: `test(dashboard): cover chart object end to end`

---

### Phase 3: Frontend e E2E

#### T9: `ChartRenderer`

**What**: ECharts modular com `applyFormatters` (tooltip, rótulos, eixos, datas `MM/yyyy`), avisos, estado vazio e resize.
**Where**: `frontend/src/features/screens/objects/chart/ChartRenderer.tsx`
**Depends on**: None
**Reuses**: `maskToEChartsFormatter` (F13/F14)
**Requirement**: CHT-03, CHT-05, CHT-07
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `dataviz`

**Done when**:
- [ ] Testes: tooltip formata `brl`; avisos exibem ícone com lista; `empty` → "Sem dados"; resize chama `chart.resize`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): render chart objects with echarts`

---

#### T10: `chartConfigCompat`

**What**: Validação por tipo no front, espelhando o backend.
**Where**: `frontend/src/features/screens/objects/chart/chartConfigCompat.ts`
**Depends on**: T9
**Reuses**: —
**Requirement**: CHT-02, CHT-06

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] Testes: mesmas regras do T7 (tabela de casos idêntica)

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): validate chart config per type`

---

#### T11: `ChartEditor`

**What**: Editor com seletor de tipo, dados, aparência, meta e troca de tipo sem perder configuração, registrado no `objectRegistry`.
**Where**: `frontend/src/features/screens/objects/chart/ChartEditor.tsx`
**Depends on**: T10
**Reuses**: `ObjectConfigDrawer` (F19), `useMasks`
**Requirement**: CHT-01, CHT-03, CHT-04, CHT-06

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `frontend-design`

**Done when**:
- [ ] Testes: `BAR`→`LINE` mantém medidas e cores; `BAR` (2 medidas)→`PIE` destaca o campo e bloqueia salvar; config salva bate com o design

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add chart object editor`

---

#### T12: E2E de tela completa

**What**: Montar pela UI uma tela com 6 objetos (2 KPIs, 1 tabela, 3 gráficos), um deles sobre visão quebrada: os outros 5 renderizam (CA-19.1); com atualização a cada 5 min e relógio do Playwright avançado 10 min, os dados recarregam 2 vezes (CA-19.2).
**Where**: `frontend/e2e/screen-full.spec.ts`
**Depends on**: T11
**Reuses**: Fixtures de UDT/visões (F09/F12)
**Requirement**: CHT-08

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): cover full screen with isolated object errors and auto-refresh`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1 | None | início | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T3, T4, T5 | T2 | T2 → T3/T4/T5 | ✅ |
| T6 | T3, T4, T5 | T3/T4/T5 → T6 | ✅ |
| T7 | None | início da fase 2 | ✅ |
| T8 | T7 | T7 → T8 | ✅ |
| T9 | None | início da fase 3 | ✅ |
| T10 | T9 | T9 → T10 | ✅ |
| T11 | T10 | T10 → T11 | ✅ |
| T12 | T11 | T11 → T12 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1–T6 | analytics charts | unit | unit | ✅ |
| T7 | Backend handler | unit | unit | ✅ |
| T8 | Backend integração | integration | integration | ✅ |
| T9–T11 | Frontend | unit | unit | ✅ |
| T12 | Ponta a ponta | e2e | e2e | ✅ |
