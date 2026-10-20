# F07 — Conexões de Dados Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F07-conexoes/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio (validador SQL, dialetos) | unit | Tabela de SQLs aceitos/rejeitados; URLs por dialeto | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend persistência/controller/registry | integration (Testcontainers Postgres, MySQL, SQL Server) | Feliz + erro por dialeto | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Formulário, teste, explorador | `frontend/src/**/*.test.tsx` | `make test-frontend` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-backend` ou `make test-frontend` |
| Full | Integração | `make test-backend it-backend` |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Núcleo de conexão

```
T1 → T4
T2 → T3 → T4 → T5
```

### Phase 2: Serviço, API e explorador

```
T6 → T7
T6 → T8
T9
```

### Phase 3: Frontend

```
T10 → T11
T12
```

---

## Task Breakdown

### Phase 1: Núcleo de conexão

#### T1: Migração de conexões e role somente leitura

**What**: `V11__datasource.sql` com `datasource`, conexão interna, `datasource_schema_cache` e role `databoard_ro`; `ReadViewGenerator` (F09) passa a conceder `SELECT` nas views.
**Where**: `backend/src/main/resources/db/migration/V11__datasource.sql`
**Depends on**: None
**Reuses**: `ReadViewGenerator` (F09)
**Requirement**: DS-01, DS-07
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `DatasourceSchemaIT`: conexão interna existe; com `SET ROLE databoard_ro`, `SELECT` em `udt.v_x` funciona e em `app.user_account` falha com permissão negada; `INSERT` em `lake` falha

**Tests**: integration
**Gate**: full
**Commit**: `feat(datasource): add datasource schema and read-only role`

---

#### T2: Dialetos SQL

**What**: `SqlDialect` + `PostgresDialect`, `MySqlDialect`, `SqlServerDialect` (URL, driver, whitelist de parâmetros, versão, quoting, limit/offset, date trunc).
**Where**: `backend/src/main/java/io/databoard/datasource/dialect/SqlDialect.java`
**Depends on**: None
**Reuses**: —
**Requirement**: DS-01, DS-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: URL com SSL e parâmetros por dialeto; `limitOffset` gera `LIMIT/OFFSET` (PG/MySQL) e `OFFSET ... FETCH` (SQL Server); `quote` escapa o caractere de quoting; `allowLoadLocalInfile` não está na whitelist do MySQL

**Tests**: unit
**Gate**: quick
**Commit**: `feat(datasource): add sql dialects for postgres, mysql and sql server`

---

#### T3: `ConnectionTester`

**What**: Teste com timeout de 5 s, latência, versão e classificação de erro.
**Where**: `backend/src/main/java/io/databoard/datasource/ConnectionTester.java`
**Depends on**: T2
**Reuses**: `SqlDialect` (T2)
**Requirement**: DS-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ConnectionTesterIT` com os 3 bancos: sucesso com versão em < 5 s (CA-07.1 para SQL Server); senha errada → `AUTH_FAILED`; porta fechada → `HOST_UNREACHABLE`

**Tests**: integration
**Gate**: full
**Commit**: `feat(datasource): test connections with error classification`

---

#### T4: `DataSourceRegistry`

**What**: Pools Hikari sob demanda (`readOnly`, limite, timeout), pool da conexão interna com `SET ROLE`, `evict` e fechamento por ociosidade.
**Where**: `backend/src/main/java/io/databoard/datasource/DataSourceRegistry.java`
**Depends on**: T1, T3
**Reuses**: `CryptoService` (F01)
**Requirement**: DS-07, DS-09

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `DataSourceRegistryIT`: `jdbc(id)` reutiliza o pool; `evict` fecha; consulta pela interna lê `udt.v_*` e falha em `app.*`; `UPDATE` numa externa via registry falha por `readOnly`; pool respeita `pool_max`

**Tests**: integration
**Gate**: full
**Commit**: `feat(datasource): manage per-datasource read-only pools`

---

#### T5: `SqlSafetyValidator`

**What**: Validação somente-SELECT com JSqlParser.
**Where**: `backend/src/main/java/io/databoard/datasource/SqlSafetyValidator.java`
**Depends on**: T4
**Reuses**: —
**Requirement**: DS-08

**Tools**: MCP: NONE · Skill: `java-springboot`, `security-and-hardening`

**Done when**:
- [ ] Teste de tabela: `SELECT`, `WITH ... SELECT`, `UNION` aceitos; `DELETE FROM x` (CA-07.2), `SELECT 1; DROP TABLE x`, `SELECT * INTO t FROM x`, `SELECT ... FOR UPDATE`, `CREATE`, texto inválido → `ONLY_SELECT_ALLOWED`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(datasource): validate select-only sql`

---

### Phase 2: Serviço, API e explorador

#### T6: `DataSourceService`

**What**: CRUD com cifragem, máscara, preservação de senha, whitelist de parâmetros, `evict` no update, `DATASOURCE_IN_USE`, proteção da interna, registro do recurso.
**Where**: `backend/src/main/java/io/databoard/datasource/DataSourceService.java`
**Depends on**: None
**Reuses**: `DependencyRegistry` (F13), `AccessControl` (F04), `DataSourceRegistry` (T4)
**Requirement**: DS-01, DS-02, DS-04, DS-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: senha `"********"` preserva a anterior; parâmetro fora da whitelist → erro; dependente → `DATASOURCE_IN_USE`; interna → `INTERNAL_DATASOURCE_READONLY`; update chama `evict`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(datasource): implement datasource management service`

---

#### T7: `DataSourceController`

**What**: `GET|POST /datasources`, `GET|PUT|DELETE /datasources/{id}`, `POST /datasources/test` (ADMIN para escrita e teste; listagem filtrada por `VIEW`).
**Where**: `backend/src/main/java/io/databoard/datasource/DataSourceController.java`
**Depends on**: T6
**Reuses**: `ConnectionTester` (T3)
**Requirement**: DS-01, DS-02, DS-03, DS-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `DataSourceControllerIT`: criar/listar/atualizar com senha mascarada; BUILDER em POST → 403; BUILDER lista só conexões com `VIEW`; `/test` sem salvar não cria registro

**Tests**: integration
**Gate**: full
**Commit**: `feat(datasource): expose datasource endpoints`

---

#### T8: `SchemaExplorer`

**What**: `GET /datasources/{id}/schema?refresh=` com cache, limite de 5.000 tabelas e filtro da conexão interna.
**Where**: `backend/src/main/java/io/databoard/datasource/SchemaExplorer.java`
**Depends on**: T6
**Reuses**: `DataSourceRegistry` (T4)
**Requirement**: DS-06, DS-07

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SchemaExplorerIT`: Postgres com 2 schemas listado; segunda chamada vem do cache (sem consultar a origem); `refresh=true` vê tabela nova; interna mostra só `udt.v_*` com rótulo e `lake`; sem `VIEW` → 403

**Tests**: integration
**Gate**: full
**Commit**: `feat(datasource): explore and cache source schemas`

---

#### T9: Testes de integração multi-dialeto

**What**: Suíte parametrizada (Postgres, MySQL, SQL Server com tag `mssql`) cobrindo teste, consulta somente leitura e explorador.
**Where**: `backend/src/test/java/io/databoard/datasource/MultiDialectIT.java`
**Depends on**: None
**Reuses**: Testcontainers
**Requirement**: DS-01, DS-03, DS-09

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `MultiDialectIT` verde nos 3 bancos em CI amd64

**Tests**: integration
**Gate**: full
**Commit**: `test(datasource): cover all mvp dialects end to end`

---

### Phase 3: Frontend

#### T10: `DataSourcesPage`

**What**: Lista de conexões (tipo, modo, criado por) e ações; conexão interna marcada como "Sistema".
**Where**: `frontend/src/features/admin/datasources/DataSourcesPage.tsx`
**Depends on**: None
**Reuses**: TanStack Table, `DependentsModal` (F09)
**Requirement**: DS-01, DS-04
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: interna sem editar/excluir; exclusão com 409 abre `DependentsModal`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add datasources admin page`

---

#### T11: `DataSourceForm` com teste

**What**: Formulário por tipo (porta padrão por dialeto), parâmetros extras, senha mascarada, botão "Testar conexão" com resultado e ACL.
**Where**: `frontend/src/features/admin/datasources/DataSourceForm.tsx`
**Depends on**: T10
**Reuses**: `AclEditor` (F04)
**Requirement**: DS-01, DS-02, DS-03, DS-05

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: trocar tipo para SQL Server ajusta a porta para 1433; teste com `success:false` mostra a causa; editar sem tocar na senha envia `"********"`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add datasource form with connection test`

---

#### T12: `SchemaExplorer` reutilizável

**What**: Árvore schema → tabela → colunas com tipo, busca e "Atualizar metadados"; callback `onPick(table|column)`.
**Where**: `frontend/src/shared/schema/SchemaExplorer.tsx`
**Depends on**: None
**Reuses**: Mantine `Tree`
**Requirement**: DS-06

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: busca filtra tabelas; "Atualizar" chama com `refresh=true`; `onPick` emite a coluna clicada

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add reusable schema explorer`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T2 | None | inícios da fase 1 | ✅ |
| T3 | T2 | T2 → T3 | ✅ |
| T4 | T1, T3 | T1 → T4, T3 → T4 | ✅ |
| T5 | T4 | T4 → T5 | ✅ |
| T6, T9 | None | inícios da fase 2 | ✅ |
| T7 | T6 | T6 → T7 | ✅ |
| T8 | T6 | T6 → T8 | ✅ |
| T10, T12 | None | inícios da fase 3 | ✅ |
| T11 | T10 | T10 → T11 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T2, T5, T6 | Backend domínio | unit | unit | ✅ |
| T1, T3, T4, T7, T8, T9 | Backend persistência/controller/registry | integration | integration | ✅ |
| T10–T12 | Frontend | unit | unit | ✅ |
