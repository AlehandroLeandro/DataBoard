# F26 — Destaques e Sinalizadores Specification

**Prioridade:** V1 · **Depende de:** F20, F21, F22 · **Origem:** F26 do `produto.specs.md`
**Artefatos:** só spec (V1).

## Problem Statement

Gestores precisam enxergar exceções de relance: metas abaixo de 80% em vermelho, faixas de desempenho com semáforo, gradiente de calor numa tabela. Regras reutilizáveis evitam reconfigurar o mesmo critério em cada objeto.

## Goals

- [ ] Uma regra "atingimento: <80% vermelho, 80–100% amarelo, ≥100% verde" cadastrada uma vez e aplicada a KPIs, tabelas e gráficos.
- [ ] A exportação XLSX reproduz as cores (RN-26.1).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Regras por expressão SQL | Condições cobrem comparação com valor, coluna, meta e token |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Onde avalia | Frontend sobre valores crus (RN-26.1); o XLSX reproduz no analytics com `conditional_format`/formatos do `xlsxwriter` | RN-26.1 | y |
| Precedência | Regras locais do objeto antes das reutilizáveis; dentro de cada grupo, a primeira que casa vence | Previsível | y |
| Escala de cor | Gradiente de 2 ou 3 pontos (mín, meio opcional, máx), calculado sobre o resultado visível | RF-26.1 | y |
| Acessibilidade | Sinalizadores sempre acompanham ícone com forma distinta, não só cor | Daltonismo | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Formatação condicional [V1]

**User Story**: Como construtor, quero colorir células, linhas e números conforme condições.

**Why P1**: RF-26.1, RF-26.4.

**Acceptance Criteria**:
1. WHEN uma regra "valor < 0 → texto vermelho" é aplicada a uma coluna THEN o frontend SHALL colorir as células que satisfazem a condição.
2. WHERE a condição compara com outra coluna da linha, com a meta ou com um token the frontend SHALL avaliar usando os valores crus da linha.
3. WHERE a regra é de escala de cor the frontend SHALL aplicar o gradiente entre o mínimo e o máximo da coluna no resultado.
4. WHEN a regra se aplica a um Objeto Valor THEN o frontend SHALL colorir o número ou o card; WHEN se aplica a um gráfico THEN SHALL colorir a barra ou o ponto correspondente.

**Independent Test**: Testes de componente por tipo de objeto.

---

### P1: Sinalizadores e regras reutilizáveis [V1]

**User Story**: Como construtor, quero ícones por faixa e um catálogo central de regras.

**Why P1**: RF-26.2, RF-26.3.

**Acceptance Criteria**:
1. WHEN uma regra de sinalizador tem faixas <80%, 80–100% e ≥100% THEN o frontend SHALL exibir o ícone correspondente a cada valor (semáforo, setas, estrelas ou bandeiras).
2. WHEN uma regra reutilizável é alterada no catálogo THEN todos os objetos que a usam SHALL refletir a mudança na próxima renderização.
3. IF uma regra reutilizável usada por objetos é excluída THEN o sistema SHALL responder 409 `HIGHLIGHT_RULE_IN_USE`.
4. WHEN o objeto é exportado em XLSX THEN o analytics SHALL reproduzir cores de texto e fundo das regras de valor fixo.

**Independent Test**: IT do catálogo + pytest do XLSX com cores.

---

## Edge Cases

- WHEN o valor é `null` THEN nenhuma regra SHALL casar, exceto uma regra explícita "está vazio".
- IF as faixas de um sinalizador se sobrepõem THEN o sistema SHALL rejeitar a regra com `HIGHLIGHT_RANGES_OVERLAP`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| HL-01 | P1: Condições e estilos | RF-26.1 | - | Pending |
| HL-02 | P1: Escala de cor | RF-26.1 | - | Pending |
| HL-03 | P1: Aplicação por tipo de objeto | RF-26.4 | - | Pending |
| HL-04 | P1: Sinalizadores | RF-26.2 | - | Pending |
| HL-05 | P1: Regras reutilizáveis | RF-26.3 | - | Pending |
| HL-06 | P1: Cores no XLSX | RN-26.1 | - | Pending |

**Coverage:** 6 total, 0 mapped to tasks (V1).

---

## Success Criteria

- [ ] Mesma regra produz mesmas cores na tela e no XLSX.
