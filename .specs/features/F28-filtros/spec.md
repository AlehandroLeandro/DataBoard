# F28 — Filtros Specification

**Prioridade:** MVP (tela, objeto, barra superior, tipos de controle, padrões, atalhos de data relativos, obrigatório, URL, indicador) / V1 (encadeamento, visões salvas, LOV) · **Depende de:** F19, F12, F15 · **Origem:** F28 do `produto.specs.md`

## Problem Statement

Sem filtros, cada tela mostra um recorte fixo. O consumidor precisa escolher período, região e vendedor e ver todos os objetos reagirem, enquanto o construtor fixa recortes por objeto. Os filtros precisam virar predicados do motor, nunca filtragem no navegador, e somar-se à RLS.

## Goals

- [ ] Filtro de período padrão "mês atual" reavaliado a cada abertura (CA-28.1).
- [ ] Um filtro de tela afeta todos os objetos cuja visão tem a coluna, com mapeamento explícito quando os nomes diferem.
- [ ] Link com filtros na URL reproduz o estado para quem o abre.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Filtros encadeados (RF-28.5, CA-28.2) | V1 (junto com LOV, F16) |
| Visões salvas de filtros por usuário (RF-28.9) | V1 |
| LOV associada a coluna | V1 (F16); no MVP as opções vêm dos valores distintos da coluna |
| Filtro cruzado por clique em gráfico | V1 (F22 RF-22.3) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Valores relativos de data no MVP | Atalhos (hoje, ontem, últimos 7/30 dias, mês atual, mês anterior, ano atual) guardados como relativos e reavaliados a cada abertura | RF-28.2 é MVP e CA-28.1 exige "mês atual" dinâmico; o RF-28.4 (V1) fica coberto para datas | y |
| Aplicação automática | Filtro de tela aplica-se a todo objeto cuja visão tem coluna com o mesmo nome da coluna-fonte do filtro; mapeamento por objeto pode apontar outra coluna, um parâmetro `:param` da visão SQL, ou `IGNORE` | RF-28.1 | y |
| Parâmetros `:param` em visões SQL | Substituídos por bind(s) a partir do filtro mapeado; sem valor → `NULL` | Parte do RF-12.6 delegada pela F12 | y |
| Opções de seleção | Valores distintos da coluna-fonte via motor (com RLS), busca `q` e paginação de 100 | RF-28.2; RLS restringe opções (espírito do CA-28.2) | y |
| "Lembrar última escolha" | Persistido por usuário e tela em `app.user_screen_filter_state` ao aplicar | RF-28.3 | y |
| Precedência do valor inicial | URL > "lembrar última escolha" > padrão configurado | Link compartilhado deve prevalecer | y |
| Formato na URL | `?f.<slug>=valor`, múltiplos separados por vírgula; intervalo `inicio..fim` (datas ISO ou `yyyy-MM`); relativo pelo nome do atalho (`mes_atual`) | RF-28.8 | y |
| Filtro de objeto | Predicados fixos na config do objeto, aplicados com E; ocultos ao consumidor salvo `visible=true` | RF-28.1 | y |
| Filtro de data e KPI | O intervalo ativo do filtro mapeado para a `dateColumn` do KPI substitui o `defaultPeriod` (F20) | RN-20.1 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Filtros de tela e barra superior ⭐ MVP

**User Story**: Como consumidor, quero filtrar a tela inteira por período, região ou vendedor numa barra no topo.

**Why P1**: Interação básica de BI.

**Acceptance Criteria**:
1. WHEN o construtor define um filtro de tela com rótulo, tipo de controle, coluna-fonte e padrão THEN o sistema SHALL gravá-lo em `definition.filters` e validar coluna e tipo.
2. WHEN o consumidor aplica valores na barra superior THEN o frontend SHALL reenviar `filterValues` a todos os objetos afetados, e o backend SHALL converter cada valor em predicado do motor (RN-28.1).
3. WHEN um objeto usa visão com coluna de mesmo nome da coluna-fonte THEN o filtro SHALL ser aplicado automaticamente a ele.
4. WHERE o mapeamento do objeto aponta outra coluna, um parâmetro `:param` ou `IGNORE` the sistema SHALL aplicar o filtro àquela coluna, preencher o parâmetro ou não aplicar, respectivamente.
5. The predicados de filtro SHALL ser combinados com E à RLS e aos filtros de objeto (RN-28.2).
6. The tipos de controle SHALL incluir seleção única, seleção múltipla, busca de texto (`CONTAINS`), intervalo numérico, intervalo de datas com atalhos e booleano.
7. WHEN `GET /api/v1/screens/{id}/filters/{filterId}/values?q=&page=` é chamado THEN o sistema SHALL devolver valores distintos da coluna-fonte visíveis ao usuário (com RLS), paginados de 100 em 100.
8. IF um valor não é compatível com o tipo da coluna (ex.: texto em filtro numérico) THEN o objeto SHALL responder 400 `INVALID_FILTER_VALUE`.

**Independent Test**: IT com tela de 3 objetos (dois com a coluna, um com mapeamento para outra coluna) e um filtro de região.

---

### P1: Valores padrão e datas relativas ⭐ MVP

**User Story**: Como construtor, quero que filtros abram com um valor padrão fixo, por token, relativo ou com a última escolha do usuário.

**Why P1**: RF-28.3 e CA-28.1.

**Acceptance Criteria**:
1. WHEN o padrão é `RELATIVE_DATE: CURRENT_MONTH` e a tela é aberta em 15/05/2026 THEN o período aplicado SHALL ser de 01/05/2026 a 31/05/2026.
2. WHEN o padrão é `TOKEN` (ex.: `{{usuario.atributo.filial}}`) THEN o valor inicial SHALL ser o valor do token para o usuário.
3. WHERE o padrão é `LAST_USED` the sistema SHALL gravar os valores aplicados pelo usuário naquela tela e usá-los na próxima abertura.
4. WHEN a URL contém valores de filtro THEN eles SHALL prevalecer sobre a última escolha e sobre o padrão.
5. The atalhos de data SHALL incluir hoje, ontem, últimos 7 dias, últimos 30 dias, mês atual, mês anterior, ano atual e personalizado.

**Independent Test**: CA-28.1 com relógio fixo + teste de precedência.

---

### P1: Filtros de objeto ⭐ MVP

**User Story**: Como construtor, quero fixar recortes num objeto específico (ex.: só status FATURADO).

**Why P1**: RF-28.1.

**Acceptance Criteria**:
1. WHEN um objeto tem filtros fixos THEN o sistema SHALL aplicá-los com E a todas as consultas do objeto sobre a visão principal.
2. WHILE `visible=false` (padrão) the frontend SHALL não exibir o filtro de objeto ao consumidor; WHERE `visible=true` the indicador do objeto SHALL listá-lo.
3. IF um filtro de objeto referencia coluna inexistente na visão THEN o sistema SHALL responder 400 ao salvar a tela.

**Independent Test**: IT com objeto Valor com filtro fixo.

---

### P1: Obrigatório, URL e indicador ⭐ MVP

**User Story**: Como consumidor, quero compartilhar links com filtros aplicados e saber quais filtros afetam cada objeto.

**Why P1**: RF-28.6, RF-28.7, RF-28.8.

**Acceptance Criteria**:
1. WHILE um filtro obrigatório não tem valor the frontend SHALL não carregar os objetos afetados e SHALL exibir "Selecione <rótulo> para ver os dados".
2. WHEN o consumidor aplica filtros THEN o frontend SHALL atualizar a URL com `?f.<slug>=...`, e abrir essa URL SHALL restaurar o mesmo estado.
3. WHEN o consumidor passa o mouse no ícone de filtros de um objeto THEN o frontend SHALL listar os filtros de tela e de objeto visíveis que afetam aquele objeto, com seus valores.

**Independent Test**: Teste de componente + E2E de URL compartilhada.

---

### P2: Encadeamento, visões salvas e LOV [V1]

**User Story**: Como consumidor, quero filtros dependentes (Estado → Cidade) e salvar combinações de filtros.

**Why P2**: V1.

**Acceptance Criteria**:
1. WHEN seleciono "SC" no filtro Estado THEN a lista de cidades SHALL conter só cidades de SC presentes nos dados que posso ver (CA-28.2).

**Independent Test**: Diferido para a V1.

---

## Edge Cases

- IF a URL traz um filtro inexistente THEN o frontend SHALL ignorá-lo sem erro.
- WHEN a seleção múltipla está vazia THEN o filtro SHALL não ser aplicado (sem predicado), não `IN ()`.
- IF um parâmetro `:param` não tem filtro mapeado nem valor THEN o sistema SHALL usar `NULL`.
- WHEN o intervalo de datas tem início maior que o fim THEN o frontend SHALL impedir aplicar e o backend SHALL responder 400 `INVALID_FILTER_VALUE`.
- WHEN a tela tem mais de 20 filtros THEN o sistema SHALL responder 400 `SCREEN_TOO_MANY_FILTERS`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| FLT-01 | P1: Filtros de tela (definição e aplicação) | RF-28.1, RN-28.1, RN-28.2 | Tasks | Pending |
| FLT-02 | P1: Mapeamento por objeto e `:param` | RF-28.1, RF-12.6 | Tasks | Pending |
| FLT-03 | P1: Tipos de controle e valores distintos | RF-28.2 | Tasks | Pending |
| FLT-04 | P1: Padrões e datas relativas | RF-28.3, RF-28.4 (datas), CA-28.1 | Tasks | Pending |
| FLT-05 | P1: Filtros de objeto | RF-28.1 | Tasks | Pending |
| FLT-06 | P1: Obrigatório | RF-28.7 | Tasks | Pending |
| FLT-07 | P1: URL | RF-28.8 | Tasks | Pending |
| FLT-08 | P1: Indicador por objeto | RF-28.6 | Tasks | Pending |
| FLT-09 | P1: Integração com período do KPI | RN-20.1 | Tasks | Pending |
| FLT-10 | P2: Encadeamento, visões salvas, LOV | RF-28.5, RF-28.9, CA-28.2 | - | Pending |

**Coverage:** 10 total, 9 mapped to tasks, 1 diferido (V1).

---

## Success Criteria

- [ ] CA-28.1 automatizado.
- [ ] Nenhum filtro é aplicado no navegador: teste confirma que o predicado aparece no SQL gerado.
