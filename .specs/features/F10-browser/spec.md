# F10 — Browser (Grade de Dados) Specification

**Prioridade:** MVP (grade, edição, master-detail) / V1 (colar do Excel) · **Depende de:** F09 · **Origem:** F10 do `produto.specs.md`

## Problem Statement

As Tabelas Personalizadas (F09) têm API, mas o usuário de negócio precisa de uma grade parecida com planilha para navegar, filtrar e editar dados: inline, por formulário e em lote. É por ela que metas, orçamentos e cadastros são alimentados no dia a dia, respeitando permissões e concorrência.

## Goals

- [ ] Toda UDT tem um Browser padrão disponível assim que é criada.
- [ ] Usuário edita uma célula e vê fórmulas e rollups recalculados após salvar, sem recarregar a página.
- [ ] Botões e ações refletem as permissões de dados, e o backend revalida tudo (CA-10.1).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Colar bloco do Excel (RF-10.5) | V1 |
| Browser com fonte Visão (somente leitura) | Implementado nas tasks da F12, que cria as visões; o modelo de dados aqui já prevê `sourceType=VIEW` |
| Botão "Exportar" da grade (RF-10.10) | Implementado na F29, reaproveitando os filtros atuais da grade |
| Item de menu apontando para Browser | F30 |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Browser padrão | Criado automaticamente ao receber `UdtSchemaChanged` de uma tabela nova, com todas as colunas visíveis, ordenação `created_at desc` e 50 por página | RF-10.2 | y |
| ACL do Browser | Recurso `BROWSER` com ACL própria (`VIEW`/`EDIT`/`MANAGE`); o acesso aos dados exige também `DATA_*` na UDT de origem | Recortes de Browser podem ser compartilhados sem abrir a tabela inteira | y |
| Filtros por coluna | JSON `[{ column, op, value }]` com `op` ∈ `contains`, `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `between`, `in`, `isEmpty`, `isNotEmpty`, conforme o tipo | RF-10.3 | y |
| Busca global `q` | `ILIKE '%q%'` nas colunas de texto visíveis (TEXT, LONG_TEXT, EMAIL, URL, PHONE, SELECT e rótulo de referência) | RF-10.3 | y |
| Filtros padrão do Browser | Somados aos filtros do usuário com E; o usuário não remove os filtros padrão | Recorte do tipo "Metas 2026 – Sul" é intencional | y |
| Preferências pessoais | Colunas ocultas, larguras e colunas congeladas por usuário e Browser | RF-10.3 | y |
| Conflito de versão | UI oferece "Recarregar" (descarta edição) e "Sobrescrever" (reenvia com o `version` atual) | RN-10.1 | y |
| Edição em lote | Excluir selecionados e "alterar valor de um campo" usam `records/bulk` (atômico) | RF-10.7 + F09 | y |
| Sub-grade master-detail | Mostra até 200 filhos por relação inversa, com inclusão inline já com a referência preenchida | RF-10.6 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Navegar dados na grade ⭐ MVP

**User Story**: Como usuário, quero navegar, ordenar, filtrar e buscar registros numa grade para encontrar o que preciso rapidamente.

**Why P1**: Porta de entrada de dados da plataforma.

**Acceptance Criteria**:
1. WHEN uma UDT é criada THEN o sistema SHALL criar o Browser padrão dela, visível a quem tem `DATA_READ` na tabela.
2. WHEN `GET /api/v1/browsers/{id}/data?page&size&sort&filters&q` é chamado THEN o sistema SHALL devolver a página com paginação no servidor, aplicando filtros padrão do Browser, filtros do usuário e busca global, combinados com E.
3. IF um filtro usa coluna inexistente ou operador incompatível com o tipo THEN o sistema SHALL responder 400 com `code=INVALID_FILTER` e o índice do filtro.
4. WHEN o usuário oculta, redimensiona ou congela colunas THEN o sistema SHALL gravar a preferência pessoal e reaplicá-la na próxima abertura.
5. The SQL de dados do Browser SHALL usar identificadores apenas do metadado e valores apenas como bind parameters.
6. IF o usuário não tem `DATA_READ` na UDT de origem THEN o sistema SHALL responder 403 mesmo tendo `VIEW` no Browser.

**Independent Test**: IT de `data` com filtros combinados + teste de componente da grade.

---

### P1: Editar registros (inline, formulário, inclusão rápida) ⭐ MVP

**User Story**: Como usuário com permissão, quero editar células, abrir o formulário do registro e incluir linhas rapidamente.

**Why P1**: Alimentação de dados.

**Acceptance Criteria**:
1. WHEN o usuário edita uma célula e confirma THEN o frontend SHALL enviar `PATCH` com o campo e o `version`, e atualizar a linha com o registro devolvido (inclusive derivados recalculados).
2. IF o backend devolve `VALIDATION_ERROR` THEN o frontend SHALL exibir a mensagem inline na célula ou no campo do formulário correspondente.
3. IF o backend devolve `RECORD_VERSION_CONFLICT` THEN o frontend SHALL exibir o diálogo com "Recarregar" e "Sobrescrever".
4. WHEN o usuário abre o formulário de um registro THEN o frontend SHALL exibir os campos agrupados nas seções configuradas no Browser.
5. WHILE o usuário não tem `DATA_INSERT` the frontend SHALL ocultar "Novo" e a linha de inclusão rápida; WHILE não tem `DATA_UPDATE` the frontend SHALL desabilitar a edição; WHILE não tem `DATA_DELETE` the frontend SHALL ocultar "Excluir".
6. The campos `FORMULA`, `LOOKUP`, `ROLLUP` e de sistema SHALL ser exibidos como somente leitura, com ícone indicativo.
7. WHEN um campo `REFERENCE` é editado THEN o frontend SHALL exibir um combo pesquisável alimentado por `reference-options` com paginação no servidor.

**Independent Test**: CA-10.1 (botão "Novo" oculto e `POST` direto 403) e CA-10.2 (conflito entre dois usuários).

---

### P1: Master-detail e histórico ⭐ MVP

**User Story**: Como usuário, quero ver e incluir os itens de um pedido dentro do próprio pedido, e consultar quem alterou o registro.

**Why P1**: RF-10.6 e RF-10.9.

**Acceptance Criteria**:
1. WHEN o formulário de um registro é aberto e a tabela tem relações inversas THEN o frontend SHALL exibir uma sub-grade por relação com os filhos do registro.
2. WHEN o usuário inclui uma linha na sub-grade THEN o frontend SHALL criar o registro filho com a referência ao pai já preenchida.
3. WHEN o usuário abre a aba "Histórico" do formulário THEN o frontend SHALL listar usuário, data/hora, campo, valor antigo e valor novo.

**Independent Test**: Pedido com 3 itens; incluir o 4º pela sub-grade; ver o rollup do pedido atualizado.

---

### P1: Ações em lote ⭐ MVP

**User Story**: Como usuário, quero selecionar várias linhas para excluir ou alterar um campo de uma vez.

**Why P1**: RF-10.7.

**Acceptance Criteria**:
1. WHEN o usuário seleciona N linhas e escolhe "Excluir" e confirma THEN o frontend SHALL enviar um único `bulk.delete` com os N ids.
2. WHEN o usuário escolhe "Alterar campo", um campo e um valor THEN o frontend SHALL enviar um único `bulk.update` com os N ids e versões.
3. IF o bulk falhar THEN o frontend SHALL exibir os erros por linha e manter a seleção, sem alterar nenhum registro.

**Independent Test**: Teste de componente com MSW verificando o payload do bulk.

---

### P1: Configurar Browsers adicionais ⭐ MVP

**User Story**: Como construtor, quero criar recortes da mesma tabela (ex.: "Metas 2026 – Sul") com colunas, filtros e seções próprias.

**Why P1**: RF-10.1/RF-10.2.

**Acceptance Criteria**:
1. WHEN um construtor com `EDIT` na UDT chama `POST /api/v1/browsers` com fonte, colunas, ordem, larguras, ordenação, filtros padrão, tamanho de página e seções THEN o sistema SHALL criar o Browser, registrar o recurso com `MANAGE` para o criador e responder 201.
2. WHEN uma coluna configurada no Browser é excluída da UDT THEN o sistema SHALL removê-la da configuração do Browser automaticamente.
3. IF alguém tenta excluir o Browser padrão de uma UDT existente THEN o sistema SHALL responder 409 com `code=DEFAULT_BROWSER`.

**Independent Test**: IT de CRUD de Browser.

---

### P2: Colar do Excel [V1]

**User Story**: Como usuário, quero colar um bloco de células do Excel para inserir ou atualizar em lote.

**Why P2**: V1.

**Acceptance Criteria**:
1. WHEN o usuário cola um bloco na grade THEN o sistema SHALL pré-validar e exibir um relatório de erros por linha antes de gravar.

**Independent Test**: Diferido para a V1.

---

## Edge Cases

- IF `size` exceder 500 THEN o sistema SHALL limitar a 500.
- WHEN o filtro `between` recebe só um extremo THEN o sistema SHALL tratar como `gte` ou `lte`.
- IF a UDT de origem estiver na lixeira THEN `GET /browsers/{id}/data` SHALL responder 410 com `code=SOURCE_DELETED`.
- WHEN a edição inline é cancelada com Esc THEN o frontend SHALL restaurar o valor original sem chamar a API.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| BRW-01 | P1: Navegar (Browser padrão) | RF-10.2 | Tasks | Pending |
| BRW-02 | P1: Navegar (dados, filtros, busca) | RF-10.3 | Tasks | Pending |
| BRW-03 | P1: Navegar (preferências pessoais) | RF-10.3 | Tasks | Pending |
| BRW-04 | P1: Editar (inline, formulário, inclusão) | RF-10.4, RF-10.8 | Tasks | Pending |
| BRW-05 | P1: Editar (erros, conflito) | RN-10.1, RN-10.3 | Tasks | Pending |
| BRW-06 | P1: Editar (permissões na UI) | RN-10.2, CA-10.1 | Tasks | Pending |
| BRW-07 | P1: Master-detail | RF-10.6 | Tasks | Pending |
| BRW-08 | P1: Histórico | RF-10.9 | Tasks | Pending |
| BRW-09 | P1: Lote | RF-10.7 | Tasks | Pending |
| BRW-10 | P1: Configurar Browsers | RF-10.1, RF-10.2 | Tasks | Pending |
| BRW-11 | P2: Colar do Excel | RF-10.5 | - | Pending |

**Coverage:** 11 total, 10 mapped to tasks, 1 diferido (V1).

---

## Success Criteria

- [ ] CA-10.1 e CA-10.2 automatizados.
- [ ] Grade com 50 linhas e 20 colunas responde a filtro em < 500 ms com 100 mil registros (Postgres local).
