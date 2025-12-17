# Backup & Restore

This stack uses Docker named volumes for service data. This guide covers backing up and restoring your configuration.

## What Gets Backed Up

The backup script (`scripts/backup-volumes.sh`) backs up **essential configs only** - small files that are hard to recreate:

| Volume | Size | Contents |
|--------|------|----------|
| gluetun-config | ~7MB | VPN provider settings |
| qbittorrent-config | ~9MB | Client settings, categories |
| prowlarr-config | ~22MB | Indexer configs, API keys |
| bazarr-config | ~2MB | Subtitle provider credentials |
| wireguard-easy-config | ~8KB | VPN peer configs (critical!) |
| uptime-kuma-data | ~14MB | Monitor configurations |
| pihole-etc-dnsmasq | ~4KB | Custom DNS settings |
| jellyseerr-config | ~5MB | User accounts, requests |

**Total: ~60MB uncompressed, ~13MB compressed**

## What's NOT Backed Up

Large volumes that regenerate automatically are excluded:

| Volume | Size | Why Excluded |
|--------|------|--------------|
| jellyfin-config | ~407MB | Re-scan library to rebuild metadata |
| sonarr-config | ~43MB | Re-scan library to rebuild |
| radarr-config | ~110MB | Re-scan library to rebuild |
| pihole-etc-pihole | ~138MB | Blocklists auto-download on startup |
| jellyfin-cache | ~43MB | Transcoding cache, fully regenerates |
| duc-index | ~20MB | Disk usage index, regenerates on restart |

> **Note:** If you want to preserve watch history (Jellyfin) or avoid re-scanning, you can manually backup these volumes using the same docker command shown below.

---

## Running a Backup

### On the NAS

```bash
cd /volume1/docker/arr-stack
./scripts/backup-volumes.sh --tar
```

Output:
```
=== Arr-Stack Backup ===
Volume prefix: arr-stack_*
Backup dir:    /tmp/arr-stack-backup-20241217

Backing up gluetun-config... OK (7.1M)
Backing up qbittorrent-config... OK (8.9M)
...
Summary: 8 backed up, 0 skipped, 0 failed
Total size: 58M

Created: /tmp/arr-stack-backup-20241217.tar.gz (13M)
```

### Copying Off-NAS

**Ugreen NAS** (scp doesn't work with /tmp):
```bash
ssh user@nas "cat /tmp/arr-stack-backup-*.tar.gz" > ./backup.tar.gz
```

**Other systems** (Synology, QNAP, Linux):
```bash
scp user@nas:/tmp/arr-stack-backup-*.tar.gz ./backup.tar.gz
```

> **Important:** Backups in `/tmp` are cleared on reboot. Copy off-NAS promptly!

---

## Restore

### Full Restore (New Installation)

1. Deploy the stack normally (see [Setup Guide](SETUP.md))
2. Stop the services:
   ```bash
   docker compose -f docker-compose.arr-stack.yml down
   ```
3. Extract backup and restore each volume:
   ```bash
   tar -xzf backup.tar.gz
   cd arr-stack-backup-20241217

   for dir in */; do
     vol="arr-stack_${dir%/}"
     echo "Restoring $vol..."
     docker run --rm \
       -v "$(pwd)/$dir":/source:ro \
       -v "$vol":/dest \
       alpine cp -a /source/. /dest/
   done
   ```
4. Start services:
   ```bash
   docker compose -f docker-compose.arr-stack.yml up -d
   ```

### Single Volume Restore

```bash
# Example: restore jellyseerr config
docker compose -f docker-compose.arr-stack.yml stop jellyseerr

docker run --rm \
  -v ./backup/jellyseerr-config:/source:ro \
  -v arr-stack_jellyseerr-config:/dest \
  alpine cp -a /source/. /dest/

docker compose -f docker-compose.arr-stack.yml start jellyseerr
```

---

## Script Options

```bash
./scripts/backup-volumes.sh [OPTIONS] [BACKUP_DIR]

Options:
  --tar           Create .tar.gz archive (recommended)
  --prefix NAME   Override volume prefix (default: auto-detect)

Examples:
  ./scripts/backup-volumes.sh --tar                    # Default location
  ./scripts/backup-volumes.sh --tar /path/to/backup    # Custom location
  ./scripts/backup-volumes.sh --prefix media-stack     # Custom prefix
```

### Volume Prefix Auto-Detection

The script auto-detects your volume prefix from running containers. If you cloned the repo to a different directory (e.g., `media-stack` instead of `arr-stack`), it will detect this automatically.

If auto-detection fails, use `--prefix`:
```bash
./scripts/backup-volumes.sh --tar --prefix media-stack
```

### Jellyfin vs Plex

The script auto-detects which variant you're using and backs up the appropriate request manager:
- Jellyfin stack: `jellyseerr-config`
- Plex stack: `overseerr-config`

---

## Backup Schedule

Consider setting up automated backups with cron:

```bash
# Weekly backup, keep for 30 days
0 3 * * 0 /volume1/docker/arr-stack/scripts/backup-volumes.sh --tar /volume1/backups/arr-stack-$(date +\%Y\%m\%d) && find /volume1/backups -name "arr-stack-*.tar.gz" -mtime +30 -delete
```

---

## Troubleshooting

### "Permission denied" errors
The backup script runs docker containers which handle permissions internally. If you see permission errors, ensure:
- You're in the docker group: `groups` should show `docker`
- Docker daemon is running: `docker ps`

### "Volume not found"
- Ensure services have been started at least once (volumes are created on first run)
- Check the volume prefix matches: `docker volume ls | grep config`

### Backup too large
If backup exceeds ~60MB, check if optional volumes were accidentally included. The script only backs up essential configs by default.
