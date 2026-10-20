# F09 — Tabelas Personalizadas Specification

**Prioridade:** MVP ⭐ (diferencial) · **Depende de:** F04, F06, F13/F14 · **Origem:** F09 do `produto.specs.md`

## Problem Statement

PMEs têm dados que o ERP não guarda: metas por vendedor, orçamento, de-para de produtos, classificações, controles operacionais. Hoje elas mantêm isso em planilhas soltas, que não cruzam com os dashboards. A plataforma precisa permitir criar tabelas próprias, sem SQL, com campos tipados, referências entre tabelas, campos calculados e agregações de registros relacionados. Essas tabelas viram fonte de dados como qualquer outra.

## Goals

- [ ] Um construtor cria as tabelas *Vendedor* e *Meta* (com referência e unicidade composta) em menos de 5 minutos, sem SQL.
- [ ] Fórmulas, lookups e rollups são recalculados na leitura, sem job.
- [ ] Nenhuma exclusão quebra fórmulas, visões ou objetos silenciosamente.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Referência a tabela do `lake` (RF-09.1b, RN-09.2, CA-09.7) | V1, junto com a F08 (Q26) |
| Cardinalidade `MANY_TO_MANY` | V1 (RF-09.2) |
| Regras de validação de linha (RF-09.17) | V1 |
| Exportar/importar definição em JSON (RF-09.15) | V1 |
| Materializar rollup; índice manual por campo | V1 (§9.9) |
| Campos `FILE`/`IMAGE` | Entram com a F11, que cria o armazenamento de arquivos (RF-11.5) |
| Grade de edição de dados (Browser) | F10 |
| Criar tabela a partir de planilha | F11 |
| Visão semântica `SYSTEM_UDT` (RF-09.21, CA-09.6) | F12 consome o evento `UdtSchemaChanged` publicado aqui |
| Alimentação por API pública e Processos | F39 (V2) e F34 (V1) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Armazenamento | DDL físico `udt.t_<shortId>`, colunas `f_<shortId>`, view de leitura `udt.v_<shortId>` | AD-006 | y |
| `shortId` | 8 caracteres hex aleatórios, únicos por tipo | Nomes físicos curtos e imutáveis | y |
| Nomes das colunas da view | `apiName` do campo (ex.: `valor_total`) | Legível no Editor SQL (F17) e nas visões; RN-09.4 limita a renomeação a campos sem dependentes | y |
| Leitura × escrita | Leitura de registros via `udt.v_*`; escrita via `udt.t_*` | Derivados só existem na view | y |
| Colunas de sistema | `id uuid`, `version int`, `created_at`, `updated_at`, `created_by`, `updated_by` em toda tabela | §9.2 e concorrência otimista (RN-10.1) | y |
| Exclusão de registro | Física, com histórico na auditoria (F06) | Lixeira é para estrutura (RF-09.14), não para dados | y |
| Erros de validação de dados | 400 `VALIDATION_ERROR` com `errors: [{ field, code, message }]` | RN-10.3 + padrão F00 | y |
| Compilador OBExpr | ANTLR4 em Java, dois modos: literais embutidos com escape (DDL de view) e bind parameters (filtros de consulta) | AD-007 e RN-12.2 | y |
| Valor padrão por OBExpr | Só expressões sem referência a campos (literais, `TODAY()`, `NOW()`), compiladas para `DEFAULT` da coluna | Atende o exemplo `TODAY()` sem avaliar fórmula por linha na inserção | y |
| AUTONUMBER | Sequência `udt.s_<shortId>` como `DEFAULT nextval`; prefixo e zeros à esquerda são aplicados na exibição | Armazena número puro; formato muda sem migração | y |
| Simulação de mudança de tipo | `pg_input_is_valid(valor::text, '<tipo>')` (Postgres 16+) | Conversão exata, sem heurística | y |
| Visão semântica do sistema | F09 publica `UdtSchemaChanged(tableId)` após cada mudança de estrutura; F12 mantém a visão `SYSTEM_UDT` | F12 é implementada depois da F09 | y |
| Limites (RN-09.5) | 200 campos, 3 níveis de navegação, 10 rollups por tabela, lidos de `limits` (F02) | Configuráveis | y |
| Lixeira | 30 dias; colunas e tabelas excluídas logicamente continuam no banco até o job de limpeza | RF-09.14 | y |
| Permissão de estrutura | `EDIT`/`MANAGE` na tabela **e** flag `MANAGE_UDT_SCHEMA` (ADMIN dispensa a flag) | RF-09.18 | y |
| Criar tabela | Exige papel BUILDER ou ADMIN + `MANAGE_UDT_SCHEMA` | Coerente com RF-09.18 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Criar tabelas e campos armazenados ⭐ MVP

**User Story**: Como construtor, quero criar uma tabela com campos tipados e validações para guardar dados que o ERP não tem.

**Why P1**: Núcleo da feature (fase 1 da Q26).

**Acceptance Criteria**:
1. WHEN um construtor com `MANAGE_UDT_SCHEMA` chama `POST /api/v1/udt/tables` com `label`, `description`, `icon` e `categoryId` opcionais THEN o sistema SHALL criar o metadado, executar `CREATE TABLE udt.t_<shortId>` com as colunas de sistema na mesma transação, registrar o recurso com `MANAGE` para o criador e responder 201.
2. WHEN um campo armazenado (`TEXT`, `LONG_TEXT`, `INTEGER`, `DECIMAL`, `CURRENCY`, `PERCENT`, `BOOLEAN`, `DATE`, `DATETIME`, `SELECT`, `MULTI_SELECT`, `EMAIL`, `URL`, `PHONE`, `AUTONUMBER`) é adicionado THEN o sistema SHALL executar `ALTER TABLE ... ADD COLUMN f_<shortId>` com o tipo nativo correspondente (`MULTI_SELECT` → `text[]`, `CURRENCY` → `numeric(18,2)`) na mesma transação do metadado.
3. IF o DDL falhar THEN o sistema SHALL desfazer também o metadado e responder 500 `UDT_DDL_FAILED`, sem deixar estrutura parcial.
4. WHEN um campo é renomeado ou reordenado THEN o sistema SHALL alterar só o metadado (`label`, `position`), sem DDL sobre a coluna física.
5. The sistema SHALL gerar o `apiName` em `snake_case` a partir do rótulo, sem acentos, único por tabela (sufixo `_2`, `_3` em colisão).
6. IF o `apiName` de um campo com dependentes for alterado THEN o sistema SHALL responder 409 com `code=UDT_FIELD_IN_USE` e a lista de dependentes.
7. WHEN uma restrição de unicidade composta (ex.: `vendedor + mes`) é criada THEN o sistema SHALL criar um índice único físico sobre as colunas.
8. IF a tabela atingiria mais de 200 campos THEN o sistema SHALL responder 422 com `code=UDT_LIMIT_FIELDS`.
9. IF quem chama não tem `EDIT` na tabela ou não tem `MANAGE_UDT_SCHEMA` THEN o sistema SHALL responder 403.
10. WHEN qualquer estrutura da tabela muda THEN o sistema SHALL regenerar `udt.v_<shortId>` e publicar `UdtSchemaChanged(tableId)` após o commit.

**Independent Test**: IT: criar *Vendedor* e *Meta*, verificar tabelas físicas, colunas, índice composto e view.

---

### P1: Referências entre tabelas ⭐ MVP

**User Story**: Como construtor, quero que um campo aponte para um registro de outra tabela (Meta → Vendedor) para relacionar dados sem duplicá-los.

**Why P1**: Base de lookups, rollups e master-detail.

**Acceptance Criteria**:
1. WHEN um campo `REFERENCE` `MANY_TO_ONE` para outra UDT (ou para a própria) é criado THEN o sistema SHALL adicionar coluna `uuid` com FK física e a ação `onDelete` configurada (`RESTRICT` padrão, `SET_NULL`, `CASCADE`), além de índice na coluna.
2. WHEN uma referência é criada THEN a tabela alvo SHALL expor uma relação inversa navegável com o rótulo `inverseLabel` (ex.: Vendedor → "Metas").
3. WHEN `GET /api/v1/udt/tables/{tableId}/reference-options/{fieldId}?q=&page=` é chamado THEN o sistema SHALL devolver `{ id, label }` da tabela alvo, usando o campo de exibição, filtrado por `q` (contém, case-insensitive) e paginado.
4. IF um registro referenciado com `RESTRICT` é excluído THEN o sistema SHALL responder 409 com `code=UDT_RECORD_REFERENCED`.
5. IF a tabela alvo não tem campo de exibição definido THEN as opções de referência SHALL usar o `id` como rótulo.

**Independent Test**: IT com Pedido → Cliente nas três ações de exclusão.

---

### P1: CRUD de registros com validação, permissões e concorrência ⭐ MVP

**User Story**: Como usuário com permissão de dados, quero incluir, alterar e excluir registros com validação clara.

**Why P1**: Sem escrita não há dados para os dashboards.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/udt/tables/{tableId}/records` recebe `values` válidos por `apiName` THEN o sistema SHALL inserir o registro com `version=1`, `created_by` e `created_at` preenchidos, e responder 201 com o registro lido da view (incluindo derivados).
2. IF um valor viola tipo, obrigatoriedade, mínimo/máximo, regex, formato (`EMAIL`/`URL`/`PHONE`) ou opção de `SELECT` THEN o sistema SHALL responder 400 `VALIDATION_ERROR` com `errors: [{ field, code, message }]`, um item por campo inválido.
3. IF a gravação viola unicidade simples ou composta THEN o sistema SHALL responder 400 com erro `UNIQUE_VIOLATION` nos campos envolvidos e a mensagem "Já existe um registro com <campos> iguais".
4. WHEN `PATCH .../records/{recordId}` recebe `values` e o `version` atual THEN o sistema SHALL atualizar apenas os campos enviados e incrementar `version`.
5. IF o `version` enviado difere do atual THEN o sistema SHALL responder 409 com `code=RECORD_VERSION_CONFLICT` e o registro atual no corpo.
6. IF o usuário não tem o nível `DATA_INSERT`, `DATA_UPDATE`, `DATA_DELETE` ou `DATA_READ` exigido pela operação THEN o sistema SHALL responder 403.
7. WHERE a tabela tem "usuário só edita os registros que criou" ativo, IF o usuário tenta alterar ou excluir registro criado por outro THEN o sistema SHALL responder 403 com `code=UDT_NOT_RECORD_OWNER`.
8. IF o `values` contém campo derivado ou de sistema THEN o sistema SHALL responder 400 com erro `READ_ONLY_FIELD` no campo.
9. WHEN `POST .../records/bulk` recebe `insert`, `update` e `delete` THEN o sistema SHALL aplicar tudo numa única transação e, se qualquer item falhar, desfazer tudo e responder 400 com os erros indexados (`insert[2].valor`).
10. WHEN qualquer escrita de registro é confirmada THEN o sistema SHALL publicar `UdtRecordChanged` com valores antes e depois (F06).
11. WHEN `GET .../records?sort=&page=&size=&fields=` é chamado THEN o sistema SHALL devolver a página lida da view, com apenas os campos pedidos quando `fields` é informado.
12. WHEN `GET .../records/{recordId}/history` é chamado por quem tem `DATA_READ` THEN o sistema SHALL devolver o histórico do registro (F06 `AuditQuery`).

**Independent Test**: CA-09.1 e o fluxo de conflito de versão em IT.

---

### P1: Linguagem de fórmula OBExpr ⭐ MVP

**User Story**: Como construtor, quero escrever expressões simples (`{quantidade} * {preco_unitario}`) com validação em tempo real.

**Why P1**: Base de FORMULA, filtros, padrões e colunas calculadas (F12).

**Acceptance Criteria**:
1. The OBExpr SHALL suportar referências `{campo}`, navegação `{a.b.c}` até 3 níveis, operadores `+ - * / %`, `= <> > >= < <=`, `AND OR NOT`, `&` e as funções `IF`, `IFS`, `COALESCE`, `ISBLANK`, `ROUND`, `ABS`, `MIN`, `MAX`, `POWER`, `CONCAT`, `UPPER`, `LOWER`, `LEFT`, `RIGHT`, `LEN`, `TRIM`, `REPLACE`, `TODAY`, `NOW`, `YEAR`, `MONTH`, `DAY`, `DATEADD`, `DATEDIFF`, `FORMAT_DATE`, `TO_NUMBER` e `TO_TEXT`.
2. WHEN `POST /api/v1/udt/expressions/validate` recebe `{ tableId, expression }` válida THEN o sistema SHALL responder `{ valid: true, resultType, errors: [], preview }` com o resultado para as 5 primeiras linhas.
3. IF a expressão tem erro de sintaxe, campo inexistente, função desconhecida, aridade errada ou tipos incompatíveis THEN o sistema SHALL responder `valid: false` com `errors: [{ message, start, end }]` apontando a posição do trecho.
4. IF a navegação passa de 3 níveis THEN o sistema SHALL responder erro `UDT_LIMIT_NAVIGATION_DEPTH` na posição da referência.
5. The compilador SHALL gerar SQL usando somente identificadores vindos do metadado e escapar literais de texto (aspas simples duplicadas) no modo DDL, ou usar bind parameters no modo consulta.
6. WHEN `GET .../records?filter=<OBExpr booleana>` é chamado THEN o sistema SHALL aplicar a expressão como `WHERE` com bind parameters.
7. IF o filtro não é booleano THEN o sistema SHALL responder 400 com `code=FILTER_NOT_BOOLEAN`.

**Independent Test**: Testes unitários de parser e compilador (tabela de expressões → SQL esperado) + IT do endpoint de validação.

---

### P1: Campos FORMULA e LOOKUP ⭐ MVP

**User Story**: Como construtor, quero campos calculados e campos trazidos de registros referenciados, atualizados sozinhos.

**Why P1**: Fase 2 da Q26.

**Acceptance Criteria**:
1. WHEN um campo `FORMULA` com expressão válida é salvo THEN o sistema SHALL inferir `resultType` (ou usar o forçado), registrar as dependências e regenerar a view com a expressão compilada.
2. WHEN um campo `LOOKUP` é salvo com um campo de referência e um campo alvo THEN a view SHALL trazer o valor do registro referenciado via `LEFT JOIN`.
3. WHEN um campo usado por uma FORMULA ou LOOKUP tem o valor alterado THEN a próxima leitura do registro SHALL refletir o novo valor calculado.
4. IF salvar um campo derivado criaria um ciclo (A usa B, B usa A, inclusive através de referências) THEN o sistema SHALL responder 422 com `code=UDT_CIRCULAR_DEPENDENCY` e `path` (ex.: `["B","A","B"]`).
5. WHEN uma mudança de estrutura afeta views de outras tabelas (via lookup ou rollup) THEN o sistema SHALL regenerar todas as views afetadas na mesma transação, em ordem topológica.

**Independent Test**: CA-09.3 (ciclo) + atualização de uma FORMULA após alterar a entrada.

---

### P1: Campos ROLLUP ⭐ MVP

**User Story**: Como construtor, quero agregar registros filhos (total do pedido = soma dos itens) sem SQL.

**Why P1**: Fase 3 da Q26.

**Acceptance Criteria**:
1. WHEN um campo `ROLLUP` é salvo com relação inversa, função (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `COUNT_DISTINCT`), campo agregado e filtro OBExpr opcional THEN a view SHALL calcular o valor por subconsulta `LATERAL` sobre a view da tabela filha.
2. WHEN `Item.qtd` de um item é alterado THEN a próxima leitura do `Pedido` SHALL refletir o novo `SUM(itens.total)`, inclusive quando `total` é FORMULA do Item.
3. WHEN o filtro `{itens.cancelado} = false` está configurado THEN o rollup SHALL ignorar itens cancelados.
4. IF a tabela atingiria mais de 10 rollups THEN o sistema SHALL responder 422 com `code=UDT_LIMIT_ROLLUPS`.
5. WHEN não há filhos THEN `COUNT` SHALL devolver 0 e as demais funções SHALL devolver `null`.

**Independent Test**: CA-09.2 em IT.

---

### P1: Mudança de tipo, exclusão e lixeira ⭐ MVP

**User Story**: Como construtor, quero mudar tipos e excluir campos e tabelas com segurança, podendo desfazer exclusões.

**Why P1**: Fase 4 da Q26; evita perda de dados.

**Acceptance Criteria**:
1. WHEN `POST .../fields/{fieldId}/type-change/simulate` recebe `newType` THEN o sistema SHALL responder `{ totalRows, failingRows, sample: [até 10 valores inválidos] }` sem alterar dados.
2. WHEN um campo TEXT com valores "10", "abc" e "20" é simulado para INTEGER THEN o sistema SHALL informar `failingRows=1`.
3. WHEN o construtor confirma a conversão THEN o sistema SHALL executar `ALTER COLUMN TYPE` convertendo valores inválidos para `null`, na mesma transação do metadado.
4. IF um campo ou tabela com dependentes (fórmulas, lookups, rollups, referências, visões, objetos, filtros, RLS) é excluído THEN o sistema SHALL responder 409 com `code=UDT_FIELD_IN_USE` (ou `UDT_TABLE_IN_USE`) e a lista de dependentes.
5. WHEN um campo ou tabela sem dependentes é excluído THEN o sistema SHALL marcá-lo `deleted_at`, removê-lo da API e da view e mantê-lo restaurável por 30 dias.
6. WHEN `POST /api/v1/udt/trash/{type}/{id}/restore` é chamado dentro de 30 dias THEN o sistema SHALL restaurar o item com seus dados.
7. WHEN o job diário de limpeza roda THEN o sistema SHALL executar `DROP COLUMN`/`DROP TABLE` dos itens excluídos há mais de 30 dias.
8. WHEN `GET /api/v1/udt/tables/{tableId}/dependencies` é chamado THEN o sistema SHALL listar os dependentes de cada campo e da tabela.
9. WHEN `POST /api/v1/udt/tables/{tableId}/duplicate` é chamado THEN o sistema SHALL criar uma nova tabela com a mesma estrutura (campos, validações, unicidades), sem registros.

**Independent Test**: CA-09.4 e CA-09.5 em IT.

---

### P2: Recursos V1 de tabelas personalizadas [V1]

**User Story**: Como construtor, quero referência ao lake, N:N, regras de linha, exportação de definição e rollups materializados.

**Why P2**: V1.

**Acceptance Criteria**:
1. WHERE a F08 está disponível, WHEN um registro referencia uma chave do lake removida por carga FULL THEN o Browser SHALL sinalizá-lo como "referência quebrada" (CA-09.7).
2. WHEN uma regra de linha `{data_fim} >= {data_inicio}` é violada THEN o sistema SHALL rejeitar a gravação com a mensagem configurada.

**Independent Test**: Diferido para a V1.

---

## Edge Cases

- IF o rótulo gera `apiName` reservado (`id`, `version`, `created_at`, `updated_at`, `created_by`, `updated_by`) THEN o sistema SHALL acrescentar o sufixo `_campo`.
- IF uma expressão referencia um campo excluído logicamente THEN a validação SHALL retornar erro de campo inexistente.
- WHEN uma fórmula divide por zero numa linha THEN a view SHALL devolver `null` para essa linha (compilação com `NULLIF(divisor, 0)`).
- IF `bulk` recebe mais de 5.000 operações THEN o sistema SHALL responder 413 com `code=BULK_TOO_LARGE`.
- WHEN dois construtores alteram a estrutura da mesma tabela simultaneamente THEN o sistema SHALL serializar via `SELECT ... FOR UPDATE` no metadado da tabela.
- IF um valor `MULTI_SELECT` contém opção inexistente THEN o sistema SHALL responder erro `INVALID_OPTION` no campo.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| UDT-01 | P1: Criar tabelas (DDL transacional) | RF-09.11, §9.9 | Tasks | Pending |
| UDT-02 | P1: Campos armazenados e tipos | §9.2, RF-09.12 | Tasks | Pending |
| UDT-03 | P1: apiName e renomeação | RN-09.4 | Tasks | Pending |
| UDT-04 | P1: Validações de campo e unicidade composta | RF-09.16, CA-09.1 | Tasks | Pending |
| UDT-05 | P1: Limites | RN-09.5 | Tasks | Pending |
| UDT-06 | P1: Permissões de estrutura | RF-09.18 | Tasks | Pending |
| UDT-07 | P1: Referências N:1 + inversa | RF-09.1a, RF-09.2, RF-09.3, RF-09.4, RN-09.1 | Tasks | Pending |
| UDT-08 | P1: Opções de referência | RF-09.5 | Tasks | Pending |
| UDT-09 | P1: CRUD de registros | Contrato de dados F09, RN-09.6 | Tasks | Pending |
| UDT-10 | P1: Permissões de dados e dono | RF-09.18, RF-09.19 | Tasks | Pending |
| UDT-11 | P1: Concorrência otimista | RN-10.1 | Tasks | Pending |
| UDT-12 | P1: Bulk atômico | Contrato de dados F09 | Tasks | Pending |
| UDT-13 | P1: Histórico de registro | RF-10.9, CA-06.1 | Tasks | Pending |
| UDT-14 | P1: OBExpr (parser, tipos, compilador, validação) | RF-09.6, RF-09.7, RF-09.8, D-04 | Tasks | Pending |
| UDT-15 | P1: Filtro OBExpr em registros | Contrato de dados F09 | Tasks | Pending |
| UDT-16 | P1: FORMULA e LOOKUP | RF-09.6, RF-09.10, §9.9 | Tasks | Pending |
| UDT-17 | P1: Ciclos | RN-09.3, CA-09.3 | Tasks | Pending |
| UDT-18 | P1: ROLLUP | RF-09.9, RF-09.10, CA-09.2 | Tasks | Pending |
| UDT-19 | P1: Valor padrão OBExpr e AUTONUMBER | RF-09.16, §9.2 | Tasks | Pending |
| UDT-20 | P1: Mudança de tipo | RF-09.13, CA-09.5 | Tasks | Pending |
| UDT-21 | P1: Exclusão com dependentes e lixeira | RF-09.14, CA-09.4 | Tasks | Pending |
| UDT-22 | P1: Dependências e duplicação | RF-09.11, contrato | Tasks | Pending |
| UDT-23 | P1: Evento de mudança de estrutura | RF-09.21 (via F12) | Tasks | Pending |
| UDT-24 | P2: Recursos V1 | RF-09.1b, RF-09.2 N:N, RF-09.15, RF-09.17, CA-09.7 | - | Pending |

**Coverage:** 24 total, 23 mapped to tasks, 1 diferido (V1).

---

## Success Criteria

- [ ] CA-09.1 a CA-09.5 automatizados (CA-09.6 na F12; CA-09.7 na V1).
- [ ] Nenhum SQL gerado concatena valor vindo do usuário fora do escape de literais do modo DDL.
