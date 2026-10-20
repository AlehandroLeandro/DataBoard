# F04 — Usuários, Grupos e Permissões Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F04-usuarios-permissoes/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio (resolver, serviços) | unit | Todos os ramos; 1:1 com ACs; edge cases | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend controller/persistência/aspecto | integration | Rotas: feliz + edge + erro | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Estados e interações | `frontend/src/**/*.test.tsx` | `make test-frontend` |
| Ponta a ponta | e2e | CA-04.1 | `frontend/e2e/*.spec.ts` | `make e2e` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-backend` ou `make test-frontend` |
| Full | Integração/e2e | `make test-backend it-backend` (+ `make e2e`) |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Modelo e resolução

```
T1 → T2
T3
```

### Phase 2: Serviço central de autorização

```
T4 → T5
T6
```

### Phase 3: Usuários e grupos

```
T7 → T8 → T9
T10 → T11
```

### Phase 4: Administração de ACL

```
T12 → T13
```

### Phase 5: Frontend e E2E

```
T14 → T18
T15 → T16
T17
T19
T16 → T20
```

---

## Task Breakdown

### Phase 1: Modelo e resolução

#### T1: Migração de controle de acesso

**What**: `V4__access_control.sql` (soft delete, grupos, membros, flags, atributos, `resource_node`, `acl_entry`) + entidades/repositórios.
**Where**: `backend/src/main/resources/db/migration/V4__access_control.sql`
**Depends on**: None
**Reuses**: `UserAccount` (F01)
**Requirement**: ACL-01, ACL-06, ACL-07
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AccessSchemaIT`: e-mail de excluído pode ser reutilizado; e-mail duplicado entre ativos falha; cascata de grupo remove membros

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): add access control schema`

---

#### T2: `CurrentUserProvider`

**What**: Carrega `CurrentUser` (papel, grupos, flags efetivas, atributos multivalorados) com uma consulta, em escopo de requisição.
**Where**: `backend/src/main/java/io/databoard/security/access/CurrentUserProvider.java`
**Depends on**: T1
**Reuses**: —
**Requirement**: ACL-07, ACL-10

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `CurrentUserProviderIT`: flags = usuário ∪ grupos; admin → todas; atributo `filiais` com 2 valores vira lista

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): load current user with groups and flags`

---

#### T3: `AclResolver`

**What**: Lógica pura de nível efetivo: admin, união, herança só sem entradas próprias, implicações, teto do VIEWER, limite de 10 níveis.
**Where**: `backend/src/main/java/io/databoard/security/access/AclResolver.java`
**Depends on**: None
**Reuses**: —
**Requirement**: ACL-07, ACL-08

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes 1:1 com ACs 2–5 da story ACL: união grupo+direto; admin tudo; VIEWER com `EDIT` concedido → nega `EDIT` e permite `VIEW` (CA-04.2); herança com e sem entradas próprias; `MANAGE` implica `DATA_DELETE`; `EDIT` implica `DATA_READ`; cadeia de 11 níveis → nega

**Tests**: unit
**Gate**: quick
**Commit**: `feat(security): resolve effective access levels`

---

### Phase 2: Serviço central de autorização

#### T4: `AccessControl` com registro de recursos

**What**: Implementação de `can/require/filterVisible/registerResource/moveResource/unregisterResource`, com CTE recursiva para lotes.
**Where**: `backend/src/main/java/io/databoard/security/access/DefaultAccessControl.java`
**Depends on**: None
**Reuses**: `AclResolver` (T3), `CurrentUserProvider` (T2)
**Requirement**: ACL-07, ACL-08, ACL-09

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AccessControlIT`: `registerResource` concede `MANAGE` ao dono; CA-04.1 (remover do grupo → `can` falso na chamada seguinte); `filterVisible` com 500 ids executa 1 consulta (contador do datasource-proxy)

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): implement central access control service`

---

#### T5: `PermissionEvaluator` e `@RequiresFlag`

**What**: Integração com `@PreAuthorize("hasPermission(...)")` e aspecto `@RequiresFlag` → 403 `FLAG_REQUIRED`.
**Where**: `backend/src/main/java/io/databoard/security/access/FlagAspect.java`
**Depends on**: T4
**Reuses**: `AccessControl` (T4)
**Requirement**: ACL-07, ACL-10

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AuthorizationAnnotationsIT` com controller de teste: sem flag → 403 `FLAG_REQUIRED` com `flag=EXPORT`; com flag via grupo → 200; `hasPermission` negado → 403 `ACCESS_DENIED`

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): add permission evaluator and flag guard`

---

#### T6: `permissions` em `/auth/me`

**What**: Preencher `permissions` com as flags efetivas do `CurrentUser`.
**Where**: `backend/src/main/java/io/databoard/security/auth/AuthController.java` (modify)
**Depends on**: None
**Reuses**: `CurrentUserProvider` (T2)
**Requirement**: ACL-10

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AuthControllerIT` estendido: usuário com `UPLOAD` direto e `EXPORT` por grupo → `permissions` contém ambos; admin → as 5 flags

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): expose effective flags on me endpoint`

---

### Phase 3: Usuários e grupos

#### T7: `LastAdminGuard`

**What**: Garante ≥ 1 admin ativo em inativar, bloquear, excluir ou trocar papel, com lock pessimista.
**Where**: `backend/src/main/java/io/databoard/security/user/LastAdminGuard.java`
**Depends on**: None
**Reuses**: —
**Requirement**: ACL-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `LastAdminGuardIT`: último admin inativado → `LAST_ADMIN`; com 2 admins, duas inativações concorrentes → uma falha

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): protect the last active admin`

---

#### T8: `UserService`

**What**: Criar (senha temporária + `must_change_password`), atualizar, mudar status (publica `UserAccessRevoked`), soft delete, substituir atributos, listar com filtros.
**Where**: `backend/src/main/java/io/databoard/security/user/UserService.java`
**Depends on**: T7
**Reuses**: `PasswordPolicyValidator` (F03), `LastAdminGuard` (T7)
**Requirement**: ACL-01, ACL-02, ACL-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: criação marca troca obrigatória; e-mail em uso → `EMAIL_IN_USE`; status `BLOCKED` publica evento; delete remove dos grupos e publica evento; atributos substituem os anteriores

**Tests**: unit
**Gate**: quick
**Commit**: `feat(security): implement user management service`

---

#### T9: `UserController`

**What**: `GET|POST /users`, `GET|PUT|DELETE /users/{id}`, `PUT /users/{id}/attributes`, somente admin.
**Where**: `backend/src/main/java/io/databoard/security/user/UserController.java`
**Depends on**: T8
**Reuses**: `PageResponse` (F00)
**Requirement**: ACL-01, ACL-02, ACL-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `UserControllerIT`: CRUD feliz; filtros `status/groupId/role`; `lastLoginAt` presente; BUILDER em POST → 403; inativar → sessões do usuário 401

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): expose user management endpoints`

---

#### T10: `GroupService`

**What**: CRUD de grupos, substituição de membros e exclusão em cascata de ACL e flags.
**Where**: `backend/src/main/java/io/databoard/security/group/GroupService.java`
**Depends on**: None
**Reuses**: —
**Requirement**: ACL-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: nome duplicado → `GROUP_NAME_IN_USE`; delete remove entradas de `acl_entry` e `principal_flag` do grupo

**Tests**: unit
**Gate**: quick
**Commit**: `feat(security): implement group management service`

---

#### T11: `GroupController`

**What**: `GET|POST /groups`, `GET|PUT|DELETE /groups/{id}`, `PUT /groups/{id}/members`.
**Where**: `backend/src/main/java/io/databoard/security/group/GroupController.java`
**Depends on**: T10
**Reuses**: `PageResponse`
**Requirement**: ACL-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `GroupControllerIT`: criar, adicionar membros, excluir; ACL do grupo some após exclusão

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): expose group endpoints`

---

### Phase 4: Administração de ACL

#### T12: `AclController`

**What**: `GET|PUT /acl/{resourceType}/{resourceId}` (exige `MANAGE`), validando principals e níveis `DATA_*` só em `UDT_TABLE`.
**Where**: `backend/src/main/java/io/databoard/security/access/AclController.java`
**Depends on**: None
**Reuses**: `AccessControl` (T4)
**Requirement**: ACL-09

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AclControllerIT`: dono com `MANAGE` substitui entradas; sem `MANAGE` → 403; principal inexistente → 400 em `entries[0].principalId`; `DATA_READ` em `SCREEN` → 400

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): expose resource acl endpoints`

---

#### T13: Acesso efetivo e verificação de e-mails

**What**: `GET /users/{id}/effective-access` (com origem) e `GET /users/email-check`.
**Where**: `backend/src/main/java/io/databoard/security/access/EffectiveAccessController.java`
**Depends on**: T12
**Reuses**: `AclResolver` (T3)
**Requirement**: ACL-11

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `EffectiveAccessIT`: origens `DIRECT`, `GROUP:Comercial`, `INHERITED:<pasta>` e `ROLE_ADMIN` corretas; e-mails `A@x.com`/`a@x.com` aparecem como duplicados

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): add effective access and email check reports`

---

### Phase 5: Frontend e E2E

#### T14: `usePermission` e `<Can>`

**What**: Hook e componente que ocultam UI por flag ou papel a partir de `/auth/me`.
**Where**: `frontend/src/shared/auth/usePermission.ts`
**Depends on**: None
**Reuses**: `AuthProvider` (F03)
**Requirement**: ACL-10
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: sem flag → filho oculto; admin → visível

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add permission hook and can component`

---

#### T15: `UsersPage`

**What**: Lista paginada com filtros (status, grupo, papel) e coluna "Último acesso".
**Where**: `frontend/src/features/admin/users/UsersPage.tsx`
**Depends on**: None
**Reuses**: TanStack Table, `http.ts`
**Requirement**: ACL-05

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: filtros viram query string; paginação; estado vazio

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add users admin list`

---

#### T16: `UserFormDrawer`

**What**: Formulário de criação/edição: dados, papel, grupos, flags, atributos chave/valor (valor múltiplo) e status, tratando `EMAIL_IN_USE` e `LAST_ADMIN`.
**Where**: `frontend/src/features/admin/users/UserFormDrawer.tsx`
**Depends on**: T15
**Reuses**: Mantine `Drawer`, `TagsInput`
**Requirement**: ACL-01, ACL-02, ACL-04

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: cria com payload esperado; atributo com 2 valores vira array; `LAST_ADMIN` exibe a mensagem da spec

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add user form drawer`

---

#### T17: `GroupsPage`

**What**: CRUD de grupos com modal de membros (busca de usuários) e flags do grupo.
**Where**: `frontend/src/features/admin/groups/GroupsPage.tsx`
**Depends on**: None
**Reuses**: TanStack Table
**Requirement**: ACL-06

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: cria grupo; salva membros via `PUT /members`; nome duplicado mostra erro

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add groups admin page`

---

#### T18: `AclEditor` reutilizável

**What**: Componente com lista de principals (usuário/grupo) e nível, níveis de dados só quando `resourceType=UDT_TABLE`, indicador "herdado da pasta".
**Where**: `frontend/src/shared/acl/AclEditor.tsx`
**Depends on**: T14
**Reuses**: —
**Requirement**: ACL-09

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: adiciona grupo com `VIEW` e envia `PUT`; níveis `DATA_*` ausentes para `SCREEN`; badge "Herdado" quando a API indica herança

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add reusable acl editor`

---

#### T19: `EffectiveAccessPage`

**What**: Página por usuário com recursos, nível e origem.
**Where**: `frontend/src/features/admin/users/EffectiveAccessPage.tsx`
**Depends on**: None
**Reuses**: TanStack Table
**Requirement**: ACL-11

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Teste: renderiza origens retornadas pela API

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add effective access page`

---

#### T20: E2E de grupos e acesso

**What**: Admin cria usuário Pedro e grupo Comercial, concede `VIEW` a um recurso de teste (seed), Pedro vê; admin remove Pedro do grupo; Pedro recebe "Sem acesso".
**Where**: `frontend/e2e/access.spec.ts`
**Depends on**: T16
**Reuses**: Fixture de login (F03)
**Requirement**: ACL-07

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde (CA-04.1)

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): cover group-based access revocation`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T3 | None | inícios da fase 1 | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T4, T6 | None (deps de fases anteriores no corpo via Reuses) | inícios da fase 2 | ✅ |
| T5 | T4 | T4 → T5 | ✅ |
| T7, T10 | None | inícios da fase 3 | ✅ |
| T8 | T7 | T7 → T8 | ✅ |
| T9 | T8 | T8 → T9 | ✅ |
| T11 | T10 | T10 → T11 | ✅ |
| T12 | None | início da fase 4 | ✅ |
| T13 | T12 | T12 → T13 | ✅ |
| T14, T15, T17, T19 | None | inícios da fase 5 | ✅ |
| T16 | T15 | T15 → T16 | ✅ |
| T18 | T14 | T14 → T18 | ✅ |
| T20 | T16 | T16 → T20 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T3, T8, T10 | Backend domínio | unit | unit | ✅ |
| T1, T2, T4–T7, T9, T11–T13 | Backend persistência/controller/aspecto | integration | integration | ✅ |
| T14–T19 | Frontend | unit | unit | ✅ |
| T20 | Ponta a ponta | e2e | e2e | ✅ |
