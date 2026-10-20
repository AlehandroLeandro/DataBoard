# F21 — Objeto Consulta (Tabela e Pivot) Specification

**Prioridade:** MVP (tabela agregada, total geral, colunas calculadas, formatação de colunas) / V1 (pivot, subtotais, total com agregação diferente) · **Depende de:** F19 · **Origem:** F21 do `produto.specs.md`

## Problem Statement

Muitas análises são tabelas: vendas por região e vendedor, top 10 produtos, participação no total. O objeto Consulta exibe dimensões e medidas em colunas com ordenação, total geral, colunas calculadas (% do total, acumulado, ranking, diferença para a linha anterior) e formatação rica (máscara, alinhamento, imagem, barra de dados, link).

## Goals

- [ ] Tabela "Top 10 produtos por faturamento" com % do total e ranking, configurada sem SQL.
- [ ] Colunas calculadas do objeto computadas no analytics de forma consistente na tela e na exportação.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Modo pivot (RF-21.3) | V1 |
| Subtotais por agrupamento e total com agregação diferente da coluna (RF-21.2 avançado) | V1 |
| Linhas calculadas de DRE | F25 (V1) |
| Destaques condicionais | F26 (V1) |
| Drill | F27 (V1) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Total geral | Calculado pelo motor (`totals: true`, F12) e devolvido como linha de total | Correto para medidas não aditivas | y |
| Colunas calculadas do objeto | `PERCENT_OF_TOTAL`, `DIFF_PREVIOUS_ROW`, `PCT_DIFF_PREVIOUS_ROW`, `CUMULATIVE_SUM`, `CUMULATIVE_PCT`, `RANK` (denso, decrescente), calculadas no analytics sobre as linhas na ordem final | RF-21.5 | y |
| Base do % do total | Total geral do motor (não a soma das linhas exibidas no top N) | "% do total" deve ser do total real | y |
| Limite de linhas | `topN` (padrão 1.000, máx. `limits.maxQueryRows`); acima disso `truncated=true` | RF-21.6 | y |
| Ordenação por clique | No cliente quando o resultado não está truncado; com `truncated=true`, refaz a consulta com `sortOverride` | Evita ida ao banco desnecessária | y |
| Paginação | Rolagem virtual no cliente (sem paginação no servidor) | Até 1.000 linhas típicas; RF-21.1 aceita scroll virtual | y |
| Barra de dados | Mín. e máx. da coluna calculados no analytics; barra proporcional; negativos à esquerda do eixo | RF-21.4 | y |
| Imagem na célula | Coluna com URL `http(s)` ou campo `IMAGE` (fileId → `/api/v1/files/{id}`), miniatura 32 px | RF-21.4 | y |
| Link | Template `https://.../{valor}` com o valor da célula codificado em URL; só esquemas `http`/`https` | Segurança | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Tabela agregada com total ⭐ MVP

**User Story**: Como analista, quero uma tabela com dimensões e medidas agregadas, ordenável e com total geral.

**Why P1**: Objeto base de análise.

**Acceptance Criteria**:
1. WHEN um objeto `QUERY` com dimensões, medidas, ordenação padrão e `topN` é carregado THEN o sistema SHALL devolver as colunas na ordem configurada e até `topN` linhas agregadas pelo motor.
2. WHERE `showTotals=true` the sistema SHALL devolver a linha de total calculada pelo motor sem `GROUP BY`.
3. WHEN o usuário clica no cabeçalho de uma coluna e o resultado não está truncado THEN o frontend SHALL reordenar localmente, sem nova requisição.
4. WHEN o usuário clica no cabeçalho e o resultado está truncado THEN o frontend SHALL refazer a requisição com `sortOverride` e o sistema SHALL ordenar no banco.
5. The frontend SHALL permitir redimensionar colunas e rolar com virtualização.
6. IF o resultado excede `topN` THEN a resposta SHALL trazer `truncated=true` e o frontend SHALL exibir "Mostrando N primeiras linhas".

**Independent Test**: IT do handler + teste de componente da tabela.

---

### P1: Colunas calculadas do objeto ⭐ MVP

**User Story**: Como analista, quero % do total, acumulado, ranking e variação entre linhas sem criar campos na visão.

**Why P1**: RF-21.5.

**Acceptance Criteria**:
1. WHERE uma coluna `PERCENT_OF_TOTAL` sobre `valor` está configurada the analytics SHALL devolver `valor_linha / total_geral` para cada linha, usando o total do motor.
2. WHERE `CUMULATIVE_SUM` está configurada the analytics SHALL devolver a soma acumulada na ordem final das linhas.
3. WHERE `RANK` está configurada the analytics SHALL devolver o ranking denso decrescente (empates com o mesmo número).
4. WHERE `DIFF_PREVIOUS_ROW` está configurada the analytics SHALL devolver a diferença para a linha anterior, com `null` na primeira linha.
5. IF o total geral é 0 THEN `PERCENT_OF_TOTAL` SHALL ser `null` em todas as linhas.

**Independent Test**: pytest do transform `table` com tabela de casos.

---

### P1: Formatação de colunas ⭐ MVP

**User Story**: Como construtor, quero máscara, alinhamento, imagem, barra de dados e link por coluna.

**Why P1**: RF-21.4.

**Acceptance Criteria**:
1. The frontend SHALL aplicar a máscara da coluna do objeto ou, na ausência, a da coluna da visão (RF-14.3).
2. WHERE `display=DATA_BAR` the frontend SHALL desenhar a barra proporcional entre o mínimo e o máximo devolvidos pelo analytics.
3. WHERE `display=IMAGE` the frontend SHALL exibir miniatura de 32 px para URL `http(s)` ou `fileId`.
4. WHERE `linkTemplate` está configurado the frontend SHALL renderizar link com o valor codificado em URL, e o sistema SHALL rejeitar templates com esquema diferente de `http`/`https` (400 em `columns[i].linkTemplate`).
5. The alinhamento padrão SHALL ser à direita para números e à esquerda para os demais, sobrescrevível por coluna.

**Independent Test**: Testes de componente por tipo de exibição.

---

### P2: Pivot e subtotais [V1]

**User Story**: Como analista, quero dimensões em linhas e colunas com expandir/recolher e subtotais.

**Why P2**: V1.

**Acceptance Criteria**:
1. WHEN o modo pivot é ativado THEN o analytics SHALL pivotar o resultado com as dimensões de coluna e devolver os níveis expansíveis com subtotais.

**Independent Test**: Diferido para a V1.

---

## Edge Cases

- WHEN não há dimensões THEN a tabela SHALL ter uma única linha com as medidas.
- IF a mesma coluna é configurada duas vezes THEN o sistema SHALL responder 400 `DUPLICATE_COLUMN`.
- WHEN uma célula de imagem tem URL inválida THEN o frontend SHALL exibir placeholder sem erro.
- IF uma coluna calculada referencia medida ausente da tabela THEN o sistema SHALL responder 400 em `calculatedColumns[i].baseColumn`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| TBL-01 | P1: Tabela (dados, top N, truncamento) | RF-21.1, RF-21.6 | Tasks | Pending |
| TBL-02 | P1: Tabela (total geral) | RF-21.2 | Tasks | Pending |
| TBL-03 | P1: Tabela (ordenação, larguras, scroll) | RF-21.1 | Tasks | Pending |
| TBL-04 | P1: Colunas calculadas | RF-21.5 | Tasks | Pending |
| TBL-05 | P1: Formatação de colunas | RF-21.4, RF-14.3 | Tasks | Pending |
| TBL-06 | P2: Pivot e subtotais | RF-21.2, RF-21.3 | - | Pending |

**Coverage:** 6 total, 5 mapped to tasks, 1 diferido (V1).

---

## Success Criteria

- [ ] Transform `table` com todos os tipos de coluna calculada cobertos por pytest.
