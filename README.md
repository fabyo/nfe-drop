# NFE-DROP
## Stack

[![Go](https://img.shields.io/badge/Go-1.23+-00ADD8?logo=go&logoColor=white)]()
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-DB-4169E1?logo=postgresql&logoColor=white)]()
[![Docker](https://img.shields.io/badge/Docker-Container-2496ED?logo=docker&logoColor=white)]()
[![Docker Compose](https://img.shields.io/badge/Docker_Compose-Orquestração-2496ED?logo=docker&logoColor=white)]()
[![Ansible](https://img.shields.io/badge/Ansible-Automação-EE0000?logo=ansible&logoColor=white)]()
[![Linux](https://img.shields.io/badge/Linux-Host-FCC624?logo=linux&logoColor=black)]()
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-DB-4169E1?logo=postgresql&logoColor=white)]()
[![Redis](https://img.shields.io/badge/Redis-Cache-DC382D?logo=redis&logoColor=white)]()
[![RabbitMQ](https://img.shields.io/badge/RabbitMQ-Fila-FF6600?logo=rabbitmq&logoColor=white)]()
[![Prometheus](https://img.shields.io/badge/Prometheus-Métricas-E6522C?logo=prometheus&logoColor=white)]()
[![Grafana](https://img.shields.io/badge/Grafana-Dashboards-F46800?logo=grafana&logoColor=white)]()
[![Graylog](https://img.shields.io/badge/Graylog-Logs-FF3633?logo=graylog&logoColor=white)]()
[![Wazuh](https://img.shields.io/badge/Wazuh-SIEM-025E8C?logo=wazuh&logoColor=white)]()
[![Git](https://img.shields.io/badge/Git-Controle_de_Versão-F05032?logo=git&logoColor=white)]()
[![GitHub](https://img.shields.io/badge/GitHub-Repo-181717?logo=github&logoColor=white)]()


<img src="nfe-drop.png" alt="Golang" width="200" />

## Objetivo do Projeto:
- O **nfe-drop** é um motor de ingestão de NF-e, preparado para grandes volumes, que valida, parseia, enfileira, processa e persiste cada nota com logs estruturados e métricas exportadas, pronto para rodar em produção com monitoramento completo
- Monitora pastas (incoming/, processing/, ZIP e XML) usando watcher em **Go**.
- Consegue receber lotes gigantes de arquivos e ir empilhando na fila e processando tudo muito rapido.

### Fluxo de alto nível

```mermaid
flowchart LR
  subgraph Ingest
    A["Linux inotify / fsnotify watcher"] --> B["incoming/"]
    B --> C["processing/"]
  end

  C -->|job JSON| Q["RabbitMQ nfe-drop-jobs"]
  Q --> WK["nfe-drop-worker - pool de workers"]

  WK -->|parse XML + valida XSD| P["PostgreSQL (nfe, itens, duplicatas, pagamentos, xml)"]
  WK -->|logs JSON| L["Graylog / Filebeat"]
  WK -->|métricas| M["Prometheus → Grafana"]

  H["Hosts / agentes"] --> Z["Wazuh"]
  Z --> L
```

Pipeline de processamento de NF-e em **Go**, com:

- monitoramento de diretório via **inotify** (`fsnotify`)
- validação de XML com **XSD da Sefaz** (libxml2/Go)
- **fila RabbitMQ** + **workers paralelos** em Go
- persistência em **PostgreSQL**
- **logs estruturados em JSON**
- métricas expostas para **Prometheus / Grafana**
- integração preparada para **Graylog** e **Wazuh**
- provisionamento completo via **Ansible**

Focado em **alto throughput**, robustez e observabilidade. O sistema esta preparado para consumir grandes volumes de NFE.

---

## 1. Visão Geral da Arquitetura

Componentes:

- **Watcher**  
  Serviço systemd em Go que monitora a pasta `incoming/` via fsnotify/inotify:
  - move arquivos válidos para `processing/`
  - descarta qualquer arquivo que nao seja .zip/.xml e lixo (ex.: `Zone.Identifier`) em `ignored/`
  - publica jobs em **RabbitMQ** (tipo `xml` ou `zip`)

- **Worker**  
  Serviço systemd em Go que:
  - consome jobs do RabbitMQ (`nfe-drop-jobs`)
  - valida XML com XSD da **Sefaz**
  - faz parse completo dos campos relevantes (cabeçalho, itens, duplicatas, pagamentos)
  - persiste no **PostgreSQL**
  - move o arquivo para `processed/` ou `failed/`
  - emite métricas para Prometheus e logs JSON (prontos pra Graylog / Wazuh)

- **Infra de apoio**
  - **RabbitMQ** (fila de jobs, com DLX configurado)
  - **Redis** (já previsto; hoje não é obrigatório no fluxo principal)
  - **PostgreSQL** (banco da aplicação + backup diário)
  - **Prometheus + Grafana** (observabilidade)
  - **Graylog** (central de logs)
  - **Wazuh** (SIEM / segurança, via docker-compose oficial)

- **Automação**
  - **Ansible** provisiona tudo: Go, Postgres + DB + migrations, RabbitMQ (docker), Redis, systemd services, observabilidade e Wazuh.

---

## 2. Tecnologias Principais

- **Linguagem: Go**
- **Watcher / Worker:**
  - `github.com/fsnotify/fsnotify` – inotify para monitorar diretórios
  - `github.com/lestrrat-go/libxml2` + `github.com/terminalstatic/go-xsd-validate` – validação XSD
  - `encoding/xml` – parse nativo de XML
  - PostgreSQL via `github.com/jackc/pgx/v5/stdlib`
  - Logging estruturado em JSON (ex.: `log/slog` / logger custom estruturado)
- **Fila:**
  - **RabbitMQ** via `github.com/rabbitmq/amqp091-go`
- **Métricas:**
  - `github.com/prometheus/client_golang/prometheus/promhttp`
- **Infra:**
  - **PostgreSQL** nativo
  - **Redis** nativo (reservado para cache/fila futura)
  - **Wazuh** (single-node) via docker-compose oficial
  - **Graylog + OpenSearch + Mongo** via docker-compose
  - **Prometheus + Grafana** via docker-compose
- **Automação / DevOps:**
  - **Ansible** (playbook `ansible/site.yml` + roles dedicadas)
  - **systemd** (services gerados dinamicamente via templates)

---

## 3. Modelo de Dados (PostgreSQL)

### Tabelas principais otimizadas

- `nfe` – cabeçalho da NF-e
- `nfe_xml` – XML bruto + representação JSON
- `nfe_item` – itens da nota
- `nfe_duplicatas` – duplicatas (cobranca)
- `nfe_pagamentos` – formas de pagamento

### Diagrama simplificado

```mermaid
erDiagram
    NFE {
        BIGSERIAL id PK
        CHAR(44)  chave_acesso  "única"
        CHAR(64)  hash_integridade "SHA-256 do XML"
        SMALLINT  modelo
        INTEGER   serie
        INTEGER   numero
        TIMESTAMP emissao
        SMALLINT  tipo_operacao
        SMALLINT  tipo_ambiente
        VARCHAR   natureza_operacao
        VARCHAR   protocolo_autorizacao
        TIMESTAMP data_autorizacao
        SMALLINT  codigo_status
        CHAR(14)  emitente_cnpj
        VARCHAR   emitente_razao
        CHAR(14)  dest_cnpj_cpf
        VARCHAR   dest_razao
        NUMERIC   valor_total_nota
        NUMERIC   valor_produtos
        NUMERIC   valor_desconto
        NUMERIC   valor_icms
        NUMERIC   valor_ipi
        NUMERIC   valor_pis
        NUMERIC   valor_cofins
        NUMERIC   valor_ii
        NUMERIC   valor_frete
        NUMERIC   valor_seguro
        SMALLINT  modalidade_frete
        TIMESTAMP created_at
        TIMESTAMP updated_at
    }

    NFE_XML {
        BIGINT  nfe_id PK,FK
        TEXT    xml_raw
        JSONB   xml_json
        TIMESTAMP created_at
        TIMESTAMP updated_at
    }

    NFE_ITEM {
        BIGSERIAL id PK
        BIGINT  nfe_id FK
        INTEGER n_item
        VARCHAR codigo
        VARCHAR codigo_ean
        VARCHAR descricao
        CHAR(8) ncm
        CHAR(4) cfop
        VARCHAR unidade
        NUMERIC quantidade
        NUMERIC valor_unit
        NUMERIC valor_total_bruto
        NUMERIC valor_frete
        NUMERIC valor_seguro
        NUMERIC valor_desconto
        NUMERIC valor_outros
        SMALLINT ind_total
        NUMERIC base_calculo_icms
        NUMERIC valor_icms
        NUMERIC base_calculo_icms_st
        NUMERIC valor_icms_st
        NUMERIC valor_ipi
        NUMERIC valor_pis
        NUMERIC valor_cofins
    }

    NFE_DUPLICATAS {
        BIGSERIAL id PK
        BIGINT nfe_id FK
        VARCHAR numero_duplicata
        DATE    data_vencimento
        NUMERIC valor_duplicata
    }

    NFE_PAGAMENTOS {
        BIGSERIAL id PK
        BIGINT nfe_id FK
        SMALLINT indicador_pagamento
        VARCHAR  meio_pagamento
        NUMERIC  valor_pagamento
        CHAR(14) cnpj_credenciadora
        CHAR(2)  bandeira_cartao
        VARCHAR  codigo_autorizacao
    }

    NFE ||--|{ NFE_XML        : "1:1 XML"
    NFE ||--|{ NFE_ITEM       : "1:N Itens"
    NFE ||--|{ NFE_DUPLICATAS : "1:N Duplicatas"
    NFE ||--|{ NFE_PAGAMENTOS : "1:N Pagamentos"
```

---

## 4. Pré-requisitos

No host (Ubuntu/Debian-like):

- `sudo` configurado
- Python 3.x (para Ansible)
- Acesso à internet (para baixar dependências, docker images, etc.)

Tudo o resto o **Ansible** cuida:

- Go
- PostgreSQL (servidor, DB, usuário, tabelas)
- Redis
- Docker + Docker Compose plugin
- RabbitMQ (via Docker)
- Grafana + Prometheus (via Docker)
- Graylog + OpenSearch + Mongo (via Docker)
- Wazuh (single-node, via Docker)
- systemd units para watcher / worker
- 💾🛡️ backup diário do Postgres

---

## 5. Instalação Rápida (modo “1 comando”)

1. Entre na pasta do projeto:

```bash
cd ~/golang/nfe-drop/ansible
```

2. Teste a conexão Ansible:

```bash
ansible -i inventory.ini local -m ping
```

3. Rode o playbook (vai pedir senha do sudo):

```bash
ansible-playbook -i inventory.ini site.yml -K
```

Esse playbook vai:

- instalar Go (caso não tenha)
- instalar e configurar PostgreSQL
  - criar DB `nfe_drop` se não existir
  - rodar migrations (tabelas nfe, nfe_item, etc.)
- instalar Docker + Compose plugin
- subir RabbitMQ em Docker
- instalar Redis
- compilar o projeto Go (watcher + worker)
- instalar systemd units do `nfe-drop-watcher` e `nfe-drop-worker`
- garantir que os serviços estejam **habilitados e rodando**
- subir stack de observabilidade (Prometheus, Grafana, Graylog)
- subir Wazuh single-node com certificados gerados

---

## 6. Diretórios Importantes

Por padrão, o Ansible usa:

- **Código-fonte / build:**

  ```bash
  /home/<user>/golang/nfe-drop
  ```

- **Diretório de runtime / serviços (pastas da aplicação, stacks docker):**

  ```bash
  /opt/nfe-drop-services
  ```

Dentro de `/opt/nfe-drop-services`:

- `incoming/` – onde o usuário dropa XML/ZIP
- `processing/` – arquivos em processamento
- `processed/` – arquivos já processados com sucesso
- `failed/` – arquivos que falharam (XML inválido, duplicidade, erro de DB, etc.)
- `ignored/` – arquivos que nao sao xml e **Zone.Identifier** é sempre apagado, problema comum em **WSL** 
- `tmp/` – temporários (ex.: extração de ZIP)
- `deploy/` – docker-compose de Rabbit, Grafana, Prometheus, Graylog, Wazuh, etc.

---

## 7. Serviços e Portas

### Aplicação Go

- `nfe-drop-watcher.service`
  - systemd
  - monitora `incoming/`
- `nfe-drop-worker.service`
  - systemd
  - consome fila e processa os XML/ZIP

Portas de métricas (Prometheus scrape):

- Watcher: `:9100` → `http://localhost:9100/metrics`
- Worker: `:9101` → `http://localhost:9101/metrics`

### RabbitMQ

- Porta AMQP: `5672`
- Management UI: `15672` → `http://localhost:15672`  
  - usuário padrão (no compose): `guest` / `guest` (ou ajustado via env/role)

### Redis

- Porta padrão: `6379`

### Prometheus

- `http://localhost:9090`

### Grafana

- `http://localhost:3000`  
  (usuário / senha padrão: `admin` / `admin` – troque na primeira vez)

### Graylog

- Web UI (via compose observability):  
  `http://localhost:9000`  
  (usuário padrão geralmente `admin` — senha definida via env no compose)

### Wazuh

- Dashboard:
  - **HTTPS** em: `https://localhost:5601` (ajustado via Ansible)
- Indexer: `http://localhost:9200` (não é pra acesso direto em produção)
- Manager:
  - Porta API: `55000`
  - Syslog/agents: `1514-1516`, etc.

---

## 8. Fluxo de Processamento (Detalhado)

1. **Upload / Drop de arquivos**
   - Você copia arquivos `.xml` ou `.zip` para `incoming/`
   - Qualquer lixo é detectado e movido para `ignored/`

2. **Watcher**
   - Vê o arquivo novo via inotify
   - Move para `processing/`
   - Monta um job JSON:
     - tipo (`xml` ou `zip`)
     - caminho completo do arquivo
   - Publica na fila RabbitMQ (`nfe-drop-jobs`)

3. **Worker**
   - Consome o job da fila
   - Se for ZIP:
     - extrai todos os XML para `tmp/<zipname>/`
     - processa cada XML individualmente
     - você pode ter cenário:
       - algumas notas novas → sucesso
       - algumas duplicadas → inserção ignorada apenas para aquelas
     - o zip é movido para `processed/` **se pelo menos uma nota foi processada com sucesso** e nenhuma falha crítica
   - Se for XML:
     - valida contra XSD da Sefaz
     - faz parse completo:
       - cabeçalho (nfe)
       - itens (nfe_item)
       - duplicatas (nfe_duplicatas)
       - pagamentos (nfe_pagamentos)
       - XML bruto em `nfe_xml`
     - calcula `hash_integridade` (ex. SHA-256 do XML) para garantir unicidade
     - insere tudo num **único fluxo transacional** no PostgreSQL
     - em caso de `duplicate key` (chave_acesso):
       - trata a NF-e como já processada
       - loga evento e envia para métricas
   - Ao final:
     - sucesso → move arquivo para `processed/`
     - falha validada → `failed/` (com logs indicando o motivo)

4. **Observabilidade**
   - Cada NF-e gera:
     - incremento em `nfe_processed_total{status="success|failed|duplicate", source="xml|zip"}`
     - duração em `nfe_process_duration_seconds`
   - Logs estruturados permitem:
     - filtros no Graylog/Wazuh por `chave_acesso`, `emitente_cnpj`, tipo de erro, etc.

---

## 9. Ansible: o que ele faz por você

Playbook principal: `ansible/site.yml`

Roles (resumo):

- `go`
  - instala Go no host (se não tiver)
- `nfe-drop-db`
  - instala PostgreSQL
  - cria banco `nfe_drop`
  - roda migrations via binário `nfe-drop-migrator` ou `go run`
- `postgres-backup`
  - configura backup diário (cron) com `pg_dump` do `nfe_drop`
- `docker-engine`
  - instala `docker.io` + plugin `docker compose`
- `redis-server`
  - instala Redis nativo
- `nfe-drop-rabbitmq-docker`
  - sobe RabbitMQ via docker-compose (com env configurado)
- `nfe-drop-app`
  - garante diretórios `incoming/`, `processing/`, `processed/`, `failed/`, `ignored/`, `tmp/`
  - compila watcher e worker (`go build`)
  - instala systemd units com usuário dinâmico (`login_user`)
  - habilita e inicia `nfe-drop-watcher` e `nfe-drop-worker`
- `nfe-drop-observability`
  - sobe stack Prometheus + Grafana (docker-compose)
  - adiciona targets para `:9100` e `:9101`
- `nfe-drop-graylog`
  - sobe Graylog + Mongo + OpenSearch em docker-compose
- `nfe-drop-wazuh`
  - clona `wazuh-docker` na tag estável `v4.8.0`
  - ajusta `docker-compose.yml` da topologia `single-node`
  - gera certificados via `generate-indexer-certs.yml`
  - sobe stack e exibe instruções de acesso

---

## 10. Como rodar localmente (sem Ansible, modo dev)

Se quiser debugar só o Go, ignorando Ansible:

1. Ajuste o `.env` na raiz:

```env
# DB
NFE_DROP_DB_HOST=localhost
NFE_DROP_DB_PORT=5432
NFE_DROP_DB_USER=...
NFE_DROP_DB_PASSWORD=...
NFE_DROP_DB_NAME=nfe_drop
NFE_DROP_DB_SSLMODE=disable

# RabbitMQ
NFE_DROP_RABBITMQ_URL=amqp://guest:guest@localhost:5672/

# Base de diretórios (opcional)
NFE_DROP_BASE_DIR=/home/<user>/nfe-drop

# Logs / métricas
NFE_DROP_LOG_LEVEL=info
NFE_DROP_WATCHER_METRICS_ADDR=:9100
NFE_DROP_WORKER_METRICS_ADDR=:9101
```

2. Rode migrations:

```bash
go run ./cmd/nfe-drop-migrator
# ou
go run ./cmd/nfe-drop-migrator --force   # para recriar banco do zero
```

3. Suba o watcher e o worker:

```bash
go run ./cmd/nfe-drop-watcher
go run ./cmd/nfe-drop-worker
```

---

## 11. Comandos úteis

### Systemd

```bash
# status
systemctl status nfe-drop-watcher
systemctl status nfe-drop-worker

# logs
journalctl -u nfe-drop-watcher -f
journalctl -u nfe-drop-worker -f

# restart
sudo systemctl restart nfe-drop-watcher
sudo systemctl restart nfe-drop-worker
```

### Docker

```bash
# RabbitMQ
docker ps | grep rabbit
docker logs -f nfe-drop-rabbitmq

# Observabilidade
docker ps | egrep "grafana|prometheus|graylog|opensearch|mongo"

# Wazuh
cd /opt/nfe-drop-services/deploy/wazuh-docker/single-node
docker compose ps
docker logs -f single-node-wazuh.indexer-1
docker logs -f single-node-wazuh.dashboard-1
```

### Ansible

```bash
cd ansible

# Executar playbook completo
ansible-playbook -i inventory.ini site.yml -K
```

---

## 12. Roadmap / Ideias Futuras

- Enriquecimento de NF-e (ex.: cache de NCM, CFOP, CST em Redis)
- Exposição de API REST para consulta das NF-e (Go ou PHP/Laravel separado)
- UI para monitoramento da fila e reprocessamento manual
- Feature flags para:
  - modo somente-validação (sem persistir)
- Exportação de métricas específicas por:
  - CNPJ emitente
  - código de status da Sefaz
  - volume por período (hora/dia/semana)

---

- **Go idiomático**, com separação clara de responsabilidades:
  - `internal/watcher`
  - `internal/worker`
  - `internal/nfe` (parse e validação)
  - `internal/storage` (persistência)
  - `internal/config` (config centralizada)
- Integração com **RabbitMQ**, **PostgreSQL**, **Redis**
- Uso de **XSD real da Sefaz** para validação forte de NF-e
- **infra as code** com **Ansible**
- Observabilidade completa:
  - **Prometheus/Grafana** (métricas)
  - **Graylog** (logs estruturados)
  - **Wazuh** (segurança e SIEM)
- Provisionamento automatizado:  
  `ansible-playbook -i inventory.ini site.yml -K`

---
