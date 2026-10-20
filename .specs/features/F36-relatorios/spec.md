# F36 — Relatórios (Report) Specification

**Prioridade:** V2 · **Depende de:** F12, F28, F29 · **Origem:** F36 do `produto.specs.md`
**Artefatos:** só spec (V2).

## Problem Statement

Dashboards interativos não substituem relatórios paginados para impressão: listagem de pedidos por cliente com quebras e totais, fechamento mensal. O Report gera documentos com cabeçalho e rodapé de página, grupos com subtotais e totais gerais, prontos para PDF e XLSX.

## Goals

- [ ] Relatório "Pedidos por cliente" com quebra por cliente, subtotais e total geral, em PDF A4 retrato.
- [ ] Mesmos dados e totais no HTML paginado, no PDF e no XLSX.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Editor de layout pixel a pixel (estilo Jasper) | Layout estruturado (bandas) é suficiente para PME |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Geração de PDF (D-07) | HTML + CSS de impressão renderizado por Playwright/Chromium no analytics | AD-011 | y |
| Agrupamentos e subtotais | Calculados no analytics (polars `group_by` com subtotais) sobre linhas detalhadas do motor (com RLS) | Manipulação tabular em Python (AD-001) | y |
| Parâmetros | Mesmos tipos de controle da F28 | RF-36.1 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Definir e executar relatórios [V2]

**User Story**: Como construtor, quero montar um relatório com parâmetros, grupos, colunas e totais.

**Why P1**: RF-36.1, RF-36.2.

**Acceptance Criteria**:
1. WHEN um relatório com visão, parâmetros, quebras, colunas com máscara, totais, orientação e tamanho de página é executado com parâmetros preenchidos THEN o sistema SHALL exibir um HTML paginado com cabeçalho e rodapé de página (logo, título, parâmetros usados, número da página).
2. WHEN o usuário baixa em PDF THEN o sistema SHALL gerar o PDF no servidor com o mesmo conteúdo e paginação do HTML.
3. WHEN o usuário baixa em XLSX THEN o sistema SHALL gerar planilha com os grupos, subtotais e total geral.
4. The relatório SHALL aplicar a RLS do usuário e respeitar `maxExportRows`.

**Independent Test**: IT comparando totais entre HTML, PDF (texto extraído) e XLSX.

---

### P2: Blocos de resumo [V2]

**User Story**: Como construtor, quero um gráfico ou KPI no topo do relatório.

**Why P2**: RF-36.3.

**Acceptance Criteria**:
1. WHEN o relatório tem blocos de objetos no topo THEN o PDF SHALL incluí-los renderizados antes da listagem.

**Independent Test**: Snapshot visual do PDF.

---

## Edge Cases

- IF o relatório excede 2.000 páginas THEN o sistema SHALL recusar com `REPORT_TOO_LARGE` e sugerir filtros.
- WHEN um grupo não tem linhas THEN ele SHALL ser omitido.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| RPT-01 | P1: Definição (parâmetros, quebras, colunas, totais, página) | RF-36.1 | - | Pending |
| RPT-02 | P1: HTML paginado | RF-36.2 | - | Pending |
| RPT-03 | P1: PDF no servidor | RF-36.2, RN-36.1 | - | Pending |
| RPT-04 | P1: XLSX | RF-36.2 | - | Pending |
| RPT-05 | P2: Blocos de resumo | RF-36.3 | - | Pending |

**Coverage:** 5 total, 0 mapped to tasks (V2).

---

## Success Criteria

- [ ] Totais idênticos nos três formatos.
