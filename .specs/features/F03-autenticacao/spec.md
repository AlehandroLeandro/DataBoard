# F03 — Autenticação e Sessão Specification

**Prioridade:** MVP (login local) / V1 (verificação de e-mail) / V2 (OIDC/LDAP) · **Depende de:** F01 · **Origem:** F03 do `produto.specs.md`

## Problem Statement

Depois do setup existe um admin, mas ninguém consegue entrar. A plataforma precisa de login com política de senha, bloqueio por tentativas, expiração de sessão e controle de sessões ativas. Sem isso, nenhuma regra de permissão (F04) ou RLS (F05) tem a quem se aplicar.

## Goals

- [ ] Usuário entra com e-mail e senha e recebe sessão por cookie seguro.
- [ ] 5 falhas seguidas bloqueiam a conta por 15 min (CA-03.1).
- [ ] Admin encerra uma sessão e a próxima requisição do dono recebe 401 (CA-03.2).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Verificação de e-mail por convite (RF-03.9) | V1 |
| OIDC/LDAP (RF-03.10) | V2 |
| CRUD de usuários, papéis e grupos; regra do último admin (RN-03.3) | F04 |
| Tela de edição dos parâmetros de segurança | F02 (aqui são lidos com padrões) |
| JWT | AD-008 |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Mecanismo de sessão | Spring Session JDBC, cookie `DBSESSION` `HttpOnly`, `Secure` (desligável só com `DATABOARD_BASE_URL` http), `SameSite=Lax` | AD-008 | y |
| CSRF | `CookieCsrfTokenRepository` (cookie `XSRF-TOKEN` legível pelo JS, header `X-XSRF-TOKEN`) | Padrão SPA; casa com `http.ts` da F00 | y |
| Parâmetros padrão | Senha mín. 10, exige letra e número, expiração 0 (nunca), histórico 3; bloqueio 5 tentativas / 15 min; ociosidade 30 min; duração máxima 12 h | Valores do produto; F02 permite alterar | y |
| Onde ficam os parâmetros | `SettingsStore` seção `security`, com padrões quando ausente | F02 só cria a UI/API de edição | y |
| Login com senha expirada ou troca forçada | Login responde 200 com `passwordChangeRequired=true`; a sessão só acessa `/auth/password/change`, `/auth/me` e `/auth/logout` (demais → 403 `PASSWORD_CHANGE_REQUIRED`) | Atende CA-03.3 sem segundo fluxo de autenticação | y |
| Envio de e-mail | `MailService` mínimo nesta feature, lendo SMTP do `SettingsStore` | "Esqueci minha senha" é MVP; F02 cria a tela de SMTP | y |
| Auditoria de login | A F03 publica eventos de domínio (`LoginSucceeded`, `LoginFailed`, `LoggedOut`); a F06 os persiste | Desacoplamento entre módulos (Modulith) | y |
| Contagem de tentativas | Por conta (e-mail existente). Sem rate limit por IP no MVP | Escopo do produto; rate limit por IP registrado como risco | y |
| Token de reset | 32 bytes aleatórios, guardado como SHA-256, uso único, validade 30 min | RF-03.5 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Login e logout ⭐ MVP

**User Story**: Como usuário, quero entrar com e-mail e senha para acessar os dashboards a que tenho direito.

**Why P1**: Porta de entrada de tudo.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/auth/login` recebe e-mail e senha corretos de um usuário ativo THEN o sistema SHALL responder 200, criar uma sessão nova (ID rotacionado) e definir o cookie `DBSESSION` com `HttpOnly` e `SameSite=Lax`.
2. IF o e-mail não existe ou a senha está errada THEN o sistema SHALL responder 401 com `code=INVALID_CREDENTIALS` e a mesma mensagem "Credenciais inválidas" nos dois casos.
3. IF o usuário está inativo THEN o sistema SHALL responder 401 com `code=INVALID_CREDENTIALS`.
4. WHEN `GET /api/v1/auth/me` é chamado com sessão válida THEN o sistema SHALL responder `{ id, name, email, roles, permissions, preferences }`.
5. WHEN `POST /api/v1/auth/logout` é chamado THEN o sistema SHALL invalidar a sessão e a próxima requisição com o mesmo cookie SHALL receber 401.
6. IF uma requisição mutável (`POST/PUT/PATCH/DELETE`) chega sem token CSRF válido THEN o sistema SHALL responder 403 com `code=CSRF_INVALID`.
7. IF uma rota `/api/v1/**` não pública é chamada sem sessão THEN o sistema SHALL responder 401 com `code=UNAUTHENTICATED`.

**Independent Test**: IT de login → `/me` → logout → `/me` 401.

---

### P1: Bloqueio por tentativas ⭐ MVP

**User Story**: Como administrador, quero que contas sejam bloqueadas após tentativas falhas para dificultar ataques de força bruta.

**Why P1**: Requisito de segurança básico.

**Acceptance Criteria**:
1. WHEN a 5ª tentativa falha seguida ocorre para uma conta THEN o sistema SHALL bloqueá-la por 15 min (valores de `security.lockout`).
2. WHILE a conta está bloqueada, WHEN o usuário tenta entrar, mesmo com a senha correta, THEN o sistema SHALL responder 423 com `code=ACCOUNT_LOCKED`.
3. WHEN um login bem-sucedido ocorre THEN o sistema SHALL zerar o contador de falhas.
4. WHEN o admin chama `POST /api/v1/admin/users/{id}/unlock` THEN o sistema SHALL desbloquear a conta e zerar o contador.
5. WHEN o período de bloqueio expira THEN o sistema SHALL aceitar novamente a senha correta.

**Independent Test**: CA-03.1 automatizado com relógio controlado (`Clock` injetável).

---

### P1: Política e troca de senha ⭐ MVP

**User Story**: Como usuário, quero trocar minha senha, e como admin quero forçar a troca, para manter as credenciais seguras.

**Why P1**: RF-03.3/RF-03.4 no MVP.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/auth/password/change` recebe a senha atual correta e uma nova senha válida THEN o sistema SHALL gravar o novo hash, registrar `password_changed_at` e limpar `must_change_password`.
2. IF a senha atual está errada THEN o sistema SHALL responder 400 com erro no campo `currentPassword`.
3. IF a nova senha viola tamanho mínimo ou complexidade THEN o sistema SHALL responder 400 com erro em `newPassword` descrevendo a regra violada.
4. IF a nova senha é igual a uma das últimas N senhas (`security.passwordHistory`) THEN o sistema SHALL responder 400 com `code=PASSWORD_REUSED`.
5. WHEN o usuário com `must_change_password=true` ou com senha mais antiga que `security.passwordExpiryDays` (> 0) faz login THEN o sistema SHALL responder 200 com `passwordChangeRequired=true`.
6. WHILE `passwordChangeRequired` vale para a sessão THEN o sistema SHALL responder 403 `PASSWORD_CHANGE_REQUIRED` a qualquer rota fora de `/auth/password/change`, `/auth/me` e `/auth/logout`.

**Independent Test**: CA-03.3: senha com 91 dias e expiração em 90 → login retorna `passwordChangeRequired=true` e `/api/v1/screens` retorna 403.

---

### P1: Esqueci minha senha ⭐ MVP

**User Story**: Como usuário, quero redefinir a senha por e-mail quando a esqueço.

**Why P1**: Evita depender do admin para toda senha esquecida.

**Acceptance Criteria**:
1. WHERE SMTP está configurado, WHEN `POST /api/v1/auth/password/forgot` recebe um e-mail THEN o sistema SHALL responder 202 e, se o e-mail pertencer a um usuário ativo, enviar um link com token de uso único válido por 30 min.
2. The resposta de `password/forgot` SHALL ser 202 com corpo idêntico para e-mails existentes e inexistentes.
3. WHEN `POST /api/v1/auth/password/reset` recebe token válido e nova senha válida THEN o sistema SHALL trocar a senha, marcar o token como usado e encerrar todas as sessões do usuário.
4. IF o token está expirado, já usado ou não existe THEN o sistema SHALL responder 400 com `code=RESET_TOKEN_INVALID`.
5. IF SMTP não está configurado THEN `GET /api/v1/auth/options` SHALL responder `passwordResetAvailable=false` e a tela de login SHALL ocultar o link "Esqueci minha senha".

**Independent Test**: IT com GreenMail: forgot → captura link → reset → login com a nova senha.

---

### P1: Expiração e gestão de sessões ⭐ MVP

**User Story**: Como administrador, quero ver e encerrar sessões ativas, e que sessões expirem sozinhas, para controlar acessos.

**Why P1**: RF-03.7, RF-03.8 e RN-03.2.

**Acceptance Criteria**:
1. WHILE uma sessão fica sem requisições por mais que `security.sessionIdleMinutes` (padrão 30) THEN o sistema SHALL tratá-la como expirada e responder 401.
2. WHEN uma sessão atinge `security.sessionMaxHours` (padrão 12) desde o login THEN o sistema SHALL invalidá-la na próxima requisição e responder 401.
3. WHEN o admin chama `GET /api/v1/admin/sessions` THEN o sistema SHALL listar as sessões ativas com usuário, IP, user-agent, início e último acesso.
4. WHEN o admin chama `DELETE /api/v1/admin/sessions/{id}` THEN o sistema SHALL encerrar a sessão, e a próxima requisição do dono SHALL receber 401.
5. WHEN um usuário é bloqueado ou inativado THEN o sistema SHALL encerrar imediatamente todas as sessões dele.
6. IF um não-admin chama `/api/v1/admin/sessions` THEN o sistema SHALL responder 403.

**Independent Test**: CA-03.2 em IT com duas sessões (admin e João).

---

### P2: Verificação de e-mail por convite [V1]

**User Story**: Como admin, quero convidar usuários por e-mail com link de ativação.

**Why P2**: V1.

**Acceptance Criteria**:
1. WHERE a verificação de e-mail está ativa, WHEN um usuário não verificado tenta entrar THEN o sistema SHALL responder 401 com `code=EMAIL_NOT_VERIFIED`.

**Independent Test**: Diferido para a V1.

---

### P3: Provedores externos OIDC/LDAP [V2]

**User Story**: Como admin, quero autenticar usuários pelo Google, Microsoft, Keycloak ou AD.

**Why P3**: V2.

**Acceptance Criteria**:
1. WHERE um provedor OIDC está configurado, WHEN o usuário escolhe "Entrar com <provedor>" THEN o sistema SHALL autenticar via OIDC e associar a conta pelo e-mail.

---

## Edge Cases

- IF o corpo do login excede 10 KB ou o e-mail excede 320 caracteres THEN o sistema SHALL responder 400 sem consultar o banco.
- WHEN dois logins falhos simultâneos ocorrem para a mesma conta THEN o contador SHALL incrementar duas vezes (update atômico `failed_attempts = failed_attempts + 1`).
- IF o envio de e-mail do reset falhar THEN o sistema SHALL logar o erro e ainda responder 202 (não revela existência).
- WHEN o usuário troca a própria senha THEN o sistema SHALL encerrar as demais sessões dele e manter a atual.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| AUTH-01 | P1: Login/logout | RF-03.1, RF-03.2, RN-03.1 | Tasks | Pending |
| AUTH-02 | P1: Login/logout (CSRF, 401) | RF-03.2 | Tasks | Pending |
| AUTH-03 | P1: Login/logout (`/me`) | Contrato F03 | Tasks | Pending |
| AUTH-04 | P1: Bloqueio | RF-03.6, CA-03.1 | Tasks | Pending |
| AUTH-05 | P1: Política/troca de senha | RF-03.3, RF-03.4 | Tasks | Pending |
| AUTH-06 | P1: Troca forçada/expiração | RF-03.3, CA-03.3 | Tasks | Pending |
| AUTH-07 | P1: Esqueci minha senha | RF-03.5 | Tasks | Pending |
| AUTH-08 | P1: Sessões (expiração) | RF-03.7 | Tasks | Pending |
| AUTH-09 | P1: Sessões (admin) | RF-03.8, CA-03.2 | Tasks | Pending |
| AUTH-10 | P1: Sessões (bloqueio encerra) | RN-03.2 | Tasks | Pending |
| AUTH-11 | P2: Verificação de e-mail | RF-03.9 | - | Pending |
| AUTH-12 | P3: OIDC/LDAP | RF-03.10 | - | Pending |

**Coverage:** 12 total, 10 mapped to tasks, 2 diferidos (V1/V2).

---

## Success Criteria

- [ ] CA-03.1, CA-03.2 e CA-03.3 cobertos por testes automatizados.
- [ ] Nenhuma resposta de autenticação diferencia e-mail existente de inexistente.
