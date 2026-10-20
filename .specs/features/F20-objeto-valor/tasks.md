# F20 — Objeto Valor Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F20-objeto-valor/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| analytics transform `kpi` | unit | 100% dos ramos; 1:1 com ACs | `analytics/tests/unit/test_kpi.py` | `make test-analytics` |
| Backend domínio (períodos, handler) | unit | Viradas de mês/ano, bissexto, validações | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend integração do handler | integration (analytics real) | Cenário completo | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Renderer e editor | `frontend/src/**/*.test.tsx` | `make test-frontend` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-analytics` / `make test-backend` / `make test-frontend` |
| Full | Integração | `make test-backend it-backend test-analytics` |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Cálculo

```
T1
T2 → T3 → T4
```

### Phase 2: Frontend

```
T5 → T6
```

---

## Task Breakdown

### Phase 1: Cálculo

#### T1: Transform `kpi`

**What**: Função pura `kpi` registrada como kind `kpi`, com `KpiResult`.
**Where**: `analytics/src/databoard_analytics/transforms/kpi.py`
**Depends on**: None
**Reuses**: Registry de transforms (F19)
**Requirement**: KPI-02, KPI-03, KPI-04
**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] Testes: delta e deltaPct; base 0 → `deltaPct=None`; `higherIsBetter=false` inverte sentiment; |0,4%| → FLAT; meta 100k/80k → 0.8; grãos faltantes → 0 para SUM e None para AVG

**Tests**: unit
**Gate**: quick
**Commit**: `feat(analytics): compute kpi comparisons and targets`

---

#### T2: `PeriodResolver`

**What**: Período atual por `defaultPeriod` (ou filtro ativo), anterior calendário/por comprimento, mesmo período do ano anterior e grãos da sparkline, no fuso da instância.
**Where**: `backend/src/main/java/io/databoard/dashboard/object/value/PeriodResolver.java`
**Depends on**: None
**Reuses**: `GeneralSettings` (F02)
**Requirement**: KPI-01, KPI-02, KPI-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: 15/05/2026 `CURRENT_MONTH` → 01–31/05; anterior → 01–30/04; ano anterior → 01–31/05/2025; janeiro → anterior dezembro do ano anterior; 29/02/2028 → 28/02/2027; intervalo de 10 dias → anterior de 10 dias imediatamente antes; 12 grãos mensais terminando em maio

**Tests**: unit
**Gate**: quick
**Commit**: `feat(dashboard): resolve kpi periods`

---

#### T3: `ValueObjectHandler`

**What**: Validação, dependências e plano de até 4 consultas com `spec` para o `kpi`; a consulta da meta é marcada `optional=true` e o `ObjectDataService` (F19) passa a tratar falha de consulta opcional como resultado `null` em vez de erro do objeto.
**Where**: `backend/src/main/java/io/databoard/dashboard/object/value/ValueObjectHandler.java`
**Depends on**: T2
**Reuses**: `ObjectTypeHandler` (F19)
**Requirement**: KPI-01, KPI-02, KPI-03, KPI-04, KPI-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: comparativo sem `dateColumn` → erro em `config.dateColumn`; `ALL` + `PREVIOUS_PERIOD` → erro; plano com filtros `BETWEEN` corretos; dependências incluem visão da meta e tela destino

**Tests**: unit
**Gate**: quick
**Commit**: `feat(dashboard): add value object handler`

---

#### T4: Integração do KPI

**What**: Teste de integração ponta a ponta do objeto `VALUE` com visão de vendas e visão de metas (UDT), analytics real.
**Where**: `backend/src/test/java/io/databoard/dashboard/object/value/ValueObjectIT.java`
**Depends on**: T3
**Reuses**: `ObjectDataService` (F19)
**Requirement**: KPI-01, KPI-02, KPI-03, KPI-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ValueObjectIT` (relógio em 15/05/2026): valor de maio, comparação com abril, meta da UDT com % atingido, sparkline de 12 meses; meta em visão quebrada → `target=null` e valor presente

**Tests**: integration
**Gate**: full
**Commit**: `test(dashboard): cover value object end to end`

---

### Phase 2: Frontend

#### T5: `ValueRenderer`

**What**: Card com ícone, título, valor formatado, delta com seta/cor por `sentiment`, barra de meta, sparkline e navegação condicionada a `VIEW` no destino.
**Where**: `frontend/src/features/screens/objects/value/ValueRenderer.tsx`
**Depends on**: None
**Reuses**: `formatValue` (F13/F14), ECharts, `objectRegistry` (F19)
**Requirement**: KPI-05, KPI-06
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `frontend-design`, `dataviz`

**Done when**:
- [ ] Testes: `GOOD` → verde com seta para cima; `deltaPct=null` → "—"; meta 0.8 → barra 80%; clique navega só com permissão

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): render value objects`

---

#### T6: `ValueEditor`

**What**: Editor do objeto (visão, medida, agregação, máscara, ícone, data, período, comparativo, direção, meta, sparkline, navegação), registrado no `objectRegistry`.
**Where**: `frontend/src/features/screens/objects/value/ValueEditor.tsx`
**Depends on**: T5
**Reuses**: `ObjectConfigDrawer` (F19), `useMasks` (F13/F14)
**Requirement**: KPI-01, KPI-02, KPI-03, KPI-04, KPI-06

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: comparativo de período sem coluna de data bloqueia salvar; meta MEASURE lista visões acessíveis; config gerada bate com o formato do design

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add value object editor`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T2 | None | inícios da fase 1 | ✅ |
| T3 | T2 | T2 → T3 | ✅ |
| T4 | T3 | T3 → T4 | ✅ |
| T5 | None | início da fase 2 | ✅ |
| T6 | T5 | T5 → T6 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1 | analytics transform | unit | unit | ✅ |
| T2, T3 | Backend domínio | unit | unit | ✅ |
| T4 | Backend integração | integration | integration | ✅ |
| T5, T6 | Frontend | unit | unit | ✅ |
