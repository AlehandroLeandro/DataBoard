# F34 — Processos (Rotinas SQL) Specification

**Prioridade:** V1 · **Depende de:** F33, F08 · **Origem:** F34 do `produto.specs.md`
**Artefatos:** só spec (V1).

## Problem Statement

Depois das cargas, é comum consolidar tabelas do lake, popular tabelas personalizadas e pré-calcular agregados pesados. Os Processos são o "dbt leve" da plataforma: scripts SQL transacionais na conexão interna, agendados e encadeados.

## Goals

- [ ] Encadear "carga do ERP → processo de consolidação → invalidação de cache" sem intervenção manual.
- [ ] Um processo que falha não deixa escrita parcial (RF-34.4).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Processos em conexões externas | Escrita só nos schemas `lake` e `udt` internos (RF-34.1) |
| Linguagens além de SQL (Python) | Fora do produto; o analytics é stateless e não acessa banco (AD-002) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Role de execução | Role `databoard_proc` com escrita em `lake` e `udt` (sem DDL em `udt`), sem acesso a `app`/`audit` | RF-34.1, RN-17.2 | y |
| Escrita em `udt` | Bypass de validação (RN-09.6), restrito a ADMIN; registra auditoria resumida e dispara `UdtRecordChanged` agregado para invalidar cache | RN-34.2 | y |
| Transação | Um processo roda numa transação única; qualquer erro faz rollback de todos os comandos | RF-34.4 | y |
| "Materializar visão" | Executa a visão pelo motor (com contexto técnico, sem RLS de usuário) e grava em `lake.<slug>` com FULL/INCREMENTAL, como na F08 | RF-34.2 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Processos SQL [V1]

**User Story**: Como admin, quero escrever scripts SQL que transformam dados no lake e nas tabelas personalizadas.

**Why P1**: RF-34.1, RF-34.4.

**Acceptance Criteria**:
1. WHEN um ADMIN salva um processo SQL livre com um ou mais comandos THEN o sistema SHALL validar que todos os objetos referenciados estão em `lake` ou `udt`.
2. WHEN o processo executa THEN o sistema SHALL rodar todos os comandos numa transação e registrar as linhas afetadas por comando.
3. IF qualquer comando falha THEN o sistema SHALL fazer rollback de todos e marcar a execução como `FAILED` com o comando e o erro.
4. IF o script tenta acessar `app` ou `audit` THEN o banco SHALL negar a operação e a execução SHALL falhar.
5. IF um não-ADMIN tenta criar ou editar processo SQL livre THEN o sistema SHALL responder 403.

**Independent Test**: IT com script de 3 comandos, um falhando.

---

### P1: Materializar visão e encadeamento [V1]

**User Story**: Como admin, quero materializar visões pesadas e encadear processos após cargas.

**Why P1**: RF-34.2, RF-34.3.

**Acceptance Criteria**:
1. WHEN um processo "Materializar visão" executa THEN o sistema SHALL gravar o resultado da visão em `lake.<slug>` no modo configurado.
2. WHEN a carga X conclui e o processo P está configurado "após X" THEN o sistema SHALL executar P (F33).
3. WHEN um processo grava em `udt` ou `lake` THEN o sistema SHALL invalidar o cache das visões dependentes e registrar auditoria.

**Independent Test**: IT de encadeamento carga → processo → cache invalidado.

---

## Edge Cases

- IF o script tem mais de 1 MB THEN o sistema SHALL rejeitar com `PROCESS_TOO_LARGE`.
- WHEN o processo excede o timeout configurado (padrão 30 min) THEN o sistema SHALL cancelar e fazer rollback.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| PROC-01 | P1: SQL livre restrito a lake/udt | RF-34.1, RN-34.1 | - | Pending |
| PROC-02 | P1: Transacional com log por comando | RF-34.4 | - | Pending |
| PROC-03 | P1: Materializar visão | RF-34.2 | - | Pending |
| PROC-04 | P1: Encadeamento | RF-34.3 | - | Pending |
| PROC-05 | P1: Bypass auditado + invalidação | RN-34.2 | - | Pending |

**Coverage:** 5 total, 0 mapped to tasks (V1).

---

## Success Criteria

- [ ] Nenhum processo consegue ler ou escrever no schema `app` (teste de permissão).
