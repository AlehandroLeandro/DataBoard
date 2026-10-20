# F11 — Importação de Dados e Upload Specification

**Prioridade:** MVP (planilha → UDT, arquivos) / V1 (modelos de importação, criar registro na tabela alvo) · **Depende de:** F09 · **Origem:** F11 do `produto.specs.md`

## Problem Statement

Os dados das PMEs vivem em planilhas: metas, orçamentos, cadastros. Digitá-los no Browser não escala. O usuário precisa subir um `.xlsx`/`.xls`/`.csv`, mapear colunas para campos (inclusive referências por nome), pré-validar e importar com segurança. Também falta armazenamento de arquivos para campos `FILE`/`IMAGE` e para o objeto Imagem das telas (F24).

## Goals

- [ ] Importar 200.000 linhas de uma planilha de metas para uma UDT existente, com referências resolvidas por nome, sem travar a UI.
- [ ] O usuário vê todos os erros por célula **antes** de gravar e pode baixar um CSV de erros.
- [ ] Upload de arquivos com validação de MIME real, disponível para campos `FILE`/`IMAGE` e objetos de imagem.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Modelos de importação salvos e planilha-modelo (RF-11.3) | V1 |
| "Criar registro na tabela alvo" para referência não encontrada | V1 (RF-11.2) |
| Armazenamento compatível com S3 | V2 (RF-11.5) |
| Importação para tabelas do lake | F08 (V1) |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Onde o arquivo é lido | O analytics (Python) faz parse, detecção de encoding/separador, inferência de tipos e normalização; o Spring valida contra a UDT e grava | Manipulação tabular em Python (AD-001); o Spring continua o único a acessar o banco (AD-002) | y |
| Formato de troca | `POST /v1/import/inspect` devolve abas, cabeçalhos, tipos inferidos e prévia de 50 linhas; `POST /v1/import/normalize` devolve CSV UTF-8 normalizado (datas ISO, decimal com ponto) em streaming | Evita JSON de 200 mil linhas; CSV normalizado é trivial de ler no Java | y |
| Execução | `analyze` é síncrono (até 200.000 linhas, devolve as 1.000 primeiras falhas + CSV completo de erros); `execute` é sempre assíncrono (job + progresso) | RN-11.4; a UI acompanha o progresso | y |
| Atomicidade | Padrão "tudo ou nada" em uma transação; opção `onlyValid=true` grava só as linhas válidas | RN-11.2 | y |
| UPSERT | Casa registros existentes pelos campos-chave escolhidos (armazenados); atualiza se encontrar, insere se não | RF-11.1 | y |
| REPLACE | Exige `DATA_DELETE`, `confirmReplace=true` no corpo e apaga todos os registros na mesma transação da inserção | RF-11.1 | y |
| Resolução de referência | Valor da célula comparado ao campo de exibição da tabela alvo, sem distinguir maiúsculas e ignorando espaços nas pontas; mais de um match → erro `AMBIGUOUS_REFERENCE` | RF-11.2 sem surpresas | y |
| Inferência de tipos (tabela nova) | Amostra de até 1.000 linhas: inteiro, decimal, data (`dd/MM/yyyy`, ISO), booleano (`sim/não`, `true/false`), senão TEXT; mais de 255 caracteres → LONG_TEXT | Previsível e editável antes de confirmar | y |
| Armazenamento de arquivos | Volume `databoard-files` em `/var/lib/databoard/files/<yyyy>/<mm>/<uuid>`, metadados em `app.stored_file` | RF-11.5 | y |
| Acesso a arquivos | Leitura permitida a quem enviou ou a quem tem acesso ao recurso dono (`DATA_READ` na UDT; `VIEW` na tela) | Arquivo herda a proteção do recurso | y |
| Whitelist de upload | Imagens: PNG, JPEG, WEBP, GIF, SVG (sanitizado); documentos: PDF, XLSX, XLS, CSV, DOCX, TXT; máx. 20 MB; MIME real via Apache Tika | RN-11.3 | y |
| Retenção | Arquivo original da importação e CSV de erros mantidos por 30 dias (job diário) | RF-11.4 | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Enviar planilha e escolher destino ⭐ MVP

**User Story**: Como usuário com a flag `UPLOAD`, quero enviar uma planilha e escolher se ela vira uma tabela nova ou alimenta uma existente.

**Why P1**: Início do fluxo de importação.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/imports/upload` recebe um `.xlsx`, `.xls` ou `.csv` de até 20 MB THEN o sistema SHALL guardar o arquivo, chamar o analytics e responder `{ importId, sheets: [{ name, columns: [{ header, inferredType }], rowCount }], preview }`.
2. WHEN o arquivo é CSV THEN o analytics SHALL detectar encoding (UTF-8, Windows-1252, ISO-8859-1) e separador (`,` `;` tab `|`).
3. IF o arquivo excede 20 MB, tem extensão fora da lista ou MIME real divergente da extensão THEN o sistema SHALL responder 400 com `code=FILE_REJECTED` e o motivo.
4. IF o usuário não tem a flag `UPLOAD` THEN o sistema SHALL responder 403 `FLAG_REQUIRED`.
5. WHEN o destino é "nova tabela" THEN o sistema SHALL propor um campo por coluna com nome e tipo inferidos, editáveis antes da confirmação, e criar a UDT via F09 ao executar.

**Independent Test**: IT com arquivos de fixture: xlsx com 2 abas, CSV em Windows-1252 com `;`, arquivo `.xlsx` que é na verdade um ZIP qualquer (rejeitado).

---

### P1: Mapear, pré-validar e executar ⭐ MVP

**User Story**: Como usuário, quero mapear colunas para campos, ver todos os erros antes de gravar e então importar.

**Why P1**: Núcleo da importação segura.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/imports/{importId}/analyze` recebe `sheet`, `target`, `mapping` (coluna → campo, formato de data, separador decimal), `mode` e `keyFields` THEN o sistema SHALL responder `{ validRows, errorRows, errors: [{ row, column, field, code, message }] (até 1.000), errorsCsvUrl }`.
2. WHEN uma coluna mapeada para `REFERENCE` contém o valor de exibição (ex.: nome do vendedor) THEN o sistema SHALL resolvê-lo para o id do registro alvo; valor inexistente SHALL virar erro `REFERENCE_NOT_FOUND` com a linha.
3. IF o valor de exibição corresponde a mais de um registro alvo THEN o sistema SHALL registrar erro `AMBIGUOUS_REFERENCE` na linha.
4. WHEN `POST /api/v1/imports/{importId}/execute` é chamado após um `analyze` THEN o sistema SHALL iniciar um job assíncrono e responder 202 `{ jobId }`.
5. WHILE o job executa, `GET /api/v1/imports/{importId}` SHALL responder `status=RUNNING` e `progress` (0–100).
6. WHEN o modo é `INSERT` e `onlyValid=false` THEN o sistema SHALL gravar tudo numa transação ou nada, se houver qualquer erro.
7. WHERE `onlyValid=true` the sistema SHALL gravar apenas as linhas válidas e registrar as contagens de gravadas e ignoradas.
8. WHEN o modo é `UPSERT` com chave `vendedor+mes` e o mesmo arquivo é importado duas vezes com um valor alterado THEN o sistema SHALL atualizar o registro existente, sem duplicar.
9. IF o modo é `REPLACE` sem `confirmReplace=true` ou sem `DATA_DELETE` THEN o sistema SHALL responder 400 `REPLACE_NOT_CONFIRMED` ou 403, respectivamente.
10. The importação SHALL exigir a flag `UPLOAD` e os níveis de dados correspondentes (`DATA_INSERT`, `DATA_UPDATE` para UPSERT, `DATA_DELETE` para REPLACE).
11. IF a planilha tem mais de 200.000 linhas THEN o sistema SHALL responder 400 com `code=IMPORT_TOO_MANY_ROWS`.
12. WHEN a importação termina THEN cada registro gravado SHALL ter seu histórico de dados em `audit.data_change` (gravado em lote, fora da fila assíncrona) e o sistema SHALL gravar um evento `DATA/IMPORT_EXECUTED` com as contagens.

**Independent Test**: CA-11.1 e CA-11.2 em IT.

---

### P1: Histórico de importações ⭐ MVP

**User Story**: Como usuário, quero ver minhas importações anteriores com contagens e baixar o arquivo original.

**Why P1**: Rastreabilidade de cargas manuais.

**Acceptance Criteria**:
1. WHEN `GET /api/v1/imports?targetTableId=` é chamado THEN o sistema SHALL listar quem, quando, arquivo, modo, status e contagens (lidas, gravadas, com erro), paginado.
2. WHEN o arquivo original tem menos de 30 dias THEN `GET /api/v1/imports/{id}/original` SHALL devolvê-lo para quem fez a importação ou para ADMIN.
3. WHEN o job diário de retenção roda THEN o sistema SHALL apagar arquivos originais e CSVs de erro com mais de 30 dias, mantendo o registro do histórico.

**Independent Test**: IT com relógio controlado.

---

### P1: Upload de arquivos e campos FILE/IMAGE ⭐ MVP

**User Story**: Como usuário, quero anexar arquivos e imagens a registros e telas.

**Why P1**: F24 (objeto Imagem) e tipos `FILE`/`IMAGE` da F09 dependem disso.

**Acceptance Criteria**:
1. WHEN `POST /api/v1/files` recebe um arquivo da whitelist de até 20 MB THEN o sistema SHALL validar o MIME real, guardar o arquivo e responder `{ fileId, url, name, mime, size }`.
2. IF o MIME real não está na whitelist ou diverge da extensão THEN o sistema SHALL responder 400 `FILE_REJECTED`.
3. WHEN um SVG é enviado THEN o sistema SHALL remover scripts, handlers `on*` e referências externas antes de guardar.
4. WHEN um campo `FILE` ou `IMAGE` é adicionado a uma UDT THEN o sistema SHALL criar uma coluna `uuid` referenciando `app.stored_file`, com config de tipos aceitos e tamanho máximo.
5. IF um registro recebe um `fileId` inexistente, de tipo não aceito pelo campo ou enviado por outro usuário e ainda não vinculado THEN o sistema SHALL responder 400 com erro no campo.
6. WHEN `GET /api/v1/files/{id}` é chamado por quem não enviou o arquivo nem tem acesso ao recurso dono THEN o sistema SHALL responder 404.

**Independent Test**: IT: upload de PNG → vincula a registro → outro usuário sem `DATA_READ` recebe 404.

---

### P2: Modelos de importação [V1]

**User Story**: Como usuário, quero salvar o mapeamento para importações recorrentes e baixar uma planilha-modelo.

**Why P2**: V1.

**Acceptance Criteria**:
1. WHEN o usuário salva um modelo THEN o sistema SHALL guardar mapeamento, modo e conversões com um nome reutilizável em `POST /api/v1/import-templates`.

**Independent Test**: Diferido para a V1.

---

## Edge Cases

- IF a aba escolhida está vazia ou só tem cabeçalho THEN o `analyze` SHALL responder 400 com `code=IMPORT_EMPTY_SHEET`.
- IF há cabeçalhos duplicados THEN o analytics SHALL renomeá-los com sufixo (`Valor`, `Valor_2`).
- IF o analytics estiver indisponível THEN o `upload` SHALL responder 503 `ANALYTICS_UNAVAILABLE` e não manter o arquivo.
- WHEN uma célula numérica vem como texto `"1.234,56"` e o separador decimal mapeado é vírgula THEN a normalização SHALL produzir `1234.56`.
- IF o job falhar no meio de uma importação atômica THEN o sistema SHALL desfazer tudo e marcar `status=FAILED` com a mensagem.
- WHEN dois `execute` são chamados para o mesmo `importId` THEN o segundo SHALL responder 409 com `code=IMPORT_ALREADY_STARTED`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| IMP-01 | P1: Upload e inspeção | RF-11.1 (etapa 1) | Tasks | Pending |
| IMP-02 | P1: Destino nova tabela | RF-11.1 (etapa 2), RF-09.11 | Tasks | Pending |
| IMP-03 | P1: Mapeamento e normalização | RF-11.1 (etapa 3) | Tasks | Pending |
| IMP-04 | P1: Referências por valor de exibição | RF-11.2, CA-11.1 | Tasks | Pending |
| IMP-05 | P1: Pré-validação e CSV de erros | RF-11.1 (etapa 5) | Tasks | Pending |
| IMP-06 | P1: Execução assíncrona e modos | RF-11.1 (etapas 4 e 6), RN-11.2, RN-11.4, CA-11.2 | Tasks | Pending |
| IMP-07 | P1: Permissões e limites | RN-11.1, RN-11.3 | Tasks | Pending |
| IMP-08 | P1: Histórico e retenção | RF-11.4 | Tasks | Pending |
| IMP-09 | P1: Arquivos | RF-11.5, RN-11.3 | Tasks | Pending |
| IMP-10 | P1: Campos FILE/IMAGE | §9.2 (F09) | Tasks | Pending |
| IMP-11 | P2: Modelos de importação | RF-11.3 | - | Pending |

**Coverage:** 11 total, 10 mapped to tasks, 1 diferido (V1).

---

## Success Criteria

- [ ] CA-11.1 e CA-11.2 automatizados.
- [ ] Importação de 200 mil linhas × 10 colunas conclui em < 2 min num ambiente local.
