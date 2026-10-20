# F00 — Fundação Specification

**Prioridade:** MVP · **Depende de:** — · **Origem:** seção 2 do `produto.specs.md` + AD-001…AD-017

## Problem Statement

Nenhuma feature pode começar sem o esqueleto das três partes (Spring, Python, React), o contrato entre elas, o banco com os schemas base e os gates de teste automatizados. A F00 entrega esse chão comum, para que cada feature seguinte só adicione comportamento.

## Goals

- [ ] `make dev` sobe Postgres + analytics + app localmente, e `GET /actuator/health` responde `UP` em ≤ 60 s.
- [ ] `make ci` roda lint, typecheck, build e testes das três partes, e passa num repositório recém-clonado.
- [ ] O Spring chama o analytics com token interno e trata indisponibilidade com erro padronizado.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Wizard de setup, variáveis `DATABOARD_*` obrigatórias, imagem publicada | F01 |
| Autenticação | F03 |
| Qualquer transformação real de dados no analytics | F12/F19–F22 (aqui só um endpoint `echo` de contrato) |
| Kubernetes/Helm | V2 (fora do produto) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Estrutura do repositório | `backend/`, `frontend/`, `analytics/`, `deploy/`, `Makefile` na raiz | AD-013 | y |
| Build do backend | Maven Wrapper (`backend/mvnw`), Java 25, Spring Boot 4.x | AD-012 | y |
| Gerenciador Python | `uv` com `pyproject.toml` e `uv.lock` | AD-012 | y |
| Gerenciador JS | npm com `package-lock.json` | Padrão do Vite; sem necessidade de pnpm | y |
| Servir a SPA | Build do Vite copiado para `backend/src/main/resources/static` no build Maven (`frontend-maven-plugin`); fallback de rotas SPA para `index.html` | Uma origem só (AD-008) | y |
| Timeout Spring→analytics | 10 s, sem retry automático | Transformações são CPU-bound e rápidas; um retry cego dobra a carga | y |
| Correlação de logs | Header `X-Request-Id` gerado no Spring e propagado ao analytics | Rastreio ponta a ponta nos logs JSON | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Ambiente local em um comando ⭐ MVP

**User Story**: Como desenvolvedor, quero subir todo o ambiente com um comando para começar a contribuir sem configurar nada à mão.

**Why P1**: Pré-requisito de todas as features.

**Acceptance Criteria**:
1. WHEN o desenvolvedor executa `make dev` num clone limpo THEN o sistema SHALL subir os serviços `postgres`, `analytics` e `app`, e `GET http://localhost:8080/actuator/health` SHALL responder 200 com `status=UP` em até 60 s.
2. WHEN o app sobe THEN o Flyway SHALL criar os schemas `app`, `udt`, `lake` e `audit` se não existirem.
3. WHEN o desenvolvedor executa `make dev-front` THEN o Vite SHALL servir o frontend com proxy de `/api` e `/actuator` para `localhost:8080`.
4. The app SHALL servir a SPA buildada na raiz `/` e devolver `index.html` para qualquer rota não-API desconhecida (fallback de SPA).

**Independent Test**: Clonar, `make dev`, abrir `http://localhost:8080` e ver a página inicial placeholder.

---

### P1: Contrato Spring ↔ analytics ⭐ MVP

**User Story**: Como desenvolvedor, quero um cliente tipado e um contrato versionado entre Spring e Python para que mudanças quebrem nos testes, não em produção.

**Why P1**: AD-003/AD-004 tornam o analytics parte do caminho de todo objeto.

**Acceptance Criteria**:
1. WHEN o analytics recebe uma requisição em `/v1/*` sem o header `X-Internal-Token` correto THEN o analytics SHALL responder 401 sem processar o corpo.
2. WHEN o Spring chama `POST /v1/echo` com token válido THEN o analytics SHALL devolver o mesmo payload e o header `X-Request-Id` recebido.
3. IF o analytics não responder em 10 s ou recusar a conexão THEN o Spring SHALL responder ao cliente 503 `application/problem+json` com `code=ANALYTICS_UNAVAILABLE`.
4. WHEN `make contract` roda THEN o sistema SHALL regenerar `analytics/openapi.json` e falhar se o arquivo versionado divergir do gerado.
5. The analytics SHALL expor `GET /health` sem token, usado pelo healthcheck do compose.

**Independent Test**: Teste de integração no Spring contra um analytics real em Testcontainers: echo ok, token errado → 401, container parado → 503.

---

### P1: Padrões transversais de API ⭐ MVP

**User Story**: Como consumidor da API, quero erros e paginação com formato estável para tratar respostas de forma genérica no front.

**Why P1**: Convenções da seção 1 do produto valem para todos os endpoints.

**Acceptance Criteria**:
1. IF qualquer endpoint `/api/v1/**` lança uma exceção de domínio THEN o sistema SHALL responder `application/problem+json` com os campos `type`, `title`, `status`, `detail` e `code`.
2. IF a validação de um corpo de requisição falha THEN o sistema SHALL responder 400 com `code=VALIDATION_ERROR` e `errors: [{ field, code, message }]`.
3. WHEN um endpoint paginado recebe `?page=0&size=50&sort=campo,asc` THEN o sistema SHALL responder `{ content, page, size, totalElements }`.
4. IF uma exceção não tratada ocorre THEN o sistema SHALL responder 500 com `code=INTERNAL_ERROR`, sem stack trace no corpo, e logar o erro com o `requestId`.
5. The app e o analytics SHALL emitir logs em stdout no formato JSON, contendo `timestamp`, `level`, `logger`, `message` e `requestId`.

**Independent Test**: Controller de teste que lança exceções conhecidas; asserts sobre o JSON de resposta e sobre o log capturado.

---

### P1: Gates de qualidade e CI ⭐ MVP

**User Story**: Como mantenedor, quero que cada PR rode os mesmos gates que rodo localmente.

**Why P1**: O fluxo tlc-spec-driven exige gates determinísticos por task.

**Acceptance Criteria**:
1. The `Makefile` SHALL expor os alvos `test-backend`, `it-backend`, `test-analytics`, `test-frontend`, `lint`, `build`, `e2e`, `contract` e `ci`.
2. WHEN um PR é aberto THEN o GitHub Actions SHALL executar `make ci` e marcar o PR como falho se qualquer alvo falhar.
3. WHEN um módulo Spring acessa uma classe interna de outro módulo THEN o teste de verificação do Spring Modulith SHALL falhar.

**Independent Test**: Introduzir uma violação de módulo num branch e ver `make test-backend` falhar.

---

## Edge Cases

- IF a porta 5432 ou 8080 estiver ocupada THEN `make dev` SHALL falhar com a mensagem do Docker indicando a porta (sem retry silencioso).
- IF o analytics responder 4xx/5xx com corpo de erro THEN o Spring SHALL repassar `code=ANALYTICS_ERROR` com status 502 e logar o corpo original.
- WHEN o analytics recebe um payload acima de 20 MB THEN o analytics SHALL responder 413.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| FND-01 | P1: Ambiente local | §2 Arquitetura | Tasks | Pending |
| FND-02 | P1: Ambiente local (schemas Flyway) | §2, RF-01.6 | Tasks | Pending |
| FND-03 | P1: Ambiente local (SPA servida) | §2 | Tasks | Pending |
| FND-04 | P1: Contrato (token interno) | AD-002 | Tasks | Pending |
| FND-05 | P1: Contrato (cliente + 503) | AD-004 | Tasks | Pending |
| FND-06 | P1: Contrato (OpenAPI versionado) | AD-004 | Tasks | Pending |
| FND-07 | P1: Padrões (problem+json) | §1 Erros | Tasks | Pending |
| FND-08 | P1: Padrões (paginação) | §1 Paginação | Tasks | Pending |
| FND-09 | P1: Padrões (logs JSON) | RF-06.5 | Tasks | Pending |
| FND-10 | P1: Gates (Makefile + CI) | AD-017 | Tasks | Pending |
| FND-11 | P1: Gates (Modulith) | AD-012 | Tasks | Pending |

**Coverage:** 11 total, 11 mapped to tasks, 0 unmapped.

---

## Success Criteria

- [ ] Clone limpo → `make dev` → health `UP` em ≤ 60 s.
- [ ] `make ci` verde no GitHub Actions no primeiro PR.
