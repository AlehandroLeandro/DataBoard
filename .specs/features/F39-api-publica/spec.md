# F39 — API Pública Specification

**Prioridade:** V2 · **Depende de:** F04, F12, F09 · **Origem:** F39 do `produto.specs.md`
**Artefatos:** só spec (V2).

## Problem Statement

Empresas querem integrar a plataforma a outros sistemas: consultar dados de visões, alimentar tabelas personalizadas a partir de outros sistemas e automatizar a administração (n8n, scripts). Isso exige tokens de API com escopos, documentação e limites.

## Goals

- [ ] Um script com token `udt:write` alimenta uma tabela personalizada respeitando validações, permissões e auditoria.
- [ ] Documentação OpenAPI navegável na própria instância.

## Out of Scope

| Feature | Reason |
| --- | --- |
| OAuth2 para apps de terceiros | Tokens pessoais e de serviço bastam para PME |
| GraphQL | Fora do produto |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Autenticação | Header `Authorization: Bearer <token>`; token exibido uma vez, guardado como hash (SHA-256 + prefixo identificador) | RF-39.1 | y |
| CSRF | Não se aplica a requisições com Bearer (sem cookie) | Padrão de API | y |
| Rate limit | Bucket4j em memória por token (60 req/min padrão) com `429` e `Retry-After` | RF-39.4; single-node | y |
| Documentação | springdoc-openapi em `/api/docs` (Swagger UI) e `/api/v1/openapi.json` | RF-39.3 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Tokens de API [V2]

**User Story**: Como usuário, quero criar tokens com escopos e validade para integrações.

**Why P1**: RF-39.1, RN-39.1, RN-39.2.

**Acceptance Criteria**:
1. WHEN o usuário cria um token com nome, escopos (`query:read`, `udt:read`, `udt:write`, `admin`) e expiração THEN o sistema SHALL exibi-lo uma única vez e guardar só o hash.
2. WHEN uma requisição usa o token THEN o sistema SHALL aplicar a ACL e a RLS do dono, limitadas pelos escopos.
3. IF o escopo não cobre o endpoint THEN o sistema SHALL responder 403 `TOKEN_SCOPE_INSUFFICIENT`.
4. IF o token expirou ou foi revogado THEN o sistema SHALL responder 401.
5. WHEN o token é usado THEN o sistema SHALL auditar o uso e atualizar o "último uso" exibido na tela de tokens.
6. IF um usuário sem papel ADMIN cria token com escopo `admin` THEN o sistema SHALL responder 403.

**Independent Test**: IT de escopos, expiração e revogação.

---

### P1: Endpoints públicos, documentação e limites [V2]

**User Story**: Como integrador, quero endpoints estáveis documentados e previsíveis.

**Why P1**: RF-39.2, RF-39.3, RF-39.4.

**Acceptance Criteria**:
1. The API pública SHALL expor `POST /api/v1/query`, o CRUD de records de UDT, a listagem de telas e visões e o disparo de cargas e processos, em `/api/v1`.
2. WHEN `/api/docs` é aberto THEN a instância SHALL servir a Swagger UI com o OpenAPI atual.
3. WHEN um token excede o limite (padrão 60 req/min) THEN o sistema SHALL responder 429 com `Retry-After`.
4. WHEN records são escritos via API THEN as validações, permissões, RLS e auditoria da F09 SHALL se aplicar como na UI.

**Independent Test**: IT de rate limit + teste de contrato do OpenAPI.

---

## Edge Cases

- IF um token é enviado também com cookie de sessão THEN o sistema SHALL usar apenas o token.
- WHEN o dono do token é inativado THEN todos os seus tokens SHALL deixar de funcionar.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| API-01 | P1: Tokens com escopo e hash | RF-39.1 | - | Pending |
| API-02 | P1: ACL/RLS do dono limitadas por escopo | RN-39.1 | - | Pending |
| API-03 | P1: Auditoria e último uso | RN-39.2 | - | Pending |
| API-04 | P1: Endpoints públicos | RF-39.2 | - | Pending |
| API-05 | P1: Documentação | RF-39.3 | - | Pending |
| API-06 | P1: Rate limit | RF-39.4 | - | Pending |

**Coverage:** 6 total, 0 mapped to tasks (V2).

---

## Success Criteria

- [ ] Nenhum endpoint aceita token com escopo insuficiente (teste por rota).
