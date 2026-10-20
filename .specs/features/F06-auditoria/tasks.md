# F06 — Auditoria Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F06-auditoria/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio (redactor, mapper, fila) | unit | Todos os ramos; edge cases | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend persistência/listeners/controller/job | integration | Feliz + erro + append-only | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Filtros e grade | `frontend/src/**/*.test.tsx` | `make test-frontend` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-backend` ou `make test-frontend` |
| Full | Integração | `make test-backend it-backend` |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Núcleo de gravação

```
T1
T2 → T3
T4 → T5
```

### Phase 2: Eventos, consulta e retenção

```
T6 → T7 → T8
T9
T10
```

---

## Task Breakdown

### Phase 1: Núcleo de gravação

#### T1: Schema de auditoria append-only

**What**: `V6__audit.sql` com `audit.event`, `audit.data_change`, índices e triggers de imutabilidade.
**Where**: `backend/src/main/resources/db/migration/V6__audit.sql`
**Depends on**: None
**Reuses**: —
**Requirement**: AUD-02
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AuditSchemaIT`: `UPDATE` e `DELETE` falham; `DELETE` com `SET LOCAL databoard.audit_retention='on'` passa; `UPDATE` com a flag continua falhando

**Tests**: integration
**Gate**: full
**Commit**: `feat(audit): create append-only audit schema`

---

#### T2: `Redactor`

**What**: Mascaramento recursivo de chaves sensíveis e truncamento acima de 64 KB.
**Where**: `backend/src/main/java/io/databoard/audit/internal/Redactor.java`
**Depends on**: None
**Reuses**: —
**Requirement**: AUD-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: `password`, `smtpPassword`, `apiKey`, `nested.token` mascarados; chaves comuns intactas; JSON de 70 KB → truncado com `"truncated": true`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(audit): redact sensitive values`

---

#### T3: `AuditMapper`

**What**: Mapeia cada evento conhecido para `AuditRecord` (categoria, tipo, recurso, detalhes) e calcula o diff por campo de `UdtRecordChanged`.
**Where**: `backend/src/main/java/io/databoard/audit/internal/AuditMapper.java`
**Depends on**: T2
**Reuses**: Classes de evento de F02/F03/F04 + `audit.api`
**Requirement**: AUD-04, AUD-05, AUD-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: um caso por tipo de evento com categoria/tipo esperados; `LoginFailed` usa o e-mail tentado em `actor_label`; diff só de campos alterados; `UPDATE` sem mudança → lista vazia; `INSERT` → `old_value` nulo

**Tests**: unit
**Gate**: quick
**Commit**: `feat(audit): map domain events to audit records`

---

#### T4: `AuditQueue`

**What**: Fila limitada (50.000) com `offer` não bloqueante, contador `audit.dropped` e log limitado.
**Where**: `backend/src/main/java/io/databoard/audit/internal/AuditQueue.java`
**Depends on**: None
**Reuses**: Micrometer
**Requirement**: AUD-01

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: fila cheia descarta e incrementa a métrica; `drain(n)` devolve até n itens na ordem de chegada

**Tests**: unit
**Gate**: quick
**Commit**: `feat(audit): add bounded in-memory audit queue`

---

#### T5: `AuditBatchWriter`

**What**: Gravação em lote a cada 1 s ou 200 itens, 3 tentativas, métrica de falha e drenagem no shutdown.
**Where**: `backend/src/main/java/io/databoard/audit/internal/AuditBatchWriter.java`
**Depends on**: T4
**Reuses**: `JdbcTemplate.batchUpdate`
**Requirement**: AUD-01

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AuditBatchWriterIT`: 1.000 eventos gravados em ≤ 2 s; publicação com p95 < 5 ms; banco indisponível → `audit.write.failures` incrementa e a requisição de origem responde normalmente

**Tests**: integration
**Gate**: full
**Commit**: `feat(audit): write audit events in batches`

---

### Phase 2: Eventos, consulta e retenção

#### T6: Listeners de eventos

**What**: `@EventListener`s para AUTH, ACCESS, SECURITY, EXPORT e DATA que mapeiam e enfileiram.
**Where**: `backend/src/main/java/io/databoard/audit/internal/AuditEventListeners.java`
**Depends on**: None
**Reuses**: `AuditMapper` (T3), `AuditQueue` (T4)
**Requirement**: AUD-04, AUD-05, AUD-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AuditListenersIT`: login real gera `AUTH/LOGIN_SUCCEEDED` com IP; `PUT /settings/smtp` gera `SECURITY` com senha mascarada; `UdtRecordChanged` publicado gera linhas em `data_change`

**Tests**: integration
**Gate**: full
**Commit**: `feat(audit): persist auth, access, security, export and data events`

---

#### T7: `AuditQuery`

**What**: `search(filter, pageable)` com padrão de 7 dias e `recordHistory(tableId, recordId)` decrescente.
**Where**: `backend/src/main/java/io/databoard/audit/AuditQuery.java`
**Depends on**: T6
**Reuses**: `PageResponse` (F00)
**Requirement**: AUD-06, AUD-07

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AuditQueryIT`: filtros combinados; sem período → 7 dias; histórico do registro em ordem decrescente com campo, antigo e novo (CA-06.1)

**Tests**: integration
**Gate**: full
**Commit**: `feat(audit): query audit events and record history`

---

#### T8: `AuditController`

**What**: `GET /api/v1/audit/events` restrito a ADMIN.
**Where**: `backend/src/main/java/io/databoard/audit/internal/AuditController.java`
**Depends on**: T7
**Reuses**: `AuditQuery` (T7)
**Requirement**: AUD-07

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AuditControllerIT`: admin 200 paginado; BUILDER 403; `from > to` → 400

**Tests**: integration
**Gate**: full
**Commit**: `feat(audit): expose audit events endpoint`

---

#### T9: `RetentionJob`

**What**: Job diário às 03:00 (fuso da instância), exclusão em lotes de 10.000 e evento `SYSTEM/AUDIT_RETENTION`; campo `auditRetentionDays` em `GeneralSettings`.
**Where**: `backend/src/main/java/io/databoard/audit/internal/RetentionJob.java`
**Depends on**: None
**Reuses**: `SettingsStore` (F01/F02)
**Requirement**: AUD-08

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `RetentionJobIT` (relógio controlado): evento de 366 dias removido, de 364 mantido; evento de retenção gravado com a contagem

**Tests**: integration
**Gate**: full
**Commit**: `feat(audit): purge audit records past retention`

---

#### T10: `AuditPage`

**What**: Tela admin com filtros (período com atalhos, usuário, categoria/tipo, recurso) e grade com detalhes expansíveis.
**Where**: `frontend/src/features/admin/audit/AuditPage.tsx`
**Depends on**: None
**Reuses**: TanStack Table, Mantine `DatePickerInput`
**Requirement**: AUD-07
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: filtros viram query string; expandir linha mostra `details` formatado; paginação

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add audit log page`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T2, T4 | None | inícios da fase 1 | ✅ |
| T3 | T2 | T2 → T3 | ✅ |
| T5 | T4 | T4 → T5 | ✅ |
| T6, T9, T10 | None | inícios da fase 2 | ✅ |
| T7 | T6 | T6 → T7 | ✅ |
| T8 | T7 | T7 → T8 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T2, T3, T4 | Backend domínio | unit | unit | ✅ |
| T1, T5–T9 | Backend persistência/listeners/controller/job | integration | integration | ✅ |
| T10 | Frontend | unit | unit | ✅ |
