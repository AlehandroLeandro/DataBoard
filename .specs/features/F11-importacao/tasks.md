# F11 — Importação e Upload Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F11-importacao/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| analytics transformações | unit | Todos os formatos, encodings, separadores, inferência, normalização | `analytics/tests/unit/test_*.py` | `make test-analytics` |
| analytics rotas | integration | Feliz + erro + token | `analytics/tests/api/test_*.py` | `make test-analytics` |
| Backend domínio (validador de linha, sanitizador) | unit | Todos os ramos; 1:1 com ACs | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend controller/persistência/job | integration | Feliz + edge + erro; com analytics real | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Passos do wizard, erros, progresso | `frontend/src/**/*.test.tsx` | `make test-frontend` |
| Ponta a ponta | e2e | CA-11.1, CA-11.2 | `frontend/e2e/*.spec.ts` | `make e2e` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-analytics` / `make test-backend` / `make test-frontend` |
| Full | Integração/e2e | `make test-backend it-backend test-analytics` (+ `make e2e`) |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: analytics — leitura de planilhas

```
T1 → T2 → T3
```

### Phase 2: Arquivos

```
T4 → T6
T5 → T6 → T7
T6 → T8
```

### Phase 3: Importação no backend

```
T9
T10 → T12
T11 → T12 → T13 → T14
```

### Phase 4: Frontend

```
T15 → T16 → T17 → T18 → T19 → T20
T21
T22
```

### Phase 5: E2E

```
T23
```

---

## Task Breakdown

### Phase 1: analytics — leitura de planilhas

#### T1: `spreadsheet.inspect`

**What**: Leitura de xlsx/xls (calamine) e csv (encoding + separador), cabeçalhos únicos, inferência de tipos em 1.000 linhas e prévia de 50 linhas.
**Where**: `analytics/src/databoard_analytics/transforms/spreadsheet.py`
**Depends on**: None
**Reuses**: —
**Requirement**: IMP-01, IMP-02
**Tools**: MCP: NONE · Skill: `data:explore-data`

**Done when**:
- [ ] Testes com fixtures: xlsx 2 abas; xls; CSV Windows-1252 com `;`; cabeçalho duplicado → `Valor_2`; coluna `01/02/2026` → DATE; `sim/não` → BOOLEAN; texto de 300 caracteres → LONG_TEXT

**Tests**: unit
**Gate**: quick
**Commit**: `feat(analytics): inspect spreadsheets and infer column types`

---

#### T2: `spreadsheet.normalize`

**What**: Geração em streaming de CSV UTF-8 normalizado conforme formatos de data e separador decimal.
**Where**: `analytics/src/databoard_analytics/transforms/spreadsheet_normalize.py`
**Depends on**: T1
**Reuses**: Leitores da T1
**Requirement**: IMP-03

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] Testes: `"1.234,56"` com vírgula → `1234.56`; `15/05/2026` → `2026-05-15`; data inválida → célula vazia + coluna de diagnóstico `__errors`; 200 mil linhas geradas sem carregar tudo em memória (gerador)

**Tests**: unit
**Gate**: quick
**Commit**: `feat(analytics): normalize spreadsheet rows to canonical csv`

---

#### T3: Rotas `/v1/import/inspect` e `/v1/import/normalize`

**What**: Routers FastAPI multipart; `normalize` com `StreamingResponse`; limite de 200.001 linhas (`IMPORT_TOO_MANY_ROWS`); OpenAPI atualizado.
**Where**: `analytics/src/databoard_analytics/api/v1/imports.py`
**Depends on**: T2
**Reuses**: `InternalTokenMiddleware` (F00)
**Requirement**: IMP-01, IMP-03, IMP-07

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `tests/api/test_imports.py`: feliz nos dois endpoints; sem token → 401; arquivo corrompido → 422 com mensagem; `make contract` atualizado

**Tests**: integration
**Gate**: full
**Commit**: `feat(analytics): expose spreadsheet inspect and normalize endpoints`

---

### Phase 2: Arquivos

#### T4: Migração de arquivos e importações

**What**: `V10__importing.sql` com `stored_file` e `import_job`.
**Where**: `backend/src/main/resources/db/migration/V10__importing.sql`
**Depends on**: None
**Reuses**: —
**Requirement**: IMP-08, IMP-09
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ImportingSchemaIT`: inserção e FK `ON DELETE SET NULL` do arquivo funcionam

**Tests**: integration
**Gate**: full
**Commit**: `feat(importing): add stored files and import jobs schema`

---

#### T5: `SvgSanitizer`

**What**: Sanitização de SVG por whitelist (remove `script`, `foreignObject`, atributos `on*`, `href` externos).
**Where**: `backend/src/main/java/io/databoard/importing/files/SvgSanitizer.java`
**Depends on**: None
**Reuses**: —
**Requirement**: IMP-09

**Tools**: MCP: NONE · Skill: `security-and-hardening`

**Done when**:
- [ ] Testes com payloads: `<script>`, `onload=`, `<use href="http://...">`, `javascript:` → removidos; SVG simples preservado

**Tests**: unit
**Gate**: quick
**Commit**: `feat(importing): sanitize uploaded svg files`

---

#### T6: `FileStore`

**What**: Gravação no volume com caminho por data, SHA-256, MIME real via Tika, whitelist e limite de 20 MB, `link` a recurso dono, `open`, `delete`.
**Where**: `backend/src/main/java/io/databoard/importing/files/FileStore.java`
**Depends on**: T4, T5
**Reuses**: Apache Tika
**Requirement**: IMP-09

**Tools**: MCP: NONE · Skill: `java-springboot`, `security-and-hardening`

**Done when**:
- [ ] `FileStoreIT`: PNG aceito; ZIP renomeado para `.xlsx` sem estrutura OOXML → `FILE_REJECTED`; 21 MB → rejeitado; SVG gravado sanitizado

**Tests**: integration
**Gate**: full
**Commit**: `feat(importing): store files with real mime validation`

---

#### T7: `FileController` + `FileAccessPolicy`

**What**: `POST /files` e `GET /files/{id}` com `nosniff`, CSP restritiva e regra de acesso (quem enviou ou acesso ao dono; senão 404).
**Where**: `backend/src/main/java/io/databoard/importing/files/FileController.java`
**Depends on**: T6
**Reuses**: `AccessControl` (F04)
**Requirement**: IMP-09

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `FileControllerIT`: upload 201 com `{fileId,url}`; dono acessa; terceiro sem acesso ao recurso → 404; com `DATA_READ` na UDT dona → 200

**Tests**: integration
**Gate**: full
**Commit**: `feat(importing): expose file upload and download`

---

#### T8: Tipos de campo `FILE` e `IMAGE`

**What**: Registrar os tipos na F09 (coluna `uuid` + FK para `stored_file`), validação no `RecordValidator` e `link` ao gravar.
**Where**: `backend/src/main/java/io/databoard/udt/schema/FileFieldTypeSupport.java`
**Depends on**: T6
**Reuses**: `FieldTypeRegistry`, `RecordValidator` (F09), `FileStore` (T6)
**Requirement**: IMP-10

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `FileFieldIT`: campo IMAGE aceita PNG e rejeita PDF; `fileId` de outro usuário não vinculado → erro no campo; após gravar, arquivo vinculado à UDT

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): add file and image field types`

---

### Phase 3: Importação no backend

#### T9: `DataChangeBulkRecorder` na auditoria

**What**: API síncrona em `audit.api` que grava `audit.data_change` em lotes de 1.000, usada por importações.
**Where**: `backend/src/main/java/io/databoard/audit/api/DataChangeBulkRecorder.java`
**Depends on**: None
**Reuses**: `AuditMapper.diff` (F06)
**Requirement**: IMP-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `DataChangeBulkRecorderIT`: 5.000 alterações gravadas; histórico do registro (F06) as devolve

**Tests**: integration
**Gate**: full
**Commit**: `feat(audit): record data changes in bulk`

---

#### T10: Upload de importação

**What**: `POST /imports/upload`: guarda o original, chama `inspect`, cria `import_job` `UPLOADED`; exige `UPLOAD`; apaga o original se o analytics falhar.
**Where**: `backend/src/main/java/io/databoard/importing/ImportController.java`
**Depends on**: None
**Reuses**: `FileStore` (T6), `AnalyticsClient` (F00)
**Requirement**: IMP-01, IMP-07

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ImportUploadIT` (analytics real): resposta com abas e tipos; sem `UPLOAD` → 403; analytics parado → 503 e nenhum arquivo restante

**Tests**: integration
**Gate**: full
**Commit**: `feat(importing): upload and inspect spreadsheets`

---

#### T11: `ImportRowValidator`

**What**: Conversão de linha normalizada para tipos de campo, validação via `RecordValidator`, resolução de referências (normalização de rótulo, `REFERENCE_NOT_FOUND`, `AMBIGUOUS_REFERENCE`) e chave de UPSERT.
**Where**: `backend/src/main/java/io/databoard/importing/ImportRowValidator.java`
**Depends on**: None
**Reuses**: `RecordValidator` (F09)
**Requirement**: IMP-04, IMP-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: " ana silva " casa "Ana Silva"; nome inexistente → `REFERENCE_NOT_FOUND` com linha; dois "João" → `AMBIGUOUS_REFERENCE`; chave composta montada na ordem dos campos

**Tests**: unit
**Gate**: quick
**Commit**: `feat(importing): validate rows and resolve references`

---

#### T12: Analyze

**What**: `POST /imports/{id}/analyze`: chama `normalize`, guarda o CSV normalizado, valida todas as linhas, gera `errors.csv`, devolve contagens + 1.000 erros; destino "nova tabela" valida a especificação de campos.
**Where**: `backend/src/main/java/io/databoard/importing/ImportAnalyzer.java`
**Depends on**: T10, T11
**Reuses**: `FileStore` (T6)
**Requirement**: IMP-02, IMP-03, IMP-04, IMP-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ImportAnalyzeIT`: CA-11.1 (nome inexistente aparece com a linha); `errorsCsvUrl` baixável; aba vazia → `IMPORT_EMPTY_SHEET`

**Tests**: integration
**Gate**: full
**Commit**: `feat(importing): analyze imports with error report`

---

#### T13: Execução assíncrona

**What**: `POST /imports/{id}/execute` → job com lotes de 1.000, modos INSERT/UPSERT/REPLACE, atômico × `onlyValid`, criação da tabela nova, progresso, histórico em lote e evento resumo.
**Where**: `backend/src/main/java/io/databoard/importing/ImportExecutor.java`
**Depends on**: T12
**Reuses**: `UdtRecordService` (F09), `UdtSchemaService` (F09), `DataChangeBulkRecorder` (T9), `JobRunRecorder`
**Requirement**: IMP-02, IMP-06, IMP-07

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ImportExecuteIT`: CA-11.2 (UPSERT duas vezes atualiza sem duplicar); atômico com 1 erro → nada gravado, `FAILED`; `onlyValid` grava só válidas; REPLACE sem confirmação → 400 e sem `DATA_DELETE` → 403; segundo execute → 409; destino novo cria a UDT

**Tests**: integration
**Gate**: full
**Commit**: `feat(importing): execute imports asynchronously`

---

#### T14: Histórico, original e retenção

**What**: `GET /imports`, `GET /imports/{id}` (status/progresso), `GET /imports/{id}/original` e `ImportRetentionJob` diário.
**Where**: `backend/src/main/java/io/databoard/importing/ImportRetentionJob.java`
**Depends on**: T13
**Reuses**: `FileStore` (T6)
**Requirement**: IMP-08

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ImportHistoryIT`: lista com contagens; original para o autor e ADMIN, 403 para outros; após 31 dias (relógio controlado) arquivos removidos e histórico mantido

**Tests**: integration
**Gate**: full
**Commit**: `feat(importing): list imports and purge old files`

---

### Phase 4: Frontend

#### T15: `ImportWizard` + passo Upload

**What**: Shell do Stepper e passo de upload (drag-and-drop, limite de 20 MB no cliente, escolha da aba).
**Where**: `frontend/src/features/import/ImportWizard.tsx`
**Depends on**: None
**Reuses**: Mantine `Dropzone`, `Stepper`
**Requirement**: IMP-01
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: arquivo de 21 MB bloqueado no cliente; `FILE_REJECTED` exibe motivo; abas listadas após upload

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add import wizard upload step`

---

#### T16: Passo Destino

**What**: Escolha entre tabela existente (com permissão) e nova tabela com campos inferidos editáveis (nome, tipo).
**Where**: `frontend/src/features/import/steps/DestinationStep.tsx`
**Depends on**: T15
**Reuses**: —
**Requirement**: IMP-02

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: tipos inferidos pré-preenchidos e editáveis; tabelas sem `DATA_INSERT` não aparecem

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add import destination step`

---

#### T17: Passo Mapeamento

**What**: Coluna → campo (auto-match por nome normalizado), formato de data e separador decimal por coluna.
**Where**: `frontend/src/features/import/steps/MappingStep.tsx`
**Depends on**: T16
**Reuses**: —
**Requirement**: IMP-03, IMP-04

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: "Vendedor" mapeia automaticamente para o campo `vendedor`; campo obrigatório não mapeado bloqueia avançar

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add import mapping step`

---

#### T18: Passo Modo

**What**: INSERT, UPSERT (escolha de chaves) e REPLACE (confirmação digitando o nome da tabela; oculto sem `DATA_DELETE`).
**Where**: `frontend/src/features/import/steps/ModeStep.tsx`
**Depends on**: T17
**Reuses**: —
**Requirement**: IMP-06

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: UPSERT exige ao menos uma chave; REPLACE só habilita com o nome digitado corretamente

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add import mode step`

---

#### T19: Passo Pré-validação

**What**: Contagens, tabela dos erros (linha, coluna, mensagem), botão "Baixar CSV de erros", opção "importar só as válidas".
**Where**: `frontend/src/features/import/steps/ValidationStep.tsx`
**Depends on**: T18
**Reuses**: TanStack Table
**Requirement**: IMP-05

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: com erros e sem "só válidas" → botão Importar desabilitado; link do CSV presente

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add import validation step`

---

#### T20: Passo Execução

**What**: Dispara `execute`, acompanha `progress` por polling a cada 1 s, mostra resultado final ou falha e link para o Browser.
**Where**: `frontend/src/features/import/steps/ExecutionStep.tsx`
**Depends on**: T19
**Reuses**: TanStack Query `refetchInterval`
**Requirement**: IMP-06

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: barra avança com respostas do polling; `FAILED` mostra mensagem; polling para em `SUCCESS`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add import execution step with progress`

---

#### T21: `ImportHistoryPage`

**What**: Lista de importações com status, contagens e download do original.
**Where**: `frontend/src/features/import/ImportHistoryPage.tsx`
**Depends on**: None
**Reuses**: TanStack Table
**Requirement**: IMP-08

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: renderiza contagens; original expirado sem link

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add import history page`

---

#### T22: `FileCell` e `ImageCell`

**What**: Células do Browser para FILE/IMAGE: upload via `/files`, miniatura para imagem, link de download.
**Where**: `frontend/src/features/browser/cells/FileCell.tsx`
**Depends on**: None
**Reuses**: `EditableCell` (F10)
**Requirement**: IMP-10

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: upload seguido de `PATCH` com o `fileId`; imagem renderiza miniatura

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add file and image cells`

---

### Phase 5: E2E

#### T23: E2E de importação

**What**: Importar planilha de metas com nomes de vendedores (um inexistente aparece nos erros); importar duas vezes em UPSERT `vendedor+mes` com um valor alterado e conferir no Browser que não duplicou.
**Where**: `frontend/e2e/import.spec.ts`
**Depends on**: None
**Reuses**: Fixtures de UDT (F09)
**Requirement**: IMP-04, IMP-06

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde (CA-11.1, CA-11.2)

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): cover spreadsheet import with references and upsert`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1 | None | início | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T3 | T2 | T2 → T3 | ✅ |
| T4, T5 | None | inícios da fase 2 | ✅ |
| T6 | T4, T5 | T4 → T6, T5 → T6 | ✅ |
| T7 | T6 | T6 → T7 | ✅ |
| T8 | T6 | T6 → T8 | ✅ |
| T9, T10, T11 | None | inícios da fase 3 | ✅ |
| T12 | T10, T11 | T10 → T12, T11 → T12 | ✅ |
| T13 | T12 | T12 → T13 | ✅ |
| T14 | T13 | T13 → T14 | ✅ |
| T15, T21, T22 | None | inícios da fase 4 | ✅ |
| T16–T20 | anterior na cadeia | T15 → … → T20 | ✅ |
| T23 | None | fase 5 isolada | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1, T2 | analytics transformações | unit | unit | ✅ |
| T3 | analytics rotas | integration | integration | ✅ |
| T5, T11 | Backend domínio | unit | unit | ✅ |
| T4, T6–T10, T12–T14 | Backend persistência/controller/job | integration | integration | ✅ |
| T15–T22 | Frontend | unit | unit | ✅ |
| T23 | Ponta a ponta | e2e | e2e | ✅ |
