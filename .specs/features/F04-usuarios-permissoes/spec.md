# F04 — Usuários, Grupos e Permissões Specification

**Prioridade:** MVP · **Depende de:** F03 · **Origem:** F04 do `produto.specs.md`

## Problem Statement

Hoje só existe o admin do setup. A empresa precisa cadastrar pessoas, organizá-las em grupos e definir o que cada uma pode **fazer** (papel + flags) e **ver/editar** (ACL por recurso). Todas as features de conteúdo (telas, visões, tabelas, conexões) dependem de um serviço central de autorização que esta feature entrega.

## Goals

- [ ] Admin gerencia usuários e grupos pela UI.
- [ ] Um serviço único (`AccessControl`) responde "usuário X pode nível N no recurso R?" para todos os módulos.
- [ ] Remover alguém de um grupo tira o acesso na próxima requisição, sem novo login (CA-04.1).

## Out of Scope

| Feature | Reason |
| --- | --- |
| "Deny" explícito | RF-04.3: não existe no MVP |
| Convite por e-mail com ativação | F03 V1 (RF-03.9) |
| Pastas de menu em si (criação/edição) | F30; aqui só o mecanismo de herança |
| RLS | F05 (usa os atributos definidos aqui) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Criação de usuário sem convite | Admin define senha temporária; usuário nasce com `must_change_password=true` | Convite é V1 | y |
| Atributos multivalorados | `PUT /users/{id}/attributes` aceita string ou array de strings por chave | Necessário para `filial IN ({{usuario.atributo.filiais}})` (F05) | y |
| Hierarquia de níveis | `VIEW < EDIT < MANAGE`; níveis de dados independentes: `MANAGE` implica todos os `DATA_*`, `EDIT` implica `DATA_READ` | Evita configurar dados duas vezes para quem gerencia a tabela | y |
| Teto do VIEWER (RN-04.2) | VIEWER fica limitado a `VIEW` + níveis `DATA_*` em qualquer tipo de recurso | A spec permite VIEWER alimentar UDTs | y |
| Flags e papel de BUILDER | Flags são concedidas a usuários ou grupos; `ADMIN` tem todas implicitamente | RN-04.1 | y |
| Herança de ACL | Recurso com `inherit=true` e sem entradas próprias usa a ACL da pasta pai, recursivamente; havendo entradas próprias, só elas valem | RF-04.4 ("salvo se tiver ACL própria") | y |
| Resolução de permissão | Por requisição, direto no banco (sem cache em sessão) | CA-04.1 exige efeito imediato | y |
| Tipos de recurso | `SCREEN`, `VIEW`, `UDT_TABLE`, `MENU_FOLDER`, `DATASOURCE`, `BROWSER`, `REPORT` | Lista do RF-04 + Browser (F10) | y |
| Exclusão de usuário | Soft delete (`deleted_at`), encerra sessões, remove das listas e grupos | RN-04.3 | y |
| "Verificar e-mails" (RF-04.7) | Relatório de e-mails duplicados (case-insensitive) e de formato inválido; "não verificado" entra na V1 | Verificação de e-mail é V1 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Gerenciar usuários ⭐ MVP

**User Story**: Como admin, quero cadastrar, editar, inativar e excluir usuários para controlar quem acessa a plataforma.

**Why P1**: Sem usuários, não há consumidores.

**Acceptance Criteria**:
1. WHEN o admin chama `POST /api/v1/users` com nome, e-mail, papel, senha temporária, grupos e flags válidos THEN o sistema SHALL criar o usuário com `must_change_password=true` e responder 201.
2. IF o e-mail já existe (case-insensitive, entre não excluídos) THEN o sistema SHALL responder 409 com `code=EMAIL_IN_USE`.
3. WHEN o admin chama `PUT /api/v1/users/{id}/attributes` com `{ "vendedor_id": "123", "filiais": ["SC01","SC02"] }` THEN o sistema SHALL substituir todos os atributos do usuário por esses.
4. WHEN o admin altera o status de um usuário para `INACTIVE` ou `BLOCKED` THEN o sistema SHALL publicar `UserAccessRevoked` e as sessões dele SHALL ser encerradas (F03).
5. WHEN o admin chama `DELETE /api/v1/users/{id}` THEN o sistema SHALL marcar `deleted_at`, remover o usuário dos grupos e encerrar as sessões, preservando os registros de auditoria.
6. IF a operação deixaria zero admins ativos (inativar, bloquear, excluir ou trocar o papel do último admin) THEN o sistema SHALL responder 409 com `code=LAST_ADMIN`.
7. WHEN o admin chama `GET /api/v1/users?status=&groupId=&role=` THEN o sistema SHALL responder a lista paginada com a coluna `lastLoginAt`.
8. IF um não-admin chama qualquer endpoint `/api/v1/users` de escrita THEN o sistema SHALL responder 403.

**Independent Test**: IT de CRUD completo + regra do último admin.

---

### P1: Gerenciar grupos ⭐ MVP

**User Story**: Como admin, quero agrupar usuários para conceder acessos em lote.

**Why P1**: ACL por grupo é o caso comum em PMEs.

**Acceptance Criteria**:
1. WHEN o admin cria um grupo com nome único THEN o sistema SHALL responder 201.
2. WHEN o admin chama `PUT /api/v1/groups/{id}/members` com `userIds` THEN o sistema SHALL substituir os membros do grupo.
3. WHEN o admin exclui um grupo THEN o sistema SHALL remover suas entradas de ACL, flags e associações.
4. IF o nome do grupo já existe THEN o sistema SHALL responder 409 com `code=GROUP_NAME_IN_USE`.

**Independent Test**: IT: criar grupo, conceder ACL, excluir grupo, verificar que as entradas sumiram.

---

### P1: Autorização por recurso (ACL) ⭐ MVP

**User Story**: Como construtor, quero definir quem vê ou edita cada recurso para compartilhar conteúdo com segurança.

**Why P1**: Base de todo o controle de conteúdo.

**Acceptance Criteria**:
1. WHEN um módulo registra um recurso criado pelo usuário U THEN o `AccessControl` SHALL conceder `MANAGE` a U nesse recurso.
2. The permissão efetiva SHALL ser a união das entradas do usuário e de todos os seus grupos.
3. WHILE o usuário tem papel `ADMIN` the `AccessControl` SHALL conceder qualquer nível em qualquer recurso.
4. WHILE o usuário tem papel `VIEWER` the `AccessControl` SHALL negar `EDIT` e `MANAGE` mesmo que a ACL conceda, permitindo `VIEW` e `DATA_*`.
5. WHEN um recurso tem `inherit=true` e nenhuma entrada própria THEN o `AccessControl` SHALL usar as entradas da pasta pai, recursivamente até a raiz.
6. WHEN `PUT /api/v1/acl/{resourceType}/{resourceId}` é chamado por quem tem `MANAGE` THEN o sistema SHALL substituir as entradas do recurso.
7. IF quem chama `PUT /acl/...` não tem `MANAGE` no recurso THEN o sistema SHALL responder 403.
8. WHEN Pedro, no grupo "Comercial" com `VIEW` na tela "Vendas", é removido do grupo THEN a próxima consulta de acesso de Pedro SHALL negar `VIEW` em "Vendas".

**Independent Test**: CA-04.1 e CA-04.2 em IT sobre um recurso fictício `SCREEN`.

---

### P1: Permissões granulares (flags) ⭐ MVP

**User Story**: Como admin, quero liberar recursos sensíveis (Editor SQL, upload, exportação) só para quem precisa.

**Why P1**: Flags são exigidas por F09, F11, F17, F29.

**Acceptance Criteria**:
1. The sistema SHALL suportar as flags `SQL_EDITOR`, `CUSTOM_QUERY`, `UPLOAD`, `EXPORT` e `MANAGE_UDT_SCHEMA`, atribuíveis a usuário ou grupo.
2. WHEN `GET /api/v1/auth/me` é chamado THEN `permissions` SHALL conter as flags efetivas (usuário ∪ grupos; todas para `ADMIN`).
3. IF um endpoint protegido por flag é chamado por quem não a tem THEN o sistema SHALL responder 403 com `code=FLAG_REQUIRED` e `flag=<nome>`.

**Independent Test**: Usuário sem `EXPORT` recebe 403 num endpoint de teste anotado; ao entrar num grupo com `EXPORT`, recebe 200.

---

### P1: Visão de acesso efetivo ⭐ MVP

**User Story**: Como admin, quero ver exatamente o que um usuário enxerga para auditar e depurar acessos.

**Why P1**: RF-04.5.

**Acceptance Criteria**:
1. WHEN o admin chama `GET /api/v1/users/{id}/effective-access` THEN o sistema SHALL listar cada recurso acessível com tipo, nome, nível efetivo e origem (`DIRECT`, `GROUP:<nome>`, `INHERITED:<pasta>`, `ROLE_ADMIN`).
2. WHEN o admin chama `GET /api/v1/users/email-check` THEN o sistema SHALL listar e-mails duplicados (case-insensitive) e de formato inválido.

**Independent Test**: IT com acessos diretos, por grupo e herdados.

---

## Edge Cases

- IF `PUT /acl` inclui um principal inexistente ou excluído THEN o sistema SHALL responder 400 com erro em `entries[i].principalId`.
- IF `PUT /acl` inclui um nível `DATA_*` em recurso que não é `UDT_TABLE` THEN o sistema SHALL responder 400.
- WHEN a herança forma um ciclo de pastas (dado corrompido) THEN o `AccessControl` SHALL parar após 10 níveis e negar o acesso, logando um erro.
- IF um usuário excluído tenta entrar THEN o sistema SHALL responder 401 `INVALID_CREDENTIALS`.
- WHEN um e-mail de usuário excluído é reutilizado num novo cadastro THEN o sistema SHALL permitir (unicidade só entre não excluídos).

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| ACL-01 | P1: Usuários (CRUD) | RF-04.1, RN-04.3 | Tasks | Pending |
| ACL-02 | P1: Usuários (atributos) | RF-04.1 | Tasks | Pending |
| ACL-03 | P1: Usuários (status → sessões) | RN-03.2 | Tasks | Pending |
| ACL-04 | P1: Usuários (último admin) | RN-03.3 | Tasks | Pending |
| ACL-05 | P1: Usuários (listagem) | RF-04.7 | Tasks | Pending |
| ACL-06 | P1: Grupos | RF-04.2, RN-04.3 | Tasks | Pending |
| ACL-07 | P1: ACL (resolução, união, papéis) | RF-04.3, RN-04.1, RN-04.2, CA-04.1, CA-04.2 | Tasks | Pending |
| ACL-08 | P1: ACL (herança) | RF-04.4 | Tasks | Pending |
| ACL-09 | P1: ACL (criador MANAGE, API de ACL) | RF-04.6 | Tasks | Pending |
| ACL-10 | P1: Flags | Tabela de flags F04 | Tasks | Pending |
| ACL-11 | P1: Acesso efetivo | RF-04.5, RF-04.7 | Tasks | Pending |

**Coverage:** 11 total, 11 mapped to tasks, 0 unmapped.

---

## Success Criteria

- [ ] CA-04.1 e CA-04.2 automatizados.
- [ ] Todos os módulos posteriores autorizam exclusivamente via `AccessControl`, sem consultas próprias de ACL.
