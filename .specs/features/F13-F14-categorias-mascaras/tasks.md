# F13/F14 — Categorias e Máscaras Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F13-F14-categorias-mascaras/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio | unit | Todos os ramos; 1:1 com ACs | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend controller/persistência | integration | Feliz + edge + erro | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend formatação | unit | Tabela com todas as 18 predefinidas + edge cases | `frontend/src/shared/format/*.test.ts` | `make test-frontend` |
| Frontend componentes | unit | Interações principais | `frontend/src/**/*.test.tsx` | `make test-frontend` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-backend` ou `make test-frontend` |
| Full | Integração | `make test-backend it-backend` |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Backend

```
T1 → T2 → T5 → T6
T1 → T3 → T4
```

### Phase 2: Frontend

```
T7 → T8
T9 → T10
```

---

## Task Breakdown

### Phase 1: Backend

#### T1: Migração do catálogo e dependências

**What**: `V7__catalog.sql` com `category`, `resource_category`, `mask` e `dependency`.
**Where**: `backend/src/main/resources/db/migration/V7__catalog.sql`
**Depends on**: None
**Reuses**: —
**Requirement**: CAT-01, MASK-01, DEP-01
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `CatalogSchemaIT`: nome irmão duplicado viola o índice; cascata da categoria apaga `resource_category`

**Tests**: integration
**Gate**: full
**Commit**: `feat(catalog): add categories, masks and dependency schema`

---

#### T2: `DependencyRegistry`

**What**: `replaceOutgoing`, `removeAll`, `dependentsOf` com `NameResolver` SPI.
**Where**: `backend/src/main/java/io/databoard/platform/dependency/DependencyRegistry.java`
**Depends on**: T1
**Reuses**: —
**Requirement**: DEP-01

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `DependencyRegistryIT`: substituição remove arestas antigas; `dependentsOf` traz nome via resolver fictício; rollback do chamador desfaz as arestas

**Tests**: integration
**Gate**: full
**Commit**: `feat(platform): add generic dependency registry`

---

#### T3: `CategoryService`

**What**: Árvore, criação, renomeação/movimento (profundidade da subárvore, ciclo, nome irmão) e exclusão em cascata.
**Where**: `backend/src/main/java/io/databoard/catalog/category/CategoryService.java`
**Depends on**: T1
**Reuses**: —
**Requirement**: CAT-01, CAT-02

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: 4º nível → `CATEGORY_DEPTH_EXCEEDED`; mover subárvore de 2 níveis para nível 2 → excede; mover para descendente → `CATEGORY_CYCLE`; nome irmão "vendas" vs "Vendas" → `CATEGORY_NAME_IN_USE`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(catalog): implement category tree rules`

---

#### T4: `CategoryController` + `CategoryAssignments`

**What**: `GET|POST /categories`, `PUT|DELETE /categories/{id}` e API `CategoryAssignments` (assign, resourcesIn com descendentes, categoryOf).
**Where**: `backend/src/main/java/io/databoard/catalog/category/CategoryController.java`
**Depends on**: T3
**Reuses**: `ResourceRef` (F04)
**Requirement**: CAT-01, CAT-02, CAT-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `CategoryControllerIT`: CRUD feliz; VIEWER → 403; excluir raiz deixa recursos sem categoria; `resourcesIn` da raiz inclui recursos da neta; `assign` substitui a categoria anterior

**Tests**: integration
**Gate**: full
**Commit**: `feat(catalog): expose categories and assignments`

---

#### T5: `MaskService` com predefinidas

**What**: Catálogo `masks-builtin.json` (18 máscaras), CRUD das customizadas, validação ICU e bloqueio de predefinidas e em uso.
**Where**: `backend/src/main/java/io/databoard/catalog/mask/MaskService.java`
**Depends on**: T2
**Reuses**: `DependencyRegistry` (T2)
**Requirement**: MASK-01, MASK-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: lista contém os 18 slugs; alterar `brl` → `MASK_BUILT_IN`; padrão ICU `yyyy-QQQQ-zz9` inválido → erro em `datePattern`; customizada com dependente → `MASK_IN_USE`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(catalog): implement mask catalog`

---

#### T6: `MaskController`

**What**: `GET|POST /masks`, `PUT|DELETE /masks/{id}`.
**Where**: `backend/src/main/java/io/databoard/catalog/mask/MaskController.java`
**Depends on**: T5
**Reuses**: —
**Requirement**: MASK-01, MASK-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `MaskControllerIT`: criar NUMBER, DATE e TEXT; excluir em uso → 409 com `dependents`; VIEWER em POST → 403

**Tests**: integration
**Gate**: full
**Commit**: `feat(catalog): expose mask endpoints`

---

### Phase 2: Frontend

#### T7: `formatValue` e `maskToEChartsFormatter`

**What**: Biblioteca de formatação pt-BR para NUMBER/DATE/TEXT, escala, percentual, cor de negativos e conversão para formatter do ECharts.
**Where**: `frontend/src/shared/format/formatValue.ts`
**Depends on**: None
**Reuses**: `Intl.NumberFormat`, `Intl.DateTimeFormat`
**Requirement**: MASK-02
**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] Teste de tabela: os 5 exemplos do AC 5 + as 18 predefinidas; `null` → `""`; texto em máscara numérica → original; CPF incompleto → cru; negativo com `negativeColor` → cor

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add value formatting library`

---

#### T8: `MasksPage` com pré-visualização

**What**: Lista (predefinidas somente leitura) + modal de máscara customizada com pré-visualização ao vivo usando `formatValue`.
**Where**: `frontend/src/features/catalog/MasksPage.tsx`
**Depends on**: T7
**Reuses**: `formatValue` (T7)
**Requirement**: MASK-01, MASK-03

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: predefinida sem botões de editar/excluir; prévia atualiza ao mudar casas decimais; 409 `MASK_IN_USE` lista dependentes

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add masks catalog page`

---

#### T9: `CategoryTree`

**What**: Árvore com criar, renomear, mover (arrastar ou select de pai) e excluir com confirmação em modal.
**Where**: `frontend/src/features/catalog/CategoryTree.tsx`
**Depends on**: None
**Reuses**: Mantine `Tree`
**Requirement**: CAT-01, CAT-02

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: cria subcategoria; erro de profundidade exibido; exclusão chama `DELETE`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add category tree manager`

---

#### T10: `CategorySelect` reutilizável

**What**: Select hierárquico (indentação por nível, opção "Sem categoria") para formulários de recurso e filtros de listagem.
**Where**: `frontend/src/features/catalog/CategorySelect.tsx`
**Depends on**: T9
**Reuses**: —
**Requirement**: CAT-03

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: renderiza a hierarquia indentada; "Sem categoria" emite `null`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add hierarchical category select`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1 | None | início | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T3 | T1 | T1 → T3 | ✅ |
| T4 | T3 | T3 → T4 | ✅ |
| T5 | T2 | T2 → T5 | ✅ |
| T6 | T5 | T5 → T6 | ✅ |
| T7, T9 | None | inícios da fase 2 | ✅ |
| T8 | T7 | T7 → T8 | ✅ |
| T10 | T9 | T9 → T10 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T3, T5 | Backend domínio | unit | unit | ✅ |
| T1, T2, T4, T6 | Backend persistência/controller | integration | integration | ✅ |
| T7–T10 | Frontend | unit | unit | ✅ |
