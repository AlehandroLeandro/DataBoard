# STATE

> Projeto **DataBoard** — BI open source self-hosted para PMEs.
> Documento de origem: `produto.specs.md` (v0.1, 19/10/2026). Decisões abaixo saem da sessão de grilling de 20/10/2026 (Q1–Q27).
> Convenção das specs: texto em pt-BR; palavras-chave EARS (`WHEN/WHILE/WHERE/IF/THEN/SHALL`) em inglês para que os validadores do `tlc-spec-driven` funcionem.

## Decisions

### AD-001
- **Decision**: Arquitetura em três serviços: `app` (Java/Spring Boot, API + SPA React estática), `analytics` (Python/FastAPI) e `postgres`.
- **Reason**: O mantenedor quer a manipulação de tabelas e gráficos em Python (pandas/polars é mais legível para pivot, totais e cálculos), mantendo segurança e persistência no ecossistema Spring.
- **Trade-off**: Um salto de rede extra por requisição de objeto e duas imagens Docker em vez de uma (a spec original pedia imagem única).
- **Scope**: Todas as features; F00, F01, F12, F19–F29.
- **Date**: 2026-10-20
- **Status**: active

### AD-002
- **Decision**: O Spring é o único portão de segurança e o único que acessa bancos. O `analytics` é stateless, não acessa banco, não é exposto fora da rede do compose e só aceita chamadas do `app` com header `X-Internal-Token` (`DATABOARD_INTERNAL_TOKEN`).
- **Reason**: RLS, ACL, auditoria e limites ficam centralizados em uma linguagem; elimina duplicação de regras de segurança em Python.
- **Trade-off**: O Python não pode otimizar consultas por conta própria; tudo chega a ele via Spring.
- **Scope**: F05, F12, F19–F29, F35, F37.
- **Date**: 2026-10-20
- **Status**: active

### AD-003
- **Decision**: Pipeline uniforme de dados de objeto: `Spring (SQL agregado no banco) → analytics (pós-processamento) → front`. Todo objeto Valor, Consulta e Gráfico passa pelo `analytics`; o Browser de UDT (F10) não passa.
- **Reason**: Uma só via para testar; agregação pesada fica no banco (onde é eficiente); o Python recebe resultado já agregado (centenas de linhas, não 100k).
- **Trade-off**: ~5–20 ms de latência extra mesmo em objetos que não precisam de transformação.
- **Scope**: F12, F19–F22, F25, F26, F29.
- **Date**: 2026-10-20
- **Status**: active

### AD-004
- **Decision**: Contrato Spring↔Python via HTTP/JSON, versionado em OpenAPI gerado pelo FastAPI (`analytics/openapi.json`), com testes de contrato nos dois lados. Endpoints: `POST /v1/transform/{table|pivot|chart|kpi}` e `POST /v1/export/xlsx`.
- **Reason**: Simples de depurar e de entender; FastAPI gera o contrato de graça.
- **Trade-off**: Serialização JSON é mais lenta que Arrow/gRPC; revisitar só com gargalo medido.
- **Scope**: F00, F12, F19–F22, F29.
- **Date**: 2026-10-20
- **Status**: active

### AD-005
- **Decision**: Gráficos são renderizados no front com Apache ECharts; o `analytics` devolve o objeto `option` do ECharts pronto (séries, eixos, "Outros", formatação de rótulos via `maskId`).
- **Reason**: Mantém interatividade (tooltip, clique, drill, filtro cruzado) e coloca a lógica de montagem do gráfico em Python.
- **Trade-off**: O schema de `option` do ECharts vira parte do contrato; upgrades do ECharts exigem cuidado.
- **Scope**: F22, F23, F20 (sparkline).
- **Date**: 2026-10-20
- **Status**: active

### AD-006
- **Decision**: Tabelas Personalizadas armazenadas como tabelas físicas geradas por DDL no schema `udt` (`udt.t_<id>`, colunas `f_<id>`), com campos derivados compilados em view `udt.v_<id>`.
- **Reason**: Performance nativa, FK real, índices e SQL comum para BI.
- **Trade-off**: Gestão de DDL e migração de tipo; nomes físicos opacos.
- **Scope**: F09, F10, F11, F12.
- **Date**: 2026-10-20
- **Status**: active

### AD-007
- **Decision**: Linguagem de fórmula OBExpr com parser ANTLR em Java, compilada para SQL Postgres. Linhas calculadas de DRE (F25) usam um avaliador próprio e seguro em Python (sem `eval`).
- **Reason**: Fórmulas de UDT viram SQL dentro de views, logo o compilador mora onde o DDL é gerado; F25 avalia após a consulta, no pós-processamento.
- **Trade-off**: Duas pequenas linguagens de expressão, em duas linguagens de programação.
- **Scope**: F09, F12, F25.
- **Date**: 2026-10-20
- **Status**: active

### AD-008
- **Decision**: Sessão por cookie server-side com Spring Session JDBC (`HttpOnly`, `Secure`, `SameSite=Lax`, CSRF).
- **Reason**: Encerrar sessão pelo admin e invalidação imediata ao bloquear são requisitos (CA-03.2, RN-03.2); SPA e API na mesma origem.
- **Trade-off**: Estado de sessão no banco.
- **Scope**: F03, F38, F39.
- **Date**: 2026-10-20
- **Status**: active

### AD-009
- **Decision**: Single-tenant: uma instância atende uma empresa.
- **Reason**: Multi-tenant por schema tem custo alto e não é necessidade de PME self-hosted.
- **Trade-off**: Hospedeiros terceiros precisam de uma instância por cliente.
- **Scope**: Todas.
- **Date**: 2026-10-20
- **Status**: active

### AD-010
- **Decision**: Montagem de SQL multi-dialeto com builder próprio sobre `JdbcTemplate` + camada `SqlDialect` (Postgres, MySQL/MariaDB, SQL Server no MVP) e JSqlParser para validar somente-SELECT. Sem jOOQ.
- **Reason**: jOOQ open source não suporta SQL Server; o escopo de SQL gerado é pequeno e controlado (identificadores só do metadado — RN-12.2).
- **Trade-off**: Manutenção própria das diferenças entre dialetos.
- **Scope**: F07, F12, F17.
- **Date**: 2026-10-20
- **Status**: active

### AD-011
- **Decision**: Exportação dividida: CSV por streaming no Spring; XLSX no `analytics` (`xlsxwriter`); PNG de gráfico via ECharts no front (MVP); PDF/PNG de tela no `analytics` com Playwright/Chromium headless (V1).
- **Reason**: CSV volumoso não deve atravessar o Python; XLSX com máscaras e linhas calculadas é mais simples em Python.
- **Trade-off**: Imagem do `analytics` fica ~400 MB quando o Chromium entrar (V1).
- **Scope**: F29, F36, F37.
- **Date**: 2026-10-20
- **Status**: active

### AD-012
- **Decision**: Stack e versões: Java 25 + Spring Boot 4.x + Maven + Spring Modulith (monólito modular) + Flyway; Python 3.13 + FastAPI + polars, gerenciado com uv, ruff, pyright; React 19 + TypeScript + Vite + Mantine + TanStack Query/Table + React Router + react-grid-layout + ECharts; PostgreSQL 17.
- **Reason**: LTS e versões estáveis atuais; Mantine traz date-range, combobox assíncrono e tema por CSS variables (white-label).
- **Trade-off**: Spring Boot 4 tem menos material de comunidade que o 3.x.
- **Scope**: Todas.
- **Date**: 2026-10-20
- **Status**: active

### AD-013
- **Decision**: Monorepo: `backend/`, `frontend/`, `analytics/`, `deploy/`, `.specs/`. Specs em pt-BR; código, identificadores e commits em inglês (Conventional Commits). Nome do produto `DataBoard`, pacote `io.databoard`, variáveis `DATABOARD_*`.
- **Reason**: Um repositório versiona contrato e as três partes juntos.
- **Trade-off**: CI precisa filtrar por caminho para não rodar tudo sempre.
- **Scope**: Todas.
- **Date**: 2026-10-20
- **Status**: active

### AD-014
- **Decision**: Interface somente em português (pt-BR), sem infraestrutura de i18n.
- **Reason**: Decisão do produto para o momento.
- **Trade-off**: Adicionar outros idiomas no futuro exigirá extrair strings.
- **Scope**: Frontend, e-mails, mensagens de erro.
- **Date**: 2026-10-20
- **Status**: active

### AD-015
- **Decision**: No MVP, jobs assíncronos usam Spring `@Scheduled` + executor assíncrono + tabela `app.job_run` (status/progresso). Quartz com JobStore JDBC entra na V1 (F33), reaproveitando `app.job_run`.
- **Reason**: MVP precisa de importação assíncrona e limpeza, mas não de agendamento configurável pelo usuário.
- **Trade-off**: Migração para Quartz na V1.
- **Scope**: F06, F09, F11, F19, F29, F33.
- **Date**: 2026-10-20
- **Status**: active

### AD-016
- **Decision**: Licença AGPL-3.0.
- **Reason**: Protege contra SaaS fechado sobre o projeto.
- **Trade-off**: Menor adoção corporativa que Apache-2.0.
- **Scope**: Repositório.
- **Date**: 2026-10-20
- **Status**: active

### AD-017
- **Decision**: Estratégia de testes: backend JUnit 5 + Testcontainers (Postgres; MySQL e SQL Server nos testes de dialeto); analytics pytest + testes de contrato; frontend Vitest + Testing Library; E2E Playwright cobrindo o corte do MVP. CI no GitHub Actions. Comandos expostos via `Makefile` na raiz.
- **Reason**: Banco real nos testes evita falsa confiança com H2; Makefile unifica gates entre as três linguagens.
- **Trade-off**: CI mais lenta (containers).
- **Scope**: Todas.
- **Date**: 2026-10-20
- **Status**: active

### AD-018
- **Decision**: Leitura de planilhas (xlsx/xls/csv: encoding, separador, inferência de tipos e normalização) acontece no `analytics`, que devolve um CSV UTF-8 normalizado ao Spring; o Spring valida contra a UDT e grava.
- **Reason**: Manipulação tabular em Python (AD-001); polars/calamine e charset-normalizer resolvem formatos e encodings com pouco código.
- **Trade-off**: A importação depende do `analytics` estar no ar; o arquivo atravessa a rede interna duas vezes (inspect e normalize).
- **Scope**: F11, F08 (prévia de cargas, se útil).
- **Date**: 2026-10-20
- **Status**: active

### AD-019
- **Decision**: Registro genérico de dependências `app.dependency(from_type, from_id, to_type, to_id)`, mantido pelos módulos ao salvar (`DependencyRegistry.replaceOutgoing`), é a única fonte para erros `*_IN_USE` e para análise de impacto.
- **Reason**: Exclusões seguras e impacto (RN-09.14, RN-12.4, RN-07.4, RF-12.7) sem que cada módulo consulte as tabelas dos outros.
- **Trade-off**: Cada feature precisa manter suas arestas corretamente; um esquecimento permite exclusão indevida.
- **Scope**: F07, F09, F12, F13/F14, F16, F19–F28, F35.
- **Date**: 2026-10-20
- **Status**: active

### AD-020
- **Decision**: Autorização por recurso exclusivamente via `security.access.AccessControl`, com registro genérico `app.resource_node` (tipo, id, pasta pai, herança, dono) e `app.acl_entry`, avaliada por requisição sem cache.
- **Reason**: Uma regra de ACL/herança/teto de papel para todos os tipos; efeito imediato de mudanças de grupo (CA-04.1).
- **Trade-off**: Uma consulta extra por autorização; listagens exigem `filterVisible` em lote.
- **Scope**: Todas as features que criam recursos (F07, F09, F10, F12, F19, F30, F36, F38).
- **Date**: 2026-10-20
- **Status**: active

### AD-021
- **Decision**: Módulos se comunicam por eventos de domínio Spring (`ApplicationEventPublisher`) para efeitos colaterais (auditoria, sessão, visão do sistema, Browser padrão), e por interfaces públicas (`@NamedInterface`) para consultas síncronas; o Spring Modulith verifica as fronteiras.
- **Reason**: Baixo acoplamento entre `security`, `audit`, `udt`, `semantic` e demais módulos do monólito.
- **Trade-off**: Fluxos ficam menos óbvios de seguir no código; exige testes de integração por evento.
- **Scope**: Backend inteiro.
- **Date**: 2026-10-20
- **Status**: active

### AD-022
- **Decision**: Objetos de tela são plug-ins em três partes: `ObjectTypeHandler` no Spring (validação, dependências, plano de consultas), transform registrado no analytics (`/v1/transform/{kind}`) e renderer/editor no `objectRegistry` do front. Cada objeto tem endpoint de dados próprio.
- **Reason**: Novos tipos (F23, V1/V2) entram sem tocar no núcleo da tela; erro de um objeto não afeta os outros (RN-19.1).
- **Trade-off**: Três pontos a manter por tipo; contrato do `option` ECharts acoplado ao analytics.
- **Scope**: F19–F24, F18, F23, F25, F26.
- **Date**: 2026-10-20
- **Status**: active

### AD-023
- **Decision**: `VIEW` numa tela autoriza a leitura dos dados dos objetos dela (`QueryContext.authorizedVia=SCREEN`) sem exigir `VIEW` em cada visão usada; a RLS continua sempre aplicada e o endpoint só executa a configuração salva.
- **Reason**: Compartilhar uma tela não deve obrigar a compartilhar cada visão; a proteção de linhas é da RLS.
- **Trade-off**: Quem pode editar a tela decide quais dados de uma visão ficam expostos a quem vê a tela.
- **Scope**: F19, F20–F22, F27, F29, F31, F38.
- **Date**: 2026-10-20
- **Status**: active

## Handoff

- **Feature**: nenhuma em execução — specs geradas, aguardando revisão do usuário
- **Phase / Task**: —
- **Completed**: none
- **In-progress** (file:line): none
- **Next step**: Usuário revisa `.specs/`; depois iniciar Execute por `.specs/features/F00-fundacao/tasks.md`.
- **Blockers**: none
- **Uncommitted files**: none
- **Branch**: main
