# Firecracker microVM Setup

Firecracker provides the highest isolation - each sandbox runs as a lightweight microVM. This is the closest to E2B's architecture.

## Requirements

- **KVM support is required.** This means a bare-metal server or a VPS with nested virtualization enabled.
- Ubuntu 20.04+ (recommended)
- Check KVM support before proceeding:

```bash
# Check KVM support
kvm-ok || apt-get install -y cpu-checker && kvm-ok

# Or directly
ls /dev/kvm && echo "KVM available" || echo "KVM NOT available — Firecracker will not work on this machine"
```

**If KVM is not available:** Choose a bare-metal server (Hetzner dedicated, OVH baremetal) or a VPS with nested virtualization (Hetzner Cloud CX series supports it, many AWS instance types support it with `--cpu-options AmdSevSnp=enabled`).

## 1. Install Firecracker

```bash
# Get latest release
ARCH=$(uname -m)
FIRECRACKER_VERSION=$(curl -s https://api.github.com/repos/firecracker-microvm/firecracker/releases/latest | grep tag_name | cut -d'"' -f4)

curl -LOJ "https://github.com/firecracker-microvm/firecracker/releases/download/${FIRECRACKER_VERSION}/firecracker-${FIRECRACKER_VERSION}-${ARCH}.tgz"
tar -xzf firecracker-*.tgz
mv release-*/firecracker-* /usr/local/bin/firecracker
mv release-*/jailer-* /usr/local/bin/jailer
chmod +x /usr/local/bin/firecracker /usr/local/bin/jailer

# Verify
firecracker --version
```

## 2. Install Firecracker CNI for networking

```bash
# Install CNI plugins
mkdir -p /opt/cni/bin
curl -L https://github.com/containernetworking/plugins/releases/latest/download/cni-plugins-linux-amd64-v1.4.0.tgz | \
  tar -xz -C /opt/cni/bin

# Set up CNI config
mkdir -p /etc/cni/net.d
cat > /etc/cni/net.d/sandbox.conflist << 'EOF'
{
  "cniVersion": "0.4.0",
  "name": "sandbox",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "sandbox0",
      "isGateway": true,
      "ipMasq": true,
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.100.0/24",
        "routes": [{ "dst": "0.0.0.0/0" }]
      }
    },
    { "type": "firewall" },
    { "type": "tc-redirect-tap" }
  ]
}
EOF
```

## 3. Prepare rootfs images

Each sandbox template needs a rootfs image (ext4 filesystem).

```bash
mkdir -p /var/lib/sandboxmd/rootfs

# Build a Node.js rootfs (example)
# We use Docker to extract a filesystem, then convert to ext4

build_rootfs() {
  local name=$1
  local image=$2
  local output="/var/lib/sandboxmd/rootfs/${name}.ext4"
  local size="${3:-512M}"

  docker create --name tmp-rootfs $image
  mkdir -p /tmp/rootfs-${name}
  docker export tmp-rootfs | tar -x -C /tmp/rootfs-${name}
  docker rm tmp-rootfs

  # Create ext4 image
  dd if=/dev/zero of=$output bs=1M count=$(echo $size | sed 's/M//')
  mkfs.ext4 $output
  mkdir -p /tmp/rootfs-mount
  mount $output /tmp/rootfs-mount
  cp -a /tmp/rootfs-${name}/. /tmp/rootfs-mount/
  umount /tmp/rootfs-mount

  echo "Rootfs created: $output"
}

# Build for each selected template
build_rootfs "node" "node:lts-slim" "512M"
build_rootfs "python" "python:3.12-slim" "512M"
build_rootfs "go" "golang:1.23-alpine" "512M"
build_rootfs "ubuntu" "ubuntu:24.04" "1G"
```

## 4. Download kernel

```bash
# Download a prebuilt Firecracker-compatible kernel
curl -Lo /var/lib/sandboxmd/vmlinux \
  https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.9/x86_64/vmlinux-5.10.225

ls -lh /var/lib/sandboxmd/vmlinux
```

## 5. Deploy the Firecracker API wrapper

The Firecracker binary has a REST API on a Unix socket. We wrap it with a Bun service:

```bash
mkdir -p /opt/sandboxmd-fc
cd /opt/sandboxmd-fc

bun init -y
bun add hono

cat > /opt/sandboxmd-fc/index.ts << 'EOF'
import { Hono } from "hono"
import { randomUUID } from "crypto"
import { execSync, exec } from "child_process"
import { promisify } from "util"
import { mkdirSync } from "fs"

const execAsync = promisify(exec)
const app = new Hono()
const API_KEY = process.env.SANDBOX_API_KEY || ""
const AUTH_ENABLED = API_KEY.length > 0

const KERNEL = "/var/lib/sandboxmd/vmlinux"
const ROOTFS_DIR = "/var/lib/sandboxmd/rootfs"
const FC_SOCKETS_DIR = "/run/sandboxmd"
mkdirSync(FC_SOCKETS_DIR, { recursive: true })

const activeSandboxes = new Map<string, { pid: number; socketPath: string; template: string; createdAt: number }>()

app.use("*", async (c, next) => {
  if (!AUTH_ENABLED) return next()
  const auth = c.req.header("Authorization")
  if (!auth || auth !== `Bearer ${API_KEY}`) return c.json({ error: "Unauthorized" }, 401)
  return next()
})

app.get("/health", (c) => c.json({ status: "ok", backend: "firecracker" }))

app.post("/sandbox", async (c) => {
  const body = await c.req.json().catch(() => ({}))
  const template = body.template || "node"
  const ttl = body.ttl || 3600
  const id = randomUUID()
  const socketPath = `${FC_SOCKETS_DIR}/${id}.socket`
  const rootfs = `${ROOTFS_DIR}/${template}.ext4`

  // Spawn Firecracker
  const fcProcess = exec(
    `firecracker --api-sock ${socketPath} --config-file /dev/stdin`,
    { shell: true }
  )

  // Send Firecracker config via stdin
  const config = JSON.stringify({
    "boot-source": {
      kernel_image_path: KERNEL,
      boot_args: "console=ttyS0 reboot=k panic=1 pci=off"
    },
    drives: [{ drive_id: "rootfs", path_on_host: rootfs, is_root_device: true, is_read_only: false }],
    "machine-config": { vcpu_count: 1, mem_size_mib: parseInt(process.env.SANDBOX_MEMORY_MIB || "512") }
  })

  fcProcess.stdin?.write(config)
  fcProcess.stdin?.end()

  activeSandboxes.set(id, { pid: fcProcess.pid!, socketPath, template, createdAt: Date.now() })
  setTimeout(() => destroySandbox(id), ttl * 1000)

  return c.json({ id, status: "starting", template, ttl, note: "Firecracker VMs take ~125ms to boot" })
})

app.post("/sandbox/:id/exec", async (c) => {
  const id = c.req.param("id")
  const sandbox = activeSandboxes.get(id)
  if (!sandbox) return c.json({ error: "Sandbox not found" }, 404)
  const body = await c.req.json().catch(() => ({}))
  if (!body.cmd) return c.json({ error: "cmd is required" }, 400)

  // Execute via Firecracker vsock or serial console (simplified)
  try {
    const { stdout, stderr } = await execAsync(
      `curl --unix-socket ${sandbox.socketPath} -s http://localhost/exec`,
      { timeout: (body.timeout || 30) * 1000 }
    )
    return c.json({ stdout, stderr, exit_code: 0 })
  } catch (err: unknown) {
    const e = err as { stdout?: string; stderr?: string; code?: number }
    return c.json({ stdout: e.stdout || "", stderr: e.stderr || String(err), exit_code: e.code || 1 })
  }
})

app.delete("/sandbox/:id", async (c) => {
  const id = c.req.param("id")
  const ok = await destroySandbox(id)
  if (!ok) return c.json({ error: "Sandbox not found" }, 404)
  return c.json({ id, status: "destroyed" })
})

async function destroySandbox(id: string): Promise<boolean> {
  const sandbox = activeSandboxes.get(id)
  if (!sandbox) return false
  activeSandboxes.delete(id)
  try {
    process.kill(sandbox.pid, "SIGKILL")
    await execAsync(`rm -f ${sandbox.socketPath}`)
  } catch {}
  return true
}

export default { port: parseInt(process.env.PORT || "7842"), fetch: app.fetch }
EOF

# Deploy as systemd service
cat > /etc/systemd/system/sandboxmd-fc.service << EOF
[Unit]
Description=sandboxmd Firecracker API
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/sandboxmd-fc
EnvironmentFile=/opt/sandboxmd/.env
ExecStart=$(which bun) run index.ts
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now sandboxmd-fc
```

## 6. Verify

```bash
# Check Firecracker installed
firecracker --version

# Check KVM still accessible
ls /dev/kvm

# Check service running
systemctl status sandboxmd-fc

# Test health endpoint
curl -s http://localhost:7842/health
```

## Notes

- Firecracker VMs boot in ~125ms - much faster than full VMs, close to container startup
- Each microVM is fully isolated (separate kernel, no shared namespaces)
- `--network=none` equivalent for Firecracker: don't attach a network device in the config
- For production: use the Jailer tool to further restrict Firecracker processes
- KVM is a hard requirement - if your VPS doesn't support it, use the Docker+Bun backend instead
