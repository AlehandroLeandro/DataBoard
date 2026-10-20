# F13/F14 — Categorias e Máscaras de Formatação Specification

**Prioridade:** MVP · **Depende de:** F04 · **Origem:** F13 e F14 do `produto.specs.md`

## Problem Statement

Visões, tabelas, telas e relatórios vão se acumular, e o usuário precisa organizá-los em categorias. Números, datas e documentos precisam aparecer formatados de modo consistente (`R$ 1.234,56`, `12,5%`, `123.456.789-09`) em qualquer objeto, a partir de um catálogo central de máscaras. As duas features são catálogos simples consumidos por todas as outras e por isso ficam juntas.

## Goals

- [ ] Catálogo de categorias em árvore de até 3 níveis, atribuível a qualquer recurso.
- [ ] Catálogo de máscaras predefinidas e customizadas, com formatação idêntica em toda a UI.
- [ ] Registro de dependências genérico disponível para todas as features (usado aqui para máscaras em uso).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Aplicação da máscara em XLSX | F29 (o analytics converte para formato nativo do Excel) |
| Associação de máscara a colunas de visão/campos/objetos | F09, F12, F21 (só consomem o `maskId`) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Profundidade máxima | 3 níveis, fixo no MVP | RF-13.1 ("padrão máximo de 3") | y |
| Onde guardar a categoria de um recurso | Tabela `app.resource_category(resource_type, resource_id, category_id)` no módulo `catalog` | Uma regra só para todos os tipos; excluir categoria é um `DELETE` | y |
| Quem pode gerenciar categorias | ADMIN e BUILDER | Construtores organizam o próprio conteúdo | y |
| IDs de máscaras predefinidas | Slugs estáveis: `int`, `dec2`, `brl`, `usd`, `eur`, `pct0`, `pct1`, `pct2`, `thousand`, `million`, `date_short`, `date_long`, `month_year`, `datetime`, `cpf`, `cnpj`, `cep`, `phone` | Referência estável no JSON exportado de telas | y |
| ID de máscara customizada | UUID | Padrão da API | y |
| Predefinidas | Somente leitura | Evitar quebrar telas existentes | y |
| Formatação | Biblioteca TS única `formatValue(value, mask)` usando `Intl` com `pt-BR` | RN-14.1 (frontend formata; API devolve cru) | y |
| Abreviação de escala | `1,2 mil` (÷1.000, 1 casa) e `3,4 mi` (÷1.000.000, 1 casa) | Exemplos do RF-14.1 | y |
| Padrão de texto | `#` = dígito da entrada, outros caracteres literais; entrada com menos dígitos → exibida crua | RF-14.2 (`###.###.###-##`) | y |
| Registro de dependências | Tabela `app.dependency(from_type, from_id, to_type, to_id)` mantida pelos módulos ao salvar | Base para `*_IN_USE` e análise de impacto (RF-12.7, RN-09.14) | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Categorias em árvore ⭐ MVP

**User Story**: Como construtor, quero organizar visões, tabelas, telas e relatórios em categorias e subcategorias para encontrá-los facilmente.

**Why P1**: Organização básica do conteúdo.

**Acceptance Criteria**:
1. WHEN um ADMIN ou BUILDER chama `POST /api/v1/categories` com `name` e `parentId` opcional THEN o sistema SHALL criar a categoria e responder 201.
2. IF a criação ou o movimento resultaria em profundidade maior que 3 THEN o sistema SHALL responder 400 com `code=CATEGORY_DEPTH_EXCEEDED`.
3. IF já existe uma categoria irmã com o mesmo nome (case-insensitive) THEN o sistema SHALL responder 409 com `code=CATEGORY_NAME_IN_USE`.
4. WHEN `GET /api/v1/categories` é chamado THEN o sistema SHALL devolver a árvore completa ordenada por nome.
5. WHEN uma categoria é excluída THEN o sistema SHALL excluir também as subcategorias e remover as associações, deixando os itens em "Sem categoria", sem excluí-los.
6. IF um `PUT` tornaria uma categoria descendente de si mesma THEN o sistema SHALL responder 400 com `code=CATEGORY_CYCLE`.
7. WHEN um módulo chama `CategoryAssignments.assign(resourceRef, categoryId)` THEN o recurso SHALL pertencer a no máximo uma categoria (substitui a anterior).
8. WHEN uma listagem de recursos recebe `?categoryId=<id>` THEN o sistema SHALL filtrar pelos recursos daquela categoria e de suas subcategorias.

**Independent Test**: IT: criar árvore de 3 níveis, tentar o 4º (400), excluir a raiz e ver as associações removidas.

---

### P1: Máscaras predefinidas e customizadas ⭐ MVP

**User Story**: Como construtor, quero escolher uma máscara pronta ou criar a minha para exibir valores no formato que a empresa usa.

**Why P1**: Todo objeto de visualização formata valores.

**Acceptance Criteria**:
1. WHEN `GET /api/v1/masks` é chamado THEN o sistema SHALL devolver as 18 máscaras predefinidas (marcadas `builtIn=true`) e as customizadas.
2. WHEN um BUILDER chama `POST /api/v1/masks` com tipo `NUMBER` (prefixo, sufixo, casas, separadores, escala, cor para negativos), `DATE` (padrão ICU) ou `TEXT` (padrão com `#`) THEN o sistema SHALL criar a máscara e responder 201.
3. IF alguém tenta alterar ou excluir uma máscara predefinida THEN o sistema SHALL responder 409 com `code=MASK_BUILT_IN`.
4. IF uma máscara customizada referenciada no registro de dependências é excluída THEN o sistema SHALL responder 409 com `code=MASK_IN_USE` e a lista de dependentes.
5. The função `formatValue` SHALL formatar `1234.5` com `brl` como `R$ 1.234,50`, `0.155` com `pct1` como `15,5%`, `1234` com `thousand` como `1,2 mil`, `12345678909` com `cpf` como `123.456.789-09` e `2026-05-15` com `date_short` como `15/05/2026`.
6. WHERE a máscara numérica tem `negativeColor` the função de formatação SHALL devolver a cor junto do texto para valores negativos.
7. IF o valor é `null` THEN `formatValue` SHALL devolver string vazia.

**Independent Test**: Testes de tabela em `formatValue` cobrindo todas as predefinidas.

---

### P1: Registro de dependências ⭐ MVP

**User Story**: Como desenvolvedor, quero um registro único de "quem usa o quê" para bloquear exclusões perigosas e mostrar impacto.

**Why P1**: Exigido por F07, F09, F12 e por esta feature.

**Acceptance Criteria**:
1. WHEN um módulo chama `DependencyRegistry.replaceOutgoing(from, targets)` THEN o sistema SHALL substituir todas as arestas que saem de `from` pelas informadas, na mesma transação do chamador.
2. WHEN `DependencyRegistry.dependentsOf(target)` é chamado THEN o sistema SHALL devolver os recursos que apontam para `target`, com tipo, id e nome legível.
3. WHEN um recurso é excluído THEN o módulo dono SHALL chamar `removeAll(from)` e o sistema SHALL apagar as arestas que saem dele.

**Independent Test**: IT do registro com arestas fictícias.

---

## Edge Cases

- IF o nome da categoria for vazio ou tiver mais de 80 caracteres THEN o sistema SHALL responder 400.
- IF o padrão ICU de uma máscara de data for inválido THEN o sistema SHALL responder 400 em `datePattern`.
- WHEN `formatValue` recebe texto não numérico com máscara numérica THEN a função SHALL devolver o texto original sem lançar erro.
- WHEN uma máscara `TEXT` recebe entrada com menos dígitos que o padrão THEN a função SHALL devolver a entrada crua.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| CAT-01 | P1: Categorias (árvore, profundidade, nomes) | RF-13.1 | Tasks | Pending |
| CAT-02 | P1: Categorias (exclusão) | RN-13.1 | Tasks | Pending |
| CAT-03 | P1: Categorias (associação e filtro) | RF-13.2 | Tasks | Pending |
| MASK-01 | P1: Máscaras (catálogo e CRUD) | RF-14.1, RF-14.2 | Tasks | Pending |
| MASK-02 | P1: Máscaras (formatação no front) | RN-14.1, RF-14.1 | Tasks | Pending |
| MASK-03 | P1: Máscaras (em uso) | RF-14.3 | Tasks | Pending |
| DEP-01 | P1: Registro de dependências | RN-09.14, RN-12.4, RN-07.4 | Tasks | Pending |

**Coverage:** 7 total, 7 mapped to tasks, 0 unmapped.

---

## Success Criteria

- [ ] As 18 máscaras predefinidas têm teste de formatação.
- [ ] Excluir categoria nunca exclui recurso.
