# F01 — Distribuição e Instalação Specification

**Prioridade:** MVP · **Depende de:** F00 · **Origem:** F01 do `produto.specs.md`

## Problem Statement

Uma PME precisa subir o DataBoard com um único comando, sem compilar nada, e atualizar de versão sem perder dados. Hoje (pós-F00) só existe um compose de desenvolvimento. Falta o artefato de distribuição, o setup inicial guiado e as proteções de operação (chave obrigatória, proteção contra downgrade, backup).

## Goals

- [ ] `docker compose up -d` com o compose oficial exibe o wizard em `http://localhost:8080` em ≤ 60 s.
- [ ] Atualizar a tag de imagem e reiniciar migra o banco sem perda de dados.
- [ ] Credenciais sensíveis ficam cifradas com chave derivada de `DATABOARD_SECRET_KEY`.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Kubernetes/Helm, instalador ISO/VM, alta disponibilidade multi-nó | Fora do produto (escopo F01 original) |
| Login, troca de senha, sessões | F03 |
| Tela completa de configurações | F02 |
| Seleção de idioma no wizard | AD-014: apenas pt-BR |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Imagens | `databoard/app:<semver>` e `databoard/analytics:<semver>` + `:latest`, multi-arch (`amd64`, `arm64`) | AD-001 (duas imagens) substitui a imagem única do RF-01.1 | y |
| Variável de token interno | `DATABOARD_INTERNAL_TOKEN` obrigatória, como `DATABOARD_SECRET_KEY` | AD-002 | y |
| Campo `locale` do `POST /setup` | Removido; idioma fixo pt-BR | AD-014 | y |
| Tabela de usuários | Criada nesta feature (mínima), estendida na F03/F04 | O wizard precisa criar o primeiro admin | y |
| Hash de senha | Argon2id (Spring Security `Argon2PasswordEncoder`) definido aqui | RF-03.1; o wizard já grava senha | y |
| Derivação de chave | HKDF-SHA256 de `DATABOARD_SECRET_KEY` com `info="databoard-credentials-v1"`, AES-256-GCM com nonce de 12 bytes por valor | RN-01.2 | y |
| Tamanho mínimo de `DATABOARD_SECRET_KEY` | 32 caracteres | Entropia mínima razoável para HKDF | y |
| Proteção contra downgrade | Flyway com `ignore-migration-patterns` vazio: migração aplicada desconhecida pela app aborta o startup | RF-01.6 | y |
| CA-01.2 (upgrade sem perda) | Verificado por E2E de upgrade no pipeline de release a partir da 1ª versão publicada; no MVP, coberto pela regra "migrações só aditivas" + teste do guard | Ainda não há versão anterior para testar | y |
| Senha do primeiro admin | Regras padrão da política de senha (mín. 10 caracteres, letras e números) | F02/F03 tornam configurável depois | y |

**Open questions:** none - todas resolvidas ou registradas acima.

---

## User Stories

### P1: Subir com compose oficial ⭐ MVP

**User Story**: Como administrador de TI de uma PME, quero subir o DataBoard com o compose oficial para não precisar compilar nada.

**Why P1**: Porta de entrada do produto.

**Acceptance Criteria**:
1. WHEN o administrador executa `docker compose up -d` com `deploy/docker-compose.yml` e um `.env` preenchido THEN o sistema SHALL exibir o wizard de setup em `http://localhost:8080` em até 60 s.
2. The compose oficial SHALL declarar os serviços `app`, `analytics` e `postgres` e os volumes nomeados `databoard-db` e `databoard-files`.
3. IF `DATABOARD_SECRET_KEY` estiver ausente ou com menos de 32 caracteres THEN o app SHALL encerrar no startup com código de saída diferente de zero e log `DATABOARD_SECRET_KEY ausente ou curta (mínimo 32 caracteres)`.
4. IF `DATABOARD_INTERNAL_TOKEN` estiver ausente THEN o app e o analytics SHALL encerrar no startup com log explicativo.
5. WHEN o healthcheck do compose chama `GET /actuator/health/readiness` THEN o app SHALL responder `UP` apenas quando banco e analytics estiverem acessíveis.

**Independent Test**: E2E com o compose oficial: subir, ver o wizard; subir sem `DATABOARD_SECRET_KEY` e ver o container sair com o log esperado.

---

### P1: Wizard de setup inicial ⭐ MVP

**User Story**: Como administrador, quero configurar empresa, fuso e o primeiro admin no primeiro acesso para começar a usar o sistema.

**Why P1**: Sem o primeiro admin, ninguém entra.

**Acceptance Criteria**:
1. WHILE não existir nenhum usuário THEN `GET /api/v1/setup/status` SHALL responder `{ "setupRequired": true }`.
2. WHEN `POST /api/v1/setup` recebe `companyName`, `timezone`, `admin { name, email, password }` e `smtp` opcional válidos THEN o sistema SHALL criar o usuário com papel `ADMIN`, gravar as configurações e responder 201.
3. IF existe pelo menos um usuário WHEN `POST /api/v1/setup` é chamado THEN o sistema SHALL responder 404.
4. IF a senha do admin não cumpre a política padrão THEN o sistema SHALL responder 400 `VALIDATION_ERROR` com erro no campo `admin.password`.
5. WHEN o frontend carrega e `setupRequired=true` THEN o frontend SHALL redirecionar qualquer rota para `/setup`.
6. The senha do admin SHALL ser armazenada com hash Argon2id, nunca em texto puro.

**Independent Test**: IT: status true → POST 201 → status false → novo POST 404.

---

### P1: Cifragem de credenciais ⭐ MVP

**User Story**: Como administrador, quero que senhas de conexões e de SMTP fiquem cifradas em repouso para que um dump do banco não as exponha.

**Why P1**: Pré-requisito da F02 (SMTP) e da F07 (conexões).

**Acceptance Criteria**:
1. The `CryptoService` SHALL cifrar valores com AES-256-GCM e chave derivada por HKDF de `DATABOARD_SECRET_KEY`, gerando um nonce aleatório por valor.
2. WHEN um valor cifrado é decifrado com a mesma chave THEN o `CryptoService` SHALL devolver o texto original.
3. IF a chave mudou desde a cifragem THEN o `CryptoService` SHALL lançar `CREDENTIAL_UNREADABLE`, sem devolver lixo.

**Independent Test**: Teste unitário de ida e volta e de chave trocada.

---

### P1: Proteção de versão e informações do sistema ⭐ MVP

**User Story**: Como administrador, quero que o sistema recuse rodar sobre um banco mais novo e mostre a versão instalada, para operar atualizações com segurança.

**Why P1**: Evita corromper dados num downgrade acidental.

**Acceptance Criteria**:
1. WHEN o app sobe com migrações pendentes THEN o Flyway SHALL aplicá-las antes de aceitar requisições.
2. IF o banco contém uma migração aplicada que a versão do app não conhece THEN o app SHALL abortar o startup com log `Banco em versão mais nova que a aplicação; downgrade não suportado`.
3. WHEN `GET /api/v1/system/info` é chamado THEN o sistema SHALL responder `{ version, commit, schemaVersion, buildDate }`.
4. WHEN um usuário autenticado abre a página **Sobre** THEN o frontend SHALL exibir versão, commit, versão do schema e link para o changelog.

**Independent Test**: IT que insere manualmente uma linha futura em `flyway_schema_history` e verifica que o contexto falha ao subir.

---

### P1: Backup, restore e upgrade documentados ⭐ MVP

**User Story**: Como administrador, quero scripts de backup e restore para recuperar a instalação em caso de falha.

**Why P1**: Self-hosted sem backup é risco inaceitável.

**Acceptance Criteria**:
1. WHEN o administrador executa `deploy/scripts/backup.sh <destino>` THEN o script SHALL gerar um `pg_dump` em formato custom e um `tar.gz` do volume `databoard-files` com timestamp no nome.
2. WHEN o administrador executa `deploy/scripts/restore.sh <arquivo-db> <arquivo-files>` THEN o script SHALL restaurar banco e arquivos com o app parado e abortar se o app estiver rodando.
3. The documentação `docs/operacao.md` SHALL descrever instalação, upgrade (trocar tag + reiniciar) e restauração.

**Independent Test**: Executar backup, apagar volumes, restaurar e ver o admin criado ainda conseguir ser listado.

---

### P2: Publicação de imagens por CI

**User Story**: Como mantenedor, quero que cada tag gere as imagens multi-arch automaticamente.

**Why P2**: Necessário para a primeira release pública, não para o desenvolvimento do MVP.

**Acceptance Criteria**:
1. WHEN uma tag `vX.Y.Z` é criada THEN o workflow de release SHALL publicar `databoard/app` e `databoard/analytics` com tags `X.Y.Z` e `latest` para `amd64` e `arm64`.

**Independent Test**: Executar o workflow em modo `--dry-run` (buildx sem push).

---

## Edge Cases

- IF dois `POST /api/v1/setup` concorrentes chegam com zero usuários THEN o sistema SHALL criar apenas um admin (lock transacional) e responder 404 ao segundo.
- IF o e-mail do admin no wizard é inválido THEN o sistema SHALL responder 400 com erro em `admin.email`.
- IF o SMTP informado no wizard falhar no teste THEN o sistema SHALL concluir o setup assim mesmo (SMTP opcional) e gravar SMTP como "não verificado".
- WHEN o `timezone` não é um ID IANA válido THEN o sistema SHALL responder 400 em `timezone`.

---

## Requirement Traceability

| Requirement ID | Story | Origem | Phase | Status |
| --- | --- | --- | --- | --- |
| INST-01 | P1: Compose oficial | RF-01.2 | Tasks | Pending |
| INST-02 | P1: Compose oficial (variáveis) | RF-01.3, RF-01.4 | Tasks | Pending |
| INST-03 | P1: Compose oficial (readiness) | RF-01.7 | Tasks | Pending |
| INST-04 | P1: Wizard | RF-01.5, RN-01.1, CA-01.3 | Tasks | Pending |
| INST-05 | P1: Wizard (front) | RF-01.5 | Tasks | Pending |
| INST-06 | P1: Cifragem | RN-01.2 | Tasks | Pending |
| INST-07 | P1: Proteção de versão | RF-01.6, CA-01.2 | Tasks | Pending |
| INST-08 | P1: Info do sistema | RF-01.9 | Tasks | Pending |
| INST-09 | P1: Backup/restore | RF-01.8 | Tasks | Pending |
| INST-10 | P2: Publicação de imagens | RF-01.1 | Tasks | Pending |

**Coverage:** 10 total, 10 mapped to tasks, 0 unmapped.

---

## Success Criteria

- [ ] CA-01.1: wizard em ≤ 60 s após `docker compose up -d`.
- [ ] CA-01.3: setup concluído → `POST /setup` responde 404.
