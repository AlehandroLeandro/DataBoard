# F17 — Editor SQL Specification

**Prioridade:** MVP · **Depende de:** F07, F12, F15 · **Origem:** F17 do `produto.specs.md`

## Problem Statement

Analistas que sabem SQL precisam explorar dados das conexões rapidamente, testar consultas e transformá-las em visões reutilizáveis, sem sair da plataforma e sem risco de alterar dados.

## Goals

- [ ] Analista com `SQL_EDITOR` escreve, executa e salva uma consulta como Visão em < 1 min.
- [ ] Nenhuma execução altera dados nem passa do timeout e do limite de linhas.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Execução de scripts com vários statements | RN-17.3: somente um `SELECT`/`WITH` por execução |
| Auditoria de cada execução (hash, duração) | V1 (F06) |
| Abertura do editor de objeto com o resultado | O botão navega para a criação de objeto da F19 com a visão pré-selecionada; a tela de destino é entregue na F19 |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| O que é executado | A seleção, se houver; senão, o conteúdo inteiro, que precisa ser um único statement | RF-17.1 + RN-17.3 | y |
| Acesso a conexões | Exige `SQL_EDITOR` e `VIEW` no recurso `DATASOURCE` | RN-17.1 + ACL | y |
| RLS no Editor SQL | Não se aplica: o SQL é livre sobre a conexão. Na conexão interna isso significa ler todas as `udt.v_*` e o `lake`; a UI de ACL da conexão interna mostra esse aviso | RLS é definida sobre visões; quem concede `SQL_EDITOR` + acesso à interna o faz conscientemente | y |
| Tokens | Permitidos e resolvidos como binds (F15) | Testar SQL de visão com tokens | y |
| Editor | CodeMirror 6 com `@codemirror/lang-sql` e schema da conexão para autocomplete | Mesma base do editor de fórmulas (F09) | y |
| Histórico | Últimas 50 execuções por usuário (SQL, conexão, data, duração, linhas, status); mais antigas descartadas | RF-17.3 | y |
| Snippets | Consultas salvas pessoais com nome; sem compartilhamento no MVP | RF-17.3 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Escrever e executar SQL ⭐ MVP

**User Story**: Como analista, quero escrever SQL com autocomplete e ver o resultado numa grade.

**Why P1**: Núcleo da feature.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/sql/execute` recebe `datasourceId` e um `SELECT` válido de quem tem `SQL_EDITOR` e `VIEW` na conexão THEN o sistema SHALL executá-lo e responder `{ columns: [{ name, type }], rows, truncated, durationMs }`.
2. IF o SQL não é um único `SELECT`/`WITH` THEN o sistema SHALL responder 400 com `code=ONLY_SELECT_ALLOWED`.
3. IF o usuário não tem a flag `SQL_EDITOR` THEN o sistema SHALL responder 403 `FLAG_REQUIRED`.
4. WHEN o resultado excede `limits.maxQueryRows` THEN o sistema SHALL devolver `maxQueryRows` linhas com `truncated=true`.
5. IF a execução excede `limits.queryTimeoutSeconds` THEN o sistema SHALL cancelar e responder 504 `QUERY_TIMEOUT`.
6. WHEN o usuário seleciona um trecho e executa THEN o frontend SHALL enviar só o trecho selecionado.
7. WHEN o usuário digita no editor THEN o frontend SHALL sugerir tabelas e colunas da conexão escolhida.
8. WHEN uma consulta na conexão interna lê `app.user_account` THEN o sistema SHALL responder 400 com `code=PERMISSION_DENIED_SOURCE` (permissão negada pelo banco).

**Independent Test**: IT de `/sql/execute` nos casos acima + teste de componente do editor.

---

### P1: Salvar como visão ⭐ MVP

**User Story**: Como analista, quero transformar uma consulta testada numa visão reutilizável.

**Why P1**: RF-17.2.

**Acceptance Criteria**:
1. WHEN o usuário escolhe "Salvar como Visão" com um nome THEN o frontend SHALL criar uma visão `SQL` via `POST /api/v1/views` com o SQL e a conexão atuais e navegar para o editor da visão.
2. WHEN o usuário escolhe "Criar objeto a partir deste resultado" THEN o frontend SHALL criar a visão e navegar para `/telas/nova-com-objeto?viewId=<id>`.

**Independent Test**: Teste de componente com MSW verificando o `POST /views` e a navegação.

---

### P1: Histórico e snippets ⭐ MVP

**User Story**: Como analista, quero reabrir consultas recentes e salvar as que uso sempre.

**Why P1**: RF-17.3.

**Acceptance Criteria**:
1. WHEN uma execução termina (sucesso ou erro) THEN o sistema SHALL registrá-la no histórico pessoal, mantendo só as últimas 50.
2. WHEN `GET /api/v1/sql/history` é chamado THEN o sistema SHALL devolver o histórico do usuário atual, mais recente primeiro.
3. WHEN o usuário salva um snippet com nome THEN o sistema SHALL guardá-lo em `POST /api/v1/sql/snippets` e listá-lo em `GET /api/v1/sql/snippets` apenas para ele.
4. IF um usuário tenta ler ou excluir snippet de outro THEN o sistema SHALL responder 404.

**Independent Test**: IT com 51 execuções → 50 no histórico.

---

### P1: Explorador lateral ⭐ MVP

**User Story**: Como analista, quero navegar o schema ao lado do editor e inserir nomes com um clique.

**Why P1**: RF-17.4.

**Acceptance Criteria**:
1. WHEN o usuário escolhe uma conexão THEN o frontend SHALL exibir o `SchemaExplorer` (F07) dessa conexão ao lado do editor.
2. WHEN o usuário clica numa tabela ou coluna do explorador THEN o frontend SHALL inserir o nome qualificado na posição do cursor.

**Independent Test**: Teste de componente.

---

## Edge Cases

- IF o SQL estiver vazio ou só com comentários THEN o sistema SHALL responder 400 `SQL_EMPTY`.
- IF o SQL exceder 100 KB THEN o sistema SHALL responder 413.
- WHEN a conexão externa está fora do ar THEN o sistema SHALL responder 502 `DATASOURCE_UNAVAILABLE`.
- IF a execução lança erro de sintaxe no banco THEN o sistema SHALL responder 400 `SQL_ERROR` com a mensagem do banco e registrar no histórico com status de erro.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| SQL-01 | P1: Executar (endpoint e limites) | RF-17.1, RN-17.1, RN-17.3 | Tasks | Pending |
| SQL-02 | P1: Executar (conexão interna restrita) | RN-17.2 | Tasks | Pending |
| SQL-03 | P1: Executar (editor e autocomplete) | RF-17.1 | Tasks | Pending |
| SQL-04 | P1: Salvar como visão / criar objeto | RF-17.2 | Tasks | Pending |
| SQL-05 | P1: Histórico e snippets | RF-17.3 | Tasks | Pending |
| SQL-06 | P1: Explorador lateral | RF-17.4 | Tasks | Pending |

**Coverage:** 6 total, 6 mapped to tasks, 0 unmapped.

---

## Success Criteria

- [ ] Nenhum teste consegue alterar dados via `/sql/execute`.
