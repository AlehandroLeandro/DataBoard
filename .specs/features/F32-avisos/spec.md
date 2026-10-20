# F32 — Avisos Specification

**Prioridade:** V2 · **Depende de:** F04 · **Origem:** F32 do `produto.specs.md`
**Artefatos:** só spec (V2).

## Problem Statement

O admin precisa comunicar manutenções e novidades aos usuários, e o sistema precisa avisar cada usuário sobre eventos próprios (exportação pronta, importação concluída, falha de job). Hoje isso depende de e-mail ou de polling em painéis específicos.

## Goals

- [ ] Sino com indicador de não lidos na barra superior.
- [ ] Notificações de sistema substituem o polling do painel de exportações e importações.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Push no celular | Sem app nativo (§5) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Entrega em tempo real | Server-Sent Events (`/api/v1/notifications/stream`) com fallback de polling de 60 s | Simples, sem broker | y |
| Retenção | Notificações lidas apagadas após 90 dias | Evita crescimento | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Avisos do admin [V2]

**User Story**: Como admin, quero publicar avisos para grupos por um período.

**Why P1**: RF-32.1, RF-32.2.

**Acceptance Criteria**:
1. WHEN o admin publica um aviso com título, texto, período, público (grupos) e nível (info/alerta) THEN os usuários do público SHALL vê-lo no sino durante o período.
2. WHERE "exibir no login" está ativo the aviso SHALL aparecer como banner na tela de login durante o período.
3. WHEN o usuário abre o aviso THEN o sistema SHALL marcá-lo como lido para ele.

**Independent Test**: IT de público por grupo e período.

---

### P1: Notificações do sistema [V2]

**User Story**: Como usuário, quero ser avisado quando minha exportação ou importação termina ou quando meu job falha.

**Why P1**: RF-32.3.

**Acceptance Criteria**:
1. WHEN uma exportação assíncrona fica pronta THEN o sistema SHALL criar uma notificação para o dono com o link.
2. WHEN uma importação termina (sucesso ou falha) THEN o sistema SHALL notificar o autor.
3. WHEN um job de propriedade do usuário falha após as retentativas THEN o sistema SHALL notificá-lo.
4. WHILE o usuário está com a aplicação aberta the frontend SHALL receber novas notificações via SSE sem recarregar.

**Independent Test**: IT com SSE.

---

## Edge Cases

- IF o SSE cai THEN o frontend SHALL reconectar com backoff e usar polling enquanto isso.
- WHEN um aviso expira THEN ele SHALL sumir do sino mesmo sem ter sido lido.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| NOTE-01 | P1: Publicar avisos | RF-32.1 | - | Pending |
| NOTE-02 | P1: Sino e banner de login | RF-32.2 | - | Pending |
| NOTE-03 | P1: Notificações do sistema | RF-32.3 | - | Pending |
| NOTE-04 | P1: Tempo real (SSE) | RF-32.2 | - | Pending |

**Coverage:** 4 total, 0 mapped to tasks (V2).

---

## Success Criteria

- [ ] Notificação de exportação pronta chega em < 5 s após o fim do job.
