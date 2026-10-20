# F15 — Tokens Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F15-tokens/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio (catálogo, serviço, lexer, rewriter) | unit | Todos os tokens; literais/comentários; multivalor; relógio fixo | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend integração com motor/controller | integration | Feliz + erro | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Autocomplete | `frontend/src/**/*.test.ts(x)` | `make test-frontend` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-backend` ou `make test-frontend` |
| Full | Integração | `make test-backend it-backend` |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Núcleo

```
T1 → T2
T3 → T4
T2 → T4
```

### Phase 2: Integração

```
T5 → T6
T7
```

---

## Task Breakdown

### Phase 1: Núcleo

#### T1: `TokenCatalog`

**What**: Definições dos 14 tokens de sistema (tipo, multivalor, descrição) e prefixo `usuario.atributo.<chave>` com validação da chave.
**Where**: `backend/src/main/java/io/databoard/semantic/token/TokenCatalog.java`
**Depends on**: None
**Reuses**: —
**Requirement**: TOK-01
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: os 14 nomes encontrados com o tipo certo; `usuario.atributo.filial-x` → desconhecido

**Tests**: unit
**Gate**: quick
**Commit**: `feat(semantic): define system token catalog`

---

#### T2: `TokenService`

**What**: `resolve`, `interpolateText` e `validateSqlTokens` com fuso da instância e `Clock`.
**Where**: `backend/src/main/java/io/databoard/semantic/token/TokenService.java`
**Depends on**: T1
**Reuses**: `CurrentUserProvider` (F04), `GeneralSettings` (F02)
**Requirement**: TOK-01, TOK-03, TOK-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes com relógio em 15/05/2026: `inicio_mes`=2026-05-01, `fim_mes`=2026-05-31, `mes_anterior.fim`=2026-04-30, `ano_atual`=2026; título do AC interpolado exatamente; token desconhecido em texto mantido; atributo ausente → lista vazia

**Tests**: unit
**Gate**: quick
**Commit**: `feat(semantic): resolve and interpolate system tokens`

---

#### T3: `SqlTokenLexer`

**What**: Localiza `{{...}}` ignorando literais (`'...'`, `E'...'`, `N'...'`, `$tag$...$tag$`), identificadores entre aspas e comentários.
**Where**: `backend/src/main/java/io/databoard/semantic/token/SqlTokenLexer.java`
**Depends on**: None
**Reuses**: —
**Requirement**: TOK-02

**Tools**: MCP: NONE · Skill: `java-springboot`, `security-and-hardening`

**Done when**:
- [ ] Teste de tabela: token em literal, em comentário de linha e de bloco, em `$$` → ignorado; token solto → encontrado com posição

**Tests**: unit
**Gate**: quick
**Commit**: `feat(semantic): scan sql for tokens outside literals`

---

#### T4: `TokenSqlRewriter`

**What**: Troca tokens por binds, expande multivalor dentro de `IN (...)`, `NULL` para lista vazia, `TOKEN_MULTIVALUE_IN_SCALAR`.
**Where**: `backend/src/main/java/io/databoard/semantic/token/TokenSqlRewriter.java`
**Depends on**: T2, T3
**Reuses**: —
**Requirement**: TOK-02

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: `= {{usuario.atributo.vendedor_id}}` → `= ?` com `"7"`; `IN ({{usuario.atributo.filiais}})` com 2 valores → `IN (?, ?)`; vazio → `IN (?)` com `null`; 2 valores em `=` → erro; SQL final não contém nenhum valor literal

**Tests**: unit
**Gate**: quick
**Commit**: `feat(semantic): rewrite tokens to typed bind parameters`

---

### Phase 2: Integração

#### T5: Tokens no motor e no salvamento de visões

**What**: Registrar o rewriter como `QueryParameterResolver` para visões SQL e chamar `validateSqlTokens` no `ViewService`.
**Where**: `backend/src/main/java/io/databoard/semantic/token/TokenQueryIntegration.java`
**Depends on**: None
**Reuses**: `QueryEngine`, `ViewService` (F12)
**Requirement**: TOK-02, TOK-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `TokensInQueryIT`: Ana (`vendedor_id=7`) e Bruno (`vendedor_id=9`) consultam a mesma visão e recebem linhas diferentes; `{{nao_existe}}` no salvamento → 400 `UNKNOWN_TOKEN`; log do SQL executado não contém `7`

**Tests**: integration
**Gate**: full
**Commit**: `feat(semantic): apply tokens in view execution and validation`

---

#### T6: `TokenController`

**What**: `GET /api/v1/tokens` e `POST /api/v1/tokens/preview`.
**Where**: `backend/src/main/java/io/databoard/semantic/token/TokenController.java`
**Depends on**: T5
**Reuses**: `TokenService` (T2)
**Requirement**: TOK-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `TokenControllerIT`: catálogo com 14 itens + modelo de atributo; preview com os atributos do usuário logado

**Tests**: integration
**Gate**: full
**Commit**: `feat(semantic): expose token catalog and preview`

---

#### T7: Autocomplete de tokens no front

**What**: `useTokenCatalog` + extensão CodeMirror que sugere tokens ao digitar `{{`.
**Where**: `frontend/src/shared/tokens/tokenCompletion.ts`
**Depends on**: None
**Reuses**: CodeMirror (F09)
**Requirement**: TOK-05

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: `{{usu` sugere os tokens de usuário; `{{usuario.atributo.` sugere as chaves vindas do preview

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): autocomplete tokens in editors`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T3 | None | inícios da fase 1 | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T4 | T2, T3 | T2 → T4, T3 → T4 | ✅ |
| T5, T7 | None | inícios da fase 2 | ✅ |
| T6 | T5 | T5 → T6 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1–T4 | Backend domínio | unit | unit | ✅ |
| T5, T6 | Backend integração/controller | integration | integration | ✅ |
| T7 | Frontend | unit | unit | ✅ |
