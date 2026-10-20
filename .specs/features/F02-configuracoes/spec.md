# F02 — Configurações Gerais e Identidade Visual Specification

**Prioridade:** MVP (geral, segurança, limites, SMTP, tema) / V1 (white-label) · **Depende de:** F01, F03, F04 · **Origem:** F02 do `produto.specs.md`

## Problem Statement

Os parâmetros da instância (empresa, fuso, formatos, política de segurança, limites de consulta, SMTP) hoje só existem como padrões no código ou como o que o wizard gravou. O admin precisa ajustá-los pela interface, e as mudanças precisam ser rastreáveis.

## Goals

- [ ] Admin altera cada seção de configuração pela UI sem reiniciar o sistema.
- [ ] Toda alteração gera evento com valor anterior e novo, com segredos mascarados.
- [ ] SMTP pode ser testado com um clique.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Idioma padrão (parte do RF-02.1) | AD-014: somente pt-BR |
| White-label: logo, favicon, cor primária, título da aba (RF-02.2) | V1 (registrado como P2 abaixo) |
| Aplicação dos limites nas consultas (truncamento) | F12 (motor de consulta) consome `limits` |
| Persistência do evento de auditoria | F06 |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Seções | `general`, `security`, `limits`, `smtp`, `appearance` (tema) e `branding` (V1) | Contrato `PUT /settings` por seção | y |
| Formato da API | `GET /settings` devolve todas as seções; `PUT /settings/{section}` substitui a seção inteira | "Parcial por seção" do contrato original, com rota explícita | y |
| Padrões de `general` | Moeda `BRL`, data `dd/MM/yyyy`, número `#.##0,00`, tamanho de página 50 | Padrão pt-BR | y |
| Padrões de `limits` | Timeout de query 60 s, máx. linhas por consulta 100.000, máx. linhas exportadas 100.000 | RF-02.5 | y |
| Faixas válidas | Timeout 5–600 s; linhas 100–1.000.000; tamanho de página 10–500; ociosidade 5–480 min; duração máxima 1–72 h; tentativas 3–20; bloqueio 1–1440 min; senha mínima 8–128 | Evita configurações que derrubam a instância | y |
| Tema | Admin define o padrão (`light`/`dark`/`auto`); usuário sobrescreve em suas preferências | RF-02.3 | y |
| Senha SMTP em `GET` | Devolvida como `"********"`; `PUT` com campo vazio ou `"********"` mantém a atual | Mesmo padrão do RN-07.3 | y |
| Propagação | Leitura direta do `SettingsStore` por requisição (sem cache) | Mudança vale imediatamente; volume de leitura baixo | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Editar configurações por seção ⭐ MVP

**User Story**: Como admin, quero ajustar empresa, formatos, segurança e limites pela interface para adaptar a plataforma sem mexer em arquivos.

**Why P1**: F03 e F12 consomem esses parâmetros.

**Acceptance Criteria**:
1. WHEN o admin chama `GET /api/v1/settings` THEN o sistema SHALL devolver as seções `general`, `security`, `limits`, `smtp` e `appearance`, preenchendo com padrões o que não foi gravado.
2. WHEN o admin chama `PUT /api/v1/settings/{section}` com valores válidos THEN o sistema SHALL gravar a seção e responder 200 com a seção atualizada.
3. IF um valor está fora da faixa válida THEN o sistema SHALL responder 400 `VALIDATION_ERROR` com erro no campo.
4. IF um não-admin chama `PUT /api/v1/settings/{section}` THEN o sistema SHALL responder 403.
5. WHEN a seção `security` é alterada THEN a próxima tentativa de login SHALL usar os novos parâmetros, sem reinício.
6. WHEN qualquer seção é alterada THEN o sistema SHALL publicar `SettingsChanged { section, before, after, userId }` com campos secretos substituídos por `"********"`.

**Independent Test**: IT: alterar `security.lockoutAttempts` para 3 e ver a conta bloquear na 3ª falha.

---

### P1: SMTP com e-mail de teste ⭐ MVP

**User Story**: Como admin, quero configurar e testar o SMTP para que os e-mails do sistema funcionem.

**Why P1**: Reset de senha (F03) depende dele.

**Acceptance Criteria**:
1. WHEN o admin grava a seção `smtp` THEN o sistema SHALL armazenar a senha cifrada (`CryptoService`).
2. The `GET /api/v1/settings` SHALL devolver a senha SMTP como `"********"`.
3. WHEN `PUT /settings/smtp` recebe a senha vazia ou `"********"` THEN o sistema SHALL manter a senha armazenada.
4. WHEN o admin chama `POST /api/v1/settings/smtp/test` com `{ to }` THEN o sistema SHALL enviar um e-mail de teste e responder 200 `{ delivered: true }`.
5. IF o servidor SMTP recusar ou não responder em 15 s THEN o sistema SHALL responder 422 com `code=SMTP_TEST_FAILED` e a mensagem do servidor em `detail`.

**Independent Test**: IT com GreenMail: enviar teste e receber a mensagem.

---

### P1: Tema claro/escuro ⭐ MVP

**User Story**: Como usuário, quero escolher tema claro ou escuro, e como admin quero definir o padrão.

**Why P1**: Custo baixo com Mantine e pedido pelo produto.

**Acceptance Criteria**:
1. WHEN o usuário não tem preferência de tema THEN o frontend SHALL aplicar `appearance.defaultTheme`.
2. WHEN o usuário escolhe um tema no menu do perfil THEN o sistema SHALL gravar `preferences.theme` e o frontend SHALL aplicá-lo imediatamente.
3. WHERE o tema é `auto` the frontend SHALL seguir `prefers-color-scheme` do navegador.

**Independent Test**: Teste de componente alternando tema e verificando o `data-mantine-color-scheme`.

---

### P2: White-label [V1]

**User Story**: Como admin, quero aplicar logo, favicon, cor primária e nome da aba da minha empresa.

**Why P2**: V1.

**Acceptance Criteria**:
1. WHEN o admin altera a cor primária THEN todos os usuários SHALL ver a nova cor no próximo carregamento, sem rebuild.
2. WHEN o admin envia um logo por `POST /api/v1/settings/branding/logo` THEN o sistema SHALL validar o MIME real (PNG, SVG, WEBP; até 1 MB) e servi-lo no cabeçalho.

**Independent Test**: Diferido para a V1.

---

## Edge Cases

- IF o `PUT` contém campos desconhecidos THEN o sistema SHALL responder 400 (`FAIL_ON_UNKNOWN_PROPERTIES`).
- IF o `PUT` em `general` traz um fuso inválido THEN o sistema SHALL responder 400 em `timezone`.
- WHEN dois admins salvam a mesma seção simultaneamente THEN a última gravação SHALL prevalecer, e ambas SHALL gerar evento.
- IF o e-mail de destino do teste é inválido THEN o sistema SHALL responder 400 em `to`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| CFG-01 | P1: Seções (general/limits) | RF-02.1, RF-02.5 | Tasks | Pending |
| CFG-02 | P1: Seções (security) | RF-02.4 | Tasks | Pending |
| CFG-03 | P1: Seções (permissão e evento) | RN-02.1, RN-02.2 | Tasks | Pending |
| CFG-04 | P1: SMTP | RF-02.6 | Tasks | Pending |
| CFG-05 | P1: Tema | RF-02.3 | Tasks | Pending |
| CFG-06 | P2: White-label | RF-02.2, CA-02.1 | - | Pending |

**Coverage:** 6 total, 5 mapped to tasks, 1 diferido (V1).

---

## Success Criteria

- [ ] Nenhum parâmetro do RF-02.4/RF-02.5 exige reinício para valer.
- [ ] Segredos nunca aparecem em respostas nem em eventos.
