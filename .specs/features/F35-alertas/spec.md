# F35 — Alertas e Ações Specification

**Prioridade:** V1 · **Depende de:** F33, F12, F05 · **Origem:** F35 do `produto.specs.md`
**Artefatos:** só spec (V1).

## Problem Statement

Gestores não ficam olhando dashboards o dia todo. Eles precisam ser avisados **antes** do problema: estoque baixo, queda de vendas, meta em risco. Alertas monitoram indicadores por agendamento e disparam e-mail, notificação ou webhook, cada destinatário vendo só os próprios dados.

## Goals

- [ ] "Estoque < 10 por produto" notifica uma vez na mudança de estado e não reenvia enquanto persistir (CA-35.1).
- [ ] Gerentes de SC e PR recebem só as próprias filiais (CA-35.2).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Ação "Executar Processo" | V2 (RF-35.4) |
| WhatsApp nativo | Via webhook (§5) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Avaliação | No Spring, pelo motor (F12) com `QueryContext.user` = destinatário (RLS por destinatário) | RN-35.1 | y |
| Destinatários externos | Avaliados no contexto do dono do alerta, com aviso explícito na UI | RN-35.1 | y |
| Estado por destinatário/grupo | Tabela de estado `(alerta, destinatário, chave do grupo) → OK/ALERTA, último disparo` | RF-35.5 com "por linha" | y |
| Webhook | POST JSON, timeout 10 s, 3 retentativas com backoff exponencial, `X-DataBoard-Signature: sha256=<HMAC>` com segredo por webhook | RN-35.2 (nome do header adaptado a AD-013) | y |
| Proteção SSRF no webhook | Bloquear destinos em redes privadas/loopback por padrão, com allowlist configurável pelo admin | Segurança | y |
| E-mail | Template com tokens (F15) + valores + link para a tela | RF-35.4 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Definir e avaliar alertas [V1]

**User Story**: Como construtor, quero monitorar uma medida com uma condição e avaliá-la periodicamente.

**Why P1**: RF-35.1, RF-35.2.

**Acceptance Criteria**:
1. WHEN um alerta com fonte (visão + medida + filtros, ou objeto de tela), condição (`>`, `<`, `=`, entre, variação % vs. período anterior, "nenhum dado") e agendamento é salvo THEN o sistema SHALL registrar a tarefa na F33.
2. WHEN o alerta é avaliado THEN o sistema SHALL executar a consulta no contexto RLS de cada destinatário usuário.
3. WHERE o modo "por linha" está ativo (ex.: por filial) the sistema SHALL avaliar a condição por grupo e listar no aviso os grupos que violaram.
4. WHEN `POST /api/v1/alerts/{id}/test` é chamado THEN o sistema SHALL avaliar e devolver o resultado sem notificar ninguém.

**Independent Test**: CA-35.2 em IT com dois gerentes sob RLS.

---

### P1: Ações e anti-spam [V1]

**User Story**: Como destinatário, quero ser avisado uma vez quando algo muda, por e-mail, notificação ou webhook.

**Why P1**: RF-35.3, RF-35.4, RF-35.5, RF-35.6.

**Acceptance Criteria**:
1. WHERE "notificar só na mudança de estado" está ativo, WHEN o produto X passa de 12 para 8 THEN o sistema SHALL enviar um e-mail uma vez e não reenviar enquanto o estoque continuar < 10.
2. WHERE "notificar também ALERTA→OK" está ativo the sistema SHALL avisar quando a condição deixar de valer.
3. WHERE há período de silêncio (cooldown) the sistema SHALL não reenviar dentro dele, mesmo no modo "a cada avaliação".
4. WHEN a ação é webhook THEN o sistema SHALL enviar o POST JSON assinado com HMAC, timeout de 10 s e até 3 retentativas com backoff.
5. IF o destino do webhook resolve para rede privada ou loopback e não está na allowlist THEN o sistema SHALL recusar o envio com `WEBHOOK_DESTINATION_BLOCKED`.
6. WHEN um disparo ocorre THEN o sistema SHALL registrar valor avaliado, destinatários e status de envio, visíveis em `GET /api/v1/alerts/{id}/events`.

**Independent Test**: CA-35.1 em IT com relógio controlado; webhook contra servidor de teste verificando a assinatura.

---

## Edge Cases

- IF o SMTP não está configurado THEN a ação de e-mail SHALL ficar desabilitada com explicação.
- WHEN a consulta do alerta falha THEN o sistema SHALL registrar o evento `FAILED` e não alterar o estado.
- IF um destinatário perde o acesso à visão THEN ele SHALL deixar de ser avaliado e notificado.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| ALR-01 | P1: Definição e agendamento | RF-35.1 | - | Pending |
| ALR-02 | P1: RLS por destinatário | RN-35.1, CA-35.2, CA-05.3 | - | Pending |
| ALR-03 | P1: Por linha | RF-35.2 | - | Pending |
| ALR-04 | P1: Destinatários e ações | RF-35.3, RF-35.4 | - | Pending |
| ALR-05 | P1: Anti-spam | RF-35.5, CA-35.1 | - | Pending |
| ALR-06 | P1: Webhook seguro | RN-35.2 | - | Pending |
| ALR-07 | P1: Histórico de disparos e teste | RF-35.6 | - | Pending |

**Coverage:** 7 total, 0 mapped to tasks (V1).

---

## Success Criteria

- [ ] CA-35.1 e CA-35.2 automatizados.
