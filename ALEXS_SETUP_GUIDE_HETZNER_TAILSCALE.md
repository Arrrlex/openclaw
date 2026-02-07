# OpenClaw VPS Setup Guide

How to set up an OpenClaw agent on a fresh Hetzner VPS (Ubuntu), accessible via a public HTTPS URL using Tailscale Funnel.

## Architecture

```
Browser ──HTTPS──▶ Tailscale Funnel ──▶ 127.0.0.1:18789 (host) ──Docker──▶ 0.0.0.0:18789 (container)
```

- The gateway runs inside Docker, bound to `0.0.0.0` within the container
- Docker maps the port to `127.0.0.1:18789` on the host (loopback only, not publicly exposed)
- Tailscale Funnel provides public HTTPS access with automatic TLS
- Password auth protects the gateway since it's publicly reachable

## Prerequisites

- A Hetzner VPS (or similar) running Ubuntu
- SSH root access
- A Tailscale account (free tier works)
- A Moonshot API key (for Kimi K2.5 model)

## 1. Server Base Setup

```bash
apt-get update
apt-get install -y git curl ca-certificates

# Install Docker
curl -fsSL https://get.docker.com | sh
```

## 2. Clone the Repo

```bash
cd /root
git clone https://github.com/Arrrlex/openclaw.git
cd openclaw
```

This fork includes a fix for password persistence in the web UI (upstream bug: password passed via URL parameter was not saved to localStorage).

## 3. Create Config Directories

```bash
mkdir -p /root/.openclaw /root/.openclaw/workspace
chown -R 1000:1000 /root/.openclaw /root/.openclaw/workspace
```

UID 1000 is the `node` user inside the container.

## 4. Create `.env` File

```bash
cd /root/openclaw
cat > .env << 'EOF'
OPENCLAW_IMAGE=openclaw:local
OPENCLAW_GATEWAY_TOKEN=<generate with: openssl rand -hex 24>
OPENCLAW_GATEWAY_BIND=lan
OPENCLAW_GATEWAY_PORT=18789

OPENCLAW_CONFIG_DIR=/root/.openclaw
OPENCLAW_WORKSPACE_DIR=/root/.openclaw/workspace

GOG_KEYRING_PASSWORD=<generate with: openssl rand -hex 32>
XDG_CONFIG_HOME=/home/node/.openclaw
OPENCLAW_BRIDGE_PORT=18790
OPENCLAW_EXTRA_MOUNTS=
OPENCLAW_HOME_VOLUME=
OPENCLAW_DOCKER_APT_PACKAGES=
EOF
```

## 5. Build and Onboard

```bash
# Build the Docker image
docker compose build

# Run interactive onboarding (sets up model provider, API key, gateway auth)
docker compose run --rm openclaw-gateway node dist/index.js onboard
```

During onboarding, when prompted:
- **Gateway bind**: `lan`
- **Gateway auth mode**: `password`
- **Gateway password**: choose something secure, or generate with `openssl rand -base64 24`
- **Gateway token**: paste the same `OPENCLAW_GATEWAY_TOKEN` value from `.env`
- **Model provider**: Moonshot
- **Model**: Kimi K2.5
- **API key**: your Moonshot API key
- **Tailscale**: off (we manage it on the host instead)

**Important**: The gateway token in `.env` and in `~/.openclaw/openclaw.json` (`gateway.auth.token`) must match. If they diverge, the web UI won't connect. The config file value takes precedence over the env var.

## 6. Start the Gateway

```bash
docker compose up -d openclaw-gateway

# Verify it's running
docker logs openclaw-openclaw-gateway-1
# Should show: listening on ws://0.0.0.0:18789
```

## 7. Install and Configure Tailscale

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Authenticate (opens a URL to visit in your browser)
tailscale up

# Enable Funnel (may prompt you to enable it in the admin console)
tailscale funnel --bg 18789
```

After this, your agent is accessible at:
```
https://<hostname>.tailbb80e7.ts.net/
```

## 8. Connect

Open the URL in your browser with the password as a query parameter:

```
https://<hostname>.tailbb80e7.ts.net/?password=<your-gateway-password>
```

The password is saved to localStorage and stripped from the URL. Subsequent visits just need the base URL.

If you see "pairing required", approve the device from the server:

```bash
docker exec openclaw-openclaw-gateway-1 node dist/index.js devices list
docker exec openclaw-openclaw-gateway-1 node dist/index.js devices approve <request-id>
```

## Common Operations

```bash
# View logs
docker compose logs -f openclaw-gateway

# Restart gateway
docker compose restart openclaw-gateway

# Rebuild after code changes (use `up -d` to pick up .env changes)
docker compose build && docker compose up -d openclaw-gateway

# Check gateway health
docker exec openclaw-openclaw-gateway-1 node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"

# List/approve/reject devices
docker exec openclaw-openclaw-gateway-1 node dist/index.js devices list
docker exec openclaw-openclaw-gateway-1 node dist/index.js devices approve <request-id>

# Reconfigure
docker compose run --rm openclaw-gateway node dist/index.js configure
```

## Key Files

| File | Purpose |
|------|---------|
| `/root/openclaw/.env` | Docker environment variables (tokens, paths, bind settings) |
| `/root/.openclaw/openclaw.json` | Main openclaw config (auth, model, gateway settings) |
| `/root/.openclaw/agents/main/agent/auth-profiles.json` | LLM API keys (plain text) |
| `/root/.openclaw/agents/main/agent/models.json` | Model provider configuration |
| `/root/.openclaw/devices/paired.json` | Approved device list |
| `/root/.openclaw/devices/pending.json` | Devices waiting for approval |

## Security Notes

- The Docker port mapping (`127.0.0.1:18789:18789`) ensures the gateway is never directly exposed to the internet
- Tailscale Funnel provides HTTPS with automatic certificate management
- Password auth is required when using Funnel since the URL is publicly reachable
- API keys are stored in plain text at `~/.openclaw/agents/main/agent/auth-profiles.json` - protect this directory
- The `.env` file contains secrets - do not commit it to version control
