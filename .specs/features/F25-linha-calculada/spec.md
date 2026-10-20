# F25 — Linha Calculada Specification

**Prioridade:** V1 · **Depende de:** F21 · **Origem:** F25 do `produto.specs.md`
**Artefatos:** só spec (V1).

## Problem Statement

DREs e demonstrativos precisam de linhas que não vêm do banco: "Margem Bruta = Receita − CMV" e "Margem % = Margem Bruta / Receita", em ordem fixa e com estilo próprio. Essas linhas devem ser calculadas de forma idêntica na tela e na exportação.

## Goals

- [ ] Um DRE com 5 linhas calculadas, ordem fixa e linhas de origem ocultas é configurado sem SQL.
- [ ] O resultado chega pronto ao frontend e ao XLSX (RN-25.1).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Linhas calculadas em gráficos | Apenas no Objeto Consulta (tabela/pivot) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Onde calcula | No analytics (Python), no transform `table`/`pivot`, após a consulta | AD-007 e RN-25.1 | y |
| Linguagem | Expressão aritmética sobre rótulos de linha `[Receita] - [CMV]`, com `+ - * /`, parênteses, números e `ABS`/`ROUND`; avaliador próprio por AST (sem `eval`) | Segurança (AD-007) | y |
| Rótulo da linha | Valor da primeira dimensão da linha (ex.: conta contábil) | RF-25.1 | y |
| Divisão por zero | Resultado `null` | Consistente com OBExpr (F09) | y |
| Ciclos | Detectados por grafo, erro `CALC_ROW_CIRCULAR` com o caminho | RF-25.3 / RN-09.3 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Linhas calculadas por expressão [V1]

**User Story**: Como controller, quero inserir linhas calculadas a partir de outras linhas da tabela.

**Why P1**: Núcleo do DRE.

**Acceptance Criteria**:
1. WHEN uma linha "Margem Bruta" com `[Receita] - [CMV]` é configurada THEN o analytics SHALL calculá-la para cada coluna de medida (e para cada coluna do pivot) e inseri-la após a linha indicada.
2. WHEN uma linha calculada referencia outra linha calculada THEN o analytics SHALL avaliá-las em ordem topológica.
3. IF as linhas calculadas formam um ciclo THEN o sistema SHALL rejeitar a configuração com `CALC_ROW_CIRCULAR` e o caminho.
4. IF a expressão referencia um rótulo inexistente no resultado THEN a linha SHALL ter valores `null` e um aviso `CALC_ROW_MISSING_REF`.
5. The avaliador SHALL aceitar apenas números, rótulos entre colchetes, `+ - * /`, parênteses, `ABS` e `ROUND`, e rejeitar qualquer outra construção.

**Independent Test**: pytest do avaliador com casos de DRE e payloads maliciosos.

---

### P1: Ordem fixa, ocultação e estilo [V1]

**User Story**: Como controller, quero fixar a ordem das linhas, ocultar linhas de origem e destacar totais.

**Why P1**: RF-25.1, RF-25.4.

**Acceptance Criteria**:
1. WHERE uma ordem fixa por lista de rótulos está configurada the analytics SHALL ordenar as linhas por essa lista, com as não listadas ao final.
2. WHERE uma linha de origem está marcada como oculta the analytics SHALL usá-la nos cálculos e removê-la do resultado.
3. The linha calculada SHALL carregar máscara e estilo próprios (negrito, fundo), aplicados pelo frontend e pelo XLSX.

**Independent Test**: Snapshot do transform + XLSX com estilos.

---

## Edge Cases

- WHEN a divisão tem denominador 0 THEN o valor SHALL ser `null`.
- IF a expressão tem mais de 500 caracteres THEN o sistema SHALL rejeitar com `CALC_ROW_TOO_LONG`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| CROW-01 | P1: Expressões por coluna/pivot | RF-25.1, RF-25.2 | - | Pending |
| CROW-02 | P1: Encadeamento sem ciclo | RF-25.3 | - | Pending |
| CROW-03 | P1: Avaliador seguro | AD-007 | - | Pending |
| CROW-04 | P1: Ordem, ocultação, estilo | RF-25.4 | - | Pending |
| CROW-05 | P1: Cálculo no backend/analytics para tela e exportação | RN-25.1 | - | Pending |

**Coverage:** 5 total, 0 mapped to tasks (V1).

---

## Success Criteria

- [ ] DRE de exemplo idêntico na tela e no XLSX.
