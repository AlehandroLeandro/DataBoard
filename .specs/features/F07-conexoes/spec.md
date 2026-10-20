# F07 — Conexões de Dados Specification

**Prioridade:** MVP (PostgreSQL, MySQL/MariaDB, SQL Server, conexão interna) / V1 (Oracle, Firebird, drivers por volume) / V2 (SQLite, ClickHouse, BigQuery) · **Depende de:** F04 · **Origem:** F07 do `produto.specs.md`

## Problem Statement

Os dados operacionais das PMEs estão no ERP, no CRM e em sistemas legados. Para criar visões sobre eles, o admin precisa cadastrar conexões com segurança: credenciais cifradas, somente leitura, pools limitados para não derrubar o ERP e exploração de schema. A plataforma também precisa de uma conexão interna para consultar as tabelas personalizadas e o lake.

## Goals

- [ ] Admin cadastra e testa uma conexão SQL Server e vê sucesso e versão em < 5 s (CA-07.1).
- [ ] Nenhuma consulta de usuário numa conexão `LIVE` executa algo além de `SELECT`/`WITH` (CA-07.2).
- [ ] Construtores navegam schemas, tabelas e colunas das conexões a que têm acesso.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Oracle, Firebird e drivers montados por volume | V1 (RF-07.1) |
| SQLite, ClickHouse, BigQuery | V2 |
| Cargas ETL | F08 (V1); o modo `ETL_ONLY` já é cadastrável |
| Criação de visões e execução de consultas | F12 / F17 (usam o validador e o pool daqui) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Quem cria e edita conexões | Apenas ADMIN | Persona "Administrador configura conexões" (§0.2) | y |
| Quem usa uma conexão | Quem tem `VIEW` no recurso `DATASOURCE` pode criar visões e usar o Editor SQL sobre ela (sujeito às flags) | ACL por recurso (F04) | y |
| Pool | HikariCP por conexão, `maximumPoolSize` padrão 5 (máx. 20), `connectionTimeout` 5 s, `readOnly=true`, pool criado sob demanda e fechado após 15 min ocioso | RN-07.2 | y |
| Teste de conexão | Timeout de login 5 s; executa a consulta de versão do dialeto (`SELECT version()`, `SELECT @@VERSION`) | RF-07.3 | y |
| Cache de schema | Tabela `app.datasource_schema_cache` com o JSON do explorador; `?refresh=true` recarrega via `DatabaseMetaData`; sem expiração automática | RF-07.4 | y |
| Conexão interna | ID fixo `00000000-0000-0000-0000-000000000001`, nome `internal`, não editável nem excluível, visível a quem tem papel BUILDER ou ADMIN | RF-07.6 | y |
| Somente leitura da conexão interna | Pool próprio com `SET ROLE databoard_ro` e `default_transaction_read_only=on`; o role só tem `USAGE` em `udt`/`lake` e `SELECT` nas views `udt.v_*` e tabelas `lake.*` | RN-17.2: schema `app` nunca acessível | y |
| Validação somente-SELECT | JSqlParser: exatamente um statement, do tipo `Select` (inclui `WITH`); `SELECT ... INTO` e `FOR UPDATE` são rejeitados | RN-07.1 | y |
| Parâmetros JDBC extras | Pares chave=valor em whitelist por dialeto (ex.: `encrypt`, `trustServerCertificate`, `sslmode`, `useSSL`, `serverTimezone`, `applicationName`) | Evita parâmetros perigosos (ex.: `allowLoadLocalInfile`, `socketFactory`) | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Cadastrar e testar conexões ⭐ MVP

**User Story**: Como admin, quero cadastrar conexões com PostgreSQL, MySQL/MariaDB e SQL Server e testá-las antes de salvar.

**Why P1**: Sem conexão, não há dados externos.

**Acceptance Criteria**:
1. WHEN o admin chama `POST /api/v1/datasources` com nome, tipo (`POSTGRES`, `MYSQL`, `SQLSERVER`), host, porta, banco, usuário, senha, parâmetros extras, SSL, timezone da origem, pool máximo e modo (`LIVE`/`ETL_ONLY`) THEN o sistema SHALL salvar com a senha cifrada (`CryptoService`), registrar o recurso `DATASOURCE` e responder 201.
2. WHEN `POST /api/v1/datasources/test` recebe os dados de uma conexão válida THEN o sistema SHALL responder `{ success: true, latencyMs, serverVersion }` sem salvar nada, em até 5 s.
3. IF o teste falha por credencial, host inacessível ou timeout de 5 s THEN o sistema SHALL responder 200 `{ success: false, errorCode, message }` com `errorCode` ∈ `AUTH_FAILED`, `HOST_UNREACHABLE`, `TIMEOUT`, `UNKNOWN`.
4. The API SHALL devolver a senha sempre como `"********"`.
5. WHEN `PUT /api/v1/datasources/{id}` recebe a senha vazia ou `"********"` THEN o sistema SHALL manter a senha atual.
6. WHEN uma conexão é atualizada THEN o sistema SHALL fechar e recriar o pool dela.
7. IF um parâmetro JDBC extra não está na whitelist do dialeto THEN o sistema SHALL responder 400 com erro em `extraParams.<chave>`.
8. IF um não-admin chama `POST`, `PUT` ou `DELETE` em `/api/v1/datasources` THEN o sistema SHALL responder 403.
9. IF a conexão é usada por visões ou cargas WHEN o admin a exclui THEN o sistema SHALL responder 409 com `code=DATASOURCE_IN_USE` e a lista de dependentes.

**Independent Test**: IT com Testcontainers dos três bancos: testar, salvar, ver a senha mascarada, atualizar sem senha.

---

### P1: Explorar schema ⭐ MVP

**User Story**: Como construtor, quero ver schemas, tabelas e colunas com tipos para montar visões e consultas.

**Why P1**: Usado por F12 e F17.

**Acceptance Criteria**:
1. WHEN `GET /api/v1/datasources/{id}/schema` é chamado por quem tem `VIEW` na conexão THEN o sistema SHALL devolver `[{ schema, tables: [{ name, type, columns: [{ name, type, nullable }] }] }]` do cache, carregando-o na primeira chamada.
2. WHEN `?refresh=true` é informado THEN o sistema SHALL recarregar os metadados da origem e atualizar o cache.
3. WHEN o schema da conexão interna é consultado THEN o sistema SHALL listar apenas `udt` (views `v_*` com o rótulo da UDT) e `lake`, nunca `app`, `audit` ou tabelas `udt.t_*`.
4. IF quem chama não tem `VIEW` na conexão THEN o sistema SHALL responder 403.

**Independent Test**: IT sobre Postgres com 2 schemas e sobre a conexão interna.

---

### P1: Execução segura de consultas ⭐ MVP

**User Story**: Como admin, quero garantir que consultas de usuários nunca alterem dados nem sobrecarreguem o banco de origem.

**Why P1**: RN-07.1/RN-07.2; base de F12 e F17.

**Acceptance Criteria**:
1. WHEN `SqlSafetyValidator.validateSelectOnly(sql)` recebe `SELECT` ou `WITH ... SELECT` THEN o validador SHALL aceitar.
2. IF o SQL contém `INSERT`, `UPDATE`, `DELETE`, `MERGE`, DDL, `SELECT ... INTO`, `FOR UPDATE`, mais de um statement ou não pode ser analisado THEN o validador SHALL rejeitar com `code=ONLY_SELECT_ALLOWED`.
3. The `DataSourceRegistry` SHALL abrir conexões das fontes externas com `readOnly=true` e tamanho de pool limitado ao configurado (máx. 20).
4. WHEN uma consulta é executada pela conexão interna THEN a sessão SHALL estar com o role `databoard_ro` e `default_transaction_read_only=on`.
5. IF uma consulta pela conexão interna tenta ler `app.user_account` THEN o banco SHALL negar com erro de permissão.

**Independent Test**: Testes unitários do validador com tabela de SQLs + IT de permissão do role.

---

## Edge Cases

- IF a porta estiver fora de 1–65535 ou o host vazio THEN o sistema SHALL responder 400.
- WHEN dois admins atualizam a mesma conexão ao mesmo tempo THEN o sistema SHALL aplicar a última gravação e recriar o pool uma vez por gravação.
- IF a chave mestra mudou e a senha não pode ser decifrada THEN o teste e o uso da conexão SHALL falhar com `CREDENTIAL_UNREADABLE` e a UI SHALL pedir para recadastrar a senha (RN-01.2).
- WHEN o explorador encontra mais de 5.000 tabelas THEN o sistema SHALL devolver as 5.000 primeiras por ordem alfabética com `truncated: true`.
- IF alguém tenta `PUT` ou `DELETE` na conexão interna THEN o sistema SHALL responder 409 com `code=INTERNAL_DATASOURCE_READONLY`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| DS-01 | P1: Cadastro (tipos e campos) | RF-07.1, RF-07.2, RF-07.5 | Tasks | Pending |
| DS-02 | P1: Cadastro (senha mascarada/cifrada) | RN-07.3, RN-01.2 | Tasks | Pending |
| DS-03 | P1: Teste de conexão | RF-07.3, CA-07.1 | Tasks | Pending |
| DS-04 | P1: Cadastro (exclusão em uso) | RN-07.4 | Tasks | Pending |
| DS-05 | P1: Cadastro (parâmetros extras) | RF-07.2 | Tasks | Pending |
| DS-06 | P1: Explorar schema | RF-07.4 | Tasks | Pending |
| DS-07 | P1: Conexão interna | RF-07.6, RN-17.2 | Tasks | Pending |
| DS-08 | P1: Validador somente-SELECT | RN-07.1, CA-07.2 | Tasks | Pending |
| DS-09 | P1: Pools limitados | RN-07.2 | Tasks | Pending |

**Coverage:** 9 total, 9 mapped to tasks, 0 unmapped.

---

## Success Criteria

- [ ] CA-07.1 e CA-07.2 automatizados.
- [ ] Teste de permissão prova que a conexão interna não lê o schema `app`.
