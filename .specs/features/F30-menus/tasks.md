# F30 — Menus e Tela Inicial Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F30-menus/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio (poda, precedência) | unit | Todos os ramos | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend integração | integration | Menu por usuário, herança, busca, home | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Shell por papel, árvore, editor, busca | `frontend/src/**/*.test.tsx` | `make test-frontend` |
| Ponta a ponta | e2e | Corte do MVP completo | `frontend/e2e/*.spec.ts` | `make e2e` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-backend` ou `make test-frontend` |
| Full | Integração/e2e | `make test-backend it-backend` (+ `make e2e`) |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Backend

```
T1 → T3
T2 → T3 → T4
T5 → T6
T7
```

### Phase 2: Frontend e E2E

```
T8 → T9
T8 → T10
T11
T9 → T12
```

---

## Task Breakdown

### Phase 1: Backend

#### T1: Migração de navegação

**What**: `V18__navigation.sql` (`menu_node`, `unaccent`, `user_group.home_screen_id`) e `homeScreenId` em `GeneralSettings`.
**Where**: `backend/src/main/resources/db/migration/V18__navigation.sql`
**Depends on**: None
**Reuses**: —
**Requirement**: MENU-02, MENU-04
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `NavigationSchemaIT`: inserção de pastas/itens; `unaccent('Ação')` = `Acao`

**Tests**: integration
**Gate**: full
**Commit**: `feat(navigation): add menu schema`

---

#### T2: `MenuTreeBuilder` (poda)

**What**: Montagem da árvore a partir dos nós e dos conjuntos visíveis, com poda de pastas vazias e ordenação.
**Where**: `backend/src/main/java/io/databoard/navigation/menu/MenuTreeBuilder.java`
**Depends on**: None
**Reuses**: —
**Requirement**: MENU-01

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: pasta só com subpasta vazia some; item invisível some; `LINK` visível se a pasta é visível; ordem por `position`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(navigation): build access-filtered menu tree`

---

#### T3: `MenuService` e `MenuController`

**What**: `GET /menu`, `GET /menu/admin-tree`, CRUD de nós, `PUT /menu/order`, herança por pasta-mãe, validações (link, profundidade, ciclo, pasta não vazia) e remoção de itens órfãos por evento.
**Where**: `backend/src/main/java/io/databoard/navigation/menu/MenuService.java`
**Depends on**: T1, T2
**Reuses**: `AccessControl` (F04)
**Requirement**: MENU-01, MENU-02, MENU-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `MenuIT`: três usuários veem menus diferentes; número de consultas de autorização ≤ 2 com 200 itens; primeiro item define a pasta-mãe e a ACL da pasta passa a valer para a tela; link `javascript:` → 400; pasta com filhos → 409; tela excluída some do menu

**Tests**: integration
**Gate**: full
**Commit**: `feat(navigation): manage and serve the menu`

---

#### T4: Ordem e movimentação

**What**: `PUT /menu/order` transacional com detecção de ciclo e profundidade, e aviso de recursos que herdam acesso na resposta do `admin-tree`.
**Where**: `backend/src/main/java/io/databoard/navigation/menu/MenuOrderService.java`
**Depends on**: T3
**Reuses**: —
**Requirement**: MENU-02, MENU-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `MenuOrderIT`: reordenação persistida; mover para descendente → `MENU_CYCLE`; 7º nível → `MENU_TOO_DEEP`; `inheritingCount` por pasta correto

**Tests**: integration
**Gate**: full
**Commit**: `feat(navigation): reorder and move menu nodes`

---

#### T5: `HomeScreenResolver`

**What**: Precedência usuário > grupo (menor `priority`) > global > primeira tela visível > `null`, pulando telas sem `VIEW`.
**Where**: `backend/src/main/java/io/databoard/navigation/home/HomeScreenResolver.java`
**Depends on**: None
**Reuses**: `AccessControl` (F04)
**Requirement**: MENU-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: cada nível escolhido quando o anterior está vazio; tela do usuário sem `VIEW` → vai para o grupo; dois grupos → o de menor `priority`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(navigation): resolve home screen precedence`

---

#### T6: `GET /me/home` e configuração por grupo

**What**: Endpoint da home, `homeScreenId` validado em grupos (F04) e nas configurações gerais (F02).
**Where**: `backend/src/main/java/io/databoard/navigation/home/HomeController.java`
**Depends on**: T5
**Reuses**: `GroupService` (F04), `SettingsService` (F02)
**Requirement**: MENU-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `HomeIT`: home de usuário, de grupo e global; tela inexistente em grupo → 400

**Tests**: integration
**Gate**: full
**Commit**: `feat(navigation): expose home screen endpoint`

---

#### T7: Busca global

**What**: `GET /search?q=` em telas e browsers visíveis, sem acento, máx. 20, "começa com" primeiro.
**Where**: `backend/src/main/java/io/databoard/navigation/search/GlobalSearchService.java`
**Depends on**: None
**Reuses**: `AccessControl.filterVisible` (F04)
**Requirement**: MENU-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `GlobalSearchIT`: "acao" encontra "Ação Comercial"; invisível não aparece; `q` de 1 caractere → vazio; "Vendas" antes de "Metas de vendas" para `q=vend`

**Tests**: integration
**Gate**: full
**Commit**: `feat(navigation): add global search`

---

### Phase 2: Frontend e E2E

#### T8: `AppShell`

**What**: Layout com `Sidebar` (`MenuTree`), `AdminMenu` por papel e `TopBar` (`GlobalSearch` Spotlight, `ExportsPanel`, `ProfileMenu`).
**Where**: `frontend/src/app/layout/AppShell.tsx`
**Depends on**: None
**Reuses**: `ExportsPanel` (F29), `ThemeMenu` (F02), `usePermission` (F04)
**Requirement**: MENU-01, MENU-05, MENU-06, MENU-07
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `frontend-design`

**Done when**:
- [ ] Testes: ADMIN vê todas as entradas de administração; BUILDER só Catálogos; VIEWER nenhuma; busca abre resultados e navega; "Nada encontrado" quando vazio

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add application shell with menu and top bar`

---

#### T9: `HomeRedirect` e `WelcomePage`

**What**: Rota `/` que consulta `/me/home` e navega; página "Bem-vindo"; ação "Usar como minha tela inicial" na tela.
**Where**: `frontend/src/features/navigation/HomeRedirect.tsx`
**Depends on**: T8
**Reuses**: `PUT /me/preferences` (F02)
**Requirement**: MENU-04

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: home definida → navega para a tela; `null` → Bem-vindo; ação grava `homeScreenId`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): redirect to home screen`

---

#### T10: `MenuTree`

**What**: Árvore do menu com pastas recolhíveis (estado lembrado em `localStorage` com try/catch), ícones e item ativo; links externos em nova aba.
**Where**: `frontend/src/app/layout/MenuTree.tsx`
**Depends on**: T8
**Reuses**: Mantine `NavLink`
**Requirement**: MENU-01

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: item ativo destacado pela rota; link externo com `target=_blank` e `rel` seguro; falha do `localStorage` não quebra

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): render navigation menu tree`

---

#### T11: `MenuEditorPage`

**What**: Editor arrastável (pastas/itens), criação de pasta e item (tela, browser, link), ícone, ACL da pasta e aviso de herança ao mover.
**Where**: `frontend/src/features/navigation/MenuEditorPage.tsx`
**Depends on**: None
**Reuses**: `@dnd-kit`, `AclEditor` (F04)
**Requirement**: MENU-02, MENU-03

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: arrastar envia `PUT /menu/order`; mover pasta com `inheritingCount>0` mostra confirmação; pasta sem `EDIT` não arrastável

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add menu editor`

---

#### T12: E2E do corte do MVP

**What**: Fluxo completo: instalação (wizard) → login do admin → criar UDT de vendas e importar planilha → criar visão e regra de RLS → montar tela com KPI, tabela e gráfico → adicionar ao menu numa pasta compartilhada com o grupo Comercial → usuária Ana (Comercial) entra, cai na tela inicial do grupo, filtra por período, vê só as próprias linhas e exporta XLSX.
**Where**: `frontend/e2e/mvp-cut.spec.ts`
**Depends on**: T9
**Reuses**: Fixtures de todas as features
**Requirement**: MENU-01, MENU-04

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): cover the full mvp cut end to end`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T2, T5, T7 | None | inícios da fase 1 | ✅ |
| T3 | T1, T2 | T1 → T3, T2 → T3 | ✅ |
| T4 | T3 | T3 → T4 | ✅ |
| T6 | T5 | T5 → T6 | ✅ |
| T8, T11 | None | inícios da fase 2 | ✅ |
| T9, T10 | T8 | T8 → T9/T10 | ✅ |
| T12 | T9 | T9 → T12 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T2, T5 | Backend domínio | unit | unit | ✅ |
| T1, T3, T4, T6, T7 | Backend integração | integration | integration | ✅ |
| T8–T11 | Frontend | unit | unit | ✅ |
| T12 | Ponta a ponta | e2e | e2e | ✅ |
