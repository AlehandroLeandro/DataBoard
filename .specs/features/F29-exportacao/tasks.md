# F29 — Exportação Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F29-exportacao/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| analytics export (xlsx, masks) | unit | Formatos por máscara, tipos nativos, injeção, total | `analytics/tests/unit/test_xlsx*.py` | `make test-analytics` |
| analytics rota de export | integration | Feliz + erro + limite | `analytics/tests/api/test_export.py` | `make test-analytics` |
| Backend domínio (CsvWriter, estimador) | unit | Formato pt-BR, normalizado, injeção, truncamento | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend integração | integration (analytics real) | Objeto, detalhado, Browser, assíncrono, expiração, RLS | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Menu, PNG, painel | `frontend/src/**/*.test.tsx` | `make test-frontend` |
| Ponta a ponta | e2e | Baixar XLSX de uma tabela | `frontend/e2e/*.spec.ts` | `make e2e` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-analytics` / `make test-backend` / `make test-frontend` |
| Full | Integração/e2e | `make test-backend it-backend test-analytics` (+ `make e2e`) |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Geradores

```
T1 → T2 → T3
T4
```

### Phase 2: Serviço de exportação

```
T5 → T6 → T7 → T8
T6 → T9
```

### Phase 3: Frontend e E2E

```
T10 → T11
T12
T10 → T13
```

---

## Task Breakdown

### Phase 1: Geradores

#### T1: Máscara → formato do Excel

**What**: `mask_to_excel_format` para todas as máscaras NUMBER/DATE e tratamento de TEXT.
**Where**: `analytics/src/databoard_analytics/export/masks.py`
**Depends on**: None
**Reuses**: Definição das máscaras (F13/F14)
**Requirement**: EXP-02
**Tools**: MCP: NONE · Skill: `anthropic-skills:xlsx`

**Done when**:
- [ ] Testes: as 18 predefinidas mapeadas (`brl` → `"R$" #,##0.00`, `pct1` → `0.0%`, `thousand` → `#,##0.0, "mil"`, `date_short` → `dd/mm/yyyy`); customizada com prefixo/sufixo/escala; TEXT → `None` (formatado como texto)

**Tests**: unit
**Gate**: quick
**Commit**: `feat(analytics): map masks to excel number formats`

---

#### T2: Escritor XLSX

**What**: `write_xlsx(meta, csv_stream) -> bytes stream` com tipos nativos, formatos, cabeçalho em negrito, congelar primeira linha, linha de total, aba "Aviso" quando truncado e proteção contra fórmula.
**Where**: `analytics/src/databoard_analytics/export/xlsx.py`
**Depends on**: T1
**Reuses**: `xlsxwriter` (`constant_memory`)
**Requirement**: EXP-02, EXP-03, EXP-08

**Tools**: MCP: NONE · Skill: `anthropic-skills:xlsx`

**Done when**:
- [ ] Testes (lendo com `openpyxl`): número é `float` com formato; data é `datetime`; `=SOMA(A1)` gravado como texto `'=SOMA(A1)`; total em negrito; truncado → aba "Aviso"

**Tests**: unit
**Gate**: quick
**Commit**: `feat(analytics): write xlsx exports with native types`

---

#### T3: Rota `/v1/export/xlsx`

**What**: Endpoint multipart (meta + CSV normalizado) com resposta em streaming e limite de 100 MB.
**Where**: `analytics/src/databoard_analytics/api/v1/export.py`
**Depends on**: T2
**Reuses**: `InternalTokenMiddleware` (F00)
**Requirement**: EXP-02

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `tests/api/test_export.py`: XLSX válido; meta inválida → 422; sem token → 401; `make contract` atualizado

**Tests**: integration
**Gate**: full
**Commit**: `feat(analytics): expose xlsx export endpoint`

---

#### T4: `CsvWriter`

**What**: Escrita em streaming nos modos pt-BR (BOM, `;`, vírgula decimal, `dd/MM/yyyy`) e normalizado (`,`, ponto, ISO), proteção contra fórmula e linha de truncamento.
**Where**: `backend/src/main/java/io/databoard/dashboard/export/CsvWriter.java`
**Depends on**: None
**Reuses**: —
**Requirement**: EXP-01, EXP-03, EXP-08

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: `1234.5` → `1234,5` (pt-BR) e `1234.5` (normalizado); texto com `;` e aspas escapado; `@cmd` → `'@cmd`; BOM só no pt-BR; aviso de truncamento na primeira linha

**Tests**: unit
**Gate**: quick
**Commit**: `feat(dashboard): stream csv exports`

---

### Phase 2: Serviço de exportação

#### T5: Migração e `ExportSource`s

**What**: `V17__exports.sql`; fontes `ObjectExportSource` (resultado do pipeline), `DetailedExportSource` (linhas cruas via `QueryEngine.stream`) e `BrowserExportSource`.
**Where**: `backend/src/main/java/io/databoard/dashboard/export/ExportSources.java`
**Depends on**: None
**Reuses**: `ObjectDataService` (F19), `ScreenFilterApplier` (F28), `BrowserDataService` (F10)
**Requirement**: EXP-03, EXP-04, EXP-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ExportSourcesIT`: objeto devolve colunas calculadas e total; detalhado devolve linhas cruas com filtros e RLS (Ana × Bruno); Browser sem paginação com filtros; objeto estático → `EXPORT_NOT_SUPPORTED`

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): add export sources for objects, details and browsers`

---

#### T6: `ExportService` síncrono

**What**: Flag `EXPORT`, limite `maxExportRows`, CSV direto ou XLSX via analytics, nome de arquivo seguro, `DataExported`.
**Where**: `backend/src/main/java/io/databoard/dashboard/export/ExportService.java`
**Depends on**: T5
**Reuses**: `CsvWriter` (T4), `AnalyticsClient` (F00)
**Requirement**: EXP-01, EXP-02, EXP-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ExportServiceIT`: XLSX da tabela top 10 lido com POI tem formatos e total; sem flag → 403; limite 1.000 com 5.000 linhas → 1.000 + aviso; evento `DataExported` publicado; analytics parado → 503

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): export objects to csv and xlsx`

---

#### T7: Exportação assíncrona

**What**: `RowEstimator` (timeout 5 s), job acima de 50 mil linhas, `FileStore` com 24 h, e-mail opcional, limite de 3 simultâneas, `GET /exports`, `GET /exports/{id}`, `GET /exports/{id}/download`.
**Where**: `backend/src/main/java/io/databoard/dashboard/export/ExportJobRunner.java`
**Depends on**: T6
**Reuses**: `JobRunRecorder`, `FileStore` (F11), `MailService` (F03)
**Requirement**: EXP-07

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AsyncExportIT`: 60 mil linhas → 202; polling até `READY`; download pelo dono; outro usuário → 404; 4ª simultânea → 429; e-mail enviado (GreenMail) com SMTP configurado

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): run large exports asynchronously`

---

#### T8: Limpeza de exportações

**What**: `ExportCleanupJob` horário apagando arquivos expirados; `download` expirado → 410.
**Where**: `backend/src/main/java/io/databoard/dashboard/export/ExportCleanupJob.java`
**Depends on**: T7
**Reuses**: `FileStore` (F11)
**Requirement**: EXP-07

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ExportCleanupIT` (relógio controlado): após 24 h arquivo removido e `download` → 410 `EXPORT_EXPIRED`

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): purge expired exports`

---

#### T9: Controllers de exportação

**What**: `POST /screens/{id}/objects/{oid}/export`, `POST /browsers/{id}/export` e `POST /exports/audit` (PNG).
**Where**: `backend/src/main/java/io/databoard/dashboard/export/ExportController.java`
**Depends on**: T6
**Reuses**: —
**Requirement**: EXP-01, EXP-04, EXP-05, EXP-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ExportControllerIT`: `Content-Type` e `Content-Disposition` corretos; `detailed=true`; `/exports/audit` publica `DataExported` com `PNG`; sem `VIEW` na tela → 403

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): expose export endpoints`

---

### Phase 3: Frontend e E2E

#### T10: `ObjectExportMenu`

**What**: Menu de exportação no `ObjectFrame` (CSV, XLSX, PNG em gráficos, dados detalhados), oculto sem `EXPORT`, tratando 202 (adiciona ao painel) e download direto.
**Where**: `frontend/src/features/screens/objects/ObjectExportMenu.tsx`
**Depends on**: None
**Reuses**: `usePermission` (F04), `shared/download.ts`
**Requirement**: EXP-01, EXP-04, EXP-05
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: sem flag → menu ausente; PNG chama `getDataURL({pixelRatio: 2})` e `POST /exports/audit`; resposta 202 adiciona item ao painel; objeto estático sem opções

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add object export menu`

---

#### T11: `ExportsPanel`

**What**: Painel "Minhas exportações" com polling a cada 3 s enquanto houver `RUNNING`, link de download e expiradas desabilitadas.
**Where**: `frontend/src/features/exports/ExportsPanel.tsx`
**Depends on**: T10
**Reuses**: TanStack Query `refetchInterval`
**Requirement**: EXP-07

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: polling para quando não há `RUNNING`; expirada sem link

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add exports panel`

---

#### T12: `BrowserExportButton`

**What**: Botão "Exportar" no Browser enviando filtros, busca e ordenação atuais.
**Where**: `frontend/src/features/browser/BrowserExportButton.tsx`
**Depends on**: None
**Reuses**: `useBrowserData` (F10)
**Requirement**: EXP-06

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: payload contém os filtros atuais; oculto sem `EXPORT`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): export browser grid`

---

#### T13: E2E de exportação

**What**: Exportar uma tabela em XLSX pela UI e validar o arquivo baixado (células numéricas, formato `R$`, linha de total) com uma biblioteca de leitura no teste.
**Where**: `frontend/e2e/export.spec.ts`
**Depends on**: T10
**Reuses**: Fixtures de telas (F21)
**Requirement**: EXP-01, EXP-02

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): cover xlsx export of a table object`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T4 | None | inícios da fase 1 | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T3 | T2 | T2 → T3 | ✅ |
| T5 | None | início da fase 2 | ✅ |
| T6 | T5 | T5 → T6 | ✅ |
| T7 | T6 | T6 → T7 | ✅ |
| T8 | T7 | T7 → T8 | ✅ |
| T9 | T6 | T6 → T9 | ✅ |
| T10, T12 | None | inícios da fase 3 | ✅ |
| T11 | T10 | T10 → T11 | ✅ |
| T13 | T10 | T10 → T13 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1, T2 | analytics export | unit | unit | ✅ |
| T3 | analytics rota | integration | integration | ✅ |
| T4 | Backend domínio | unit | unit | ✅ |
| T5–T9 | Backend integração | integration | integration | ✅ |
| T10–T12 | Frontend | unit | unit | ✅ |
| T13 | Ponta a ponta | e2e | e2e | ✅ |
