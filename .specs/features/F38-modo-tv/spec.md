# F38 — Modo TV (Kiosk) Specification

**Prioridade:** V2 · **Depende de:** F19 · **Origem:** F38 do `produto.specs.md`
**Artefatos:** só spec (V2).

## Problem Statement

Chão de fábrica, comercial e expedição querem telas rodando em TVs sem ninguém fazer login: qualquer navegador de Smart TV, Chromecast, Raspberry Pi ou mini-PC. O acesso precisa ser seguro, revogável e monitorável.

## Goals

- [ ] Uma URL `/tv/{token}` exibe uma playlist de telas em rotação, em tela cheia, sem login interativo.
- [ ] O admin vê quais TVs estão online e pode recarregá-las remotamente.

## Out of Scope

| Feature | Reason |
| --- | --- |
| App instalável para TV | Baseado em URL (objetivo da feature) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Autenticação | Token opaco de 32 bytes, guardado como hash, troca por sessão técnica de escopo restrito (só leitura das telas da playlist) | RN-38.1 + AD-008 | y |
| Identidade | Usuário técnico escolhido; RLS e ACL como se fosse ele; não pode ser ADMIN | RF-38.3, RN-38.2 | y |
| Comando remoto | SSE do dispositivo para "recarregar agora"; heartbeat a cada 60 s | RF-38.4 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Dispositivos e playlist [V2]

**User Story**: Como admin, quero cadastrar TVs com playlists de telas e gerar a URL de acesso.

**Why P1**: RF-38.1, RF-38.2, RF-38.3.

**Acceptance Criteria**:
1. WHEN o admin cadastra um dispositivo com nome, local, playlist (telas e tempos), filtros fixos, intervalo de atualização, tema e usuário técnico THEN o sistema SHALL gerar a URL `/tv/{token}` exibida uma única vez.
2. WHEN a URL é aberta THEN o frontend SHALL exibir as telas em rotação, em tela cheia, sem menus e sem login.
3. The dados exibidos SHALL respeitar ACL e RLS do usuário técnico.
4. IF o usuário técnico escolhido é ADMIN THEN o sistema SHALL recusar com `TV_USER_CANNOT_BE_ADMIN`.

**Independent Test**: E2E abrindo a URL num contexto sem cookies.

---

### P1: Gestão e resiliência [V2]

**User Story**: Como admin, quero ver TVs online, revogar acessos e recarregar remotamente; e como operador quero que a TV se recupere sozinha.

**Why P1**: RF-38.4, RF-38.5, RN-38.1.

**Acceptance Criteria**:
1. WHEN o admin abre o portal THEN o sistema SHALL listar dispositivos com último heartbeat (online se < 2 min), IP e versão.
2. WHEN o admin revoga ou regenera o token THEN a URL antiga SHALL deixar de funcionar na próxima requisição.
3. WHEN o admin aciona "recarregar agora" THEN o dispositivo SHALL recarregar em até 10 s.
4. WHERE há restrição por faixa de IP the acesso de fora da faixa SHALL ser negado.
5. IF o servidor fica indisponível THEN a TV SHALL exibir uma tela de erro amigável e tentar novamente a cada 30 s.
6. The acessos de dispositivos SHALL ser auditados.

**Independent Test**: IT de revogação e de restrição de IP.

---

## Edge Cases

- IF uma tela da playlist é excluída ou o usuário técnico perde acesso THEN a rotação SHALL pulá-la e registrar aviso no portal.
- WHEN o token é usado de dois IPs simultaneamente THEN o portal SHALL mostrar alerta de possível vazamento.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| TV-01 | P1: Dispositivos e playlist | RF-38.1 | - | Pending |
| TV-02 | P1: URL com token e kiosk | RF-38.2, RN-38.1 | - | Pending |
| TV-03 | P1: Usuário técnico | RF-38.3, RN-38.2 | - | Pending |
| TV-04 | P1: Portal, revogação, recarregar | RF-38.4 | - | Pending |
| TV-05 | P1: Resiliência | RF-38.5 | - | Pending |
| TV-06 | P1: Restrição de IP e auditoria | RN-38.1 | - | Pending |

**Coverage:** 6 total, 0 mapped to tasks (V2).

---

## Success Criteria

- [ ] Token revogado bloqueia a TV em < 10 s.
