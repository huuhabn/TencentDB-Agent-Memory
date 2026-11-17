# TDAI Global Image Local Deployment

Local launch scripts for the global image trio: `memory-core` + `memory-hub` + `proxy`. Each can run independently, or all can be started with a single command.

## Components and Ports

| Component | Container | Image (public on Docker Hub) | Host port | Purpose |
|---|---|---|---|---|
| **memory-core** | `tdai-memory-core` | [`agentmemory/memory-core`](https://hub.docker.com/r/agentmemory/memory-core) | `8420` | Kernel gateway: memory read/write, authentication, skill/RAG data plane |
| **memory-hub**  | `tdai-memory-hub`  | [`agentmemory/memory-hub`](https://hub.docker.com/r/agentmemory/memory-hub)   | `8125` / `8424` | Combined image of the management panel (Panel) + knowledge service (Knowledge) |
| **proxy**       | `tdai-proxy`       | [`agentmemory/memory-proxy`](https://hub.docker.com/r/agentmemory/memory-proxy) | `8096` | LLM request forwarding proxy; API entry point for coding agents |

> All three images are published under the [`agentmemory`](https://hub.docker.com/u/agentmemory) namespace on Docker Hub,
> multi-arch (`linux/amd64` + `linux/arm64`), publicly pullable, no login required. To pin a version, change the tag in `.env` from
> `:latest` to a specific version, e.g. `:1.0.0-beta.1`.
>
> Tencent internal colleagues can also override to the internal private registry `mirrors.tencent.com/memory-team-control/` — see the
> commented-out alternative block in `.env.example`.

## Requirements

- macOS / Linux
- Docker (any of Docker Desktop / colima / OrbStack)
- `bash` 4+ (the bash 3.2 bundled with macOS also works)

## Quick Start

```bash
cd ukh/deploy/global-images

# One command: auto-copies .env → interactively fills in LLM settings → auto-validates connectivity → starts all three
./start-all.sh
```

`start-all.sh` is now **interactive**. When run, it will:

1. If `.env` does not exist, automatically copy it from `.env.example` (no manual `cp` needed)
2. Guide you through filling in two groups of LLM settings (**Enter = keep the current default**):
   - `memory group`: `BASE_URL` / `API_KEY` / `MODEL` (protocol defaults to `openai`)
   - `proxy group`: first asks "reuse the memory group configuration?"; if yes, this group is skipped
3. After you fill them in, **immediately check whether the LLM connection works**; if not, it prompts you to re-enter until it passes
4. **Write the values back to `.env`** for persistence (reused by default on the next start)
5. Once checks pass, start all three components in one go

> You can also skip the interaction and read `.env` directly: manually `cp .env.example .env` and fill in the LLM settings, then
> run `./start-all.sh` and just press Enter all the way through to confirm (the defaults are the values already in `.env`).

### MongoDB Storage Backend (experimental, optional)

The default storage is still **sqlite** (zero dependencies, data stored in a container volume). The MongoDB data plane is an **experimental feature**,
disabled by default, and not recommended as the default backend for production. When enabled, it uses L0/L1/profile/skill documents + native mongot
BM25 search, and metadata is also stored in Mongo by default:

```bash
./start-all-mongo.sh    # Same flow as start-all.sh; writes MEMORY_CORE_STORE_MODE=mongodb to .env
```

- The script writes `MEMORY_CORE_STORE_MODE=mongodb` to `.env`, so afterwards `./start-all.sh`
  will also stay on MongoDB and will not silently fall back to sqlite. To go back to sqlite: comment out that line or set it to
  `sqlite`, then run `./start-all.sh`;
- If `MONGODB_ENDPOINT` is not set, the script automatically starts a local `mongodb-atlas-local` container
  (mongod + mongot combined, **not** cloud Atlas; data persisted in `mongo-local-*` volumes,
  cleaned up together by `stop-all.sh --purge`);
- To use an external Mongo (cloud Atlas / a self-hosted replica set with mongot), just fill in
  `MONGODB_ENDPOINT` in `.env`;
- **Switching storage backends does not migrate existing data.** sqlite lives in the `MEMORY_CORE_VOLUME` volume, mongo
  lives in the `mongo-local-*` volumes (or an external instance); after switching, the original data stays in the original backend. The current version requires
  you to back up and migrate manually; an official migration tool will be provided in a later version. L2/L3 files are in the
  `MEMORY_CORE_VOLUME` volume in both modes.

### Dry-run Validation (optional)

`verify.sh` can still be used on its own; it only checks the environment and does not start containers:

```bash
./verify.sh              # Full check by default (including LLM connectivity pre-check)
./verify.sh --skip-llm   # Skip the LLM check (offline environments)
```

## LLM Connectivity Pre-check

By default, `verify.sh` pre-checks the connectivity of both LLM groups (turn off with `--skip-llm`):

- **OpenAI-compatible protocol**: `GET {base}/models`; only verifies the API key + URL, **consumes no tokens**
- **Anthropic protocol**: `POST {base}/v1/messages` with a minimal message of `max_tokens=1`, consuming ≤ 10 tokens
- The **memory group** and **proxy group** are verified independently; if both groups have identical configuration, the duplicate check is skipped automatically
- **When containers are already running**, it additionally runs a curl from inside the container to verify "container → LLM" network reachability (in some corporate proxy/DNS-isolated environments the host can reach the LLM but the container cannot)

Example failure:

```
[error] memory group API key invalid (HTTP 401): https://api.deepseek.com/v1/models
{"error":{"message":"Authentication Fails, Your api key: ****abcd is invalid",...}}
```

— A wrong API key, wrong URL, or wrong model name is caught before startup, instead of failing with a 401 only at wiki ingest / chat time.

After startup completes:

- Panel UI: <http://localhost:8125/>
- Knowledge API: <http://localhost:8424/v3/>
- Knowledge Swagger: <http://localhost:8424/docs>
- Memory Gateway: <http://localhost:8420/>
- Proxy: <http://localhost:8096/>

## Two Independent Parameter Groups

**This is the core design of the scripts** — the LLMs of the memory group and the proxy group are completely independent and can point to different providers / different models.

### Memory group (used by memory-core + memory-hub)

The kernel's memory embed/summarize and knowledge's wiki ingest / summarization use this group's configuration.

| Variable | Description | Example |
|---|---|---|
| `MEMORY_LLM_BASE_URL` | OpenAI-compatible base URL | `https://api.deepseek.com/v1` |
| `MEMORY_LLM_API_KEY` | API key for the endpoint above | `sk-xxxxxxxx` |
| `MEMORY_LLM_MODEL` | Model ID | `deepseek-chat` |
| `MEMORY_LLM_PROTOCOL` | `openai` or `anthropic`, defaults to `openai` | `openai` |

### Proxy group (used by proxy)

After the proxy receives a user request, it forwards it to this group's endpoint.

| Variable | Description | Example |
|---|---|---|
| `PROXY_UPSTREAM_URL` | Forwarding target base URL | `https://api.deepseek.com/v1` |
| `PROXY_UPSTREAM_API_KEY` | API key used for forwarding | `sk-xxxxxxxx` |
| `PROXY_UPSTREAM_MODEL` | User-facing model ID | `deepseek-chat` |

> The two groups can use identical values (both pointing to the same LLM) or be completely different: for example, the memory group can use a cheap model for embedding while the proxy group uses a strong model for the main conversation.

When parameters are missing, the script **lists all missing items at once before startup** and runs `exit 1`, instead of failing halfway through.

## Memory Prompt Mode (chat / code)

memory-core switches the prompt family of the L1/L2/L3 pipeline via `MEMORY_PROMPT_MODE`:

| Mode | `.env` value | Extracted content | L3 output | Suitable scenarios |
|---|---|---|---|---|
| **code** (default) | `MEMORY_PROMPT_MODE=code` | Project facts / tasks / decisions / SOPs / taboos | Team Operating Doctrine | Coding agents, team collaboration, engineering projects |
| chat | `MEMORY_PROMPT_MODE=chat` | persona / episodic / instruction | persona.md (personal profile) | Personal assistants, small talk, teaching |

> **Note**: In `code` mode, pure small talk may yield 0 memories (the LLM sees no engineering content worth retaining). If L1 never produces output, first check whether `MEMORY_PROMPT_MODE` matches the actual conversation scenario.

## Internal Credentials (must-read for production)

The three components authenticate to each other using `MEMORY_CORE_GATEWAY_API_KEY`, and the first start also creates a `system_admin` account via
`init-admin`. For a **zero-config local experience**, the script defaults are:

| Variable | Default | Purpose |
|---|---|---|
| `MEMORY_CORE_GATEWAY_API_KEY` | `local` | Bearer for memory-hub / proxy → memory-core |
| `MEMORY_CORE_ADMIN_USERNAME` | `admin` | Username of the initialized system_admin |
| `MEMORY_CORE_ADMIN_USER_KEY` | `admin` | Login key of that admin user |
| `KNOWLEDGE_SERVICE_KEY` | **Auto-generated random value** | Panel ↔ Knowledge service-to-service Bearer (enforced on write/admin endpoints) |

> `KNOWLEDGE_SERVICE_KEY` has no fixed default: on first start the script automatically generates a random `ks-svc-*` string
> and writes it back to `.env` (reused on restart, so it does not drift), and the same value is injected into the memory-hub container twice —
> `KNOWLEDGE_SERVICE_KEY` (Knowledge validation side) + `KNOWLEDGE_AUTH_TOKEN` (Panel calling side).
> To distribute it yourself (multi-machine / external orchestration), just set it explicitly in `.env`; the script respects an existing value.

> These three defaults are only suitable for getting things running locally on a personal machine. **Before production / integration testing / public exposure, they must be replaced with long random strings**,
> otherwise anyone who can reach the port can obtain system_admin privileges.
>
> Uncomment the corresponding three lines in `.env` and override them (`_lib.sh` runs `require_vars`
> to validate the other required items, but since these three variables have fallback defaults, the script prints a `[warn]` at startup reminding you to change them).

## Using Each Component Independently

The three scripts can be run separately, which is handy for debugging or when you only need part of the functionality:

```bash
./start-memory-core.sh       # Run only the kernel gateway (8420)
./start-memory-hub.sh   # Run only panel + knowledge (8125 + 8424); requires the MEMORY_LLM_* parameters
./start-proxy.sh        # Run only the proxy (8096); requires the PROXY_UPSTREAM_* parameters
```

Dependencies:

- **memory-core**: no external dependencies; can be started on its own
- **memory-hub**: can start on its own (LLM_MODE=custom connects directly to the LLM), but the internal knowledge service will fail when it calls memory-core for RAG → starting memory-core first is recommended
- **proxy**: can start on its own (when cost-guard is unavailable it automatically degrades to passthrough and forwards directly), but auth / tdai memory / skill injection only work with memory-core

If any component is missing, the script prints a `warn` but does not block.

## Data Persistence

- `tdai-memory-core-data` (named volume) → memory-core's SQLite / memory data
- `tdai-panel-data` (named volume) → knowledge's SQLite / git clone / wiki files inside memory-hub

Data is retained until you run `docker volume rm`. To rename the volumes, change `MEMORY_CORE_VOLUME` / `PANEL_VOLUME` in `.env`.

## Stop / Clean Up

```bash
./stop-all.sh            # Stop containers, keep volumes (data is still there on next start)
./stop-all.sh --purge    # Stop containers + delete volumes + delete network (complete cleanup)
```

## Viewing Logs

```bash
docker logs -f tdai-memory-core
docker logs -f tdai-memory-hub
docker logs -f tdai-proxy
```

memory-hub runs two processes internally (panel + knowledge); their logs are inside the container at `/data/knowledge/logs/panel.log` and `.../knowledge.log` respectively.

## Port Conflicts

If `8125` / `8420` / `8424` / `8096` conflict with existing local services, just change them in `.env`:

```bash
MEMORY_CORE_PORT=18420
PANEL_PORT=18125
KNOWLEDGE_PORT=18424
PROXY_PORT=18096
# The externally reachable knowledge address must follow KNOWLEDGE_PORT
KNOWLEDGE_PUBLIC_BASE_URL=http://host.docker.internal:18424/v3
```

## Using the Proxy as the API Base for a Coding Agent

Taking Claude Code as an example:

```bash
export ANTHROPIC_BASE_URL=http://localhost:8096
export ANTHROPIC_API_KEY=any-string-if-auth-disabled
# Clients using the openai protocol are similar: OPENAI_BASE_URL=http://localhost:8096/v1
```

The Panel UI "client access address" card automatically combines the host's LAN IP + `PROXY_PORT` (for example
`http://192.168.1.100:8096/codebuddy/default`), so other people's machines can copy it and connect directly.
This is injected into `metadata-instances.json.proxy_endpoint` inside memory-hub via `MEMORY_HUB_PROXY_PUBLIC_URL`
(when unset, the script auto-detects using `hostname -I` / macOS `ipconfig getifaddr en0`,
and only falls back to `localhost` if detection fails).
Panel backend → Kernel forwarding is not affected by this variable (it always goes through `REMOTE_INSTANCE_URL` → memory-core:8420).
When the auto-detected address is wrong (multiple NICs / public domain / reverse proxy in front), explicitly set
`MEMORY_HUB_PROXY_PUBLIC_URL=http://<real-value>:8096` in `.env`. To make the UI card use the old behavior (fall back to
gateway_endpoint), explicitly set `MEMORY_HUB_PROXY_PUBLIC_URL` to an empty string.

By default, `proxy` disables `auth` / `sessionInit` / `costGuard` (these depend on internal services) and only does pure forwarding + `tdai-memory` context injection (the injector name, not the container name). To enable the full pipeline, additional configuration is required — see `context_proxy/config.example.yaml`.

## FAQ

**Q: `./start-all.sh` hangs at wait_healthy?**
The images may still be pulling. Pre-pull manually once with `docker pull <IMAGE>` and then run the script again.

**Q: memory-hub is up but the Panel won't open?**

Check whether `KNOWLEDGE_PUBLIC_BASE_URL` in `.env` includes `/v3` — without `/v3` the panel will report an error.

**Q: Proxy forwarding returns 401?**
`PROXY_UPSTREAM_API_KEY` is invalid or `PROXY_UPSTREAM_URL` does not match. Use `docker logs tdai-proxy` to see the error.

**Q: How do I access other services on the host (Ollama, Langfuse, etc.) from inside the containers?**
The scripts already add `--add-host=host.docker.internal:host-gateway` by default. Inside the container, use `http://host.docker.internal:<port>`.