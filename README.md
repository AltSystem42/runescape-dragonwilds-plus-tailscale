# Runescape Dragonwilds Server Files

Run your own Runescape: Dragonwilds server with Docker, Tailscale, and Traefik on a VPS.

[![Watch the setup guide](https://img.shields.io/badge/YouTube-Watch%20Tutorial-red)](https://www.youtube.com/watch?v=M8jxPDxR4W4)

---

## Overview

This project provides Docker configurations to run a self-hosted Runescape: Dragonwilds dedicated server. It uses a two-network architecture:

- **Home Server**: Runs the game server and Tailscale client inside Docker
- **VPS**: Runs Traefik as a reverse proxy to relay UDP traffic to your home server via Tailscale VPN

The game server uses the [`andyaltsys/dragonwilds-dedicated-server`](https://github.com/AltSystem42/runescape-dragonwilds-dedicated-server-docker) image, which installs the official dedicated server (Steam App ID `4019830`) via SteamCMD (no Steam account needed) and manages it for you:

- **Automatic updates** — checks for new builds every `UPDATE_TIME` seconds and updates only when no players are online
- **Scheduled backups** — daily backups with retention, also idle-aware, with catch-up if the server was down at the scheduled time
- **Player monitoring** — tails the game log for join/leave events so updates and backups never interrupt anyone
- **Discord notifications** — optional alerts for installs, updates, backups, and player join/leave
- **Environment-driven config** — server name, world name, passwords, and `OwnerId` are written into `DedicatedServer.ini` from `.env` on every start

## Architecture

```
Players → VPS (Traefik :45000/UDP) → Tailscale VPN → Home Server (Dragonwilds :45000/UDP)
```

| Component | Location | Purpose |
|-----------|----------|---------|
| Dragonwilds Server | Home | Game server container |
| Tailscale | Home | VPN connection to VPS |
| Traefik | VPS | UDP proxy routing players to home via Tailscale |

The game container shares the Tailscale container's network (`network_mode: service:tailscale`), so it is only reachable through the Tailscale IP that Traefik forwards to.

---

## Quick Reference

### Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 45000 | UDP | Game server port — set `SERVER_PORT=45000` in `home-server/.env` (image default is 7777, this project overrides it) |
| 45000 | UDP | Traefik relay port (players connect here) |

Both ends must match: Traefik relays to `100.x.x.x:45000`, so the container must listen on **45000**.

### Environment Variables

Configure these in `home-server/.env`:

| Variable | Default | Description |
|----------|---------|-------------|
| `SERVER_PORT` | 7777 | UDP port for server connections (use `45000` in this project) |
| `TZ` | UTC | Timezone for scheduled backups |
| `ENABLE_AUTO_UPDATE` | true | Enable automatic server updates |
| `UPDATE_TIME` | 3600 | Seconds between update checks (default: 1 hour) |
| `SERVER_STOP_TIMEOUT` | 120 | Max seconds to wait for the server to stop cleanly before force-killing it during updates/backups |
| `BACKUP_AFTER_UPDATE` | true | Backup saves after each update |
| `BACKUP_DAILY` | true | Run daily scheduled backup |
| `BACKUP_TIME` | 3:00 AM | Daily backup time (12-hour format with AM/PM) |
| `BACKUP_RETENTION_DAYS` | 30 | Days to keep backups |
| `POLL_INTERVAL` | 60 | Seconds between checks of the daily-backup schedule — also the retry interval while waiting for the server to become idle |
| `IDLE_WAIT` | 360 | Seconds of no players before an update/backup proceeds (default: 6 min) |
| `ENABLE_DISCORD_NOTIF` | false | Enable Discord webhook notifications |
| `DISCORD_WEBHOOK_URL` | (empty) | Discord webhook URL |
| `LOG_TO_STDOUT` | true | Also echo script log lines to `docker logs`. Set `false` to keep script messages in the log file only |
| `MAX_LOG_SIZE` | 5242880 (5 MB) | Rotate the script log once it reaches this many bytes |
| `LOG_RETENTION` | 5 | Number of rotated script log generations to keep (`MAX_LOG_SIZE` × `LOG_RETENTION` bounds disk usage) |

### Server Settings (written to `DedicatedServer.ini`)

These are applied to `server/RSDragonwilds/Saved/Config/LinuxServer/DedicatedServer.ini` (container path: `/home/ubuntu/Steam/RSDragonwilds/Saved/Config/LinuxServer/DedicatedServer.ini`) on every container start. Set a variable to override the ini value; leave it empty to keep whatever is already in the ini. The defaults only apply when the key is missing entirely (fresh install).

| Variable | Default | Description |
|----------|---------|-------------|
| `OWNER_ID` | (empty) | **Required.** Your in-game "My Player Id" (bottom of the game's Settings menu) — makes that player the server owner (only they can ban/unban). The server will not start without it |
| `SERVER_NAME` | `Server-<timestamp>` | Server name shown in the world browser |
| `DEFAULT_WORLD_NAME` | `World-<timestamp>` | World name used to find the server in the world browser |
| `ADMIN_PASSWORD` | random string | Admin password (generated once if absent) |
| `WORLD_PASSWORD` | (empty) | Optional password required to join the world |
| `SERVER_GUID` | (empty) | Leave empty unless you know what you're doing |

### Example `.env`

Create `home-server/.env` (it is referenced by `docker-compose.yml` via `env_file`). Keep it out of version control — it holds your authkey and passwords.

```bash
# --- Required ---
# Your in-game "My Player Id" (bottom of the game's Settings menu, use the copy button).
# The server will NOT start without this.
OWNER_ID=your-in-game-player-id

# Tailscale authkey (https://login.tailscale.com/admin/settings/keys)
TS_AUTHKEY=tskey-auth-kxxxxx

# --- Networking ---
# Must match the Traefik relay port (45000/UDP in this project)
SERVER_PORT=45000

# --- Timezone ---
TZ=UTC

# --- Updates & backups ---
ENABLE_AUTO_UPDATE=true
UPDATE_TIME=3600
SERVER_STOP_TIMEOUT=120
BACKUP_AFTER_UPDATE=true
BACKUP_DAILY=true
BACKUP_TIME="3:00 AM"
BACKUP_RETENTION_DAYS=30
POLL_INTERVAL=60
IDLE_WAIT=360

# --- Discord notifications ---
ENABLE_DISCORD_NOTIF=false
DISCORD_WEBHOOK_URL=

# --- Logging ---
LOG_TO_STDOUT=true
MAX_LOG_SIZE=5242880
LOG_RETENTION=5

# --- Server settings (written to DedicatedServer.ini on start) ---
# Leave a value empty to keep whatever is already in the ini.
SERVER_NAME=My Dragonwilds Server
DEFAULT_WORLD_NAME=MyWorld
ADMIN_PASSWORD=change-me
WORLD_PASSWORD=
SERVER_GUID=
```

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

# View logs (game console + script messages)
docker compose logs -f

# Stop the server
docker compose down

# Restart a specific service
docker compose restart dragonwilds

# Follow the script log (updates, backups, idle-wait reasons)
docker exec dragonwilds tail -f /home/ubuntu/Steam/logs/entrypoint.log

# Follow the game server log
tail -f server/RSDragonwilds/Saved/Logs/RSDragonwilds.log
```

---

## Configuration

### 1. Set `OWNER_ID` (required)

`OwnerId` identifies your player as the server's owner — the only one who can ban/unban — and the [official docs](https://dragonwilds.runescape.com/news/how-to-dedicated-servers) state the server will not start without it. This is the single most common setup mistake.

**Option A — via `.env` (recommended):** add your in-game "My Player Id" to `home-server/.env`:

```
OWNER_ID=your-in-game-player-id
```

The container writes it into `DedicatedServer.ini` automatically on every start.

**Option B — edit the ini:** after the `server/` folder has been created (first `docker compose up`), stop the container and edit `server/RSDragonwilds/Saved/Config/LinuxServer/DedicatedServer.ini`, setting `OwnerId` to the value found under "My Player Id" in the game's settings menu.

> ⚠️ **The server will not start until `OwnerId` is set.**

### 2. Configure Tailscale Authkey

1. Go to [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys)
2. Create an auth key (reusable recommended)
3. Add to `home-server/.env`:
   ```
   TS_AUTHKEY=tskey-auth-kxxxxx
   ```

### 3. Configure Exit Node

In `home-server/docker-compose.yml`, update the exit node IP:

```yaml
command: >
  sh -c "tailscaled &
        sleep 5
        tailscale up --authkey=$TS_AUTHKEY --exit-node=100.x.x.x --accept-routes --operator=root
        tail -f /dev/null"
```

Replace `100.x.x.x` with your VPS Tailscale IP.

### 4. Configure Traefik

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

## How the Container Behaves

- **First run** installs the server via SteamCMD (anonymous login, up to 5 attempts with `validate`). Later starts do not re-verify files — the container compares local and remote Steam `buildid` every `UPDATE_TIME` seconds and only runs SteamCMD when they differ.
- **Updates and backups are idle-aware**: they wait until no players are online and the server has been idle for `IDLE_WAIT` seconds (6 min default). Nothing is skipped — the update check blocks and logs its reason every 60 seconds, and the backup loop retries every `POLL_INTERVAL` seconds until it can proceed.
- **Maintenance never kills the container**: before an update or backup the wrapper stops the server, does the work, then restarts the server *in the same container* (backing up and restoring `DedicatedServer.ini` around updates). Players just see a restart; the container stays up.
- **Missed backups catch up**: if the container was down when a daily backup was scheduled, the backup runs on the next start once the scheduled time has passed and the server is idle.
- **Hang guard**: if an update or backup never finishes, the wrapper force-restarts after 1 hour instead of hanging forever.
- **Crash handling**: if the game process exits on its own, the wrapper stops the container rather than restarting the game in place. With `restart: unless-stopped` (already set in `docker-compose.yml`), Docker brings the whole container back automatically.
- **Player state survives restarts**: online-player count and last-activity timestamp are stored under the mounted volume, so idle timers don't reset when the container restarts.

---

## Files & Logs

The volume mount is `./server:/home/ubuntu/Steam` (see `home-server/docker-compose.yml`).

| What | In container | On host (relative to `home-server/`) |
|------|--------------|---------|
| Server files / saves / backups | `/home/ubuntu/Steam` | `server/` |
| Saves | `/home/ubuntu/Steam/RSDragonwilds/Saved/SaveGames` | `server/RSDragonwilds/Saved/SaveGames` |
| Config | `/home/ubuntu/Steam/RSDragonwilds/Saved/Config/LinuxServer/DedicatedServer.ini` | `server/RSDragonwilds/Saved/Config/LinuxServer/DedicatedServer.ini` |
| Game server log | `/home/ubuntu/Steam/RSDragonwilds/Saved/Logs/RSDragonwilds.log` | `server/RSDragonwilds/Saved/Logs/RSDragonwilds.log` |
| Script (entrypoint) log | `/home/ubuntu/Steam/logs/entrypoint.log` | `server/logs/entrypoint.log` |
| Backups | `/home/ubuntu/Steam/backup/` | `server/backup/` |

**Container logs** — mixed script messages and game console output (`docker compose logs -f`).

**Script log** — dedicated file for the entrypoint script (updates, backups, idle-wait reasons, player events). It is appended across container restarts so update/backup history survives, prefixed with timestamps, and rotated by size — disk usage is bounded by `MAX_LOG_SIZE` × `LOG_RETENTION`.

**Game server log** — the running server's own output: `server/RSDragonwilds/Saved/Logs/RSDragonwilds.log`.

---

## Troubleshooting

### Server Won't Start

1. **Check `OwnerId` first** — per the official docs the server will not start without it. Verify `OWNER_ID` in `home-server/.env` (or `OwnerId` in the ini) matches your in-game "My Player Id" exactly, then restart the container.
2. Check for a SteamCMD install error in the logs — usually a disk space or permissions issue on the mounted volume:
   ```bash
   docker compose logs dragonwilds
   ```
3. Check if ports are in use:
   ```bash
   sudo netstat -tulpn | grep 45000
   ```
4. Verify the `.env` file exists:
   ```bash
   ls -la .env
   ```

### Updates or Backups Never Seem to Run

They only run once the server has been idle for `IDLE_WAIT` seconds (default 360) with no players online. Nothing is skipped — the update check blocks and logs its reason, and the daily backup loop retries — but if players stay online for a long time, that is why nothing has happened yet. Watch the script log to see what it's waiting for:

```bash
docker exec dragonwilds tail -f /home/ubuntu/Steam/logs/entrypoint.log
```

### My Server-Setting Changes Aren't Applied

Settings are applied from the environment on every container start, but an **empty** value deliberately preserves what's already in `DedicatedServer.ini`. To change a setting, put an actual value in `.env` and restart the container. If you edit the ini directly, do it while the container is stopped and make sure the matching environment variable is empty — non-empty env values re-override the ini on every start (and the game itself overwrites the file if you edit it while running).

### Tailscale Connection Issues

```bash
# Check Tailscale status
docker exec tailscale tailscale status

# Restart Tailscale container
docker compose restart tailscale

# View Tailscale logs
docker compose logs tailscale
```

### Players Can't Connect

1. Verify Traefik is running on VPS: `docker compose ps`
2. Check UDP port 45000 is open on VPS firewall
3. Verify Tailscale routes are accepted on both ends
4. Confirm exit node is configured on home server
5. Confirm `SERVER_PORT=45000` in `home-server/.env` matches the Traefik relay port — UDP ports are frequently missed in NAT/firewall rules that only forward TCP

### Container Exits Immediately After Starting

Check `docker compose logs dragonwilds` for a SteamCMD install error — this is usually a disk space or permissions issue on the mounted volume. If it happens after running for a while, the game process likely crashed; with `restart: unless-stopped` Docker will bring it back automatically, and `server/logs/entrypoint.log` will show why.

## Credits

Based on the tutorial by [Andy Druid](https://www.youtube.com/watch?v=M8jxPDxR4W4)

Container image: [`andyaltsys/dragonwilds-dedicated-server`](https://github.com/AltSystem42/runescape-dragonwilds-dedicated-server-docker)
