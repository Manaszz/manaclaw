# ManaClaw Deployment Guide

This guide covers local development, container images, Docker Compose profiles, configuration, production hardening, Windows WSL2, and common troubleshooting.

---

## Prerequisites

- **Node.js 20+** on the host  
- **Docker** or **Podman** for workspace containers  
- **OpenCode** available inside the `manaclaw-agent` image (see container build)  
- For local models: **Ollama** (or another OpenAI-compatible endpoint)

---

## Quick Start (Local Development with SQLite and Ollama)

1. **Clone the repository** and install dependencies:

   ```bash
   npm install
   ```

2. **Copy configuration:**

   ```bash
   cp config.example.toml config.toml
   ```

3. **Edit `config.toml`:**
   - Set `[database] provider = "sqlite"` and confirm `[database.sqlite] path`.
   - Under `[provider]`, set `base_url = "http://localhost:11434"` and choose a pulled Ollama model (for example `qwen2.5:14b`).
   - Leave `[server] api_key` empty only on trusted localhost; set a random token for any shared network interface.

4. **Start Ollama** and pull a model:

   ```bash
   ollama pull qwen2.5:14b
   ```

5. **Build the agent image** (see [Docker setup](#docker-setup)) if a prebuilt image is not used.

6. **Run the host** (development):

   ```bash
   npm run build
   npm start
   ```

7. **Verify the HTTP server** (default port **8080**):

   ```bash
   curl -sS http://127.0.0.1:8080/api/v1/health
   ```

   If authentication is enabled, pass `Authorization: Bearer <token>` as documented in [API.md](./API.md).

8. **Send a test message** via the API (adjust `workspaceId` to a registered workspace):

   ```bash
   curl -sS -X POST "http://127.0.0.1:8080/api/v1/messages" \
     -H "Content-Type: application/json" \
     -d '{"workspaceId":"ws_dev","content":"Hello","sender":"cli"}'
   ```

---

## Docker Setup

### Building the `manaclaw-agent` image

From the repository root (after `container/Dockerfile` exists in your tree):

```bash
docker build -t manaclaw-agent:latest -f container/Dockerfile container/
```

Align the image name with `config.toml`:

```toml
[container]
image = "manaclaw-agent:latest"
runtime = "docker"
```

### Running the host

The host process can run **on the metal** (recommended for development) or in a **lightweight container** that mounts the Docker socket to spawn workspace containers. For production, run the host as a systemd service or container orchestration workload with access to the container runtime.

Ensure these paths are reachable from the host process:

- `store/` (database, auth)  
- `data/` (sessions, IPC, sanitized env)  
- `groups/` (workspace files)  
- `logs/`

---

## Docker Compose

Compose files may live at the repository root (for example `docker-compose.yml`) once added. Use **profiles** to opt in to dependencies.

### Profile: `core`

Runs **ManaClaw host** (when containerized) with **SQLite** volume mounts and default ports.

```yaml
# Illustrative fragment — adjust to match your published image and build
services:
  manaclaw:
    image: manaclaw:latest
    profiles: ["core"]
    volumes:
      - ./store:/app/store
      - ./data:/app/data
      - ./groups:/app/groups
      - ./logs:/app/logs
      - ./config.toml:/app/config.toml:ro
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "8080:8080"
```

### Profile: `postgres`

Adds **PostgreSQL** and sets:

```toml
[database]
provider = "postgres"
```

Use secrets or env substitution for `MANACLAW_DB_PASSWORD` rather than committing passwords.

### Profile: `ollama`

Runs **Ollama** as a sidecar and points `[provider].base_url` to `http://ollama:11434` on the Compose network.

### Profile: `browser`

Optional **browser automation** dependencies (Chromium, display-less profile) when your agent image or MCP tools require them. Mount volumes and capabilities according to the browser MCP documentation you enable.

**Typical combined invocation:**

```bash
docker compose --profile core --profile postgres --profile ollama up -d
```

---

## Configuration

### `config.toml`

Primary source of truth for assistant name, triggers, polling intervals, server bind, database provider, container image, provider defaults, memory mode, and security lists. See `config.example.toml` in the repository root.

### Environment variables

Use environment variables for secrets and environment-specific overrides (exact names follow implementation; common patterns include):

| Variable | Purpose |
|----------|---------|
| `MANACLAW_DB_PASSWORD` | PostgreSQL password when not stored in `config.toml` |
| `MANACLAW_API_KEY` | Override or supplement HTTP Bearer token |
| Provider keys | OpenAI, OpenRouter, or other LLM credentials consumed by OpenCode inside the container |

### Secrets

- Never commit real `config.toml` with production secrets.  
- Prefer a secret manager or `.env` excluded from version control, with only sanctioned keys copied into `data/env/env` for the container.  
- Rotate API keys if logs or workspaces may have exposed them.

---

## Production Recommendations

| Area | Recommendation |
|------|----------------|
| **HTTPS** | Terminate TLS at a reverse proxy (nginx, Traefik, Caddy) or load balancer; do not expose plain HTTP publicly. |
| **Authentication** | Set a strong `server.api_key`; integrate upstream SSO or API gateway for human access if needed. |
| **Network** | Bind `server.host` to loopback if a reverse proxy handles external traffic; restrict security groups/firewall rules. |
| **Resource limits** | Set CPU/memory limits on workspace containers; tune `max_concurrent_containers` and timeouts to protect the host. |
| **Backups** | Schedule backups for PostgreSQL or SQLite files, plus `data/sessions/` and critical `groups/` content. |
| **Observability** | Centralize `logs/manaclaw.log`; monitor health endpoint and container failure rates. |
| **Updates** | Pin the `manaclaw-agent` image by digest or version tag; document rollback. |

---

## Windows WSL2 Notes

- Install **Docker Desktop** with WSL2 backend, or run Docker Engine inside your preferred WSL distribution.  
- Keep the repository on the **Linux filesystem** (`\\wsl$\...`) for acceptable I/O with SQLite and volume mounts.  
- Ensure `config.toml` paths and mount allowlists use paths valid **inside WSL** when the host runs there.  
- From Windows PowerShell, invoke `wsl -d <Distro>` to run CLI and `curl` against `localhost` if the server listens inside WSL.  
- If using Ollama on Windows vs WSL, align `[provider].base_url` with where Ollama actually listens (`http://localhost:11434` vs WSL IP).

---

## Troubleshooting Common Issues

| Symptom | Likely cause | What to check |
|---------|----------------|---------------|
| HTTP 401 on API | Auth enabled | Send `Authorization: Bearer` matching `server.api_key` |
| Connection refused on 8080 | Host not running or wrong bind | Process status, `[server] host` / `port`, firewall |
| Container spawn failures | Docker not running or socket permissions | `docker ps`, user in `docker` group, WSL Docker integration |
| OpenCode errors in logs | Model URL wrong or model missing | Ollama `ollama list`, `[provider]` base URL and model id |
| Workspace not responding | Trigger pattern, registration, queue | Registered workspace, trigger prefix, scheduler and poll logs |
| SQLite locked | Multiple processes on one file | Single writer host; avoid NFS locks for SQLite in prod |
| PostgreSQL auth errors | Wrong credentials or host | Env vars, network from host/container to DB |

For a structured checklist, see [DEBUG_CHECKLIST.md](./DEBUG_CHECKLIST.md).
