# TrueNAS SCALE + Portainer + Cloudflare R2 Backup Workflow

## Overview

This setup provides:

* ZFS snapshot protection
* Automatic Docker appdata backup
* PostgreSQL dump backup
* Offsite backup to Cloudflare R2
* Automatic snapshot cleanup
* Disaster recovery workflow

---

# Storage Structure

```text
/mnt/Theh_1/docker-data/
├── mattermost/
├── wordpress/
├── postgres/
├── compose/
├── backups/
└── ...
```

---

# Important Notes

## Docker Runtime

TrueNAS SCALE internal Docker runtime:

```text
Theh_1/ix-apps/docker
```

is NOT backed up directly.

Only persistent app data is backed up.

---

## Excluded Files

Large VM disk image excluded:

```text
vdsm/
```

Reason:

* VDSM VM only accesses SMB shares
* VM image itself is disposable
* Reduces backup size significantly

---

# Cloudflare R2 Setup

## Install rclone

```bash
apt update
apt install rclone -y
```

---

## Configure R2

```bash
rclone config
```

Create remote:

```text
name: r2
storage: s3
provider: Cloudflare
region: auto
endpoint:
https://ACCOUNT_ID.r2.cloudflarestorage.com
```

---

## Test Connection

```bash
rclone ls r2:hvfx-dockerbackup
```

---

# Dataset Setup

## Create dataset

```bash
zfs create Theh_1/docker-data
```

---

## Move existing data

```bash
rsync -avh --progress /mnt/Theh_1/docker_old/ /mnt/Theh_1/docker-data/
```

---

## Symlink for compatibility

```bash
ln -s /mnt/Theh_1/docker-data /mnt/Theh_1/docker
```

---

# Backup Script

Create:

```bash
nano /root/docker-backup.sh
```

Contents:

```bash
#!/bin/bash

DATE=$(date +%Y-%m-%d_%H-%M)

SOURCE="/mnt/Theh_1/docker-data"
BACKUP_DIR="${SOURCE}/backups"

mkdir -p ${BACKUP_DIR}

echo "[1] postgres dump"

POSTGRES=$(docker ps --format '{{.Names}}' | grep -Ei 'postgres|postgre' | head -n 1)

if [ ! -z "$POSTGRES" ]; then
    docker exec $POSTGRES pg_dumpall -U postgres > ${BACKUP_DIR}/postgres-${DATE}.sql
fi

echo "[2] zfs snapshot"

zfs snapshot Theh_1/docker-data@${DATE}

echo "[3] compress"

tar --zstd -cf ${BACKUP_DIR}/dockerdata-${DATE}.tar.zst \
    --exclude='backups' \
    --exclude='vdsm/' \
    ${SOURCE}

echo "[4] upload to cloudflare r2"

rclone copy ${BACKUP_DIR} r2:hvfx-dockerbackup/daily \
    --transfers 4 \
    --checkers 8 \
    --fast-list \
    --progress

echo "[5] cleanup local backups older than 7 days"

find ${BACKUP_DIR} -type f -mtime +7 -delete

echo "done"
```

---

## Make executable

```bash
chmod +x /root/docker-backup.sh
```

---

# Snapshot Cleanup Script

Create:

```bash
nano /root/zfs-cleanup.sh
```

Contents:

```bash
#!/bin/bash

zfs list -H -o name -t snapshot | grep 'Theh_1/docker-data@' | while read snap
do
    creation=$(zfs get -H -o value creation "$snap")
    snap_epoch=$(date -d "$creation" +%s)
    now_epoch=$(date +%s)

    age_days=$(( (now_epoch - snap_epoch) / 86400 ))

    if [ $age_days -gt 30 ]; then
        zfs destroy -r "$snap"
    fi
done
```

---

## Make executable

```bash
chmod +x /root/zfs-cleanup.sh
```

---

# Cron Automation

Edit crontab:

```bash
crontab -e
```

Add:

```cron
0 3 * * * /root/docker-backup.sh >> /var/log/docker-backup.log 2>&1
30 3 * * * /root/zfs-cleanup.sh >> /var/log/zfs-cleanup.log 2>&1
```

---

# Manual Backup Test

```bash
/root/docker-backup.sh
```

---

# Verify Upload

```bash
rclone ls r2:hvfx-dockerbackup/daily
```

---

# Recovery Guide

# 1. Snapshot Rollback

List snapshots:

```bash
zfs list -t snapshot
```

Rollback:

```bash
zfs rollback -r Theh_1/docker-data@SNAPSHOT_NAME
```

---

# 2. Full Restore from R2

## Create dataset

```bash
zfs create Theh_1/docker-data
```

---

## Download backup

```bash
mkdir /restore

rclone copy r2:hvfx-dockerbackup/daily /restore
```

---

## Extract backup

```bash
tar --zstd -xf /restore/dockerdata-YYYY-MM-DD.tar.zst -C /
```

---

## Restore PostgreSQL

```bash
docker exec -i postgres_container psql -U postgres < postgres-YYYY-MM-DD.sql
```

---

## Restart containers

```bash
docker compose up -d
```

or redeploy through Portainer.

---

# Recommended Future Improvements

* Encrypt backups using rclone crypt
* Store compose files separately
* Add automatic restore script
* Multi-region R2 replication
* Secondary NAS replication

---
