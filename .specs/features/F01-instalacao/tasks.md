# F01 — Distribuição e Instalação Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F01-instalacao/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Guidelines found: `.specs/STATE.md` AD-017; padrões da F00 (`.specs/features/F00-fundacao/tasks.md`).

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio/serviço | unit | Todos os ramos; 1:1 com ACs; edge cases | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend controller/persistência/startup | integration | Rotas: feliz + edge + erro | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| Frontend páginas/componentes | unit | Fluxos e estados de erro | `frontend/src/**/*.test.tsx` | `make test-frontend` |
| Fluxo ponta a ponta | e2e | Stories P1 | `frontend/e2e/*.spec.ts` | `make e2e` |
| Compose / scripts / workflow | none | build gate | - | `make build` |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Só unit | `make test-backend` ou `make test-frontend` |
| Full | Com integração/e2e | `make test-backend it-backend` (+ `make e2e` quando houver e2e) |
| Build | Fim de fase / config | `make ci` |

---

## Execution Plan

### Phase 1: Guardas e cifragem

```
T1
T2
T3
```

### Phase 2: Usuário, configurações e setup

```
T4 → T5 → T6 → T7
```

### Phase 3: Informações e saúde

```
T8
T9
```

### Phase 4: Frontend

```
T10 → T11
T12
```

### Phase 5: Distribuição

```
T13 → T14 → T15
T13 → T16
```

---

## Task Breakdown

### Phase 1: Guardas e cifragem

#### T1: `StartupGuard` para variáveis obrigatórias

**What**: `EnvironmentPostProcessor` que aborta sem `DATABOARD_SECRET_KEY` (≥ 32) ou sem `DATABOARD_INTERNAL_TOKEN`, com mensagens da spec.
**Where**: `backend/src/main/java/io/databoard/platform/config/StartupGuard.java`
**Depends on**: None
**Reuses**: —
**Requirement**: INST-02
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes: chave ausente, chave com 31 caracteres, token ausente → exceção com a mensagem exata; ambos válidos → passa

**Tests**: unit
**Gate**: quick
**Commit**: `feat(platform): fail fast on missing required secrets`

---

#### T2: `CryptoService` AES-GCM + HKDF

**What**: Cifragem `v1:<base64>` com nonce aleatório e erro `CREDENTIAL_UNREADABLE` com chave trocada.
**Where**: `backend/src/main/java/io/databoard/platform/crypto/CryptoService.java`
**Depends on**: None
**Reuses**: `DomainException` (F00)
**Requirement**: INST-06
**Tools**: MCP: NONE · Skill: `java-springboot`, `security-and-hardening`

**Done when**:
- [ ] Testes: ida e volta; dois `encrypt` do mesmo valor diferem; chave trocada → `CREDENTIAL_UNREADABLE`; token adulterado → `CREDENTIAL_UNREADABLE`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(platform): add aes-gcm credential encryption`

---

#### T3: Proteção contra downgrade do schema

**What**: `spring.flyway.ignore-migration-patterns=` vazio + `FlywayFailureAnalyzer` com mensagem pt-BR.
**Where**: `backend/src/main/java/io/databoard/platform/config/FlywayDowngradeFailureAnalyzer.java`
**Depends on**: None
**Reuses**: —
**Requirement**: INST-07
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `DowngradeGuardIT`: insere versão `999` em `flyway_schema_history` e verifica que o contexto falha com a mensagem `Banco em versão mais nova que a aplicação; downgrade não suportado`
- [ ] Sem linha futura, o contexto sobe e aplica migrações pendentes

**Tests**: integration
**Gate**: full
**Commit**: `feat(platform): abort startup on schema downgrade`

---

### Phase 2: Usuário, configurações e setup

#### T4: Tabela e repositório de usuários

**What**: Migração `V2__user_and_settings.sql` (parte `user_account`), entidade `UserAccount`, `UserAccountRepository` e bean `Argon2PasswordEncoder`.
**Where**: `backend/src/main/java/io/databoard/security/user/UserAccountRepository.java`
**Depends on**: None
**Reuses**: —
**Requirement**: INST-04
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `UserAccountRepositoryIT`: salvar/buscar por e-mail case-insensitive; e-mail duplicado viola `ux_user_email`
- [ ] Hash gerado começa com `$argon2id$`

**Tests**: integration
**Gate**: full
**Commit**: `feat(security): add user account persistence`

---

#### T5: `SettingsStore` chave-valor JSON

**What**: Tabela `app.setting` (mesma migração V2) e `SettingsStore` com `get/put` tipados.
**Where**: `backend/src/main/java/io/databoard/platform/settings/SettingsStore.java`
**Depends on**: T4
**Reuses**: —
**Requirement**: INST-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SettingsStoreIT`: `put` e `get` de um record; seção ausente → `Optional.empty()`

**Tests**: integration
**Gate**: full
**Commit**: `feat(platform): add json settings store`

---

#### T6: `SetupService`

**What**: `setupRequired()` e `complete()` com advisory lock, validação de senha/fuso/e-mail, SMTP opcional cifrado.
**Where**: `backend/src/main/java/io/databoard/platform/setup/SetupService.java`
**Depends on**: T5
**Reuses**: `CryptoService` (T2), `SettingsStore` (T5), `UserAccountRepository` (T4)
**Requirement**: INST-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Testes unitários (repositórios mockados): cria admin `ADMIN` com hash; com usuários → `SetupNotAvailableException`; SMTP gravado com senha cifrada; sem SMTP não grava a seção

**Tests**: unit
**Gate**: quick
**Commit**: `feat(platform): implement initial setup service`

---

#### T7: `SetupController` público

**What**: `GET /api/v1/setup/status` e `POST /api/v1/setup` liberados sem autenticação.
**Where**: `backend/src/main/java/io/databoard/platform/setup/SetupController.java`
**Depends on**: T6
**Reuses**: `ProblemDetailsHandler` (F00)
**Requirement**: INST-04

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SetupControllerIT`: status true → POST 201 → status false → POST 404 (CA-01.3)
- [ ] Senha fraca → 400 em `admin.password`; fuso inválido → 400 em `timezone`; e-mail inválido → 400 em `admin.email`
- [ ] Dois POST concorrentes → um 201 e um 404

**Tests**: integration
**Gate**: full
**Commit**: `feat(platform): expose setup endpoints`

---

### Phase 3: Informações e saúde

#### T8: `SystemInfoController`

**What**: `GET /api/v1/system/info` com `version`, `commit`, `schemaVersion`, `buildDate`; plugins `build-info` e `git-commit-id` no `pom.xml`.
**Where**: `backend/src/main/java/io/databoard/platform/system/SystemInfoController.java`
**Depends on**: None
**Reuses**: —
**Requirement**: INST-08
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SystemInfoIT`: os quatro campos presentes; `schemaVersion` igual à última migração

**Tests**: integration
**Gate**: full
**Commit**: `feat(platform): expose system info endpoint`

---

#### T9: Readiness com analytics

**What**: `AnalyticsHealthIndicator` + grupos `liveness`/`readiness` (`readiness` = `db` + `analytics`).
**Where**: `backend/src/main/java/io/databoard/platform/analytics/AnalyticsHealthIndicator.java`
**Depends on**: None
**Reuses**: `AnalyticsClient` (F00)
**Requirement**: INST-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `ReadinessIT`: analytics ativo → `UP`; analytics parado → readiness `DOWN` e liveness `UP`

**Tests**: integration
**Gate**: full
**Commit**: `feat(platform): include analytics in readiness probe`

---

### Phase 4: Frontend

#### T10: `SetupGate` com redirecionamento

**What**: Layout raiz que consulta `/setup/status` e redireciona para `/setup` quando necessário.
**Where**: `frontend/src/features/setup/SetupGate.tsx`
**Depends on**: None
**Reuses**: `http.ts` (F00)
**Requirement**: INST-05
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes (MSW): `setupRequired=true` em `/qualquer` → navega para `/setup`; `false` → renderiza filhos

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): redirect to setup wizard when required`

---

#### T11: `SetupWizardPage`

**What**: Stepper Empresa → Administrador → SMTP (opcional, pulável), com erros por campo vindos de `ApiError.errors` e redirecionamento ao login no sucesso.
**Where**: `frontend/src/features/setup/SetupWizardPage.tsx`
**Depends on**: T10
**Reuses**: Mantine `Stepper`, `useForm`
**Requirement**: INST-05

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`, `frontend-design`

**Done when**:
- [ ] Testes: fluxo feliz envia o payload esperado; erro 400 em `admin.password` aparece no campo; botão "Pular" no SMTP envia sem `smtp`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add initial setup wizard`

---

#### T12: `AboutPage`

**What**: Página **Sobre** com versão, commit, schema e link para o changelog.
**Where**: `frontend/src/features/system/AboutPage.tsx`
**Depends on**: None
**Reuses**: `http.ts` (F00)
**Requirement**: INST-08

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Teste: renderiza os quatro valores retornados pela API (MSW)

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add about page`

---

### Phase 5: Distribuição

#### T13: Compose oficial e `.env.example`

**What**: `deploy/docker-compose.yml` com imagens publicadas, volumes `databoard-db`/`databoard-files`, healthchecks em readiness, `analytics` sem porta, e `deploy/.env.example`.
**Where**: `deploy/docker-compose.yml`
**Depends on**: None
**Reuses**: `deploy/docker-compose.dev.yml` (F00)
**Requirement**: INST-01, INST-02
**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `docker compose -f deploy/docker-compose.yml config` válido
- [ ] Todas as `DATABOARD_*` documentadas no `.env.example`

**Tests**: none
**Gate**: build
**Commit**: `build(deploy): add official compose file`

---

#### T14: Scripts de backup/restore e documentação de operação

**What**: `backup.sh`, `restore.sh` (aborta com app rodando) e `docs/operacao.md` (instalação, upgrade, restauração, aviso sobre o wizard).
**Where**: `deploy/scripts/backup.sh`
**Depends on**: T13
**Reuses**: —
**Requirement**: INST-09

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `shellcheck` limpo nos dois scripts
- [ ] Execução manual: backup → `docker compose down -v` → restore → admin continua existindo

**Tests**: none
**Gate**: build
**Commit**: `docs(deploy): add backup, restore and operations guide`

---

#### T15: E2E de instalação

**What**: Teste Playwright contra o compose oficial (imagens locais): wizard em ≤ 60 s, conclui setup, `/setup` redireciona para o login e `POST /setup` responde 404.
**Where**: `frontend/e2e/install.spec.ts`
**Depends on**: T14
**Reuses**: Smoke E2E (F00)
**Requirement**: INST-01, INST-04, INST-05

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde (CA-01.1, CA-01.3)

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): cover first-run installation flow`

---

#### T16: Workflow de release multi-arch

**What**: `.github/workflows/release.yml`, disparado em tag `v*`, publica as duas imagens para `amd64`/`arm64` com tags `X.Y.Z` e `latest`.
**Where**: `.github/workflows/release.yml`
**Depends on**: T13
**Reuses**: Dockerfiles (F00)
**Requirement**: INST-10

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `actionlint` limpo
- [ ] Build local `docker buildx build --platform linux/amd64,linux/arm64` das duas imagens conclui

**Tests**: none
**Gate**: build
**Commit**: `ci: publish multi-arch images on release tags`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1, T2, T3 | None | nós isolados na fase 1 | ✅ |
| T4 | None | início da cadeia | ✅ |
| T5 | T4 | T4 → T5 | ✅ |
| T6 | T5 | T5 → T6 | ✅ |
| T7 | T6 | T6 → T7 | ✅ |
| T8, T9 | None | nós isolados na fase 3 | ✅ |
| T10 | None | início | ✅ |
| T11 | T10 | T10 → T11 | ✅ |
| T12 | None | isolado | ✅ |
| T13 | None | início | ✅ |
| T14 | T13 | T13 → T14 | ✅ |
| T15 | T14 | T14 → T15 | ✅ |
| T16 | T13 | T13 → T16 | ✅ |

## Test Co-location Validation

| Task | Code Layer | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1, T2, T6 | Backend domínio | unit | unit | ✅ |
| T3, T4, T5, T7, T8, T9 | Backend startup/persistência/controller | integration | integration | ✅ |
| T10, T11, T12 | Frontend | unit | unit | ✅ |
| T13, T14, T16 | Compose/scripts/workflow | none | none | ✅ |
| T15 | Fluxo ponta a ponta | e2e | e2e | ✅ |
