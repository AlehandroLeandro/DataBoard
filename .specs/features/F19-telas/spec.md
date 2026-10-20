# F19 — Telas (Dashboards) Specification

**Prioridade:** MVP (estrutura, edição, atualização, duplicar, lixeira, tela cheia) / V1 (abas, rascunho/publicação, exportar/importar JSON) · **Depende de:** F12, F04, F15 · **Origem:** F19 do `produto.specs.md`

## Problem Statement

Com visões prontas, o construtor precisa de um canvas para posicionar KPIs, tabelas, gráficos, imagens e textos numa grade responsiva. O consumidor precisa abrir, atualizar e analisar a tela, com cada objeto carregando de forma independente. Esta feature entrega a tela e o **framework de objetos**: tipos plugáveis no backend, no analytics e no frontend, e o pipeline de dados `Spring → banco → analytics → front`.

## Goals

- [ ] Construtor monta uma tela com 6 objetos em grade de 12 colunas, arrastando e redimensionando.
- [ ] Um objeto com erro não afeta os outros (CA-19.1).
- [ ] Atualização automática a cada N minutos sem recarregar a página (CA-19.2).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Abas dentro da tela (RF-19.4) | V1 |
| Rascunho e publicação (RF-19.6) | V1; no MVP salvar já publica |
| Exportar/importar definição JSON entre instâncias (RF-19.7) | V1 |
| Tipos de objeto concretos | F20 (Valor), F21 (Consulta), F22 (Gráfico), F24 (Imagem/Texto/Separador) |
| Filtros de tela e superiores | F28 (aqui só o ponto de extensão `filterValues`) |
| Exportação | F29 |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Armazenamento | Uma linha por tela em `app.screen` com `definition jsonb` (layout, cabeçalho, rodapé, opções, objetos com `id` UUID) e `version` otimista | "Metadado é dado" (§0.3): exportável como JSON | y |
| Acesso aos dados via tela | `VIEW` na tela basta para ver os dados dos seus objetos; não é preciso `VIEW` em cada visão usada. A RLS continua valendo sempre | Compartilhar uma tela não exige compartilhar cada visão; a RLS protege as linhas | y |
| Quem edita | `EDIT` na tela (VIEWER nunca, RN-04.2); criar tela exige papel BUILDER/ADMIN | F04 | y |
| Layout responsivo | Grade de 12 colunas, `rowHeight` 40 px, breakpoints `lg` (≥ 1200), `md` (≥ 768), `sm` (< 768); posições por breakpoint, `md`/`sm` derivados de `lg` quando ausentes | RF-19.1 | y |
| Pipeline de dados | `POST /screens/{id}/objects/{oid}/data` → handler do tipo monta `QueryRequest`(s) → `QueryEngine` (RLS) → analytics `/v1/transform/{kind}` → resposta | AD-003 | y |
| Deduplicação (RN-19.2) | Coalescência de consultas idênticas em andamento (mesmo SQL final + binds) no `QueryEngine`, com janela de até 2 s | Cobre a mesma renderização sem cache persistente | y |
| Atualização automática | Mínimo de 60 s; desligada por padrão; pausa quando a aba do navegador fica oculta | RF-19.5; evita carga inútil | y |
| Cabeçalho/rodapé com tokens | Interpolados no backend (`TokenService.interpolateText`) e devolvidos prontos no `GET /screens/{id}` | F15 | y |
| Lixeira | 30 dias, restaurável; job diário de limpeza | RF-19.8 | y |
| Duplicar objeto entre telas | Copiar para a área de transferência da aplicação (estado do front) e colar em outra tela em edição | RF-19.2 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Criar e editar telas ⭐ MVP

**User Story**: Como construtor, quero criar uma tela e posicionar objetos numa grade para montar um dashboard.

**Why P1**: Núcleo da visualização.

**Acceptance Criteria**:
1. WHEN um BUILDER chama `POST /api/v1/screens` com nome, descrição, categoria e ícone THEN o sistema SHALL criar a tela vazia com layout padrão, registrar o recurso `SCREEN` com `MANAGE` para o criador e responder 201.
2. WHEN `PUT /api/v1/screens/{id}` recebe a definição completa com `version` atual THEN o sistema SHALL validar cada objeto pelo handler do seu tipo, gravar, incrementar `version` e registrar as dependências de cada objeto (`SCREEN_OBJECT → VIEW`/`VIEW_COLUMN`/`MASK`).
3. IF o `version` enviado difere do atual THEN o sistema SHALL responder 409 `SCREEN_VERSION_CONFLICT`.
4. IF um objeto tem tipo desconhecido ou configuração inválida THEN o sistema SHALL responder 400 com erro em `objects[i].<campo>`.
5. IF quem salva não tem `EDIT` na tela ou é VIEWER THEN o sistema SHALL responder 403.
6. WHILE a tela está em modo edição the frontend SHALL permitir inserir objetos pelo menu (Valor, Consulta, Gráfico, Imagem, Texto, Separador), arrastar, redimensionar, duplicar e excluir.
7. The layout SHALL ter 12 colunas e posições por breakpoint (`lg`, `md`, `sm`), derivando `md` e `sm` de `lg` quando ausentes.
8. WHEN o construtor configura cabeçalho (título com tokens, logo, mostrar última atualização) e rodapé (texto com tokens) THEN `GET /api/v1/screens/{id}` SHALL devolver os textos já interpolados para o usuário.

**Independent Test**: IT de criação/atualização com objetos de um tipo de teste + teste de componente do editor.

---

### P1: Carregar objetos de forma independente ⭐ MVP

**User Story**: Como consumidor, quero que cada objeto carregue sozinho e que um erro num deles não quebre a tela.

**Why P1**: RN-19.1, CA-19.1.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/screens/{id}/objects/{objectId}/data` é chamado com `filterValues` por quem tem `VIEW` na tela THEN o sistema SHALL executar as consultas do objeto pelo motor (com RLS), enviar os resultados ao analytics para o pós-processamento do tipo e devolver o resultado pronto.
2. IF a visão de um objeto falha (erro de SQL, coluna ausente, timeout, origem fora) THEN o endpoint desse objeto SHALL responder o erro com `code`, e os demais objetos SHALL continuar respondendo normalmente.
3. WHEN a tela abre com 6 objetos THEN o frontend SHALL disparar as 6 requisições em paralelo e renderizar cada objeto assim que sua resposta chegar.
4. WHEN um objeto está carregando THEN o frontend SHALL exibir skeleton só nele; WHEN falha THEN SHALL exibir a mensagem de erro e o botão "Tentar novamente" só nele.
5. WHEN duas requisições idênticas (mesmo SQL final e binds) chegam em até 2 s uma da outra enquanto a primeira executa THEN o sistema SHALL executar a consulta uma vez e responder às duas.
6. IF o usuário não tem `VIEW` na tela THEN o sistema SHALL responder 403 sem executar consultas.
7. WHEN o usuário abre uma tela THEN o sistema SHALL publicar `ScreenOpened` (auditoria F06).

**Independent Test**: CA-19.1: tela com 6 objetos, um apontando para visão quebrada.

---

### P1: Atualização ⭐ MVP

**User Story**: Como consumidor, quero atualizar a tela manualmente, por objeto ou automaticamente.

**Why P1**: RF-19.5.

**Acceptance Criteria**:
1. WHEN o usuário clica "Atualizar tela" THEN o frontend SHALL recarregar todos os objetos com `bypassCache=true`.
2. WHEN o usuário clica "Atualizar" no menu de um objeto THEN o frontend SHALL recarregar só esse objeto.
3. WHERE a tela tem `refreshIntervalSec` ≥ 60 the frontend SHALL recarregar os objetos a cada intervalo sem recarregar a página.
4. IF `refreshIntervalSec` é menor que 60 e diferente de 0 THEN o sistema SHALL responder 400 em `refreshIntervalSec`.
5. WHILE a aba do navegador está oculta the frontend SHALL pausar a atualização automática e retomá-la ao voltar.
6. WHERE o cabeçalho mostra a última atualização the frontend SHALL exibir a hora da última carga concluída.

**Independent Test**: CA-19.2 com timers falsos (10 min → 2 recargas).

---

### P1: Duplicar, excluir e restaurar ⭐ MVP

**User Story**: Como construtor, quero duplicar telas e objetos e recuperar telas excluídas por engano.

**Why P1**: RF-19.2, RF-19.7 (duplicar), RF-19.8.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/screens/{id}/duplicate` é chamado por quem tem `VIEW` e papel BUILDER THEN o sistema SHALL criar uma cópia com novos ids de objeto, nome "<nome> (cópia)" e `MANAGE` para quem duplicou.
2. WHEN o construtor duplica um objeto THEN o frontend SHALL inserir uma cópia com novo id ao lado do original; WHEN cola um objeto copiado de outra tela THEN o frontend SHALL inseri-lo com novo id.
3. WHEN `DELETE /api/v1/screens/{id}` é chamado por quem tem `MANAGE` THEN o sistema SHALL mover a tela para a lixeira e removê-la do menu e das listagens.
4. WHEN `POST /api/v1/screens/trash/{id}/restore` é chamado em até 30 dias THEN o sistema SHALL restaurar a tela.
5. WHEN o job diário roda THEN o sistema SHALL excluir definitivamente telas na lixeira há mais de 30 dias e remover suas dependências.

**Independent Test**: IT de duplicar, excluir, restaurar e purgar.

---

### P1: Tela cheia e listagem ⭐ MVP

**User Story**: Como consumidor, quero listar as telas que posso ver e apresentá-las em tela cheia.

**Why P1**: RF-19.9 e navegação básica.

**Acceptance Criteria**:
1. WHEN `GET /api/v1/screens?categoryId=&q=` é chamado THEN o sistema SHALL listar só as telas com `VIEW` para o usuário.
2. WHEN o usuário ativa o modo tela cheia THEN o frontend SHALL usar a Fullscreen API, ocultar menus e manter a atualização automática.

**Independent Test**: Teste de componente + IT da listagem.

---

### P2: Abas, rascunho e exportação JSON [V1]

**User Story**: Como construtor, quero abas na tela, publicar só quando estiver pronto e levar telas entre instâncias.

**Why P2**: V1.

**Acceptance Criteria**:
1. WHEN o construtor publica uma tela THEN os consumidores SHALL ver somente a versão publicada, e as edições seguintes SHALL ficar em rascunho.
2. WHEN `GET /api/v1/screens/{id}/export` é chamado THEN o sistema SHALL devolver o JSON importável por `POST /api/v1/screens/import`.

**Independent Test**: Diferido para a V1.

---

## Edge Cases

- IF a definição tem mais de 100 objetos THEN o sistema SHALL responder 400 `SCREEN_TOO_MANY_OBJECTS`.
- IF dois objetos têm o mesmo `id` THEN o sistema SHALL responder 400 `DUPLICATE_OBJECT_ID`.
- WHEN um objeto referencia uma visão excluída THEN o endpoint de dados desse objeto SHALL responder 410 `VIEW_DELETED`, sem afetar os outros.
- IF o analytics está indisponível THEN cada objeto que depende dele SHALL responder 503 `ANALYTICS_UNAVAILABLE`, e objetos estáticos SHALL continuar renderizando.
- WHEN a tela está na lixeira THEN `GET /screens/{id}` SHALL responder 404 para não-donos e 410 `SCREEN_IN_TRASH` para quem tem `MANAGE`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| SCR-01 | P1: Criar/editar (CRUD, versão, permissões) | RF-19.1, RF-19.2 | Tasks | Pending |
| SCR-02 | P1: Criar/editar (framework de tipos e dependências) | RF-19.2, RN-12.1 | Tasks | Pending |
| SCR-03 | P1: Criar/editar (grade responsiva) | RF-19.1 | Tasks | Pending |
| SCR-04 | P1: Criar/editar (cabeçalho/rodapé com tokens) | RF-19.3 | Tasks | Pending |
| SCR-05 | P1: Objetos independentes (pipeline e isolamento de erro) | RN-19.1, CA-19.1, AD-003 | Tasks | Pending |
| SCR-06 | P1: Objetos independentes (deduplicação) | RN-19.2 | Tasks | Pending |
| SCR-07 | P1: Objetos independentes (auditoria de abertura) | RF-06.1 | Tasks | Pending |
| SCR-08 | P1: Atualização | RF-19.5, CA-19.2 | Tasks | Pending |
| SCR-09 | P1: Duplicar/excluir/restaurar | RF-19.2, RF-19.7, RF-19.8 | Tasks | Pending |
| SCR-10 | P1: Tela cheia e listagem | RF-19.9 | Tasks | Pending |
| SCR-11 | P2: Abas, rascunho, JSON | RF-19.4, RF-19.6, RF-19.7 | - | Pending |

**Coverage:** 11 total, 10 mapped to tasks, 1 diferido (V1).

---

## Success Criteria

- [ ] CA-19.1 e CA-19.2 automatizados.
- [ ] Adicionar um novo tipo de objeto exige só um handler no backend, um transform no analytics (se aplicável) e um renderer no front, sem tocar no núcleo da tela.
