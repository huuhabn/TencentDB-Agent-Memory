# Deployment and Integration Guide (Open-source Standalone / Cloud Service)

> 📖 **This document covers deployment forms, Hermes integration, and end-to-end verification.**
> For the plugin's core capabilities, configuration parameters, and CLI tools, go back to the **[main README](README.md)**.

`memory-tencentdb` offers **two independent deployment forms**. Both can be called by external Agents (typically Hermes) through the HTTP API, and each fits a different deployment scale and operations requirement:

| Form | Storage backend | State backend | Multi-tenancy | Suitable scenarios |
|------|----------|----------|--------|----------|
| **Standalone (open-source single-machine)** | SQLite + local files | In-process Map / Timer | Single space | Local development, single-Agent sidecar, all-in-one Docker, offline deployment |
| **Service (cloud service)** | TCVDB + COS | Redis (distributed lock + task queue) | Multiple spaces, per-`service_id` | K8s multi-replica, multi-tenant SaaS, shared memory across multiple Agents |

```
L0  Raw conversation records (Conversation)   ← written automatically
L1  Atomic structured memory (Atomic Memory)  ← LLM extraction + deduplication
L2  Scene blocks (Scene Blocks)               ← LLM scene extraction
L3  User profile (Persona)                    ← LLM persona synthesis
```

Both forms share the same Gateway binary and the same v1/v2 HTTP API; only the configuration and backends differ. Switching forms only requires changing the `TDAI_DEPLOY_MODE` environment variable.

---

## Quick Start (3 steps)

```bash
# 1. Enter MemoryCore and install dependencies
cd MemoryCore
npm install

# 2. Configure the LLM
export TDAI_LLM_API_KEY="your-api-key"
export TDAI_LLM_BASE_URL="https://api.deepseek.com/v1"
export TDAI_LLM_MODEL="deepseek-chat"

# 3. Start the Gateway
npx tsx src/gateway/server.ts
```

The Gateway listens on `http://127.0.0.1:8420` by default, and data is stored in `~/.memory-tencentdb/memory-tdai/`.

---

## Deployment Modes

### Standalone mode (single machine)

Zero external dependencies; all data is stored locally. Suitable for: local development, single-Agent sidecar, all-in-one Docker deployment.

**Storage**: SQLite (vectors + records) + local filesystem (L2/L3 documents)
**State management**: in-process Map/Timer

#### Environment variable configuration

```bash
# Required — LLM configuration
export TDAI_LLM_API_KEY="sk-xxx"
export TDAI_LLM_BASE_URL="https://api.deepseek.com/v1"   # Default https://api.openai.com/v1
export TDAI_LLM_MODEL="deepseek-chat"                     # Default gpt-4o
export TDAI_LLM_MAX_TOKENS=4096
export TDAI_LLM_TIMEOUT_MS=120000

# Optional — service configuration
export TDAI_GATEWAY_PORT=8420            # Listening port, default 8420
export TDAI_GATEWAY_HOST="127.0.0.1"    # Listening address, default 127.0.0.1
export TDAI_DATA_DIR="~/.memory-tencentdb/memory-tdai"  # Data directory
```

#### YAML configuration file (optional)

Config file search order: `$TDAI_GATEWAY_CONFIG` → `./tdai-gateway.yaml` → `<dataDir>/tdai-gateway.yaml`

```yaml
# tdai-gateway.yaml — Standalone mode
server:
  port: 8420
  host: "127.0.0.1"

data:
  baseDir: "~/.memory-tencentdb/memory-tdai"

llm:
  baseUrl: "https://api.deepseek.com/v1"
  apiKey: "${TDAI_LLM_API_KEY}"
  model: "deepseek-chat"
  maxTokens: 4096
  timeoutMs: 120000

# memory configuration (optional, all have reasonable defaults)
memory:
  capture:
    enabled: true
    excludeAgents: []
  recall:
    maxResults: 5
    scoreThreshold: 0.3
    strategy: "hybrid"            # hybrid / embedding / keyword
  embedding:
    enabled: true
    provider: "openai"            # none / openai / deepseek / qclaw
    baseUrl: "${TDAI_LLM_BASE_URL}"
    apiKey: "${TDAI_LLM_API_KEY}"
    model: "text-embedding-3-small"
    dimensions: 1536
  bm25:
    enabled: true
    language: "zh"
  storeBackend: "sqlite"          # sqlite (standalone) or tcvdb (service)
  pipeline:
    everyNConversations: 5
    enableWarmup: true
    l1IdleTimeoutMs: 30000
    l2IntervalMs: 300000
    l3IntervalMs: 600000
```

#### Docker deployment

```bash
# Gateway only
docker run -d \
  -e TDAI_LLM_API_KEY="sk-xxx" \
  -e TDAI_LLM_BASE_URL="https://api.deepseek.com/v1" \
  -e TDAI_LLM_MODEL="deepseek-chat" \
  -e TDAI_GATEWAY_HOST="0.0.0.0" \
  -p 8420:8420 \
  -v tdai-data:/root/.memory-tencentdb/memory-tdai \
  agentmemory/hermes-memory:latest
```

#### Data directory structure

```
~/.memory-tencentdb/memory-tdai/
  ├── vectors.db              # SQLite vector database (L0 + L1)
  ├── conversations/          # L0 raw conversation JSONL
  ├── records/                # L1 structured memory
  ├── scene_blocks/           # L2 scene Markdown files
  ├── persona.md              # L3 user profile
  └── checkpoint.json         # Pipeline progress
```

---

### Service mode (service-oriented)

Uses external storage (TCVDB vector database + COS object storage) and supports horizontal scaling with multiple replicas. Suitable for: K8s clusters, multi-tenant SaaS, shared memory across multiple Agents.

**Storage**: TCVDB (vector search) + COS (L2/L3 documents, per-serviceId path isolation)
**State management**: Redis (distributed lock + task queue)
**Configuration source**: Shark service (dynamic VDB/COS credentials) or environment variables (static credentials)

#### Environment variable configuration

```bash
# ── Deployment mode ──
export TDAI_DEPLOY_MODE="service"           # Key: enables service mode

# ── LLM (same as standalone) ──
export TDAI_LLM_API_KEY="sk-xxx"
export TDAI_LLM_BASE_URL="https://api.deepseek.com/v1"
export TDAI_LLM_MODEL="deepseek-chat"

# ── Service port ──
export TDAI_GATEWAY_PORT=3100
export TDAI_GATEWAY_HOST="0.0.0.0"

# ── Redis (distributed state backend) ──
export STATE_BACKEND="redis"                # redis or local (single-machine testing)
export REDIS_HOST="redis.example.com"
export REDIS_PORT=6379
export REDIS_PASSWORD="your-password"
export REDIS_KEY_PREFIX="tdai_memory"

# ── VDB vector database (direct-connect mode) ──
export VDB_ENDPOINT="http://vdb.example.com:8100"
export VDB_USER="root"
export VDB_API_KEY="your-vdb-api-key"
export VDB_DATABASE="memory-production"

# ── COS object storage (direct-connect mode) ──
export COS_SECRET_ID="AKIDxxxx"
export COS_SECRET_KEY="xxxxx"
export COS_TOKEN=""                         # Fill in when using STS temporary credentials
export COS_URL="https://your-bucket.cos.ap-guangzhou.myqcloud.com"
export COS_PATH_PREFIX="tenants/prod/"

# ── Or use the Shark configuration service (recommended for production) ──
export SHARK_BASE_URL="http://shark.example.com:8080"
# Shark automatically provides per-instance VDB and COS configuration

# ── Optional tuning ──
export CONFIG_VDB_TTL_MS=300000             # VDB config cache TTL, default 5 minutes
export CONFIG_COS_BUFFER_MS=120000          # How early to refresh COS credentials
export CONFIG_MAX_INSTANCES=1000            # Maximum number of cached instances
export SCANNER_SPACES="space1,space2"       # List of spaces scanned by the Timer Scanner
export TDAI_SPACE_ID="default"              # Space ID of the current instance
```

#### YAML configuration file

```yaml
# tdai-gateway.yaml — Service mode
deployMode: service

server:
  port: 3100
  host: "0.0.0.0"

data:
  baseDir: "/data/tdai-memory"

llm:
  baseUrl: "${TDAI_LLM_BASE_URL}"
  apiKey: "${TDAI_LLM_API_KEY}"
  model: "deepseek-chat"

memory:
  storeBackend: "tcvdb"
  tcvdb:
    embeddingModel: "bge-large-zh"    # VDB server-side embedding model
    timeout: 10000
  embedding:
    enabled: false                     # TCVDB has server-side embedding built in; the client does not need it
    provider: "none"
  bm25:
    enabled: true
    language: "zh"
  recall:
    strategy: "hybrid"
    maxResults: 10
```

#### K8s deployment

```yaml
# Core environment variables (injected via ConfigMap/Secret)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tdai-memory-gateway
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: gateway
          image: agentmemory/hermes-memory:latest
          env:
            - name: TDAI_DEPLOY_MODE
              value: "service"
            - name: TDAI_GATEWAY_PORT
              value: "3100"
            - name: TDAI_GATEWAY_HOST
              value: "0.0.0.0"
            - name: STATE_BACKEND
              value: "redis"
            - name: REDIS_HOST
              valueFrom:
                configMapKeyRef:
                  name: tdai-config
                  key: redis-host
            - name: SHARK_BASE_URL
              value: "http://shark-svc:8080"
          ports:
            - containerPort: 3100
---
apiVersion: v1
kind: Service
metadata:
  name: tdai-memory-gateway
spec:
  selector:
    app: tdai-memory-gateway
  ports:
    - port: 3100
      targetPort: 3100
```

#### Multi-replica architecture

```
                    ┌─────────────┐
                    │  Hermes #1  │─┐
                    └─────────────┘ │
                    ┌─────────────┐ │    ┌──────────────────┐    ┌──────────┐
                    │  Hermes #2  │─┼───→│  TDAI Gateway    │───→│  TCVDB   │
                    └─────────────┘ │    │  (N replicas)    │    │  vector  │
                    ┌─────────────┐ │    │                  │───→│  store   │
                    │  Hermes #3  │─┘    │  ┌─ Scanner ─┐   │    └──────────┘
                    └─────────────┘      │  │  Worker   │   │    ┌──────────┐
                                         │  └───────────┘   │───→│   COS    │
  Each Hermes uses a unique               └──────────────────┘    │  object  │
  x-tdai-service-id                              │               │  storage │
  for data isolation                      ┌──────────────┐        └──────────┘
                                          │    Redis     │
                                          │ state + tasks│
                                          └──────────────┘
```

---

## Hermes Plugin Configuration

Two Hermes plugins are provided, for different deployment scenarios.

### v1 plugin: `memory_tencentdb` (single-machine, self-managed)

Automatically starts and manages the Gateway child process, so you do not need to deploy the Gateway manually. Suitable for single-Agent local/Docker deployment.

**Install the plugin**:

```bash
# Symlink (recommended for development)
ln -s "$(pwd)/MemoryCore/hermes-plugin/memory/memory_tencentdb" \
      <hermes-agent>/plugins/memory/memory_tencentdb

# Copy (production deployment)
cp -r MemoryCore/hermes-plugin/memory/memory_tencentdb \
      <hermes-agent>/plugins/memory/memory_tencentdb
```

**Hermes configuration** (`~/.hermes/config.yaml`):

```yaml
memory:
  provider: memory_tencentdb
```

**Environment variables**:

| Variable | Default | Description |
|------|--------|------|
| `TDAI_LLM_API_KEY` | (required) | LLM API key |
| `TDAI_LLM_BASE_URL` | `https://api.openai.com/v1` | LLM API address |
| `TDAI_LLM_MODEL` | `gpt-4o` | LLM model name |
| `MEMORY_TENCENTDB_GATEWAY_PORT` | `8420` | Gateway listening port |
| `MEMORY_TENCENTDB_GATEWAY_HOST` | `127.0.0.1` | Gateway listening address |
| `MEMORY_TENCENTDB_GATEWAY_CMD` | (auto-detected) | Custom Gateway start command |

**Tool list**:

| Tool | Purpose |
|------|------|
| `memory_tencentdb_memory_search` | Search L1 structured memory |
| `memory_tencentdb_conversation_search` | Search L0 raw conversations |

**Features**: automatic Gateway child process startup, health-check watchdog (10s interval), automatic recovery, circuit-breaker protection, background sync thread.

---

### v2 plugin: `memory_tencentdb_v2` (external Gateway)

Connects to an already running Gateway service (local or remote) and communicates through the v2 REST API. Suitable for multiple Agents sharing a Gateway and for K8s cluster deployment.

**Install the plugin**:

```bash
ln -s "$(pwd)/MemoryCore/hermes-plugin/memory/memory_tencentdb_v2" \
      <hermes-agent>/plugins/memory/memory_tencentdb_v2
```

**Install the Python SDK**:

```bash
pip install tdai-memory
```

**Hermes configuration** (`~/.hermes/config.yaml`):

```yaml
memory:
  provider: memory_tencentdb_v2
```

**Environment variables**:

| Variable | Default | Description |
|------|--------|------|
| `TDAI_MEMORY_ENDPOINT` | `http://127.0.0.1:8420` | Gateway service address |
| `TDAI_MEMORY_API_KEY` | `""` | Bearer token (required in service mode) |
| `TDAI_MEMORY_SERVICE_ID` | `""` | Instance/space ID (multi-tenant isolation key) |

**Tool list**:

| Tool | Purpose | Parameters |
|------|------|------|
| `tdai_memory_search` | Search L1 structured memory | `query` (required), `limit` (default 5) |
| `tdai_conversation_search` | Search L0 raw conversations | `query` (required), `limit` (default 5) |
| `tdai_read_scene` | Read L2 scene content | `scene_id` (required) |

**Features**: based on the `tdai_memory` Python SDK (httpx), Bearer token authentication, multi-tenant isolation, circuit breaker (5 failures → 60s cooldown), thread-safe.

---

### Recommendations

| Scenario | Recommended plugin | Deployment mode | Gateway |
|------|----------|----------|---------|
| Local development / single Agent | `memory_tencentdb` (v1) | standalone | Managed by the plugin |
| Single Docker container | `memory_tencentdb` (v1) | standalone | Managed by the plugin |
| Shared memory across multiple Agents | `memory_tencentdb_v2` (v2) | service | Deployed separately |
| K8s cluster | `memory_tencentdb_v2` (v2) | service | K8s Service |
| Multi-tenant SaaS | `memory_tencentdb_v2` (v2) | service | Multiple replicas + Redis |

---

## API Overview

### v1 API (Standalone-compatible)

| Method | Path | Description |
|------|------|------|
| GET | `/health` | Health check |
| POST | `/recall` | Memory recall (prefetch) |
| POST | `/capture` | Conversation capture (sync_turn) |
| POST | `/search/memories` | L1 memory search |
| POST | `/search/conversations` | L0 conversation search |
| POST | `/session/end` | End session + flush |
| POST | `/seed` | Bulk import of historical conversations |

### v2 API (multi-tenant, requires Bearer token + x-tdai-service-id)

| Method | Path | Description |
|------|------|------|
| POST | `/v2/conversation/add` | L0 add conversation |
| POST | `/v2/conversation/query` | L0 query conversations |
| POST | `/v2/conversation/search` | L0 search conversations |
| POST | `/v2/conversation/delete` | L0 delete conversations |
| POST | `/v2/atomic/add` | L1 add memory |
| POST | `/v2/atomic/query` | L1 query memory |
| POST | `/v2/atomic/search` | L1 search memory |
| POST | `/v2/atomic/delete` | L1 delete memory |
| POST | `/v2/scenario/ls` | L2 list scenes |
| POST | `/v2/scenario/read` | L2 read scene |
| POST | `/v2/scenario/write` | L2 write scene |
| POST | `/v2/scenario/rm` | L2 delete scene |
| POST | `/v2/persona/read` | L3 read persona |
| POST | `/v2/persona/write` | L3 write persona |

---

## Configuration Reference

### All environment variables

| Variable | Default | Applicable mode | Description |
|------|--------|----------|------|
| **Gateway basics** |
| `TDAI_DEPLOY_MODE` | `standalone` | All | `standalone` or `service` |
| `TDAI_GATEWAY_PORT` | `8420` | All | Listening port |
| `TDAI_GATEWAY_HOST` | `127.0.0.1` | All | Listening address |
| `TDAI_DATA_DIR` | `~/.memory-tencentdb/memory-tdai` | All | Data directory |
| `TDAI_GATEWAY_CONFIG` | (searched) | All | Config file path |
| **LLM** |
| `TDAI_LLM_API_KEY` | `""` | All | LLM API key |
| `TDAI_LLM_BASE_URL` | `https://api.openai.com/v1` | All | LLM API address |
| `TDAI_LLM_MODEL` | `gpt-4o` | All | Model name |
| `TDAI_LLM_MAX_TOKENS` | `4096` | All | Maximum output tokens |
| `TDAI_LLM_TIMEOUT_MS` | `120000` | All | LLM request timeout |
| **Service mode** |
| `STATE_BACKEND` | (auto) | service | `redis` or `local` |
| `REDIS_HOST` | `127.0.0.1` | service | Redis address |
| `REDIS_PORT` | `6379` | service | Redis port |
| `REDIS_PASSWORD` | (none) | service | Redis password |
| `REDIS_KEY_PREFIX` | `tdai_memory` | service | Redis key prefix |
| **VDB (direct-connect mode)** |
| `VDB_ENDPOINT` | `""` | service | VDB address |
| `VDB_USER` | `root` | service | VDB username |
| `VDB_API_KEY` | `""` | service | VDB API key |
| `VDB_DATABASE` | `default` | service | VDB database name |
| **COS (direct-connect mode)** |
| `COS_SECRET_ID` | (none) | service | COS AK |
| `COS_SECRET_KEY` | (none) | service | COS SK |
| `COS_TOKEN` | (none) | service | COS STS token |
| `COS_URL` | (none) | service | COS bucket URL |
| `COS_PATH_PREFIX` | (none) | service | COS path prefix |
| **Shark (production mode)** |
| `SHARK_BASE_URL` | (none) | service | Shark configuration service address |
| **Tuning** |
| `CONFIG_VDB_TTL_MS` | `300000` | service | VDB config cache TTL |
| `CONFIG_COS_BUFFER_MS` | `120000` | service | How early to refresh COS credentials |
| `CONFIG_MAX_INSTANCES` | `1000` | service | Maximum number of cached instances |
| `SCANNER_SPACES` | `default` | service | Scanner space list |
| `TDAI_SPACE_ID` | `default` | service | Current space ID |

---

## Typical Deployment Examples

### Example 1: Local development (simplest)

```bash
export TDAI_LLM_API_KEY="sk-xxx"
export TDAI_LLM_BASE_URL="https://api.deepseek.com/v1"
export TDAI_LLM_MODEL="deepseek-chat"
npx tsx src/gateway/server.ts
```

### Example 2: Docker All-in-One (Hermes + Gateway)

```bash
docker run -d \
  -e MODEL_API_KEY="sk-xxx" \
  -e MODEL_BASE_URL="https://api.deepseek.com/v1" \
  -e MODEL_NAME="deepseek-chat" \
  -p 8420:8420 \
  -v hermes-data:/home/agentuser \
  agentmemory/hermes-memory:latest
```

### Example 3: Multiple Agents + shared Gateway

```bash
# 1. Start the Gateway (service mode)
cd MemoryCore
TDAI_DEPLOY_MODE=service \
TDAI_GATEWAY_PORT=3100 \
TDAI_GATEWAY_HOST=0.0.0.0 \
STATE_BACKEND=local \
VDB_ENDPOINT="http://vdb.example.com:8100" \
VDB_API_KEY="your-key" \
VDB_DATABASE="memory-shared" \
npx tsx src/gateway/server.ts

# 2. Configure a different service_id for each Hermes Agent
# Agent A:
export TDAI_MEMORY_ENDPOINT="http://gateway-host:3100"
export TDAI_MEMORY_API_KEY="shared-key"
export TDAI_MEMORY_SERVICE_ID="agent-code-assistant"

# Agent B:
export TDAI_MEMORY_ENDPOINT="http://gateway-host:3100"
export TDAI_MEMORY_API_KEY="shared-key"
export TDAI_MEMORY_SERVICE_ID="agent-customer-support"
```

### Example 4: K8s production deployment

This repository does not currently ship ready-to-apply Kubernetes manifests; use the Deployment, ConfigMap, and Secret examples in this section as configuration templates, and adjust them to your actual image, domain, and cluster environment.

---

## End-to-End Verification (E2E)

The repository provides two ready-to-use E2E scripts, covering the two deployment forms respectively. Both run the entire chain with a real Hermes API Server + a real LLM + a real Gateway process.

### Standalone E2E: `__tests__/e2e/test_hermes_standalone_e2e.py`

Verifies the open-source single-machine deployment chain:

```
Hermes API Server → memory_tencentdb (v1 plugin) → self-managed Gateway child process → SQLite + local FS
```

Coverage:
- Hermes API Server starts and `/health` passes
- The first chat triggers the v1 plugin's `initialize()`, which automatically launches the Node child process with `pnpm exec tsx src/gateway/server.ts`
- Gateway `/health` reports `vectorStore: true`
- 3 rounds of conversation: plant a marker → the model recalls and echoes it → cross-session recall via the v1 plugin's prefetch
- Side-channel: query the Gateway's `/search/conversations` directly and find this run's marker
- Tool layer: `/search/conversations` / `/search/memories` respond normally

```bash
hermes-agent/.venv/bin/python MemoryCore/__tests__/e2e/test_hermes_standalone_e2e.py
```

Measured result: **16 / 16 passed**.

### Service E2E: `MemoryCore/__tests__/e2e/test_hermes_service_e2e.py`

Verifies the cloud-service multi-replica deployment chain:

```
mock-shark (Shark stub: provides VDB/COS configuration)
2 Gateway processes (service mode, sharing TCVDB)
Hermes → memory_tencentdb_v2 (v2 plugin, tdai_memory SDK) → Gateway-1 → TCVDB
Side-channel verification on Gateway-2 → proves TCVDB is truly shared
```

Coverage:
- mock-shark + GW1 + GW2 + Hermes are all ready
- Both Gateways are in service mode (`stateBackend=connected` + `timerScanner` running)
- Hermes `/v1/models` returns 200 and the v2 plugin loads successfully
- 3 rounds of conversation written to GW1 → real TCVDB through the v2 plugin
- **Cross-Gateway consistency**: a GW2 search can find the marker written by GW1
- GW2 `/conversation/query` retrieves all messages of the main session
- Cross-session prefetch: in a new session, the model recalls the marker through the v2 plugin
- L1 add on GW1 → immediately visible in GW2 `/atomic/query` (proves TCVDB shared reads and writes)
- Automatically backs up/restores the `memory.provider` field of `~/.hermes/config.yaml`

```bash
# Prerequisite: install the SDK into the Hermes venv (one-time)
hermes-agent/.venv/bin/python -m pip install -e sdk/memory-core/python/

# Run
hermes-agent/.venv/bin/python __tests__/e2e/test_hermes_service_e2e.py
```

Measured result: **23 / 23 passed** (cross-Gateway consistency, cross-session recall, and L1 cross-GW sharing all pass).

### Common prerequisites for both scripts

1. The `hermes` CLI is installed (default path `~/.hermes/bin/hermes`)
2. `model.api_key` / `model.base_url` / `model.default` in `~/.hermes/config.yaml` are configured with a usable LLM
3. The v1 / v2 plugins are linked into `hermes-agent/plugins/memory/` (installed by default)
4. Service mode additionally requires: `pnpm add cos-nodejs-sdk-v5` (Gateway dependency) + `pip install -e sdk/memory-core/python/` (SDK used by Hermes)