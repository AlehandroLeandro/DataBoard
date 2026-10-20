# F02 — Configurações Gerais Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F02-configuracoes/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend seções/serviço | unit | Todos os ramos; faixas; máscara de segredos | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend controller/persistência | integration | Rotas: feliz + edge + erro | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend | unit | Abas, validação, teste de SMTP, tema | `frontend/src/**/*.test.tsx` | `make test-frontend` |

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
T1 → T2 → T3 → T4
T2 → T5
T6
```

### Phase 2: Frontend

```
T7 → T8
T9
```

---

## Task Breakdown

### Phase 1: Backend

#### T1: Records de seção com padrões e validação

**What**: `GeneralSettings`, `SecuritySettings`, `LimitsSettings`, `SmtpSettings`, `AppearanceSettings` com `defaults()` e faixas, e o `SectionRegistry`.
**Where**: `backend/src/main/java/io/databoard/platform/settings/sections/SectionRegistry.java`
**Depends on**: None
**Reuses**: —
**Requirement**: CFG-01, CFG-02, CFG-05
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: padrões da spec; cada faixa com valor no limite (aceita) e fora (rejeita); `SmtpSettings.password` marcado como segredo

**Tests**: unit
**Gate**: quick
**Commit**: `feat(platform): define settings sections with defaults`

---

#### T2: `SettingsService`

**What**: `getAll` com padrões e máscara; `update` estrito com preservação de segredo, cifragem e evento `SettingsChanged` mascarado.
**Where**: `backend/src/main/java/io/databoard/platform/settings/SettingsService.java`
**Depends on**: T1
**Reuses**: `SettingsStore`, `CryptoService` (F01)
**Requirement**: CFG-01, CFG-03, CFG-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: senha `"********"` mantém a anterior; senha nova é cifrada; evento traz `before/after` com senha mascarada; campo desconhecido → erro

**Tests**: unit
**Gate**: quick
**Commit**: `feat(platform): implement settings service with secret masking`

---

#### T3: `SettingsController`

**What**: `GET /settings`, `PUT /settings/{section}` (admin), 404 para seção inexistente.
**Where**: `backend/src/main/java/io/databoard/platform/settings/SettingsController.java`
**Depends on**: T2
**Reuses**: F04 `@PreAuthorize`
**Requirement**: CFG-01, CFG-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SettingsControllerIT`: GET com padrões; PUT válido 200; fora de faixa 400; BUILDER → 403; seção `foo` → 404

**Tests**: integration
**Gate**: full
**Commit**: `feat(platform): expose settings endpoints`

---

#### T4: Segurança e limites consumindo as seções

**What**: `SecurityPolicyProvider` (F03) passa a ler `SecuritySettings`; criar `LimitsProvider`.
**Where**: `backend/src/main/java/io/databoard/platform/settings/LimitsProvider.java`
**Depends on**: T3
**Reuses**: `SecurityPolicyProvider` (F03)
**Requirement**: CFG-02

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SecuritySettingsEffectIT`: `PUT security` com `lockoutAttempts=3` → 3ª falha de login retorna 423 sem reinício
- [ ] `LimitsProvider` devolve o valor gravado

**Tests**: integration
**Gate**: full
**Commit**: `feat(platform): apply security and limits settings at runtime`

---

#### T5: Teste de SMTP

**What**: `POST /settings/smtp/test` com timeout de 15 s e `SMTP_TEST_FAILED`.
**Where**: `backend/src/main/java/io/databoard/platform/settings/SmtpTestController.java`
**Depends on**: T2
**Reuses**: `MailService` (F03)
**Requirement**: CFG-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SmtpTestIT` (GreenMail): entrega → `{delivered:true}`; servidor parado → 422 `SMTP_TEST_FAILED`; `to` inválido → 400

**Tests**: integration
**Gate**: full
**Commit**: `feat(platform): add smtp test endpoint`

---

#### T6: Preferências do usuário

**What**: `V5__preferences.sql`, `PUT /api/v1/me/preferences` e `preferences` em `/auth/me`.
**Where**: `backend/src/main/java/io/databoard/security/user/PreferencesController.java`
**Depends on**: None
**Reuses**: —
**Requirement**: CFG-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `PreferencesIT`: grava `theme=dark` e `/auth/me` devolve; tema inválido → 400

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): store user preferences`

---

### Phase 2: Frontend

#### T7: `SettingsPage` com abas

**What**: Abas Geral, Segurança, Limites e E-mail, com erros por campo e senha mascarada.
**Where**: `frontend/src/features/admin/settings/SettingsPage.tsx`
**Depends on**: None
**Reuses**: `http.ts`, Mantine `Tabs`/`useForm`
**Requirement**: CFG-01, CFG-02, CFG-04
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: cada aba envia `PUT` só da sua seção; erro 400 aparece no campo; senha exibida como `********`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add admin settings page`

---

#### T8: Botão "Enviar e-mail de teste"

**What**: Modal pedindo o destinatário e exibindo sucesso ou `detail` do erro.
**Where**: `frontend/src/features/admin/settings/SmtpTestButton.tsx`
**Depends on**: T7
**Reuses**: —
**Requirement**: CFG-04

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: 200 → "E-mail enviado"; 422 → mostra `detail`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add smtp test action`

---

#### T9: `ThemeController` e menu de tema

**What**: Aplica preferência ou padrão (`auto` segue o navegador) e grava a escolha do usuário.
**Where**: `frontend/src/app/theme/ThemeController.tsx`
**Depends on**: None
**Reuses**: `useMantineColorScheme`, `AuthProvider` (F03)
**Requirement**: CFG-05

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: sem preferência usa `defaultTheme`; escolher "Escuro" chama `PUT /me/preferences` e muda o esquema

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): apply theme preference`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T6 | None | inícios da fase 1 | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T3 | T2 | T2 → T3 | ✅ |
| T4 | T3 | T3 → T4 | ✅ |
| T5 | T2 | T2 → T5 | ✅ |
| T7, T9 | None | inícios da fase 2 | ✅ |
| T8 | T7 | T7 → T8 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1, T2 | Backend seções/serviço | unit | unit | ✅ |
| T3–T6 | Backend controller/persistência | integration | integration | ✅ |
| T7–T9 | Frontend | unit | unit | ✅ |
