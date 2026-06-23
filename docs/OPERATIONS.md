# HR Leave Management — Operations Manual  

> **Version**: 1.0  
> **Audience**: System administrators and DevOps engineers  
> **References**: [SPEC.md](./SPEC.md) · [API-IMPL-SPEC.md](./API-IMPL-SPEC.md)  

---

## Table of Contents  

1. [System Architecture](#1-system-architecture)  
2. [Production Deployment](#2-production-deployment)  
3. [Backup & Restore](#3-backup--restore)  
4. [Monitoring & Alerting](#4-monitoring--alerting)  
5. [Database Maintenance](#5-database-maintenance)  
6. [Certificate & Key Rotation](#6-certificate--key-rotation)  
7. [Software Updates](#7-software-updates)  
8. [Disaster Recovery](#8-disaster-recovery)  
9. [Troubleshooting](#9-troubleshooting)  
10. [Retention & Cleanup](#10-retention--cleanup)  
11. [Security Hardening Checklist](#11-security-hardening-checklist)  
12. [Runbooks](#12-runbooks)  

---

## 1. System Architecture  

### 1.1 Production Topology  

```
┌─────────────────────────────────────────────────────────────┐
│  Production Server (Docker Compose)                         │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │  Nginx   │  │  Quarkus │  │  Vue SPA │  │  Valkey   │  │
│  │  :443    │  │  :8080   │  │  :3000   │  │  :6379    │  │
│  └──────────┘  └──────────┘  └──────────┘  └───────────┘  │
│                                                             │
│  ┌──────────┐  ┌──────────┐                                │
│  │PostgreSQL│  │ RabbitMQ │                                │
│  │  :5432   │  │ :5672    │                                │
│  └────┬─────┘  └──────────┘                                │
│       │                                                     │
│  ┌────┴────────────────────────────────────────────┐       │
│  │  Volumes                                        │       │
│  │  /data/pgdata     — PostgreSQL data (encrypted) │       │
│  │  /data/documents  — Leave documents (encrypted) │       │
│  │  /data/backups    — Local backups (encrypted)   │       │
│  │  /data/keys       — JWT keys, GPG keys (ro)     │       │
│  └─────────────────────────────────────────────────┘       │
│                                                             │
│  ┌──────────────────────────────────────────────────┐      │
│  │  Cron (systemd timer)                            │      │
│  │  • Nightly backup → /data/backups                │      │
│  │  • Weekly remote sync → backup.example.com       │      │
│  │  • Weekly DB vacuum                              │      │
│  │  • Monthly retention cleanup                     │      │
│  └──────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
         │
         │ rsync over SSH (weekly)
         ▼
┌─────────────────────────────────────────────────────────────┐
│  Remote Backup Server (off-site)                            │
│  /backups/hr-leave/                                         │
│    ├── db/           — Encrypted DB dumps                   │
│    ├── files/        — Encrypted document tarballs          │
│    └── retention/    — Monthly/yearly long-term archives    │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow  

| Path | Protocol | Encryption | Purpose |
|------|----------|------------|---------|
| User → Nginx | HTTPS (TLS 1.3) | In transit | All user traffic |
| Nginx → API | HTTP (internal Docker network) | None (container network) | Reverse proxy |
| API → PostgreSQL | TCP (internal Docker network) | None (container network) | Database queries |
| API → Valkey | TCP (internal Docker network) | None (container network) | Cache |
| API → RabbitMQ | AMQP (internal Docker network) | None (container network) | Events |
| API → Filesystem | Local path | AES-256-GCM | Document storage |
| Backup → Remote | SSH + rsync | In transit (SSH) | Off-site replication |
| Backup files | GPG (at rest) | AES-256 | Backup encryption |

---

## 2. Production Deployment  

### 2.1 Prerequisites  

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| OS | Ubuntu 24.04 LTS | Same |
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 100 GB SSD | 200 GB SSD (for backups + growth) |
| Docker | 27.x+ | Latest stable |
| Docker Compose | v2.x (plugin) | Latest stable |
| SSH access | To remote backup server | Key-based auth only |
| GPG | 2.4+ | Latest stable |

### 2.2 Directory Setup  

Run these commands once on the production server:

```bash
#!/bin/bash
# Run as root or user with sudo

# Create directory structure
mkdir -p /opt/hr-leave-management
mkdir -p /data/{pgdata,documents,backups/{db,files},keys}
mkdir -p /etc/hr-leave-management
mkdir -p /var/log/hr-leave-management

# Set restrictive permissions
chmod 750 /data/{pgdata,documents,backups,keys}
chmod 700 /data/backups  # Backup encryption keys live here
chown -R 1000:1000 /data/pgdata      # PostgreSQL UID in container
chown -R 1000:1000 /data/documents   # API UID in container

# Create backup user (for remote rsync)
useradd -r -s /bin/bash -m hr-backup
mkdir -p /home/hr-backup/.ssh
chmod 700 /home/hr-backup/.ssh
```

### 2.3 LUKS Encryption (Disk-Level)  

The `/data` volume must be encrypted at rest. Use LUKS:

```bash
# One-time setup (destroys existing data!)
cryptsetup luksFormat /dev/sdb             # Use your data disk device
cryptsetup luksOpen /dev/sdb data-crypt
mkfs.ext4 /dev/mapper/data-crypt
mount /dev/mapper/data-crypt /data

# Add to /etc/crypttab for auto-mount at boot
echo "data-crypt /dev/sdb none luks" >> /etc/crypttab

# Add to /etc/fstab
echo "/dev/mapper/data-crypt /data ext4 defaults 0 2" >> /etc/fstab
```

The LUKS passphrase is required at boot. Store it securely (password manager, not on the server).

### 2.4 Docker Compose — Production  

`/opt/hr-leave-management/docker-compose.yml`:

```yaml
version: "3.9"

services:
  # ── Reverse Proxy ──────────────────────────────────
  nginx:
    image: nginx:1.27-alpine
    container_name: hr-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /etc/hr-leave-management/nginx.conf:/etc/nginx/nginx.conf:ro
      - /etc/hr-leave-management/certs:/etc/nginx/certs:ro
      - /etc/hr-leave-management/dhparam.pem:/etc/nginx/dhparam.pem:ro
    networks:
      - frontend
    depends_on:
      - api
      - frontend
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ── Frontend (Vue SPA) ─────────────────────────────
  frontend:
    image: registry.example.com/hr-leave-management/frontend:${APP_VERSION:-latest}
    container_name: hr-frontend
    restart: unless-stopped
    networks:
      - frontend
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ── API (Quarkus) ──────────────────────────────────
  api:
    image: registry.example.com/hr-leave-management/api:${APP_VERSION:-latest}
    container_name: hr-api
    restart: unless-stopped
    environment:
      # Database
      QUARKUS_DATASOURCE_JDBC_URL: jdbc:postgresql://postgres:5432/hr_leave
      QUARKUS_DATASOURCE_USERNAME: ${DB_USERNAME}
      QUARKUS_DATASOURCE_PASSWORD: ${DB_PASSWORD}
      # Valkey
      QUARKUS_REDIS_HOSTS: redis://valkey:6379
      # RabbitMQ
      QUARKUS_MESSAGING_RABBITMQ_URL: amqp://${RABBITMQ_USER}:${RABBITMQ_PASSWORD}@rabbitmq:5672
      # JWT
      JWT_PRIVATE_KEY_PATH: /etc/keys/privateKey.pem
      JWT_PUBLIC_KEY_PATH: /etc/keys/publicKey.pem
      # Encryption
      ENCRYPTION_KEY: ${ENCRYPTION_KEY}
      # URLs
      FRONTEND_URL: ${FRONTEND_URL}
      APP_DOMAIN: ${APP_DOMAIN}
      # SMTP
      QUARKUS_MAILER_HOST: ${SMTP_HOST}
      QUARKUS_MAILER_PORT: ${SMTP_PORT:-587}
      QUARKUS_MAILER_USERNAME: ${SMTP_USERNAME}
      QUARKUS_MAILER_PASSWORD: ${SMTP_PASSWORD}
      # SSO
      GOOGLE_CLIENT_ID: ${GOOGLE_CLIENT_ID}
      GOOGLE_CLIENT_SECRET: ${GOOGLE_CLIENT_SECRET}
      ZOHO_CLIENT_ID: ${ZOHO_CLIENT_ID}
      ZOHO_CLIENT_SECRET: ${ZOHO_CLIENT_SECRET}
      # Logging
      QUARKUS_LOG_LEVEL: ${LOG_LEVEL:-INFO}
      JAVA_OPTS: "-Xmx2g -Xms512m"
    volumes:
      - /data/documents:/var/lib/hr-leave/documents
      - /data/keys:/etc/keys:ro
    networks:
      - backend
      - frontend
    depends_on:
      postgres:
        condition: service_healthy
      valkey:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/q/health/ready"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"

  # ── PostgreSQL ─────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    container_name: hr-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: hr_leave
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_INITDB_ARGS: "--data-checksums"
    volumes:
      - /data/pgdata:/var/lib/postgresql/data
      - /etc/hr-leave-management/postgresql.conf:/etc/postgresql/postgresql.conf:ro
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USERNAME} -d hr_leave"]
      interval: 10s
      timeout: 5s
      retries: 5
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ── Valkey ─────────────────────────────────────────
  valkey:
    image: valkey/valkey:8-alpine
    container_name: hr-valkey
    restart: unless-stopped
    command: valkey-server --save "" --appendonly no --maxmemory 256mb --maxmemory-policy allkeys-lru
    networks:
      - backend
    healthcheck:
      test: ["CMD", "valkey-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ── RabbitMQ ───────────────────────────────────────
  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    container_name: hr-rabbitmq
    restart: unless-stopped
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - backend
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access to DB/Valkey/RabbitMQ

volumes:
  rabbitmq_data:
```

### 2.5 Environment File  

`/etc/hr-leave-management/.env` (permissions: `600`, owned by root):

```bash
# ── Required Secrets ──────────────────────────────────
DB_USERNAME=hr_app
DB_PASSWORD=<generate: openssl rand -base64 32>
RABBITMQ_USER=hr_app
RABBITMQ_PASSWORD=<generate: openssl rand -base64 32>
ENCRYPTION_KEY=<generate: openssl rand -base64 32>

# ── URLs ─────────────────────────────────────────────
FRONTEND_URL=https://leave.company.com
APP_DOMAIN=leave.company.com

# ── SSO ──────────────────────────────────────────────
GOOGLE_CLIENT_ID=123456789-xxxxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=<from Google Cloud Console>
ZOHO_CLIENT_ID=<from Zoho API Console>
ZOHO_CLIENT_SECRET=<from Zoho API Console>

# ── SMTP ─────────────────────────────────────────────
SMTP_HOST=smtp.company.com
SMTP_PORT=587
SMTP_USERNAME=noreply@company.com
SMTP_PASSWORD=<smtp-password>

# ── Version ──────────────────────────────────────────
APP_VERSION=1.0.0

# ── Optional ─────────────────────────────────────────
LOG_LEVEL=INFO
```

### 2.6 Startup  

```bash
# Pull images
cd /opt/hr-leave-management
docker compose pull

# Start all services
docker compose --env-file /etc/hr-leave-management/.env up -d

# Verify
docker compose ps                    # All services should be "Up" and "healthy"
curl -f https://localhost/q/health/ready  # Should return 200
```

### 2.7 Shutdown  

```bash
# Graceful shutdown (waits for in-flight requests)
docker compose stop

# Full stop + remove containers (volumes preserved)
docker compose down
```

---

## 3. Backup & Restore  

### 3.1 GPG Encryption Setup  

Backup files must be encrypted at rest before leaving the server. Generate a GPG key pair:

```bash
# Generate on the production server (one-time)
gpg --batch --generate-key <<EOF
Key-Type: RSA
Key-Length: 4096
Name-Real: HR Leave Backup
Name-Email: backup@company.com
Expire-Date: 0
Passphrase: $(openssl rand -base64 32)
%commit
EOF

# Export public key (for verification — never leaves the server)
gpg --export -a "HR Leave Backup" > /data/keys/backup-public.asc

# Store the passphrase in /data/keys/backup-passphrase.txt (chmod 600)
# Also store in company password manager for disaster recovery
```

### 3.2 Backup Script  

`/opt/hr-leave-management/scripts/backup.sh`:

```bash
#!/bin/bash
set -euo pipefail

# ── Configuration ────────────────────────────────────
DATE=$(date -u +%Y-%m-%d)
BACKUP_BASE="/data/backups"
DB_BACKUP_DIR="${BACKUP_BASE}/db"
FILES_BACKUP_DIR="${BACKUP_BASE}/files"
DOCS_DIR="/data/documents"
DB_NAME="hr_leave"
DB_USER="${DB_USERNAME:-hr_app}"
GPG_RECIPIENT="HR Leave Backup"
GPG_PASSPHRASE_FILE="/data/keys/backup-passphrase.txt"
LOG_FILE="/var/log/hr-leave-management/backup.log"
RETENTION_DAYS=7

# ── Logging ──────────────────────────────────────────
exec 1> >(tee -a "${LOG_FILE}")
exec 2>&1
echo "=== Backup started: $(date -u -Iseconds) ==="

# ── 1. Database backup ───────────────────────────────
echo "[1/5] Backing up PostgreSQL..."
DB_FILE="${DB_BACKUP_DIR}/hr_leave_${DATE}.dump"

# pg_dump inside a transaction for consistency
docker exec hr-postgres pg_dump \
    --format=custom \
    --compress=9 \
    --no-owner \
    --no-acl \
    --username="${DB_USER}" \
    "${DB_NAME}" > "${DB_FILE}.tmp"

# Move atomically (no partial files)
mv "${DB_FILE}.tmp" "${DB_FILE}"

# Verify the dump is readable
docker exec -i hr-postgres pg_restore --list "${DB_FILE}" > /dev/null 2>&1 || {
    echo "FATAL: Database dump verification failed"
    rm -f "${DB_FILE}"
    exit 1
}
echo "   DB dump: $(du -h "${DB_FILE}" | cut -f1)"

# ── 2. Files backup ──────────────────────────────────
echo "[2/5] Backing up documents..."
FILES_TAR="${FILES_BACKUP_DIR}/documents_${DATE}.tar.gz"

tar -czf "${FILES_TAR}.tmp" -C "$(dirname "${DOCS_DIR}")" "$(basename "${DOCS_DIR}")"
mv "${FILES_TAR}.tmp" "${FILES_TAR}"
echo "   Files: $(du -h "${FILES_TAR}" | cut -f1)"

# ── 3. Encrypt both backups ──────────────────────────
echo "[3/5] Encrypting backups..."
GPG_PASSPHRASE=$(cat "${GPG_PASSPHRASE_FILE}")

# Encrypt DB dump
gpg --batch --yes --passphrase "${GPG_PASSPHRASE}" \
    --recipient "${GPG_RECIPIENT}" \
    --encrypt --sign \
    --output "${DB_FILE}.gpg" \
    "${DB_FILE}"

# Encrypt files tarball
gpg --batch --yes --passphrase "${GPG_PASSPHRASE}" \
    --recipient "${GPG_RECIPIENT}" \
    --encrypt --sign \
    --output "${FILES_TAR}.gpg" \
    "${FILES_TAR}"

# Remove unencrypted originals (keep only .gpg)
shred -u "${DB_FILE}" "${FILES_TAR}"
echo "   Encryption complete"

# ── 4. Tag the backup set ────────────────────────────
echo "[4/5] Tagging backup set..."
echo "${DATE}T02:00:00Z" > "${DB_BACKUP_DIR}/LATEST"
echo "${DATE}T02:00:00Z" > "${FILES_BACKUP_DIR}/LATEST"

# ── 5. Cleanup old backups ───────────────────────────
echo "[5/5] Cleaning up backups older than ${RETENTION_DAYS} days..."
find "${DB_BACKUP_DIR}" -name "*.gpg" -mtime "+${RETENTION_DAYS}" -delete
find "${FILES_BACKUP_DIR}" -name "*.gpg" -mtime "+${RETENTION_DAYS}" -delete

echo "=== Backup completed successfully: $(date -u -Iseconds) ==="
```

### 3.3 Systemd Timer (Nightly at 2 AM)  

`/etc/systemd/system/hr-backup.service`:

```ini
[Unit]
Description=HR Leave Management — Nightly Backup
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/opt/hr-leave-management/scripts/backup.sh
User=root
StandardOutput=journal
StandardError=journal
```

`/etc/systemd/system/hr-backup.timer`:

```ini
[Unit]
Description=Nightly backup at 2:00 AM UTC
Requires=hr-backup.service

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
systemctl daemon-reload
systemctl enable --now hr-backup.timer
systemctl status hr-backup.timer   # Verify
```

### 3.4 Remote Sync (Weekly)  

Sync encrypted backups to the off-site server. This requires SSH key-based authentication from the production server to the remote backup server.

**One-time SSH setup** (on production server):

```bash
# Generate SSH key for backup user
sudo -u hr-backup ssh-keygen -t ed25519 -f /home/hr-backup/.ssh/id_ed25519 -N ""

# Copy public key to remote server
sudo -u hr-backup ssh-copy-id backup-user@backup.example.com

# Test connection
sudo -u hr-backup ssh backup-user@backup.example.com "echo OK"
```

**Remote sync script** (`/opt/hr-leave-management/scripts/remote-sync.sh`):

```bash
#!/bin/bash
set -euo pipefail

BACKUP_BASE="/data/backups"
REMOTE_HOST="backup-user@backup.example.com"
REMOTE_PATH="/backups/hr-leave"
LOG_FILE="/var/log/hr-leave-management/remote-sync.log"

exec 1> >(tee -a "${LOG_FILE}")
exec 2>&1
echo "=== Remote sync started: $(date -u -Iseconds) ==="

# Sync encrypted backups (only .gpg and LATEST files)
rsync -avz --delete \
    --include="*/" \
    --include="*.gpg" \
    --include="LATEST" \
    --exclude="*" \
    -e "ssh -i /home/hr-backup/.ssh/id_ed25519" \
    "${BACKUP_BASE}/" \
    "${REMOTE_HOST}:${REMOTE_PATH}/"

echo "=== Remote sync complete: $(date -u -Iseconds) ==="
```

**Systemd timer** (weekly, Sunday at 4 AM):

```ini
[Unit]
Description=Weekly remote backup sync

[Timer]
OnCalendar=Sun *-*-* 04:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

### 3.5 Restore Procedures  

#### Scenario A: Restore Database + Files (Full Recovery)  

```bash
# 1. Stop the API (prevents writes during restore)
docker compose stop api

# 2. Identify the backup to restore
RESTORE_DATE="2026-06-23"  # Change to the desired date
GPG_PASSPHRASE=$(cat /data/keys/backup-passphrase.txt)

# 3. Decrypt DB backup
gpg --batch --yes --passphrase "${GPG_PASSPHRASE}" \
    --decrypt "/data/backups/db/hr_leave_${RESTORE_DATE}.dump.gpg" \
    > "/tmp/hr_leave_${RESTORE_DATE}.dump"

# 4. Drop and recreate the database
docker exec -i hr-postgres psql -U hr_app -d postgres <<SQL
    DROP DATABASE IF EXISTS hr_leave;
    CREATE DATABASE hr_leave OWNER hr_app;
SQL

# 5. Restore
docker exec -i hr-postgres pg_restore \
    --dbname=hr_leave \
    --username=hr_app \
    --no-owner \
    --no-acl \
    --verbose \
    "/tmp/hr_leave_${RESTORE_DATE}.dump"

# 6. Decrypt and restore files
gpg --batch --yes --passphrase "${GPG_PASSPHRASE}" \
    --decrypt "/data/backups/files/documents_${RESTORE_DATE}.tar.gz.gpg" \
    > "/tmp/documents_${RESTORE_DATE}.tar.gz"

rm -rf /data/documents/*
tar -xzf "/tmp/documents_${RESTORE_DATE}.tar.gz" -C /data/

# 7. Verify consistency
docker exec hr-api java -jar /app/quarkus-run.jar check-document-consistency

# 8. Cleanup
shred -u "/tmp/hr_leave_${RESTORE_DATE}.dump" "/tmp/documents_${RESTORE_DATE}.tar.gz"

# 9. Restart
docker compose start api
docker compose ps

# 10. Smoke test
curl -f -c /tmp/cookies.txt -b /tmp/cookies.txt \
    -H "Content-Type: application/json" \
    -d '{"email":"admin@company.com","password":"test"}' \
    https://localhost/api/v1/auth/login
```

#### Scenario B: Database-Only Restore (No Document Loss)  

```bash
# Same as Scenario A but skip steps 6–7 (file restore)
# Useful when only the database was corrupted, not the filesystem
RESTORE_DATE="2026-06-23"
docker compose stop api
gpg --batch --yes --passphrase "${GPG_PASSPHRASE}" \
    --decrypt "/data/backups/db/hr_leave_${RESTORE_DATE}.dump.gpg" \
    > "/tmp/hr_leave_${RESTORE_DATE}.dump"
docker exec -i hr-postgres psql -U hr_app -d postgres -c "DROP DATABASE IF EXISTS hr_leave; CREATE DATABASE hr_leave OWNER hr_app;"
docker exec -i hr-postgres pg_restore --dbname=hr_leave --username=hr_app --no-owner --no-acl "/tmp/hr_leave_${RESTORE_DATE}.dump"
docker compose start api
```

#### Scenario C: Single-Table Restore (Accidental Deletion)  

```bash
# Restore only the leave_requests table from a backup
RESTORE_DATE="2026-06-23"

# Decrypt
gpg --batch --yes --passphrase "${GPG_PASSPHRASE}" \
    --decrypt "/data/backups/db/hr_leave_${RESTORE_DATE}.dump.gpg" \
    > "/tmp/hr_leave_${RESTORE_DATE}.dump"

# Restore only one table
docker exec -i hr-postgres pg_restore \
    --dbname=hr_leave \
    --username=hr_app \
    --table=leave_requests \
    --data-only \
    --no-owner \
    --no-acl \
    "/tmp/hr_leave_${RESTORE_DATE}.dump"
```

#### Scenario D: Remote Backup Restore (Server Loss)  

```bash
# 1. Provision a new server with Docker + Docker Compose
# 2. Copy encrypted backups from remote server
scp backup-user@backup.example.com:/backups/hr-leave/db/hr_leave_2026-06-23.dump.gpg /tmp/
scp backup-user@backup.example.com:/backups/hr-leave/files/documents_2026-06-23.tar.gz.gpg /tmp/

# 3. Copy keys from secure storage (password manager / offline backup)
#    - JWT private key
#    - JWT public key
#    - GPG backup passphrase
#    - .env file with all secrets

# 4. Follow Scenario A from step 3
```

### 3.6 Backup Validation (Weekly)  

This script validates that the most recent backup is restorable:

`/opt/hr-leave-management/scripts/validate-backup.sh`:

```bash
#!/bin/bash
set -euo pipefail

BACKUP_BASE="/data/backups"
RESTORE_TEST_DIR="/tmp/backup-validation-$$"
GPG_PASSPHRASE=$(cat /data/keys/backup-passphrase.txt)
LATEST=$(cat "${BACKUP_BASE}/db/LATEST" | cut -d'T' -f1)

echo "Validating backup from ${LATEST}..."

# 1. Can we decrypt the DB dump?
gpg --batch --yes --passphrase "${GPG_PASSPHRASE}" \
    --decrypt "${BACKUP_BASE}/db/hr_leave_${LATEST}.dump.gpg" \
    > "${RESTORE_TEST_DIR}/db.dump" 2>/dev/null

# 2. Is the dump structurally valid?
pg_restore --list "${RESTORE_TEST_DIR}/db.dump" > /dev/null 2>&1

# 3. Can we decrypt the files tarball?
gpg --batch --yes --passphrase "${GPG_PASSPHRASE}" \
    --decrypt "${BACKUP_BASE}/files/documents_${LATEST}.tar.gz.gpg" \
    > "${RESTORE_TEST_DIR}/files.tar.gz" 2>/dev/null

# 4. Is the tarball valid?
tar -tzf "${RESTORE_TEST_DIR}/files.tar.gz" > /dev/null 2>&1

# Cleanup
rm -rf "${RESTORE_TEST_DIR}"

echo "Backup validation PASSED: ${LATEST}"
```

Run weekly via systemd timer:

```bash
systemctl enable --now hr-backup-validate.timer
```

---

## 4. Monitoring & Alerting  

### 4.1 Health Endpoints  

| Endpoint | Purpose | Expected |
|----------|---------|----------|
| `GET /q/health/live` | Liveness — is the process running? | `200` |
| `GET /q/health/ready` | Readiness — can it serve traffic? | `200` (checks DB + Valkey + RabbitMQ) |
| `GET /q/health/started` | Startup — has initialization finished? | `200` |
| `GET /q/metrics` | Prometheus metrics | `200` with text/plain |

### 4.2 Docker Health Checks  

All services have `healthcheck` blocks in the Docker Compose file. View status:

```bash
docker compose ps                    # Status column
docker inspect hr-api | jq '.[].State.Health'
docker events --filter 'event=health_status'  # Real-time
```

### 4.3 Log Monitoring  

Logs are written to `json-file` driver with rotation (max 10 MB × 3–5 files per service).

```bash
# View logs
docker compose logs --tail=100 api
docker compose logs --since=1h postgres
docker compose logs -f api              # Follow (tail)

# Search for errors
docker compose logs api | grep -i "ERROR\|FATAL\|Exception" | tail -50
```

**Critical log patterns to alert on** (configure in your monitoring tool):

| Pattern | Severity | Action |
|---------|----------|--------|
| `OutOfMemoryError` | CRITICAL | Restart API, investigate heap dump |
| `FATAL: database .* does not exist` | CRITICAL | Database connection lost, check PostgreSQL |
| `Connection refused` (to Valkey/RabbitMQ) | HIGH | Check service health, restart if needed |
| `rate-limited` (repeated) | MEDIUM | Check for abuse or misconfigured client |
| `backup FAILED` | HIGH | Investigate disk space, permissions |
| `disk space` < 20% | HIGH | Cleanup old backups, expand volume |

### 4.4 Prometheus Metrics (Optional)  

If you run Prometheus, scrape `/q/metrics` from the API container:

```yaml
scrape_configs:
  - job_name: 'hr-leave-api'
    metrics_path: '/q/metrics'
    static_configs:
      - targets: ['hr-api:8080']
```

Key metrics to alert on:

```
# API error rate
rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.05

# Cache hit rate below threshold
cache_gets_total{cache="eligible-types",result="hit"} / cache_gets_total < 0.8

# Database connection pool exhaustion
agroal_available_count < 2

# Backup age (from pushgateway)
max_over_time(backup_last_success_timestamp[24h]) < time() - 86400
```

### 4.5 Disk Space Monitoring  

```bash
#!/bin/bash
# Check disk space and alert if below threshold
THRESHOLD=20  # Percentage

USAGE=$(df /data | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$USAGE" -gt $((100 - THRESHOLD)) ]; then
    echo "WARNING: /data disk usage at ${USAGE}% (threshold: ${THRESHOLD}% free)" >&2
    # Send alert (email, Slack webhook, etc.)
fi
```

---

## 5. Database Maintenance  

### 5.1 Weekly Vacuum  

PostgreSQL's autovacuum handles most cleanup, but a weekly manual `VACUUM ANALYZE` ensures optimal query planning:

`/opt/hr-leave-management/scripts/vacuum.sh`:

```bash
#!/bin/bash
set -euo pipefail
echo "Running VACUUM ANALYZE on hr_leave..."
docker exec hr-postgres psql -U hr_app -d hr_leave -c "VACUUM ANALYZE;"
echo "VACUUM complete: $(date -u -Iseconds)"
```

Schedule: weekly (Sunday at 3 AM), between backup (2 AM) and remote sync (4 AM).

### 5.2 Index Maintenance  

```bash
# Reindex (monthly, during low traffic)
docker exec hr-postgres psql -U hr_app -d hr_leave -c "REINDEX DATABASE hr_leave;"
```

### 5.3 Connection Check  

```bash
docker exec hr-postgres psql -U hr_app -d hr_leave -c "
    SELECT count(*) AS active_connections FROM pg_stat_activity WHERE state = 'active';
    SELECT count(*) AS idle_connections FROM pg_stat_activity WHERE state = 'idle';
"
```

### 5.4 Database Size Monitoring  

```bash
docker exec hr-postgres psql -U hr_app -d hr_leave -c "
    SELECT
        table_name,
        pg_size_pretty(pg_total_relation_size(quote_ident(table_name))) AS size
    FROM information_schema.tables
    WHERE table_schema = 'public'
    ORDER BY pg_total_relation_size(quote_ident(table_name)) DESC
    LIMIT 10;
"
```

---

## 6. Certificate & Key Rotation  

### 6.1 TLS Certificates (Let's Encrypt)  

If using Let's Encrypt with Certbot, renewal is automatic. Verify:

```bash
certbot renew --dry-run
```

Manual renewal if auto-renewal fails:

```bash
certbot renew --force-renewal
docker compose restart nginx
```

### 6.2 JWT Signing Keys (Annual)  

JWT keys should be rotated annually. The old key must be kept for the duration of the longest-lived refresh token (30 days) so existing sessions don't break.

```bash
# 1. Generate new key pair
openssl genpkey -algorithm RSA -out /data/keys/privateKey_v2.pem -pkeyopt rsa_keygen_bits:2048
openssl rsa -pubout -in /data/keys/privateKey_v2.pem -out /data/keys/publicKey_v2.pem

# 2. Update API config to use both keys (old for verification, new for signing)
#    In application.properties or env:
#    smallrye.jwt.sign.key.location=/data/keys/privateKey_v2.pem
#    mp.jwt.verify.publickey.location=/data/keys/publicKey_v2.pem
#    mp.jwt.decrypt.key.location=/data/keys/privateKey_v1.pem   # Keep old for verification

# 3. Restart API
docker compose restart api

# 4. After 30 days (all refresh tokens from old key expired), remove old key
rm /data/keys/privateKey_v1.pem /data/keys/publicKey_v1.pem
docker compose restart api
```

### 6.3 Document Encryption Key (Biannual)  

The `ENCRYPTION_KEY` env var is used for AES-256-GCM document encryption. Rotating it requires re-encrypting all existing documents:

```bash
# 1. Generate new key
NEW_KEY=$(openssl rand -base64 32)

# 2. Run re-encryption job (built into the API — a dedicated endpoint)
curl -X POST https://localhost/api/v1/admin/maintenance/reencrypt-documents \
    -H "X-CSRF-Token: $(get_csrf_token)" \
    -H "Content-Type: application/json" \
    -d "{\"new_key\": \"${NEW_KEY}\"}"

# 3. Update .env with new key
# 4. Restart API
docker compose restart api
```

### 6.4 GPG Backup Key (Biannual)  

Regenerate the GPG key pair and securely store the new passphrase.

---

## 7. Software Updates  

### 7.1 Rolling Update (Zero Downtime)  

```bash
# 1. Pull new images
cd /opt/hr-leave-management
APP_VERSION=1.1.0 docker compose pull api frontend

# 2. Update the version in .env
sed -i 's/APP_VERSION=.*/APP_VERSION=1.1.0/' /etc/hr-leave-management/.env

# 3. Rolling restart (Docker Compose replaces containers one at a time)
#    --no-deps: don't restart dependent services (DB, Valkey, etc.)
#    API has health checks — new container must pass before old is stopped
docker compose up -d --no-deps --build api frontend

# 4. Verify
docker compose ps                    # All "healthy"
curl -f https://localhost/q/health/ready
docker compose logs --tail=20 api    # Check for startup errors
```

### 7.2 Database Migration on Update  

Flyway runs migrations automatically at API startup. If a migration fails:

```bash
# Check migration status
docker exec hr-postgres psql -U hr_app -d hr_leave -c "SELECT * FROM flyway_schema_history;"

# If a migration failed and was rolled back, fix the issue and restart
docker compose restart api

# If Flyway is stuck (rare), repair:
docker exec hr-api java -jar /app/quarkus-run.jar flyway:repair
```

### 7.3 Rollback  

```bash
# 1. Stop API
docker compose stop api

# 2. Restore DB from pre-update backup (see §3.5 Scenario B)
RESTORE_DATE="2026-06-22"  # Day before update

# 3. Downgrade version
sed -i 's/APP_VERSION=.*/APP_VERSION=1.0.0/' /etc/hr-leave-management/.env

# 4. Pull old images and start
docker compose pull api frontend
docker compose up -d --no-deps api frontend
```

---

## 8. Disaster Recovery  

### 8.1 Recovery Time Objectives (RTO)  

| Scenario | RTO | Procedure |
|----------|-----|-----------|
| Single container crash | < 1 min | Docker auto-restarts (`restart: unless-stopped`) |
| API unresponsive | < 2 min | `docker compose restart api` |
| Database corruption | < 30 min | Restore latest nightly backup (§3.5 Scenario B) |
| Filesystem failure | < 30 min | Restore files from backup (§3.5 Scenario A) |
| Full server loss | < 2 hours | Provision new server + restore from remote backup (§3.5 Scenario D) |
| Cross-tenant data leak | < 1 hour + review | PITR restore + legal review |

### 8.2 Disaster Recovery Checklist  

```
[ ] Remote backup server is reachable (test SSH monthly)
[ ] GPG passphrase is stored in password manager (not only on server)
[ ] JWT private key is stored in password manager  
[ ] .env file with all secrets is backed up securely
[ ] Docker images are pushed to a registry (not just local)
[ ] Restore procedure has been tested within the last 3 months
[ ] DNS records are documented (for pointing to a new server)
[ ] SSL certificate renewal email contact is current
```

### 8.3 Emergency Contacts  

Maintain a list of who to call (stored outside the system):

| Role | Name | Phone | Email |
|------|------|-------|-------|
| Primary sysadmin | — | — | — |
| Secondary sysadmin | — | — | — |
| DBA (if separate) | — | — | — |
| HR system owner | — | — | — |

---

## 9. Troubleshooting  

### 9.1 Common Issues  

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| API returns 502 | API container crashed | `docker compose logs api --tail=50`, check for `OutOfMemoryError` |
| API returns 503, health check fails | Database unreachable | `docker compose ps postgres`, check disk space |
| Login loops (SSO redirects back to login) | Cookie domain mismatch | Verify `APP_DOMAIN` in .env, check `SameSite` settings |
| "CSRF token mismatch" errors | Stale `csrf_token` cookie | Clear browser cookies; check clock sync between server and client |
| Slow response times | Missing database indexes or cache cold | Check §5.2 index maintenance; Valkey `docker compose logs valkey` |
| Disk full | Backups not being cleaned up | Run retention cleanup manually (§10); check cron/systemd timers |
| Flyway migration fails on startup | Manual schema change conflicts with migration | Fix the conflict, then `docker compose restart api` |

### 9.2 Emergency Access (Break-Glass)  

If all admin accounts are locked out, create a temporary admin directly in the database:

```bash
# ⚠️  ONLY in emergency. Log this action and rotate credentials afterward.
docker exec -i hr-postgres psql -U hr_app -d hr_leave <<SQL
    -- Create emergency admin user (valid for 1 hour)
    INSERT INTO users (id, tenant_id, email, first_name, last_name, status, auth_provider)
    VALUES (gen_random_uuid(), 
            (SELECT id FROM tenants LIMIT 1),
            'emergency@company.com', 'Emergency', 'Admin', 'ACTIVE', 'LOCAL');
    
    -- Set temporary password (bcrypt hash of "Emergency123!")
    UPDATE users SET password_hash = '\$2a\$12\$...' WHERE email = 'emergency@company.com';
    
    -- Assign System Administrator role
    INSERT INTO user_roles (user_id, role_id)
    SELECT u.id, r.id FROM users u, roles r
    WHERE u.email = 'emergency@company.com' AND r.name = 'System Administrator';
SQL

# After regaining access, DELETE this user and rotate all credentials.
```

### 9.3 Diagnostic Commands  

```bash
# Full system status
docker compose ps
df -h /data
free -h
uptime

# Service-specific logs
docker compose logs --tail=200 api      # API errors
docker compose logs --tail=50 postgres  # DB errors
docker compose logs --tail=50 nginx     # 4xx/5xx errors

# Database health
docker exec hr-postgres psql -U hr_app -d hr_leave -c "
    SELECT count(*) AS total_leave_requests FROM leave_requests;
    SELECT status, count(*) FROM leave_requests GROUP BY status;
    SELECT count(*) AS active_sessions FROM refresh_tokens WHERE revoked = false AND expires_at > now();
"

# Valkey health
docker exec hr-valkey valkey-cli PING
docker exec hr-valkey valkey-cli INFO stats | grep -E "keyspace|evicted|hit_rate"

# RabbitMQ health
docker exec hr-rabbitmq rabbitmq-diagnostics status
```

---

## 10. Retention & Cleanup  

### 10.1 Automated Retention (GDPR)  

The API runs a scheduled job (`@Scheduled(cron="0 0 3 * * SUN")`) that:

1. Identifies leave requests > 5 years past `end_date` → anonymizes (replaces employee ID with hash, removes notes).  
2. Identifies medical documents > 5 years past `end_date` → deletes files + DB rows.  
3. Identifies terminated employees > 5 years past termination → anonymizes employee record.  
4. Identifies notifications > 1 year → deletes.  
5. Identifies refresh tokens past expiry → deletes.  

Monitor this job via logs:

```bash
docker compose logs api | grep "RetentionJob"
```

### 10.2 Backup Retention Cleanup  

The backup script (§3.2) handles the 7-day rolling cleanup of nightly backups. Monthly and yearly backups need separate handling:

`/opt/hr-leave-management/scripts/archive-monthly.sh`:

```bash
#!/bin/bash
# Run on the 1st of each month — copies the most recent nightly backup
# into the monthly archive, which is retained for 12 months.

DATE=$(date -u +%Y-%m)
LATEST=$(cat /data/backups/db/LATEST | cut -d'T' -f1)
ARCHIVE_DIR="/data/backups/archive/${DATE}"

mkdir -p "${ARCHIVE_DIR}"
cp "/data/backups/db/hr_leave_${LATEST}.dump.gpg" "${ARCHIVE_DIR}/"
cp "/data/backups/files/documents_${LATEST}.tar.gz.gpg" "${ARCHIVE_DIR}/"

# Cleanup archives older than 12 months
find /data/backups/archive -maxdepth 1 -type d -mtime +365 -exec rm -rf {} \;
```

### 10.3 Log Rotation  

Docker's `json-file` driver handles rotation automatically (configured in docker-compose.yml). For system logs:

```bash
# journald has built-in rotation
journalctl --vacuum-time=30d    # Keep 30 days of system logs
```

---

## 11. Security Hardening Checklist  

```
[ ] LUKS encryption enabled on /data volume
[ ] SSH: password authentication disabled (keys only)
[ ] SSH: root login disabled
[ ] Firewall: only ports 80/443 open to public (ufw or cloud firewall)
[ ] Firewall: port 5432 (PostgreSQL), 6379 (Valkey), 5672 (RabbitMQ) NOT exposed externally
[ ] Docker: backend network is `internal: true`
[ ] .env file permissions: 600, owned by root
[ ] GPG passphrase stored in password manager
[ ] JWT private key stored in password manager
[ ] Database password: minimum 32 characters, randomly generated
[ ] TLS: only TLS 1.2 and 1.3 enabled in nginx
[ ] TLS: HSTS header enabled (max-age=31536000)
[ ] Automatic security updates enabled: `unattended-upgrades` (Ubuntu) or equivalent
[ ] Docker daemon: `live-restore` enabled for zero-downtime daemon updates
[ ] Regular `docker scout` or `trivy` vulnerability scans on images
[ ] No secrets in git repository (all in .env or secrets manager)
```

---

## 12. Runbooks  

### 12.1 Restart All Services  

```bash
cd /opt/hr-leave-management
docker compose restart
# Wait for health checks to pass
sleep 10
docker compose ps
curl -f https://localhost/q/health/ready
```

### 12.2 Check System Health  

```bash
/opt/hr-leave-management/scripts/health-check.sh
```

Create `/opt/hr-leave-management/scripts/health-check.sh`:

```bash
#!/bin/bash
set -e
echo "=== System Health Check — $(date) ==="
echo ""
echo "Disk:"
df -h /data | tail -1
echo ""
echo "Memory:"
free -h | grep Mem
echo ""
echo "Containers:"
docker compose -f /opt/hr-leave-management/docker-compose.yml ps
echo ""
echo "API Health:"
curl -sf https://localhost/q/health/ready && echo "OK" || echo "FAILED"
echo ""
echo "Last Backup:"
cat /data/backups/db/LATEST 2>/dev/null || echo "NO BACKUP FOUND"
```

### 12.3 Emergency Stop  

```bash
cd /opt/hr-leave-management
docker compose stop    # Graceful
# OR
docker compose down    # Full removal (volumes preserved)
```

### 12.4 Add a New Tenant  

```bash
# Via API (requires Super Admin session)
curl -X POST https://localhost/api/v1/admin/tenants \
    -H "Content-Type: application/json" \
    -H "X-CSRF-Token: $(get_csrf_token)" \
    -d '{"name":"New Company","domain":"newcompany.com","country":"ES"}'
```

### 12.5 Reset a User's Password (Local Auth)  

```bash
# Generate bcrypt hash
HASH=$(htpasswd -nbBC 12 "" "NewPassword123!" | tr -d ':\n' | sed 's/^ //')

# Update in database
docker exec -i hr-postgres psql -U hr_app -d hr_leave <<SQL
    UPDATE users SET password_hash = '${HASH}' WHERE email = 'user@company.com';
SQL
```

### 12.6 View Active User Sessions  

```bash
docker exec -i hr-postgres psql -U hr_app -d hr_leave <<SQL
    SELECT u.email, rt.created_at, rt.expires_at
    FROM refresh_tokens rt
    JOIN users u ON rt.user_id = u.id
    WHERE rt.revoked = false AND rt.expires_at > now()
    ORDER BY rt.created_at DESC;
SQL
```

### 12.7 Force Revoke All Sessions for a User  

```bash
docker exec -i hr-postgres psql -U hr_app -d hr_leave <<SQL
    UPDATE refresh_tokens SET revoked = true
    WHERE user_id = (SELECT id FROM users WHERE email = 'user@company.com');
SQL
```

---

## Appendix A: Directory Layout Reference  

```
/opt/hr-leave-management/
├── docker-compose.yml              # Production deployment
├── scripts/
│   ├── backup.sh                   # Nightly backup
│   ├── remote-sync.sh              # Weekly remote sync
│   ├── validate-backup.sh          # Weekly backup validation
│   ├── vacuum.sh                   # Weekly DB vacuum
│   ├── archive-monthly.sh          # Monthly archive
│   └── health-check.sh             # On-demand health check
│
/etc/hr-leave-management/
├── .env                            # Secrets (chmod 600)
├── nginx.conf                      # Nginx configuration
├── postgresql.conf                 # PostgreSQL overrides
├── dhparam.pem                     # DH parameters for TLS
└── certs/
    ├── fullchain.pem               # TLS certificate
    └── privkey.pem                 # TLS private key
│
/data/
├── pgdata/                         # PostgreSQL data (LUKS encrypted)
├── documents/                      # Leave documents (AES-256-GCM encrypted)
├── backups/
│   ├── db/                         # Encrypted DB dumps (.gpg)
│   │   └── LATEST                  # Timestamp of latest backup
│   ├── files/                      # Encrypted document tarballs (.gpg)
│   │   └── LATEST
│   └── archive/                    # Monthly/yearly long-term archives
└── keys/
    ├── privateKey.pem              # JWT signing key (chmod 400)
    ├── publicKey.pem               # JWT verification key (chmod 444)
    ├── backup-passphrase.txt       # GPG backup passphrase (chmod 400)
    └── backup-public.asc           # GPG public key (chmod 444)
│
/var/log/hr-leave-management/
├── backup.log
├── remote-sync.log
└── health-check.log
```

---

> **End of Operations Manual**  
> Last reviewed: 2026-06-23 · Next review: 2026-12-23  
> For emergencies, contact the system administrator listed in §8.3.
