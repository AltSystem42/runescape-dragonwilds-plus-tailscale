# Runescape Dragonwilds Server Files

Run your own Runescape: Dragonwilds server with Docker, Tailscale, and Traefik on a VPS.

[![Watch the setup guide](https://img.shields.io/badge/YouTube-Watch%20Tutorial-red)](https://www.youtube.com/watch?v=M8jxPDxR4W4)

---

## Overview

This project provides Docker configurations to run a self-hosted Runescape: Dragonwilds dedicated server. It uses a two-network architecture:

- **Home Server**: Runs the game server and Tailscale client inside Docker
- **VPS**: Runs Traefik as a reverse proxy to relay UDP traffic to your home server via Tailscale VPN

## Architecture

```
Players → VPS (Traefik :45000/UDP) → Tailscale VPN → Home Server (Dragonwilds :45000/UDP)
```

| Component | Location | Purpose |
|-----------|----------|---------|
| Dragonwilds Server | Home | Game server container |
| Tailscale | Home | VPN connection to VPS |
| Traefik | VPS | UDP proxy routing players to home via Tailscale |

---

## Quick Reference

### Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 45000 | UDP | Game server port (configure in .env) |
| 45000 | UDP | Traefik relay port (players connect here) |

### Environment Variables

Configure these in `home-server/.env`:

| Variable | Default | Description |
|----------|---------|-------------|
| `SERVER_PORT` | 45000 | Game server port (UDP) |
| `TZ` | America/New_York | Timezone for backups |
| `BACKUP_TIME` | 3:00 AM | Daily backup schedule |
| `BACKUP_DAILY` | true | Enable daily backups |
| `BACKUP_AFTER_UPDATE` | true | Backup before auto-updates |
| `ENABLE_AUTO_UPDATE` | true | Enable automatic server updates |
| `BACKUP_RETENTION_DAYS` | 30 | Days to keep backups |
| `ENABLE_DISCORD_NOTIF` | false | Enable Discord webhooks |
| `DISCORD_WEBHOOK_URL` | (empty) | Discord webhook URL |

### Tailscale Variables

Set these in `home-server/.env` (required):

| Variable | Description |
|----------|-------------|
| `TS_AUTHKEY` | Your Tailscale authkey |

---

## Commands Reference

### Tailscale Setup (VPS)

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Start Tailscale as exit node
sudo tailscale up --accept-routes --advertise-exit-node

# Get Tailscale IP (both methods)
tailscale ip -4
docker exec tailscale tailscale ip -4
```

### Docker Setup (Home Server)

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh
systemctl enable docker
systemctl start docker

# Install Docker Compose
sudo apt update && sudo apt install docker-compose-plugin
docker compose version
```

### Server Management

```bash
# Start the server (from home-server directory)
docker compose up -d

# View logs
docker compose logs -f

# Stop the server
docker compose down

# Restart a specific service
docker compose restart dragonwilds
```

---

## Configuration

### 1. Configure Tailscale Authkey

1. Go to [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys)
2. Create an auth key (reusable recommended)
3. Add to `home-server/.env`:
   ```
   TS_AUTHKEY=tskey-auth-kxxxxx
   ```

### 2. Configure Exit Node

In `home-server/docker-compose.yml`, update the exit node IP:

```yaml
command: >
  sh -c "tailscaled &
        sleep 5
        tailscale up --authkey=$TS_AUTHKEY --exit-node=100.x.x.x --accept-routes --operator=root
        tail -f /dev/null"
```

Replace `100.x.x.x` with your VPS Tailscale IP.

### 3. Configure Traefik

In `vps/traefik.yml`, update the server address:

```yaml
services:
  service_45000:
    loadBalancer:
      servers:
        - address: "100.x.x.x:45000"
```

Replace `100.x.x.x` with your home server Tailscale IP.

---

## Troubleshooting

### Tailscale Connection Issues

```bash
# Check Tailscale status
docker exec tailscale tailscale status

# Restart Tailscale container
docker compose restart tailscale

# View Tailscale logs
docker compose logs tailscale
```

### Server Won't Start

```bash
# Check if ports are in use
sudo netstat -tulpn | grep 45000

# Verify .env file exists
ls -la .env

# Check Docker logs
docker compose logs dragonwilds
```

### Players Can't Connect

1. Verify Traefik is running on VPS: `docker compose ps`
2. Check UDP port 45000 is open on VPS firewall
3. Verify Tailscale routes are accepted on both ends
4. Confirm exit node is configured on home server

## Credits

Based on the tutorial by [Your Name/Channel](https://www.youtube.com/watch?v=M8jxPDxR4W4)
