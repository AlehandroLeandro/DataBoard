# F16 — Listas de Valores (LOV) Specification

**Prioridade:** V1 · **Depende de:** F12, F28 · **Origem:** F16 do `produto.specs.md`
**Artefatos:** só spec (V1).

## Problem Statement

Filtros e campos de seleção precisam de opções com rótulos amigáveis ("SC — Santa Catarina"), às vezes dependentes de outro filtro (Estado → Cidade) e sempre limitadas ao que o usuário pode ver. No MVP, os filtros usam os valores distintos da coluna. A LOV dá controle sobre valor, rótulo, ordem e encadeamento.

## Goals

- [ ] Filtro Cidade dependente de Estado mostra só cidades do estado selecionado presentes nos dados visíveis ao usuário (CA-28.2).
- [ ] LOVs grandes (> 500 itens) com busca e paginação no servidor.

## Out of Scope

| Feature | Reason |
| --- | --- |
| LOV como tipo de campo de UDT | Coberto por `SELECT`/`REFERENCE` (F09) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Execução das LOVs dinâmicas | Pelo motor de consulta (F12) com RLS do usuário | RF-16.2 | y |
| Encadeamento | `dependsOn` mapeia filtros-pai para colunas da visão da LOV, aplicados como filtros `IN` | RF-16.1, RF-28.5 | y |
| Ao mudar o pai | Valores do filho que não existem mais nas opções são removidos da seleção | RF-28.5 | y |
| Dependências | LOV → VIEW/VIEW_COLUMN; coluna de visão → LOV no registro de dependências | AD-019 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: LOVs estáticas e dinâmicas [V1]

**User Story**: Como construtor, quero cadastrar listas de valores fixas ou vindas de uma visão.

**Why P1**: RF-16.1.

**Acceptance Criteria**:
1. WHEN uma LOV estática é criada com pares valor/rótulo THEN `GET /api/v1/lovs/{id}/values` SHALL devolvê-los na ordem cadastrada.
2. WHEN uma LOV dinâmica é criada com visão, coluna valor, coluna rótulo e filtro opcional THEN `values` SHALL devolver os pares distintos ordenados pelo rótulo.
3. The LOV dinâmica SHALL aplicar a RLS do usuário, devolvendo só opções presentes nos dados que ele pode ver.
4. WHEN a LOV tem mais de 500 itens THEN `values` SHALL exigir paginação (`page`, máx. 100 por página) e aceitar busca `q`.

**Independent Test**: IT com dois usuários sob RLS.

---

### P1: Encadeamento e associação a colunas [V1]

**User Story**: Como construtor, quero filtros dependentes e LOVs associadas a colunas da visão.

**Why P1**: RF-16.1, RF-16.4, RF-28.5, CA-28.2.

**Acceptance Criteria**:
1. WHEN `values` recebe `dependsOn={"estado":["SC"]}` THEN o sistema SHALL devolver só cidades de SC presentes nos dados visíveis ao usuário.
2. WHEN uma coluna de visão tem LOV associada THEN filtros sobre essa coluna SHALL usar a LOV automaticamente em vez dos valores distintos.
3. WHEN o filtro pai muda THEN o frontend SHALL limpar as seleções do filho que não estão nas novas opções.
4. IF uma LOV usada por colunas ou filtros é excluída THEN o sistema SHALL responder 409 `LOV_IN_USE`.

**Independent Test**: CA-28.2 em E2E.

---

## Edge Cases

- IF a visão da LOV dinâmica falha THEN o filtro SHALL exibir erro só no controle, sem quebrar a tela.
- WHEN um valor da LOV estática se repete THEN o sistema SHALL responder 400 `DUPLICATE_LOV_VALUE`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| LOV-01 | P1: Estáticas e dinâmicas | RF-16.1 | - | Pending |
| LOV-02 | P1: RLS nas opções | RF-16.2 | - | Pending |
| LOV-03 | P1: Busca e paginação | RF-16.3 | - | Pending |
| LOV-04 | P1: Encadeamento | RF-16.1, RF-28.5, CA-28.2 | - | Pending |
| LOV-05 | P1: Associação a colunas | RF-16.4, RF-12.3 | - | Pending |

**Coverage:** 5 total, 0 mapped to tasks (V1).

---

## Success Criteria

- [ ] CA-28.2 automatizado.
