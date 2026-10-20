# F29 — Exportação Specification

**Prioridade:** MVP (CSV/XLSX de objeto e de Browser, PNG de gráfico, dados detalhados, exportação assíncrona) / V1 (PDF/PNG de tela) · **Depende de:** F19, F21, F22, F10 · **Origem:** F29 do `produto.specs.md`

## Problem Statement

O usuário precisa levar dados para o Excel, para apresentações e para e-mail: a tabela como está na tela, o gráfico como imagem ou as linhas detalhadas por trás de um número. A exportação precisa respeitar permissão, RLS e limites, ser auditada e não travar o servidor com arquivos grandes.

## Goals

- [ ] Exportar uma tabela em XLSX com tipos nativos, máscaras como formatos do Excel e colunas calculadas.
- [ ] Exportações acima de 50 mil linhas rodam em segundo plano e ficam disponíveis por 24 h.

## Out of Scope

| Feature | Reason |
| --- | --- |
| PDF e PNG da tela inteira (RF-29.2) | V1 (Playwright no analytics, AD-011) |
| Envio agendado por e-mail | F37 (V1) |
| Notificação no sino | F32 (V2); no MVP o painel "Minhas exportações" mostra o status |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| CSV | UTF-8 com BOM, separador `;`, decimal com vírgula, datas `dd/MM/yyyy`, gerado em streaming pelo Spring | Abre corretamente no Excel pt-BR; AD-011 | y |
| XLSX | Gerado no analytics com `xlsxwriter` (modo `constant_memory`), a partir de CSV normalizado + metadados de colunas enviados pelo Spring | AD-011; volume grande sem estourar memória | y |
| O que é exportado ("como na tela") | O resultado final do pipeline do objeto: consultas do motor + transform (colunas calculadas, total) | XLSX = o que se vê | y |
| Dados detalhados | Linhas cruas da visão principal do objeto com os filtros atuais (tela + objeto) e RLS, só colunas visíveis da visão | RF-29.3 | y |
| PNG de gráfico | Gerado no navegador com `echartsInstance.getDataURL({ pixelRatio: 2 })`; o front registra a exportação via `POST /exports/audit` | Fidelidade total, sem servidor | y |
| Limite | `limits.maxExportRows` (padrão 100.000); acima → `truncated` com aviso na primeira linha do arquivo (CSV) ou numa aba "Aviso" (XLSX) | RN-29.1 | y |
| Assíncrono | Estimativa por `COUNT(*)` da consulta; > 50.000 linhas → job (`app.export_job`), arquivo no `FileStore` com expiração de 24 h, download só pelo dono | RN-29.2 | y |
| Notificação no MVP | Painel "Minhas exportações" com polling e, se houver SMTP, e-mail com o link | F32 é V2 | y |
| Máscara → Excel | `brl` → `"R$" #,##0.00`, `pct1` → `0.0%`, `thousand` → `#,##0.0, "mil"`, datas → `dd/mm/yyyy`, máscaras TEXT (CPF/CNPJ) → texto já formatado | RN-14.1 (exceção do XLSX) | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Exportar objeto em CSV e XLSX ⭐ MVP

**User Story**: Como usuário com a flag `EXPORT`, quero baixar os dados de uma tabela ou gráfico em CSV ou XLSX.

**Why P1**: RF-29.1.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/screens/{id}/objects/{objectId}/export` recebe `format=CSV` e os `filterValues` atuais THEN o sistema SHALL devolver o CSV do resultado do objeto (UTF-8 com BOM, `;`, decimal com vírgula).
2. WHEN `format=XLSX` THEN o sistema SHALL devolver um XLSX com números e datas como tipos nativos, formatos do Excel derivados das máscaras, colunas calculadas e linha de total quando o objeto a exibe.
3. IF o usuário não tem a flag `EXPORT` THEN o sistema SHALL responder 403 `FLAG_REQUIRED`.
4. The exportação SHALL aplicar RLS, filtros de tela e de objeto exatamente como na tela.
5. WHEN uma exportação é concluída THEN o sistema SHALL publicar `DataExported` com formato, linhas e recurso (auditoria F06).
6. IF o resultado excede `maxExportRows` THEN o arquivo SHALL conter exatamente `maxExportRows` linhas e o aviso de truncamento.

**Independent Test**: IT: exportar tabela de top 10 com % do total em XLSX e conferir células, formatos e total com `openpyxl`/Apache POI.

---

### P1: Dados detalhados ⭐ MVP

**User Story**: Como analista, quero exportar as linhas brutas por trás de um objeto.

**Why P1**: RF-29.3.

**Acceptance Criteria**:
1. WHEN `export` recebe `detailed=true` THEN o sistema SHALL exportar as linhas cruas da visão principal do objeto com os filtros atuais e a RLS, sem agregação, com as colunas visíveis da visão.
2. IF o objeto é estático (imagem/texto) THEN o sistema SHALL responder 400 `EXPORT_NOT_SUPPORTED`.

**Independent Test**: IT comparando a contagem de linhas detalhadas com o `COUNT(*)` filtrado.

---

### P1: PNG de gráfico ⭐ MVP

**User Story**: Como usuário, quero baixar um gráfico como imagem para apresentações.

**Why P1**: RF-29.1.

**Acceptance Criteria**:
1. WHEN o usuário escolhe "Exportar PNG" num gráfico THEN o frontend SHALL gerar a imagem com `getDataURL` em `pixelRatio` 2 e fundo do tema, e iniciar o download `<titulo>.png`.
2. WHEN o PNG é gerado THEN o frontend SHALL chamar `POST /api/v1/exports/audit` e o sistema SHALL publicar `DataExported` com `format=PNG`.
3. WHILE o usuário não tem `EXPORT` the frontend SHALL ocultar todas as opções de exportação.

**Independent Test**: Teste de componente com instância ECharts simulada.

---

### P1: Exportação do Browser ⭐ MVP

**User Story**: Como usuário, quero exportar a grade do Browser com os filtros atuais.

**Why P1**: RF-10.10.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/browsers/{id}/export` recebe formato, filtros, busca e ordenação atuais THEN o sistema SHALL exportar todas as linhas que satisfazem os filtros (sem paginação), com as colunas visíveis do Browser e a RLS, até `maxExportRows`.

**Independent Test**: IT de exportação de Browser com filtro.

---

### P1: Exportação assíncrona ⭐ MVP

**User Story**: Como usuário, quero que exportações grandes rodem em segundo plano e eu baixe depois.

**Why P1**: RN-29.2.

**Acceptance Criteria**:
1. WHEN a contagem estimada passa de 50.000 linhas THEN o endpoint de exportação SHALL responder 202 `{ exportId }` e gerar o arquivo num job.
2. WHEN `GET /api/v1/exports/{exportId}` é chamado pelo dono THEN o sistema SHALL devolver `status` (`RUNNING`, `READY`, `FAILED`), `rows` e `expiresAt`.
3. WHEN o arquivo está pronto THEN `GET /api/v1/exports/{exportId}/download` SHALL devolvê-lo ao dono por até 24 h; WHERE há SMTP the sistema SHALL enviar e-mail com o link.
4. IF outro usuário tenta baixar THEN o sistema SHALL responder 404.
5. WHEN passam 24 h THEN o job de limpeza SHALL apagar o arquivo e `download` SHALL responder 410 `EXPORT_EXPIRED`.
6. The painel "Minhas exportações" SHALL listar as exportações recentes do usuário com status e link.

**Independent Test**: IT com 60 mil linhas: 202 → polling → download → expiração (relógio controlado).

---

### P2: PDF/PNG da tela [V1]

**User Story**: Como usuário, quero exportar a tela inteira em PDF ou PNG com cabeçalho e rodapé.

**Why P2**: V1.

**Acceptance Criteria**:
1. WHEN exporto a tela em PDF paisagem THEN o sistema SHALL gerar o PDF no servidor com o layout da tela, cabeçalho e rodapé.

**Independent Test**: Diferido para a V1.

---

## Edge Cases

- IF o analytics estiver fora durante um XLSX THEN o sistema SHALL responder 503 `ANALYTICS_UNAVAILABLE` (síncrono) ou marcar o job `FAILED` com a mensagem.
- WHEN um valor de texto começa com `=`, `+`, `-` ou `@` THEN o CSV e o XLSX SHALL prefixá-lo com `'` para evitar injeção de fórmula.
- IF o nome do arquivo contém caracteres inválidos THEN o sistema SHALL substituí-los por `_`.
- WHEN o mesmo usuário dispara mais de 3 exportações assíncronas simultâneas THEN a quarta SHALL receber 429 `EXPORT_LIMIT`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| EXP-01 | P1: CSV de objeto | RF-29.1 | Tasks | Pending |
| EXP-02 | P1: XLSX de objeto | RF-29.1, RN-14.1 | Tasks | Pending |
| EXP-03 | P1: Permissão, RLS, limite, auditoria | RN-29.1 | Tasks | Pending |
| EXP-04 | P1: Dados detalhados | RF-29.3 | Tasks | Pending |
| EXP-05 | P1: PNG de gráfico | RF-29.1 | Tasks | Pending |
| EXP-06 | P1: Exportação do Browser | RF-10.10 | Tasks | Pending |
| EXP-07 | P1: Assíncrona | RN-29.2 | Tasks | Pending |
| EXP-08 | P1: Injeção de fórmula | Segurança | Tasks | Pending |
| EXP-09 | P2: PDF/PNG da tela | RF-29.2 | - | Pending |

**Coverage:** 9 total, 8 mapped to tasks, 1 diferido (V1).

---

## Success Criteria

- [ ] XLSX abre no Excel/LibreOffice com números somáveis e formatos corretos.
- [ ] Nenhuma exportação ignora RLS (teste com dois usuários).
