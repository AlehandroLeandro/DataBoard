# Especificação do Produto — BI Open Source para PMEs

> **Nome provisório:** `OpenBI` (placeholder; evitar "UpQuery" no nome por questão de marca)
> **Versão da spec:** 0.1 (rascunho) · **Data:** 19/10/2026
> **Origem:** catálogo de engenharia reversa `claude/upquery-catalogo-funcionalidades.md`
> **Status:** RASCUNHO — itens marcados com `[DECISÃO]` precisam de confirmação antes da implementação.

---

## Sumário

- [0. Visão geral](#0-visão-geral)
- [1. Convenções da spec](#1-convenções-da-spec)
- [2. Arquitetura de referência](#2-arquitetura-de-referência)
- [3. Glossário](#3-glossário)
- [4. Mapa de features e roadmap](#4-mapa-de-features-e-roadmap)
- **Plataforma**
  - [F01 — Distribuição e Instalação (Docker)](#f01--distribuição-e-instalação-docker)
  - [F02 — Configurações Gerais e Identidade Visual](#f02--configurações-gerais-e-identidade-visual)
- **Segurança**
  - [F03 — Autenticação e Sessão](#f03--autenticação-e-sessão)
  - [F04 — Usuários, Grupos e Permissões](#f04--usuários-grupos-e-permissões)
  - [F05 — Segurança em Nível de Linha (Filtro de Usuários)](#f05--segurança-em-nível-de-linha-filtro-de-usuários)
  - [F06 — Auditoria e Logs](#f06--auditoria-e-logs)
- **Dados**
  - [F07 — Conexões de Dados](#f07--conexões-de-dados)
  - [F08 — ETL e Data Lake Interno](#f08--etl-e-data-lake-interno)
  - [F09 — Tabelas Personalizadas](#f09--tabelas-personalizadas) ⭐
  - [F10 — Browser (Grade de Dados)](#f10--browser-grade-de-dados)
  - [F11 — Importação de Dados e Upload](#f11--importação-de-dados-e-upload)
- **Modelagem semântica**
  - [F12 — Visões e Colunas](#f12--visões-e-colunas)
  - [F13 — Categorias](#f13--categorias)
  - [F14 — Máscaras de Formatação](#f14--máscaras-de-formatação)
  - [F15 — Tokens (Variáveis)](#f15--tokens-variáveis)
  - [F16 — Listas de Valores (LOV)](#f16--listas-de-valores-lov)
- **Consultas**
  - [F17 — Editor SQL](#f17--editor-sql)
  - [F18 — Consulta Customizada (Construtor Visual)](#f18--consulta-customizada-construtor-visual)
- **Visualização**
  - [F19 — Telas (Dashboards)](#f19--telas-dashboards)
  - [F20 — Objeto Valor (KPI)](#f20--objeto-valor-kpi)
  - [F21 — Objeto Consulta (Tabela e Pivot)](#f21--objeto-consulta-tabela-e-pivot)
  - [F22 — Objeto Gráfico (Tipos Básicos)](#f22--objeto-gráfico-tipos-básicos)
  - [F23 — Gráficos Avançados](#f23--gráficos-avançados)
  - [F24 — Objetos Estáticos (Imagem e Texto)](#f24--objetos-estáticos-imagem-e-texto)
  - [F25 — Linha Calculada](#f25--linha-calculada)
  - [F26 — Destaques e Sinalizadores](#f26--destaques-e-sinalizadores)
  - [F27 — Drill Down e Drill Through](#f27--drill-down-e-drill-through)
  - [F28 — Filtros](#f28--filtros)
  - [F29 — Exportação](#f29--exportação)
- **Navegação**
  - [F30 — Menus e Tela Inicial](#f30--menus-e-tela-inicial)
  - [F31 — Favoritos](#f31--favoritos)
  - [F32 — Avisos](#f32--avisos)
- **Automação**
  - [F33 — Agendador de Tarefas](#f33--agendador-de-tarefas)
  - [F34 — Processos (Rotinas SQL)](#f34--processos-rotinas-sql)
  - [F35 — Alertas e Ações](#f35--alertas-e-ações)
  - [F36 — Relatórios (Report)](#f36--relatórios-report)
  - [F37 — Agendamento e Envio de Relatórios](#f37--agendamento-e-envio-de-relatórios)
- **Distribuição de conteúdo**
  - [F38 — Modo TV (Kiosk)](#f38--modo-tv-kiosk)
  - [F39 — API Pública](#f39--api-pública)
- [5. Fora de escopo](#5-fora-de-escopo)
- [6. Decisões em aberto](#6-decisões-em-aberto)

---

## 0. Visão geral

### 0.1 Propósito

Plataforma web de Business Intelligence **open source e self-hosted**. Ela permite a pequenas e médias empresas:

1. conectar suas bases de dados;
2. criar **estruturas de dados próprias** (tabelas personalizadas com referências e campos calculados);
3. modelar indicadores;
4. montar dashboards altamente personalizados;
5. automatizar alertas e relatórios.

Tudo isso roda em um único `docker compose up`.

### 0.2 Público

| Persona | Necessidade |
|---|---|
| **Administrador** (TI da PME) | Instalar, configurar conexões, usuários e segurança. |
| **Analista / Construtor** | Criar tabelas personalizadas, visões, consultas, telas e alertas. |
| **Consumidor** (gestor, operador) | Ver dashboards, filtrar, exportar, receber relatórios e alertas, alimentar dados em tabelas personalizadas. |

### 0.3 Princípios

- **Self-hosted first:** nada depende de serviço externo para funcionar. E-mail (SMTP) é opcional.
- **Uma instância = uma empresa** (single-tenant). `[DECISÃO D-03]`
- **Metadado é dado:** telas, visões, tabelas e objetos são registros versionáveis e exportáveis em JSON.
- **SQL é cidadão de primeira classe**, mas nunca é obrigatório para o consumidor.
- **Segurança por padrão:** tudo é negado até ser concedido; queries de usuário rodam com privilégios mínimos.

---

## 1. Convenções da spec

- **IDs de feature:** `Fnn`. Requisito funcional: `RF-nn.x`. Regra de negócio: `RN-nn.x`. Critério de aceite: `CA-nn.x`.
- **Prioridade:**
  - `MVP` — primeira versão utilizável;
  - `V1` — primeira release pública estável;
  - `V2` — evolução.
- **Contratos:** os endpoints listados são o **contrato mínimo**. A fonte da verdade é o OpenAPI gerado em `/api/v1/openapi.json`.
  - Prefixo `/api/v1`.
  - JSON com `camelCase`.
  - IDs UUID.
  - Datas ISO-8601 em UTC.
- **Erros:** corpo de erro no padrão RFC 9457 (`application/problem+json`), com um campo `code` estável (ex.: `TABLE_FIELD_IN_USE`).
- **Paginação:** `?page=0&size=50&sort=campo,asc`. A resposta traz `{ content, page, size, totalElements }`.
- **Critérios de aceite** em formato Dado / Quando / Então.

---

## 2. Arquitetura de referência

> `[DECISÃO D-01]` — proposta alinhada à stack do mantenedor. Confirmar antes do scaffold.

```
┌──────────────────────── docker compose ───────────────────────────┐
│                                                                    │
│  ┌──────────────┐      ┌──────────────────────────────────────┐   │
│  │  openbi-app  │      │            postgres:16               │   │
│  │──────────────│ JDBC │──────────────────────────────────────│   │
│  │ Spring Boot  │─────▶│ schema app     → metadados (Flyway)  │   │
│  │ (API + jobs) │      │ schema lake    → dados do ETL (F08)  │   │
│  │ + React SPA  │      │ schema udt     → tabelas pers. (F09) │   │
│  │ (static)     │      │ schema audit   → auditoria (F06)     │   │
│  └──────┬───────┘      └──────────────────────────────────────┘   │
│         │ JDBC (somente leitura)                                   │
└─────────┼──────────────────────────────────────────────────────────┘
          ▼
   Bancos externos da empresa (ERP, CRM…) — F07
```

| Camada | Proposta |
|---|---|
| Backend | Java 21, Spring Boot 3.x, Spring Security, Spring Data JPA (metadados) + `JdbcTemplate`/jOOQ (queries dinâmicas), Flyway |
| Agendamento | Quartz com JobStore JDBC no Postgres (sem Redis/broker extra) |
| Frontend | React + TypeScript + Vite. Grid/drag-and-drop de layout: `react-grid-layout`. Gráficos: Apache ECharts (cobre cascata, sankey, treemap, calendário, mapa). Tabelas: TanStack Table |
| Banco interno | PostgreSQL 16 (metadados + lake + tabelas personalizadas) |
| Exportação | Apache POI (xlsx), OpenPDF ou Chromium headless para PDF de tela `[DECISÃO D-07]` |
| Imagem | Uma imagem Docker `openbi/openbi:<versão>` (SPA servida pelo Spring). Compose oficial com Postgres |

**Módulos do backend (pacotes):**

| Área | Módulos |
|---|---|
| Base | `platform`, `security`, `audit` |
| Dados | `datasource`, `etl`, `udt` (tabelas personalizadas), `importing` |
| Modelagem e consulta | `semantic`, `query` |
| Visualização | `dashboard`, `report` |
| Automação | `scheduler`, `alerting`, `notification` |
| Distribuição | `kiosk`, `publicapi` |

---

## 3. Glossário

| Termo | Definição |
|---|---|
| **Conexão** | Credenciais e parâmetros para acessar um banco externo. |
| **Lake** | Schema interno onde o ETL grava cópias de dados externos. |
| **Tabela Personalizada (UDT)** | Tabela criada pelo usuário dentro da plataforma, com campos tipados, referências e cálculos. |
| **Visão** | Dataset semântico (SQL ou construído sobre uma tabela) com colunas tipadas como dimensão/medida. É a fonte de todos os objetos. |
| **Tela** | Dashboard. Contém Objetos posicionados em grade. |
| **Objeto** | Elemento visual de uma tela: Valor, Consulta, Gráfico, Imagem, Texto. |
| **Token** | Variável resolvida em tempo de execução (`{{usuario.id}}`, `{{hoje}}`). |
| **LOV** | Lista de valores para filtros e campos de seleção. |
| **Construtor** | Usuário com permissão de criar/editar conteúdo. |

---

## 4. Mapa de features e roadmap

| ID | Feature | Prioridade | Depende de |
|---|---|---|---|
| F01 | Distribuição e Instalação (Docker) | MVP | — |
| F02 | Configurações Gerais e Identidade Visual | MVP (parcial) / V1 | F01 |
| F03 | Autenticação e Sessão | MVP | F01 |
| F04 | Usuários, Grupos e Permissões | MVP | F03 |
| F05 | Segurança em Nível de Linha | MVP | F04, F12, F15 |
| F06 | Auditoria e Logs | V1 (login/acesso no MVP) | F03 |
| F07 | Conexões de Dados | MVP | F04 |
| F08 | ETL e Data Lake Interno | V1 | F07, F33 |
| **F09** | **Tabelas Personalizadas** | **MVP** | F04 |
| F10 | Browser (Grade de Dados) | MVP | F09, F12 |
| F11 | Importação de Dados e Upload | MVP (planilha → UDT) / V1 | F09 |
| F12 | Visões e Colunas | MVP | F07, F09 |
| F13 | Categorias | MVP | — |
| F14 | Máscaras de Formatação | MVP | — |
| F15 | Tokens | MVP (tokens de sistema) / V1 | F03 |
| F16 | LOV | V1 | F12 |
| F17 | Editor SQL | MVP | F07 |
| F18 | Consulta Customizada | V1 | F12 |
| F19 | Telas | MVP | F12 |
| F20 | Objeto Valor | MVP | F19 |
| F21 | Objeto Consulta | MVP (tabela) / V1 (pivot) | F19 |
| F22 | Objeto Gráfico (básicos) | MVP | F19 |
| F23 | Gráficos Avançados | V2 | F22 |
| F24 | Objetos Estáticos | MVP | F19 |
| F25 | Linha Calculada | V1 | F21 |
| F26 | Destaques e Sinalizadores | V1 | F20, F21 |
| F27 | Drill Down / Through | V1 | F19 |
| F28 | Filtros | MVP (tela/objeto) / V1 (dinâmico, cascata) | F12, F16 |
| F29 | Exportação | MVP (CSV/XLSX) / V1 (PDF) | F19 |
| F30 | Menus e Tela Inicial | MVP | F19, F04 |
| F31 | Favoritos | V1 | F19 |
| F32 | Avisos | V2 | F04 |
| F33 | Agendador de Tarefas | V1 | F01 |
| F34 | Processos | V1 | F33 |
| F35 | Alertas e Ações | V1 | F33, F12 |
| F36 | Relatórios (Report) | V2 | F12 |
| F37 | Agendamento de Relatórios | V1 (tela/objeto) / V2 (report) | F33, F29 |
| F38 | Modo TV | V2 | F19 |
| F39 | API Pública | V2 | F04, F12 |

**Corte do MVP:** instalar → conectar banco ou criar tabelas personalizadas → alimentar dados (browser/planilha) → criar visão → montar tela com KPIs, tabelas e gráficos → filtrar → exportar → controlar quem vê o quê.

---

# PLATAFORMA

## F01 — Distribuição e Instalação (Docker)

**Prioridade:** MVP

### Objetivo

Qualquer empresa sobe a plataforma com um único comando, sem compilar nada, e atualiza de versão sem perder dados.

### Escopo

| Inclui | Não inclui |
|---|---|
| Imagem no Docker Hub | Kubernetes/Helm (V2) |
| `docker-compose.yml` oficial | Instalador ISO/VM |
| Setup inicial (wizard) | Alta disponibilidade multi-nó |
| Migrações automáticas | |
| Healthcheck | |
| Backup/restore documentado | |
| Página "Sobre/Atualizações" | |

> Nota: Docker Hub distribui **imagens**, não ISOs. O artefato será a imagem `openbi/openbi`.

### Requisitos funcionais

**Imagem e configuração**

- **RF-01.1** Imagem multi-arch (`amd64`, `arm64`) publicada como `openbi/openbi:<semver>` e `:latest`, gerada por CI a cada tag.
- **RF-01.2** Compose oficial com dois serviços: `app` e `postgres`. Os dados ficam em volumes nomeados (`openbi-db`, `openbi-files`).
- **RF-01.3** Toda configuração é feita por variáveis de ambiente:
  - `OPENBI_DB_URL`, `OPENBI_DB_USER`, `OPENBI_DB_PASSWORD`
  - `OPENBI_BASE_URL`
  - `OPENBI_SECRET_KEY` — usada para cifrar credenciais de conexões
  - `OPENBI_SMTP_*`
  - `OPENBI_TIMEZONE`
- **RF-01.4** Sem `OPENBI_SECRET_KEY`, a aplicação **não sobe** e mostra uma mensagem explicativa no log.

**Inicialização e atualização**

- **RF-01.5** No primeiro acesso, sem nenhum usuário cadastrado, o sistema exibe o **wizard de setup**: nome da empresa, idioma/fuso, criação do primeiro administrador e SMTP (opcional).
- **RF-01.6** Migrações Flyway rodam no startup. Se a versão do banco for **mais nova** que a da aplicação, o startup aborta (proteção contra downgrade).

**Operação**

- **RF-01.7** `GET /actuator/health` (liveness/readiness) é usado pelo `healthcheck` do compose.
- **RF-01.8** Backup é um `pg_dump` mais o volume de arquivos. A documentação traz o script `backup.sh`/`restore.sh` e o procedimento de upgrade.
- **RF-01.9** A página **Sobre** mostra a versão, o commit, a versão do schema e o link para o changelog.

### Regras de negócio

- **RN-01.1** O wizard só fica acessível enquanto `count(users) = 0`. Depois disso, a rota retorna 404.
- **RN-01.2** Credenciais de conexões externas (F07) são cifradas em repouso (AES-GCM) com chave derivada de `OPENBI_SECRET_KEY`. Se a chave mudar, as conexões ficam inválidas até que as senhas sejam recadastradas.

### Contrato

```
GET  /api/v1/setup/status   → { setupRequired: boolean }
POST /api/v1/setup          { companyName, locale, timezone,
                              admin: { name, email, password },
                              smtp?: {...} }
                            → 201
GET  /api/v1/system/info    → { version, commit, schemaVersion, buildDate }
```

### Critérios de aceite

- **CA-01.1** Dada uma máquina com Docker, quando executo `docker compose up -d` com o compose oficial, então em até 60 s `http://localhost:8080` exibe o wizard.
- **CA-01.2** Dado o sistema na versão 1.0 com dados, quando troco a tag para 1.1 e reinicio, então as migrações rodam e nenhum dado é perdido.
- **CA-01.3** Dado um setup já concluído, quando chamo `POST /api/v1/setup`, então recebo 404.

---

## F02 — Configurações Gerais e Identidade Visual

**Prioridade:** MVP (geral) / V1 (white-label)

### Objetivo

Adaptar a plataforma à empresa: identidade visual, formatos regionais e parâmetros globais. Equivale a "Configurações BI" e "Config" da UpQuery.

### Requisitos funcionais

- **RF-02.1** Configurações globais:
  - nome da empresa;
  - idioma padrão (pt-BR, en, es);
  - fuso horário;
  - formato de data e número (padrão pt-BR);
  - moeda padrão;
  - tamanho de página padrão das grades.
- **RF-02.2** White-label (V1):
  - logo (claro e escuro) e favicon;
  - cor primária, aplicada via CSS variables;
  - nome exibido na aba do navegador.
- **RF-02.3** Tema claro/escuro. O usuário escolhe; o padrão é definido pelo admin.
- **RF-02.4** Parâmetros de segurança (consumidos pela F03): política de senha, timeout de sessão, tentativas antes de bloqueio.
- **RF-02.5** Limites operacionais:
  - timeout de query (padrão 60 s);
  - máximo de linhas por consulta (padrão 100.000);
  - máximo de linhas exportadas.
- **RF-02.6** Configuração de SMTP com botão "enviar e-mail de teste".

### Regras de negócio

- **RN-02.1** Somente o perfil `ADMIN` altera configurações.
- **RN-02.2** Toda alteração de configuração gera um evento de auditoria (F06) com valor anterior e novo. Segredos (senha SMTP) são mascarados no log.

### Contrato

```
GET  /api/v1/settings
PUT  /api/v1/settings              (parcial por seção: general | branding | security | limits | smtp)
POST /api/v1/settings/smtp/test    { to }
POST /api/v1/settings/branding/logo   (multipart)
```

### Critérios de aceite

- **CA-02.1** Dado um admin, quando altera a cor primária, então todos os usuários veem a nova cor no próximo carregamento, sem rebuild.
- **CA-02.2** Dado o limite de linhas em 1.000, quando uma consulta retorna 5.000 linhas, então são exibidas 1.000 com o aviso "resultado truncado".

---

# SEGURANÇA

## F03 — Autenticação e Sessão

**Prioridade:** MVP (local) / V2 (OIDC/LDAP)

### Objetivo

Controlar o acesso com login, políticas de senha e de sessão, e bloqueio. Equivale a Login, Alteração de Senha, Bloqueios, Expirar Senha/Sessão/E-mail e Métodos de Autenticação da UpQuery.

### Requisitos funcionais

**Login e senha**

- **RF-03.1** Login com e-mail e senha. Hash com Argon2id (ou BCrypt cost ≥ 12).
- **RF-03.2** Sessão via cookie `HttpOnly`, `Secure`, `SameSite=Lax`, com proteção CSRF. `[DECISÃO D-05]` — alternativa: JWT stateless.
- **RF-03.3** Política de senha configurável: tamanho mínimo, complexidade, expiração em N dias (0 = nunca) e histórico das últimas N senhas.
- **RF-03.4** Troca de senha pelo próprio usuário (exige a senha atual) e troca forçada no próximo login (flag definida pelo admin).
- **RF-03.5** "Esqueci minha senha" por e-mail, com token de uso único válido por 30 min. Só fica disponível quando há SMTP configurado.

**Bloqueio e sessões**

- **RF-03.6** Bloqueio automático após N tentativas falhas (padrão 5) por X minutos (padrão 15). O admin pode desbloquear manualmente e também bloquear/inativar um usuário.
- **RF-03.7** Expiração de sessão por inatividade (padrão 30 min) e por duração máxima (padrão 12 h).
- **RF-03.8** Tela **Usuários Logados** (admin): sessões ativas com usuário, IP, user-agent, início e último acesso, e ação "encerrar sessão".

**Integrações**

- **RF-03.9** Verificação de e-mail (V1): convite por e-mail com link de ativação. Contas não verificadas não fazem login.
- **RF-03.10** (V2) Provedores externos: OIDC (Google, Microsoft, Keycloak) e LDAP/AD.

### Regras de negócio

- **RN-03.1** Mensagens de erro de login são genéricas ("credenciais inválidas"), para não revelar se o e-mail existe.
- **RN-03.2** Usuário bloqueado ou inativo tem todas as sessões encerradas imediatamente.
- **RN-03.3** Sempre existe pelo menos 1 admin ativo. O sistema impede inativar, bloquear ou remover o perfil do último admin.

### Contrato

```
POST   /api/v1/auth/login            { email, password }  → 200 + cookie | 401 | 423 (bloqueado)
POST   /api/v1/auth/logout
GET    /api/v1/auth/me               → { id, name, email, roles, permissions, preferences }
POST   /api/v1/auth/password/change  { currentPassword, newPassword }
POST   /api/v1/auth/password/forgot  { email }  → 202 (sempre)
POST   /api/v1/auth/password/reset   { token, newPassword }
GET    /api/v1/admin/sessions
DELETE /api/v1/admin/sessions/{id}
```

### Critérios de aceite

- **CA-03.1** Dadas 5 senhas erradas seguidas, quando tento a 6ª (mesmo correta), então recebo 423 até que expire o período de bloqueio.
- **CA-03.2** Dado o admin encerrando a sessão de João, quando João faz qualquer requisição, então recebe 401.
- **CA-03.3** Dada a expiração de senha em 90 dias e a última troca há 91 dias, quando o usuário faz login, então é redirecionado para a troca obrigatória.

---

## F04 — Usuários, Grupos e Permissões

**Prioridade:** MVP

### Objetivo

Definir quem pode fazer o quê e quem vê qual conteúdo. Equivale a Usuários, Grupos, Acessos, Telas Disponíveis, Liberação de Upload e Consulta Customizada – Liberação.

### Modelo

O modelo combina dois mecanismos:

1. **RBAC** — papéis globais definem o que o usuário pode *fazer*.
2. **ACL por recurso** — define o que o usuário pode *ver* e *editar*.

**Papéis globais (fixos):**

| Papel | Pode |
|---|---|
| `ADMIN` | Tudo, incluindo configurações, usuários, conexões e auditoria. |
| `BUILDER` | Criar e editar visões, tabelas personalizadas, telas, alertas e consultas — dentro do que tem acesso. |
| `VIEWER` | Consumir telas e alimentar dados em tabelas personalizadas onde tem permissão. |

**Permissões granulares (flags extras atribuíveis a usuário/grupo):**

| Flag | Libera |
|---|---|
| `SQL_EDITOR` | Uso do Editor SQL (F17) |
| `CUSTOM_QUERY` | Construtor de consultas (F18) |
| `UPLOAD` | Upload/importação de arquivos (F11) |
| `EXPORT` | Exportar dados (F29) |
| `MANAGE_UDT_SCHEMA` | Criar/alterar estrutura de tabelas personalizadas (F09) |

**ACL por recurso:** cada Tela, Visão, Tabela Personalizada, Pasta de menu, Conexão e Relatório tem uma lista de `{ principal: usuário|grupo, nível }`.

| Nível | Significado |
|---|---|
| `VIEW` | Visualizar |
| `EDIT` | Editar (inclui `VIEW`) |
| `MANAGE` | Editar + compartilhar + excluir |

Para tabelas personalizadas (F09) há níveis de dados adicionais: `DATA_READ`, `DATA_INSERT`, `DATA_UPDATE`, `DATA_DELETE`.

### Requisitos funcionais

**Usuários e grupos**

- **RF-04.1** CRUD de usuários: nome, e-mail, papel, grupos, status (ativo/inativo), flags e **atributos customizados** chave-valor (ex.: `vendedor_id=123`, `filial=SC01`), usados pela RLS (F05) e pelos tokens (F15).
- **RF-04.2** CRUD de grupos com membros. Um usuário pode estar em N grupos.

**Permissões efetivas**

- **RF-04.3** A permissão efetiva é a **união** das permissões do usuário com as de seus grupos. Não existe "deny" explícito no MVP.
- **RF-04.4** Tela "Acessos" por recurso: listar e editar quem tem acesso, com herança da pasta do menu (o recurso herda a ACL da pasta, salvo se tiver ACL própria).
- **RF-04.5** Tela "Telas disponíveis" por usuário: mostra o resultado efetivo, ou seja, o que aquele usuário enxerga. Serve para auditoria e debug.
- **RF-04.6** O criador de um recurso recebe `MANAGE` automaticamente.

**Administração**

- **RF-04.7** Listagem de usuários com filtros (status, grupo, papel) e coluna "último acesso". Inclui a ação "verificar e-mails cadastrados" (lista e-mails duplicados, inválidos ou não verificados).

### Regras de negócio

- **RN-04.1** `ADMIN` ignora ACL (vê tudo).
- **RN-04.2** `VIEWER` nunca recebe `EDIT`/`MANAGE` em telas e visões, mesmo que a ACL conceda. A ACL é limitada pelo papel.
- **RN-04.3** Excluir um grupo remove suas entradas de ACL. Excluir um usuário é soft delete (preserva a auditoria).

### Contrato

```
GET|POST         /api/v1/users
GET|PUT|DELETE   /api/v1/users/{id}
PUT              /api/v1/users/{id}/attributes   { "vendedor_id": "123", ... }
GET|POST         /api/v1/groups
GET|PUT|DELETE   /api/v1/groups/{id}
PUT              /api/v1/groups/{id}/members     { userIds: [] }
GET              /api/v1/acl/{resourceType}/{resourceId}
PUT              /api/v1/acl/{resourceType}/{resourceId}
                 { entries: [{ principalType, principalId, level }] }
GET              /api/v1/users/{id}/effective-access
```

### Critérios de aceite

- **CA-04.1** Dado Pedro no grupo "Comercial" com `VIEW` na tela "Vendas", quando Pedro abre o menu, então vê "Vendas". Quando é removido do grupo, deixa de ver.
- **CA-04.2** Dado um VIEWER, quando a ACL lhe dá `EDIT` numa tela, então ele continua sem conseguir editá-la (403).

---

## F05 — Segurança em Nível de Linha (Filtro de Usuários)

**Prioridade:** MVP

### Objetivo

Garantir que cada usuário veja apenas as linhas a que tem direito em qualquer visão — independentemente da tela, do filtro ou da exportação. Exemplo: vendedor vê só a própria carteira; gerente vê a filial.

### Requisitos funcionais

- **RF-05.1** Uma **regra RLS** é definida por Visão (F12). Ela tem:
  - nome;
  - alvo (usuários e/ou grupos);
  - **predicado** sobre colunas da visão, usando tokens (F15). Ex.: `vendedor_id = {{usuario.atributo.vendedor_id}}` ou `filial IN ({{usuario.atributo.filiais}})`.
- **RF-05.2** O predicado pode ser escrito:
  - no **modo simples** — coluna + operador + token/valor, combinados com E/OU;
  - no **modo avançado** — expressão SQL, restrita a quem tem `SQL_EDITOR`.
- **RF-05.3** Combinação das regras:
  - várias regras que se aplicam ao mesmo usuário combinam com **OU** entre si;
  - o resultado é aplicado com **E** sobre a query final.
- **RF-05.4** Cada visão escolhe o comportamento padrão para quem não tem nenhuma regra aplicável: `ALLOW_ALL` ou `DENY_ALL` (padrão: `DENY_ALL` quando a visão tem pelo menos uma regra).
- **RF-05.5** Pré-visualização: "ver como usuário X" — executa a visão aplicando as regras de X (somente ADMIN).

### Regras de negócio

- **RN-05.1** A RLS é aplicada **no backend**, na montagem da query (wrapping: `SELECT * FROM (<visão>) v WHERE <predicado>`). Nunca é aplicada no frontend.
- **RN-05.2** A RLS vale para todos os caminhos de saída: objetos, drill, exportação, API, alertas e relatórios agendados. Os dois últimos rodam **no contexto do destinatário**.
- **RN-05.3** Token sem valor para o usuário (atributo ausente) gera um predicado falso (`1=0`). Nunca é ignorado.
- **RN-05.4** `ADMIN` também é filtrado, salvo se a visão tiver a opção "admin ignora RLS" marcada.
- **RN-05.5** Os valores são passados por **bind parameters**, nunca por concatenação.

### Contrato

```
GET|POST        /api/v1/views/{viewId}/rls-rules
PUT|DELETE      /api/v1/views/{viewId}/rls-rules/{ruleId}
POST            /api/v1/views/{viewId}/preview?asUser={userId}
```

### Critérios de aceite

- **CA-05.1** Dada a regra `vendedor_id = {{usuario.atributo.vendedor_id}}` e Ana com `vendedor_id=7`, quando Ana abre qualquer tela, exporta ou faz drill, então só aparecem linhas com `vendedor_id=7`.
- **CA-05.2** Dado Bruno sem o atributo `vendedor_id`, quando abre a tela, então vê 0 linhas.
- **CA-05.3** Dado um relatório agendado para Ana e Bruno, quando é enviado, então cada um recebe apenas os seus dados.

---

## F06 — Auditoria e Logs

**Prioridade:** MVP (login e acesso) / V1 (completo)

### Objetivo

Rastreabilidade de quem acessou, consultou, alterou ou exportou o quê.

### Requisitos funcionais

- **RF-06.1** Eventos auditados:

  | Categoria | Eventos |
  |---|---|
  | Autenticação | login (sucesso/falha), logout |
  | Acesso | abertura de tela |
  | Consulta | execução de consulta (visão, duração, linhas, hash do SQL) |
  | Exportação | exportação (formato, linhas) |
  | Metadados | criação, alteração e exclusão de qualquer metadado (diff JSON) |
  | Dados (F09) | alteração de dados em tabela personalizada (linha, campo, antes/depois) |
  | Segurança | alteração de ACL/RLS, alteração de configuração |
  | Automação | execução de jobs |

- **RF-06.2** Tela de consulta de auditoria (admin): filtros por período, usuário, tipo de evento e recurso; exportação CSV.
- **RF-06.3** Histórico por recurso: em cada tela, visão ou tabela, aba "Histórico" com as alterações, com a opção de **restaurar versão** para metadados (V1).
- **RF-06.4** Retenção configurável (padrão 365 dias). Um job diário (F33) faz a limpeza.
- **RF-06.5** Logs técnicos da aplicação em stdout, no formato JSON estruturado (para quem usa Loki/ELK).

### Regras de negócio

- **RN-06.1** A auditoria é append-only. Nem o ADMIN edita ou apaga registros, exceto via job de retenção.
- **RN-06.2** Valores de parâmetros sensíveis (senhas, tokens) nunca são gravados.
- **RN-06.3** A gravação é assíncrona (fila em memória + batch). Falha de auditoria não derruba a requisição, mas gera log de erro e métrica.

### Contrato

```
GET /api/v1/audit/events?from&to&userId&type&resourceType&resourceId
GET /api/v1/{resourceType}/{id}/history
POST /api/v1/{resourceType}/{id}/history/{versionId}/restore
```

### Critérios de aceite

- **CA-06.1** Dada a alteração de um valor numa tabela personalizada, quando abro o histórico da linha, então vejo usuário, data/hora, campo, valor antigo e valor novo.

---

# DADOS

## F07 — Conexões de Dados

**Prioridade:** MVP

### Objetivo

Cadastrar bancos externos (ERP, CRM, sistemas legados) para consulta direta ou para ETL.

### Requisitos funcionais

- **RF-07.1** Tipos suportados:

  | Fase | Bancos |
  |---|---|
  | MVP | PostgreSQL, MySQL/MariaDB, SQL Server |
  | V1 | Oracle, Firebird (comum em ERPs brasileiros) |
  | V2 | SQLite, ClickHouse, BigQuery |

  Os drivers JDBC vêm embarcados na imagem. Os de licença restritiva podem ser montados via volume `/opt/openbi/drivers`.
- **RF-07.2** Campos: nome, tipo, host, porta, banco, usuário, senha (cifrada), parâmetros JDBC extras, SSL on/off, timezone da origem, pool máx.
- **RF-07.3** Botão "testar conexão": retorna sucesso, latência e versão do servidor.
- **RF-07.4** Explorador de schema: lista schemas, tabelas e colunas com seus tipos, com cache e botão "atualizar metadados".
- **RF-07.5** Modo de uso por conexão:
  - `LIVE` — visões consultam direto a origem;
  - `ETL_ONLY` — só é usada como fonte de cargas (F08).
- **RF-07.6** Existe uma **conexão interna** implícita (`internal`), não editável, que aponta para o Postgres da plataforma (schemas `lake` e `udt`).

### Regras de negócio

- **RN-07.1** Recomenda-se um usuário de banco somente leitura. Para conexões `LIVE`, o sistema executa apenas `SELECT`/`WITH` (validado por parser — JSqlParser).
- **RN-07.2** Um pool por conexão (HikariCP), com tamanho limitado, para não derrubar o ERP do cliente.
- **RN-07.3** Senha nunca é retornada pela API (`"password": "********"`). No update, um campo vazio mantém a senha atual.
- **RN-07.4** Não é possível excluir uma conexão usada por visões ou cargas. A API retorna `409 DATASOURCE_IN_USE` com a lista de dependentes.

### Contrato

```
GET|POST         /api/v1/datasources
GET|PUT|DELETE   /api/v1/datasources/{id}
POST             /api/v1/datasources/test        (body = dados da conexão, sem salvar)
GET              /api/v1/datasources/{id}/schema?refresh=false
                 → [{ schema, tables: [{ name, columns: [{ name, type, nullable }] }] }]
```

### Critérios de aceite

- **CA-07.1** Dada uma conexão SQL Server válida, quando clico em "testar", então vejo sucesso e versão em < 5 s.
- **CA-07.2** Dada uma visão LIVE com `DELETE FROM x`, quando salvo, então recebo erro `ONLY_SELECT_ALLOWED`.

---

## F08 — ETL e Data Lake Interno

**Prioridade:** V1

### Objetivo

Copiar dados das conexões externas para o schema `lake` do Postgres interno. Isso:

- tira carga do ERP;
- permite cruzar dados de bancos diferentes;
- permite referenciar esses dados nas Tabelas Personalizadas (F09).

### Requisitos funcionais

- **RF-08.1** Uma **Carga** tem:
  - conexão de origem;
  - fonte: tabela inteira ou query SQL;
  - tabela destino no `lake` (nome amigável → nome físico `lake.<slug>`);
  - modo: `FULL` (truncate + insert) ou `INCREMENTAL` (por coluna de controle: data de alteração ou ID crescente, com chave para upsert);
  - agendamento (F33).
- **RF-08.2** Mapeamento de colunas: incluir/excluir, renomear e converter tipo. A estrutura da tabela destino é criada automaticamente na primeira execução e evolui com novas colunas (`ALTER TABLE ADD`). Colunas removidas na origem viram nulas.
- **RF-08.3** Execução em lotes (`fetchSize` e batch insert configuráveis), com a carga `FULL` feita em uma tabela temporária + swap atômico (sem janela de tabela vazia).
- **RF-08.4** Histórico de execuções: início, fim, linhas lidas/gravadas, status e erro. É exibido em "Tarefas Executadas" (F33).
- **RF-08.5** "Executar agora" manual e "executar a partir de" (reprocessamento incremental).
- **RF-08.6** Dependências entre cargas (a carga B roda após a A), mais o disparo de Processos (F34) após a conclusão.
- **RF-08.7** As tabelas do `lake` aparecem para a criação de Visões (F12) e como alvo de **referência** em Tabelas Personalizadas (F09).

### Regras de negócio

- **RN-08.1** Uma carga não roda em paralelo com ela mesma (lock por carga).
- **RN-08.2** Uma falha no meio de uma carga `FULL` mantém a versão anterior da tabela intacta.
- **RN-08.3** A tabela destino é gerida pela plataforma. Usuários não alteram a sua estrutura manualmente.

### Contrato

```
GET|POST         /api/v1/etl/jobs
GET|PUT|DELETE   /api/v1/etl/jobs/{id}
POST             /api/v1/etl/jobs/{id}/run          { fromValue? }
GET              /api/v1/etl/jobs/{id}/runs
POST             /api/v1/etl/jobs/preview           (fonte + mapeamento → 100 primeiras linhas)
```

### Critérios de aceite

- **CA-08.1** Dada uma carga FULL agendada a cada hora, quando uma execução falha na metade, então as telas continuam mostrando os dados da execução anterior.
- **CA-08.2** Dada uma carga INCREMENTAL por `updated_at`, quando rodo duas vezes sem mudanças na origem, então a segunda grava 0 linhas.

---

## F09 — Tabelas Personalizadas

**Prioridade:** MVP ⭐ — feature diferencial

### Objetivo

Permitir que o usuário **crie suas próprias estruturas de dados** dentro da plataforma, sem SQL. São "tabelas" com campos tipados, **referências** a outras tabelas (personalizadas ou do lake), **campos calculados** e **agregações** de registros relacionados.

Casos de uso típicos de PME:

- metas por vendedor/mês;
- orçamento;
- de-para de produtos;
- cadastros que o ERP não tem (ex.: classificação de clientes);
- controles operacionais simples (ocorrências, checklists, apontamentos).

Essas tabelas se tornam fontes de dados para dashboards como qualquer outra.

### 9.1 Conceitos

| Conceito | Descrição |
|---|---|
| **Tabela** | Nome, descrição, categoria, ícone. Possui campos e registros. |
| **Campo** | Coluna tipada, com regras de validação. |
| **Registro** | Linha da tabela. Tem `id` (UUID) e metadados automáticos. |
| **Campo de exibição** | Campo usado para representar o registro quando ele é referenciado (ex.: `nome` do cliente). |

### 9.2 Tipos de campo

| Tipo | Armazenado? | Descrição | Opções |
|---|---|---|---|
| `TEXT` | Sim | Texto curto | maxLength, regex, único |
| `LONG_TEXT` | Sim | Texto longo / multilinha | — |
| `INTEGER` | Sim | Inteiro | min, max, único |
| `DECIMAL` | Sim | Decimal | casas decimais (0–6), min, max |
| `CURRENCY` | Sim | Valor monetário (`numeric(18,2)`) | moeda |
| `PERCENT` | Sim | Percentual (armazena fração: 0.15 = 15%) | casas |
| `BOOLEAN` | Sim | Sim/Não | — |
| `DATE` | Sim | Data | min, max |
| `DATETIME` | Sim | Data e hora (UTC) | — |
| `SELECT` | Sim | Uma opção de uma lista fixa | opções (valor, rótulo, cor) |
| `MULTI_SELECT` | Sim | Várias opções | opções |
| `EMAIL` / `URL` / `PHONE` | Sim | Texto com validação de formato | — |
| `FILE` / `IMAGE` | Sim (referência ao arquivo) | Anexo armazenado no volume de arquivos | tipos aceitos, tamanho máx. |
| `AUTONUMBER` | Sim | Sequencial gerado automaticamente | prefixo, zeros à esquerda |
| `REFERENCE` | Sim (chave) | **Aponta para um registro de outra tabela** | tabela alvo, cardinalidade, ação ao excluir |
| `LOOKUP` | Não (derivado) | Traz um campo do registro referenciado | campo de referência + campo alvo |
| `FORMULA` | Não (derivado) | **Campo calculado** por expressão | expressão, tipo de resultado |
| `ROLLUP` | Não (derivado) | **Agrega registros relacionados** (filhos) | relação inversa, função, campo, filtro |
| `CREATED_AT` / `UPDATED_AT` / `CREATED_BY` / `UPDATED_BY` | Sim (sistema) | Metadados automáticos, sempre presentes | — (podem ser ocultados) |

### 9.3 Referências

- **RF-09.1** Um campo `REFERENCE` aponta para:
  - **(a)** outra Tabela Personalizada (inclusive a própria — auto-referência para hierarquias); ou
  - **(b)** uma tabela do `lake` (F08), escolhendo a coluna-chave.
- **RF-09.2** Cardinalidade:

  | Fase | Cardinalidade | Implementação |
  |---|---|---|
  | MVP | `MANY_TO_ONE` | FK simples (ex.: Meta → Vendedor) |
  | V1 | `MANY_TO_MANY` | Tabela de junção gerenciada automaticamente |

- **RF-09.3** Ação ao excluir o registro referenciado (só para referência a UDT): `RESTRICT` (padrão), `SET_NULL` ou `CASCADE`.
- **RF-09.4** Toda referência cria automaticamente uma **relação inversa** navegável na tabela alvo (ex.: Vendedor → "Metas"), usada por `ROLLUP` e mostrada no Browser (F10) como sub-lista.
- **RF-09.5** Na edição (F10), um campo de referência é exibido como um combo pesquisável, com o **campo de exibição** da tabela alvo e paginação no servidor.

**Regras de integridade:**

- **RN-09.1** Referência a UDT → FK física com a ação configurada. Integridade garantida pelo banco.
- **RN-09.2** Referência a tabela do `lake` → **referência suave**: o valor da chave é validado na gravação (deve existir no lake no momento). Não há FK física, porque as cargas `FULL` recriam as tabelas. Registros "órfãos" após uma carga são sinalizados no Browser com um ícone de alerta e um filtro "referências quebradas".

### 9.4 Campos calculados (FORMULA)

- **RF-09.6** Linguagem de expressão própria (**OBExpr**), compilada para SQL do Postgres:
  - **Referência a campo:** `{quantidade}`, `{preco_unitario}`.
  - **Navegação por referência:** `{cliente.estado}`, `{pedido.cliente.nome}` — até 3 níveis.
  - **Operadores:**

    | Tipo | Operadores |
    |---|---|
    | Aritméticos | `+ - * / %` |
    | Comparação | `= <> > >= < <=` |
    | Lógicos | `AND OR NOT` |
    | Texto | `&` (concatenação) |

  - **Funções:**

    | Grupo | Funções |
    |---|---|
    | Lógicas | `IF(cond, a, b)`, `IFS(...)`, `COALESCE(...)`, `ISBLANK(x)` |
    | Matemáticas | `ROUND(x, n)`, `ABS`, `MIN`, `MAX`, `POWER` |
    | Texto | `CONCAT`, `UPPER`, `LOWER`, `LEFT`, `RIGHT`, `LEN`, `TRIM`, `REPLACE` |
    | Data | `TODAY()`, `NOW()`, `YEAR`, `MONTH`, `DAY`, `DATEADD(d, n, 'day'\|'month'\|'year')`, `DATEDIFF(a, b, unidade)`, `FORMAT_DATE` |
    | Conversão | `TO_NUMBER`, `TO_TEXT` |

  - **Exemplos:**
    - `{quantidade} * {preco_unitario}`
    - `IF({realizado} >= {meta}, "Atingida", "Abaixo")`
    - `DATEDIFF({data_entrega}, {data_pedido}, 'day')`
- **RF-09.7** O tipo do resultado é inferido e pode ser forçado. Uma máscara (F14) pode ser aplicada.
- **RF-09.8** O editor de fórmula oferece autocomplete de campos e funções, validação em tempo real (erro com posição) e pré-visualização do resultado nas 5 primeiras linhas.

### 9.5 Agregações de relacionados (ROLLUP)

- **RF-09.9** `ROLLUP` agrega os registros que referenciam o registro atual:
  - **Funções:** `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `COUNT_DISTINCT`.
  - **Filtro opcional** em OBExpr sobre a tabela filha.
  - **Exemplo:** na tabela *Pedido*, `total = SUM(itens.valor_total)` e `qtd_itens = COUNT(itens)`, filtrando `{itens.cancelado} = false`.
- **RF-09.10** Um `FORMULA` pode usar `LOOKUP`s e `ROLLUP`s, e vice-versa (ex.: `ROLLUP` sobre um `FORMULA` da tabela filha).

### 9.6 Ciclo de vida da estrutura (schema)

- **RF-09.11** Criar tabela:
  - do zero;
  - a partir de uma planilha (F11) — infere os campos;
  - duplicando a estrutura de outra tabela.
- **RF-09.12** Adicionar, renomear e reordenar campos. Também alterar validações, descrição e o campo de exibição.
- **RF-09.13** Alterar o tipo de um campo armazenado:
  1. O sistema executa uma **simulação de conversão** e informa quantos registros falhariam.
  2. O usuário escolhe: cancelar, ou converter (valores inválidos viram nulos).
- **RF-09.14** Excluir campo ou tabela:
  - É bloqueado se houver dependentes (fórmulas, lookups, rollups, referências, visões, objetos, filtros, RLS). A API responde `409 UDT_FIELD_IN_USE` com a lista de dependentes.
  - Sem dependentes, a exclusão é **lógica**: vai para a lixeira por 30 dias, restaurável. A exclusão física é feita pelo job de limpeza.
- **RF-09.15** Exportar/importar a definição de uma tabela em JSON (para levar entre instâncias) — V1.

### 9.7 Validações de dados

- **RF-09.16** Validações por campo:
  - obrigatório;
  - único (inclusive **unicidade composta** de N campos — ex.: vendedor + mês);
  - mínimo/máximo;
  - regex;
  - valor padrão (literal ou OBExpr, ex.: `TODAY()`).
- **RF-09.17** Regras de validação da linha (V1): uma expressão OBExpr booleana + mensagem de erro. Ex.: `{data_fim} >= {data_inicio}` → "Data fim deve ser posterior ao início".

### 9.8 Permissões de dados

- **RF-09.18** A ACL da tabela (F04) controla:
  - a estrutura: `EDIT`/`MANAGE` + flag `MANAGE_UDT_SCHEMA`;
  - os dados: `DATA_READ`, `DATA_INSERT`, `DATA_UPDATE`, `DATA_DELETE`.
- **RF-09.19** Opção "usuário só edita os registros que criou" (por tabela).
- **RF-09.20** As regras de RLS (F05) também se aplicam às tabelas personalizadas, via a visão gerada automaticamente (9.10).

### 9.9 Armazenamento — decisão técnica `[DECISÃO D-02]`

**Proposta: tabelas físicas geradas por DDL** no schema `udt`.

**Mapeamento físico:**

| Elemento | Nome físico |
|---|---|
| Tabela | `udt.t_<id curto>` (ex.: `udt.t_8f3a2c`) |
| Campo armazenado | `f_<id curto>` |

- Os nomes físicos são **imutáveis**. Renomear afeta só o metadado (`app.udt_field.label`/`api_name`), sem DDL.
- Tipos nativos do Postgres; `MULTI_SELECT` vira `text[]`.
- `REFERENCE` vira uma coluna `uuid` + FK.
- O DDL é executado numa transação junto da atualização do metadado. O Postgres suporta DDL transacional.

**Campos derivados** (`FORMULA`, `LOOKUP`, `ROLLUP`) **não são armazenados**. Eles são compilados para uma **view de leitura** `udt.v_<id>` com:

- `LEFT JOIN`s para lookups e navegação por referência;
- subqueries correlacionadas ou `LATERAL` para rollups.

A view é regenerada a cada mudança de estrutura.

**Alternativas avaliadas:**

| Abordagem | Prós | Contras |
|---|---|---|
| **DDL físico (proposta)** | Performance nativa, índices, FK real, consultável por SQL/visões como qualquer tabela | Exige gestão de DDL e migração de tipo; nomes físicos opacos |
| JSONB (uma tabela `records(table_id, data jsonb)`) | Zero DDL, schema flexível | Sem FK, índices limitados, agregações lentas em volume, SQL de visão complexo |
| EAV | Flexível | Péssimo para BI (pivot em toda consulta); descartado |

**Desempenho:**

- `ROLLUP` sobre tabelas grandes pode ficar lento na view. Em V1, oferecer a opção **"materializar"**: o valor é persistido numa coluna e mantido por trigger ou job.
- Índices são criados automaticamente em FKs e em campos marcados como `único`. Índice manual por campo é opção avançada (V1).

### 9.10 Integração com o resto da plataforma

- **RF-09.21** Toda tabela personalizada gera e mantém automaticamente uma **Visão** (F12) "do sistema":
  - todos os campos, inclusive os derivados;
  - nomes amigáveis;
  - dimensões e medidas inferidas: numéricos viram medidas com `SUM`, os demais viram dimensões.

  Essa visão pode ser usada imediatamente em Telas. O usuário pode criar visões adicionais com joins.
- **RF-09.22** Tabelas personalizadas podem ser alimentadas:
  - pelo Browser (F10);
  - por importação de planilha (F11);
  - pela API (F39);
  - por Processos (F34).

### Regras de negócio (resumo)

- **RN-09.3** **Ciclos são proibidos.** Ao salvar um campo derivado, o sistema monta o grafo de dependências (campo → campos usados, atravessando referências) e rejeita ciclos com `422 UDT_CIRCULAR_DEPENDENCY`, indicando o caminho do ciclo.
- **RN-09.4** O nome técnico (`apiName`, `snake_case`) é único por tabela e gerado a partir do rótulo. É editável enquanto não houver dependentes.
- **RN-09.5** Limites padrão, configuráveis em F02:
  - 200 campos por tabela;
  - 3 níveis de navegação em fórmulas;
  - 10 rollups por tabela.
- **RN-09.6** Toda escrita de dados passa por uma camada de serviço que aplica validações, permissões e auditoria. Nada escreve direto no schema `udt`, exceto Processos (F34), que são explicitamente marcados como bypass e restritos a ADMIN.

### Contrato

```
# Estrutura
GET|POST         /api/v1/udt/tables
GET|PUT|DELETE   /api/v1/udt/tables/{tableId}
POST             /api/v1/udt/tables/{tableId}/fields
PUT|DELETE       /api/v1/udt/tables/{tableId}/fields/{fieldId}
POST             /api/v1/udt/tables/{tableId}/fields/{fieldId}/type-change/simulate  { newType, options }
GET              /api/v1/udt/tables/{tableId}/dependencies
POST             /api/v1/udt/expressions/validate   { tableId, expression }
                 → { valid, resultType, errors: [{ message, start, end }], preview: [...] }

# Dados
GET    /api/v1/udt/tables/{tableId}/records?filter=<OBExpr>&sort=&page=&size=&fields=
GET    /api/v1/udt/tables/{tableId}/records/{recordId}
POST   /api/v1/udt/tables/{tableId}/records             { values: { apiName: valor, ... } }
PATCH  /api/v1/udt/tables/{tableId}/records/{recordId}  { values: {...}, version }   # otimista
DELETE /api/v1/udt/tables/{tableId}/records/{recordId}
POST   /api/v1/udt/tables/{tableId}/records/bulk        { insert: [], update: [], delete: [] }  # atômico
GET    /api/v1/udt/tables/{tableId}/records/{recordId}/history
GET    /api/v1/udt/tables/{tableId}/reference-options/{fieldId}?q=&page=   # combo de referência
```

**Exemplo de definição de campo:**

```json
{
  "label": "Valor total",
  "apiName": "valor_total",
  "type": "FORMULA",
  "config": {
    "expression": "{quantidade} * {preco_unitario} * (1 - {desconto})",
    "resultType": "CURRENCY",
    "maskId": "brl"
  }
}
```

```json
{
  "label": "Cliente",
  "apiName": "cliente",
  "type": "REFERENCE",
  "required": true,
  "config": {
    "target": { "kind": "UDT", "tableId": "..." },
    "cardinality": "MANY_TO_ONE",
    "onDelete": "RESTRICT",
    "inverseLabel": "Pedidos"
  }
}
```

### Critérios de aceite

- **CA-09.1** Dadas as tabelas *Vendedor (nome)* e *Meta (vendedor → Vendedor, mes DATE, valor CURRENCY)* com unicidade `vendedor+mes`, quando cadastro duas metas para o mesmo vendedor e mês, então a segunda é rejeitada com mensagem clara.
- **CA-09.2** Dado *Item (pedido → Pedido, qtd, preco, total = {qtd}*{preco})* e *Pedido.total = SUM(itens.total)*, quando altero a `qtd` de um item, então o total do pedido reflete a alteração na próxima leitura.
- **CA-09.3** Dado o campo A = `{B} + 1` e B = `{A} * 2`, quando salvo B, então recebo `UDT_CIRCULAR_DEPENDENCY` com o caminho `B → A → B`.
- **CA-09.4** Dado um campo usado por uma fórmula e por um objeto de tela, quando tento excluí-lo, então recebo 409 listando a fórmula e o objeto.
- **CA-09.5** Dado um campo TEXT com valores "10", "abc" e "20", quando simulo a conversão para INTEGER, então o sistema informa "1 registro não pode ser convertido".
- **CA-09.6** Dada uma nova tabela personalizada, quando salvo, então sua visão aparece imediatamente disponível para uso em Telas.
- **CA-09.7** Dada uma referência a `lake.clientes` e uma carga FULL que remove o cliente 42, quando abro o Browser, então os registros que apontam para 42 aparecem sinalizados como "referência quebrada".

---

## F10 — Browser (Grade de Dados)

**Prioridade:** MVP

### Objetivo

Interface de grade para **navegar e editar dados**. É a porta de entrada de dados nas Tabelas Personalizadas e também uma listagem de consulta sobre qualquer Visão. Equivale ao "Browser" da UpQuery.

### Requisitos funcionais

**Configuração e navegação**

- **RF-10.1** Um Browser é um recurso salvo com:
  - fonte: tabela personalizada **ou** visão (visão = somente leitura);
  - colunas visíveis, ordem e largura;
  - ordenação e filtros padrão;
  - tamanho de página.
- **RF-10.2** Cada tabela personalizada tem um **Browser padrão** criado automaticamente. Browsers adicionais podem ser criados com recortes diferentes (ex.: "Metas 2026 – Sul").
- **RF-10.3** Funções da grade:
  - paginação no servidor;
  - ordenação por coluna;
  - filtro por coluna, conforme o tipo (texto contém, intervalo numérico/data, seleção);
  - busca global;
  - congelar colunas;
  - ocultar/exibir colunas (preferência pessoal).

**Edição (tabela personalizada + permissão)**

- **RF-10.4** Modos de edição:
  - **edição inline** na célula;
  - **formulário** de registro (modal/lateral) com campos agrupáveis em seções;
  - **inclusão rápida** de linha.
- **RF-10.5** Colar do Excel: colar um bloco de células atualiza ou insere em lote, com pré-validação e relatório de erros por linha (V1).
- **RF-10.6** No formulário do registro, as **relações inversas** aparecem como sub-grades (ex.: Pedido → Itens) com inclusão inline (master-detail).
- **RF-10.7** Seleção múltipla com ações em lote: excluir e alterar o valor de um campo.
- **RF-10.8** Campos derivados (`FORMULA`, `LOOKUP`, `ROLLUP`) aparecem como somente leitura, com indicação visual. São recalculados após salvar.
- **RF-10.9** Histórico do registro (F06) acessível pelo formulário.
- **RF-10.10** Exportar a grade (F29) respeitando os filtros atuais.

### Regras de negócio

- **RN-10.1** Concorrência otimista: cada registro tem `version`. Um conflito retorna 409, e a UI oferece "recarregar" ou "sobrescrever".
- **RN-10.2** As permissões de dados (F09 §9.8) controlam a habilitação dos botões **e** são revalidadas no backend.
- **RN-10.3** Erros de validação retornam por campo (`errors: [{ field, code, message }]`) e são exibidos inline.

### Contrato

```
GET|POST         /api/v1/browsers
GET|PUT|DELETE   /api/v1/browsers/{id}
GET              /api/v1/browsers/{id}/data?page&size&sort&filters=<json>&q=
# Escrita usa os endpoints de records da F09.
```

### Critérios de aceite

- **CA-10.1** Dado um VIEWER com `DATA_READ`+`DATA_UPDATE` (sem `DATA_INSERT`), quando abre o Browser, então pode editar células, mas o botão "Novo" fica oculto, e um `POST` direto na API retorna 403.
- **CA-10.2** Dados dois usuários editando o mesmo registro, quando o segundo salva, então recebe aviso de conflito.

---

## F11 — Importação de Dados e Upload

**Prioridade:** MVP (planilha → tabela personalizada) / V1 (modelos, anexos)

### Objetivo

Carregar dados de planilhas para dentro da plataforma, de forma pontual ou recorrente. Equivale a Importar Planilha, Modelo de Importação, Upload de Arquivos e Liberação de Upload.

### Requisitos funcionais

**Fluxo de importação (wizard)**

- **RF-11.1** Etapas do wizard:
  1. **Upload** do arquivo (`.xlsx`, `.xls`, `.csv` — com detecção de separador e encoding) e escolha da aba.
  2. **Destino:**
     - nova tabela personalizada — infere nome e tipo dos campos, editáveis antes de confirmar;
     - **ou** tabela existente.
  3. **Mapeamento** coluna da planilha → campo, com conversões (formato de data, separador decimal).
  4. **Modo:**
     - `INSERT` — só adiciona;
     - `UPSERT` — atualiza por uma chave escolhida, insere o resto;
     - `REPLACE` — apaga tudo e insere. Exige confirmação e permissão `DATA_DELETE`.
  5. **Pré-validação:** linhas válidas, linhas com erro (motivo por célula) e opção de baixar um CSV de erros.
  6. **Confirmação** e execução.
- **RF-11.2** Referências na importação: uma coluna com o **valor de exibição** (ex.: nome do vendedor) é resolvida para o ID na tabela alvo. Valores não encontrados viram erro, ou "criar registro na tabela alvo" (opcional, V1).
- **RF-11.3** **Modelo de Importação** (V1): salvar o mapeamento, modo e conversões com um nome, para reusar em importações recorrentes. Inclui a opção de baixar uma planilha-modelo com os cabeçalhos corretos.
- **RF-11.4** Histórico de importações: quem, quando, arquivo, contagens e opção de baixar o arquivo original (retido por 30 dias).

**Arquivos**

- **RF-11.5** Upload de arquivos para campos `FILE`/`IMAGE` e para objetos de imagem (F24). Os arquivos ficam no volume `openbi-files` (V2: compatível com S3).

### Regras de negócio

- **RN-11.1** A importação exige a flag `UPLOAD` mais as permissões de dados correspondentes na tabela.
- **RN-11.2** A importação é **atômica** por padrão: um erro aborta tudo. Há a opção "importar só as válidas".
- **RN-11.3** Limites:

  | Item | Padrão |
  |---|---|
  | Tamanho de arquivo | 20 MB |
  | Linhas por importação | 200.000 |
  | Extensões de upload | Whitelist |
  | Imagens | Validação de MIME real |

- **RN-11.4** Importações grandes rodam de forma assíncrona (job F33), com progresso exibido na UI.

### Contrato

```
POST /api/v1/imports/upload                  (multipart) → { importId, sheets: [...], preview: [...] }
POST /api/v1/imports/{importId}/analyze      { sheet, target, mapping, mode, keyFields }
                                             → { validRows, errorRows, errors: [...] }
POST /api/v1/imports/{importId}/execute      → { jobId }
GET  /api/v1/imports/{importId}
GET|POST       /api/v1/import-templates
GET|PUT|DELETE /api/v1/import-templates/{id}
POST /api/v1/files                           (multipart) → { fileId, url }
```

### Critérios de aceite

- **CA-11.1** Dada uma planilha de metas com a coluna "Vendedor" (nomes), quando importo em *Meta*, então os nomes são resolvidos para referências, e um nome inexistente aparece na lista de erros com a linha.
- **CA-11.2** Dado o modo UPSERT pela chave `vendedor+mes`, quando importo duas vezes o mesmo arquivo com um valor alterado, então o registro é atualizado e não duplicado.

---

# MODELAGEM SEMÂNTICA

## F12 — Visões e Colunas

**Prioridade:** MVP

### Objetivo

A **Visão** é a camada semântica: um dataset nomeado, com colunas descritas como dimensões ou medidas, que alimenta todos os objetos. Ela isola o consumidor do SQL e centraliza regras como RLS, máscaras e agregações.

### Requisitos funcionais

- **RF-12.1** Tipos de visão:

  | Tipo | Definição |
  |---|---|
  | `TABLE` | Uma tabela de uma conexão, do `lake` ou de uma UDT |
  | `SQL` | Uma query SQL livre sobre uma conexão (exige `SQL_EDITOR`) |
  | `JOIN` (V1) | Montada visualmente: tabela base + joins (tipo, chaves) + filtros fixos, sem SQL |
  | `SYSTEM_UDT` | Gerada automaticamente pela F09; não editável na estrutura, mas editável nos metadados de coluna |

- **RF-12.2** Ao salvar, o sistema executa a visão com `LIMIT 0` para descobrir as colunas e seus tipos. Colunas novas são adicionadas e colunas removidas são marcadas "ausentes", com alerta de impacto.
- **RF-12.3** Metadados por **Coluna**:
  - nome técnico e rótulo amigável;
  - descrição;
  - papel: `DIMENSION` ou `MEASURE`;
  - tipo de dado;
  - agregação padrão (`SUM`, `AVG`, `COUNT`, `COUNT_DISTINCT`, `MIN`, `MAX`, `NONE`);
  - máscara (F14);
  - LOV associada (F16);
  - visível sim/não;
  - hierarquia de drill (F27).
- **RF-12.4** **Colunas calculadas da visão**: expressão OBExpr (mesma linguagem da F09) ou SQL. Exemplos:
  - medida `margem = SUM({receita}) - SUM({custo})`;
  - dimensão `ano_mes = FORMAT_DATE({data}, 'yyyy-MM')`.
  - Colunas calculadas de medida podem usar agregações.
- **RF-12.5** Hierarquias de data automáticas para colunas DATE/DATETIME: ano › trimestre › mês › dia, disponíveis para agrupamento e drill.
- **RF-12.6** Parâmetros: visões SQL podem conter tokens (F15) e parâmetros de filtro (`:param`) resolvidos em tempo de execução.
- **RF-12.7** Pré-visualização dos dados (100 linhas) e "análise de impacto": quais objetos, telas, alertas e relatórios usam a visão e cada coluna.
- **RF-12.8** Cache de resultados por visão: TTL configurável, invalidado por fim de carga ETL (F08) ou por escrita na UDT de origem. Padrão: sem cache para LIVE e invalidação por evento para UDT/lake.

### Regras de negócio

- **RN-12.1** **Toda** consulta de objeto é gerada pelo **motor de consulta** a partir de `visão + colunas + agregações + filtros + RLS + ordenação + limite`. Objetos nunca carregam SQL próprio, exceto o Editor SQL (F17), que cria uma visão SQL por baixo.
- **RN-12.2** O SQL gerado usa bind parameters. Identificadores vêm apenas do metadado (whitelist), nunca do input do usuário.
- **RN-12.3** Timeout e limite de linhas da F02 se aplicam a todas as execuções.
- **RN-12.4** Uma visão não pode ser excluída se tiver dependentes (`409 VIEW_IN_USE`).

### Contrato (motor de consulta)

```
GET|POST         /api/v1/views
GET|PUT|DELETE   /api/v1/views/{id}
POST             /api/v1/views/{id}/refresh-columns
GET              /api/v1/views/{id}/impact
POST             /api/v1/query
```

Exemplo de requisição ao motor de consulta:

```json
{
  "viewId": "...",
  "dimensions": [{ "column": "regiao" }, { "column": "data", "dateGrain": "MONTH" }],
  "measures": [{ "column": "valor", "aggregation": "SUM" }],
  "filters": [{ "column": "status", "op": "IN", "values": ["FATURADO"] }],
  "sort": [{ "column": "valor", "direction": "DESC" }],
  "limit": 1000,
  "context": { "screenId": "...", "objectId": "..." }
}
```

Resposta:

```json
{
  "columns": [{ "key", "label", "type", "maskId" }],
  "rows": [[...]],
  "totals": {...},
  "truncated": false,
  "durationMs": 42
}
```

### Critérios de aceite

- **CA-12.1** Dada a visão "Vendas" com a medida `valor (SUM)`, quando um objeto pede `valor` por `regiao`, então o SQL gerado contém `GROUP BY regiao` e `SUM(valor)` e aplica a RLS do usuário.
- **CA-12.2** Dada uma coluna removida da query de uma visão SQL e usada em 2 objetos, quando salvo a visão, então vejo um alerta listando os 2 objetos afetados.

---

## F13 — Categorias

**Prioridade:** MVP

### Requisitos

- **RF-13.1** Árvore de categorias e subcategorias (n níveis, padrão máximo de 3) para organizar visões, tabelas personalizadas, telas e relatórios.
- **RF-13.2** Cada recurso pertence a 0..1 categoria. As listagens podem ser filtradas e agrupadas por categoria.
- **RN-13.1** Excluir uma categoria move seus itens para "Sem categoria". Não exclui os itens.

### Contrato

```
GET|POST /api/v1/categories
PUT|DELETE /api/v1/categories/{id}
```

---

## F14 — Máscaras de Formatação

**Prioridade:** MVP

### Requisitos

- **RF-14.1** Máscaras pré-definidas:

  | Grupo | Máscaras |
  |---|---|
  | Números | inteiro, decimal 2, moeda BRL/USD/EUR, percentual 0/1/2 casas |
  | Escala | milhar abreviado (`1,2 mil`), milhão abreviado (`3,4 mi`) |
  | Datas | data curta, data longa, mês/ano, data-hora |
  | Documentos | CPF, CNPJ, CEP, telefone |

- **RF-14.2** Máscaras customizadas: prefixo, sufixo, casas decimais, separadores, escala (÷1000), cor para negativos, padrão de data (formato ICU) e padrão de texto (`###.###.###-##`).
- **RF-14.3** A máscara é aplicável a: coluna de visão (padrão), campo de UDT e coluna de objeto (sobrescreve).
- **RN-14.1** A formatação é feita no **frontend**, com base no metadado `maskId`. A API devolve valores crus. Exceção: exportação XLSX, que aplica formato nativo do Excel.

---

## F15 — Tokens (Variáveis)

**Prioridade:** MVP (sistema) / V1 (customizados)

### Objetivo

Valores dinâmicos resolvidos em tempo de execução. São usados em SQL de visões, RLS, filtros dinâmicos, títulos de objetos, alertas e relatórios.

### Requisitos funcionais

- **RF-15.1** Tokens de sistema:

  | Grupo | Tokens |
  |---|---|
  | Usuário | `{{usuario.id}}`, `{{usuario.email}}`, `{{usuario.nome}}`, `{{usuario.grupos}}`, `{{usuario.atributo.<chave>}}` |
  | Data | `{{hoje}}`, `{{agora}}`, `{{inicio_mes}}`, `{{fim_mes}}`, `{{inicio_ano}}`, `{{mes_anterior.inicio}}`, `{{mes_anterior.fim}}`, `{{ano_atual}}` |
  | Empresa | `{{empresa.nome}}` |

- **RF-15.2** Tokens customizados (V1), definidos pelo admin:
  - **constante** — ex.: `{{meta_margem}} = 0.25`;
  - **consulta** — valor resultante de uma visão, com cache de N minutos. Ex.: `{{ultimo_fechamento}}`.
- **RF-15.3** Em SQL e predicados, o token é sempre convertido em **bind parameter** com tipo. Em textos (títulos), é interpolado como string formatada.
- **RN-15.1** Token inexistente em SQL/RLS gera erro de validação ao salvar. Token sem valor em tempo de execução segue a RN-05.3.

---

## F16 — Listas de Valores (LOV)

**Prioridade:** V1

### Requisitos

- **RF-16.1** Tipos de LOV:

  | Tipo | Definição |
  |---|---|
  | **Estática** | Pares valor/rótulo cadastrados |
  | **Dinâmica** | Coluna valor + coluna rótulo de uma visão, com filtro opcional e possibilidade de depender de outro filtro (encadeamento, F28) |

- **RF-16.2** LOVs dinâmicas respeitam a RLS: o usuário só vê opções que existem nos seus dados.
- **RF-16.3** Busca server-side e paginação para LOVs grandes (> 500 itens).
- **RF-16.4** Uma LOV pode ser associada a uma coluna de visão. Filtros sobre essa coluna a usam automaticamente.

### Contrato

```
GET|POST /api/v1/lovs
PUT|DELETE /api/v1/lovs/{id}
GET /api/v1/lovs/{id}/values?q=&page=&dependsOn=<json>
```

---

# CONSULTAS

## F17 — Editor SQL

**Prioridade:** MVP

### Requisitos

- **RF-17.1** Editor com:
  - syntax highlight e autocomplete de tabelas/colunas da conexão (Monaco/CodeMirror);
  - execução da seleção ou do script inteiro;
  - resultado em grade.
- **RF-17.2** Ações sobre o resultado:
  - "salvar como Visão";
  - "criar objeto a partir deste resultado" — cria a visão e abre o editor de objeto.
- **RF-17.3** Histórico pessoal das últimas 50 queries e queries salvas (snippets).
- **RF-17.4** Explorador lateral de schema (F07 RF-07.4).
- **RN-17.1** Exige a flag `SQL_EDITOR`.
- **RN-17.2** Na conexão interna, só são liberados os schemas `lake` e `udt` (via views), em modo somente leitura. O schema `app` nunca é acessível.
- **RN-17.3** Somente `SELECT`/`WITH`, com timeout e limite de linhas (RN-07.1, F02).

---

## F18 — Consulta Customizada (Construtor Visual)

**Prioridade:** V1

### Objetivo

Permitir que usuários sem SQL montem consultas ad hoc sobre visões: escolher dimensões, medidas, filtros e ordenação, e ver o resultado.

### Requisitos

- **RF-18.1** Interface de arrastar e soltar:
  - lista de colunas da visão, agrupadas por dimensões e medidas;
  - áreas "Linhas", "Colunas (pivot)", "Valores" e "Filtros".
- **RF-18.2** O resultado é exibido como tabela/pivot, com troca rápida para gráfico.
- **RF-18.3** "Salvar como objeto" em uma tela existente ou nova.
- **RF-18.4** Qualquer usuário com `VIEW` numa tela pode usar "explorar" em um objeto. Isso abre o construtor com a configuração do objeto, sem salvar sobre o original.
- **RN-18.1** Exige a flag `CUSTOM_QUERY`. Opera somente via o motor de consulta (F12), com RLS.

---

# VISUALIZAÇÃO

## F19 — Telas (Dashboards)

**Prioridade:** MVP

### Objetivo

Canvas onde o construtor posiciona Objetos e o consumidor analisa os dados. Equivale a Criação de Tela, Dashboard, Inserir Objeto, Menu de Objetos, Cabeçalho/Rodapé, Atualização Tela e Excluir.

### Requisitos funcionais

**Estrutura e edição**

- **RF-19.1** Uma Tela tem:
  - nome, descrição, categoria e ícone;
  - **layout em grade responsiva** — 12 colunas, altura em unidades, breakpoints desktop/tablet/mobile-web;
  - lista de objetos;
  - filtros de tela (F28);
  - cabeçalho e rodapé;
  - opções.
- **RF-19.2** Modo **edição** × modo **visualização**. No modo edição, o construtor:
  - insere objetos pelo **menu de objetos** (Valor, Consulta, Gráfico, Imagem, Texto);
  - arrasta e redimensiona;
  - duplica (inclusive entre telas);
  - exclui.
- **RF-19.3** Cabeçalho configurável: título (com tokens), logo, filtros superiores, data/hora da última atualização. Rodapé: texto livre e tokens.
- **RF-19.4** **Abas** dentro da tela (V1): várias páginas compartilhando os filtros de tela.

**Atualização e versões**

- **RF-19.5** Atualização:
  - botão "atualizar tela";
  - atualização automática a cada N minutos (configurável por tela; mínimo 1 min);
  - atualização individual por objeto.
- **RF-19.6** Rascunho e publicação: edições ficam em rascunho até "publicar". Os consumidores veem apenas a versão publicada (V1; no MVP, salvar já publica).
- **RF-19.7** Duplicar a tela inteira. Exportar/importar a definição em JSON, entre instâncias (V1).
- **RF-19.8** Excluir vai para a lixeira por 30 dias (restaurável).
- **RF-19.9** Modo tela cheia (apresentação).

### Regras de negócio

- **RN-19.1** Os objetos de uma tela carregam em paralelo e de forma independente. Um objeto com erro mostra a mensagem só nele, sem quebrar a tela.
- **RN-19.2** Consultas idênticas (mesma visão + mesmos parâmetros + mesmo contexto RLS) disparadas na mesma renderização são deduplicadas no backend.

### Contrato

```
GET|POST         /api/v1/screens
GET|PUT|DELETE   /api/v1/screens/{id}
POST             /api/v1/screens/{id}/duplicate
POST             /api/v1/screens/{id}/publish
GET              /api/v1/screens/{id}/export           → JSON
POST             /api/v1/screens/import
POST             /api/v1/screens/{id}/objects/{objectId}/data   { filterValues }
                 → resultado do motor
```

Definição de tela (resumo):

```json
{
  "name": "Vendas Diário",
  "layout": { "columns": 12, "rowHeight": 40 },
  "header": { "title": "Vendas – {{inicio_mes}} a {{hoje}}", "showLastRefresh": true },
  "refreshIntervalSec": 300,
  "filters": [ ... ],
  "objects": [
    {
      "id": "...",
      "type": "VALUE",
      "position": { "x": 0, "y": 0, "w": 3, "h": 3 },
      "config": { ... }
    }
  ]
}
```

### Critérios de aceite

- **CA-19.1** Dada uma tela com 6 objetos, quando um deles tem a visão com erro, então os outros 5 renderizam normalmente.
- **CA-19.2** Dada uma tela com atualização a cada 5 min, quando deixo a tela aberta por 10 min, então os dados são recarregados 2 vezes sem recarregar a página.

---

## F20 — Objeto Valor (KPI)

**Prioridade:** MVP

### Requisitos

- **RF-20.1** Exibe uma medida agregada (ex.: faturamento do mês), com:
  - título e ícone;
  - máscara;
  - **comparativo** opcional: período anterior, mesmo período do ano anterior, ou outra medida/meta;
  - variação em valor e %, com seta e cor.
- **RF-20.2** Sparkline opcional: mini-série temporal da medida.
- **RF-20.3** Meta: um valor fixo, um token ou outra medida (ex.: SUM de Metas da UDT). Exibe o % atingido.
- **RF-20.4** Clique: abre drill (F27) ou navega para outra tela.
- **RN-20.1** O comparativo de período exige uma coluna de data configurada no objeto. O período é derivado do filtro de data ativo.

---

## F21 — Objeto Consulta (Tabela e Pivot)

**Prioridade:** MVP (tabela agregada) / V1 (pivot, totais avançados)

### Requisitos

- **RF-21.1** Modo **tabela**: dimensões e medidas em colunas, ordenação clicável, paginação ou scroll virtual, larguras ajustáveis e redimensionamento.
- **RF-21.2** **Totais** (geral) e **subtotais** por agrupamento. A agregação do total pode ser diferente da agregação da coluna — ex.: média das linhas versus total ponderado.
- **RF-21.3** Modo **pivot** (V1): dimensões em linhas e em colunas, medidas nos valores, expandir/recolher níveis.
- **RF-21.4** Colunas:
  - máscara por coluna;
  - alinhamento;
  - **imagem na célula** (coluna com URL ou campo `IMAGE` → miniatura);
  - **barra de dados** inline;
  - link para URL.
- **RF-21.5** Colunas calculadas do objeto, por exemplo: % do total, diferença para a linha anterior, acumulado, ranking.
- **RF-21.6** Ordenação padrão configurável e limite de linhas (top N).

---

## F22 — Objeto Gráfico (Tipos Básicos)

**Prioridade:** MVP

### Requisitos

- **RF-22.1** Tipos disponíveis:
  - **barras** (vertical/horizontal, agrupadas/empilhadas/100%);
  - **linha** e **área** (simples/empilhada);
  - **pizza** e **rosca**;
  - **combinado** (barra + linha com eixo secundário);
  - **dispersão**;
  - **funil**.
- **RF-22.2** Configuração:
  - eixo X (dimensão) e série (dimensão opcional);
  - valores (1..n medidas);
  - eixos (mín/máx, escala log);
  - legenda (posição/oculta);
  - rótulos de dados;
  - cores (paleta da plataforma ou cor fixa por série/valor);
  - linha de meta/referência.
- **RF-22.3** Tooltip formatado com a máscara. Clicar num elemento aplica filtro cruzado na tela (V1) ou faz drill (F27).
- **RF-22.4** Trocar o tipo de gráfico mantém a configuração de dados.
- **RN-22.1** Séries com mais de N categorias (padrão 50) mostram aviso e agrupam o restante em "Outros" (configurável).

---

## F23 — Gráficos Avançados

**Prioridade:** V2

| Tipo | Uso | Dados exigidos |
|---|---|---|
| **Cascata** (waterfall) | DRE, ponte de resultado | dimensão ordenada + medida; marcação de subtotais |
| **Árvore** (treemap) | Participação hierárquica | 1..3 dimensões + medida |
| **Calendário** (heatmap) | Valor por dia | coluna data + medida |
| **Fluxo** (sankey) | Fluxo origem → destino | dimensão origem, dimensão destino, medida |
| **Mapa** | Geolocalização | lat/long **ou** UF/município (código IBGE) + medida; pontos, bolhas ou coroplético |
| **Progresso / Gauge** | Atingimento de meta | medida + meta (fixa/token/medida) + faixas de cor |

- **RF-23.1** Todos usam o mesmo motor de consulta (F12) e as mesmas opções de máscara, drill e destaque.
- **RF-23.2** Os mapas usam tiles OpenStreetMap, com URL de tiles configurável (para ambientes offline). Os GeoJSON de UFs e municípios do Brasil vêm embarcados.
- **RF-23.3** **Gráfico temporário** (interpretação do item ❓ da UpQuery): no modo visualização, o consumidor pode "ver como gráfico" o resultado de uma tabela, sem salvar.

---

## F24 — Objetos Estáticos (Imagem e Texto)

**Prioridade:** MVP

- **RF-24.1** **Imagem:** upload (F11) ou URL; ajuste (cobrir/conter); link opcional.
- **RF-24.2** **Texto:** rich text/markdown com tokens. Usado para títulos de seção, instruções e notas.
- **RF-24.3** **Separador / espaçador** para organizar o layout.

---

## F25 — Linha Calculada

**Prioridade:** V1

### Objetivo

Inserir em um Objeto Consulta linhas que não vêm do banco, calculadas a partir de outras linhas. Exemplo típico: um DRE em que "Margem Bruta = Receita − CMV" e "Margem % = Margem Bruta / Receita".

### Requisitos

- **RF-25.1** No objeto tabela, o construtor define linhas calculadas com:
  - rótulo;
  - posição (após a linha X);
  - expressão sobre os **rótulos de outras linhas**: `[Receita] - [CMV]`;
  - máscara própria;
  - estilo (negrito, fundo).
- **RF-25.2** A expressão é avaliada **para cada coluna de medida**. Para pivot, é avaliada para cada coluna do pivot.
- **RF-25.3** Linhas calculadas podem referenciar outras linhas calculadas (sem ciclos — mesma regra RN-09.3).
- **RF-25.4** Opção de ocultar linhas de origem e de fixar a ordem das linhas por uma lista definida (essencial para DRE).
- **RN-25.1** A avaliação ocorre no backend, após a consulta, e o resultado já vem pronto para o frontend e para a exportação.

---

## F26 — Destaques e Sinalizadores

**Prioridade:** V1

### Requisitos

- **RF-26.1** **Destaque** (formatação condicional):
  - regra com condição sobre o valor da célula, sobre outra coluna da linha ou em comparação com uma meta/token;
  - estilo: cor do texto, cor de fundo, negrito.
  - **Escalas de cor** (gradiente de mín a máx).
- **RF-26.2** **Sinalizador**: ícone por faixa (semáforo, setas, estrelas, bandeiras) com limites configuráveis — ex.: <80% vermelho, 80–100% amarelo, ≥100% verde.
- **RF-26.3** Regras **reutilizáveis**, cadastradas centralmente e aplicáveis a vários objetos e colunas. Também é possível ter regras locais ao objeto.
- **RF-26.4** Aplicam-se a: Objeto Valor (cor do número/card), Objeto Consulta (célula/linha) e Gráficos (cor da barra/ponto).
- **RN-26.1** Avaliação no frontend sobre valores crus. A exportação XLSX tenta reproduzir as cores.

---

## F27 — Drill Down e Drill Through

**Prioridade:** V1

### Requisitos

- **RF-27.1** **Drill down:** num objeto com uma dimensão que faz parte de uma hierarquia (F12, ex.: Região › Estado › Cidade, ou Ano › Mês › Dia), clicar num valor desce um nível, filtrando pelo valor clicado. Um **breadcrumb** permite subir.
- **RF-27.2** **Drill through:** ação configurada no objeto que abre:
  - outra tela, passando o valor clicado como filtro; ou
  - um **detalhe** em modal: lista das linhas que compõem aquele valor, com as colunas escolhidas.
- **RF-27.3** Vários drills por objeto: menu de contexto "detalhar por..." para escolher dimensão ou destino.
- **RF-27.4** **Favoritar drill** (F31): salvar o caminho atual (objeto + níveis + valores) para voltar direto a ele.
- **RN-27.1** O drill respeita RLS e os filtros da tela. O detalhe de linhas respeita o limite de linhas e a flag `EXPORT` para exportação.

---

## F28 — Filtros

**Prioridade:** MVP (tela, objeto, valores padrão) / V1 (dinâmicos, encadeados, salvos)

### Requisitos funcionais

**Escopos**

- **RF-28.1** Escopos de filtro:
  - **Filtro de tela** — aplicado a todos os objetos cuja visão possui a coluna. Quando as visões têm colunas com nomes diferentes, existe um mapeamento explícito por objeto.
  - **Filtro de objeto** — fixo, configurado pelo construtor. Não é visível ao consumidor, salvo se for marcado como "exibir".
  - **Filtros superiores** — barra no topo da tela com os filtros de tela interativos.

**Tipos e valores**

- **RF-28.2** Tipos de controle:
  - seleção única ou múltipla (com LOV ou valores distintos da coluna);
  - busca de texto;
  - intervalo numérico;
  - **intervalo de datas com atalhos** (hoje, ontem, últimos 7/30 dias, mês atual, mês anterior, ano atual, personalizado);
  - booleano.
- **RF-28.3** **Valores padrão:** fixo, token (`{{inicio_mes}}`) ou "lembrar última escolha do usuário" (por usuário e tela).
- **RF-28.4** **Filtros dinâmicos:** um valor relativo é reavaliado a cada abertura. Ex.: "mês atual" em abril mostra abril; em maio, mostra maio.
- **RF-28.5** **Encadeamento:** o filtro B depende de A (Estado → Cidade). As opções de B se restringem à seleção de A. Mudar A limpa as seleções inválidas de B.

**Visibilidade e persistência**

- **RF-28.6** **Visualizar filtro no objeto:** um ícone/indicador no objeto mostra quais filtros estão afetando aquele objeto (tela + objeto + drill).
- **RF-28.7** Filtro obrigatório: a tela não consulta enquanto o valor não for preenchido.
- **RF-28.8** Filtros na URL (`?f.regiao=SUL&f.periodo=2026-01..2026-03`) para compartilhar links com o estado aplicado.
- **RF-28.9** "Visões salvas" de filtros por usuário (V1). Ex.: "Minha região – Trimestre".

### Regras de negócio

- **RN-28.1** Filtros são convertidos em predicados do motor de consulta. Nunca filtram no frontend.
- **RN-28.2** Filtros não sobrepõem a RLS (F05): eles se combinam com E.

### Critérios de aceite

- **CA-28.1** Dado o filtro de período padrão "mês atual", quando abro a tela em 15/05, então o período aplicado é de 01/05 a 31/05.
- **CA-28.2** Dados os filtros Estado → Cidade, quando seleciono "SC", então a lista de cidades só contém cidades de SC presentes nos dados que posso ver.

---

## F29 — Exportação

**Prioridade:** MVP (CSV/XLSX de objeto) / V1 (PDF/PNG de tela)

### Requisitos

- **RF-29.1** Exportar objeto:
  - **CSV**;
  - **XLSX** — com tipos nativos, máscaras e linhas calculadas;
  - **PNG** — gráficos.
- **RF-29.2** Exportar tela em **PDF** (layout da tela, com cabeçalho/rodapé e paisagem/retrato) e **PNG** (V1).
- **RF-29.3** Opção "exportar dados detalhados" (linhas brutas da visão com os filtros atuais), além do agregado exibido.
- **RN-29.1** Exige a flag `EXPORT`. A RLS é aplicada. O limite de linhas exportadas vem da F02. A exportação é auditada (F06).
- **RN-29.2** Exportações acima de 50.000 linhas são assíncronas: um job gera o arquivo e notifica o usuário com um link válido por 24 h.

---

# NAVEGAÇÃO

## F30 — Menus e Tela Inicial

**Prioridade:** MVP

### Requisitos

- **RF-30.1** Editor de **menu** (admin/builder):
  - árvore de pastas e itens;
  - um item aponta para uma Tela, um Browser, um Relatório ou um link externo;
  - ícone e ordem configuráveis.
- **RF-30.2** O menu mostrado a cada usuário é **filtrado pela ACL**. Pastas sem itens visíveis ficam ocultas.
- **RF-30.3** **Tela inicial:** definida por grupo (admin) e sobrescrevível pelo usuário (preferência). Precedência: usuário > grupo (primeiro pela ordem de prioridade) > global.
- **RF-30.4** Barra superior com:
  - busca global (telas, browsers e relatórios pelo nome);
  - favoritos;
  - avisos;
  - perfil do usuário (preferências, troca de senha, tema, sair).
- **RF-30.5** Menu de administração separado (configurações, usuários, conexões, auditoria, tarefas), visível conforme o papel.

---

## F31 — Favoritos

**Prioridade:** V1

- **RF-31.1** Favoritar **tela** (estrela), exibida no menu "Favoritos".
- **RF-31.2** **Objetos favoritos:** fixar um objeto de qualquer tela numa **tela pessoal "Meu Painel"**, montada pelo próprio usuário (layout próprio). O objeto continua sendo do original: alterações no original refletem no painel.
- **RF-31.3** **Favoritar drill** (F27 RF-27.4).
- **RN-31.1** Se o usuário perder o acesso ao recurso original, o favorito some (não é exibido) sem ser apagado. Volta se o acesso retornar.

---

## F32 — Avisos

**Prioridade:** V2

- **RF-32.1** O admin publica avisos (título, texto, período de exibição, público por grupo, nível info/alerta).
- **RF-32.2** Os avisos aparecem num sino na barra superior, com indicador de não lido, e opcionalmente como banner no login.
- **RF-32.3** O mesmo canal recebe notificações do sistema para o usuário: exportação pronta, importação concluída, falha de job próprio.

---

# AUTOMAÇÃO

## F33 — Agendador de Tarefas

**Prioridade:** V1 (MVP: apenas jobs internos de limpeza)

### Objetivo

Motor central de execução agendada e assíncrona. Ele é usado por:

- ETL (F08);
- Processos (F34);
- Alertas (F35);
- Envio de relatórios (F37);
- Importações e exportações grandes;
- Jobs de sistema (retenção, lixeira).

Equivale a Tarefas, Verificação de Agendamentos e Verificação de Tarefas Executadas.

### Requisitos funcionais

- **RF-33.1** Agendamento por:
  - **cron** amigável — UI com "a cada N minutos/horas", "diário às", "dias da semana", "dia do mês", e modo avançado com expressão cron;
  - **evento** — "após a carga X concluir";
  - **manual**.
- **RF-33.2** Tela **Agendamentos**: todas as tarefas agendadas, com tipo, dono, próxima execução, última execução, status e ações (ativar/pausar, executar agora, editar).
- **RF-33.3** Tela **Tarefas Executadas**: histórico com início, fim, duração, status (`SUCCESS`, `FAILED`, `RUNNING`, `CANCELLED`), log resumido e detalhes do erro. Inclui filtros e a ação "cancelar" para tarefas em execução.
- **RF-33.4** Política de falha por tarefa: número de retentativas e intervalo; notificação ao dono e/ou a e-mails em caso de falha.
- **RF-33.5** Fuso horário dos agendamentos = fuso da instância (F02).

### Regras de negócio

- **RN-33.1** Quartz com JobStore JDBC: os agendamentos sobrevivem a restarts. Execuções perdidas durante uma parada seguem a política `misfire` (padrão: executar uma vez ao subir).
- **RN-33.2** Concorrência global limitada (pool padrão de 4 threads, configurável), para não competir com as requisições web.
- **RN-33.3** Uma tarefa nunca roda em duplicidade (`@DisallowConcurrentExecution` por chave).

### Contrato

```
GET   /api/v1/scheduler/tasks?type&status
PATCH /api/v1/scheduler/tasks/{id}           { enabled, schedule }
POST  /api/v1/scheduler/tasks/{id}/run
GET   /api/v1/scheduler/runs?taskId&status&from&to
POST  /api/v1/scheduler/runs/{runId}/cancel
```

---

## F34 — Processos (Rotinas SQL)

**Prioridade:** V1

### Objetivo

Executar transformações SQL dentro do banco interno: consolidar tabelas do lake, popular tabelas personalizadas e calcular agregados pesados. É o equivalente leve do "dbt" e dos "Processos" da UpQuery.

### Requisitos

- **RF-34.1** Um Processo é um script SQL (um ou mais comandos), executado na conexão **interna**, com permissão de escrita **somente** nos schemas `lake` e `udt`.
- **RF-34.2** Tipos de Processo:
  - **SQL livre** — admin;
  - **"Materializar visão"** — grava o resultado de uma visão numa tabela do lake, com `FULL`/`INCREMENTAL`.
- **RF-34.3** Agendamento e encadeamento via F33 (ex.: após a carga do ERP → rodar processo → invalidar cache).
- **RF-34.4** Execução transacional (tudo ou nada), com log de linhas afetadas por comando.
- **RN-34.1** Apenas `ADMIN` cria e edita processos de SQL livre.
- **RN-34.2** Escrita direta em `udt` via processo é bypass de validação (RN-09.6). Essa escrita é registrada na auditoria e dispara a invalidação de cache das visões dependentes.

---

## F35 — Alertas e Ações

**Prioridade:** V1

### Objetivo

Avisar as pessoas **antes** do problema: monitorar indicadores e notificar quando uma condição for atingida. Esse é o núcleo do "uAction" da UpQuery.

### Requisitos funcionais

**Definição do alerta**

- **RF-35.1** Um Alerta tem:
  - nome;
  - **fonte**: uma visão + medida agregada + filtros, ou um objeto existente de uma tela;
  - **condição**: comparação (`>`, `<`, `=`, entre, variação % vs. período anterior, "nenhum dado") com um valor fixo, um token ou outra medida;
  - **avaliação**: por agendamento (F33).
- **RF-35.2** Modo **por linha**: avaliar a condição por grupo (ex.: por filial) e notificar listando os grupos que violaram.
- **RF-35.3** **Destinatários**: usuários, grupos e e-mails externos (V1).
- **RF-35.4** **Ações** disparadas quando a condição é verdadeira:

  | Ação | Descrição |
  |---|---|
  | **E-mail** | Template com tokens + valores + link para a tela |
  | **Notificação interna** | F32 |
  | **Webhook** | POST JSON configurável — permite integrar com Slack, Teams, n8n, WhatsApp via provedor etc. |
  | **Executar Processo** | F34 (V2) |

- **RF-35.5** Anti-spam:
  - "notificar só na mudança de estado" (OK→ALERTA, e opcionalmente ALERTA→OK);
  - ou "a cada avaliação";
  - período de silêncio (cooldown).
- **RF-35.6** Histórico de disparos com valor avaliado, destinatários e status de envio.

### Regras de negócio

- **RN-35.1** Para destinatários usuários, a avaliação roda **no contexto RLS de cada destinatário** (RN-05.2). Cada um é notificado sobre os próprios dados. E-mails externos usam o contexto do **dono** do alerta, com aviso explícito na UI.
- **RN-35.2** Webhooks têm timeout de 10 s, 3 retentativas com backoff e assinatura HMAC no header `X-OpenBI-Signature`, com segredo por webhook.

### Contrato

```
GET|POST         /api/v1/alerts
GET|PUT|DELETE   /api/v1/alerts/{id}
POST             /api/v1/alerts/{id}/test     → avalia agora e mostra o resultado, sem notificar
GET              /api/v1/alerts/{id}/events
```

### Critérios de aceite

- **CA-35.1** Dado o alerta "estoque < 10 por produto" com notificação só na mudança de estado, quando o produto X cai para 8, então um e-mail é enviado uma vez. Ele não é reenviado enquanto o estoque continuar < 10.
- **CA-35.2** Dado o alerta para os gerentes de SC e PR com RLS por filial, quando é disparado, então cada gerente recebe só as filiais dele.

---

## F36 — Relatórios (Report)

**Prioridade:** V2

### Objetivo

Relatórios **paginados, prontos para impressão/PDF**, distintos dos dashboards interativos. Exemplos: listagem de pedidos por cliente com quebras e totais, fechamento mensal.

### Requisitos

- **RF-36.1** Um Report tem:
  - fonte (visão);
  - **parâmetros** (Filtros Report, mesmos tipos da F28);
  - layout com cabeçalho/rodapé de página (logo, título, parâmetros usados, número da página);
  - **quebras/grupos** com cabeçalho e subtotais;
  - colunas com máscara;
  - totais gerais;
  - orientação e tamanho de página.
- **RF-36.2** Execução interativa: preencher parâmetros → visualizar em HTML paginado → baixar PDF/XLSX.
- **RF-36.3** Pode conter blocos de objetos (gráfico, KPI) no topo, como resumo.
- **RN-36.1** Geração do PDF no servidor. `[DECISÃO D-07]` — JasperReports, OpenPDF ou HTML + Chromium headless.

---

## F37 — Agendamento e Envio de Relatórios

**Prioridade:** V1 (tela/objeto por e-mail) / V2 (Report)

### Requisitos

- **RF-37.1** Uma assinatura (subscription) tem:
  - conteúdo: Tela (PDF/PNG), Objeto (XLSX/CSV/PNG) ou Report (PDF/XLSX);
  - valores de filtro/parâmetro (fixos ou tokens — ex.: "mês anterior");
  - destinatários;
  - agendamento (F33);
  - assunto e corpo do e-mail com tokens.
- **RF-37.2** Opção "não enviar se vazio".
- **RF-37.3** O usuário pode assinar uma tela para si mesmo, a partir da própria tela ("receber por e-mail toda segunda às 8h").
- **RF-37.4** Canais: e-mail (SMTP) e webhook com o arquivo como URL temporária assinada (V1).
- **RN-37.1** O conteúdo é gerado **por destinatário**, no contexto RLS dele (RN-05.2). Para muitos destinatários com o mesmo contexto efetivo de RLS, o arquivo é gerado uma vez e reaproveitado.
- **RN-37.2** Exige SMTP configurado. Sem SMTP, a opção de e-mail fica desabilitada com explicação.

---

# DISTRIBUIÇÃO DE CONTEÚDO

## F38 — Modo TV (Kiosk)

**Prioridade:** V2

### Objetivo

Exibir telas em TVs da empresa (chão de fábrica, comercial, expedição) **sem um app instalado**: qualquer navegador em uma Smart TV, um Chromecast, um Raspberry Pi ou um mini-PC. É o equivalente ao TVCONEX, mas baseado em URL.

### Requisitos

- **RF-38.1** Cadastro de **Dispositivo/Painel TV**:
  - nome e local;
  - **playlist** de telas com tempo de exibição de cada uma (rotação);
  - valores de filtro fixos por tela;
  - intervalo de atualização;
  - tema (forçar escuro).
- **RF-38.2** Geração de **URL de acesso** com token opaco (`/tv/{token}`). Abre em tela cheia, sem menus e sem login interativo.
- **RF-38.3** Cada dispositivo roda com a identidade de um **usuário técnico** escolhido. A RLS e a ACL valem como se fosse esse usuário.
- **RF-38.4** Portal de gestão:
  - lista de dispositivos com **último heartbeat** (online/offline), IP e versão;
  - revogar ou regenerar o token;
  - "recarregar agora" remoto (via polling/SSE).
- **RF-38.5** Reconexão automática e "tela de erro amigável" se o servidor ficar indisponível, com nova tentativa a cada 30 s.
- **RN-38.1** Token com no mínimo 32 bytes aleatórios. Ele pode ser restrito por faixa de IP (opcional). Os acessos são auditados.
- **RN-38.2** O usuário técnico não pode ser ADMIN.

---

## F39 — API Pública

**Prioridade:** V2

### Objetivo

Permitir integração externa: consultar dados de visões, alimentar tabelas personalizadas e automatizar a administração.

### Requisitos

- **RF-39.1** **Tokens de API** pessoais ou de serviço:
  - nome, escopos (`query:read`, `udt:read`, `udt:write`, `admin`) e expiração;
  - exibidos uma única vez e armazenados como hash.
- **RF-39.2** Endpoints públicos estáveis (versão `/api/v1`):
  - `POST /query` (F12);
  - CRUD de records de UDT (F09);
  - listagem de telas e visões;
  - disparo de cargas e processos.
- **RF-39.3** Documentação OpenAPI/Swagger UI servida pela instância em `/api/docs`.
- **RF-39.4** Rate limit por token (padrão 60 req/min, configurável).
- **RN-39.1** Requisições com token herdam a ACL e a RLS do usuário dono do token, limitadas pelos escopos.
- **RN-39.2** O uso de tokens é auditado (F06), com último uso exibido na tela de tokens.

---

## 5. Fora de escopo

| Item | Motivo |
|---|---|
| IA conversacional (equivalente ao uSense) | Decisão do produto — fora por ora. A camada semântica (F12) já deixa o terreno pronto para o futuro. |
| App mobile nativo | Decisão do produto. A web é responsiva (F19 RF-19.1) e deve funcionar bem no navegador do celular. |
| Multi-tenant (vários clientes por instância) | Uma instância = uma empresa `[DECISÃO D-03]`. |
| Portal de chamados / suporte | Comunidade via GitHub Issues/Discussions. |
| Serviço gerenciado | O projeto é self-hosted. Terceiros podem oferecer hospedagem. |
| WhatsApp nativo | Coberto indiretamente por webhook (F35). Um conector nativo pode vir como plugin futuro. |
| "Marcadores", "JET", "Float Filter/Par" (UpQuery) | Significado não confirmado na engenharia reversa. Revisitar após a demo. |

---

## 6. Decisões em aberto

Precisam de confirmação **antes** do scaffold do projeto:

| ID | Decisão | Proposta | Alternativa |
|---|---|---|---|
| **D-01** | Stack | Java 21 + Spring Boot 3, React + TS + Vite, PostgreSQL 16, Quartz, ECharts | Kotlin; Next.js |
| **D-02** | Armazenamento das Tabelas Personalizadas | DDL físico no schema `udt` + views para campos derivados | JSONB (mais simples, pior para BI em volume) |
| **D-03** | Tenancy | Single-tenant (1 instância = 1 empresa) | Multi-tenant por schema (custo alto agora) |
| **D-04** | Linguagem de fórmula | OBExpr própria, compilada para SQL (parser em ANTLR) | Expor SQL direto (mais poder, menos segurança e usabilidade) |
| **D-05** | Sessão | Cookie de sessão server-side (Spring Session JDBC) | JWT stateless |
| **D-06** | Licença | AGPL-3.0 (protege contra SaaS fechado sobre o projeto) | Apache-2.0 (mais adoção corporativa) |
| **D-07** | Geração de PDF | HTML + Chromium headless (fidelidade com a tela) | OpenPDF/Jasper (imagem mais leve) |
| **D-08** | Nome do projeto | — | Evitar "UpQuery" e derivados |
