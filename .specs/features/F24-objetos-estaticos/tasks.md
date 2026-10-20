# F24 — Objetos Estáticos Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F24-objetos-estaticos/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend handlers | unit | Validações e `staticData` | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend integração | integration | Vínculo de arquivo, tokens no endpoint | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Renderers (inclui payloads XSS) e editores | `frontend/src/**/*.test.tsx` | `make test-frontend` |

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
T1 → T2 → T3
```

### Phase 2: Frontend

```
T4
T5
T6
```

---

## Task Breakdown

### Phase 1: Backend

#### T1: Gancho `staticData` e handlers estáticos

**What**: Método opcional `staticData` no `ObjectTypeHandler`, suporte no `ObjectDataService`, e os handlers `IMAGE`, `TEXT` e `SEPARATOR` com validações.
**Where**: `backend/src/main/java/io/databoard/dashboard/object/statics/TextObjectHandler.java`
**Depends on**: None
**Reuses**: `ObjectTypeHandler`, `ObjectDataService` (F19), `TokenService` (F15)
**Requirement**: STO-01, STO-02, STO-03, STO-04
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: URL `ftp://` → erro; link `javascript:` → erro; rota interna aceita; markdown de 20.001 caracteres → erro; espessura 5 → erro; `staticData` do texto chama a interpolação

**Tests**: unit
**Gate**: quick
**Commit**: `feat(dashboard): add static object handlers`

---

#### T2: Vínculo de imagens à tela

**What**: Gancho `onSaved` no `ScreenService` que vincula `fileId` de objetos `IMAGE` ao recurso `SCREEN` e registra `SCREEN_OBJECT → FILE`.
**Where**: `backend/src/main/java/io/databoard/dashboard/object/statics/ImageObjectHandler.java`
**Depends on**: T1
**Reuses**: `FileStore.link` (F11)
**Requirement**: STO-01

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ImageObjectIT`: salvar tela vincula o arquivo; usuário com `VIEW` na tela lê o arquivo (200); usuário sem acesso → 404; `fileId` de PDF → 400

**Tests**: integration
**Gate**: full
**Commit**: `feat(dashboard): link uploaded images to screens`

---

#### T3: Texto com tokens no endpoint de dados

**What**: IT do endpoint de dados para `TEXT` com tokens, sem consulta ao banco.
**Where**: `backend/src/test/java/io/databoard/dashboard/object/statics/TextObjectIT.java`
**Depends on**: T2
**Reuses**: `ScreenObjectDataController` (F19)
**Requirement**: STO-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `TextObjectIT` (relógio 15/05/2026): resposta `"Atualizado em 15/05/2026"`; contador de consultas = 0

**Tests**: integration
**Gate**: full
**Commit**: `test(dashboard): cover text object token interpolation`

---

### Phase 2: Frontend

#### T4: Imagem (renderer e editor)

**What**: `ImageRenderer` (fit, link, placeholder) e `ImageEditor` (upload, URL, fit, link, alt), registrados no `objectRegistry`.
**Where**: `frontend/src/features/screens/objects/statics/ImageObject.tsx`
**Depends on**: None
**Reuses**: `POST /files` (F11), Mantine `Dropzone`
**Requirement**: STO-01
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: upload grava `fileId`; `fit=cover` aplica `object-fit: cover`; erro de carga mostra placeholder com `alt`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add image object`

---

#### T5: Texto (renderer e editor)

**What**: `TextRenderer` com `react-markdown` + GFM sem HTML e `TextEditor` com CodeMirror markdown, autocomplete de tokens e prévia.
**Where**: `frontend/src/features/screens/objects/statics/TextObject.tsx`
**Depends on**: None
**Reuses**: `tokenCompletion` (F15)
**Requirement**: STO-02, STO-03

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `security-and-hardening`

**Done when**:
- [ ] Testes: tabela GFM renderizada; `<script>alert(1)</script>`, `<img onerror=...>` e `[x](javascript:alert(1))` não executam e não geram `href` perigoso

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add markdown text object`

---

#### T6: Separador (renderer e editor)

**What**: `SeparatorRenderer` (linha ou espaço) e `SeparatorEditor`.
**Where**: `frontend/src/features/screens/objects/statics/SeparatorObject.tsx`
**Depends on**: None
**Reuses**: Tema Mantine
**Requirement**: STO-04

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: `line` com espessura 2 desenha borda de 2 px; `space` não desenha nada

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add separator object`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1 | None | início | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T3 | T2 | T2 → T3 | ✅ |
| T4, T5, T6 | None | nós isolados da fase 2 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1 | Backend handlers | unit | unit | ✅ |
| T2, T3 | Backend integração | integration | integration | ✅ |
| T4–T6 | Frontend | unit | unit | ✅ |
