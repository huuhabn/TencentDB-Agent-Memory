# TencentDB-Agent-Memory

A long-term memory service for AI Agents, providing four-layer progressive memory (L0 conversation → L1 atomic memory → L2 scene consolidation → L3 user profile) for any Agent framework.

## Image Information

| Item | Value |
|------|---|
| Image name | `tencentdb-agent-memory` |
| Base image | `node:22-slim` |
| Size | ~920MB |
| Port | 8420 |
| Run user | tdai (uid 10001) |
| PID 1 | tini |

## Quick Start

The commands below are run inside the `MemoryCore/` directory by default; if you are at the repository root, run `cd MemoryCore` first.

### 1. Build the image

```bash
docker build -t tencentdb-agent-memory:latest .
```

### 2. Prepare the configuration file

The project provides two configuration templates:

| Template | Suitable scenarios |
|------|---------|
| `tdai-gateway.standalone.yaml` | Local development, single-machine deployment, zero external dependencies |
| `tdai-gateway.service.yaml` | K8s multi-replica, multi-tenant cloud service |

Copy a template and modify it:

```bash
# Standalone mode
cp tdai-gateway.standalone.yaml tdai-gateway.yaml

# Service mode
cp tdai-gateway.service.yaml tdai-gateway.yaml
```

### 3. Start the container

**Standalone mode (simplest):**

```bash
docker run -d --name agent-memory \
  -v $(pwd)/tdai-gateway.yaml:/data/config/tdai-gateway.yaml:ro \
  -e TDAI_LLM_API_KEY=sk-your-key \
  -p 8420:8420 \
  tencentdb-agent-memory:latest
```

**Service mode (requires Redis):**

```bash
# Start Redis (if you do not have a remote Redis)
docker run -d --name redis -p 6379:6379 redis:7-alpine

# Start mock-shark (provides VDB/COS credentials locally)
VDB_ENDPOINT=http://your-vdb:8100 \
VDB_API_KEY=xxx \
VDB_DATABASE=your-db \
COS_BUCKET=your-bucket \
COS_REGION=ap-guangzhou \
COS_SECRET_ID=xxx \
COS_SECRET_KEY=xxx \
npx tsx scripts/mock-shark-server.ts &

# Start Memory Service
docker run -d --name agent-memory \
  -v $(pwd)/tdai-gateway.real.yaml:/data/config/tdai-gateway.yaml:ro \
  -e TDAI_LLM_API_KEY=sk-your-key \
  -p 8420:8420 \
  tencentdb-agent-memory:latest
```

**Docker Compose one-command start (includes Redis):**

```bash
TDAI_LLM_API_KEY=sk-your-key docker compose -f docker-compose.local.yaml up --build
```

### 4. Verify the service

```bash
curl http://localhost:8420/health
```

A normal response:

```json
{
  "status": "ok",
  "version": "0.1.0",
  "services": {
    "timerScanner": { "isLeader": true },
    "pipelineWorker": { "workerId": "worker-xxx" },
    "stateBackend": "connected"
  }
}
```

## Configuration

### Config file + environment variables (recommended)

All settings support both a **YAML config file** and **environment variables**; environment variables take higher priority.

The config file path inside the container is specified by the `TDAI_GATEWAY_CONFIG` environment variable, defaulting to `/data/config/tdai-gateway.yaml`.

```
┌─────────────────────────────────┐
│  Environment variables (highest)│  ← Secret: sensitive credentials
├─────────────────────────────────┤
│  tdai-gateway.yaml config file  │  ← Mounted via ConfigMap
├─────────────────────────────────┤
│  Code defaults                  │  ← Fallback
└─────────────────────────────────┘
```

### Config file structure

```yaml
deployMode: service          # standalone | service

server:
  port: 8420
  host: "0.0.0.0"

llm:                         # LLM API (OpenAI-compatible)
  baseUrl: "https://api.lkeap.cloud.tencent.com/v1"
  apiKey: "${TDAI_LLM_API_KEY}"
  model: "deepseek-v3.2"

redis:                       # Redis (required in service mode)
  host: "redis:6379"
  keyPrefix: "tdai_memory"

shark:                       # Shark config center (issues VDB/COS credentials)
  baseUrl: "http://shark:8000"

scanner:                     # Timer Scanner
  intervalMs: 500

worker:                      # Pipeline Worker
  pollMs: 200

memory:                      # Memory engine tuning
  pipeline:
    everyNConversations: 5
    enableWarmup: true
  recall:
    maxResults: 5
    strategy: "hybrid"
```

For the complete configuration reference, see `tdai-gateway.standalone.yaml` and `tdai-gateway.service.yaml`.

### Environment variable ↔ config file mapping

| Environment variable | YAML path | Default | Description |
|---------|----------|--------|------|
| `TDAI_DEPLOY_MODE` | `deployMode` | `standalone` | Deployment mode |
| `TDAI_GATEWAY_CONFIG` | — | `/data/config/tdai-gateway.yaml` | Config file path |
| `TDAI_LLM_API_KEY` | `llm.apiKey` | — | LLM API key |
| `TDAI_LLM_BASE_URL` | `llm.baseUrl` | `https://api.openai.com/v1` | LLM address |
| `TDAI_LLM_MODEL` | `llm.model` | `gpt-4o` | Model name |
| `REDIS_HOST` | `redis.host` | `127.0.0.1` | Redis address |
| `REDIS_PORT` | `redis.port` | `6379` | Redis port |
| `REDIS_PASSWORD` | `redis.password` | — | Redis password |
| `REDIS_KEY_PREFIX` | `redis.keyPrefix` | `tdai_memory` | Key prefix |
| `SHARK_BASE_URL` | `shark.baseUrl` | — | Shark address |
| `STATE_BACKEND` | `stateBackend` | auto | `redis` / `local` |
| `SCANNER_INTERVAL_MS` | `scanner.intervalMs` | `500` | Scan interval |
| `WORKER_POLL_MS` | `worker.pollMs` | `200` | Worker polling interval |
| `COS_DOMAIN` | `cos.domain` | — | COS internal domain |

## K8s / TKE Deployment

This repository does not currently ship ready-to-apply Kubernetes manifests; the core configuration practices needed for K8s/TKE deployment are listed below:

1. **ConfigMap** mounts `tdai-gateway.yaml` to `/app/config/`
2. **Secret** injects `TDAI_LLM_API_KEY` + `REDIS_PASSWORD` through environment variables
3. **Deployment** sets `TDAI_GATEWAY_CONFIG=/data/config/tdai-gateway.yaml`

```yaml
# Key settings in the Deployment
env:
  - name: TDAI_GATEWAY_CONFIG
    value: /data/config/tdai-gateway.yaml
  - name: TDAI_LLM_API_KEY
    valueFrom:
      secretKeyRef:
        name: tdai-memory-secrets
        key: TDAI_LLM_API_KEY
volumeMounts:
  - name: config-volume
    mountPath: /app/config
    readOnly: true
volumes:
  - name: config-volume
    configMap:
      name: tdai-memory-config
```

## API Overview

| Method | Path | Description |
|------|------|------|
| GET | `/health` | Health check |
| POST | `/recall` | Memory recall |
| POST | `/capture` | Write conversation |
| POST | `/search/memories` | L1 memory search |
| POST | `/search/conversations` | L0 conversation search |
| POST | `/session/end` | End session |
| POST | `/v2/*` | v2 multi-tenant API (requires Bearer token) |

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                 TencentDB Agent Memory               │
│                                                      │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ Gateway  │  │ TimerScanner │  │ PipelineWorker│  │
│  │ HTTP API │  │ 500ms scan   │  │ competing     │  │
│  │          │  │              │  │ consumers     │  │
│  └────┬─────┘  └──────┬───────┘  └──────┬────────┘  │
│       │               │                 │            │
│  ┌────▼─────────────────────────────────▼────────┐  │
│  │          IStateBackend (Redis / Local)         │  │
│  └───────────────────────────────────────────────┘  │
│       │                                              │
│  ┌────▼───────────┐  ┌────────────┐  ┌───────────┐  │
│  │  TdaiCore      │  │ StorePool  │  │ COS       │  │
│  │  L0→L1→L2→L3   │  │ VDB conn.  │  │ object    │  │
│  │                │  │ pool       │  │ storage   │  │
│  └────────────────┘  └────────────┘  └───────────┘  │
└─────────────────────────────────────────────────────┘
         │                    │               │
    ┌────▼────┐         ┌────▼────┐     ┌────▼────┐
    │  LLM    │         │  TCVDB  │     │  COS    │
    │ API     │         │ vector  │     │ object  │
    │         │         │ store   │     │ storage │
    └─────────┘         └─────────┘     └─────────┘
```

## File Structure

```
.
├── MemoryCore/
│   ├── Dockerfile                       # Image build
│   ├── docker-compose.local.yaml        # One-command local test (includes Redis)
│   ├── tdai-gateway.standalone.yaml     # Standalone config template
│   ├── tdai-gateway.service.yaml        # Service config template
│   ├── tdai-gateway.real.yaml           # Local test config (connects to real services)
│   ├── scripts/mock-shark-server.ts     # Mock Shark (local development)
│   └── src/gateway/server.ts            # Service entry point
```

## License

Proprietary — Tencent Cloud