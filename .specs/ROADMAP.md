# ROADMAP — DataBoard

Índice das features e ordem de execução. Fonte: `produto.specs.md` + decisões em `STATE.md`.

**Artefatos por pasta:**

- **MVP:** `spec.md` + `design.md` + `tasks.md`.
- **V1/V2:** só `spec.md`. Design e tasks são feitos quando a feature entrar em execução.

Features MVP com partes V1 têm todos os requisitos no `spec.md` (marcados `[MVP]`/`[V1]`/`[V2]`). O design e as tasks cobrem só o MVP.

## Corte do MVP

Instalar → conectar banco ou criar tabelas personalizadas → alimentar dados (browser/planilha) → criar visão → montar tela com KPIs, tabelas e gráficos → filtrar → exportar → controlar quem vê o quê.

## Fases de execução do MVP

| Fase | Ordem | Feature | Pasta | Prefixo de requisito |
|---|---|---|---|---|
| 1 — Fundação | 1 | F00 Fundação (monorepo, compose, CI, contrato) | `features/F00-fundacao` | FND |
| 2 — Plataforma e segurança | 2 | F01 Distribuição e Instalação | `features/F01-instalacao` | INST |
| | 3 | F03 Autenticação e Sessão | `features/F03-autenticacao` | AUTH |
| | 4 | F04 Usuários, Grupos e Permissões | `features/F04-usuarios-permissoes` | ACL |
| | 5 | F02 Configurações Gerais | `features/F02-configuracoes` | CFG |
| | 6 | F06 Auditoria (login/acesso/metadados básicos) | `features/F06-auditoria` | AUD |
| 3 — Dados próprios | 7 | F13/F14 Categorias e Máscaras | `features/F13-F14-categorias-mascaras` | CAT / MASK |
| | 8 | F09 Tabelas Personalizadas ⭐ | `features/F09-tabelas-personalizadas` | UDT |
| | 9 | F10 Browser | `features/F10-browser` | BRW |
| | 10 | F11 Importação de Planilha | `features/F11-importacao` | IMP |
| 4 — Semântica e consulta | 11 | F07 Conexões de Dados | `features/F07-conexoes` | DS |
| | 12 | F12 Visões e Motor de Consulta | `features/F12-visoes-motor-consulta` | VIEW |
| | 13 | F15 Tokens (sistema) | `features/F15-tokens` | TOK |
| | 14 | F05 Segurança em Nível de Linha | `features/F05-rls` | RLS |
| | 15 | F17 Editor SQL | `features/F17-editor-sql` | SQL |
| 5 — Visualização | 16 | F19 Telas | `features/F19-telas` | SCR |
| | 17 | F20 Objeto Valor | `features/F20-objeto-valor` | KPI |
| | 18 | F21 Objeto Consulta (tabela) | `features/F21-objeto-consulta` | TBL |
| | 19 | F22 Objeto Gráfico | `features/F22-objeto-grafico` | CHT |
| | 20 | F24 Objetos Estáticos | `features/F24-objetos-estaticos` | STO |
| | 21 | F28 Filtros | `features/F28-filtros` | FLT |
| | 22 | F29 Exportação | `features/F29-exportacao` | EXP |
| | 23 | F30 Menus e Tela Inicial | `features/F30-menus` | MENU |

## V1 (só spec)

| Feature | Pasta | Prefixo |
|---|---|---|
| F08 ETL e Data Lake | `features/F08-etl-lake` | ETL |
| F16 Listas de Valores | `features/F16-lov` | LOV |
| F18 Consulta Customizada | `features/F18-consulta-customizada` | CQ |
| F25 Linha Calculada | `features/F25-linha-calculada` | CROW |
| F26 Destaques e Sinalizadores | `features/F26-destaques` | HL |
| F27 Drill Down/Through | `features/F27-drill` | DRL |
| F31 Favoritos | `features/F31-favoritos` | FAV |
| F33 Agendador de Tarefas | `features/F33-agendador` | SCH |
| F34 Processos | `features/F34-processos` | PROC |
| F35 Alertas e Ações | `features/F35-alertas` | ALR |
| F37 Agendamento de Relatórios | `features/F37-assinaturas` | SUB |

## V2 (só spec)

| Feature | Pasta | Prefixo |
|---|---|---|
| F23 Gráficos Avançados | `features/F23-graficos-avancados` | ACHT |
| F32 Avisos | `features/F32-avisos` | NOTE |
| F36 Relatórios (Report) | `features/F36-relatorios` | RPT |
| F38 Modo TV | `features/F38-modo-tv` | TV |
| F39 API Pública | `features/F39-api-publica` | API |

## Convenções de rastreabilidade

- Cada requisito de spec tem ID `PREFIXO-NN` e a coluna **Origem** aponta o `RF/RN/CA` do `produto.specs.md`.
- Tasks referenciam os IDs de requisito; commits seguem Conventional Commits com escopo = módulo (`feat(udt): ...`).
