# Runescape Dragonwilds Server Files

Run your own Runescape: Dragonwilds server with Docker, Tailscale, and Traefik on a VPS.

## Requirements

- VPS with Ubuntu/Debian
- Docker & Docker Compose
- Tailscale (for remote access)
- Traefik (reverse proxy)

## Features

- Self-hosted Runescape Dragonwilds server
- Docker-based deployment
- Tailscale VPN integration
- Traefik reverse proxy setup

## YouTube Video Link

[Watch the setup guide here](https://www.youtube.com/watch?v=M8jxPDxR4W4)

## cmds to run

## tailscale install for vps
- sudo apt update && sudo apt upgrade -y
- ssh -i path/to/key.pem ubuntu@your-vps-public-ip
- sudo tailscale up --accept-routes --advertise-exit-node

## docker command to run to install docker & docker compose
- curl -fsSL https://get.docker.com | sh
- systemctl enable docker
- systemctl start docker
- docker --version
- sudo apt update && sudo apt install docker-compose-plugin
- docker compose version

## cmd to get tailscale ip
- tailscale ip -4
- In docker: docker exec tailscale tailscale ip -4

