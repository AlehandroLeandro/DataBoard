# F18 — Consulta Customizada (Construtor Visual) Specification

**Prioridade:** V1 · **Depende de:** F12, F21, F22 · **Origem:** F18 do `produto.specs.md`
**Artefatos:** só spec (V1).

## Problem Statement

Usuários sem SQL querem fazer perguntas ad hoc ("vendas por vendedor e mês, só região Sul") sem pedir ao construtor uma tela nova. O construtor visual permite arrastar dimensões e medidas de uma visão, ver o resultado em tabela/pivot ou gráfico e salvar como objeto.

## Goals

- [ ] Usuário com `CUSTOM_QUERY` monta uma consulta de 2 dimensões e 1 medida em < 1 min, sem SQL.
- [ ] "Explorar" um objeto existente abre o construtor com a configuração dele, sem alterar o original.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Joins entre visões no construtor | Visões JOIN são da F12 (V1); o construtor opera sobre uma visão |
| Salvar consultas pessoais sem tela | Coberto por "Salvar como objeto" numa tela (pode ser pessoal) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Execução | Pelo motor (F12) com RLS; pós-processamento pelos transforms `table`/`pivot`/`chart` do analytics | RN-18.1 + AD-003 | y |
| Pivot | Transform `pivot` no analytics (polars `pivot`), o mesmo do pivot da F21 (V1) | Uma implementação para os dois | y |
| "Explorar" | Disponível a quem tem `VIEW` na tela, mesmo sem `CUSTOM_QUERY`, mas limitado à visão e às colunas do objeto original | RF-18.4 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Montar consulta por arrastar e soltar [V1]

**User Story**: Como analista sem SQL, quero arrastar colunas para Linhas, Colunas, Valores e Filtros e ver o resultado.

**Why P1**: RF-18.1, RF-18.2.

**Acceptance Criteria**:
1. WHEN o usuário arrasta dimensões para "Linhas" e medidas para "Valores" THEN o frontend SHALL executar a consulta pelo motor e exibir o resultado em tabela.
2. WHEN uma dimensão é colocada em "Colunas (pivot)" THEN o analytics SHALL pivotar o resultado e o frontend SHALL exibir a tabela pivot.
3. WHEN o usuário troca para "Gráfico" THEN o frontend SHALL exibir o mesmo resultado como gráfico básico (F22), mantendo a configuração.
4. IF o usuário não tem a flag `CUSTOM_QUERY` THEN o sistema SHALL responder 403 ao abrir o construtor livre.
5. The consultas do construtor SHALL passar pelo motor com RLS (RN-18.1).

**Independent Test**: E2E montando "valor por vendedor × mês".

---

### P1: Salvar e explorar [V1]

**User Story**: Como analista, quero salvar a consulta como objeto e explorar objetos existentes sem alterá-los.

**Why P1**: RF-18.3, RF-18.4.

**Acceptance Criteria**:
1. WHEN o usuário escolhe "Salvar como objeto" numa tela existente ou nova THEN o sistema SHALL criar um objeto `QUERY` ou `CHART` com a configuração atual.
2. WHEN um usuário com `VIEW` na tela escolhe "Explorar" num objeto THEN o frontend SHALL abrir o construtor com a configuração do objeto, e nenhuma alteração SHALL ser gravada no original.

**Independent Test**: Teste de componente + IT de criação de objeto.

---

## Edge Cases

- IF a consulta resultar em mais de 1.000 colunas de pivot THEN o analytics SHALL limitar e avisar `PIVOT_TOO_WIDE`.
- WHEN nenhuma medida é escolhida THEN o resultado SHALL listar as combinações distintas das dimensões.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| CQ-01 | P1: Arrastar e soltar | RF-18.1 | - | Pending |
| CQ-02 | P1: Tabela/pivot/gráfico | RF-18.2 | - | Pending |
| CQ-03 | P1: Permissão e RLS | RN-18.1 | - | Pending |
| CQ-04 | P1: Salvar como objeto | RF-18.3 | - | Pending |
| CQ-05 | P1: Explorar | RF-18.4 | - | Pending |

**Coverage:** 5 total, 0 mapped to tasks (V1).

---

## Success Criteria

- [ ] Nenhuma consulta do construtor ignora RLS (teste com dois usuários).
