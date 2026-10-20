# F27 — Drill Down e Drill Through Specification

**Prioridade:** V1 · **Depende de:** F19, F12 · **Origem:** F27 do `produto.specs.md`
**Artefatos:** só spec (V1).

## Problem Statement

Ao ver "Sul: R$ 2 mi", o gestor quer descer para estados, depois cidades, ou abrir as notas que compõem o número. Drill down navega numa hierarquia; drill through abre outra tela ou um detalhe com as linhas.

## Goals

- [ ] Clicar em "Sul" desce para estados da região Sul, com breadcrumb para voltar.
- [ ] O detalhe de linhas respeita RLS, filtros da tela e limite de linhas (RN-27.1).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Filtro cruzado entre objetos | Item próprio da F22 (RF-22.3, V1) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Hierarquias | Definidas na visão (F12 RF-12.3): lista ordenada de colunas; datas usam a hierarquia automática ano › trimestre › mês › dia | RF-27.1 | y |
| Estado do drill | Mantido no frontend e enviado no `data` do objeto como `drillPath` (filtros de igualdade + nível atual); o backend valida contra a hierarquia | Sem estado no servidor | y |
| Detalhe de linhas | Consulta detalhada (linhas cruas) via motor com filtros da tela + objeto + caminho de drill, até `maxQueryRows`; exportação exige `EXPORT` | RN-27.1 | y |
| Favoritar drill | Entra com a F31 (RF-27.4) | Dependência | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Drill down em hierarquia [V1]

**User Story**: Como gestor, quero clicar num valor e descer um nível na hierarquia.

**Why P1**: RF-27.1.

**Acceptance Criteria**:
1. WHEN o usuário clica em "Sul" num objeto cuja dimensão `regiao` pertence à hierarquia Região › Estado › Cidade THEN o objeto SHALL ser recarregado com dimensão `estado` e filtro `regiao = 'Sul'`.
2. WHEN o usuário clica num item do breadcrumb THEN o objeto SHALL voltar ao nível correspondente, removendo os filtros dos níveis abaixo.
3. The drill SHALL respeitar RLS e filtros da tela (RN-27.1).
4. IF o objeto já está no último nível THEN o clique SHALL não descer (cursor padrão).

**Independent Test**: E2E com hierarquia de 3 níveis.

---

### P1: Drill through e múltiplos destinos [V1]

**User Story**: Como gestor, quero abrir outra tela filtrada ou ver as linhas que compõem um valor.

**Why P1**: RF-27.2, RF-27.3.

**Acceptance Criteria**:
1. WHEN o drill through para uma tela é acionado THEN o frontend SHALL navegar para a tela destino com o valor clicado como filtro na URL (F28).
2. WHEN o drill "detalhe" é acionado THEN o sistema SHALL abrir um modal com as linhas cruas que compõem o valor, nas colunas configuradas, até o limite de linhas.
3. WHERE o objeto tem vários drills configurados the frontend SHALL exibir o menu de contexto "Detalhar por..." para escolher a dimensão ou o destino.
4. IF o usuário não tem `EXPORT` THEN o botão de exportar do detalhe SHALL ficar oculto.

**Independent Test**: IT do detalhe com RLS + teste de componente do menu.

---

## Edge Cases

- IF a tela destino não existe mais ou o usuário não tem `VIEW` nela THEN a opção SHALL não aparecer.
- WHEN o valor clicado é `null` THEN o filtro SHALL ser `IS_NULL`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| DRL-01 | P1: Drill down + breadcrumb | RF-27.1 | - | Pending |
| DRL-02 | P1: Drill through para tela | RF-27.2 | - | Pending |
| DRL-03 | P1: Detalhe de linhas | RF-27.2, RN-27.1 | - | Pending |
| DRL-04 | P1: Vários drills por objeto | RF-27.3 | - | Pending |
| DRL-05 | (F31) Favoritar drill | RF-27.4 | - | Pending |

**Coverage:** 5 total, 0 mapped to tasks (V1).

---

## Success Criteria

- [ ] Drill e detalhe nunca mostram linhas fora da RLS (teste com dois usuários).
