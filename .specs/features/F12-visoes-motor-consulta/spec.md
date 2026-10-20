# F12 — Visões, Colunas e Motor de Consulta Specification

**Prioridade:** MVP (TABLE, SQL, SYSTEM_UDT, colunas, calculadas, motor, cache, impacto) / V1 (JOIN visual, LOV e drill nas colunas) · **Depende de:** F07, F09 · **Origem:** F12 do `produto.specs.md`

## Problem Statement

Objetos de tela não podem carregar SQL próprio: segurança, RLS, máscaras e agregações precisam ser centralizadas. A **Visão** é a camada semântica que nomeia datasets e descreve colunas como dimensões ou medidas. O **motor de consulta** transforma `visão + dimensões + medidas + filtros + ordenação + limite` em SQL seguro para o dialeto da conexão. Toda visualização, exportação e alerta passa por ele.

## Goals

- [ ] Um construtor cria uma visão sobre uma tabela externa, uma SQL ou usa a visão automática de uma UDT, e a consulta pelo motor em < 2 min.
- [ ] Todo SQL gerado usa identificadores do metadado e valores em bind parameters (RN-12.2).
- [ ] Remover uma coluna usada por objetos alerta o impacto antes de salvar (CA-12.2).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Visão `JOIN` montada visualmente | V1 (RF-12.1) |
| LOV associada à coluna (RF-12.3) | V1 (F16) |
| Hierarquias de drill customizadas (RF-12.3) | V1 (F27); hierarquia automática de datas está no MVP |
| Tokens em SQL de visão e parâmetros `:param` (RF-12.6) | Tokens na F15; `:param` ligado a filtros na F28. Aqui fica o ponto de extensão `QueryParameterResolver` |
| RLS (RN-05.1) | F05; aqui fica o ponto de extensão `QueryInterceptor` |
| Invalidação de cache por fim de carga ETL | V1 (F08); o mecanismo de invalidação por evento é entregue aqui |
| Pós-processamento (pivot, totais por grupo, gráficos) | F19–F22 no analytics (AD-003) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Tipos de visão no MVP | `TABLE` (tabela de conexão externa, `lake.*` ou UDT), `SQL`, `SYSTEM_UDT` | RF-12.1 | y |
| Quem cria | BUILDER/ADMIN com `VIEW` na conexão; visão `SQL` exige também `SQL_EDITOR` | RF-12.1 | y |
| Descoberta de colunas | Executa `SELECT * FROM (<fonte>) v WHERE 1=0` (equivalente portátil ao `LIMIT 0`) e lê o `ResultSetMetaData` | Funciona nos 3 dialetos | y |
| Papel inferido | Numéricos → `MEASURE` com `SUM`; demais → `DIMENSION` com `NONE`; colunas `id`/`*_id` numéricas → `DIMENSION` | RF-09.21 e bom senso para chaves | y |
| Colunas ausentes | Marcadas `status=MISSING`, mantidas no metadado; consultas que as usam falham com `VIEW_COLUMN_MISSING` | RF-12.2 | y |
| Colunas calculadas | OBExpr com funções de agregação (`SUM`, `AVG`, `COUNT`, `COUNT_DISTINCT`, `MIN`, `MAX`) permitidas só em medidas; modo SQL exige `SQL_EDITOR` | RF-12.4 | y |
| OBExpr multi-dialeto | O `SqlCompiler` da F09 ganha um parâmetro `SqlDialect` com mapeamento de funções para Postgres, MySQL e SQL Server | Visões rodam em qualquer conexão MVP | y |
| `dateGrain` | `YEAR`, `QUARTER`, `MONTH`, `DAY` via `SqlDialect.dateTrunc` | RF-12.5 | y |
| Totais | Pedidos com `totals: true` executam uma segunda consulta sem `GROUP BY`, com as mesmas medidas e filtros | Correto para medidas não aditivas (`AVG`, `COUNT_DISTINCT`) | y |
| Truncamento | O motor pede `limit+1` linhas (limitado por `maxQueryRows`) e devolve `truncated=true` se vier uma linha a mais | CA-02.2 | y |
| Cache | Caffeine em memória (máx. 500 entradas / 200 MB), chave = hash(datasource + SQL final + binds). Padrão: `LIVE` sem cache; `SYSTEM_UDT`/UDT com cache invalidado por `UdtRecordChanged`/`UdtSchemaChanged`; TTL configurável por visão | RF-12.8; o SQL final já inclui o predicado de RLS, então usuários com contextos diferentes não compartilham entrada | y |
| Visão do sistema da UDT | Criada/atualizada ao receber `UdtSchemaChanged`; nome = rótulo da UDT; estrutura não editável; metadados de coluna editáveis e preservados entre regenerações | RF-09.21, CA-09.6 | y |
| Browser sobre visão | `ViewBrowserSource` (somente leitura) implementa a interface da F10 | Pendência registrada na F10 | y |
| Dedução de dependências | Visão → conexão, visão → UDT, coluna calculada → colunas usadas; objetos (F19+) registram arestas para `VIEW` e `VIEW_COLUMN` | AD-019 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Criar visões TABLE e SQL ⭐ MVP

**User Story**: Como construtor, quero criar uma visão a partir de uma tabela ou de uma consulta SQL para disponibilizar um dataset nomeado para telas.

**Why P1**: Fonte de todos os objetos.

**Acceptance Criteria**:
1. WHEN um construtor com `VIEW` na conexão chama `POST /api/v1/views` com `type=TABLE`, conexão, schema e tabela THEN o sistema SHALL descobrir as colunas, inferir papéis e agregações, registrar o recurso `VIEW` com `MANAGE` para o criador e responder 201.
2. WHEN `type=SQL` THEN o sistema SHALL exigir a flag `SQL_EDITOR`, validar o SQL como somente-SELECT e descobrir as colunas.
3. IF o SQL de uma visão em conexão `LIVE` contém `DELETE FROM x` THEN o sistema SHALL responder 400 com `code=ONLY_SELECT_ALLOWED` (CA-07.2).
4. IF a descoberta de colunas falha (erro de SQL, tabela inexistente, timeout) THEN o sistema SHALL responder 422 com `code=VIEW_SOURCE_ERROR` e a mensagem do banco.
5. WHEN a fonte de uma visão é alterada e salva THEN o sistema SHALL adicionar colunas novas e marcar como `MISSING` as removidas.
6. IF colunas removidas são usadas por dependentes THEN a resposta do salvamento SHALL incluir `impact: [{ column, dependents: [...] }]` listando os objetos afetados (CA-12.2).
7. IF uma visão com dependentes é excluída THEN o sistema SHALL responder 409 com `code=VIEW_IN_USE` e a lista de dependentes.

**Independent Test**: IT: visão TABLE sobre Postgres externo e visão SQL; alteração da SQL removendo uma coluna usada por 2 dependentes fictícios gera o alerta.

---

### P1: Visão automática de tabela personalizada ⭐ MVP

**User Story**: Como construtor, quero que toda tabela personalizada já tenha uma visão pronta para usar em telas.

**Why P1**: RF-09.21 / CA-09.6.

**Acceptance Criteria**:
1. WHEN uma UDT é criada THEN o sistema SHALL criar uma visão `SYSTEM_UDT` sobre `udt.v_<id>` com todas as colunas (inclusive derivadas), rótulos amigáveis e papéis inferidos, disponível imediatamente.
2. WHEN a estrutura da UDT muda THEN o sistema SHALL sincronizar as colunas da visão, preservando rótulo, máscara, papel e agregação já editados das colunas que continuam existindo.
3. IF alguém tenta alterar a fonte ou excluir uma visão `SYSTEM_UDT` THEN o sistema SHALL responder 409 com `code=SYSTEM_VIEW_READONLY`.
4. The ACL da visão `SYSTEM_UDT` SHALL herdar da UDT (recurso pai = UDT) enquanto não houver entradas próprias.

**Independent Test**: CA-09.6: criar UDT e consultar sua visão pelo motor em seguida.

---

### P1: Metadados de colunas e colunas calculadas ⭐ MVP

**User Story**: Como construtor, quero dar nomes amigáveis, papéis, agregações, máscaras e criar colunas calculadas para que o consumidor não precise saber SQL.

**Why P1**: Camada semântica.

**Acceptance Criteria**:
1. WHEN `PUT /api/v1/views/{id}` altera rótulo, descrição, papel, agregação padrão, máscara ou visibilidade de colunas THEN o sistema SHALL gravar e registrar dependência `VIEW_COLUMN → MASK`.
2. WHEN uma coluna calculada de dimensão `ano_mes = FORMAT_DATE({data}, 'yyyy-MM')` é salva THEN o motor SHALL agrupá-la como qualquer dimensão.
3. WHEN uma coluna calculada de medida `margem = SUM({receita}) - SUM({custo})` é salva THEN o motor SHALL usá-la sem reaplicar agregação externa.
4. IF uma coluna calculada de dimensão usa função de agregação THEN o sistema SHALL responder 400 com `code=AGGREGATE_IN_DIMENSION`.
5. IF uma coluna calculada em modo SQL é salva por quem não tem `SQL_EDITOR` THEN o sistema SHALL responder 403 `FLAG_REQUIRED`.
6. The colunas DATE/DATETIME SHALL aceitar `dateGrain` `YEAR`, `QUARTER`, `MONTH` ou `DAY` no motor.

**Independent Test**: IT com as duas calculadas do RF-12.4 em Postgres e MySQL.

---

### P1: Motor de consulta ⭐ MVP

**User Story**: Como frontend, objeto ou exportação, quero pedir dados por dimensões, medidas e filtros e receber resultado tipado, seguro e limitado.

**Why P1**: RN-12.1: toda consulta de objeto passa por aqui.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/query` recebe `viewId`, `dimensions`, `measures`, `filters`, `sort` e `limit` THEN o sistema SHALL gerar SQL com `GROUP BY` das dimensões e as agregações das medidas, e responder `{ columns: [{ key, label, type, maskId }], rows, totals?, truncated, durationMs }`.
2. WHEN a visão "Vendas" tem a medida `valor (SUM)` e o pedido é `valor` por `regiao` THEN o SQL gerado SHALL conter `GROUP BY` sobre `regiao` e `SUM(` sobre `valor`.
3. The SQL gerado SHALL usar somente identificadores de colunas existentes no metadado da visão e valores apenas como bind parameters.
4. IF o pedido referencia coluna inexistente, oculta ou `MISSING` THEN o sistema SHALL responder 400 com `code=VIEW_COLUMN_INVALID` ou `VIEW_COLUMN_MISSING`.
5. WHEN o resultado excede `limits.maxQueryRows` THEN o sistema SHALL devolver exatamente `maxQueryRows` linhas e `truncated=true`.
6. IF a execução excede `limits.queryTimeoutSeconds` THEN o sistema SHALL cancelar a consulta e responder 504 com `code=QUERY_TIMEOUT`.
7. IF o usuário não tem `VIEW` na visão THEN o sistema SHALL responder 403.
8. WHERE `totals: true` the sistema SHALL calcular `totals` com uma consulta sem `GROUP BY` usando as mesmas medidas e filtros.
9. WHEN um pedido sem dimensões é feito THEN o sistema SHALL devolver uma linha com as medidas agregadas.
10. The motor SHALL aplicar, antes de executar, todos os `QueryInterceptor` registrados (ponto de extensão usado pela RLS na F05).
11. The filtros SHALL suportar os operadores `EQ`, `NEQ`, `GT`, `GTE`, `LT`, `LTE`, `IN`, `NOT_IN`, `BETWEEN`, `CONTAINS`, `STARTS_WITH`, `IS_NULL` e `IS_NOT_NULL`.

**Independent Test**: CA-12.1 (sem RLS ainda; a parte de RLS é verificada na F05) + truncamento CA-02.2.

---

### P1: Pré-visualização, impacto e cache ⭐ MVP

**User Story**: Como construtor, quero pré-visualizar dados, ver quem usa a visão e ter respostas rápidas em dados que mudam pouco.

**Why P1**: RF-12.7 e RF-12.8.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/views/{id}/preview` é chamado THEN o sistema SHALL devolver até 100 linhas cruas da visão (aplicando interceptors).
2. WHEN `GET /api/v1/views/{id}/impact` é chamado THEN o sistema SHALL listar dependentes da visão e de cada coluna (objetos, telas, alertas, relatórios, RLS, browsers).
3. WHEN `POST /api/v1/views/{id}/refresh-columns` é chamado THEN o sistema SHALL redescobrir colunas com as mesmas regras do salvamento.
4. WHILE uma visão sobre UDT tem resultado em cache THEN uma nova consulta idêntica SHALL ser servida do cache sem ir ao banco.
5. WHEN ocorre `UdtRecordChanged` ou `UdtSchemaChanged` de uma UDT THEN o sistema SHALL invalidar o cache das visões que dependem dela.
6. WHERE a visão tem `cacheTtlSeconds > 0` the sistema SHALL reutilizar resultados idênticos por esse tempo.

**Independent Test**: IT com contador de consultas (datasource-proxy): segunda chamada idêntica não consulta; após escrita na UDT, consulta de novo.

---

### P1: Browser sobre visão ⭐ MVP

**User Story**: Como construtor, quero criar um Browser somente leitura sobre qualquer visão para listagens de consulta.

**Why P1**: RF-10.1 (fonte Visão) depende das visões desta feature.

**Acceptance Criteria**:
1. WHEN um Browser com `sourceType=VIEW` é aberto THEN o sistema SHALL listar as linhas da visão com paginação, ordenação e filtros no servidor, pelo motor de consulta.
2. The Browser com fonte visão SHALL ser somente leitura (`meta.canInsert=false`, `canUpdate=false`, `canDelete=false`).

**Independent Test**: IT de `GET /browsers/{id}/data` sobre uma visão SQL.

---

### P2: Visão JOIN visual [V1]

**User Story**: Como construtor sem SQL, quero montar uma visão unindo tabelas por chaves.

**Why P2**: V1.

**Acceptance Criteria**:
1. WHEN uma visão `JOIN` é salva com tabela base, joins (tipo e chaves) e filtros fixos THEN o sistema SHALL gerar o SQL sem que o usuário escreva SQL.

**Independent Test**: Diferido para a V1.

---

## Edge Cases

- IF `limit` do pedido for maior que `maxQueryRows` THEN o motor SHALL usar `maxQueryRows`.
- WHEN um filtro `IN` recebe lista vazia THEN o motor SHALL gerar predicado falso (`1=0`), sem erro.
- IF o mesmo `column` aparece duas vezes em `dimensions` com o mesmo `dateGrain` THEN o sistema SHALL responder 400 `DUPLICATE_DIMENSION`.
- WHEN a conexão de origem está fora do ar THEN o motor SHALL responder 502 com `code=DATASOURCE_UNAVAILABLE`.
- IF uma coluna calculada referencia outra calculada THEN o compilador SHALL expandi-la inline, e ciclos SHALL ser rejeitados com `VIEW_CIRCULAR_DEPENDENCY`.
- WHEN o tipo JDBC não é mapeável THEN a coluna SHALL ser tipada como `TEXT` e marcada `typeFallback=true`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| VIEW-01 | P1: Visões TABLE e SQL | RF-12.1 | Tasks | Pending |
| VIEW-02 | P1: Descoberta de colunas e ausentes | RF-12.2, CA-12.2 | Tasks | Pending |
| VIEW-03 | P1: Exclusão em uso | RN-12.4 | Tasks | Pending |
| VIEW-04 | P1: Visão SYSTEM_UDT | RF-09.21, CA-09.6 | Tasks | Pending |
| VIEW-05 | P1: Metadados de coluna | RF-12.3 | Tasks | Pending |
| VIEW-06 | P1: Colunas calculadas (OBExpr multi-dialeto) | RF-12.4 | Tasks | Pending |
| VIEW-07 | P1: Hierarquia de datas | RF-12.5 | Tasks | Pending |
| VIEW-08 | P1: Motor (SQL seguro, GROUP BY, filtros) | RN-12.1, RN-12.2, CA-12.1 | Tasks | Pending |
| VIEW-09 | P1: Motor (limites, timeout, totais) | RN-12.3, CA-02.2 | Tasks | Pending |
| VIEW-10 | P1: Motor (interceptors e parâmetros) | RN-05.1 (extensão), RF-12.6 (extensão) | Tasks | Pending |
| VIEW-11 | P1: Prévia e impacto | RF-12.7 | Tasks | Pending |
| VIEW-12 | P1: Cache | RF-12.8 | Tasks | Pending |
| VIEW-13 | P1: Browser sobre visão | RF-10.1 | Tasks | Pending |
| VIEW-14 | P2: Visão JOIN | RF-12.1 | - | Pending |

**Coverage:** 14 total, 13 mapped to tasks, 1 diferido (V1).

---

## Success Criteria

- [ ] CA-12.1, CA-12.2 e CA-09.6 automatizados.
- [ ] Teste de injeção: nenhum valor de filtro altera a estrutura do SQL gerado.
