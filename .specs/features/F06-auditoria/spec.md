# F06 — Auditoria e Logs Specification

**Prioridade:** MVP (autenticação, acesso, segurança, configurações, dados de UDT, exportação, consulta admin, retenção) / V1 (consulta SQL, metadados com diff e restauração, exportação CSV, jobs) · **Depende de:** F03 · **Origem:** F06 do `produto.specs.md`

## Problem Statement

A empresa precisa saber quem entrou, o que abriu, o que alterou e o que exportou. Isso vale para segurança, para conformidade e para responder "quem mudou essa meta?". Os módulos já publicam eventos de domínio (F02, F03, F04). Falta persistir esses eventos de forma append-only sem atrasar as requisições, e permitir consultá-los.

## Goals

- [ ] Todo evento listado no MVP é gravado em ≤ 2 s após ocorrer, sem aumentar a latência da requisição de origem.
- [ ] Nenhum registro de auditoria pode ser alterado ou apagado fora do job de retenção.
- [ ] Histórico por registro de UDT mostra usuário, data/hora, campo, valor antigo e novo (CA-06.1).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Diff de metadados, aba "Histórico" e restaurar versão (RF-06.3) | V1 |
| Auditoria de execução de consulta (hash do SQL, duração) | V1 |
| Exportação CSV da auditoria | V1 |
| Auditoria de execução de jobs | V1 (F33) |
| Logs técnicos JSON em stdout (RF-06.5) | Já entregue na F00 |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Transporte | Os módulos publicam eventos de domínio; o módulo `audit` os escuta e enfileira em memória; um writer grava em lote a cada 1 s ou a cada 200 eventos | RN-06.3 | y |
| Durabilidade | Eventos na fila podem se perder num crash do processo | Trade-off aceito explicitamente pelo RN-06.3 | y |
| Capacidade da fila | 50.000 eventos; cheia → descarta o novo evento, incrementa a métrica `audit.dropped` e loga erro | Protege a memória sem bloquear requisições | y |
| Append-only | Trigger no schema `audit` rejeita `UPDATE`/`DELETE`, exceto com `SET LOCAL databoard.audit_retention = 'on'` (usado só pelo job) | RN-06.1 com garantia no banco | y |
| Retenção | 365 dias (`audit.retentionDays` em configurações gerais), job diário às 03:00 no fuso da instância | RF-06.4 + AD-015 | y |
| Valores sensíveis | Campos com nome em `password`, `secret`, `token`, `apiKey` (case-insensitive) são substituídos por `"********"` antes de gravar | RN-06.2 | y |
| Histórico de dados de UDT | Tabela própria `audit.data_change`, uma linha por campo alterado | Consulta direta por registro (CA-06.1) | y |
| Exposição do histórico de registro | API Java `AuditQuery.recordHistory(tableId, recordId)`; o endpoint HTTP fica na F09 (`.../records/{id}/history`) | O contrato do endpoint pertence à F09 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Gravação assíncrona e append-only ⭐ MVP

**User Story**: Como admin, quero que a auditoria seja gravada sem atrasar o sistema e sem poder ser adulterada.

**Why P1**: Infra comum de todos os eventos.

**Acceptance Criteria**:
1. WHEN um evento auditável é publicado THEN o módulo de auditoria SHALL gravá-lo em `audit.event` em até 2 s.
2. The publicação de um evento SHALL retornar ao chamador sem esperar a gravação no banco.
3. IF a gravação em lote falhar THEN o sistema SHALL logar o erro, incrementar a métrica `audit.write.failures` e não propagar a falha à requisição de origem.
4. IF qualquer comando tenta `UPDATE` ou `DELETE` em `audit.event` ou `audit.data_change` fora do job de retenção THEN o banco SHALL rejeitar o comando.
5. The sistema SHALL substituir por `"********"` qualquer valor de chave contendo `password`, `secret`, `token` ou `apiKey` antes de gravar.

**Independent Test**: IT publica 1.000 eventos, mede a latência de publicação (< 5 ms p95) e verifica as 1.000 linhas em ≤ 2 s; `UPDATE` direto falha.

---

### P1: Eventos de autenticação, acesso, segurança e exportação ⭐ MVP

**User Story**: Como admin, quero rastrear logins, aberturas de tela, alterações de segurança e exportações.

**Why P1**: Escopo MVP do produto ("login e acesso") + RN-02.2 + RN-29.1.

**Acceptance Criteria**:
1. WHEN ocorre `LoginSucceeded`, `LoginFailed` ou `LoggedOut` THEN o sistema SHALL gravar um evento da categoria `AUTH` com usuário (ou e-mail tentado, no caso de falha), IP e user-agent.
2. WHEN ocorre `ScreenOpened` THEN o sistema SHALL gravar um evento da categoria `ACCESS` com usuário e `resourceId` da tela.
3. WHEN ocorre `SettingsChanged`, `AclChanged` ou alteração de usuário/grupo THEN o sistema SHALL gravar um evento da categoria `SECURITY` com valores antes e depois já mascarados.
4. WHEN ocorre `DataExported` THEN o sistema SHALL gravar um evento da categoria `EXPORT` com formato, número de linhas e recurso de origem.

**Independent Test**: IT publica cada evento e verifica categoria, tipo e detalhes gravados.

---

### P1: Histórico de dados de tabelas personalizadas ⭐ MVP

**User Story**: Como usuário, quero ver quem alterou cada campo de um registro e quando.

**Why P1**: CA-06.1 e RF-10.9 são MVP.

**Acceptance Criteria**:
1. WHEN ocorre `UdtRecordChanged` com operação `INSERT`, `UPDATE` ou `DELETE` THEN o sistema SHALL gravar uma linha em `audit.data_change` por campo alterado, com usuário, data/hora, campo, valor antigo e valor novo.
2. WHEN `AuditQuery.recordHistory(tableId, recordId)` é chamado THEN o sistema SHALL devolver as alterações do registro em ordem decrescente de data.
3. WHEN um `UPDATE` não altera nenhum valor THEN o sistema SHALL não gravar linhas.

**Independent Test**: CA-06.1: alterar um valor e consultar o histórico.

---

### P1: Consulta de auditoria pelo admin ⭐ MVP

**User Story**: Como admin, quero filtrar eventos por período, usuário, tipo e recurso.

**Why P1**: Sem consulta, a auditoria não serve ao admin.

**Acceptance Criteria**:
1. WHEN o admin chama `GET /api/v1/audit/events?from&to&userId&type&resourceType&resourceId` THEN o sistema SHALL devolver os eventos paginados, mais recentes primeiro.
2. IF `from`/`to` não forem informados THEN o sistema SHALL usar os últimos 7 dias.
3. IF um não-admin chama `/api/v1/audit/events` THEN o sistema SHALL responder 403.
4. WHEN o admin abre a tela de Auditoria THEN o frontend SHALL exibir filtros e uma grade com data/hora, usuário, categoria, tipo, recurso e detalhes expansíveis.

**Independent Test**: IT com eventos de vários tipos e filtros combinados.

---

### P1: Retenção ⭐ MVP

**User Story**: Como admin, quero que eventos antigos sejam removidos automaticamente.

**Why P1**: RF-06.4; evita crescimento sem limite.

**Acceptance Criteria**:
1. WHEN o job diário de retenção roda THEN o sistema SHALL apagar eventos e alterações de dados com mais de `audit.retentionDays` dias.
2. WHEN o job conclui THEN o sistema SHALL gravar um evento `SYSTEM/AUDIT_RETENTION` com a quantidade removida.

**Independent Test**: IT com relógio controlado e eventos de 366 e 364 dias.

---

### P2: Auditoria completa [V1]

**User Story**: Como admin, quero auditar execuções de consulta, diffs de metadados (com restauração) e jobs, e exportar a auditoria em CSV.

**Why P2**: V1.

**Acceptance Criteria**:
1. WHEN um metadado é alterado THEN o sistema SHALL gravar o diff JSON e permitir `POST /api/v1/{resourceType}/{id}/history/{versionId}/restore`.
2. WHEN uma consulta é executada THEN o sistema SHALL gravar visão, duração, linhas e hash do SQL.

**Independent Test**: Diferido para a V1.

---

## Edge Cases

- IF a fila atingir 50.000 eventos THEN o sistema SHALL descartar novos eventos, incrementar `audit.dropped` e logar erro uma vez por minuto.
- WHEN a aplicação recebe sinal de desligamento THEN o writer SHALL tentar gravar os eventos pendentes por até 10 s.
- IF `details` exceder 64 KB THEN o sistema SHALL truncar o JSON e marcar `"truncated": true`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| AUD-01 | P1: Gravação assíncrona | RN-06.3 | Tasks | Pending |
| AUD-02 | P1: Append-only | RN-06.1 | Tasks | Pending |
| AUD-03 | P1: Mascaramento | RN-06.2 | Tasks | Pending |
| AUD-04 | P1: Eventos AUTH/ACCESS | RF-06.1 | Tasks | Pending |
| AUD-05 | P1: Eventos SECURITY/EXPORT | RF-06.1, RN-02.2, RN-29.1 | Tasks | Pending |
| AUD-06 | P1: Histórico de dados UDT | RF-06.1, CA-06.1 | Tasks | Pending |
| AUD-07 | P1: Consulta admin | RF-06.2 | Tasks | Pending |
| AUD-08 | P1: Retenção | RF-06.4 | Tasks | Pending |
| AUD-09 | P2: Auditoria completa | RF-06.1, RF-06.3 | - | Pending |

**Coverage:** 9 total, 8 mapped to tasks, 1 diferido (V1).

---

## Success Criteria

- [ ] p95 de publicação de evento < 5 ms.
- [ ] CA-06.1 coberto por teste automatizado.
