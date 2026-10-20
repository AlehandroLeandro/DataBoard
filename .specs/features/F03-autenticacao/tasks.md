# F03 — Autenticação e Sessão Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F03-autenticacao/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend políticas/serviços | unit | Todos os ramos; 1:1 com ACs; edge cases | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend controller/filtro/persistência | integration | Rotas: feliz + edge + erro | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend páginas/hooks | unit | Estados de sucesso, erro e carregamento | `frontend/src/**/*.test.tsx` | `make test-frontend` |
| Fluxo ponta a ponta | e2e | Login, bloqueio, logout | `frontend/e2e/*.spec.ts` | `make e2e` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-backend` ou `make test-frontend` |
| Full | Integração/e2e | `make test-backend it-backend` (+ `make e2e`) |
| Build | Fim de fase | `make ci` |

---

## Execution Plan

### Phase 1: Modelo e políticas

```
T1
T2 → T3
T2 → T4
```

### Phase 2: Login, sessão e troca de senha

```
T5 → T7
T6 → T7
T7 → T8
T7 → T9
```

### Phase 3: Reset de senha e gestão de sessões

```
T10 → T11
T12 → T13
```

### Phase 4: Frontend e E2E

```
T14 → T15 → T16
T15 → T17 → T18
T14 → T19
T16 → T20
```

---

## Task Breakdown

### Phase 1: Modelo e políticas

#### T1: Migração de autenticação e Spring Session

**What**: `V3__auth.sql` (colunas de bloqueio/senha, `password_history`, `password_reset_token`, `user_session`, tabelas do Spring Session no schema `app`) e mapeamento das novas colunas em `UserAccount`.
**Where**: `backend/src/main/resources/db/migration/V3__auth.sql`
**Depends on**: None
**Reuses**: `UserAccount` (F01)
**Requirement**: AUTH-01, AUTH-04, AUTH-05
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `UserAccountRepositoryIT` estendido: novas colunas persistem; `incrementFailedAttempts(id)` é atômico (duas threads → +2)

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): add auth and session schema`

---

#### T2: `SecurityPolicyProvider` com padrões

**What**: Record `SecurityPolicy` e provider que lê a seção `security` do `SettingsStore` com os padrões da spec.
**Where**: `backend/src/main/java/io/databoard/security/auth/SecurityPolicyProvider.java`
**Depends on**: None
**Reuses**: `SettingsStore` (F01)
**Requirement**: AUTH-05, AUTH-08

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: sem seção → padrões (10, 5, 15, 30, 12, 0, 3); seção parcial → mescla com padrões

**Tests**: unit
**Gate**: quick
**Commit**: `feat(security): add security policy provider`

---

#### T3: `PasswordPolicyValidator`

**What**: Valida tamanho, complexidade (letra + número) e reuso contra o histórico.
**Where**: `backend/src/main/java/io/databoard/security/auth/PasswordPolicyValidator.java`
**Depends on**: T2
**Reuses**: `Argon2PasswordEncoder` (F01)
**Requirement**: AUTH-05

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: 9 caracteres → erro de tamanho; só letras → erro de complexidade; igual à penúltima senha com histórico 3 → `PASSWORD_REUSED`; igual à 4ª mais antiga com histórico 3 → aceita

**Tests**: unit
**Gate**: quick
**Commit**: `feat(security): validate password policy and history`

---

#### T4: `LockoutPolicy`

**What**: Decisões puras de bloqueio com `Clock`.
**Where**: `backend/src/main/java/io/databoard/security/auth/LockoutPolicy.java`
**Depends on**: T2
**Reuses**: —
**Requirement**: AUTH-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: 4ª falha não bloqueia; 5ª bloqueia até `now+15min`; `locked_until` no passado → desbloqueado

**Tests**: unit
**Gate**: quick
**Commit**: `feat(security): add account lockout policy`

---

### Phase 2: Login, sessão e troca de senha

#### T5: `SecurityConfig` com CSRF e respostas problem+json

**What**: Filter chain com rotas públicas, CSRF por cookie, entry point 401 `UNAUTHENTICATED` e 403 `CSRF_INVALID`.
**Where**: `backend/src/main/java/io/databoard/security/config/SecurityConfig.java`
**Depends on**: None
**Reuses**: `ProblemDetailsHandler` (F00)
**Requirement**: AUTH-02

**Tools**: MCP: NONE · Skill: `java-springboot`, `security-and-hardening`

**Done when**:
- [ ] `SecurityConfigIT`: rota protegida sem sessão → 401 `UNAUTHENTICATED`; POST autenticado sem CSRF → 403 `CSRF_INVALID`; `/api/v1/setup/status` e `/actuator/health` públicos

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): configure filter chain with csrf`

---

#### T6: `LoginService`

**What**: Fluxo de login com bloqueio, contadores, `passwordChangeRequired` e publicação de eventos.
**Where**: `backend/src/main/java/io/databoard/security/auth/LoginService.java`
**Depends on**: None
**Reuses**: `LockoutPolicy` (T4), `SecurityPolicyProvider` (T2)
**Requirement**: AUTH-01, AUTH-04, AUTH-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: senha certa → sucesso e contador zerado; e-mail inexistente e senha errada → mesmo `INVALID_CREDENTIALS`; inativo → `INVALID_CREDENTIALS`; bloqueado com senha certa → `ACCOUNT_LOCKED`; senha com 91 dias e expiração 90 → `passwordChangeRequired=true`; eventos `LoginSucceeded`/`LoginFailed` publicados

**Tests**: unit
**Gate**: quick
**Commit**: `feat(security): implement login service with lockout`

---

#### T7: `AuthController` — login, logout, me, options

**What**: Endpoints `POST /auth/login`, `POST /auth/logout`, `GET /auth/me`, `GET /auth/options`, com rotação do ID de sessão e registro em `app.user_session`.
**Where**: `backend/src/main/java/io/databoard/security/auth/AuthController.java`
**Depends on**: T5, T6
**Reuses**: `HttpSessionSecurityContextRepository`
**Requirement**: AUTH-01, AUTH-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AuthControllerIT`: login 200 com cookie `DBSESSION` `HttpOnly`/`SameSite=Lax` e ID diferente do anterior; `/me` com os campos; logout → `/me` 401; 6ª tentativa → 423 (CA-03.1); corpo > 10 KB → 400

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): expose login, logout and me endpoints`

---

#### T8: `SessionLifetimeFilter` — ociosidade e duração máxima

**What**: `maxInactiveInterval` a partir da política no login; invalidação após `sessionMaxHours`.
**Where**: `backend/src/main/java/io/databoard/security/session/SessionLifetimeFilter.java`
**Depends on**: T7
**Reuses**: `SecurityPolicyProvider` (T2)
**Requirement**: AUTH-08

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SessionLifetimeIT` (relógio controlado): 13 h após o login → 401; ociosidade configurada em 1 min e 61 s sem requisição → 401; requisição dentro da janela renova a ociosidade

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): enforce idle and absolute session lifetime`

---

#### T9: Troca de senha e filtro de troca obrigatória

**What**: `POST /auth/password/change` + `PasswordChangeRequiredFilter` com whitelist; troca encerra as demais sessões do usuário.
**Where**: `backend/src/main/java/io/databoard/security/session/PasswordChangeRequiredFilter.java`
**Depends on**: T7
**Reuses**: `PasswordPolicyValidator` (T3)
**Requirement**: AUTH-05, AUTH-06

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `PasswordChangeIT`: senha atual errada → 400 `currentPassword`; senha reutilizada → `PASSWORD_REUSED`; sessão com troca obrigatória → outra rota 403 `PASSWORD_CHANGE_REQUIRED`; após trocar → acesso liberado; outra sessão do mesmo usuário → 401

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): support password change and forced change`

---

### Phase 3: Reset de senha e gestão de sessões

#### T10: `MailService`

**What**: Envio de e-mail a partir do SMTP do `SettingsStore` (senha decifrada com `CryptoService`) e `isConfigured()`.
**Where**: `backend/src/main/java/io/databoard/notification/mail/MailService.java`
**Depends on**: None
**Reuses**: `SettingsStore`, `CryptoService` (F01)
**Requirement**: AUTH-07

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `MailServiceIT` com GreenMail: mensagem entregue com assunto e corpo; sem seção `smtp` → `isConfigured()=false`

**Tests**: integration
**Gate**: full
**Commit**: `feat(notification): add smtp mail service`

---

#### T11: Reset de senha

**What**: `PasswordResetService` + endpoints `forgot` (sempre 202) e `reset` (token de uso único, 30 min, encerra sessões).
**Where**: `backend/src/main/java/io/databoard/security/auth/PasswordResetService.java`
**Depends on**: T10
**Reuses**: `MailService` (T10), `SessionRegistryService` (criado na T12; até lá, chamada via interface `UserSessionTerminator` com implementação no-op)
**Requirement**: AUTH-07

**Tools**: MCP: NONE · Skill: `java-springboot`, `security-and-hardening`

**Done when**:
- [ ] `PasswordResetIT` (GreenMail): e-mail existente e inexistente → 202 com corpo idêntico; link capturado → reset → login com a nova senha; token reutilizado → `RESET_TOKEN_INVALID`; token com 31 min → `RESET_TOKEN_INVALID`; `/auth/options` reflete SMTP configurado ou não

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): add password reset via email`

---

#### T12: Gestão de sessões pelo admin

**What**: `SessionRegistryService` (implementa `UserSessionTerminator`), `GET/DELETE /admin/sessions` e `POST /admin/users/{id}/unlock`.
**Where**: `backend/src/main/java/io/databoard/security/session/AdminSessionController.java`
**Depends on**: None
**Reuses**: `FindByIndexNameSessionRepository` do Spring Session
**Requirement**: AUTH-04, AUTH-09

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AdminSessionIT`: lista mostra usuário, IP, user-agent, início, último acesso; DELETE da sessão de João → próxima requisição de João 401 (CA-03.2); VIEWER → 403; unlock zera o bloqueio

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): let admins list and terminate sessions`

---

#### T13: Encerrar sessões ao revogar acesso

**What**: Listener do evento `UserAccessRevoked` (publicado pela F04 ao bloquear/inativar) que chama `terminateAllFor`.
**Where**: `backend/src/main/java/io/databoard/security/session/AccessRevokedListener.java`
**Depends on**: T12
**Reuses**: `SessionRegistryService` (T12)
**Requirement**: AUTH-10

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AccessRevokedIT`: publicar `UserAccessRevoked(userId)` com 2 sessões ativas → ambas recebem 401 em seguida

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): terminate sessions when access is revoked`

---

### Phase 4: Frontend e E2E

#### T14: `AuthProvider` e `RequireAuth`

**What**: Contexto de autenticação sobre `/auth/me`, redirecionamento 401 → `/login?next=` e para a troca obrigatória.
**Where**: `frontend/src/features/auth/AuthProvider.tsx`
**Depends on**: None
**Reuses**: `http.ts` (F00), `SetupGate` (F01)
**Requirement**: AUTH-03, AUTH-06
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes (MSW): `/me` 401 → navega para `/login?next=/rota`; `passwordChangeRequired` → `/trocar-senha`; sucesso → renderiza a rota

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add auth provider and route guard`

---

#### T15: `LoginPage`

**What**: Formulário de login com mensagens para 401/423 e link condicional "Esqueci minha senha".
**Where**: `frontend/src/features/auth/LoginPage.tsx`
**Depends on**: T14
**Reuses**: Mantine `useForm`
**Requirement**: AUTH-01, AUTH-07

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `frontend-design`

**Done when**:
- [ ] Testes: 401 → "Credenciais inválidas"; 423 → "Conta bloqueada temporariamente"; `passwordResetAvailable=false` → link oculto; sucesso → navega para `next`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add login page`

---

#### T16: `ForcedPasswordChangePage`

**What**: Tela de troca (obrigatória ou voluntária) com erros por campo.
**Where**: `frontend/src/features/auth/PasswordChangePage.tsx`
**Depends on**: T15
**Reuses**: —
**Requirement**: AUTH-05, AUTH-06

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: erro em `currentPassword` exibido; `PASSWORD_REUSED` exibe "Senha usada recentemente"; sucesso → home

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add password change page`

---

#### T17: `ForgotPasswordPage`

**What**: Tela que envia o e-mail e sempre mostra "Se o e-mail existir, você receberá um link".
**Where**: `frontend/src/features/auth/ForgotPasswordPage.tsx`
**Depends on**: T15
**Reuses**: —
**Requirement**: AUTH-07

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Teste: após envio, mensagem neutra independentemente da resposta

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add forgot password page`

---

#### T18: `ResetPasswordPage`

**What**: Tela que lê `?token=`, envia a nova senha e trata `RESET_TOKEN_INVALID`.
**Where**: `frontend/src/features/auth/ResetPasswordPage.tsx`
**Depends on**: T17
**Reuses**: —
**Requirement**: AUTH-07

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: token inválido → "Link expirado ou já utilizado"; sucesso → login com aviso

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add reset password page`

---

#### T19: `SessionsPage` (admin)

**What**: Tabela de sessões ativas com ação "Encerrar" e confirmação em modal Mantine (sem `window.confirm`).
**Where**: `frontend/src/features/admin/sessions/SessionsPage.tsx`
**Depends on**: T14
**Reuses**: TanStack Table
**Requirement**: AUTH-09

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: renderiza as colunas; "Encerrar" chama `DELETE` e remove a linha

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add active sessions admin page`

---

#### T20: E2E de autenticação

**What**: Playwright: login do admin criado no setup, logout, 5 senhas erradas → bloqueio visível, admin encerra a sessão de outro usuário.
**Where**: `frontend/e2e/auth.spec.ts`
**Depends on**: T16
**Reuses**: Fixture de setup (F01)
**Requirement**: AUTH-01, AUTH-04, AUTH-09

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): cover login, lockout and session termination`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T2 | None | inícios da fase 1 | ✅ |
| T3 | T2 | T2 → T3 | ✅ |
| T4 | T2 | T2 → T4 | ✅ |
| T5, T6 | None | inícios da fase 2 | ✅ |
| T7 | T5, T6 | T5 → T7, T6 → T7 | ✅ |
| T8 | T7 | T7 → T8 | ✅ |
| T9 | T7 | T7 → T9 | ✅ |
| T10, T12 | None | inícios da fase 3 | ✅ |
| T11 | T10 | T10 → T11 | ✅ |
| T13 | T12 | T12 → T13 | ✅ |
| T14 | None | início da fase 4 | ✅ |
| T15 | T14 | T14 → T15 | ✅ |
| T16 | T15 | T15 → T16 | ✅ |
| T17 | T15 | T15 → T17 | ✅ |
| T18 | T17 | T17 → T18 | ✅ |
| T19 | T14 | T14 → T19 | ✅ |
| T20 | T16 | T16 → T20 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T2, T3, T4, T6 | Backend políticas/serviços | unit | unit | ✅ |
| T1, T5, T7–T13 | Backend persistência/controller/filtro | integration | integration | ✅ |
| T14–T19 | Frontend | unit | unit | ✅ |
| T20 | Ponta a ponta | e2e | e2e | ✅ |
