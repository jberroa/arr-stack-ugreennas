# Updating the Stack

Already running an earlier version? There are two types of updates:

1. **Stack updates** — New features, bug fixes, compose changes from this repo
2. **Container image updates** — Newer versions of Sonarr, Radarr, Jellyfin, etc.

## Stack Updates (this repo)

SSH into your NAS and pull the latest changes:

```bash
ssh your-username@nas-ip
cd /volume1/docker/arr-stack  # or your deployment path

git pull origin main
docker compose -f docker-compose.arr-stack.yml up -d --force-recreate  # Updates AND restarts - no further steps needed
```

The `--force-recreate` flag ensures containers restart with new config even if the image hasn't changed.

## Container Image Updates (Sonarr, Jellyfin, etc.)

To pull the latest Docker images and restart with them:

```bash
docker compose -f docker-compose.arr-stack.yml pull
docker compose -f docker-compose.arr-stack.yml up -d  # Restarts containers with new images - no further steps needed
```

> **Note:** Docker named volumes persist across restarts. All your service configurations (Sonarr settings, API keys, library data, etc.) are preserved.

---

## Migration Notes

When upgrading across versions, check below for any action required.

### v1.1 → v1.2.x

**Breaking changes:** None

**Automatic improvements** (just redeploy to get these):
- Startup order fixes — Gluetun now waits for Pi-hole to be healthy before connecting
- Improved healthchecks — FlareSolverr actually tests Chrome, catches crashes
- Backup script improvements — smart space checking, 7-day rotation
- SABnzbd added — Usenet downloads via VPN (remove from compose if not wanted); configure in [SETUP.md](SETUP.md#usenet-sabnzbd)

**New features (optional, requires setup):**

| Feature | What it does | Setup |
|---------|--------------|-------|
| `.lan` domains | `http://sonarr.lan` etc, no ports | Router DHCP reservation + Pi-hole DNS, see [SETUP.md](SETUP.md#511-local-dns-lan-domains--optional) |
| `MEDIA_ROOT` env var | Configurable media path | Add to `.env` if not using `/volume1/Media` |
| deunhealth | Auto-restart crashed services | Deploy `docker-compose.utilities.yml` |

**New .env variables:**

| Variable | Required | Default | Purpose |
|----------|----------|---------|---------|
| `MEDIA_ROOT` | No | `/volume1/Media` | Base path for media storage |
| `TRAEFIK_LAN_IP` | Only for .lan | — | Traefik's dedicated LAN IP for local DNS |
| `LAN_INTERFACE` | Only for .lan | — | Network interface (e.g., `eth0`) |
| `LAN_SUBNET` | Only for .lan | — | Your LAN subnet (e.g., `10.10.0.0/24`) |
| `LAN_GATEWAY` | Only for .lan | — | Router IP |
| `TRAEFIK_LAN_MAC` | Only for .lan | — | Fixed MAC for DHCP reservation |

See [.env.example](../.env.example) for all available variables.
