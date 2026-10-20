# F30 — Menus e Tela Inicial Specification

**Prioridade:** MVP · **Depende de:** F19, F04, F10 · **Origem:** F30 do `produto.specs.md`

## Problem Statement

Com telas e browsers criados, o consumidor precisa encontrá-los: um menu organizado em pastas, filtrado pelo que ele pode ver, uma tela inicial adequada ao seu papel, busca global e acesso ao próprio perfil. O admin precisa de um menu separado de administração.

## Goals

- [ ] Cada usuário vê no menu só itens a que tem acesso, e pastas vazias somem.
- [ ] Ao entrar, o usuário cai na tela inicial certa (usuário > grupo > global).
- [ ] Busca global encontra telas e browsers pelo nome em < 300 ms.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Item de menu apontando para Relatório | F36 (V2) |
| Favoritos na barra superior | F31 (V1) |
| Avisos (sino) | F32 (V2) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Estrutura | Árvore `app.menu_node` com pastas (`FOLDER`) e itens (`ITEM`: `SCREEN`, `BROWSER`, `LINK`), ordem e ícone | RF-30.1 | y |
| Pasta como recurso | Cada pasta é recurso `MENU_FOLDER` com ACL (F04) | RF-04.4 | y |
| Herança de ACL | Ao criar o primeiro item de um recurso numa pasta, a pasta vira a "pasta-mãe" do recurso (`resource_node.parent_folder_id`); itens adicionais em outras pastas são atalhos e não mudam a herança | Um recurso tem um único pai para herança | y |
| Quem edita o menu | ADMIN e BUILDER com `EDIT` na pasta (raiz: ADMIN e BUILDER) | RF-30.1 | y |
| Visibilidade de item | Item visível se o usuário tem `VIEW` no recurso alvo (`LINK`: visível a quem vê a pasta) | RF-30.2 | y |
| Tela inicial | Precedência: preferência do usuário (`preferences.homeScreenId`) > grupo de menor `priority` com tela inicial definida > global (`general.homeScreenId`) > primeira tela visível do menu > página "Bem-vindo" | RF-30.3 | y |
| Tela inicial inacessível | Se o usuário perdeu `VIEW` na tela configurada, pula para o próximo nível da precedência | Evita tela de erro no login | y |
| Busca global | Nome contém `q` (sem acento, case-insensitive) em telas e browsers visíveis, máx. 20 resultados | RF-30.4 | y |
| Link externo | Só `http`/`https`, abre em nova aba com `rel="noopener noreferrer"` | Segurança | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Menu filtrado por acesso ⭐ MVP

**User Story**: Como consumidor, quero ver no menu só o que posso acessar, organizado em pastas.

**Why P1**: Navegação principal.

**Acceptance Criteria**:
1. WHEN `GET /api/v1/menu` é chamado THEN o sistema SHALL devolver a árvore com apenas os itens cujo recurso o usuário pode ver (`VIEW`), na ordem configurada.
2. WHEN uma pasta não tem nenhum item visível (direta ou em subpastas) THEN o sistema SHALL omiti-la da árvore.
3. WHEN o usuário perde `VIEW` numa tela THEN o item SHALL sumir do menu na próxima chamada.
4. The menu SHALL ser calculado com no máximo 2 consultas de autorização em lote (`filterVisible`), independentemente do número de itens.

**Independent Test**: IT com 3 usuários e pastas com itens mistos.

---

### P1: Editar o menu ⭐ MVP

**User Story**: Como admin ou construtor, quero criar pastas e itens, ordenar e escolher ícones.

**Why P1**: RF-30.1.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/menu/nodes` cria uma pasta THEN o sistema SHALL registrar o recurso `MENU_FOLDER` com `MANAGE` para o criador.
2. WHEN um item `SCREEN` ou `BROWSER` é criado numa pasta e o recurso ainda não tem pasta-mãe THEN o sistema SHALL definir essa pasta como pai do recurso para herança de ACL.
3. WHEN `PUT /api/v1/menu/order` recebe a nova ordem e os novos pais THEN o sistema SHALL gravar posições e pais numa transação.
4. IF um item `LINK` tem esquema diferente de `http`/`https` THEN o sistema SHALL responder 400 em `url`.
5. IF quem edita não tem `EDIT` na pasta (ou não é ADMIN/BUILDER na raiz) THEN o sistema SHALL responder 403.
6. WHEN uma pasta com filhos é excluída THEN o sistema SHALL responder 409 `MENU_FOLDER_NOT_EMPTY`.
7. WHEN o recurso alvo de um item é excluído THEN o item SHALL ser removido do menu.

**Independent Test**: IT de criação, herança e reordenação.

---

### P1: Tela inicial ⭐ MVP

**User Story**: Como usuário, quero cair na tela certa ao entrar, e como admin quero definir isso por grupo.

**Why P1**: RF-30.3.

**Acceptance Criteria**:
1. WHEN `GET /api/v1/me/home` é chamado THEN o sistema SHALL devolver a tela inicial segundo a precedência usuário > grupo (menor `priority`) > global > primeira tela visível > `null`.
2. IF a tela de um nível não é visível ao usuário THEN o sistema SHALL passar para o próximo nível.
3. WHEN o admin define `homeScreenId` num grupo ou nas configurações gerais THEN o sistema SHALL validar que a tela existe.
4. WHEN o usuário escolhe "Usar como minha tela inicial" numa tela THEN o sistema SHALL gravar `preferences.homeScreenId`.
5. WHEN o login conclui sem `next` THEN o frontend SHALL navegar para a tela inicial devolvida, ou para "Bem-vindo" quando `null`.

**Independent Test**: Teste unitário da precedência + IT.

---

### P1: Barra superior e menu de administração ⭐ MVP

**User Story**: Como usuário, quero buscar telas, acessar meu perfil e, se for admin, as áreas de administração.

**Why P1**: RF-30.4, RF-30.5.

**Acceptance Criteria**:
1. WHEN `GET /api/v1/search?q=vend` é chamado THEN o sistema SHALL devolver até 20 telas e browsers visíveis cujo nome contém "vend", ignorando acentos e maiúsculas.
2. The barra superior SHALL ter busca global, painel de exportações (F29) e menu de perfil com preferências (tema), trocar senha e sair.
3. WHILE o usuário é ADMIN the frontend SHALL exibir o menu de administração com Configurações, Usuários, Grupos, Conexões, Auditoria, Sessões e Catálogos; WHILE é BUILDER SHALL exibir só Catálogos (categorias e máscaras); VIEWER não vê o menu de administração.
4. IF `q` tem menos de 2 caracteres THEN o sistema SHALL responder lista vazia.

**Independent Test**: IT de busca + teste de componente do layout por papel.

---

## Edge Cases

- WHEN a árvore tem mais de 6 níveis de pastas THEN o sistema SHALL responder 400 `MENU_TOO_DEEP` ao criar.
- IF `PUT /menu/order` cria ciclo de pastas THEN o sistema SHALL responder 400 `MENU_CYCLE`.
- WHEN a busca não encontra nada THEN o frontend SHALL exibir "Nada encontrado".
- IF o usuário não tem nenhuma tela visível THEN a home SHALL ser "Bem-vindo" com orientação para pedir acesso ao admin.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| MENU-01 | P1: Menu filtrado | RF-30.2 | Tasks | Pending |
| MENU-02 | P1: Editar menu (CRUD, ordem, links) | RF-30.1 | Tasks | Pending |
| MENU-03 | P1: Editar menu (pasta como recurso e herança) | RF-04.4 | Tasks | Pending |
| MENU-04 | P1: Tela inicial | RF-30.3 | Tasks | Pending |
| MENU-05 | P1: Busca global | RF-30.4 | Tasks | Pending |
| MENU-06 | P1: Barra superior e perfil | RF-30.4 | Tasks | Pending |
| MENU-07 | P1: Menu de administração por papel | RF-30.5 | Tasks | Pending |

**Coverage:** 7 total, 7 mapped to tasks, 0 unmapped.

---

## Success Criteria

- [ ] Menu com 200 itens responde em < 200 ms com 2 consultas de autorização.
- [ ] E2E do corte do MVP completo navega pelo menu (instalação → tela filtrada → exportação).
