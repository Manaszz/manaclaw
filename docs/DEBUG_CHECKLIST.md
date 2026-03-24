# ManaClaw Debug Checklist

Structured checks for operators when ManaClaw misbehaves: HTTP API, workspace routing, containers, OpenCode, memory, and platform-specific service control.

---

## Quick Status Check

Run these from the **project root** (adjust paths and service names to your install).

### 1. Is the host process running?

**Linux / macOS (foreground or systemd):**

```bash
pgrep -af "node.*manaclaw|dist/index.js" || true
```

**Windows PowerShell:**

```powershell
Get-Process node -ErrorAction SilentlyContinue | Where-Object { $_.Path -like "*manaclaw*" }
```

If you use a process manager, check its status (systemd, PM2, Windows Service).

### 2. Is the HTTP server listening?

```bash
curl -sS -o /dev/null -w "%{http_code}" http://127.0.0.1:8080/api/v1/health
```

If auth is enabled:

```bash
curl -sS -o /dev/null -w "%{http_code}" -H "Authorization: Bearer YOUR_TOKEN" http://127.0.0.1:8080/api/v1/health
```

### 3. Are workspace containers running?

```bash
docker ps --format '{{.Names}} {{.Status}}' | grep -i manaclaw || true
```

Podman:

```bash
podman ps --format '{{.Names}} {{.Status}}' | grep -i manaclaw || true
```

### 4. Any stopped or orphaned containers?

```bash
docker ps -a --format '{{.Names}} {{.Status}}' | grep -i manaclaw || true
```

### 5. Database connectivity

**SQLite (file exists and readable):**

```bash
ls -la store/manaclaw.db
```

**Quick SQL probe:**

```bash
sqlite3 store/manaclaw.db "SELECT COUNT(*) FROM sqlite_master WHERE type='table';"
```

**PostgreSQL (example):**

```bash
psql "postgresql://manaclaw@localhost:5432/manaclaw" -c '\dt'
```

### 6. OpenCode inside the image

Confirm the agent image matches `config.toml` `[container] image` and that the entrypoint runs OpenCode (inspect image or run a one-off):

```bash
docker run --rm manaclaw-agent:latest opencode --version
```

(Exact CLI flag depends on OpenCode packaging in your image.)

### 7. Recent host errors

```bash
grep -E 'ERROR|WARN' logs/manaclaw.log 2>/dev/null | tail -30
```

---

## Workspace Not Responding

```bash
# Triggers and routing: confirm assistant name / trigger in config.toml
grep -E 'assistant_name|trigger_pattern' config.toml

# Are messages being persisted?
sqlite3 store/manaclaw.db "SELECT chat_jid, COUNT(*) FROM messages GROUP BY chat_jid ORDER BY COUNT(*) DESC LIMIT 10;" 2>/dev/null

# Poll / processing logs (wording may vary by version)
grep -E 'poll|router|workspace|queued|Processing' logs/manaclaw.log | tail -30

# Concurrency / queue
grep -E 'concurrent|queue|WorkspaceQueue' logs/manaclaw.log | tail -20
```

**Checklist:**

- Workspace is **registered** and the inbound channel maps to it.  
- Message matches **trigger** at line start (if using trigger semantics).  
- **Sender allowlist** (if enabled) includes the sender.  
- **max_concurrent_containers** not saturated; check for stuck containers.

---

## Container Issues

```bash
# Timeouts and kills
grep -E 'timeout|SIGKILL|137|Container' logs/manaclaw.log | tail -20

# Workspace container logs (path pattern)
ls -lt groups/*/logs/ 2>/dev/null | head -15

# Inspect last log file (replace path)
tail -100 groups/<workspace-folder>/logs/container-<timestamp>.log
```

**Mount validation:**

```bash
grep -E 'mount|Mount|REJECTED|allowlist' logs/manaclaw.log | tail -20
cat ~/.config/manaclaw/mount-allowlist.json 2>/dev/null
```

**Dry run listing inside container (example):**

```bash
docker run --rm --entrypoint sh manaclaw-agent:latest -c "ls -la /workspace"
```

---

## OpenCode Runtime Errors

```bash
# Provider / model errors
grep -E 'opencode|model|provider|401|429|500' logs/manaclaw.error.log | tail -30

# Ollama reachability (if used)
curl -sS http://127.0.0.1:11434/api/tags | head
```

**Session files:**

```bash
find data/sessions -type f 2>/dev/null | head -30
```

If resume fails after upgrade, compare **DB-mirrored session id** with on-disk OpenCode session state and consider a documented reset procedure for that workspace.

---

## Memory Service Issues

```bash
grep -E 'memory|MemoryService|embedding|extraction' logs/manaclaw.log | tail -30
```

**Config:**

```bash
grep -A5 '^\[memory\]' config.toml
```

**Modes:**

- **off** — expect no retrieval from MemoryService; only file-based `MEMORY.md` if used.  
- **lite** — DB-heavy; check SQLite size and WAL.  
- **full** — embedding provider errors; verify `[memory]` embedding settings and API keys inside container policy.

---

## API and Webhook Issues

```bash
# 401 / auth
grep -E '401|Unauthorized|api_key|Bearer' logs/manaclaw.log | tail -20

# Incoming HTTP errors
grep -E 'fastify|validation|400|422' logs/manaclaw.log | tail -20
```

**Manual request:**

```bash
curl -v -X POST "http://127.0.0.1:8080/api/v1/messages" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{"workspaceId":"ws_test","content":"ping","sender":"debug"}'
```

Cross-check paths and payloads with [API.md](./API.md).

---

## Service Management Commands

### Linux (systemd example)

Replace unit name with your installed service.

```bash
sudo systemctl status manaclaw
sudo journalctl -u manaclaw -f
sudo systemctl restart manaclaw
```

### macOS (launchd example)

If you use LaunchAgents:

```bash
launchctl list | grep -i manaclaw
tail -f logs/manaclaw.log
```

### Windows

- **Services.msc** — if registered as a Windows Service.  
- **PowerShell** — start/stop the process or NSSM-managed service.

```powershell
Stop-Process -Name node -Force -ErrorAction SilentlyContinue
# Then restart from your install directory:
# npm start
```

### WSL2

Run checks **inside the same WSL distro** that hosts Node and Docker. Use `localhost` for services bound in that distro; from Windows, use the WSL IP only if the server binds to `0.0.0.0`.

---

## After Changing Code or Config

```bash
npm run build
# restart host process or container
```

For Docker Compose deployments:

```bash
docker compose build --no-cache manaclaw
docker compose up -d manaclaw
```

---

## Related Documentation

- [SPEC.md](./SPEC.md) — architecture and message flow  
- [DEPLOYMENT.md](./DEPLOYMENT.md) — Compose profiles and production  
- [API.md](./API.md) — HTTP contract  
