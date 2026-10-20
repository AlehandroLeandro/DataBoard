# F23 — Gráficos Avançados Specification

**Prioridade:** V2 · **Depende de:** F22 · **Origem:** F23 do `produto.specs.md`
**Artefatos:** só spec (V2).

## Problem Statement

Algumas análises pedem gráficos específicos: cascata para pontes de resultado (DRE), treemap para participação hierárquica, calendário para sazonalidade diária, sankey para fluxos, mapa para geografia e gauge para atingimento de meta.

## Goals

- [ ] Seis tipos avançados disponíveis no mesmo editor de gráficos, com máscaras, drill e destaques (RF-23.1).
- [ ] Mapas funcionam offline com URL de tiles configurável e GeoJSON do Brasil embarcado (RF-23.2).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Mapas com roteamento ou camadas customizadas | Fora do produto |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Montagem | Novos builders no analytics (`transforms/charts/builders/`) gerando `option` do ECharts | AD-005, extensão da F22 | y |
| Mapa | ECharts `geo`/`map` com GeoJSON de UFs e municípios (código IBGE) servidos pelo app; tiles OSM via extensão Leaflet do ECharts com URL configurável em F02 | RF-23.2 | y |
| Gráfico temporário (RF-23.3) | Botão "Ver como gráfico" em objetos tabela no modo visualização, gerando um gráfico básico sem salvar | Interpretação do item da UpQuery | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Tipos avançados [V2]

**User Story**: Como construtor, quero cascata, treemap, calendário, sankey, mapa e gauge.

**Why P1**: Núcleo da feature.

**Acceptance Criteria**:
1. WHEN um gráfico `WATERFALL` é configurado com dimensão ordenada, medida e marcação de subtotais THEN o analytics SHALL gerar as barras de subida/descida e os subtotais.
2. WHEN um `TREEMAP` tem 1 a 3 dimensões e uma medida THEN o analytics SHALL gerar a hierarquia aninhada.
3. WHEN um `CALENDAR` tem coluna de data e medida THEN o analytics SHALL gerar o heatmap por dia.
4. WHEN um `SANKEY` tem dimensão origem, dimensão destino e medida THEN o analytics SHALL gerar nós e links; IF houver ciclo origem→destino THEN SHALL avisar `SANKEY_CYCLE`.
5. WHEN um `MAP` usa UF ou código IBGE de município, ou lat/long, com medida THEN o frontend SHALL exibir coroplético, bolhas ou pontos conforme a configuração.
6. WHEN um `GAUGE` tem medida, meta (fixa, token ou medida) e faixas de cor THEN o analytics SHALL gerar o medidor com o % atingido.
7. The tipos avançados SHALL usar o mesmo motor (F12), máscaras, drill (F27) e destaques (F26).

**Independent Test**: Snapshots de `option` por tipo.

---

### P2: Mapas offline e gráfico temporário [V2]

**User Story**: Como admin, quero mapas sem internet, e como consumidor quero ver uma tabela como gráfico na hora.

**Why P2**: RF-23.2, RF-23.3.

**Acceptance Criteria**:
1. WHERE a URL de tiles está configurada para um servidor interno the mapa SHALL carregar os tiles dele.
2. WHEN o consumidor escolhe "Ver como gráfico" numa tabela THEN o frontend SHALL exibir um gráfico básico com as mesmas dimensões e medidas, sem gravar nada.

**Independent Test**: Teste de componente.

---

## Edge Cases

- IF um código IBGE não existe no GeoJSON THEN a linha SHALL ser ignorada e contada no aviso `MAP_UNKNOWN_REGIONS`.
- WHEN o gauge tem meta 0 THEN o % SHALL ser `null`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| ACHT-01 | P1: Cascata | Tabela F23 | - | Pending |
| ACHT-02 | P1: Treemap | Tabela F23 | - | Pending |
| ACHT-03 | P1: Calendário | Tabela F23 | - | Pending |
| ACHT-04 | P1: Sankey | Tabela F23 | - | Pending |
| ACHT-05 | P1: Mapa | Tabela F23, RF-23.2 | - | Pending |
| ACHT-06 | P1: Gauge | Tabela F23 | - | Pending |
| ACHT-07 | P1: Reuso de motor/máscara/drill/destaque | RF-23.1 | - | Pending |
| ACHT-08 | P2: Gráfico temporário | RF-23.3 | - | Pending |

**Coverage:** 8 total, 0 mapped to tasks (V2).

---

## Success Criteria

- [ ] Mapa de UFs funciona sem acesso à internet.
