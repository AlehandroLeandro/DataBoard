# F22 — Objeto Gráfico (Tipos Básicos) Specification

**Prioridade:** MVP · **Depende de:** F19 · **Origem:** F22 do `produto.specs.md`

## Problem Statement

Tendências, composições e comparações ficam mais claras em gráficos. O construtor precisa montar barras, linhas, áreas, pizza, combinado, dispersão e funil a partir de uma visão, com eixos, legenda, rótulos, cores e linha de meta. O analytics (Python) monta o gráfico e o React o renderiza interativo (AD-005).

## Goals

- [ ] Os 7 tipos básicos configuráveis pelo mesmo editor, e trocar de tipo preserva a configuração de dados (RF-22.4).
- [ ] Séries com muitas categorias não travam a tela: aviso e agrupamento em "Outros" (RN-22.1).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Filtro cruzado ao clicar (RF-22.3) | V1 |
| Drill ao clicar | V1 (F27) |
| Cascata, treemap, calendário, sankey, mapa, gauge | V2 (F23) |
| Cor por regra de destaque | V1 (F26) |
| Exportar PNG | F29 (usa `getDataURL` do ECharts) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Quem monta o gráfico | Analytics devolve `option` do ECharts (JSON) + `meta` (máscaras por série/eixo, avisos); o front injeta formatters (funções) a partir das máscaras | AD-005; funções não trafegam em JSON | y |
| Tipos | `BAR` (vertical/horizontal; agrupado/empilhado/100%), `LINE` e `AREA` (simples/empilhada), `PIE` e `DONUT`, `COMBO` (barras + linhas com eixo secundário), `SCATTER`, `FUNNEL` | RF-22.1 | y |
| Restrições por tipo | `PIE`/`DONUT`/`FUNNEL`: 1 medida, sem série; `SCATTER`: 2 medidas (X, Y), dimensão opcional como rótulo/série; `COMBO`: ≥ 2 medidas, cada uma marcada `bar` ou `line` e eixo `primary`/`secondary` | Semântica dos gráficos | y |
| Troca de tipo (RF-22.4) | Mantém visão, dimensões, medidas, máscaras e cores; configurações incompatíveis com o novo tipo ficam guardadas e a validação aponta o que falta | Preserva o trabalho | y |
| Limite de categorias (RN-22.1) | `maxCategories` padrão 50; acima disso aviso `TOO_MANY_CATEGORIES` e, para medidas `SUM`/`COUNT`, as excedentes viram "Outros" (soma); para agregações não aditivas, mantém as N primeiras e avisa | "Outros" só é correto para medidas aditivas | y |
| Ordem das categorias | Dimensão de data → cronológica crescente; demais → pela primeira medida decrescente, configurável | Leitura natural | y |
| 100% empilhado | Valores normalizados por categoria no analytics; tooltip mostra % e valor real | RF-22.1 | y |
| Paleta | Paleta categórica da plataforma (8 cores, validada com o skill `dataviz`), com cor fixa por série ou por valor sobrescrevível | RF-22.2 | y |
| Linha de meta | `markLine` com valor fixo ou medida agregada de uma visão (mesma lógica de meta da F20, sem período) | RF-22.2 | y |
| Escala log | Só eixos de valor com todos os valores > 0; caso contrário aviso `LOG_SCALE_INVALID` e escala linear | ECharts não suporta log com ≤ 0 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Configurar gráficos básicos ⭐ MVP

**User Story**: Como construtor, quero criar gráficos dos tipos básicos a partir de uma visão.

**Why P1**: Visualização essencial.

**Acceptance Criteria**:
1. WHEN um objeto `CHART` `BAR` com eixo X `regiao`, medida `SUM(valor)` e série `canal` é carregado THEN o analytics SHALL devolver um `option` com uma série por canal e categorias por região.
2. WHERE `stack=STACKED` the `option` SHALL usar o mesmo `stack` em todas as séries; WHERE `stack=PERCENT` the analytics SHALL normalizar os valores por categoria para somar 1.
3. WHERE `orientation=HORIZONTAL` the `option` SHALL trocar os eixos de categoria e valor.
4. WHEN o tipo é `LINE` ou `AREA` THEN o `option` SHALL usar séries de linha (área com `areaStyle`), empilháveis.
5. WHEN o tipo é `PIE` ou `DONUT` THEN o `option` SHALL ter uma série `pie` (rosca com raio interno de 50%) com uma fatia por categoria.
6. WHEN o tipo é `COMBO` THEN o `option` SHALL conter séries `bar` e `line` conforme a configuração, com `yAxisIndex` 1 para as medidas do eixo secundário.
7. WHEN o tipo é `SCATTER` THEN o `option` SHALL usar a 1ª medida no eixo X e a 2ª no eixo Y.
8. WHEN o tipo é `FUNNEL` THEN o `option` SHALL ter uma série `funnel` ordenada de forma decrescente.
9. IF a configuração viola as restrições do tipo (ex.: `PIE` com 2 medidas) THEN o sistema SHALL responder 400 com erro em `config.<campo>`.

**Independent Test**: pytest do transform `chart` com snapshot de `option` por tipo.

---

### P1: Eixos, legenda, rótulos, cores e meta ⭐ MVP

**User Story**: Como construtor, quero ajustar eixos, legenda, rótulos, cores e uma linha de meta.

**Why P1**: RF-22.2.

**Acceptance Criteria**:
1. WHERE `axis.min`/`axis.max` estão definidos the `option` SHALL fixar os limites do eixo de valor.
2. WHERE `axis.log=true` e todos os valores > 0 the `option` SHALL usar `type: 'log'`; IF algum valor ≤ 0 THEN o analytics SHALL manter escala linear e incluir o aviso `LOG_SCALE_INVALID`.
3. WHERE `legend.position` é `top`, `bottom`, `left`, `right` ou `hidden` the `option` SHALL posicionar ou ocultar a legenda.
4. WHERE `dataLabels=true` the `option` SHALL exibir rótulos, e o frontend SHALL formatá-los com a máscara da medida.
5. WHERE uma série ou valor tem cor fixa the `option` SHALL usá-la; as demais SHALL seguir a paleta da plataforma na ordem.
6. WHERE `targetLine` é fixo (ex.: 100.000) ou uma medida de visão the `option` SHALL conter um `markLine` horizontal nesse valor.
7. The tooltip SHALL exibir os valores formatados com a máscara de cada medida.

**Independent Test**: Snapshots do `option` + teste de componente dos formatters.

---

### P1: Trocar tipo e muitas categorias ⭐ MVP

**User Story**: Como construtor, quero trocar o tipo de gráfico sem refazer a configuração e não travar com categorias demais.

**Why P1**: RF-22.4 e RN-22.1.

**Acceptance Criteria**:
1. WHEN o construtor troca `BAR` por `LINE` no editor THEN o frontend SHALL manter visão, dimensões, medidas, máscaras e cores.
2. WHEN o construtor troca para um tipo incompatível (ex.: `BAR` com 2 medidas → `PIE`) THEN o editor SHALL manter a configuração guardada e apontar o campo a ajustar antes de salvar.
3. WHEN o resultado tem mais categorias que `maxCategories` (padrão 50) e a medida é `SUM` ou `COUNT` THEN o analytics SHALL manter as N−1 maiores e agrupar o restante em "Outros", incluindo o aviso `TOO_MANY_CATEGORIES`.
4. WHEN a medida é não aditiva (`AVG`, `MIN`, `MAX`, `COUNT_DISTINCT`) e excede o limite THEN o analytics SHALL manter as N primeiras e incluir o aviso, sem "Outros".
5. The frontend SHALL exibir os avisos num ícone do objeto.

**Independent Test**: pytest com 80 categorias para `SUM` e para `AVG`.

---

## Edge Cases

- WHEN o resultado está vazio THEN o frontend SHALL exibir "Sem dados para o período/filtro" no lugar do gráfico.
- IF a dimensão de série tem mais de 20 valores THEN o analytics SHALL agrupar as séries excedentes em "Outras séries" (aditivas) ou avisar (não aditivas), como no limite de categorias.
- WHEN valores são `null` numa linha THEN a série SHALL ter lacuna (`null`), não zero.
- WHEN a dimensão X é data com `dateGrain=MONTH` THEN os rótulos SHALL ser formatados `MM/yyyy` no frontend.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| CHT-01 | P1: Tipos básicos | RF-22.1 | Tasks | Pending |
| CHT-02 | P1: Restrições por tipo | RF-22.1 | Tasks | Pending |
| CHT-03 | P1: Eixos, legenda, rótulos | RF-22.2 | Tasks | Pending |
| CHT-04 | P1: Cores e meta | RF-22.2 | Tasks | Pending |
| CHT-05 | P1: Tooltip formatado | RF-22.3 | Tasks | Pending |
| CHT-06 | P1: Troca de tipo | RF-22.4 | Tasks | Pending |
| CHT-07 | P1: Muitas categorias | RN-22.1 | Tasks | Pending |
| CHT-08 | (F19) E2E de tela completa | CA-19.1, CA-19.2 | Tasks | Pending |

**Coverage:** 8 total, 8 mapped to tasks, 0 unmapped.

---

## Success Criteria

- [ ] Snapshot de `option` por tipo e por variação (empilhado, 100%, horizontal, combo).
- [ ] CA-19.1 e CA-19.2 cobertos por E2E com objetos reais.
