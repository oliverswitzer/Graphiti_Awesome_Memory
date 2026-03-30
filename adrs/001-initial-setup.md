# ADR 001: Initial Graphiti Awesome Memory Setup

**Date:** 2026-03-30

## Context

Oddly Good runs a self-hosted AI automation stack split across two machines:

- **Raspberry Pi 5** — orchestration via Docker Compose (n8n, PostgreSQL)
- **Windows PC (olivers-pc)** — GPU/compute host (Ollama, ComfyUI, now Graphiti)

We needed a persistent, semantic memory backend so that n8n AI agent workflows on the Pi can maintain context across conversations. Graphiti (a temporal knowledge graph) was chosen and deployed on the PC, accessible to the Pi over the LAN.

## Decision

### Fork over pre-built image

The upstream repo (`GoGoButters/Graphiti_Awesome_Memory`) does not publish pre-built Docker images. We forked the repo to track our customizations while still being able to pull upstream updates.

### Port 8001

ComfyUI already occupies port 8000 on this machine. The adapter's external port was changed to **8001** (internal remains 8000) by hardcoding the port mapping in `docker-compose.yml`.

### OpenAI for LLM and embeddings

- **Main LLM:** `gpt-4.1` (complex reasoning, ~20% of operations)
- **Fast LLM:** `gpt-4.1-mini` (deduplication/validation, ~80% of operations)
- **Embeddings:** `text-embedding-3-small`

Anthropic was considered for LLM but the adapter only supports OpenAI-compatible APIs. Ollama was considered for embeddings but OpenAI was simpler (single API key for everything).

### Removed `apparmor:unconfined`

The upstream `docker-compose.yml` sets `security_opt: apparmor:unconfined` on all services. This is not supported on Windows/Docker Desktop and was removed.

### TypeScript build fix

The frontend build failed due to a TypeScript deprecation (`baseUrl` option). Fixed by adding `"ignoreDeprecations": "6.0"` to `frontend/tsconfig.json`.

### CORS origins

The adapter's `ALLOWED_ORIGINS` must include `http://localhost:3000` for local admin UI access. The `generate_envs.sh` script only writes the single `admin_frontend_url` from `config.yml`, so we manually edited `.env.adapter` to include multiple origins (localhost, mDNS hostname, LAN IP).

### Windows Firewall

Inbound TCP rules were added for ports 8001 (adapter), 3000 (frontend), and 7474 (Neo4j browser) so the Pi can reach these services.

## Services

| Service        | Port  | Purpose                                    |
|---------------|-------|--------------------------------------------|
| Adapter API    | 8001  | Memory API — what n8n talks to             |
| Frontend       | 3000  | Admin UI for browsing the knowledge graph  |
| Neo4j Browser  | 7474  | Graph database web UI                      |
| Neo4j Bolt     | 7687  | Graph database protocol (internal)         |
| Redis          | 6379  | Message broker (internal)                  |

## Setup steps

```bash
# 1. Copy and edit config
cp config.yml.example config.yml
# Edit config.yml with your API keys and passwords

# 2. Generate env files
bash scripts/generate_envs.sh

# 3. Manually fix ALLOWED_ORIGINS in .env.adapter if needed
# Add http://localhost:3000 and any other origins

# 4. Build and start
docker compose up -d --build

# 5. Verify
curl http://localhost:8001/docs
```
