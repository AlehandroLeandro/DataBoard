# F21 — Objeto Consulta Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F21-objeto-consulta/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| analytics transform `table` | unit | Todos os tipos de coluna calculada; total 0; stats | `analytics/tests/unit/test_table.py` | `make test-analytics` |
| Backend handler | unit | Validações e plano | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend integração | integration (analytics real) | Top N, total, sortOverride | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Ordenação local/remota, células, editor | `frontend/src/**/*.test.tsx` | `make test-frontend` |

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
T2 → T3
```

### Phase 2: Frontend

```
T4 → T5 → T6
```

---

## Task Breakdown

### Phase 1: Cálculo

#### T1: Transform `table`

**What**: Kind `table` com colunas calculadas (`PERCENT_OF_TOTAL`, `DIFF_PREVIOUS_ROW`, `PCT_DIFF_PREVIOUS_ROW`, `CUMULATIVE_SUM`, `CUMULATIVE_PCT`, `RANK`) e stats min/max.
**Where**: `analytics/src/databoard_analytics/transforms/table.py`
**Depends on**: None
**Reuses**: Registry de transforms (F19)
**Requirement**: TBL-02, TBL-04, TBL-05
**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] Testes: % do total usa o total do motor com top 3 de 10; acumulado na ordem recebida; rank denso com empate; diff primeira linha `None`; total 0 → % `None`; stats de colunas DATA_BAR

**Tests**: unit
**Gate**: quick
**Commit**: `feat(analytics): add table transform with calculated columns`

---

#### T2: `QueryObjectHandler`

**What**: Validação, dependências e plano (`totals`, `topN`, `sortOverride`) para o tipo `QUERY`.
**Where**: `backend/src/main/java/io/databoard/dashboard/object/query/QueryObjectHandler.java`
**Depends on**: None
**Reuses**: `ObjectTypeHandler` (F19)
**Requirement**: TBL-01, TBL-02, TBL-03, TBL-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: coluna duplicada → `DUPLICATE_COLUMN`; `linkTemplate` `javascript:` → erro; calculada com base inexistente → erro; `totals=true` quando há `PERCENT_OF_TOTAL`; `sortOverride` substitui a ordenação padrão

**Tests**: unit
**Gate**: quick
**Commit**: `feat(dashboard): add query table object handler`

---

#### T3: Integração do objeto Consulta

**What**: IT ponta a ponta: top 10 produtos com total, % do total, ranking e `truncated`, analytics real.
**Where**: `backend/src/test/java/io/databoard/dashboard/object/query/QueryObjectIT.java`
**Depends on**: T2
**Reuses**: `ObjectDataService` (F19)
**Requirement**: TBL-01, TBL-02, TBL-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `QueryObjectIT` verde: 10 linhas de 25 com `truncated=true`; total do motor; soma dos % das 10 linhas < 100%

**Tests**: integration
**Gate**: full
**Commit**: `test(dashboard): cover query table object end to end`

---

### Phase 2: Frontend

#### T4: `QueryTableRenderer`

**What**: Grade virtualizada com redimensionamento, ordenação local (Collator pt-BR) ou remota (`sortOverride` se truncado), linha de total e aviso de truncamento.
**Where**: `frontend/src/features/screens/objects/query/QueryTableRenderer.tsx`
**Depends on**: None
**Reuses**: TanStack Table/Virtual, `formatValue`
**Requirement**: TBL-01, TBL-02, TBL-03
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: clique sem truncamento não chama API; com truncamento chama com `sortOverride`; total fixo no rodapé; aviso "Mostrando 10 primeiras linhas"

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): render query table objects`

---

#### T5: Renderers de célula

**What**: `MaskedCell`, `DataBarCell`, `ImageCell` (URL ou fileId), `LinkCell` (valor codificado, só http/https) e alinhamento padrão por tipo.
**Where**: `frontend/src/features/screens/objects/query/cells.tsx`
**Depends on**: T4
**Reuses**: `formatValue` (F13/F14)
**Requirement**: TBL-05

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `dataviz`

**Done when**:
- [ ] Testes: barra de 50% para valor no meio do intervalo; negativo à esquerda; URL inválida → placeholder; link com `a b` → `a%20b`; número alinhado à direita

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add rich cells for query tables`

---

#### T6: `QueryTableEditor`

**What**: Editor (visão, dimensões/medidas ordenáveis, ordenação padrão, top N, total, colunas calculadas, formatação por coluna) registrado no `objectRegistry`.
**Where**: `frontend/src/features/screens/objects/query/QueryTableEditor.tsx`
**Depends on**: T5
**Reuses**: `ObjectConfigDrawer` (F19), `useMasks`
**Requirement**: TBL-01, TBL-04, TBL-05

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: config gerada bate com o formato do design; calculada só oferece medidas da tabela; `topN` acima do limite bloqueia salvar

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add query table object editor`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T2 | None | inícios da fase 1 | ✅ |
| T3 | T2 | T2 → T3 | ✅ |
| T4 | None | início da fase 2 | ✅ |
| T5 | T4 | T4 → T5 | ✅ |
| T6 | T5 | T5 → T6 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1 | analytics transform | unit | unit | ✅ |
| T2 | Backend handler | unit | unit | ✅ |
| T3 | Backend integração | integration | integration | ✅ |
| T4–T6 | Frontend | unit | unit | ✅ |
