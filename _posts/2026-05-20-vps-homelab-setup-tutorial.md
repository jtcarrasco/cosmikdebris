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

For years my homelab strategy was "open ports and hope for the best." I had services bound to `0.0.0.0`, a UFW config I vaguely remembered setting up once, and the distinct background anxiety of someone who has read too many breach postmortems. It worked, in the same way that a screen door works. Technically a door.

Then I found a better way: lock everything behind [Tailscale](https://tailscale.com/), let [Caddy](https://caddyserver.com/) handle TLS, and never think about open ports again.

**What you'll end up with:** A VPS running [Docker](https://docs.docker.com/get-docker/) containers behind Caddy, fully locked to a Tailscale network, with automatic TLS and zero port-forwarding required.

**Prerequisites:** A VPS ([Ubuntu](https://ubuntu.com/) 22.04+), a Tailscale account, basic Linux comfort.

---

## Step 1: Provision the VPS

Any cheap VPS works. I use [RackNerd](https://www.racknerd.com/): $20-30/year for a 1-2 vCPU, 2GB RAM node handles this stack comfortably with room to spare.

After provisioning, do basic hardening before you do anything else. Yes, this part is less exciting than running services. Do it anyway.

```bash
# Update packages
apt update && apt upgrade -y

# Create a non-root user
adduser youruser
usermod -aG sudo youruser

# Set up SSH key auth (from your local machine)
ssh-copy-id youruser@your-vps-ip

# Harden SSH — drop a file in /etc/ssh/sshd_config.d/
cat > /etc/ssh/sshd_config.d/60-hardening.conf << 'EOF'
PasswordAuthentication no
PermitRootLogin no
X11Forwarding no
AllowTcpForwarding no
EOF

systemctl restart sshd
```

Install [fail2ban](https://www.fail2ban.org/). Three bad login attempts and it's a 24-hour ban, long enough to be annoying for attackers and short enough that you won't brick yourself if you mistype your password from a new device.

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

The `100.64.0.0/10` in `ignoreip` is Tailscale's subnet. Once you're on Tailscale, you'll never accidentally ban yourself again.

---

## Step 2: Install Tailscale

This is the part that turns a standard VPS setup into something genuinely elegant.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

Follow the auth link to connect the VPS to your Tailscale account. Once connected, note your VPS Tailscale IP (something like `100.x.y.z`). This becomes the only IP your services will ever listen on (not the public IP, not localhost, just the Tailscale IP).

Install Tailscale on every device you want to access your homelab from (MacBook, phone, whatever), connect them to the same account.

**Enable MagicDNS** in the [Tailscale admin console](https://login.tailscale.com/admin/dns). This gives your VPS a stable hostname like `your-machine.tail35225b.ts.net` that works from any of your devices. You'll use this hostname everywhere.

---

## Step 3: Install Docker

```bash
curl -fsSL https://get.docker.com | sh
usermod -aG docker youruser
```

Log out and back in for the group membership to take effect. If you skip this step you'll be prefacing every Docker command with `sudo` and quietly resenting the tutorial that didn't warn you.

---

## Step 4: Set Up Caddy as a Reverse Proxy

Caddy handles HTTPS automatically for your Tailscale domain: no cert management, no Certbot renewals, no `let's encrypt rate limit exceeded` emails at 2am. It just works.

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

Caddy will automatically provision TLS certificates for your Tailscale domain. Adding a new service is just adding a new block here and reloading.

Start it:
```bash
docker compose -f caddy.yml up -d
```

The critical thing in the compose file: `ports` is bound to `YOUR_TAILSCALE_IP`, not `0.0.0.0`. That one distinction is the entire security model. `0.0.0.0` means "anyone on the internet." `YOUR_TAILSCALE_IP` means "only devices on my Tailscale network." Never bind to `0.0.0.0` or you'll expose services to the public internet and undo all of step 1.

---

## Step 5: Add Services

Each service gets its own directory and compose file. They all join `proxy-net` so Caddy can reach them by container name. Here's [n8n](https://n8n.io/) as an example:

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

Add the corresponding block to your Caddyfile, then reload Caddy without downtime:
```bash
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```

Repeat this pattern for each service. My full stack at the moment (I use other stuff but I'm not telling):

| Service                                                | Port | Purpose             |
| ------------------------------------------------------ | ---- | ------------------- |
| [n8n](https://n8n.io/)                                 | 8084 | Workflow automation |
| [FreshRSS](https://freshrss.org/)                      | 8080 | RSS aggregator      |
| [Uptime Kuma](https://github.com/louislam/uptime-kuma) | 8083 | Site monitoring     |
| [Glance](https://github.com/glanceapp/glance)          | 8085 | Homelab dashboard   |

---

## Step 6: Lock Down UFW

Belt and suspenders. The Tailscale IP binding already prevents external access, but UFW adds an explicit deny layer:

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

Alternatively, use [Tailscale's ACL system](https://tailscale.com/kb/1018/acls/) to control which devices on your network can reach which services. That's worth exploring once the basics are solid.

---

## Step 7: Add Watchtower for Auto-Updates

Running containers means you're running software that has CVEs and updates. [Watchtower](https://containrrr.dev/watchtower/) pulls new images nightly and removes the old ones:

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

3am updates, automatic cleanup of old layers. Set it and forget it.

---

## Accessing Everything

Once set up, every service is accessible at `https://your-machine.tail35225b.ts.net:PORT` from any device on your Tailscale network. There's no VPN to manually connect. Tailscale runs in the background and handles it. Open your phone on a coffee shop's WiFi and your homelab dashboard loads like it's local.

SSH access: `ssh youruser@your-machine.tail35225b.ts.net` or `ssh youruser@YOUR_TAILSCALE_IP`

---

## Troubleshooting

**Service not reachable:** Check that the container is on `proxy-net` (`docker network inspect proxy-net`) and that your Caddyfile block has the right upstream hostname and port.

**TLS not working:** Make sure Tailscale MagicDNS is enabled in the admin console and that the hostname in your Caddyfile matches exactly. Caddy can't provision a cert for a hostname Tailscale doesn't recognize.

**Port exposed publicly:** Check your `ports` binding in the compose file. It must be `tailscale-ip:host-port:container-port`. If you see `host-port:container-port` without the IP prefix, it's bound to `0.0.0.0` and the whole internet can knock.

---

*Part of a series on building a practical, low-cost homelab with AI agents, self-hosted automation, and a Tailscale backbone.*
