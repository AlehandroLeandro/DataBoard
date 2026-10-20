# F00 — Fundação Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/F00-fundacao/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Generated from project guidelines and spec - confirm before Execute. Guidelines found: `.specs/STATE.md` AD-017 (repositório vazio, sem testes existentes - strong defaults applied).

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Backend domínio/serviço/filtro | unit | Todos os ramos; 1:1 com ACs; todos os edge cases | `backend/src/test/java/**/*Test.java` | `make test-backend` |
| Backend controller/persistência/integração externa | integration (Testcontainers) | Todas as rotas: feliz + edge + erro | `backend/src/test/java/**/*IT.java` | `make it-backend` |
| analytics lógica pura | unit | Todos os ramos; 1:1 com ACs | `analytics/tests/unit/test_*.py` | `make test-analytics` |
| analytics rotas HTTP | integration (TestClient) | Feliz + erro (401, 413) | `analytics/tests/api/test_*.py` | `make test-analytics` |
| Frontend componentes/utilitários | unit (Vitest) | Ramos de parse e renderização | `frontend/src/**/*.test.ts(x)` | `make test-frontend` |
| Fluxo ponta a ponta | e2e (Playwright) | Fluxo P1 da feature | `frontend/e2e/*.spec.ts` | `make e2e` |
| Migração / config / Dockerfile / CI | none | build gate | - | `make build` |

## Gate Check Commands

> Generated from design - confirm before Execute. Antes da T14 (Makefile) os comandos equivalentes diretos são usados: `cd backend && ./mvnw test`, `cd analytics && uv run pytest`, `cd frontend && npm run test -- --run`.

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Tasks só com testes unitários | `make test-backend` / `make test-analytics` / `make test-frontend` (camada tocada) |
| Full | Tasks com testes de integração | `make test-backend it-backend test-analytics` |
| Build | Fim de fase ou tasks de config | `make ci` |

---

## Execution Plan

### Phase 1: Esqueleto do backend

```
T1 → T2
T1 → T3 → T4
T1 → T5
```

### Phase 2: Esqueleto do analytics

```
T6 → T7 → T8
T6 → T9
```

### Phase 3: Integração Spring ↔ analytics

```
T10
```

### Phase 4: Frontend e SPA servida

```
T11 → T12
T11 → T13
```

### Phase 5: Ambiente e CI

```
T14 → T15 → T16 → T17
```

---

## Task Breakdown

### Phase 1: Esqueleto do backend

#### T1: Criar projeto Maven Spring Boot com verificação Modulith

**What**: `pom.xml` (Java 25, Spring Boot 4.x, Web, Actuator, Validation, JDBC, Flyway, Postgres, Spring Modulith, Testcontainers, Spotless), Maven Wrapper, `DataBoardApplication` e teste `ModularityTest` com `ApplicationModules.of(DataBoardApplication.class).verify()`.
**Where**: `backend/pom.xml`
**Depends on**: None
**Reuses**: —
**Requirement**: FND-01, FND-11
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `./mvnw test` compila e roda `ModularityTest` verde
- [ ] Surefire roda `*Test`, Failsafe roda `*IT`
- [ ] Spotless configurado (`./mvnw spotless:check` passa)

**Tests**: unit
**Gate**: quick
**Commit**: `build(backend): scaffold spring boot 4 project with modulith verification`

---

#### T2: Migração Flyway com schemas base

**What**: `V1__schemas.sql` criando `app`, `udt`, `lake`, `audit`; teste `SchemasIT` sobe Postgres 17 (Testcontainers) e verifica os 4 schemas.
**Where**: `backend/src/main/resources/db/migration/V1__schemas.sql`
**Depends on**: T1
**Reuses**: —
**Requirement**: FND-02

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SchemasIT` verde (consulta `information_schema.schemata`)
- [ ] Reexecutar a aplicação não falha (idempotente)

**Tests**: integration
**Gate**: full
**Commit**: `feat(platform): create base database schemas`

---

#### T3: Handler RFC 9457 com `code` estável

**What**: `DomainException` + `ProblemDetailsHandler` cobrindo exceção de domínio, validação (`VALIDATION_ERROR` + `errors[]`) e erro inesperado (`INTERNAL_ERROR` sem stack).
**Where**: `backend/src/main/java/io/databoard/platform/web/ProblemDetailsHandler.java`
**Depends on**: T1
**Reuses**: Spring `ProblemDetail`
**Requirement**: FND-07

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Teste `@WebMvcTest` com controller de teste cobre os 3 casos (AC 1, 2, 4 da story Padrões)
- [ ] Corpo de 500 não contém `trace`/`exception`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(platform): add problem+json error handling`

---

#### T4: Envelope de paginação

**What**: `PageResponse<T>` e resolução de `page/size/sort` com limite máximo de `size` (padrão 50, máximo 500).
**Where**: `backend/src/main/java/io/databoard/platform/web/PageResponse.java`
**Depends on**: T3
**Reuses**: Spring Data `Pageable`
**Requirement**: FND-08

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Teste unitário: `of(Page)` produz `{content,page,size,totalElements}`
- [ ] `size=1000` é limitado a 500; `size=0` → `VALIDATION_ERROR`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(platform): add pagination envelope`

---

#### T5: `RequestIdFilter` e logs JSON

**What**: Filtro que gera/propaga `X-Request-Id` no MDC e na resposta; `application.yml` com structured logging em JSON.
**Where**: `backend/src/main/java/io/databoard/platform/web/RequestIdFilter.java`
**Depends on**: T1
**Reuses**: Spring Boot structured logging
**Requirement**: FND-09

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] Teste: requisição sem header recebe um UUID novo na resposta; com header, o mesmo valor é devolvido
- [ ] Teste com `OutputCaptureExtension`: linha de log é JSON válido com `requestId`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(platform): propagate request id and emit json logs`

---

### Phase 2: Esqueleto do analytics

#### T6: Projeto FastAPI com `/health` e logs JSON

**What**: `pyproject.toml` (uv, Python 3.13, fastapi, uvicorn, polars, pydantic, pytest, httpx, ruff, pyright), `main.py` com `GET /health` e `logging.configure()` lendo `X-Request-Id`.
**Where**: `analytics/src/databoard_analytics/main.py`
**Depends on**: None
**Reuses**: —
**Requirement**: FND-04, FND-09
**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `tests/api/test_health.py`: `GET /health` → 200 `{"status":"UP"}` sem token
- [ ] Teste de log: linha JSON contém `requestId` recebido
- [ ] `uv run ruff check` e `uv run pyright` limpos

**Tests**: integration
**Gate**: quick
**Commit**: `build(analytics): scaffold fastapi service with health endpoint`

---

#### T7: Middleware de token interno + `/v1/echo`

**What**: `InternalTokenMiddleware` (compara com `DATABOARD_INTERNAL_TOKEN` via `hmac.compare_digest`), limite de corpo de 20 MB (413) e router `POST /v1/echo`.
**Where**: `analytics/src/databoard_analytics/security.py`
**Depends on**: T6
**Reuses**: —
**Requirement**: FND-04

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] Testes: sem token → 401; token errado → 401; token certo → echo do payload + `X-Request-Id`
- [ ] Payload > 20 MB → 413
- [ ] `/health` continua sem token

**Tests**: integration
**Gate**: quick
**Commit**: `feat(analytics): require internal token on v1 routes`

---

#### T8: Exportação e verificação do OpenAPI

**What**: Módulo `export_openapi` que grava `analytics/openapi.json`; flag `--check` falha se divergir do versionado.
**Where**: `analytics/src/databoard_analytics/export_openapi.py`
**Depends on**: T7
**Reuses**: FastAPI `app.openapi()`
**Requirement**: FND-06

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] Teste unitário: `--check` retorna 0 com arquivo igual e 1 com arquivo alterado
- [ ] `openapi.json` versionado contém `/v1/echo`

**Tests**: unit
**Gate**: quick
**Commit**: `feat(analytics): export and check openapi contract`

---

#### T9: Dockerfile do analytics

**What**: Imagem multi-stage (uv sync em build, runtime slim, usuário não-root, `uvicorn` na porta 8000, `HEALTHCHECK` em `/health`).
**Where**: `analytics/Dockerfile`
**Depends on**: T6
**Reuses**: —
**Requirement**: FND-01

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `docker build analytics` conclui
- [ ] Container responde `/health` 200

**Tests**: none
**Gate**: build
**Commit**: `build(analytics): add container image`

---

### Phase 3: Integração Spring ↔ analytics

#### T10: `AnalyticsClient` com timeout e mapeamento de erros

**What**: Cliente `RestClient` com headers `X-Internal-Token`/`X-Request-Id`, timeout 10 s, exceções `AnalyticsUnavailableException` (503) e `AnalyticsErrorException` (502).
**Where**: `backend/src/main/java/io/databoard/platform/analytics/AnalyticsClient.java`
**Depends on**: T3, T5, T9
**Reuses**: `ProblemDetailsHandler` (T3), `RequestIdFilter` (T5)
**Requirement**: FND-05
**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `AnalyticsClientIT` com analytics real (Testcontainers `GenericContainer` a partir de `analytics/Dockerfile`): echo ok, `X-Request-Id` propagado
- [ ] Token errado → `AnalyticsErrorException` mapeada para 502 `ANALYTICS_ERROR`
- [ ] Container parado → 503 `ANALYTICS_UNAVAILABLE`

**Tests**: integration
**Gate**: full
**Commit**: `feat(platform): add analytics client with error mapping`

---

### Phase 4: Frontend e SPA servida

#### T11: Scaffold do frontend

**What**: Vite + React 19 + TS + Mantine + TanStack Query + React Router + Vitest + ESLint; `App` com página placeholder "DataBoard"; proxy de dev para `/api` e `/actuator`.
**Where**: `frontend/package.json`
**Depends on**: None
**Reuses**: —
**Requirement**: FND-01, FND-03
**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] `src/app/App.test.tsx` renderiza "DataBoard"
- [ ] `npm run lint`, `npm run typecheck` e `npm run build` limpos

**Tests**: unit
**Gate**: quick
**Commit**: `build(frontend): scaffold react app with mantine`

---

#### T12: Cliente HTTP com CSRF e `ApiError`

**What**: `http.ts` com `fetch` + cookie, header CSRF (`X-XSRF-TOKEN` lido do cookie) e parse de problem+json em `ApiError { status, code, detail, errors }`.
**Where**: `frontend/src/shared/api/http.ts`
**Depends on**: T11
**Reuses**: —
**Requirement**: FND-07

**Tools**: MCP: NONE · Skill: `vercel-react-best-practices`

**Done when**:
- [ ] Testes: 200 → JSON; 400 problem+json → `ApiError` com `errors[]`; 500 não-JSON → `ApiError` com `code=INTERNAL_ERROR`
- [ ] Métodos mutáveis enviam header CSRF

**Tests**: unit
**Gate**: quick
**Commit**: `feat(frontend): add http client with problem+json parsing`

---

#### T13: Spring servindo a SPA com fallback

**What**: `frontend-maven-plugin` (perfil padrão; `-Pskip-frontend` pula) copia o build para `static/`; `SpaForwardController` encaminha rotas não-API para `index.html`.
**Where**: `backend/src/main/java/io/databoard/platform/web/SpaForwardController.java`
**Depends on**: T11
**Reuses**: —
**Requirement**: FND-03

**Tools**: MCP: NONE · Skill: `java-springboot`

**Done when**:
- [ ] `SpaForwardIT`: `GET /qualquer/rota` → 200 HTML; `GET /api/v1/inexistente` → 404 problem+json; `/actuator/health` não é encaminhado

**Tests**: integration
**Gate**: full
**Commit**: `feat(platform): serve spa with client-side routing fallback`

---

### Phase 5: Ambiente e CI

#### T14: Makefile na raiz

**What**: Alvos `dev`, `dev-front`, `test-backend`, `it-backend`, `test-analytics`, `test-frontend`, `lint`, `contract`, `build`, `e2e` e `ci`, conforme o design.
**Where**: `Makefile`
**Depends on**: None
**Reuses**: Comandos das fases 1–4
**Requirement**: FND-10
**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make test-backend test-analytics test-frontend lint contract` verdes

**Tests**: none
**Gate**: build
**Commit**: `build: add root makefile with quality gates`

---

#### T15: Compose de desenvolvimento e Dockerfile do app

**What**: `backend/Dockerfile` (multi-stage, JRE 25, não-root) e `deploy/docker-compose.dev.yml` com `postgres:17`, `analytics` (sem porta publicada, token de dev) e `app` (porta 8080, `depends_on` com healthcheck).
**Where**: `deploy/docker-compose.dev.yml`
**Depends on**: T14
**Reuses**: `analytics/Dockerfile` (T9)
**Requirement**: FND-01

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make dev` → `curl localhost:8080/actuator/health` 200 `UP` em ≤ 60 s
- [ ] `analytics` não acessível do host

**Tests**: none
**Gate**: build
**Commit**: `build(deploy): add development compose stack`

---

#### T16: Smoke E2E com Playwright

**What**: Configuração Playwright e `smoke.spec.ts` que abre `/` contra o compose de dev e vê "DataBoard"; abre `/rota/profunda` e vê a SPA.
**Where**: `frontend/e2e/smoke.spec.ts`
**Depends on**: T15
**Reuses**: —
**Requirement**: FND-01, FND-03

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] `make e2e` verde com o compose de dev ativo

**Tests**: e2e
**Gate**: full
**Commit**: `test(e2e): add smoke test for served spa`

---

#### T17: Workflow de CI

**What**: `.github/workflows/ci.yml` rodando `make ci` em PR e push para `main` (setup Java 25, Python 3.13 + uv, Node LTS, Docker para Testcontainers).
**Where**: `.github/workflows/ci.yml`
**Depends on**: T16
**Reuses**: Makefile (T14)
**Requirement**: FND-10

**Tools**: MCP: NONE · Skill: NONE

**Done when**:
- [ ] Workflow válido (`actionlint` limpo)
- [ ] Execução local equivalente `make ci` verde

**Tests**: none
**Gate**: build
**Commit**: `ci: run quality gates on pull requests`

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1 | None | início da fase 1 | ✅ |
| T2 | T1 | T1 → T2 | ✅ |
| T3 | T1 | T1 → T3 | ✅ |
| T4 | T3 | T3 → T4 | ✅ |
| T5 | T1 | T1 → T5 | ✅ |
| T6 | None | início da fase 2 | ✅ |
| T7 | T6 | T6 → T7 | ✅ |
| T8 | T7 | T7 → T8 | ✅ |
| T9 | T6 | T6 → T9 | ✅ |
| T10 | T3, T5, T9 (fases anteriores) | fase 3 isolada | ✅ |
| T11 | None | início da fase 4 | ✅ |
| T12 | T11 | T11 → T12 | ✅ |
| T13 | T11 | T11 → T13 | ✅ |
| T14 | None | início da fase 5 | ✅ |
| T15 | T14 | T14 → T15 | ✅ |
| T16 | T15 | T15 → T16 | ✅ |
| T17 | T16 | T16 → T17 | ✅ |

## Test Co-location Validation

| Task | Code Layer Created/Modified | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1 | Config + teste de arquitetura | unit | unit | ✅ |
| T2 | Migração (verificada por IT) | none/integration | integration | ✅ |
| T3 | Backend web (handler) | unit | unit | ✅ |
| T4 | Backend utilitário | unit | unit | ✅ |
| T5 | Backend filtro | unit | unit | ✅ |
| T6 | analytics rotas | integration | integration | ✅ |
| T7 | analytics rotas/middleware | integration | integration | ✅ |
| T8 | analytics lógica | unit | unit | ✅ |
| T9 | Dockerfile | none | none | ✅ |
| T10 | Backend integração externa | integration | integration | ✅ |
| T11 | Frontend componente | unit | unit | ✅ |
| T12 | Frontend utilitário | unit | unit | ✅ |
| T13 | Backend controller | integration | integration | ✅ |
| T14 | Config (Makefile) | none | none | ✅ |
| T15 | Config (compose/Dockerfile) | none | none | ✅ |
| T16 | Fluxo ponta a ponta | e2e | e2e | ✅ |
| T17 | CI | none | none | ✅ |
