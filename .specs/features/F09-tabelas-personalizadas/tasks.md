# F09 — Tabelas Personalizadas Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F09-tabelas-personalizadas/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio (obexpr, validadores, geradores puros, grafo) | unit | Todos os ramos; 1:1 com ACs; payloads de injeção; edge cases | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend serviço com DDL/persistência, controllers, jobs | integration (Postgres real) | Rotas: feliz + edge + erro; DDL efetivamente aplicado | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Formulários por tipo, editor de fórmula, diálogos | `frontend/src/**/*.test.tsx` | `make test-frontend` |
| Ponta a ponta | e2e | CA-09.1 e CA-09.2 pela UI | `frontend/e2e/*.spec.ts` | `make e2e` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-backend` ou `make test-frontend` |
| Full | Integração/e2e | `make test-backend it-backend` (+ `make e2e`) |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Estrutura e campos armazenados

```
T1 → T5
T2 → T6
T3 → T4 → T5 → T6 → T7
```

### Phase 2: Registros

```
T8 → T9 → T10 → T11
T9 → T12
```

### Phase 3: OBExpr

```
T13 → T14 → T16
T13 → T15 → T16
T16 → T17
T16 → T18
```

### Phase 4: Campos derivados

```
T19 → T20 → T21
T22
```

### Phase 5: Ciclo de vida da estrutura

```
T23
T24 → T25
T26
```

### Phase 6: Frontend — designer

```
T27 → T28 → T29 → T30
T29 → T31
T29 → T32
```

### Phase 7: Frontend — fórmula, ciclo de vida e E2E

```
T33 → T37
T34
T35 → T36
```

---

## Task Breakdown

### Phase 1: Estrutura e campos armazenados

#### T1: Migração de metadados de UDT

**What**: `V8__udt.sql` com `udt_table`, `udt_field`, `udt_unique`.
**Where**: `backend/src/main/resources/db/migration/V8__udt.sql`
**Depends on**: None
**Reuses**: —
**Requirement**: UDT-01
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `UdtMetaSchemaIT`: `api_name` duplicado ativo na mesma tabela viola índice; excluído logicamente não viola

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): add custom table metadata schema`

---

#### T2: `ApiNameGenerator`

**What**: Geração `snake_case` sem acentos, reservados com sufixo `_campo`, colisões `_2`, `_3`, limite de 63 caracteres.
**Where**: `backend/src/main/java/io/databoard/udt/schema/ApiNameGenerator.java`
**Depends on**: None
**Reuses**: —
**Requirement**: UDT-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: "Preço Unitário" → `preco_unitario`; "ID" → `id_campo`; colisão → `preco_unitario_2`; rótulo de 100 caracteres → 63

**Tests**: unit
**Gate**: quick
**Commit**: `feat(udt): generate api names from labels`

---

#### T3: `FieldTypeRegistry`

**What**: Especificação por tipo (tipo Postgres, armazenado × derivado) e validação de `config` (opções de SELECT, casas 0–6, `maxLength`, `onDelete`, prefixo/padding do AUTONUMBER).
**Where**: `backend/src/main/java/io/databoard/udt/schema/FieldTypeRegistry.java`
**Depends on**: None
**Reuses**: —
**Requirement**: UDT-02, UDT-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: tabela tipo → tipo Postgres (inclusive `CURRENCY`→`numeric(18,2)`, `MULTI_SELECT`→`text[]`); DECIMAL com 7 casas → erro; SELECT sem opções → erro

**Tests**: unit
**Gate**: quick
**Commit**: `feat(udt): define field type registry`

---

#### T4: `DdlBuilder`

**What**: DDL de tabela (colunas de sistema), coluna, FK com `onDelete`, índice da FK, índice único (simples e composto), sequência AUTONUMBER.
**Where**: `backend/src/main/java/io/databoard/udt/schema/DdlBuilder.java`
**Depends on**: T3
**Reuses**: `FieldTypeRegistry` (T3)
**Requirement**: UDT-01, UDT-02, UDT-04, UDT-07, UDT-19

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes de snapshot do SQL gerado para cada operação; nenhum identificador vem de rótulo ou input (só `shortId`)

**Tests**: unit
**Gate**: quick
**Commit**: `feat(udt): build ddl from field metadata`

---

#### T5: `ReadViewGenerator` (armazenados)

**What**: `buildViewSql` para colunas de sistema e armazenadas (com alias = `apiName`) e `regenerate` com fecho de dependentes e drop/create em ordem topológica.
**Where**: `backend/src/main/java/io/databoard/udt/schema/ReadViewGenerator.java`
**Depends on**: T1, T4
**Reuses**: `DependencyRegistry` (F13)
**Requirement**: UDT-01, UDT-23

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ReadViewGeneratorIT`: view criada com colunas em `apiName`; regenerar após novo campo inclui a coluna; `lock_timeout` excedido → `UDT_SCHEMA_BUSY`

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): generate read views for custom tables`

---

#### T6: `UdtSchemaService` — tabelas, campos armazenados, referências e unicidade

**What**: Criar tabela, adicionar/atualizar/renomear/reordenar campo, referência N:1 com inversa, unicidade composta, campo de exibição, limites, registro de recurso/categoria, lock por tabela e `UdtSchemaChanged` após commit.
**Where**: `backend/src/main/java/io/databoard/udt/schema/UdtSchemaService.java`
**Depends on**: T2, T5
**Reuses**: `AccessControl.registerResource` (F04), `CategoryAssignments` (F13), `LimitsProvider` (F02)
**Requirement**: UDT-01, UDT-02, UDT-03, UDT-04, UDT-05, UDT-07, UDT-23

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `UdtSchemaServiceIT`: tabela física criada; DDL com erro forçado → metadado também revertido (`UDT_DDL_FAILED`); renomear não executa DDL; FK criada com `RESTRICT/SET_NULL/CASCADE`; inversa listada na tabela alvo; 201º campo → `UDT_LIMIT_FIELDS`; `apiName` de campo com dependente → `UDT_FIELD_IN_USE`; evento publicado só após commit

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): implement custom table schema service`

---

#### T7: Controllers de tabela e campo

**What**: `GET|POST /udt/tables`, `GET|PUT|DELETE /udt/tables/{id}` (DELETE delega à T24 depois; aqui responde 501 até lá), `POST /fields`, `PUT /fields/{id}`, com `EDIT` + `MANAGE_UDT_SCHEMA`.
**Where**: `backend/src/main/java/io/databoard/udt/schema/UdtTableController.java`
**Depends on**: T6
**Reuses**: `@RequiresFlag`, `AccessControl` (F04)
**Requirement**: UDT-01, UDT-02, UDT-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `UdtTableControllerIT`: criação 201 com `MANAGE` para o criador; sem flag → 403; com flag e sem `EDIT` na tabela → 403; VIEWER → 403; listagem filtra por `filterVisible` e `categoryId`

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): expose table and field endpoints`

---

### Phase 2: Registros

#### T8: `RecordValidator`

**What**: Validação por campo (tipo, obrigatório, min/max, regex, EMAIL/URL/PHONE, opções, `READ_ONLY_FIELD`) nos modos INSERT e PATCH.
**Where**: `backend/src/main/java/io/databoard/udt/data/RecordValidator.java`
**Depends on**: None
**Reuses**: `FieldTypeRegistry` (T3)
**Requirement**: UDT-09

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: um caso por regra; vários erros no mesmo payload → vários itens; PATCH sem campo obrigatório não acusa; PATCH com `null` em obrigatório acusa; FORMULA no payload → `READ_ONLY_FIELD`; MULTI_SELECT com opção inválida → `INVALID_OPTION`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(udt): validate record values by field rules`

---

#### T9: `UdtRecordService` — create, patch, delete, get, list

**What**: Escrita em `t_*` e leitura de `v_*`, versão otimista, permissões `DATA_*`, "só edita os próprios", mapeamento de `23505`/`23503`, `UdtRecordChanged`.
**Where**: `backend/src/main/java/io/databoard/udt/data/UdtRecordService.java`
**Depends on**: T8
**Reuses**: `AccessControl` (F04), `UdtRecordChanged` (F06)
**Requirement**: UDT-09, UDT-10, UDT-11

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `UdtRecordServiceIT`: CA-09.1 (segunda meta do mesmo vendedor/mês → `UNIQUE_VIOLATION` em `vendedor` e `mes`); versão errada → `RECORD_VERSION_CONFLICT` com registro atual; sem `DATA_INSERT` → 403; não dono com opção ativa → `UDT_NOT_RECORD_OWNER`; excluir referenciado com `RESTRICT` → `UDT_RECORD_REFERENCED`; evento com before/after

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): implement record crud with optimistic locking`

---

#### T10: Bulk atômico

**What**: `bulk(insert, update, delete)` numa transação, erros indexados, limite de 5.000 operações.
**Where**: `backend/src/main/java/io/databoard/udt/data/UdtBulkService.java`
**Depends on**: T9
**Reuses**: `UdtRecordService` (T9)
**Requirement**: UDT-12

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `UdtBulkIT`: 3 inserts com o 2º inválido → nenhum gravado e erro `insert[1].<campo>`; 5.001 operações → `BULK_TOO_LARGE`

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): add atomic bulk record operations`

---

#### T11: `UdtRecordController`

**What**: `GET|POST /records`, `GET|PATCH|DELETE /records/{id}`, `POST /records/bulk`, `GET /records/{id}/history`.
**Where**: `backend/src/main/java/io/databoard/udt/data/UdtRecordController.java`
**Depends on**: T10
**Reuses**: `AuditQuery.recordHistory` (F06), `PageResponse` (F00)
**Requirement**: UDT-09, UDT-12, UDT-13

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `UdtRecordControllerIT`: rotas felizes; `fields=a,b` restringe colunas; histórico após PATCH mostra campo, antigo e novo; 400 com `errors[]`; 409 de versão

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): expose record endpoints`

---

#### T12: Opções de referência

**What**: `GET /reference-options/{fieldId}?q=&page=` usando o campo de exibição (fallback `id`), respeitando `DATA_READ` na tabela alvo.
**Where**: `backend/src/main/java/io/databoard/udt/data/ReferenceOptionsController.java`
**Depends on**: T9
**Reuses**: `UdtRecordService` (T9)
**Requirement**: UDT-08

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ReferenceOptionsIT`: busca "ana" encontra "Mariana"; sem campo de exibição → rótulo = id; sem `DATA_READ` na alvo → 403

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): serve reference field options`

---

### Phase 3: OBExpr

#### T13: Gramática ANTLR e parser com posições

**What**: `OBExpr.g4`, plugin Maven, AST imutável com `start/end` e erros de sintaxe posicionados.
**Where**: `backend/src/main/antlr4/io/databoard/udt/obexpr/OBExpr.g4`
**Depends on**: None
**Reuses**: —
**Requirement**: UDT-14

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: precedência (`1 + 2 * 3`), `&`, `NOT`, chamadas aninhadas, `{a.b.c}`, strings com aspas escapadas; `{qtd} *` → erro com `start/end` no fim da expressão

**Tests**: unit
**Gate**: quick
**Commit**: `feat(udt): add obexpr grammar and parser`

---

#### T14: `FunctionCatalog` e `TypeChecker`

**What**: Catálogo das 29 funções do AC 1 (aridade, tipos, retorno) e inferência de tipo com erros posicionados.
**Where**: `backend/src/main/java/io/databoard/udt/obexpr/TypeChecker.java`
**Depends on**: T13
**Reuses**: —
**Requirement**: UDT-14

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: cada função com aridade certa/errada; `"a" + 1` → tipos incompatíveis; `IF({realizado} >= {meta}, "Atingida", "Abaixo")` → TEXT; `DATEDIFF` → INTEGER

**Tests**: unit
**Gate**: quick
**Commit**: `feat(udt): type-check obexpr expressions`

---

#### T15: `FieldScope` com navegação

**What**: Resolução de `{campo}` e `{ref.campo}` até `maxDepth`, produzindo caminhos de join e erro `UDT_LIMIT_NAVIGATION_DEPTH`.
**Where**: `backend/src/main/java/io/databoard/udt/obexpr/FieldScope.java`
**Depends on**: T13
**Reuses**: Metadados de UDT
**Requirement**: UDT-14

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: `{pedido.cliente.nome}` resolve 2 joins; 4 níveis → erro na posição; campo excluído → inexistente

**Tests**: unit
**Gate**: quick
**Commit**: `feat(udt): resolve field references with navigation`

---

#### T16: `SqlCompiler` (INLINE e BIND)

**What**: Compilação para SQL Postgres com escape de literais (INLINE) ou binds (BIND), `NULLIF` na divisão, mapeamento de funções.
**Where**: `backend/src/main/java/io/databoard/udt/obexpr/SqlCompiler.java`
**Depends on**: T14, T15
**Reuses**: —
**Requirement**: UDT-14

**Tools**: MCP: NONE · Skill: `java-springboot`, `security-and-hardening`

**Done when**:
- [ ] Testes de tabela expressão → SQL para os 3 exemplos do RF-09.6 e todas as funções; payload `"x'); DROP TABLE t;--"` em INLINE vira literal escapado; em BIND vira `?` com bind; divisão usa `NULLIF`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(udt): compile obexpr to postgres sql`

---

#### T17: Endpoint de validação com prévia

**What**: `POST /udt/expressions/validate` devolvendo `valid`, `resultType`, `errors[]` e prévia de 5 linhas.
**Where**: `backend/src/main/java/io/databoard/udt/obexpr/ExpressionController.java`
**Depends on**: T16
**Reuses**: `ReadViewGenerator` (T5) para compor a consulta de prévia
**Requirement**: UDT-14

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ExpressionControllerIT`: expressão válida → 5 valores calculados; erro de sintaxe → `start/end`; sem `EDIT` na tabela → 403

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): validate expressions with preview`

---

#### T18: Filtro OBExpr na listagem de registros

**What**: `?filter=` compilado em modo BIND como `WHERE`, com `FILTER_NOT_BOOLEAN`.
**Where**: `backend/src/main/java/io/databoard/udt/data/RecordFilterCompiler.java`
**Depends on**: T16
**Reuses**: `SqlCompiler` (T16)
**Requirement**: UDT-15

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `RecordFilterIT`: `{valor} > 100 AND {regiao} = "Sul"` filtra corretamente; `{valor} + 1` → 400 `FILTER_NOT_BOOLEAN`; filtro com injeção não altera a consulta

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): filter records with obexpr`

---

### Phase 4: Campos derivados

#### T19: `DerivedDependencyGraph`

**What**: Detecção de ciclo (DFS) atravessando referências, com caminho em rótulos.
**Where**: `backend/src/main/java/io/databoard/udt/schema/DerivedDependencyGraph.java`
**Depends on**: None
**Reuses**: —
**Requirement**: UDT-17

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: CA-09.3 (`B → A → B`); ciclo via referência entre duas tabelas; grafo acíclico passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat(udt): detect circular field dependencies`

---

#### T20: FORMULA e LOOKUP

**What**: Salvar FORMULA (inferência/forçar tipo) e LOOKUP no `UdtSchemaService`; `ReadViewGenerator` com expressões inline em ordem topológica e `LEFT JOIN` na view alvo; arestas de dependência.
**Where**: `backend/src/main/java/io/databoard/udt/schema/DerivedFieldHandler.java`
**Depends on**: T19
**Reuses**: `SqlCompiler` (T16), `ReadViewGenerator` (T5)
**Requirement**: UDT-16, UDT-17

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `DerivedFieldsIT`: alterar `qtd` → FORMULA `total` reflete; LOOKUP de campo derivado da alvo funciona; ciclo → 422 com `path`; mudança na tabela alvo regenera a view da tabela que faz lookup

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): support formula and lookup fields`

---

#### T21: ROLLUP

**What**: ROLLUP com as 6 funções, filtro OBExpr, `LATERAL` sobre a view filha, limite de 10 por tabela.
**Where**: `backend/src/main/java/io/databoard/udt/schema/RollupFieldHandler.java`
**Depends on**: T20
**Reuses**: `DerivedFieldHandler` (T20)
**Requirement**: UDT-18

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `RollupIT`: CA-09.2 (`Pedido.total = SUM(itens.total)` com `total` FORMULA); filtro `cancelado = false`; sem filhos → COUNT 0 e SUM `null`; 11º rollup → `UDT_LIMIT_ROLLUPS`

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): support rollup fields`

---

#### T22: Valor padrão por OBExpr

**What**: Padrão literal ou OBExpr sem referência a campos (`TODAY()`, `NOW()`) compilado em `ALTER COLUMN SET DEFAULT`.
**Where**: `backend/src/main/java/io/databoard/udt/schema/DefaultValueHandler.java`
**Depends on**: None
**Reuses**: `SqlCompiler` (T16), `DdlBuilder` (T4)
**Requirement**: UDT-19

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `DefaultValueIT`: `TODAY()` preenche a data na inserção sem valor; padrão com `{campo}` → erro de validação; AUTONUMBER incrementa

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): support default values and autonumber`

---

### Phase 5: Ciclo de vida da estrutura

#### T23: Mudança de tipo (simular e aplicar)

**What**: `POST /type-change/simulate` e `/type-change/apply` com `pg_input_is_valid`.
**Where**: `backend/src/main/java/io/databoard/udt/lifecycle/TypeChangeService.java`
**Depends on**: None
**Reuses**: `DdlBuilder` (T4), `ReadViewGenerator` (T5)
**Requirement**: UDT-20

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `TypeChangeIT`: CA-09.5 ("10","abc","20" → INTEGER: `failingRows=1`, amostra `["abc"]`); aplicar → "abc" vira `null`, demais convertidos; campo derivado → 400

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): simulate and apply field type changes`

---

#### T24: Exclusão com dependentes e restauração

**What**: `TrashService` (bloqueio por `dependentsOf`, exclusão lógica, restauração em 30 dias) + `DELETE` de tabela/campo + `TrashController` (`GET /udt/trash`, `POST /udt/trash/{type}/{id}/restore`).
**Where**: `backend/src/main/java/io/databoard/udt/lifecycle/TrashService.java`
**Depends on**: None
**Reuses**: `DependencyRegistry` (F13)
**Requirement**: UDT-21

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `TrashIT`: CA-09.4 (campo usado por fórmula e por dependente fictício `SCREEN_OBJECT` → 409 listando ambos); sem dependentes → some da API e da view; restaurar → volta com os dados; após 30 dias, restauração → 410 `UDT_TRASH_EXPIRED`

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): soft delete with dependency checks and restore`

---

#### T25: Job de limpeza da lixeira

**What**: Job diário às 02:30 que executa `DROP COLUMN`/`DROP TABLE` dos itens com mais de 30 dias e registra em `app.job_run`.
**Where**: `backend/src/main/java/io/databoard/udt/lifecycle/TrashPurgeJob.java`
**Depends on**: T24
**Reuses**: `JobRunRecorder` (AD-015; criar em `platform.jobs` se ainda não existir)
**Requirement**: UDT-21

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `TrashPurgeJobIT` (relógio controlado): item com 31 dias removido fisicamente; com 29 dias mantido; execução registrada

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): purge expired trash items`

---

#### T26: Duplicação e listagem de dependências

**What**: `POST /udt/tables/{id}/duplicate` e `GET /udt/tables/{id}/dependencies`.
**Where**: `backend/src/main/java/io/databoard/udt/lifecycle/TableDuplicationService.java`
**Depends on**: None
**Reuses**: `UdtSchemaService` (T6), `DependencyRegistry` (F13)
**Requirement**: UDT-22

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `TableDuplicationIT`: cópia com mesmos campos, validações e unicidades, sem registros e com novos `shortId`; dependências listadas por campo

**Tests**: integration
**Gate**: full
**Commit**: `feat(udt): duplicate tables and list dependencies`

---

### Phase 6: Frontend — designer

#### T27: `UdtListPage`

**What**: Lista de tabelas com busca, filtro por categoria e botão "Nova tabela" (visível com `MANAGE_UDT_SCHEMA`).
**Where**: `frontend/src/features/udt/UdtListPage.tsx`
**Depends on**: None
**Reuses**: `CategorySelect` (F13), `<Can>` (F04)
**Requirement**: UDT-01, UDT-06
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: sem flag → botão oculto; filtro de categoria vira query

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add custom tables list`

---

#### T28: `TableDesignerPage`

**What**: Designer com lista de campos (arrastar para reordenar), campo de exibição, unicidades compostas, "só edita os próprios" e ACL (`AclEditor`).
**Where**: `frontend/src/features/udt/TableDesignerPage.tsx`
**Depends on**: T27
**Reuses**: `AclEditor` (F04), `@dnd-kit/sortable`
**Requirement**: UDT-01, UDT-04

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `frontend-design`

**Done when**:
- [ ] Testes: reordenar envia as novas posições; criar unicidade `vendedor + mes` envia os ids; erro 422 de limite exibido

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add custom table designer`

---

#### T29: `FieldEditorDrawer` (tipos armazenados)

**What**: Formulário dinâmico por tipo para os 15 tipos armazenados (opções de SELECT com cor, casas, min/max, regex, obrigatório, único, padrão).
**Where**: `frontend/src/features/udt/fields/FieldEditorDrawer.tsx`
**Depends on**: T28
**Reuses**: Mantine `Drawer`, `useForm`
**Requirement**: UDT-02, UDT-04, UDT-19

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: trocar tipo mostra só as opções pertinentes; SELECT sem opção bloqueia salvar; erros do backend por campo

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add field editor for stored types`

---

#### T30: `ReferenceFieldForm`

**What**: Subformulário de REFERENCE (tabela alvo, `onDelete`, rótulo da inversa).
**Where**: `frontend/src/features/udt/fields/ReferenceFieldForm.tsx`
**Depends on**: T29
**Reuses**: —
**Requirement**: UDT-07

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: lista de tabelas alvo inclui a própria tabela; `onDelete` padrão `RESTRICT`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add reference field form`

---

#### T31: `LookupFieldForm`

**What**: Subformulário de LOOKUP (campo de referência → campo alvo).
**Where**: `frontend/src/features/udt/fields/LookupFieldForm.tsx`
**Depends on**: T29
**Reuses**: —
**Requirement**: UDT-16

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: campos alvo carregados conforme a referência escolhida

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add lookup field form`

---

#### T32: `RollupFieldForm`

**What**: Subformulário de ROLLUP (relação inversa, função, campo, filtro com mini editor OBExpr).
**Where**: `frontend/src/features/udt/fields/RollupFieldForm.tsx`
**Depends on**: T29
**Reuses**: —
**Requirement**: UDT-18

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: `COUNT` dispensa o campo; demais exigem; relações inversas vêm da API

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add rollup field form`

---

### Phase 7: Frontend — fórmula, ciclo de vida e E2E

#### T33: `FormulaEditor`

**What**: CodeMirror 6 com autocomplete de `{campos}` (inclui navegação) e funções, lint por `start/end` vindo do validate (debounce 300 ms), tipo resultante e prévia de 5 linhas.
**Where**: `frontend/src/features/udt/formula/FormulaEditor.tsx`
**Depends on**: None
**Reuses**: `@codemirror/autocomplete`, `@codemirror/lint`
**Requirement**: UDT-14, UDT-16

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: digitar `{qu` sugere `quantidade`; erro da API sublinha o trecho; prévia renderiza 5 valores; ciclo (422) mostra o caminho

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add formula editor with live validation`

---

#### T34: `TypeChangeDialog`

**What**: Diálogo que simula, mostra `failingRows` e amostra, e aplica após confirmação.
**Where**: `frontend/src/features/udt/fields/TypeChangeDialog.tsx`
**Depends on**: None
**Reuses**: —
**Requirement**: UDT-20

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: exibe "1 registro não pode ser convertido" e a amostra; "Converter" chama apply; "Cancelar" não chama

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add field type change dialog`

---

#### T35: `DependentsModal`

**What**: Modal genérico para respostas 409 com `dependents[]` (tipo, nome, link).
**Where**: `frontend/src/shared/dependencies/DependentsModal.tsx`
**Depends on**: None
**Reuses**: `ApiError` (F00)
**Requirement**: UDT-21

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: lista agrupada por tipo; link navega para o recurso

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add dependents modal`

---

#### T36: `UdtTrashPage`

**What**: Lista de tabelas e campos excluídos com dias restantes e ação "Restaurar".
**Where**: `frontend/src/features/udt/UdtTrashPage.tsx`
**Depends on**: T35
**Reuses**: TanStack Table
**Requirement**: UDT-21

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: dias restantes calculados; restaurar remove da lista

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add custom tables trash page`

---

#### T37: E2E de tabelas personalizadas

**What**: Pela UI: criar *Vendedor* e *Meta* com referência e unicidade composta; via API de records, inserir meta duplicada (erro claro); criar *Pedido*/*Item* com FORMULA e ROLLUP e conferir o total após alterar a quantidade.
**Where**: `frontend/e2e/udt.spec.ts`
**Depends on**: T33
**Reuses**: Fixtures de login (F03)
**Requirement**: UDT-04, UDT-16, UDT-18

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde (CA-09.1, CA-09.2)

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): cover custom tables with formulas and rollups`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T2, T3 | None | inícios da fase 1 | ✅ |
| T4 | T3 | T3 → T4 | ✅ |
| T5 | T1, T4 | T1 → T5, T4 → T5 | ✅ |
| T6 | T2, T5 | T2 → T6, T5 → T6 | ✅ |
| T7 | T6 | T6 → T7 | ✅ |
| T8 | None | início da fase 2 | ✅ |
| T9 | T8 | T8 → T9 | ✅ |
| T10 | T9 | T9 → T10 | ✅ |
| T11 | T10 | T10 → T11 | ✅ |
| T12 | T9 | T9 → T12 | ✅ |
| T13 | None | início da fase 3 | ✅ |
| T14 | T13 | T13 → T14 | ✅ |
| T15 | T13 | T13 → T15 | ✅ |
| T16 | T14, T15 | T14 → T16, T15 → T16 | ✅ |
| T17 | T16 | T16 → T17 | ✅ |
| T18 | T16 | T16 → T18 | ✅ |
| T19, T22 | None | inícios da fase 4 | ✅ |
| T20 | T19 | T19 → T20 | ✅ |
| T21 | T20 | T20 → T21 | ✅ |
| T23, T24, T26 | None | inícios da fase 5 | ✅ |
| T25 | T24 | T24 → T25 | ✅ |
| T27 | None | início da fase 6 | ✅ |
| T28 | T27 | T27 → T28 | ✅ |
| T29 | T28 | T28 → T29 | ✅ |
| T30, T31, T32 | T29 | T29 → T30/T31/T32 | ✅ |
| T33, T34, T35 | None | inícios da fase 7 | ✅ |
| T36 | T35 | T35 → T36 | ✅ |
| T37 | T33 | T33 → T37 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T2, T3, T4, T8, T13–T16, T19 | Backend domínio | unit | unit | ✅ |
| T1, T5–T7, T9–T12, T17, T18, T20–T26 | Backend DDL/persistência/controller/job | integration | integration | ✅ |
| T27–T36 | Frontend | unit | unit | ✅ |
| T37 | Ponta a ponta | e2e | e2e | ✅ |
