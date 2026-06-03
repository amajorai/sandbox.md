---
name: sandbox
description: "Self-hosted cloud sandbox skill. Provisions an isolated code-execution environment on your own server — choose Daytona, Docker+Bun, or Firecracker as the backend. Supports ephemeral exec sandboxes, long-lived dev workspaces, or both. Integrates with Claude Code via MCP."
argument-hint: [--status | --reconfigure | --add-template <name>]
---

# Sandbox

You are setting up a self-hosted cloud sandbox environment. Interview first, execute in one clean pass.

**Args:** {{args}}


## Advisor - Getting Help When Stuck

When uncertain about your approach, stuck after 2 or more failed attempts, or about to commit to a significant decision, seek a second opinion before proceeding.

**Claude Code** - call `advisor()` with no arguments. Your full conversation history is forwarded to a stronger model automatically. Use it:
- Before committing to an approach you are not sure about
- After 2 or more failed attempts at the same problem
- Before declaring a complex task complete

To use a stronger advisor model, edit `~/.claude/settings.json` and set `"advisorModel"` to `"opus"`. Changes take effect after restarting the session.

**Codex CLI** (`CODEX=true` or `CODEX_SANDBOX` is set) - the `advisor()` tool is unavailable. When stuck, surface the uncertainty to the user explicitly and ask for direction rather than guessing.


## Privacy Rule - Redact Sensitive Values by Default

Never print the following in chat, even if you just ran a command that returned them:

- Server IP addresses or hostnames
- API keys or bearer tokens
- SSH private keys or fingerprints
- Passwords or passphrases
- Cloud provider credentials or account IDs
- Any value the user passed as a secret placeholder (e.g. `<YOUR_API_KEY>`)

When one of these values would naturally appear in your response, replace it with a placeholder and offer to share on request. Examples:

> Your sandbox environment is ready. *(Ask me for the API key if you need it.)*

> API key generated and written to /etc/sandboxmd/config.json. *(Not shown in chat — ask if you need it.)*

If the user explicitly asks, output the real value in that one response only. Do not repeat it in follow-up messages unless asked again.


## Argument Routing

Parse `{{args}}` first. Route to the correct path before doing anything else:

- **Empty or no args** - proceed to Phase 1 (full setup or status if already configured)
- **`--status`** - read config and show current state, then stop
- **`--reconfigure`** - skip to Phase 2 (re-interview and reconfigure)
- **`--add-template <name>`** - skip to Phase 3E (pull one additional template image)
- **Unknown flag** - tell the user: "Unknown flag. Valid flags: `--status`, `--reconfigure`, `--add-template <name>`." Then stop.

---

### --status path

```bash
cat /etc/sandboxmd/config.json 2>/dev/null || echo "NO_CONFIG"
```

If config exists, show:
- Backend
- Use case (ephemeral / long-lived / both)
- Server (vibe-managed or standalone or local)
- Templates available
- MCP integration status
- API port

Then stop. Do not continue to Phase 1.

If `NO_CONFIG` - tell the user sandbox is not yet set up, then proceed to Phase 1 for full setup.

---

### --add-template path

Ask: "Which template do you want to add?" with the standard image list, then skip to Phase 3E.


## Phase 1: Environment Detection

### Check for existing sandboxmd config (full setup path only)

```bash
cat /etc/sandboxmd/config.json 2>/dev/null || echo "NO_CONFIG"
```

**If config exists and args are empty** - this sandbox environment is already set up. Show current state and stop:

> **Sandbox is already configured.**
>
> **Current setup (`/etc/sandboxmd/config.json`):**
> - Backend: `[backend]`
> - Use case: `[use_case]`
> - Templates: `[templates]`
> - MCP: `[mcp_enabled]`
>
> **What would you like to do?**
> - **Check status** - run `/sandbox --status`
> - **Add templates** - run `/sandbox --add-template <name>`
> - **Reconfigure** - run `/sandbox --reconfigure`
> - **Start fresh** - re-run full setup on this machine (will overwrite config)

If the user chooses "Start fresh", continue below. Otherwise stop here.

### Check for vibe.md config

```bash
VIBE_RAW=$(cat /etc/vibemd/server.json 2>/dev/null || echo "NO_VIBE")
echo "$VIBE_RAW"
```

If vibe.md is configured, extract the platform:

```bash
VIBE_PLATFORM=$(echo "$VIBE_RAW" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('platform','unknown'))" 2>/dev/null || echo "unknown")
echo "VIBE_PLATFORM=$VIBE_PLATFORM"
```

Store as `VIBE_CONFIG` (raw JSON or "NO_VIBE") and `VIBE_PLATFORM`.

### Check Docker

```bash
DOCKER_VERSION=$(docker --version 2>/dev/null || echo "NO_DOCKER")
DOCKER_RUNNING=$(docker info 2>/dev/null | grep "Server Version" || echo "DOCKER_NOT_RUNNING")
echo "DOCKER_VERSION=$DOCKER_VERSION"
echo "DOCKER_RUNNING=$DOCKER_RUNNING"
```

### Check KVM (Firecracker requirement)

```bash
KVM_STATUS=$(ls /dev/kvm 2>/dev/null && echo "KVM_OK" || echo "NO_KVM")
echo "KVM_STATUS=$KVM_STATUS"
```

Store as `KVM_STATUS`.

### Detect environment

```bash
# Run via Bash tool on Windows
uname -a 2>/dev/null || echo "WINDOWS"
cat /etc/os-release 2>/dev/null || sw_vers 2>/dev/null || echo "OS_UNKNOWN"
echo "USER=$USER HOME=$HOME"
```

Determine:
- **Local dev machine** (macOS, Windows, desktop Linux with GUI)
- **Already on a VPS / headless server**

Tell the user: "I see you're on [local machine / a server]. Is that right?"

**Do not proceed until you have confirmed the environment.**


## Phase 2: Interview

Use `AskUserQuestion` for each question below - one call at a time, waiting for the answer before asking the next. Do not start any installation until all questions are answered. If the user changes a prior answer, restart from Q1.

---

### Q1 - Server

**If vibe.md is configured (`VIBE_CONFIG != "NO_VIBE"`):**

```
AskUserQuestion(
  question: "I found a vibe.md server running [VIBE_PLATFORM] on this machine. Where should the sandbox environment be deployed?",
  options: [
    "On this vibe.md server — deploy via [VIBE_PLATFORM] alongside existing apps (Recommended)",
    "On a new dedicated VPS — provision a separate server just for sandboxes",
    "Local only — run sandboxes on this machine using Docker (no cloud server needed)"
  ]
)
```

**If no vibe.md found:**

```
AskUserQuestion(
  question: "Where should the sandbox environment run?",
  options: [
    "New cloud VPS — I'll provision one from your chosen provider",
    "Existing server — I already have a server, give me the SSH details",
    "Local only — run sandboxes on this machine using Docker (no cloud server)"
  ]
)
```

Store as `SERVER_TARGET` ("vibe" | "new-vps" | "existing-server" | "local").

---

### Q2 - Cloud provider (ask only if SERVER_TARGET = "new-vps")

```
AskUserQuestion(
  question: "Which cloud provider? I'll fetch available regions and sizes after you authenticate.",
  options: [
    "Hetzner Cloud — cheapest, EU-based (~€4/mo for CX22). Best default.",
    "DigitalOcean — simple pricing, great UX, global regions",
    "OVH — good value, global PoPs",
    "AWS — most ecosystem, free tier (t3.micro)"
  ]
)
```

Store as `CLOUD_PROVIDER`. Skip if SERVER_TARGET != "new-vps".

---

### Q3 - Sandbox backend

Note: If `KVM_STATUS = "NO_KVM"`, tell the user "Firecracker requires KVM, which is not available on this machine. It is listed below but will require a bare-metal or KVM-enabled server." before presenting the question.

```
AskUserQuestion(
  question: "Which backend should power the sandboxes?",
  options: [
    "Docker + Bun API — lightweight REST service wrapping Docker. Matches the vibe.md stack. Best default.",
    "Daytona — full self-hosted platform. Long-lived dev workspaces with git, IDE, and port forwarding.",
    "Firecracker microVMs — highest isolation, E2B-style ephemeral execution. Requires KVM."
  ]
)
```

If the user picks Firecracker but KVM_STATUS = "NO_KVM" on the target machine, warn them and ask them to confirm before proceeding.

Store as `BACKEND` ("docker-bun" | "daytona" | "firecracker").

---

### Q4 - Use case

```
AskUserQuestion(
  question: "What is the primary use case?",
  options: [
    "Ephemeral exec — spawn a sandbox, run commands/code, destroy it. Ideal for Claude Code building and testing in the cloud.",
    "Long-lived workspaces — persistent dev environments per project, each with its own git clone and process state.",
    "Both — ephemeral for CI/build tasks, long-lived for active development."
  ]
)
```

Store as `USE_CASE` ("ephemeral" | "long-lived" | "both").

---

### Q5 - Sandbox templates

```
AskUserQuestion(
  question: "Which sandbox templates do you need? (You can add more later with /sandbox --add-template.)",
  multiSelect: true,
  options: [
    "Node.js (LTS) — node:lts-slim",
    "Python 3.12 — python:3.12-slim",
    "Go 1.23 — golang:1.23-alpine",
    "Rust (latest stable) — rust:slim",
    "Full Ubuntu — ubuntu:24.04 (largest, most flexible)"
  ]
)
```

Store as `TEMPLATES` (list of selected names).

---

### Q6 - Resource limits per sandbox

```
AskUserQuestion(
  question: "What resource limits per sandbox?",
  options: [
    "Small — 256 MB RAM, 0.5 CPU. Good for scripts and quick builds.",
    "Medium — 512 MB RAM, 1 CPU. Good for most Node/Python workloads. (Recommended)",
    "Large — 1 GB RAM, 2 CPU. Good for heavy builds (Rust, Go, Docker-in-Docker).",
    "Custom — I'll set limits manually."
  ]
)
```

Store as `RESOURCE_MEMORY` and `RESOURCE_CPUS`. If "Custom" was chosen, ask: "Enter memory limit (e.g. 512m) and CPU limit (e.g. 1.5) as two values."

Default mappings:
- Small - `RESOURCE_MEMORY=256m`, `RESOURCE_CPUS=0.5`
- Medium - `RESOURCE_MEMORY=512m`, `RESOURCE_CPUS=1`
- Large - `RESOURCE_MEMORY=1g`, `RESOURCE_CPUS=2`

---

### Q7 - Authentication

```
AskUserQuestion(
  question: "How should the sandbox API be secured?",
  options: [
    "API key — a single shared secret passed as a Bearer token. Simple and sufficient for most setups.",
    "No auth — only if running locally or on a private network with no public exposure.",
    "mTLS — mutual TLS certificate auth. Best for production environments with multiple clients."
  ]
)
```

Store as `AUTH_MODE` ("api-key" | "none" | "mtls").

---

### Q8 - MCP integration for Claude Code

```
AskUserQuestion(
  question: "Enable MCP integration? This lets Claude Code spawn and exec sandboxes directly as tool calls — no manual API calls needed. (Note: requires the sandbox API to be reachable from your local machine.)",
  options: [
    "Yes — configure the MCP HTTP endpoint in Claude Code settings (Recommended)",
    "No — I'll call the sandbox API manually or from my own code"
  ]
)
```

Store as `MCP_ENABLED` (true | false).

---

Once all questions are answered, proceed.

**Do not proceed until all questions are answered.**


## Phase 3A: Provision VPS (if SERVER_TARGET = "new-vps")

Skip this phase entirely if `SERVER_TARGET` is "vibe", "existing-server", or "local".

Install the cloud provider CLI and spin up a server. These commands run on your **local machine**. Follow the same provisioning steps as vibe.md:

```
If CLOUD_PROVIDER = Hetzner    → install hcloud CLI, authenticate, list regions, create server
If CLOUD_PROVIDER = DigitalOcean → install doctl CLI, authenticate, list regions, create droplet
If CLOUD_PROVIDER = OVH        → install OVH CLI or use web console
If CLOUD_PROVIDER = AWS        → install aws CLI, authenticate, launch EC2 instance (t3.small minimum)
```

Minimum recommended spec for sandboxes: 2 vCPU, 4 GB RAM, 40 GB SSD.

After provisioning, SSH into the server. Store the server target as `SANDBOX_HOST`.

**Do not proceed until you have shell access on the target server.**


## Phase 3B: Server Bootstrap

**If SERVER_TARGET = "local":** Skip apt-get and systemctl steps. Verify Docker Desktop is running:

```bash
docker --version || echo "Install Docker Desktop from https://docs.docker.com/get-docker/ then re-run /sandbox"
bun --version || curl -fsSL https://bun.sh/install | bash
```

If Docker is not installed or not running locally, tell the user to install Docker Desktop and re-run `/sandbox`. Do not proceed without Docker.

**If SERVER_TARGET = "vibe", "new-vps", or "existing-server":** Run on the target server:

```bash
# 1. Confirm OS
cat /etc/os-release

# 2. Update packages
apt-get update && apt-get upgrade -y

# 3. Install Docker (if not already installed)
if ! command -v docker >/dev/null 2>&1; then
  curl -fsSL https://get.docker.com | sh
  systemctl enable --now docker
fi

# 4. Install jq (needed for config and verification steps)
apt-get install -y jq

# 5. Install Bun (if not already installed)
if ! command -v bun >/dev/null 2>&1; then
  curl -fsSL https://bun.sh/install | bash
  export BUN_INSTALL="$HOME/.bun"
  export PATH="$BUN_INSTALL/bin:$PATH"
  ln -sf "$HOME/.bun/bin/bun" /usr/local/bin/bun
fi

# 6. Verify
docker --version
bun --version
```

**If on a vibe.md server** with Dokploy or Coolify, Docker is already running. Skip the Docker install step.

**Do not proceed until Docker and Bun are confirmed installed.**


## Phase 3C: Install Sandbox Backend

Based on `BACKEND`:

### Docker + Bun API

See [references/setup-docker-api.md](references/setup-docker-api.md) for the full service setup, including how to apply `RESOURCE_MEMORY` and `RESOURCE_CPUS` to the Bun API's environment.

Set these environment variables before starting the service:
```bash
export SANDBOX_MEMORY=RESOURCE_MEMORY   # e.g. 512m
export SANDBOX_CPUS=RESOURCE_CPUS       # e.g. 1
```

The API uses these values as defaults for every `docker run` command it spawns.

**Long-lived use case extra steps (if USE_CASE = "long-lived" or "both"):**
- Mount a named Docker volume per sandbox for persistence: `--volume sandboxmd-<id>:/workspace`
- Do not pass `--read-only` flag
- Increase TTL default to 24h or set TTL to 0 (no auto-destroy)

### Daytona

See [references/setup-daytona.md](references/setup-daytona.md) for the full Daytona self-hosted setup.

Templates are registered in step 5 of that guide. Apply resource limits via Daytona's workspace quota settings.

### Firecracker microVMs

See [references/setup-firecracker.md](references/setup-firecracker.md) for the full Firecracker setup.

Rootfs images for each template are prepared in step 4 of that guide. Resource limits (vCPU and memory) are applied per-VM in the Firecracker config.

**Do not proceed until the chosen backend is installed and responding to health checks.**


## Phase 3D: Generate API Credentials

**If AUTH_MODE = "none":** Skip this phase.

**If AUTH_MODE = "api-key":**

```bash
mkdir -p /etc/sandboxmd
API_KEY=$(openssl rand -hex 32 2>/dev/null || tr -dc 'a-f0-9' </dev/urandom | head -c 64)
echo "API key generated. Writing to config. (Not shown in chat.)"
```

Store `API_KEY` for use in Phase 5. Do not print it.

**If AUTH_MODE = "mtls":**

```bash
mkdir -p /etc/sandboxmd/certs

# Generate CA
openssl genrsa -out /etc/sandboxmd/certs/ca.key 4096
openssl req -new -x509 -days 3650 -key /etc/sandboxmd/certs/ca.key \
  -out /etc/sandboxmd/certs/ca.crt -subj "/CN=sandboxmd-ca"

# Generate server cert
openssl genrsa -out /etc/sandboxmd/certs/server.key 2048
openssl req -new -key /etc/sandboxmd/certs/server.key \
  -out /etc/sandboxmd/certs/server.csr -subj "/CN=sandboxmd-server"
openssl x509 -req -days 365 -in /etc/sandboxmd/certs/server.csr \
  -CA /etc/sandboxmd/certs/ca.crt -CAkey /etc/sandboxmd/certs/ca.key \
  -CAcreateserial -out /etc/sandboxmd/certs/server.crt

chmod 600 /etc/sandboxmd/certs/*.key
echo "mTLS certs generated in /etc/sandboxmd/certs/"
```

Set `API_KEY=""` (not used for mTLS).


## Phase 3E: Pull Sandbox Templates

Pull the Docker images for each chosen template. Skip this phase if the backend is Daytona or Firecracker (those manage images via their own tooling - see their setup guides).

```bash
# Pull only the images selected in TEMPLATES

# Node.js (if selected)
docker pull node:lts-slim

# Python (if selected)
docker pull python:3.12-slim

# Go (if selected)
docker pull golang:1.23-alpine

# Rust (if selected)
docker pull rust:slim

# Ubuntu (if selected)
docker pull ubuntu:24.04
```

Pull only the images the user selected. Skip the rest. Fail loudly if any pull fails:

```bash
docker pull <image> || { echo "FAILED to pull <image>"; exit 1; }
```

**Do not proceed until all selected images are pulled.**


## Phase 4: MCP Integration (if MCP_ENABLED = true)

Detect which CLI environment is running:

```bash
echo "CODEX=${CODEX:-false}"
echo "CODEX_SANDBOX=${CODEX_SANDBOX:-}"
```

The sandbox API exposes an HTTP REST API. Claude Code can reach it directly via the `WebFetch` or `Bash` (curl) tools without a dedicated MCP server binary.

To configure it properly, add the sandbox API URL and key to your project's `CLAUDE.md` or environment:

```bash
# Add to project CLAUDE.md so Claude Code knows the sandbox is available
cat >> .claude/CLAUDE.md << 'EOF'

## Sandbox

A self-hosted sandbox API is available for running commands in isolated environments.

- API URL: http://<SANDBOX_HOST>:7842  (ask for the actual URL)
- Auth: Bearer token (ask for the key from /etc/sandboxmd/config.json)
- Templates: [list templates selected]

To spawn a sandbox and run a command:
  POST /sandbox {"template":"node","ttl":300}
  POST /sandbox/:id/exec {"cmd":"node -e \"console.log('hello')\""}
  DELETE /sandbox/:id
EOF
```

Replace `<SANDBOX_HOST>` with the actual server address. Do not print IP or key in chat.

**If a dedicated MCP server package becomes available** (`@sandboxmd/mcp`), it can be wired into Claude Code settings with:
```json
{
  "mcpServers": {
    "sandbox": {
      "command": "bunx",
      "args": ["@sandboxmd/mcp"],
      "env": {
        "SANDBOX_API_URL": "http://<SANDBOX_HOST>:7842",
        "SANDBOX_API_KEY": "<key from /etc/sandboxmd/config.json>"
      }
    }
  }
}
```

**Claude Code mode:** Tell the user the sandbox API is ready at `http://<SANDBOX_HOST>:7842`. Claude Code can use it via curl through the Bash tool or WebFetch. *(Ask me for the actual URL and key.)*

**Codex mode:** The sandbox API is immediately usable via bash tool calls. No reload needed.


## Phase 5: Write Config and Verify

### Write the config file

Set the API URL:

```bash
API_URL="http://localhost:7842"
# If not local, use the server's address (do not print in chat)
```

Write the config (replace ALL_CAPS values with the actual values from Phase 2 before running):

```bash
mkdir -p /etc/sandboxmd

cat > /etc/sandboxmd/config.json << EOF
{
  "backend": "BACKEND_VALUE",
  "use_case": "USE_CASE_VALUE",
  "server_target": "SERVER_TARGET_VALUE",
  "cloud_provider": "CLOUD_PROVIDER_VALUE_OR_null",
  "templates": TEMPLATES_JSON_ARRAY,
  "resource_limit": {
    "memory": "RESOURCE_MEMORY_VALUE",
    "cpus": "RESOURCE_CPUS_VALUE"
  },
  "auth_mode": "AUTH_MODE_VALUE",
  "api_key": "API_KEY_VALUE_OR_empty",
  "api_url": "API_URL_VALUE",
  "api_port": 7842,
  "mcp_enabled": MCP_ENABLED_VALUE,
  "sandbox_version": "1.0.0",
  "provisioned_at": "$(date -u +"%Y-%m-%dT%H:%M:%SZ")"
}
EOF

chmod 600 /etc/sandboxmd/config.json
echo "Config written."
```

Substitution guide before running:
- `BACKEND_VALUE` - e.g. `docker-bun`
- `USE_CASE_VALUE` - e.g. `ephemeral`
- `SERVER_TARGET_VALUE` - e.g. `vibe`
- `CLOUD_PROVIDER_VALUE_OR_null` - e.g. `hetzner` or `null`
- `TEMPLATES_JSON_ARRAY` - e.g. `["node","python"]`
- `RESOURCE_MEMORY_VALUE` - e.g. `512m`
- `RESOURCE_CPUS_VALUE` - e.g. `1`
- `AUTH_MODE_VALUE` - e.g. `api-key`
- `API_KEY_VALUE_OR_empty` - paste the actual generated key (this is the only time it touches a file — it will not appear in chat)
- `API_URL_VALUE` - e.g. `http://localhost:7842`
- `MCP_ENABLED_VALUE` - `true` or `false`

### Verify - spawn a test sandbox

Read the API key from config for the test:

```bash
API_KEY=$(jq -r '.api_key' /etc/sandboxmd/config.json)
API_URL=$(jq -r '.api_url' /etc/sandboxmd/config.json)
```

Create a test sandbox:

```bash
CREATE_RESP=$(curl -s -X POST "$API_URL/sandbox" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_KEY" \
  -d '{"template":"node","ttl":60}')
echo "$CREATE_RESP"
SANDBOX_ID=$(echo "$CREATE_RESP" | jq -r '.id')
```

Exec a command inside:

```bash
curl -s -X POST "$API_URL/sandbox/$SANDBOX_ID/exec" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_KEY" \
  -d '{"cmd":"node -e \"console.log(process.version)\""}'
```

Destroy the test sandbox and verify it is gone:

```bash
curl -s -X DELETE "$API_URL/sandbox/$SANDBOX_ID" \
  -H "Authorization: Bearer $API_KEY"

# Confirm destroyed
curl -s "$API_URL/sandbox/$SANDBOX_ID" \
  -H "Authorization: Bearer $API_KEY" | jq -r '.error'
# Expected: "Sandbox not found"
```

**Do not declare the skill complete until a sandbox spawns, executes a command successfully, and is destroyed.**


## Completion Checklist

- [ ] Args routed correctly (--status / --reconfigure / --add-template / empty)
- [ ] Environment detected and confirmed (local or server)
- [ ] vibe.md config checked and VIBE_PLATFORM extracted
- [ ] Docker and KVM status detected
- [ ] All interview questions answered
- [ ] VPS provisioned and SSH'd into (if SERVER_TARGET = new-vps)
- [ ] Docker and Bun installed on target server (or confirmed for local)
- [ ] jq installed
- [ ] Sandbox backend installed and responding to health check (Docker+Bun / Daytona / Firecracker)
- [ ] Resource limits (RESOURCE_MEMORY, RESOURCE_CPUS) applied to backend config
- [ ] API credentials generated (if AUTH_MODE != none)
- [ ] All selected templates pulled or registered
- [ ] USE_CASE-specific config applied (persistent volumes for long-lived; short TTL for ephemeral)
- [ ] MCP / CLAUDE.md integration configured (if MCP_ENABLED)
- [ ] `/etc/sandboxmd/config.json` written with all fields including api_key and api_url
- [ ] Test sandbox spawned, exec'd, and destroyed successfully
- [ ] Firewall port 7842 opened (if server-based and not behind Dokploy/Coolify)


## Final Step: Optional Skills

The sandbox is ready. Offer the skills below one at a time, in order. Wait for the user's answer before moving to the next. **Always proceed to the next offer regardless of whether the user accepted or rejected the previous one - do not skip any.**

### Detecting the CLI environment

```bash
echo "CODEX=${CODEX:-false}"
echo "CODEX_SANDBOX=${CODEX_SANDBOX:-}"
```

- If `CODEX=true` or `CODEX_SANDBOX` is set - **Codex mode**: newly installed skills reload automatically.
- Otherwise - **Claude Code mode**: the user must run `/reload-plugins` before the skill becomes available.

### 1. Install the `ship` skill?

Full-cycle dev workflow optimized for cloud sandboxes: implement in a sandbox, verify, edge cases, tests, ship.

```bash
npx --yes skills list 2>/dev/null | grep -qE '^ship$' && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

If not installed, ask: **"Would you like me to install the `ship` skill? It enables `/ship <feature>` for full-cycle development — great for shipping from inside a sandbox."**

If yes: `npx --yes skills add -g amajorai/ship.md -a claude-code -y`

- **Codex mode:** invoke `/ship` immediately.
- **Claude Code mode:** tell the user: **"Run `/reload-plugins` in this session, then run `/ship`."** Do not invoke it yourself.

### 2. Install the `hardening` skill?

Secures the sandbox host: SSH hardening, fail2ban, UFW, unattended upgrades.

```bash
npx --yes skills list 2>/dev/null | grep -qE '^hardening$' && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

Ask: **"Would you like me to install and run the `hardening` skill to lock down the sandbox host server?"**

If yes: `npx --yes skills add -g amajorai/skills/skills/hardening -a claude-code -y`

- **Codex mode:** invoke `/hardening` immediately. Once it completes (or if the user declined), continue to offer #3.
- **Claude Code mode:** tell the user: **"Run `/reload-plugins` in this session, then run `/hardening`."** Do not invoke it yourself. Then immediately continue to offer #3.

### 3. Install the `party` skill?

Autonomous build agent - drop issues into GitHub Projects and it ships them from inside sandboxes while you sleep.

```bash
npx --yes skills list 2>/dev/null | grep -qE '^party$' && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

Ask: **"Would you like me to install the `party` skill? It turns GitHub Issues into a self-driving kanban board — ideal for running builds inside your new sandboxes."**

If yes: `npx --yes skills add -g amajorai/party.md -a claude-code -y`

- **Codex mode:** invoke `/party --setup` immediately.
- **Claude Code mode:** tell the user: **"Run `/reload-plugins` in this session, then run `/party --setup`."** Do not invoke it yourself.

### 4. Install the `fix` skill?

Systematic bug-fixing workflow. Instruments the codebase with targeted logging, reads logs to confirm root cause, makes a surgical fix, and verifies clean. Great for debugging issues inside a sandbox.

```bash
npx --yes skills list 2>/dev/null | grep -qE '^fix$' && echo "ALREADY_INSTALLED" || echo "NOT_INSTALLED"
```

Ask: **"Would you like me to install the `fix` skill? It enables `/fix <bug description>` for systematic debugging — reproduce the bug inside a fresh sandbox to isolate state."**

If yes: `npx --yes skills add -g amajorai/fix.md -a claude-code -y`

- **Codex mode:** tell the user fix is ready - invoke when needed.
- **Claude Code mode:** tell the user: **"Run `/reload-plugins` in this session so `/fix` becomes available."** Do not invoke it yourself.
