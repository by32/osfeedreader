# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an infrastructure deployment repository for FreshRSS (RSS feed reader) running on Oracle Cloud Infrastructure's Always-Free tier. The project focuses on deployment automation and infrastructure management rather than application code.

## Architecture

The deployment uses a containerized architecture with:
- **FreshRSS**: RSS feed reader application (Docker container)
- **Traefik**: Reverse proxy with automatic HTTPS/TLS (Docker container)  
- **Oracle Block Volume**: Persistent storage for FreshRSS data
- **Cloudflare**: DNS and CDN proxy
- **Systemd timers**: Keep-alive services to prevent idle VM reclaim

## Key Files

- `fresh_rss_oracle_deployment.md`: Complete deployment guide and documentation
- `LICENSE`: MIT license
- No source code files - this is a deployment/infrastructure repository

## Deployment Commands

Since this is an infrastructure project, common operations involve:

### Docker Management
```bash
cd /opt/freshrss
docker compose up -d          # Start services
docker compose down           # Stop services
docker compose logs -f        # View logs
```

Configuration file: `/opt/freshrss/docker-compose.yml`

### System Management
```bash
sudo systemctl status oci-keepalive.timer    # Check keep-alive timer
sudo systemctl restart oci-keepalive.timer   # Restart timer
sudo systemctl daemon-reload                 # Reload systemd after config changes
```

Systemd service files: `/etc/systemd/system/oci-keepalive.{service,timer}`

### Backup Operations
```bash
/usr/local/bin/oci-snap.sh                   # Manual OCI snapshot
/usr/local/bin/restic-backup.sh              # Manual restic backup
```

### OCI CLI Operations
```bash
# For snapshot script, requires setting compartment OCID:
export COMP_ID=<compartment-ocid>
oci bv volume list --compartment-id $COMP_ID --display-name freshrss-data
```

## Infrastructure Components

- **VM**: Oracle Cloud A1 instance (1 OCPU, 6GB RAM)
- **Storage**: 20GB Block Volume mounted at `/srv/freshrss`
- **Networking**: Traefik handles TLS termination on ports 80/443
- **Monitoring**: Keep-alive scripts prevent idle instance reclaim
- **Backups**: Both OCI snapshots and optional off-site restic backups

## Important Notes

- This repository contains infrastructure deployment documentation, not application source code
- The actual FreshRSS application runs from the official Docker image
- All configurations are deployment-specific and designed for Oracle Cloud's Always-Free tier
- Focus is on cost-free, resilient self-hosting with automatic recovery capabilities