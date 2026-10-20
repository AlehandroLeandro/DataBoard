# F20 — Objeto Valor (KPI) Specification

**Prioridade:** MVP · **Depende de:** F19 · **Origem:** F20 do `produto.specs.md`

## Problem Statement

O gestor precisa ver números-chave de relance: faturamento do mês, comparação com o período anterior e quanto da meta foi atingido. O objeto Valor exibe uma medida agregada com comparativo, variação colorida, meta e mini-série temporal, tudo calculado de forma consistente pelo motor e pelo analytics.

## Goals

- [ ] KPI "Faturamento do mês" com comparação ao mês anterior, seta e cor, em uma configuração de até 6 campos.
- [ ] Meta vinda de outra visão (ex.: SUM das metas de uma UDT) com % atingido.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Clique abrindo drill (RF-20.4, parte drill) | V1 (F27) |
| Cor do card por regra de destaque | V1 (F26) |
| Filtro de período interativo da tela | F28 passa o intervalo ativo; aqui o objeto usa seu período padrão |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Período sem filtro de data ativo | O objeto tem `defaultPeriod` (`CURRENT_MONTH` padrão; também `CURRENT_YEAR`, `LAST_30_DAYS`, `TODAY`, `ALL`) | RN-20.1 exige período; F28 ainda não existe | y |
| Período anterior | Mesmo comprimento imediatamente antes; se o período é um mês, trimestre ou ano calendário completo, o anterior é o mês/trimestre/ano calendário anterior | Comparação intuitiva (abril vs. março, não "30 dias antes") | y |
| Mesmo período do ano anterior | Intervalo deslocado em −1 ano (29/02 → 28/02) | RF-20.1 | y |
| Direção boa | `higherIsBetter` (padrão `true`) define verde/vermelho | Custos sobem = ruim | y |
| Variação % com base zero | `null` (exibe "—") | Evita divisão por zero | y |
| "Estável" | Variação absoluta % < 0,5% → seta neutra cinza | Ruído não vira alerta | y |
| Sparkline | Últimos 12 grãos (`MONTH` padrão; `DAY`, `WEEK` não; `QUARTER`, `YEAR` sim) terminando no fim do período | RF-20.2 | y |
| Meta | `FIXED` (número), `TOKEN` (token numérico V1; no MVP só sistema não numérico → não aplicável), `MEASURE` (medida de qualquer visão acessível ao construtor, com o mesmo período e coluna de data própria) | RF-20.3; tokens numéricos customizados são V1 | y |
| Navegação por clique | Opcional: `onClick.navigateToScreenId` | RF-20.4 (parte MVP) | y |
| Cálculo | Spring executa as consultas (valor, comparativo, sparkline, meta); analytics `kpi` calcula variações, direção, % de meta e série | AD-003 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: KPI com comparativo ⭐ MVP

**User Story**: Como gestor, quero ver uma medida agregada com a variação em relação a um período de referência.

**Why P1**: Objeto mais usado em dashboards.

**Acceptance Criteria**:
1. WHEN um objeto `VALUE` com visão, medida, agregação, coluna de data e `defaultPeriod=CURRENT_MONTH` é carregado em 15/05/2026 THEN o sistema SHALL devolver o valor agregado de 01/05 a 31/05/2026.
2. WHERE `comparison=PREVIOUS_PERIOD` the sistema SHALL devolver o valor de 01/04 a 30/04/2026, a variação absoluta e a variação percentual.
3. WHERE `comparison=SAME_PERIOD_LAST_YEAR` the sistema SHALL comparar com 01/05 a 31/05/2025.
4. WHERE `comparison=MEASURE` the sistema SHALL comparar com outra medida agregada no mesmo período.
5. WHEN a variação é positiva e `higherIsBetter=true` THEN o resultado SHALL ter `direction=UP` e `sentiment=GOOD`; WHEN negativa THEN `direction=DOWN` e `sentiment=BAD`; com `higherIsBetter=false`, o `sentiment` SHALL ser invertido.
6. IF o valor de comparação é 0 ou nulo THEN `deltaPct` SHALL ser `null`.
7. IF |`deltaPct`| < 0,5% THEN `direction` SHALL ser `FLAT` e `sentiment` `NEUTRAL`.
8. IF o comparativo de período é configurado sem coluna de data THEN o sistema SHALL responder 400 em `config.dateColumn`.
9. The frontend SHALL exibir título, ícone, valor com a máscara configurada, seta e cor conforme `sentiment`.

**Independent Test**: Testes do transform `kpi` (pytest) + IT do handler com dados de 2025 e 2026.

---

### P1: Meta e sparkline ⭐ MVP

**User Story**: Como gestor, quero ver quanto da meta atingi e a tendência recente.

**Why P1**: RF-20.2 e RF-20.3.

**Acceptance Criteria**:
1. WHERE `target.kind=FIXED` com valor 100.000 e o valor é 80.000 the sistema SHALL devolver `targetAttainmentPct=0.8`.
2. WHERE `target.kind=MEASURE` aponta para `SUM(valor)` de uma visão de metas the sistema SHALL calcular a meta no mesmo período, usando a coluna de data da visão de metas.
3. WHERE `sparkline.enabled=true` the sistema SHALL devolver os 12 últimos grãos (`MONTH` por padrão) terminando no período atual, com 0 nos grãos sem dados para `SUM`/`COUNT` e `null` para as demais agregações.
4. The frontend SHALL exibir a barra de progresso da meta e a sparkline formatada com a máscara no tooltip.

**Independent Test**: IT com visão de vendas + visão de metas (UDT).

---

### P1: Navegar ao clicar ⭐ MVP

**User Story**: Como construtor, quero que o clique no KPI leve a uma tela de detalhe.

**Why P1**: Parte MVP do RF-20.4.

**Acceptance Criteria**:
1. WHERE `onClick.navigateToScreenId` está configurado, WHEN o usuário clica no KPI THEN o frontend SHALL navegar para a tela indicada.
2. IF o usuário não tem `VIEW` na tela destino THEN o frontend SHALL exibir o KPI sem cursor de clique.

**Independent Test**: Teste de componente.

---

## Edge Cases

- WHEN a medida não tem linhas no período THEN `value` SHALL ser 0 para `SUM`/`COUNT` e `null` para as demais.
- IF a visão da meta está inacessível ao usuário (sem RLS correspondente) THEN `targetAttainmentPct` SHALL ser `null` e o restante SHALL ser exibido normalmente.
- WHEN o período é `ALL` THEN o comparativo de período SHALL ficar indisponível, e a validação SHALL exigir `comparison` ∈ {`NONE`, `MEASURE`}.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| KPI-01 | P1: Valor e períodos | RF-20.1, RN-20.1 | Tasks | Pending |
| KPI-02 | P1: Comparativo e direção | RF-20.1 | Tasks | Pending |
| KPI-03 | P1: Meta | RF-20.3 | Tasks | Pending |
| KPI-04 | P1: Sparkline | RF-20.2 | Tasks | Pending |
| KPI-05 | P1: Renderização | RF-20.1 | Tasks | Pending |
| KPI-06 | P1: Navegação ao clicar | RF-20.4 | Tasks | Pending |

**Coverage:** 6 total, 6 mapped to tasks, 0 unmapped.

---

## Success Criteria

- [ ] Transform `kpi` com 100% dos ramos cobertos por pytest.
