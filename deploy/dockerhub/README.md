# Docker Hub Image Publishing

Builds the image trio and pushes it to the
[`agentmemory`](https://hub.docker.com/u/agentmemory) namespace on Docker Hub.

`publish.sh` is self-contained: it only depends on each component's own Dockerfile,
`deploy/panel-knowledge-combined/build.sh`, and `MemoryPanel/scripts/secret-scan.sh`.

## Components and Image Names

| Component | Build context | Image |
|---|---|---|
| `memory-core` | `MemoryCore/` | `agentmemory/memory-core` |
| `memory-proxy` | `MemoryProxy/` (rsync'd to a temporary context) | `agentmemory/memory-proxy` |
| `memory-hub` | `MemoryPanel/` + `MemoryKnowledge/` combined | `agentmemory/memory-hub` |

## Prerequisites

```bash
docker login docker.io          # The account must have push permission for agentmemory
docker buildx version           # buildx is required (the script creates a builder automatically)
```

## Usage

```bash
cd deploy/dockerhub

# Publish all three at once
VERSION=1.0.0 ./publish.sh all

# A single component
VERSION=1.0.0 ./publish.sh memory-core
VERSION=1.0.0 ./publish.sh memory-proxy
VERSION=1.0.0 ./publish.sh memory-hub

# Dry run: only runs secret-scan and context preparation, no build, no push
DRY_RUN=1 VERSION=1.0.0 ./publish.sh all

# Build locally for a single architecture and spot-check the image contents, no push
PUSH=0 VERSION=1.0.0 ./publish.sh memory-core

# Also update :latest
ALSO_LATEST=1 VERSION=1.0.0 ./publish.sh all
```

`VERSION` is required and values starting with `dev-` are rejected, to avoid pushing development tags to the public registry.

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `VERSION` | none (required) | Image tag |
| `NAMESPACE` | `agentmemory` | Docker Hub namespace |
| `REGISTRY` | `docker.io` | Target registry |
| `PLATFORMS` | `linux/amd64,linux/arm64` | Multi-arch build targets |
| `ALSO_LATEST` | `0` | Whether to also push `:latest` |
| `PUSH` | `1` | Set to `0` for a local single-arch `--load` build with no push |
| `DRY_RUN` | `0` | Set to `1` to only run scanning and context preparation |
| `LOAD_PLATFORM` | `linux/amd64` | Architecture built locally when `PUSH=0` |
| `KEEP_CTX` | `0` | Set to `1` to reuse the temporary context from the previous run |
| `APT_MIRROR` | `deb.debian.org` | apt source used at build time; can be set to an accelerated mirror on internal networks |

## Build-time apt Acceleration

All four Dockerfiles control the apt source through the `APT_MIRROR` build-arg. The default is the official Debian source,
which works out of the box on public networks. When building on an internal network and you want faster builds, just pass a single variable;
the resulting images are not affected:

```bash
APT_MIRROR=<your-debian-mirror> VERSION=1.0.0 ./publish.sh all
```

## About Optional Private Modules

- `MemoryProxy/packages/cost-guard` is an optional extension and is not included in public images. `publish.sh` generates a stub package
  in the temporary context so the dependency graph can resolve; at runtime, when the dynamic import in `src/guard-adapter.ts`
  fails, it automatically degrades to passthrough forwarding.
- `MemoryCore/src/integrations` is handled the same way: it is excluded in `MemoryCore/.dockerignore`,
  and the runtime uses the fallback.

## Verification

```bash
docker pull agentmemory/memory-core:1.0.0
docker buildx imagetools inspect agentmemory/memory-core:1.0.0
```