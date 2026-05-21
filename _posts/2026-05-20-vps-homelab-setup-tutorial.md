---
title: "How to Build a Self-Hosted VPS Homelab With Tailscale and Docker"
date: 2026-05-20 09:00:00 -0800
categories: [Homelab, Tutorial]
tags: [homelab, self-hosting, vps, tailscale, docker, tutorial]
author: jason
pin: false
image:
  path: /assets/img/posts/vps-homelab-tutorial.jpg
  alt: VPS homelab with Tailscale and Docker
---

This is a complete walkthrough for building a secure, always-accessible self-hosted stack on a cheap VPS — with no public ports exposed and every service reachable from any device.

**What you'll end up with:** A VPS running Docker containers behind Caddy, fully locked to a Tailscale network, with automatic TLS and zero port-forwarding required.

**Prerequisites:** A VPS (Ubuntu 22.04+), a Tailscale account, basic Linux comfort.

---

## Step 1: Provision the VPS

Any cheap VPS works. I use RackNerd — $20-30/year for a 1-2 vCPU, 2GB RAM node is enough for this stack.

After provisioning, do basic hardening:

```bash
# Update packages
apt update && apt upgrade -y

# Create a non-root user
adduser User
usermod -aG sudo User

# Set up SSH key auth (from your local machine)
ssh-copy-id user@your-vps-ip

# Harden SSH — edit /etc/ssh/sshd_config or drop a file in /etc/ssh/sshd_config.d/
cat > /etc/ssh/sshd_config.d/60-hardening.conf << 'EOF'
PasswordAuthentication no
PermitRootLogin no
X11Forwarding no
AllowTcpForwarding no
EOF

systemctl restart sshd
```

Install fail2ban:

```bash
apt install fail2ban -y
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
maxretry = 3
bantime = 86400
ignoreip = 127.0.0.1/8 100.64.0.0/10

[sshd]
enabled = true
EOF
systemctl enable --now fail2ban
```

---

## Step 2: Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

Follow the auth link to connect the VPS to your Tailscale account. Once connected, note your VPS Tailscale IP (e.g. `YOUR_TAILSCALE_IP`). This is the only IP your services will listen on.

Install Tailscale on every device you want to access your VPS from (MacBook, phone, etc.) and connect them to the same account.

**Enable MagicDNS** in the Tailscale admin console — this gives your VPS a stable hostname like `your-machine.tail35225b.ts.net`.

---

## Step 3: Install Docker

```bash
curl -fsSL https://get.docker.com | sh
usermod -aG docker youruser
```

Log out and back in for group membership to take effect.

---

## Step 4: Set Up Caddy as a Reverse Proxy

Caddy handles HTTPS automatically for your Tailscale domain. Create a project structure:

```bash
mkdir -p ~/dev/docker/caddy
cd ~/dev/docker/caddy
```

`caddy.yml`:
```yaml
services:
  caddy:
    image: caddy:latest
    container_name: caddy
    ports:
      - "YOUR_TAILSCALE_IP:80:80"     # Replace with your Tailscale IP
      - "YOUR_TAILSCALE_IP:443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - proxy-net
    restart: unless-stopped

volumes:
  caddy_data:
  caddy_config:

networks:
  proxy-net:
    name: proxy-net
```

`Caddyfile`:
```
your-machine.tail35225b.ts.net:8080 {
    reverse_proxy n8n:5678
}

your-machine.tail35225b.ts.net:8083 {
    reverse_proxy uptime-kuma:3001
}
```

Caddy will automatically provision TLS certificates for your Tailscale domain. Add a block per service.

Start it:
```bash
docker compose -f caddy.yml up -d
```

**Key point:** Note the `ports` binding to `YOUR_TAILSCALE_IP` — that's your Tailscale IP. Never bind to `0.0.0.0` or you'll expose services to the public internet.

---

## Step 5: Add Services

Each service gets its own directory and compose file. Example for n8n:

```bash
mkdir -p ~/dev/docker/n8n
```

`n8n.yml`:
```yaml
services:
  n8n:
    image: n8nio/n8n
    container_name: n8n
    hostname: n8n
    environment:
      N8N_HOST: your-machine.tail35225b.ts.net
      N8N_PORT: 5678
      N8N_PROTOCOL: https
      WEBHOOK_URL: https://your-machine.tail35225b.ts.net:8084/
      GENERIC_TIMEZONE: America/Los_Angeles
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - proxy-net
    restart: unless-stopped

volumes:
  n8n_data:

networks:
  proxy-net:
    external: true
```

Start it:
```bash
docker compose -f n8n.yml up -d
```

Add the corresponding Caddy entry and reload: `docker exec caddy caddy reload --config /etc/caddy/Caddyfile`

Repeat for each service. My full stack:

| Service | Port | Purpose |
|---------|------|---------|
| n8n | 8084 | Workflow automation |
| FreshRSS | 8080 | RSS aggregator |
| Uptime Kuma | 8083 | Site monitoring |
| Glance | 8085 | Homelab dashboard |

---

## Step 6: Lock Down UFW

Restrict all dev ports to Tailscale's subnet:

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow in on tailscale0  # Allow all Tailscale traffic
ufw deny 8080               # Block public access to service ports
ufw deny 8083
ufw deny 8084
ufw deny 8085
ufw enable
```

Or use Tailscale's built-in ACLs to control which devices can reach which services.

---

## Step 7: Add Watchtower for Auto-Updates

```yaml
services:
  watchtower:
    image: containrrr/watchtower
    container_name: watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: --schedule "0 0 3 * * *" --cleanup
    restart: unless-stopped
```

This pulls updated images nightly at 3am and removes old ones.

---

## Accessing Everything

Once set up, every service is accessible at `https://your-machine.tail35225b.ts.net:PORT` from any device on your Tailscale network. No VPN to connect to — Tailscale handles it automatically in the background.

SSH access: `ssh youruser@your-machine.tail35225b.ts.net` or `ssh youruser@YOUR_TAILSCALE_IP`

---

## Troubleshooting

**Service not reachable:** Check that the container is on `proxy-net` and Caddy's config has the right port.

**TLS not working:** Make sure Tailscale MagicDNS is enabled and the hostname matches your Caddyfile.

**Port exposed publicly:** Check your `ports` binding — it must be `tailscale-ip:host-port:container-port`, not `host-port:container-port`.

---

*Part of a series on building a practical, low-cost homelab with AI agents, self-hosted automation, and a Tailscale backbone.*
