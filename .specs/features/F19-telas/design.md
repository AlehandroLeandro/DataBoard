# F19 — Telas Design

**Spec**: `.specs/features/F19-telas/spec.md`
**Status**: Draft

---

## Architecture Overview

```mermaid
graph TD
    subgraph frontend
      SP[ScreenPage / ScreenEditor] --> GL[react-grid-layout Responsive]
      GL --> OF[ObjectFrame - skeleton/erro/refresh por objeto]
      OF --> OR[ObjectRendererRegistry: VALUE, QUERY, CHART, IMAGE, TEXT, SEPARATOR]
      OF -->|useObjectData| API
    end
    subgraph backend dashboard
      API[ScreenObjectDataController] --> ODS[ObjectDataService]
      ODS --> REGH[ObjectTypeHandlerRegistry]
      REGH --> H[ObjectTypeHandler por tipo]
      ODS --> QE[QueryEngine F12 + coalescer]
      ODS --> AC[AnalyticsClient F00]
      SC[ScreenController] --> SS[ScreenService]
      SS --> REGH
      SS --> DB[(app.screen definition jsonb)]
    end
    subgraph analytics
      AC --> TR[/v1/transform/kind/ - registry de transforms]
    end
```

### Contrato do framework de objetos

```java
public interface ObjectTypeHandler {
  String type();                                              // "VALUE", "QUERY", ...
  List<FieldError> validate(JsonNode config, ViewCatalog views);
  Set<DepRef> dependencies(JsonNode config);                  // VIEW, VIEW_COLUMN, MASK, FILE...
  ObjectDataPlan plan(JsonNode config, FilterValues filters); // 0..n QueryRequests + transform kind + spec
  default boolean isStatic() { return false; }                // IMAGE/TEXT/SEPARATOR
}
```

`ObjectDataService.load(screen, object, filters)`:

1. `handler.plan(...)`.
2. Para cada `QueryRequest`: `QueryEngine.execute(req, ctx{authorizedVia=SCREEN})`.
3. Se o plano define `transformKind`: `AnalyticsClient.post("/v1/transform/" + kind, { results, spec })`.
4. Devolve `{ objectId, type, data, meta: { lastRefresh, truncated, durationMs } }`.

No analytics, `transforms/registry.py` mapeia `kind → função pura (polars)`. A F19 entrega o registry, os modelos comuns e um kind `passthrough` (usado nos testes do framework). F20–F22 adicionam `kpi`, `table` e `chart`.

---

## Code Reuse Analysis

### Existing Components to Leverage

| Component | Location | How to Use |
| --- | --- | --- |
| `QueryEngine`, `QueryContext`, `ResultCache` | `query` (F12) | Execução com RLS; adicionar `authorizedVia=SCREEN` e o coalescer |
| `AnalyticsClient` | F00 | Transformações |
| `TokenService.interpolateText` | F15 | Cabeçalho e rodapé |
| `AccessControl`, `CategoryAssignments`, `DependencyRegistry` | F04 / F13 | Recurso, categoria, dependências |
| `ScreenOpened` | `audit.api` (F06) | Auditoria |
| `JobRunRecorder` | AD-015 | Purga da lixeira |
| `formatValue`, `DependentsModal`, `AclEditor`, `CategorySelect` | front F04/F09/F13 | Editor e objetos |

### Integration Points

| System | Integration Method |
| --- | --- |
| F20/F21/F22/F24 | Registram `ObjectTypeHandler` (backend), transform (analytics) e renderer/editor (front) |
| F28 | `FilterValues` → `ScreenFilterApplier` SPI dentro de `plan` |
| F29 | Exportação reutiliza `ObjectDataService` |
| F30 | Itens de menu do tipo `SCREEN` |
| F17 | Rota `/telas/nova-com-objeto?viewId=` |

---

## Components

### `dashboard.screen.ScreenService` / `ScreenController`

- `create`, `get` (interpola cabeçalho/rodapé, publica `ScreenOpened`), `update(definition, version)`, `list`, `duplicate`, `delete` (lixeira), `restore`, `listTrash`.
- Validação: `ScreenDefinitionValidator` (máx. 100 objetos, ids únicos, posições dentro de 12 colunas, `refreshIntervalSec`) + `handler.validate` por objeto.
- Dependências: `replaceOutgoing(SCREEN_OBJECT:oid, handler.dependencies(cfg))` para cada objeto; remove arestas de objetos excluídos.
- **Location**: `backend/src/main/java/io/databoard/dashboard/screen/`

### `dashboard.object.ObjectTypeHandlerRegistry` / `ObjectDataService` / `ScreenObjectDataController`

- `POST /api/v1/screens/{id}/objects/{objectId}/data` `{ filterValues, bypassCache }`.
- Erros por objeto mapeados para o `code` do motor ou do analytics; nunca vazam stack.

### `query.InFlightQueryCoalescer`

- `ConcurrentHashMap<key, CompletableFuture<QueryResult>>`; a chave é o hash do SQL final + binds + datasource; a entrada é removida ao concluir. Integrado no `QueryEngine.execute` antes do cache.

### `dashboard.screen.ScreenTrashPurgeJob`

- Diário às 03:30; remove telas com `deleted_at` > 30 dias e suas arestas.

### analytics — `transforms/registry.py`, `api/v1/transform.py`, `models.py`

- `POST /v1/transform/{kind}` → `registry[kind](TransformRequest) -> dict`; kind desconhecido → 404.
- `TransformRequest { results: [ { columns, rows } ], spec: dict }`.

### Frontend — `features/screens/`

- `ScreensListPage`.
- `ScreenPage` (visualização): cabeçalho, grade somente leitura, botão atualizar, tela cheia, timer de auto-refresh com `document.visibilityState`.
- `ScreenEditor` (edição): `ObjectMenu`, `react-grid-layout` com drag/resize, `ObjectConfigDrawer` (abre o editor do tipo pelo `ObjectEditorRegistry`), `HeaderFooterEditor`, `ScreenSettingsDrawer` (refresh, ACL, categoria), duplicar/copiar/colar/excluir objeto, salvar com `version`.
- `ObjectFrame` + `useObjectData(screenId, objectId, filterValues)` (TanStack Query, uma query por objeto).
- `objectRegistry.ts`: `{ type → { Renderer, Editor, defaultConfig, defaultSize } }`.

---

## Data Models

```sql
-- V15__screens.sql
CREATE TABLE app.screen (
  id uuid PRIMARY KEY, name varchar(160) NOT NULL, description text, icon varchar(40),
  definition jsonb NOT NULL, version int NOT NULL DEFAULT 1,
  created_by uuid NOT NULL, created_at timestamptz NOT NULL DEFAULT now(),
  updated_by uuid, updated_at timestamptz NOT NULL DEFAULT now(), deleted_at timestamptz);
```

```json
{
  "layout": { "columns": 12, "rowHeight": 40 },
  "header": { "title": "Vendas – {{inicio_mes}} a {{hoje}}", "logoFileId": null, "showLastRefresh": true },
  "footer": { "text": "Fonte: ERP" },
  "refreshIntervalSec": 300,
  "filters": [],
  "objects": [
    { "id": "6f1c...", "type": "VALUE", "title": "Faturamento do mês",
      "position": { "lg": { "x": 0, "y": 0, "w": 3, "h": 3 } },
      "config": { } }
  ]
}
```

---

## Error Handling Strategy

| Error Scenario | Handling | User Impact |
| --- | --- | --- |
| Conflito de versão ao salvar | 409 `SCREEN_VERSION_CONFLICT` | "A tela foi alterada por outra pessoa"; recarregar |
| Config de objeto inválida | 400 com `objects[i].campo` | Objeto destacado no editor |
| Falha de dados de um objeto | Código do motor/analytics só naquele objeto | Mensagem + "Tentar novamente" no objeto |
| Visão excluída | 410 `VIEW_DELETED` no objeto | "Visão removida" |
| Analytics fora | 503 `ANALYTICS_UNAVAILABLE` por objeto | Objetos estáticos continuam |

---

## Risks & Concerns

| Concern | Location (file:line) | Impact | Mitigation |
| --- | --- | --- | --- |
| `VIEW` na tela dá acesso aos dados das visões usadas | `ObjectDataService` | Usuário vê dados de visão sem ACL própria | Decisão explícita (assumption); RLS sempre aplicada; endpoint só executa o que está na definição salva (nunca config enviada pelo cliente) |
| JSONB grande com muitos objetos | `app.screen` | Salvar pesado | Limite de 100 objetos |
| Auto-refresh de muitas TVs/abas | Backend | Carga | Mínimo de 60 s + pausa em aba oculta + coalescer |

---

## Tech Decisions (only non-obvious ones)

| Decision | Choice | Rationale |
| --- | --- | --- |
| Definição em JSONB único | Em vez de tabela de objetos | Exportável, versionável, atômico; dependências ficam em `app.dependency` |
| Endpoint por objeto | Em vez de um endpoint por tela | Isolamento de erro e carga progressiva (RN-19.1) |
| Coalescência em voo | Em vez de cache obrigatório | RN-19.2 sem servir dado velho em visões LIVE |
