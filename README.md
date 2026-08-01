# AI Cluster on Raspberry Pi 5 ARM64: llama.cpp RPC + Open WebUI + Hermes Agent

<p align="center">
  <img src="logo.jpeg" alt="Project logo"/>
</p>

This repository deploys a local AI stack with:

- `llama.cpp` distributed inference (`llama-server` head + two `ggml-rpc-server` workers)
- `Open WebUI` as the human-facing interface
- `Hermes Agent` as a configurable agent gateway connected to the same local model endpoint

Primary orchestration is Docker Compose.

Official documentation references used in this project:

- Docker Compose Specification: https://docs.docker.com/compose/compose-file/
- Docker security best practices: https://docs.docker.com/engine/security/
- Docker multi-platform images: https://docs.docker.com/build/building/multi-platform/
- llama.cpp repository and server docs: https://github.com/ggml-org/llama.cpp
- llama.cpp RPC docs: https://github.com/ggml-org/llama.cpp/blob/master/tools/rpc/README.md
- llama.cpp function calling docs: https://github.com/ggml-org/llama.cpp/blob/master/docs/function-calling.md
- Open WebUI documentation: https://docs.openwebui.com/
- Hermes Agent quickstart: https://hermes-agent.nousresearch.com/docs/getting-started/quickstart
- Hermes Agent providers docs: https://hermes-agent.nousresearch.com/docs/integrations/providers
- Hermes Agent Docker docs: https://hermes-agent.nousresearch.com/docs/user-guide/docker

## Review Findings (Code Review)

Findings are ordered by severity and include what was changed.

1. High: Open WebUI was exposed on all host interfaces (`0.0.0.0:3000`), which increases attack surface.
   - Change applied: bind changed to loopback (`127.0.0.1:3000:8080`) in `docker-compose.yml`.
   - Rationale: least exposure by default, consistent with secure-by-default deployment.
   - Reference: https://docs.docker.com/engine/security/

2. High: Hermes dashboard and API were not present, so there was no supported way to run Hermes Agent against the local model.
   - Change applied: added `hermes-bootstrap` and `hermes-agent` services in `docker-compose.yml`.
   - Rationale: official Hermes Docker model with persisted `/opt/data` volume and gateway mode.
   - Reference: https://hermes-agent.nousresearch.com/docs/user-guide/docker

3. Medium: `llama-server` context length was set to `8192`, while Hermes guidance requires at least 64K context for reliable tool workflows.
   - Change applied: set `-c ${LLAMA_CTX_SIZE:-65536}`.
   - Rationale: aligns model server context with Hermes operational requirement.
   - References:
     - https://hermes-agent.nousresearch.com/docs/getting-started/quickstart
     - https://hermes-agent.nousresearch.com/docs/integrations/providers

4. Medium: `llama-server` did not explicitly enable Jinja tool-call templating.
   - Change applied: added `--jinja` to head node command.
   - Rationale: recommended for tool-calling compatibility with llama.cpp server.
   - Reference: https://github.com/ggml-org/llama.cpp/blob/master/docs/function-calling.md

5. Medium: ARM64 targeting was implicit.
   - Change applied: added `platform: ${DOCKER_PLATFORM:-linux/arm64}` to all runtime services.
   - Rationale: deterministic ARM64 deployment for Raspberry Pi 5.
   - Reference: https://docs.docker.com/build/building/multi-platform/

6. Low: Build stages installed packages without `--no-install-recommends`.
   - Change applied: added `--no-install-recommends` to builder package installs.
   - Rationale: smaller build surface and fewer unnecessary packages.
   - Reference: https://docs.docker.com/engine/security/

## What Was Added

### 1. New services in docker-compose.yml

- `hermes-bootstrap`
  - One-shot initializer that writes Hermes model config into `/opt/data/config.yaml`.
  - Sets Hermes to use local llama.cpp endpoint (`http://llama-server-head:8080/v1`) as a custom provider target.

- `hermes-agent`
  - Persistent gateway service (`gateway run`) using official image `nousresearch/hermes-agent:latest`.
  - Exposes:
    - `127.0.0.1:8642` for Hermes API server
    - `127.0.0.1:9119` for Hermes dashboard
  - Uses mandatory dashboard authentication environment variables.

### 2. New environment template

- Added `.env.example` with documented defaults for:
  - ARM64 platform selection
  - llama.cpp context and thread tuning
  - Hermes API key and dashboard auth
   - Hermes API server opt-in (`API_SERVER_ENABLED=false` by default)
   - Explicit gateway user policy variables (`GATEWAY_ALLOW_ALL_USERS`, `TELEGRAM_ALLOWED_USERS`)
  - Hugging Face token

## Raspberry Pi 5 ARM64 Adaptation

This stack is adapted for Raspberry Pi 5 by default:

1. Architecture pinning
   - `DOCKER_PLATFORM=linux/arm64` in `.env.example`.
   - Compose uses `platform: ${DOCKER_PLATFORM:-linux/arm64}`.
   - Reference: https://docs.docker.com/build/building/multi-platform/

2. CPU-aware defaults
   - `LLAMA_THREADS=4` default (safe baseline for Pi 5).
   - You can tune this to match your cooling and latency target.
   - Reference: https://github.com/ggml-org/llama.cpp

3. Context length for agent workloads
   - `LLAMA_CTX_SIZE=65536` to satisfy Hermes agent-context guidance.
   - If memory pressure is too high on your specific model, reduce with caution and expect degraded agent reliability.
   - References:
     - https://hermes-agent.nousresearch.com/docs/getting-started/quickstart
     - https://github.com/ggml-org/llama.cpp/tree/master/tools/server

## Security Baseline Applied

The Compose stack enforces the following controls:

- `cap_drop: [ALL]` on services
- `security_opt: [no-new-privileges:true]` on services
- Loopback-only host publication for sensitive surfaces by default:
  - llama-server: `127.0.0.1:8080`
  - Open WebUI: `127.0.0.1:3000`
  - Hermes API: `127.0.0.1:8642`
  - Hermes dashboard: `127.0.0.1:9119`

Additional required controls:

- Set strong secrets in `.env` before first run.
- Never expose Hermes dashboard/API publicly without authentication, TLS, and network controls.

References:

- https://docs.docker.com/engine/security/
- https://hermes-agent.nousresearch.com/docs/user-guide/docker
- https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard

## Deployment Steps (Detailed)

### Step 1: Create runtime environment file

Copy template and set real values.

```bash
cp .env.example .env
```

Update at least these keys in `.env`:

- `HF_TOKEN`
- `HERMES_DASHBOARD_BASIC_AUTH_USERNAME`
- `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD`
- `HERMES_DASHBOARD_BASIC_AUTH_SECRET`

If you need Hermes OpenAI-compatible API access, also set:

- `API_SERVER_ENABLED=true`
- `API_SERVER_KEY`

If you use messaging platforms, set explicit allowlists, for example:

- `TELEGRAM_ALLOWED_USERS=<your_telegram_numeric_user_id>`

Default note:

- `TELEGRAM_ALLOWED_USERS=0` is used as a secure sentinel value that grants access to nobody.
- This keeps deny-by-default behavior explicit and prevents startup warnings about missing allowlists.

Reference:

- https://hermes-agent.nousresearch.com/docs/user-guide/docker

Secret guidance:

- Use long random values for all secrets.
- Keep `.env` out of source control.

References:

- https://docs.docker.com/compose/how-tos/environment-variables/set-environment-variables/
- https://hermes-agent.nousresearch.com/docs/user-guide/docker

### Step 2: Build and start the stack

```bash
docker compose up -d --build
```

What this does:

- Builds ARM64 llama.cpp worker and head images.
- Downloads the GGUF model via `model-downloader`.
- Starts RPC workers and head server.
- Boots Hermes configuration one-shot service.
- Starts Open WebUI and Hermes gateway.

References:

- https://docs.docker.com/reference/cli/docker/compose/up/
- https://github.com/ggml-org/llama.cpp/blob/master/tools/rpc/README.md

### Step 3: Verify service health

```bash
docker compose ps
```

Check endpoints:

```bash
curl -f http://127.0.0.1:8080/health
curl -f http://127.0.0.1:3000/health
curl -f http://127.0.0.1:9119/
```

If `API_SERVER_ENABLED=true`, also verify:

```bash
curl -f http://127.0.0.1:8642/health
```

Important runtime note:

- Hermes persists its own settings in `/opt/data` inside the container (backed by the `hermes-data` volume).
- The `hermes-bootstrap` service writes secure baseline values into that persistent store on startup.
- If you changed API server policy in the past, recreate both Hermes services so persisted settings are refreshed.

References:

- https://docs.docker.com/reference/cli/docker/compose/ps/
- https://hermes-agent.nousresearch.com/docs/user-guide/docker

### Step 4: Confirm model wiring

Check llama.cpp model list:

```bash
curl -s http://127.0.0.1:8080/v1/models
```

Check Hermes logs:

```bash
docker compose logs -f hermes-agent
```

References:

- https://github.com/ggml-org/llama.cpp/tree/master/tools/server
- https://hermes-agent.nousresearch.com/docs/integrations/providers

### Step 5: Access applications

- Open WebUI: http://127.0.0.1:3000
- Hermes dashboard: http://127.0.0.1:9119

Use dashboard credentials from `.env`.

Reference:

- https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard

## Hermes Agent Linking Details

The linkage to your existing local model is done through `hermes-bootstrap`:

1. Waits for healthy `llama-server-head`.
2. Writes model provider config into Hermes data volume.
3. Sets:
   - provider: `custom`
   - base URL: `http://llama-server-head:8080/v1`
   - model name: `${HERMES_MODEL_ID}`
   - context length: `${HERMES_CONTEXT_LENGTH}`

This follows Hermes custom endpoint guidance.

Reference:

- https://hermes-agent.nousresearch.com/docs/integrations/providers

## Operational Commands

Start or rebuild:

```bash
docker compose up -d --build
```

View logs:

```bash
docker compose logs -f
```

Restart one service:

```bash
docker compose restart hermes-agent
```

Stop stack:

```bash
docker compose down
```

References:

- https://docs.docker.com/reference/cli/docker/compose/logs/
- https://docs.docker.com/reference/cli/docker/compose/restart/
- https://docs.docker.com/reference/cli/docker/compose/down/

## Notes and Constraints

- This repository currently pins a single model file (`gemma-3-1b-it-Q4_K_M.gguf`) in downloader and head server command.
- `HERMES_MODEL_ID` must match what your endpoint reports and what you want Hermes to request.
- If you expose any service beyond loopback, add a reverse proxy, TLS, and strict network ACLs.

## Validation Checklist

- `docker compose config` passes.
- `docker compose ps` shows healthy `llama-server-head`, `open-webui`, and `hermes-agent`.
- `curl` health checks succeed on local endpoints.
- Hermes dashboard requires authentication.
- Open WebUI can chat through llama.cpp endpoint.
