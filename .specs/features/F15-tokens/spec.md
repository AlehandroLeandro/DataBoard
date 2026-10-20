# F15 — Tokens (Variáveis) Specification

**Prioridade:** MVP (tokens de sistema) / V1 (tokens customizados) · **Depende de:** F03, F04, F12 · **Origem:** F15 do `produto.specs.md`

## Problem Statement

Visões, RLS, filtros e títulos precisam de valores que só existem na hora da execução: quem é o usuário, quais são seus atributos, qual é o início do mês. Sem tokens, cada tela precisaria de SQL fixo por usuário ou por data. Os tokens precisam virar **bind parameters tipados** em SQL (nunca concatenação) e texto formatado em títulos.

## Goals

- [ ] `{{usuario.atributo.vendedor_id}}` e `{{inicio_mes}}` funcionam em SQL de visão e em predicados (base da F05).
- [ ] Nenhum token é concatenado em SQL; todos viram binds.
- [ ] Token desconhecido é rejeitado no salvamento.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Tokens customizados constante/consulta (RF-15.2) | V1 |
| Uso em alertas e relatórios | F35/F36/F37 consomem o mesmo `TokenService` |
| Valores padrão de filtro por token | F28 consome o `TokenService` |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Fuso para tokens de data | `general.timezone` da instância (F02) | Datas "de negócio" | y |
| Tipos | `usuario.id` UUID; `usuario.email`/`usuario.nome`/`empresa.nome` TEXT; `usuario.grupos` lista de TEXT (nomes); `usuario.atributo.<chave>` lista de TEXT (1..n valores); tokens de data DATE (`agora` DATETIME); `ano_atual` INTEGER | RF-15.1 | y |
| Multivalor em SQL | Expandido para `?, ?, ?` (um bind por valor); lista vazia vira `NULL` (de modo que `IN (NULL)` é falso) | `filial IN ({{usuario.atributo.filiais}})` (F05) | y |
| Atributo com um valor em comparação escalar (`=`) | Usa o primeiro (único) valor; com 2+ valores → erro `TOKEN_MULTIVALUE_IN_SCALAR` em execução | Evita resultado ambíguo | y |
| Token sem valor em SQL de visão | Vira bind `NULL` (comparação falsa) | RN-15.1 → RN-05.3 | y |
| Tokens dentro de literais SQL | Não são substituídos (`'{{hoje}}'` continua literal) | Evita surpresa e injeção | y |
| Interpolação em texto | No backend (`TokenService.interpolateText`), datas `dd/MM/yyyy`, listas separadas por vírgula | F19 resolve títulos no servidor | y |
| `mes_anterior.fim` | Último dia do mês anterior (DATE) | RF-15.1 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Tokens de sistema em SQL de visão ⭐ MVP

**User Story**: Como construtor, quero usar `{{usuario.atributo.x}}` e tokens de data no SQL de visões para que cada usuário veja dados do seu contexto.

**Why P1**: Base da RLS (F05) e de filtros dinâmicos.

**Acceptance Criteria**:
1. The sistema SHALL resolver os tokens `{{usuario.id}}`, `{{usuario.email}}`, `{{usuario.nome}}`, `{{usuario.grupos}}`, `{{usuario.atributo.<chave>}}`, `{{hoje}}`, `{{agora}}`, `{{inicio_mes}}`, `{{fim_mes}}`, `{{inicio_ano}}`, `{{mes_anterior.inicio}}`, `{{mes_anterior.fim}}`, `{{ano_atual}}` e `{{empresa.nome}}`.
2. WHEN o motor executa uma visão SQL contendo tokens THEN o sistema SHALL substituir cada token por bind parameter(s) tipado(s), sem concatenar valores no SQL.
3. WHEN um token multivalor aparece dentro de `IN (...)` THEN o sistema SHALL expandi-lo em um bind por valor.
4. IF um token multivalor não tem valores THEN o sistema SHALL usar um único bind `NULL`.
5. IF o SQL de uma visão contém um token inexistente WHEN a visão é salva THEN o sistema SHALL responder 400 com `code=UNKNOWN_TOKEN` e o nome do token.
6. WHEN um token aparece dentro de um literal de texto SQL (`'...'`) THEN o sistema SHALL mantê-lo como texto, sem substituição.
7. WHEN `{{inicio_mes}}` é resolvido em 15/05/2026 THEN o valor SHALL ser `2026-05-01`, e `{{fim_mes}}` SHALL ser `2026-05-31`.

**Independent Test**: IT: visão SQL `WHERE vendedor_id = {{usuario.atributo.vendedor_id}}` consultada por dois usuários com atributos diferentes.

---

### P1: Tokens em textos ⭐ MVP

**User Story**: Como construtor, quero usar tokens em títulos e textos de telas ("Vendas – {{inicio_mes}} a {{hoje}}").

**Why P1**: RF-19.3 e RF-24.2 usam tokens em texto.

**Acceptance Criteria**:
1. WHEN `TokenService.interpolateText("Vendas – {{inicio_mes}} a {{hoje}}", ctx)` é chamado em 15/05/2026 THEN o resultado SHALL ser `Vendas – 01/05/2026 a 15/05/2026`.
2. IF o texto contém token desconhecido THEN o sistema SHALL manter o texto original do token, sem erro.
3. WHEN `GET /api/v1/tokens` é chamado THEN o sistema SHALL devolver o catálogo de tokens com nome, tipo e descrição (para autocomplete).
4. WHEN `POST /api/v1/tokens/preview` é chamado THEN o sistema SHALL devolver os valores dos tokens para o usuário atual.

**Independent Test**: Teste unitário com relógio fixo.

---

### P2: Tokens customizados [V1]

**User Story**: Como admin, quero definir tokens constantes (`{{meta_margem}} = 0.25`) e de consulta (`{{ultimo_fechamento}}`).

**Why P2**: V1.

**Acceptance Criteria**:
1. WHEN um token de consulta é usado THEN o sistema SHALL reutilizar o valor em cache por N minutos configurados.

**Independent Test**: Diferido para a V1.

---

## Edge Cases

- IF um token escalar com 2+ valores aparece fora de `IN (...)` THEN o motor SHALL responder 400 com `code=TOKEN_MULTIVALUE_IN_SCALAR`.
- WHEN o SQL tem comentários `--` ou `/* */` contendo tokens THEN o sistema SHALL ignorá-los.
- IF a chave do atributo contém caracteres fora de `[a-z0-9_]` THEN o token SHALL ser rejeitado como desconhecido.
- WHEN a virada de dia ocorre entre duas consultas THEN `{{hoje}}` SHALL refletir a nova data (sem cache de valor).

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| TOK-01 | P1: Catálogo de tokens de sistema | RF-15.1 | Tasks | Pending |
| TOK-02 | P1: SQL com binds e multivalor | RF-15.3 | Tasks | Pending |
| TOK-03 | P1: Validação no salvamento | RN-15.1 | Tasks | Pending |
| TOK-04 | P1: Interpolação em texto | RF-15.3 | Tasks | Pending |
| TOK-05 | P1: Catálogo e preview na API | RF-15.1 | Tasks | Pending |
| TOK-06 | P2: Tokens customizados | RF-15.2 | - | Pending |

**Coverage:** 6 total, 5 mapped to tasks, 1 diferido (V1).

---

## Success Criteria

- [ ] Teste prova que nenhum valor de token aparece literalmente no SQL executado.
