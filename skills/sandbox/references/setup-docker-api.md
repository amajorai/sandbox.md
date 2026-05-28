# Docker + Bun API Setup

This is the lightweight backend option: a Bun + Hono REST service that wraps the Docker daemon.

## 1. Create service directory

```bash
mkdir -p /opt/sandboxmd
cd /opt/sandboxmd
```

## 2. Initialize Bun project

```bash
bun init -y
bun add hono
```

## 3. Write the API server

```bash
cat > /opt/sandboxmd/index.ts << 'EOF'
import { Hono } from "hono"
import { randomUUID } from "crypto"
import { execSync, exec } from "child_process"
import { promisify } from "util"

const execAsync = promisify(exec)

const app = new Hono()

const API_KEY = process.env.SANDBOX_API_KEY || ""
const AUTH_ENABLED = API_KEY.length > 0

const RESOURCE_CONFIG = {
  memory: process.env.SANDBOX_MEMORY || "512m",
  cpus: process.env.SANDBOX_CPUS || "1",
}

const SANDBOX_TTL_DEFAULT = 3600 // 1 hour in seconds
const activeSandboxes = new Map<string, { containerId: string; template: string; createdAt: number; ttl: number }>()

// Auth middleware
app.use("*", async (c, next) => {
  if (!AUTH_ENABLED) return next()
  const auth = c.req.header("Authorization")
  if (!auth || auth !== `Bearer ${API_KEY}`) {
    return c.json({ error: "Unauthorized" }, 401)
  }
  return next()
})

// Health check
app.get("/health", (c) => c.json({ status: "ok", version: "1.0.0" }))

// List sandboxes
app.get("/sandboxes", (c) => {
  const list = Array.from(activeSandboxes.entries()).map(([id, s]) => ({
    id,
    template: s.template,
    created_at: new Date(s.createdAt).toISOString(),
    ttl: s.ttl,
    age_seconds: Math.floor((Date.now() - s.createdAt) / 1000),
  }))
  return c.json(list)
})

// Create sandbox
app.post("/sandbox", async (c) => {
  const body = await c.req.json().catch(() => ({}))
  const template = body.template || "node"
  const ttl = body.ttl || SANDBOX_TTL_DEFAULT

  const imageMap: Record<string, string> = {
    node: "node:lts-slim",
    python: "python:3.12-slim",
    go: "golang:1.23-alpine",
    rust: "rust:slim",
    ubuntu: "ubuntu:24.04",
  }

  const image = imageMap[template]
  if (!image) return c.json({ error: `Unknown template: ${template}` }, 400)

  const id = randomUUID()

  try {
    const { stdout } = await execAsync(
      `docker run -d --name sandboxmd-${id} ` +
      `--memory="${RESOURCE_CONFIG.memory}" ` +
      `--cpus="${RESOURCE_CONFIG.cpus}" ` +
      `--network=none ` +
      `--read-only --tmpfs /tmp:rw,size=256m ` +
      `${image} sleep ${ttl}`
    )
    const containerId = stdout.trim()
    activeSandboxes.set(id, { containerId, template, createdAt: Date.now(), ttl })

    // Auto-destroy after TTL
    setTimeout(() => destroySandbox(id), ttl * 1000)

    return c.json({ id, status: "ready", template, ttl })
  } catch (err: unknown) {
    const msg = err instanceof Error ? err.message : String(err)
    return c.json({ error: `Failed to create sandbox: ${msg}` }, 500)
  }
})

// Exec in sandbox
app.post("/sandbox/:id/exec", async (c) => {
  const id = c.req.param("id")
  const sandbox = activeSandboxes.get(id)
  if (!sandbox) return c.json({ error: "Sandbox not found" }, 404)

  const body = await c.req.json().catch(() => ({}))
  const cmd = body.cmd
  if (!cmd) return c.json({ error: "cmd is required" }, 400)

  const timeout = body.timeout || 30

  try {
    const { stdout, stderr } = await execAsync(
      `docker exec sandboxmd-${id} sh -c ${JSON.stringify(cmd)}`,
      { timeout: timeout * 1000 }
    )
    return c.json({ stdout, stderr, exit_code: 0 })
  } catch (err: unknown) {
    const e = err as { stdout?: string; stderr?: string; code?: number }
    return c.json({
      stdout: e.stdout || "",
      stderr: e.stderr || String(err),
      exit_code: e.code || 1,
    })
  }
})

// Get sandbox info
app.get("/sandbox/:id", (c) => {
  const id = c.req.param("id")
  const sandbox = activeSandboxes.get(id)
  if (!sandbox) return c.json({ error: "Sandbox not found" }, 404)
  return c.json({
    id,
    template: sandbox.template,
    created_at: new Date(sandbox.createdAt).toISOString(),
    ttl: sandbox.ttl,
    age_seconds: Math.floor((Date.now() - sandbox.createdAt) / 1000),
  })
})

// Destroy sandbox
app.delete("/sandbox/:id", async (c) => {
  const id = c.req.param("id")
  const destroyed = await destroySandbox(id)
  if (!destroyed) return c.json({ error: "Sandbox not found" }, 404)
  return c.json({ id, status: "destroyed" })
})

async function destroySandbox(id: string): Promise<boolean> {
  const sandbox = activeSandboxes.get(id)
  if (!sandbox) return false
  activeSandboxes.delete(id)
  try {
    await execAsync(`docker rm -f sandboxmd-${id}`)
  } catch {}
  return true
}

// Cleanup expired sandboxes on startup
try {
  const running = execSync("docker ps --filter 'name=sandboxmd-' --format '{{.Names}}'").toString()
  running.split("\n").filter(Boolean).forEach(async (name) => {
    await execAsync(`docker rm -f ${name}`).catch(() => {})
  })
} catch {}

export default {
  port: parseInt(process.env.PORT || "7842"),
  fetch: app.fetch,
}
EOF
```

## 4. Create environment file

```bash
cat > /opt/sandboxmd/.env << EOF
SANDBOX_API_KEY=$(openssl rand -hex 32)
SANDBOX_MEMORY=512m
SANDBOX_CPUS=1
PORT=7842
EOF

chmod 600 /opt/sandboxmd/.env
echo "Environment file written. API key is stored in /opt/sandboxmd/.env — do not print it."
```

## 5. Run as systemd service (standalone server)

```bash
cat > /etc/systemd/system/sandboxmd.service << EOF
[Unit]
Description=sandboxmd - Self-hosted sandbox API
After=docker.service
Requires=docker.service

[Service]
Type=simple
User=root
WorkingDirectory=/opt/sandboxmd
EnvironmentFile=/opt/sandboxmd/.env
ExecStart=$(which bun) run index.ts
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now sandboxmd
systemctl status sandboxmd
```

## 6. Run via Dokploy or Coolify (vibe.md server)

If on a vibe.md server, deploy as an app instead:

**Dokploy:**
```bash
# Create a Dockerfile for the service
cat > /opt/sandboxmd/Dockerfile << 'EOF'
FROM oven/bun:1
WORKDIR /app
COPY package.json bun.lockb* ./
RUN bun install --frozen-lockfile
COPY . .
EXPOSE 7842
CMD ["bun", "run", "index.ts"]
EOF

# Push to a private git repo and deploy via Dokploy UI
# Or use the Dokploy CLI if available
```

**Coolify:** Use the Dockerfile above in a new application, set environment variables via the Coolify UI.

## 7. Open firewall port

```bash
# UFW
ufw allow 7842/tcp comment "sandboxmd API"

# Or iptables
iptables -A INPUT -p tcp --dport 7842 -j ACCEPT
```

Note: if using Dokploy/Coolify, they handle port exposure. You may not need to open the port directly.

## 8. Verify

```bash
curl -s http://localhost:7842/health
# Expected: {"status":"ok","version":"1.0.0"}
```
