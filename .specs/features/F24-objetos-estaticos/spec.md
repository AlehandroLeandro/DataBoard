# F24 — Objetos Estáticos (Imagem, Texto, Separador) Specification

**Prioridade:** MVP · **Depende de:** F19, F11 (arquivos), F15 (tokens) · **Origem:** F24 do `produto.specs.md`

## Problem Statement

Dashboards precisam de elementos que não vêm de dados: logo, títulos de seção, instruções, notas e espaçamento. Sem eles, as telas ficam difíceis de ler e de organizar.

## Goals

- [ ] Construtor insere imagem (upload ou URL), texto com markdown e tokens, e separadores, sem consultas ao banco.

## Out of Scope

| Feature | Reason |
| --- | --- |
| HTML livre no texto | Risco de XSS; só markdown sanitizado |
| Vídeo/iframe | Não previsto no produto |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Formato do texto | Markdown (CommonMark + tabelas GFM), sem HTML bruto, renderizado com `react-markdown` + sanitização | RF-24.2 com segurança | y |
| Tokens no texto | Interpolados no backend pelo endpoint de dados do objeto (`TokenService.interpolateText`), antes de o markdown ser renderizado | Mesma regra de títulos (F19) | y |
| Imagem por URL | Somente `http`/`https`; carregada pelo navegador do usuário | RF-24.1 | y |
| Imagem por upload | `POST /files` (F11) vinculado ao recurso `SCREEN`; leitura segue o acesso à tela | Arquivo herda a proteção | y |
| Link da imagem | Opcional, `http`/`https` ou rota interna (`/telas/<id>`), aberto em nova aba se externo | RF-24.1 | y |
| Separador | Linha horizontal ou espaço vazio, com espessura e cor da paleta | RF-24.3 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Objeto Imagem ⭐ MVP

**User Story**: Como construtor, quero inserir imagens por upload ou URL, com ajuste e link opcional.

**Why P1**: Logos e ilustrações em telas.

**Acceptance Criteria**:
1. WHEN o construtor envia uma imagem no editor do objeto THEN o frontend SHALL fazer `POST /files` e gravar `fileId` na config do objeto.
2. WHEN a tela é salva com um objeto `IMAGE` com `fileId` THEN o sistema SHALL vincular o arquivo ao recurso `SCREEN` e registrar a dependência `SCREEN_OBJECT → FILE`.
3. WHERE `fit` é `cover` ou `contain` the frontend SHALL aplicar o `object-fit` correspondente.
4. IF a URL ou o link usa esquema diferente de `http`/`https` (ou rota interna para o link) THEN o sistema SHALL responder 400 em `config.url` ou `config.link`.
5. WHEN um usuário com `VIEW` na tela abre a imagem enviada THEN `GET /files/{id}` SHALL responder 200.

**Independent Test**: IT de salvamento com vínculo + teste de componente.

---

### P1: Objeto Texto ⭐ MVP

**User Story**: Como construtor, quero escrever títulos, instruções e notas em markdown com tokens.

**Why P1**: Organização e contexto das telas.

**Acceptance Criteria**:
1. WHEN o objeto `TEXT` com `"Atualizado em {{hoje}}"` é carregado em 15/05/2026 THEN o endpoint de dados SHALL devolver `"Atualizado em 15/05/2026"`.
2. The frontend SHALL renderizar markdown (títulos, negrito, itálico, listas, links, tabelas) sem executar HTML bruto.
3. IF o texto contém `<script>` ou atributos `on*` THEN o frontend SHALL exibi-los como texto, sem executar.
4. The objeto `TEXT` SHALL não executar nenhuma consulta ao banco.

**Independent Test**: Teste de componente com payload malicioso + IT do endpoint.

---

### P1: Separador e espaçador ⭐ MVP

**User Story**: Como construtor, quero separar seções da tela visualmente.

**Why P1**: RF-24.3.

**Acceptance Criteria**:
1. WHERE `style=line` the frontend SHALL desenhar uma linha horizontal com a espessura (1–4 px) e a cor configuradas.
2. WHERE `style=space` the frontend SHALL ocupar a área sem desenho.

**Independent Test**: Teste de componente.

---

## Edge Cases

- IF o `fileId` não existe ou não é imagem THEN o sistema SHALL responder 400 em `config.fileId` ao salvar a tela.
- WHEN o texto passa de 20.000 caracteres THEN o sistema SHALL responder 400 em `config.markdown`.
- WHEN a imagem externa falha ao carregar THEN o frontend SHALL exibir placeholder com o texto alternativo.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| STO-01 | P1: Imagem | RF-24.1 | Tasks | Pending |
| STO-02 | P1: Texto (markdown seguro) | RF-24.2 | Tasks | Pending |
| STO-03 | P1: Texto (tokens) | RF-24.2, RF-15.3 | Tasks | Pending |
| STO-04 | P1: Separador | RF-24.3 | Tasks | Pending |

**Coverage:** 4 total, 4 mapped to tasks, 0 unmapped.

---

## Success Criteria

- [ ] Nenhum payload de XSS do conjunto de testes executa script.
