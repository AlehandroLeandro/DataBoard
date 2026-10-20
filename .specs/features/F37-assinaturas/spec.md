# F37 — Agendamento e Envio de Relatórios (Assinaturas) Specification

**Prioridade:** V1 (tela e objeto) / V2 (Report da F36) · **Depende de:** F33, F29 · **Origem:** F37 do `produto.specs.md`
**Artefatos:** só spec (V1).

## Problem Statement

Muitos gestores preferem receber o painel por e-mail toda segunda às 8h a abrir o sistema. Assinaturas geram o conteúdo (tela em PDF/PNG, objeto em XLSX/CSV/PNG) com filtros fixos ou relativos e enviam a cada destinatário apenas os dados dele.

## Goals

- [ ] Assinatura semanal de uma tela em PDF entregue a Ana e Bruno, cada um com seus dados (CA-05.3).
- [ ] O próprio usuário assina uma tela para si em 3 cliques (RF-37.3).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Conteúdo do tipo Report | V2 (depende da F36) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Geração | PDF/PNG de tela pelo analytics com Playwright/Chromium (F29 V1); XLSX/CSV pelo pipeline da F29 | AD-011 | y |
| Render autenticado | O Spring emite um token de uso único (5 min) que o Chromium usa para abrir a tela no contexto do destinatário | RLS por destinatário sem compartilhar sessão | y |
| Reaproveitamento | Destinatários com o mesmo "contexto efetivo de RLS" (mesmo conjunto de predicados e binds) recebem o mesmo arquivo, gerado uma vez | RN-37.1 | y |
| Filtros | Valores fixos ou relativos (F28: "mês anterior") resolvidos na hora do envio | RF-37.1 | y |
| Webhook | Arquivo como URL temporária assinada (24 h), mesma proteção SSRF da F35 | RF-37.4 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Assinar tela ou objeto [V1]

**User Story**: Como usuário, quero receber uma tela ou objeto por e-mail periodicamente.

**Why P1**: RF-37.1, RF-37.3.

**Acceptance Criteria**:
1. WHEN uma assinatura com conteúdo (tela PDF/PNG ou objeto XLSX/CSV/PNG), filtros, destinatários, agendamento, assunto e corpo é salva THEN o sistema SHALL registrar a tarefa na F33.
2. WHEN o usuário escolhe "Receber por e-mail" numa tela THEN o sistema SHALL criar uma assinatura com ele como único destinatário e o agendamento escolhido.
3. WHEN a assinatura é executada THEN o sistema SHALL gerar o conteúdo no contexto RLS de cada destinatário e enviar o e-mail com o anexo.
4. WHEN Ana e Bruno recebem a mesma assinatura com RLS por vendedor THEN cada um SHALL receber apenas os próprios dados (CA-05.3).
5. WHEN dois destinatários têm o mesmo contexto efetivo de RLS THEN o sistema SHALL gerar o arquivo uma única vez e reaproveitá-lo.

**Independent Test**: IT com GreenMail e dois destinatários.

---

### P1: Regras de envio [V1]

**User Story**: Como dono da assinatura, quero não enviar vazio e usar webhook.

**Why P1**: RF-37.2, RF-37.4, RN-37.2.

**Acceptance Criteria**:
1. WHERE "não enviar se vazio" está ativo, WHEN o conteúdo de um destinatário não tem linhas THEN o sistema SHALL não enviar a esse destinatário.
2. WHERE o canal é webhook the sistema SHALL enviar a URL temporária assinada do arquivo com validade de 24 h.
3. IF o SMTP não está configurado THEN a opção de e-mail SHALL ficar desabilitada com a explicação.

**Independent Test**: IT de "não enviar se vazio" e de webhook.

---

## Edge Cases

- IF a geração do PDF falha para um destinatário THEN o sistema SHALL registrar a falha por destinatário e continuar com os demais.
- WHEN um destinatário perde o acesso à tela THEN ele SHALL ser ignorado no envio, com registro no histórico.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| SUB-01 | P1: Assinatura de tela/objeto | RF-37.1 | - | Pending |
| SUB-02 | P1: Autoassinatura | RF-37.3 | - | Pending |
| SUB-03 | P1: RLS por destinatário e reaproveitamento | RN-37.1, CA-05.3 | - | Pending |
| SUB-04 | P1: Não enviar se vazio | RF-37.2 | - | Pending |
| SUB-05 | P1: Webhook com URL assinada | RF-37.4 | - | Pending |
| SUB-06 | P1: Dependência de SMTP | RN-37.2 | - | Pending |
| SUB-07 | (V2) Conteúdo Report | RF-37.1 | - | Pending |

**Coverage:** 7 total, 0 mapped to tasks (V1).

---

## Success Criteria

- [ ] CA-05.3 automatizado.
