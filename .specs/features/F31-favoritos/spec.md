# F31 — Favoritos Specification

**Prioridade:** V1 · **Depende de:** F19, F27 · **Origem:** F31 do `produto.specs.md`
**Artefatos:** só spec (V1).

## Problem Statement

Usuários voltam sempre às mesmas telas e aos mesmos números. Favoritos dão acesso rápido, e o painel pessoal "Meu Painel" reúne objetos de várias telas num só lugar, sem duplicá-los.

## Goals

- [ ] Favoritar uma tela em um clique e acessá-la pelo menu "Favoritos".
- [ ] "Meu Painel" com objetos de 3 telas diferentes, refletindo mudanças feitas nos originais.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Compartilhar o "Meu Painel" | É pessoal por definição |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| "Meu Painel" | Tela pessoal especial (uma por usuário), cujos objetos são referências `{ screenId, objectId }` com layout próprio | RF-31.2 | y |
| Dados do objeto fixado | Carregados pelo endpoint da tela original (autorização via tela original + RLS) | O objeto continua sendo do original | y |
| Perda de acesso | Favorito oculto (não apagado) enquanto o usuário não tiver `VIEW` no recurso | RN-31.1 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Favoritar telas [V1]

**User Story**: Como usuário, quero marcar telas como favoritas.

**Why P1**: RF-31.1.

**Acceptance Criteria**:
1. WHEN o usuário clica na estrela de uma tela THEN o sistema SHALL adicioná-la aos favoritos e o menu "Favoritos" SHALL listá-la.
2. WHEN o usuário perde `VIEW` na tela THEN o favorito SHALL deixar de aparecer, sem ser apagado; WHEN recupera o acesso THEN SHALL reaparecer.

**Independent Test**: IT de favoritos com mudança de ACL.

---

### P1: Meu Painel [V1]

**User Story**: Como usuário, quero fixar objetos de qualquer tela num painel pessoal com layout próprio.

**Why P1**: RF-31.2.

**Acceptance Criteria**:
1. WHEN o usuário escolhe "Fixar no Meu Painel" num objeto THEN o sistema SHALL adicionar uma referência ao objeto no painel pessoal do usuário.
2. WHEN o objeto original é alterado THEN o painel SHALL exibir a versão atual na próxima carga.
3. IF o objeto original é excluído THEN o painel SHALL exibir "Objeto removido" no lugar, com ação de desafixar.
4. The layout do painel SHALL ser editável apenas pelo próprio usuário.

**Independent Test**: E2E fixando objetos de duas telas.

---

### P2: Favoritar drill [V1]

**User Story**: Como usuário, quero salvar um caminho de drill para voltar direto a ele.

**Why P2**: RF-31.3 / RF-27.4.

**Acceptance Criteria**:
1. WHEN o usuário favorita o drill atual THEN o sistema SHALL salvar objeto, níveis e valores, e abrir o favorito SHALL restaurar o mesmo caminho.

**Independent Test**: Teste de componente.

---

## Edge Cases

- WHEN o mesmo objeto é fixado duas vezes THEN o sistema SHALL manter uma só referência.
- IF o painel passa de 50 objetos THEN o sistema SHALL responder 400 `MY_PANEL_FULL`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| FAV-01 | P1: Favoritar telas | RF-31.1 | - | Pending |
| FAV-02 | P1: Ocultar sem apagar | RN-31.1 | - | Pending |
| FAV-03 | P1: Meu Painel | RF-31.2 | - | Pending |
| FAV-04 | P2: Favoritar drill | RF-31.3, RF-27.4 | - | Pending |

**Coverage:** 4 total, 0 mapped to tasks (V1).

---

## Success Criteria

- [ ] Nenhum favorito expõe tela sem `VIEW`.
