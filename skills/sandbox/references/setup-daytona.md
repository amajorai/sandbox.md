# Daytona Self-Hosted Setup

Daytona is an open-source development environment manager. This guide sets it up in server mode on your host.

## Requirements

- Linux server (Ubuntu 22.04+ recommended)
- Docker installed and running
- 2+ vCPU, 4 GB RAM minimum
- Ports 3986, 3987 open (Daytona API + dashboard)

## 1. Install Daytona

```bash
# Install the Daytona binary
curl -sfL https://download.daytona.io/daytona/install.sh | bash

# Verify
daytona --version
```

## 2. Start Daytona server

```bash
# Run Daytona in server mode (background)
daytona server &

# Or as a systemd service
cat > /etc/systemd/system/daytona.service << EOF
[Unit]
Description=Daytona Server
After=docker.service
Requires=docker.service

[Service]
Type=simple
User=root
ExecStart=$(which daytona) server
Restart=always
RestartSec=5
Environment=HOME=/root

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now daytona
```

## 3. Configure the server

```bash
# Configure Daytona server settings
daytona server config

# Set the API URL (use your server's IP)
# This is the URL clients and Claude Code will use to connect
```

## 4. Get API credentials

```bash
# Get the server API key
daytona api-key generate sandbox-key

# Get the server URL
daytona server info
```

Store these - they go into `/etc/sandboxmd/config.json` and Claude Code MCP settings.

## 5. Register workspace templates

```bash
# List available templates
daytona template list

# Add a custom template (optional)
daytona template create --name node-sandbox \
  --image node:lts-slim \
  --entrypoint "/bin/sh"
```

## 6. Open firewall

```bash
ufw allow 3986/tcp comment "Daytona API"
ufw allow 3987/tcp comment "Daytona Dashboard"
```

## 7. Verify

```bash
# Check server is running
daytona server info

# Create a test workspace
daytona create --no-ide --name test-sandbox

# List workspaces
daytona list

# Remove test workspace
daytona delete test-sandbox
```

## Usage from Claude Code (via CLI)

Once integrated via MCP, Claude Code can:

```bash
# Create a workspace
daytona create --no-ide --name my-project --repo https://github.com/user/repo

# SSH into workspace
daytona ssh my-project

# Run a command
daytona exec my-project -- node index.js

# Stop workspace
daytona stop my-project

# Delete workspace
daytona delete my-project
```

## Daytona API (for programmatic use)

The Daytona server exposes a REST API on port 3986. See the Daytona docs for the full spec. Key endpoints:

- `POST /workspace` - create workspace
- `GET /workspace` - list workspaces
- `POST /workspace/:id/start` - start
- `POST /workspace/:id/stop` - stop
- `DELETE /workspace/:id` - delete
