# Proxmox Docker Compose install

This path runs OWL on a Debian/Ubuntu Proxmox VM or LXC with server-friendly named volumes. Secrets stay in `.container/proxmox.env`, which is ignored by git.

## 1. Install Docker and Compose

On a Debian/Ubuntu VM or privileged LXC:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg git
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
. /etc/os-release
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian ${VERSION_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker "$USER"
newgrp docker
```

For Ubuntu, use the Ubuntu Docker repository instead if your VM is not Debian-based:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
newgrp docker
```

For an LXC, enable nesting/keyctl in Proxmox before installing Docker.

## 2. Clone OWL

```bash
git clone https://github.com/itsablabla/garzaos-owl.git
cd garzaos-owl
```

## 3. Configure secrets

```bash
cp .container/proxmox.env.example .container/proxmox.env
nano .container/proxmox.env
```

Replace placeholder values with your real OWL API keys. Do not commit `.container/proxmox.env`.

## 4. Start OWL

```bash
docker compose --env-file .container/proxmox.env -f .container/docker-compose.proxmox.yml up -d
```

OWL exposes Gradio on port `7860`:

```text
http://<proxmox-vm-ip>:7860
```

Check status and logs:

```bash
docker compose --env-file .container/proxmox.env -f .container/docker-compose.proxmox.yml ps
docker compose --env-file .container/proxmox.env -f .container/docker-compose.proxmox.yml logs -f owl
```

The compose file uses named volumes for OWL data, pip cache, and Playwright cache so it does not depend on host `~/.cache` paths.

## Optional Proxmox MCP API sidecar

This repository does not include a Proxmox MCP API Dockerfile/app, so the default Proxmox compose does not use `build: .`. To run a sidecar, provide an existing image and a local config file.

1. Add sidecar secrets and image to `.container/proxmox.env`:

```dotenv
PROXMOX_MCP_API_IMAGE=your-registry/proxmox-mcp-api:tag
PROXMOX_API_KEY=replace_with_proxmox_api_key
MCPO_CORS_ALLOW_ORIGINS=https://s1.garza.online,http://s1.garza.online,https://mcp.garza.online,http://mcp.garza.online
```

2. Create the mounted config file:

```bash
mkdir -p .container/proxmox-config
nano .container/proxmox-config/config.json
```

3. Start OWL plus the sidecar:

```bash
docker compose \
  --env-file .container/proxmox.env \
  -f .container/docker-compose.proxmox.yml \
  -f .container/docker-compose.proxmox-mcp.yml \
  up -d
```

The sidecar listens on `8811` and reads `PROXMOX_API_KEY` only from `.container/proxmox.env`.
