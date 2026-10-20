# F04 — Usuários, Grupos e Permissões Design

**Spec**: `.specs/features/F04-usuarios-permissoes/spec.md`
**Status**: Draft

---

## Architecture Overview

```mermaid
graph TD
    M[Módulos: dashboard, semantic, udt, datasource...] -->|check / filterVisible / registerResource| AC[AccessControl]
    AC --> CU[CurrentUserProvider]
    CU --> UA[(user_account + groups + flags)]
    AC --> ACL[(acl_entry)]
    AC --> RN[(resource_node: parent, inherit, owner)]
    PE[DataBoardPermissionEvaluator] --> AC
    FG[@RequiresFlag aspect] --> CU
    UC[UserController / GroupController / AclController] --> US[UserService / GroupService / AclAdminService]
    US -.UserAccessRevoked.-> F03[F03 AccessRevokedListener]
```

Módulos de conteúdo nunca leem `acl_entry` diretamente. Eles chamam a interface pública `AccessControl` do módulo `security` (exposta como `@NamedInterface` no Spring Modulith).

---

## Code Reuse Analysis

### Existing Components to Leverage

| Component | Location | How to Use |
| --- | --- | --- |
| `UserAccount`, repositório, `Argon2PasswordEncoder` | `security/user` (F01/F03) | Estender com `deleted_at`, grupos, flags, atributos |
| `PasswordPolicyValidator` | `security/auth` (F03) | Validar a senha temporária |
| `UserAccessRevoked` + listener | `security/session` (F03) | Publicar ao inativar, bloquear ou excluir |
| `PageResponse`, `ProblemDetailsHandler` | F00 | Listagens e erros |
| `AuthController.me` | F03 | Preencher `permissions` com as flags efetivas |

### Integration Points

| System | Integration Method |
| --- | --- |
| Spring Security | `PermissionEvaluator` para `@PreAuthorize("hasPermission(#id,'SCREEN','EDIT')")` |
| Outros módulos | `AccessControl.registerResource(type, id, parentFolderId, ownerId)` no `create` |

---

## Components

### `security.access.AccessControl` (interface pública)

- **Purpose**: API única de autorização.
- **Location**: `backend/src/main/java/io/databoard/security/access/AccessControl.java`
- **Interfaces**:
  - `boolean can(CurrentUser u, ResourceRef r, AccessLevel level)`
  - `void require(CurrentUser u, ResourceRef r, AccessLevel level)`: lança 403 `ACCESS_DENIED`.
  - `Set<UUID> filterVisible(CurrentUser u, ResourceType t, Collection<UUID> ids)`: uma query para listagens.
  - `void registerResource(ResourceRef r, UUID parentFolderId, UUID ownerId)`: grava o nó e concede `MANAGE` ao dono.
  - `void moveResource(ResourceRef r, UUID newParentFolderId)` e `void unregisterResource(ResourceRef r)`.
- **Dependencies**: `AclResolver`, `CurrentUserProvider`.

### `security.access.AclResolver`

- **Purpose**: Calcula o nível efetivo (lógica pura sobre dados carregados).
- **Interfaces**: `EffectiveAccess resolve(CurrentUser u, ResourceNodeChain chain, List<AclEntry> entries)`.
- Regras: admin → tudo; união de entradas diretas e de grupo; herança só se o nó não tem entradas próprias; implicações de nível; teto do VIEWER; limite de 10 níveis.

### `security.access.CurrentUserProvider`

- **Purpose**: Carrega por requisição `CurrentUser { id, role, groupIds, flags, attributes }` com uma consulta (cache `@RequestScope`).

### `security.access.RequiresFlag` + `FlagAspect`

- **Purpose**: `@RequiresFlag(Flag.EXPORT)` em métodos de controller → 403 `FLAG_REQUIRED`.

### `security.access.DataBoardPermissionEvaluator`

- **Purpose**: Ponte do `hasPermission(...)` para o `AccessControl`.

### `security.user.UserService` / `UserController`

- `create`, `update`, `changeStatus`, `softDelete`, `replaceAttributes`, `list(filter, pageable)`, `emailCheck()`.
- `LastAdminGuard.assertNotLastAdmin(userId, change)`: lock `SELECT ... FOR UPDATE` sobre admins ativos.

### `security.group.GroupService` / `GroupController`

- `create`, `update`, `delete` (cascata em `acl_entry`, `principal_flag` e `user_group_member`), `replaceMembers`.

### `security.access.AclAdminService` / `AclController`

- `GET/PUT /api/v1/acl/{type}/{id}`, `GET /users/{id}/effective-access`.

### Frontend

- `features/admin/users/UsersPage` (lista + filtros), `UserFormDrawer` (dados, papel, grupos, flags, atributos chave/valor).
- `features/admin/groups/GroupsPage`, `GroupMembersModal`.
- `shared/acl/AclEditor` (componente reutilizável: tabela de principals + nível; usado por telas, visões, UDTs, conexões).
- `features/admin/users/EffectiveAccessPage`.
- `shared/auth/usePermission(flag)` e `Can` (oculta UI por flag ou nível; o backend continua revalidando).

---

## Data Models

```sql
-- V4__access_control.sql
ALTER TABLE app.user_account ADD COLUMN deleted_at timestamptz;
DROP INDEX app.ux_user_email;
CREATE UNIQUE INDEX ux_user_email ON app.user_account (lower(email)) WHERE deleted_at IS NULL;

CREATE TABLE app.user_group (
  id uuid PRIMARY KEY, name varchar(120) NOT NULL UNIQUE, description text,
  priority int NOT NULL DEFAULT 100, created_at timestamptz NOT NULL DEFAULT now());

CREATE TABLE app.user_group_member (
  group_id uuid REFERENCES app.user_group(id) ON DELETE CASCADE,
  user_id  uuid REFERENCES app.user_account(id),
  PRIMARY KEY (group_id, user_id));

CREATE TABLE app.principal_flag (
  principal_type varchar(5) NOT NULL CHECK (principal_type IN ('USER','GROUP')),
  principal_id uuid NOT NULL, flag varchar(30) NOT NULL,
  PRIMARY KEY (principal_type, principal_id, flag));

CREATE TABLE app.user_attribute (
  user_id uuid REFERENCES app.user_account(id), key varchar(80) NOT NULL,
  value varchar(500) NOT NULL, PRIMARY KEY (user_id, key, value));

CREATE TABLE app.resource_node (
  resource_type varchar(20) NOT NULL, resource_id uuid NOT NULL,
  parent_folder_id uuid, inherit boolean NOT NULL DEFAULT true, owner_id uuid,
  PRIMARY KEY (resource_type, resource_id));

CREATE TABLE app.acl_entry (
  resource_type varchar(20) NOT NULL, resource_id uuid NOT NULL,
  principal_type varchar(5) NOT NULL, principal_id uuid NOT NULL,
  level varchar(15) NOT NULL,
  PRIMARY KEY (resource_type, resource_id, principal_type, principal_id, level));
CREATE INDEX ix_acl_principal ON app.acl_entry (principal_type, principal_id);
```

```java
enum AccessLevel { VIEW, EDIT, MANAGE, DATA_READ, DATA_INSERT, DATA_UPDATE, DATA_DELETE }
enum Flag { SQL_EDITOR, CUSTOM_QUERY, UPLOAD, EXPORT, MANAGE_UDT_SCHEMA }
record ResourceRef(ResourceType type, UUID id) {}
record CurrentUser(UUID id, Role role, Set<UUID> groupIds, Set<Flag> flags,
                   Map<String, List<String>> attributes) {}
```

---

## Error Handling Strategy

| Error Scenario | Handling | User Impact |
| --- | --- | --- |
| Sem nível no recurso | 403 `ACCESS_DENIED` | Tela "Sem acesso"; item some do menu |
| Sem flag | 403 `FLAG_REQUIRED` + `flag` | Botão oculto; acesso direto mostra aviso |
| Último admin | 409 `LAST_ADMIN` | "É necessário manter ao menos um administrador ativo" |
| E-mail/nome duplicado | 409 `EMAIL_IN_USE` / `GROUP_NAME_IN_USE` | Erro no campo |

---

## Risks & Concerns

| Concern | Location (file:line) | Impact | Mitigation |
| --- | --- | --- | --- |
| Resolução por requisição em listagens grandes (N recursos) | `AccessControl.filterVisible` | N+1 consultas | Uma consulta com CTE recursiva sobre `resource_node` + `acl_entry` para o lote |
| Módulo esquecer de chamar `registerResource` | Módulos de conteúdo | Recurso órfão, invisível para não-admins | Teste de contrato por módulo ("criador recebe MANAGE") nas features que criam recursos |
| Corrida na regra do último admin | `LastAdminGuard` | Zero admins | `SELECT ... FOR UPDATE` dos admins ativos na mesma transação |

---

## Tech Decisions (only non-obvious ones)

| Decision | Choice | Rationale |
| --- | --- | --- |
| Registro de recursos | Tabela genérica `resource_node` em `security` | Herança e dono uniformes para todos os tipos, sem FK entre módulos |
| Atributos | Linhas (chave, valor) multivaloradas | Suporte a `IN` na RLS |
| Avaliação | Por requisição, sem cache entre requisições | Efeito imediato exigido pelo CA-04.1 |
