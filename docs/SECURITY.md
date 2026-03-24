# ManaClaw Security Model

## Trust Model

| Entity | Trust Level | Rationale |
|--------|-------------|-----------|
| Main workspace | Trusted | Private or admin-controlled context; elevated IPC and mount privileges by design |
| Non-main workspaces | Untrusted | Other participants or integrations may be malicious or compromised |
| Container agents | Sandboxed | Execution confined to an isolated runtime with explicit mounts and policies |
| API requests (HTTP webhook) | Conditionally trusted | Only after successful Bearer token validation; treat as automation or operator-controlled input |
| Channel messages | User input | Same class of risk as any chat-originated content; susceptible to prompt injection and social engineering |

## Security Boundaries

### 1. Container Isolation (Primary Boundary)

Agents execute in containers (e.g., Docker-based Linux environments), providing:

- **Process isolation** — Container processes cannot directly affect the host OS or other workspaces’ processes.
- **Filesystem isolation** — Only explicitly approved mount points are visible inside the container.
- **Non-root execution** — Workloads run as an unprivileged user (e.g., uid 1000), reducing impact of container breakout attempts.
- **Ephemeral containers** — Fresh environment per invocation where applicable (e.g., `--rm` or equivalent lifecycle), limiting persistence of attacker-controlled state.

This is the **primary** security boundary. Application-level checks complement isolation; they do not replace it. The effective attack surface is bounded by what is mounted, what commands and egress are permitted, and what credentials or configuration are ever present inside the container.

### 2. Mount Security

**External allowlist** — Mount permissions are stored outside the project tree (default path pattern: `~/.config/manaclaw/mount-allowlist.json`), and are:

- Not bind-mounted into agent containers
- Not writable by untrusted workspace agents
- Evaluated by the host before container creation

**Default blocked patterns** (representative; align with deployment policy):

```
.ssh, .gnupg, .aws, .azure, .gcloud, .kube, .docker,
credentials, .env, .netrc, .npmrc, id_rsa, id_ed25519,
private_key, .secret
```

**Protections:**

- Resolve symlinks before validation to reduce traversal and confused-deputy issues.
- Reject unsafe container paths (e.g., `..`, unexpected absolute paths) at validation time.
- Options such as read-only mounts for non-main workspaces limit write access to sensitive host paths.

**Read-only project root (main workspace):**

The main workspace’s project root is mounted read-only where supported. Writable paths required for agent work (e.g., workspace data at `/workspace/current`, session or tool directories under `/home/node/.opencode/`, IPC artifacts) are mounted separately. This reduces the risk that an agent modifies host application code (`src/`, build outputs, `package.json`, etc.) in a way that survives restart and weakens future sandboxes.

**PolicyEngine integration:**

Mount requests are validated by the **PolicyEngine** in addition to static allowlists, so policy can encode workspace-specific rules, path classes, and denylists beyond pattern matching alone.

### 3. Session Isolation

Each workspace has isolated agent session state (e.g., under `data/sessions/{workspace}/` or equivalent):

- Workspaces cannot read another workspace’s conversation history or session artifacts by design.
- Session data may include full transcripts and contents of files the agent accessed.
- Isolation limits cross-workspace information disclosure and lateral narrative injection across tenants.

### 4. IPC Authorization

Host-side routing and task operations are authorized against **workspace identity** (main vs non-main), analogous to NanoClaw’s main vs non-main group model:

| Operation | Main workspace | Non-main workspace |
|-----------|----------------|-------------------|
| Send message to own bound channel / chat | Allowed | Allowed |
| Send message to other workspaces’ channels | Allowed | Denied |
| Schedule task for self | Allowed | Allowed |
| Schedule tasks on behalf of other workspaces | Allowed | Denied |
| View all scheduled tasks | Allowed | Own workspace only |
| Register or manage other workspaces | Allowed | Denied |

Exact behavior must match the shipped IPC and router implementation; this table describes the intended trust split.

### 5. Credential Isolation (Credential Manager)

Real API credentials and long-lived secrets **must not** be placed inside containers in plaintext forms that agents can read.

ManaClaw uses a **credential manager** that is **provider-agnostic**: the host resolves provider configuration and secrets, then injects what the OpenCode runtime needs through controlled channels:

- **Generated or merged `opencode.json`** (or equivalent runtime config) produced per workspace invocation, containing only what policy allows (e.g., base URLs, non-secret identifiers, references compatible with the runtime).
- **Environment variables** set on the container by the host, optionally via an env “proxy” pattern (placeholders or indirect references) so that agents and user-visible dumps do not expose raw keys.

**Principles:**

- Agents should not be able to recover full provider API keys from the filesystem, environment listings, or process arguments inside the container unless explicitly required by a documented, high-risk deployment mode.
- Sensitive host paths (sessions for messaging providers, allowlists, host-only stores) remain **unmounted** from containers.

**NOT mounted (non-exhaustive):**

- Messaging or channel session stores (e.g., authentication material for WhatsApp, Slack, or similar) — host only.
- Mount allowlist and policy configuration — host only unless a dedicated audit workflow requires otherwise.
- Paths matching blocked credential patterns.
- Shadow or replace risky files (e.g., `.env` in project root) with safe stubs where the architecture requires it.

### 6. API Authentication (HTTP Webhook)

HTTP endpoints that trigger work or expose workspace data **require authentication** using a **Bearer token** (or deployment-equivalent secret) validated by the host before any workspace routing.

**Expectations:**

- Tokens are long, random, and stored only on the host or in secure secret stores — not in container images or workspace-scoped agent folders.
- Use TLS in production; treat plaintext HTTP as trusted-network-only.
- Rotate tokens on compromise or staff change; scope automation credentials per integration where possible.

Unauthenticated or wrongly authenticated requests must fail closed (no partial side effects on workspaces). If the deployment leaves the API key unset, some configurations may disable HTTP auth entirely — that mode is appropriate only on trusted localhost or isolated networks.

### 7. PolicyEngine

The **PolicyEngine** enforces defense-in-depth **before** and **during** agent operations:

- **Command policy** — Blocklist or allowlist shell and tool invocations; reject dangerous patterns (e.g., recursive deletion of system paths, credential harvesting) according to deployment rules.
- **Mount validation** — Combine external allowlists with dynamic checks (workspace role, path semantics, read-only requirements).
- **Network egress** — Restrict outbound connections by host, port, or protocol where configured, so compromised agents cannot freely exfiltrate data or scan internal networks.

Policy decisions should be logged (subject to privacy and retention policy) for incident response.

### 8. OpenCode Permissions Inside the Container

OpenCode runtime permissions (example shape; additional keys such as `webfetch` may exist per deployment):

```json
{ "edit": "auto", "bash": "auto" }
```

**Meaning for security:** the agent may perform file edits and execute shell commands **within the sandbox** without per-step human approval. That increases **utility** and **risk** inside the container: unintended or attacker-steered actions can modify mounted writable paths or call allowed egress.

Mitigations rely on the boundaries above: minimal mounts, read-only project root for main workspace, PolicyEngine command and egress rules, no real secrets in-container, and ephemeral/non-root execution.

### 9. Identity Binding (Security Considerations)

Workspaces are bound to **channel identities** (e.g., chat JIDs, channel IDs, sender identifiers). Threats include:

- **Spoofed or confused sender metadata** if upstream channels do not cryptographically bind sender identity.
- **Wrong workspace registration** if operators misconfigure triggers or folder bindings.
- **Cross-workspace confusion** if routing keys collide or normalization is inconsistent.

**Recommendations:** verify channel connector documentation for identity guarantees; use main workspace only for trusted contexts; audit `registered_workspaces` and trigger rules after configuration changes; prefer channels with strong account linking over opaque display names alone.

## Privilege Comparison

| Capability | Main workspace | Non-main workspace |
|------------|----------------|-------------------|
| Project root access | `/workspace/project` (read-only) where configured | Typically none |
| Workspace folder | `/workspace/current` (read-write) | `/workspace/current` (read-write), scoped to that workspace’s host directory |
| Shared global memory | Readable via configured mount (e.g., `/workspace/global` read-only) | Same, if enabled for that deployment |
| Additional mounts | Configurable; subject to PolicyEngine | Configurable; often read-only or stricter |
| Network access | Subject to egress policy | Subject to egress policy (may be tighter per policy) |
| MCP / external tools | As enabled by host policy | As enabled; may be restricted per workspace |
| IPC to other workspaces | Yes | No |

Network and tool rows depend on `PolicyEngine` and deployment profiles; default posture should be least privilege for non-main workspaces.

## Security Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         UNTRUSTED INPUT ZONES                            │
│  Channel messages (prompt injection, abuse)    API clients (Bearer token)│
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼  Token verify, routing, normalization
┌─────────────────────────────────────────────────────────────────────────┐
│                      HOST PROCESS (TRUSTED CORE)                         │
│  • Message / webhook router (workspace binding)                         │
│  • IPC authorization (main vs non-main)                                 │
│  • Mount allowlist + PolicyEngine (mounts / commands / egress)          │
│  • Credential manager → opencode.json + env (no raw secrets in agent FS) │
│  • Container lifecycle (create, limit, destroy)                           │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼  Explicit mounts + env + generated config only
┌─────────────────────────────────────────────────────────────────────────┐
│                   CONTAINER (ISOLATED / SANDBOXED)                         │
│  • OpenCode runtime (edit:auto, bash:auto — in-sandbox automation)      │
│  • File and shell actions limited to mounts + policy                      │
│  • Egress filtered by PolicyEngine when enabled                          │
│  • No host messaging sessions; no mount allowlist; no real key material  │
│    in recoverable form (by design)                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

## Recommendations

1. **Treat non-main workspaces as hostile** — Use the strictest practical mount, command, and egress policies; avoid mounting repository-wide secrets.
2. **Protect the HTTP Bearer secret** — Store in a secret manager or host env; never commit to git; rotate on leak.
3. **Keep credential resolution on the host** — Prefer generated `opencode.json` and injected env over copying `.env` into the container.
4. **Enable and tune PolicyEngine** — Start restrictive; document exceptions; review logs for denied attempts.
5. **TLS and network placement** — Expose webhooks only behind HTTPS and, where possible, private network or allowlisted sources.
6. **Audit identity binding** — After any connector or workspace registration change, confirm triggers map to the intended workspace only.
7. **Update containers and base images** — Apply security patches to the agent image and host Docker stack regularly.
8. **Plan for multi-user RBAC (roadmap)** — Future releases may add role-based access for operators (e.g., workspace admin vs read-only auditor). Until then, assume a single trusted operator model on the host; restrict OS-level access to the ManaClaw process and config directories accordingly.

---

*This document describes the intended security model for ManaClaw. Implementation details may evolve; verify behavior against the version you deploy.*
