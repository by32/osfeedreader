# FreshRSS Self‑Hosted Deployment on Oracle Cloud (Always‑Free Ampere A1)

> **Goal:** Run a resilient, cost‑free FreshRSS server with automatic TLS, backups, and self‑healing on Oracle OCI while staying inside the Always‑Free quota.

---

## 0 · High‑Level Architecture

```text
            ┌─────────────────────────────┐
Internet ⇄▶ │  Cloudflare Proxy (free)    │ ➊
            └────────────┬───────────────┘
                         │ 80 / 443
            ┌────────────▼───────────────┐
            │ Traefik (Docker)           │ ➋  TLS + ACME
            └────────────┬───────────────┘
                         │ internal 80
            ┌────────────▼───────────────┐
            │ FreshRSS (Docker)          │ ➌  feeds, API
            └────────────┬───────────────┘
                         │ bind‑mount
            ┌────────────▼───────────────┐
            │ 20 GB Block Volume         │ ➍  /srv/freshrss
            └─────────────────────────────┘
```

*Keep‑alive timer ➎ and backups ➏ run on the host OS.*

---

## 1 · Prerequisites

| Item                 | Details                                 |
| -------------------- | --------------------------------------- |
| Oracle Cloud account | Free‑tier enabled; default compartment. |
| SSH key pair         | Add public key during VM creation.      |
| Domain name          | Managed in Cloudflare (free plan).      |
| Local tools          | `ssh`, `scp`, and a code editor.        |

---

## 2 · Step‑by‑Step Deployment

### 2.1 Provision the VM

1. **Console → Compute → Instances → Create**
2. **Shape:** `VM.Standard.A1.Flex`, *1 OCPU* · *6 GB RAM*
3. **Boot volume:** 50 GB (defaults to free tier)
4. **Network:** keep default VCN; adjust Security List → add ingress *TCP 80 & 443*.
5. Upload **SSH public key**, then *Create*.

```bash
# First login
ssh ubuntu@<public‑ip>
sudo apt update && sudo apt upgrade -y
```

### 2.2 Create & mount data Block Volume

```bash
# Console → Storage → Block Volumes → Create (20 GB, same AD)
# Attach to instance (iSCSI)
export IQN=iqn.2025-07.com.oracle:...  # from console
sudo iscsiadm -m node -T $IQN -p <target-ip>:3260 --login
sudo mkfs.ext4 /dev/oracleoci/oraclevdb
sudo mkdir -p /srv/freshrss
echo '/dev/oracleoci/oraclevdb /srv/freshrss ext4 defaults,_netdev 0 2' | sudo tee -a /etc/fstab
sudo mount -a
```

### 2.3 Install host packages

```bash
sudo apt install -y docker.io docker-compose-plugin stress-ng iscsi-initiator-utils \
                    oci-cli restic curl ufw fail2ban
sudo usermod -aG docker $USER && newgrp docker
sudo ufw allow 22 80 443/tcp && sudo ufw --force enable
```

### 2.4 Deploy FreshRSS + Traefik (Docker Compose)

Create `/opt/freshrss/docker-compose.yml`:

```yaml
version: "3.9"
services:
  freshrss:
    image: freshrss/freshrss:latest
    restart: unless-stopped
    environment:
      TZ: America/New_York
      CRON_MIN: "*/20"
    volumes:
      - /srv/freshrss/data:/var/www/FreshRSS/data
      - /srv/freshrss/ext:/var/www/FreshRSS/extensions
    networks: [web]

  traefik:
    image: traefik:v3.0
    command:
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.le.acme.tlschallenge=true"
      - "--certificatesresolvers.le.acme.email=you@example.com"
      - "--certificatesresolvers.le.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_letsencrypt:/letsencrypt
    networks: [web]

volumes:
  traefik_letsencrypt:

networks:
  web:
```

Launch:

```bash
cd /opt/freshrss
docker compose up -d
```

### 2.5 DNS & HTTPS

1. **Cloudflare → DNS**: `A  rss.example.com  <public‑ip>` (orange‑cloud ON).
2. Browse to `https://rss.example.com` → FreshRSS wizard → create admin user & enable *Google Reader API*.

### 2.6 Keep‑Alive timer (avoid idle reclaim)

```bash
cat >/usr/local/bin/oci-keepalive.sh <<'EOF'
#!/usr/bin/env bash
stress-ng --cpu 1 --vm 1 --vm-bytes 1G --timeout 600s --metrics-brief
curl -s https://speed.cloudflare.com/__down?bytes=50000000 -o /dev/null
EOF
sudo chmod +x /usr/local/bin/oci-keepalive.sh
```

`/etc/systemd/system/oci-keepalive.{service,timer}`:

```ini
# service
[Unit]
Description=OCI keep-alive burst
[Service]
Type=oneshot
ExecStart=/usr/local/bin/oci-keepalive.sh

# timer
[Unit]
Description=Run keep-alive every 90 min
[Timer]
OnBootSec=15min
OnUnitActiveSec=90min
AccuracySec=1min
[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now oci-keepalive.timer
```

### 2.7 Automatic container updates

```bash
docker run -d --name watchtower \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower --cleanup --schedule "0 3 * * *"
```

### 2.8 Block‑volume snapshots (3‑day retention)

```bash
cat >/usr/local/bin/oci-snap.sh <<'EOF'
#!/usr/bin/env bash
BV_NAME=freshrss-data
COMP_ID=<compartment-ocid>
VOL=$(oci bv volume list --compartment-id $COMP_ID \
      --display-name $BV_NAME --query 'data[0].id' --raw-output)
oci bv backup create --volume-id $VOL \
     --display-name "$BV_NAME-$(date +%F)"
oci bv backup list --volume-id $VOL --query \
  'data | sort_by(@,&"time-created") | reverse(@)[3:][].id' \
  --raw-output | xargs -r -n1 oci bv backup delete --force
EOF
sudo chmod +x /usr/local/bin/oci-snap.sh
(crontab -l; echo "30 2 * * * /usr/local/bin/oci-snap.sh") | crontab -
```

### 2.9 Off‑site restic backup (optional)

```bash
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export RESTIC_REPOSITORY=s3:s3.us-east-1.wasabisys.com/freshrss
export RESTIC_PASSWORD=<pw>
restic init
cat >/usr/local/bin/restic-backup.sh <<'EOF'
#!/usr/bin/env bash
restic backup /srv/freshrss/data --tag cron
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 3 --prune
EOF
sudo chmod +x /usr/local/bin/restic-backup.sh
(crontab -l; echo "0 3 * * * /usr/local/bin/restic-backup.sh") | crontab -
```

### 2.10 (Advanced) Event‑Driven Auto‑Recreate

- Create **Instance Configuration** from running VM.
- **OCI Events Rule:** trigger on `com.oraclecloud.computeapi.instance.terminate.end` for this instance OCID.
- **OCI Function** (Python) that:
  1. launches a new VM from the saved configuration,
  2. waits for it to boot,
  3. attaches the `freshrss` Block Volume,
  4. executes a cloud‑init script (`docker compose up -d`).
- Cost: inside Always‑Free Functions quota.

---

## 3 · Validation Checklist

-

---

## 4 · Restoration Recipe

1. Launch new A1 VM (same AD).
2. Attach existing `freshrss` Block Volume (iSCSI).
3. Mount at `/srv/freshrss` (fstab entry).
4. Re‑deploy docker‑compose stack.
5. Update DNS if IP changed.

**Downtime target:** < 15 minutes.

---

## 5 · Cost Summary (July 2025)

| Item                   | Monthly      | Notes                |
| ---------------------- | ------------ | -------------------- |
| A1 VM (1 OCPU / 6 GB)  | \$0          | Always‑Free quota    |
| 20 GB Block Volume     | \$0          | First 200 GB free    |
| 3 Snapshots × 20 GB    | \$0          | First 5 backups free |
| Wasabi off‑site (1 GB) | \$0.01       | Optional             |
| **Total**              | **\$0–0.01** |                      |

---

## 6 · Useful Links

- Oracle OCI *Always‑Free* docs  ↗ [https://bit.ly/oci-free-tier](https://bit.ly/oci-free-tier)
- FreshRSS Docker image          ↗ [https://hub.docker.com/r/freshrss/freshrss](https://hub.docker.com/r/freshrss/freshrss)
- Traefik v3 docs                ↗ [https://doc.traefik.io/traefik](https://doc.traefik.io/traefik)
- Restic manual                  ↗ [https://restic.readthedocs.io](https://restic.readthedocs.io)

---

> **Next:** run through the steps in a new tenancy and verify each box in the Validation Checklist.  Happy self‑hosting!  🚀

