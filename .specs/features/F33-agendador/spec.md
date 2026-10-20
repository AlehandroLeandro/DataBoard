# F33 — Agendador de Tarefas Specification

**Prioridade:** V1 (no MVP só jobs internos com `@Scheduled`, AD-015) · **Depende de:** F01 · **Origem:** F33 do `produto.specs.md`
**Artefatos:** só spec (V1).

## Problem Statement

ETL, processos, alertas e envio de relatórios precisam de um motor central de execução agendada e assíncrona, que sobreviva a reinícios, mostre o que está agendado e o que rodou, e não compita com as requisições web.

## Goals

- [ ] Agendamentos sobrevivem a reinícios; execuções perdidas seguem a política de misfire (RN-33.1).
- [ ] Telas "Agendamentos" e "Tarefas Executadas" com ações de pausar, executar agora e cancelar.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Execução distribuída em vários nós | Single-node (§F01 fora de escopo) |
| Filas externas (Redis/RabbitMQ) | Quartz com JobStore JDBC no Postgres (§2) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Motor | Quartz com JobStore JDBC no Postgres (schema `app`) | RN-33.1 | y |
| Migração do MVP | Jobs `@Scheduled` do MVP (lixeiras, retenção, limpezas) passam a ser tarefas de sistema no Quartz, mantendo `app.job_run` como histórico | AD-015 | y |
| Concorrência | Pool de 4 threads (configurável) e `@DisallowConcurrentExecution` por chave de tarefa | RN-33.2, RN-33.3 | y |
| Fuso | Fuso da instância (F02) | RF-33.5 | y |
| Cancelamento | Cooperativo: jobs verificam a flag de interrupção entre lotes; consultas em andamento recebem `Statement.cancel()` | RF-33.3 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Agendar tarefas [V1]

**User Story**: Como admin ou dono de uma tarefa, quero agendá-la por cron amigável, por evento ou manualmente.

**Why P1**: RF-33.1.

**Acceptance Criteria**:
1. WHEN uma tarefa é agendada com "a cada N minutos/horas", "diário às", "dias da semana" ou "dia do mês" THEN o sistema SHALL convertê-la para expressão cron e calcular a próxima execução no fuso da instância.
2. WHERE o modo avançado é usado the sistema SHALL aceitar expressão cron válida do Quartz e rejeitar inválida com `INVALID_CRON`.
3. WHEN uma tarefa é configurada "após a carga X concluir" THEN o sistema SHALL dispará-la ao fim com sucesso da carga X.
4. WHEN o app reinicia com execuções perdidas THEN o sistema SHALL executá-las uma vez ao subir (misfire padrão).

**Independent Test**: IT com Quartz real e relógio controlado.

---

### P1: Acompanhar agendamentos e execuções [V1]

**User Story**: Como admin, quero ver o que está agendado, o que rodou e agir sobre as tarefas.

**Why P1**: RF-33.2, RF-33.3.

**Acceptance Criteria**:
1. WHEN `GET /api/v1/scheduler/tasks` é chamado THEN o sistema SHALL listar tipo, dono, próxima e última execução, status e ações.
2. WHEN `PATCH /api/v1/scheduler/tasks/{id}` muda `enabled` ou `schedule` THEN o sistema SHALL pausar, retomar ou reagendar.
3. WHEN `POST /api/v1/scheduler/tasks/{id}/run` é chamado THEN o sistema SHALL executar a tarefa imediatamente, salvo se já estiver em execução (`TASK_ALREADY_RUNNING`).
4. WHEN `GET /api/v1/scheduler/runs` é chamado com filtros THEN o sistema SHALL listar início, fim, duração, status (`SUCCESS`, `FAILED`, `RUNNING`, `CANCELLED`), log resumido e erro.
5. WHEN `POST /api/v1/scheduler/runs/{runId}/cancel` é chamado numa execução `RUNNING` THEN o sistema SHALL interrompê-la e marcá-la `CANCELLED`.

**Independent Test**: IT das rotas + teste de componente das telas.

---

### P1: Política de falha [V1]

**User Story**: Como dono de uma tarefa, quero retentativas e aviso quando ela falha.

**Why P1**: RF-33.4.

**Acceptance Criteria**:
1. WHEN uma tarefa falha e tem N retentativas com intervalo I THEN o sistema SHALL reexecutá-la até N vezes com intervalo I.
2. WHEN as retentativas se esgotam THEN o sistema SHALL notificar o dono e os e-mails configurados (com SMTP), e gravar o evento de auditoria.

**Independent Test**: IT com job que falha 2 vezes e passa na 3ª.

---

## Edge Cases

- IF o pool está ocupado THEN novas execuções SHALL aguardar na fila do Quartz, sem rejeição.
- WHEN uma tarefa é excluída durante a execução THEN a execução atual SHALL terminar e nenhuma nova SHALL ser agendada.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| SCH-01 | P1: Cron amigável e avançado | RF-33.1, RF-33.5 | - | Pending |
| SCH-02 | P1: Disparo por evento | RF-33.1 | - | Pending |
| SCH-03 | P1: Persistência e misfire | RN-33.1 | - | Pending |
| SCH-04 | P1: Agendamentos | RF-33.2 | - | Pending |
| SCH-05 | P1: Tarefas executadas e cancelamento | RF-33.3 | - | Pending |
| SCH-06 | P1: Concorrência | RN-33.2, RN-33.3 | - | Pending |
| SCH-07 | P1: Retentativas e notificação | RF-33.4 | - | Pending |

**Coverage:** 7 total, 0 mapped to tasks (V1).

---

## Success Criteria

- [ ] Reinício do app não perde agendamentos (teste automatizado).
