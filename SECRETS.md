---
layout: default
title: Secrets Handling
---

# Secrets Handling — Complete Reference

How Invenzo stores, reads, protects, rotates, and recovers the sensitive
values it needs at runtime: the PostgreSQL password, Redis password, JWT
signing key, AES credential-encryption key, OpenBao root token, worker↔api
shared secret, admin password, and Grafana admin password.

This document is the single source of truth. It covers:

1. [Overview and threat model](#1-overview-and-threat-model)
2. [What's protected](#2-whats-protected)
3. [Architecture](#3-architecture)
4. [Installation flow — what `install.sh` does, step by step](#4-installation-flow)
5. [Runtime flow — what happens on `docker compose up`](#5-runtime-flow)
6. [Operations — rotation, backup, verification, preflight](#6-operations)
7. [**Recovery from a lost age key — complete procedure**](#7-recovery-from-a-lost-age-key)
8. [Migration from legacy modes (env / files)](#8-migration-from-legacy-modes)
9. [SaaS path — what the remote-provider tier looks like](#9-saas-path)
10. [What was NOT built and why](#10-what-was-not-built-and-why)
11. [FAQ](#11-faq)

---

## 1. Overview and threat model

Invenzo v1.10+ ships with **encrypted-at-rest secrets by default**. Every
application secret lives on disk as an
[age](https://age-encryption.org/)-encrypted blob. The master age private
key lives in a separate location from the encrypted blobs so a backup of
one without the other is useless.

### Threat model

| Attack | Protection level |
|---|---|
| Someone grabs a `tar` of `/opt/invenzo/` | Safe — blobs are age ciphertext, useless without the key |
| Someone grabs a filesystem snapshot of the host's data volume | Safe — same reason |
| Backup tape shipped off-site includes `/opt/invenzo/` | Safe — same reason |
| Someone steals the host's disk | Depends — if the age key also lived on the same disk (which is usually the case), they get both. This is a separate disk-encryption problem (LUKS, BitLocker). |
| Someone `docker inspect`s a running container | Safe — no secrets in env vars; values only exist in container tmpfs |
| Someone with `docker exec` or root on the host runs `cat /dev/shm/invenzo-secrets/*` | Not protected — this threat model assumes a rooted host reads everything. Encryption-at-rest is about offline copies, not live compromise. |
| Someone reads application logs | Safe — no secret values are ever logged; the preflight's provenance line shows `NAME=file` without values |

### Design decisions

| Decision | Rationale |
|---|---|
| age (not sops, not LUKS, not GPG) | Smallest sufficient tool — single-recipient symmetric encryption, 3 MB binary, no key management server |
| Master key on host (not TPM/USB/KMS) | Portability — VMs, cloud, bare-metal, air-gapped all work the same |
| One age recipient (not multi-recipient) | Simpler ops; rotation is explicit, not silent |
| Encrypt blobs NOT committed to git | Avoids "key leaks via compromised git repo" class of problem |
| Dev and prod identical flow | What you test in dev is exactly what runs in prod |

---

## 2. What's protected

Eight secrets are stored as `./secrets/<NAME>.enc` under the install dir.
Each exists for one specific reason:

| Name | Purpose | Size | Consumer |
|---|---|---|---|
| `POSTGRES_PASSWORD` | Auth for the `invenzo` DB user | ~16 hex chars | api, worker, worker-beat, pinger, postgres itself via `POSTGRES_PASSWORD_FILE` |
| `REDIS_PASSWORD` | Auth for Redis `--requirepass` | ~16 hex chars | api, worker, worker-beat, pinger, redis itself via shell-wrapped `--requirepass` |
| `SECRET_KEY` | JWT signing key | 64 hex chars | api only |
| `VAULT_KEY` | AES-256-GCM key that encrypts **stored scan credentials** in the DB `credentials` table | 64 hex chars (32 bytes) | api, worker — both sides decrypt the DB-stored SSH/SNMP/WMI passwords |
| `VAULT_ROOT_TOKEN` | OpenBao dev-mode root token | 32+ hex chars | api, worker, vault itself |
| `INTERNAL_API_SECRET` | Shared secret on `X-Internal-Secret` header for `/api/v1/agents/internal/*` routes (worker → api) | 32+ hex chars | api, worker |
| `FIRST_ADMIN_PASSWORD` | Seeds the admin row on first-ever boot (users table empty) | ≥ 12 chars | api (once, then inert) |
| `GRAFANA_ADMIN_PASSWORD` | Grafana admin login | ~16 hex chars | grafana (via `GF_SECURITY_ADMIN_PASSWORD__FILE`) |

There is ONE secret that is NOT in `./secrets/` and NOT age-encrypted:

| Name | Where | Why |
|---|---|---|
| `/etc/invenzo/age.key` | Linux prod: `/etc/invenzo/age.key` (chmod 400, root:root). Windows dev: configurable path. | This IS the master key. It can't encrypt itself. OS file permissions are the gate. |

Everything else — assets, users, software inventory, discovery results,
dashboards, schedules, audit logs, locations, tags, contracts, backups —
lives unencrypted in PostgreSQL and named Docker volumes. It's protected
by the database password and by OS-level access to the volumes, **not**
by the age key.

---

## 3. Architecture

### File layout

```
Host filesystem (customer's Linux server, or Windows dev box):

  /etc/invenzo/
    age.key                    <- master key, mode 400, root:root
                                  74 bytes, single line "AGE-SECRET-KEY-1..."

  /opt/invenzo/                (prod)  OR  /path/to/repo/ (dev)
    .env                       <- non-secret config only:
                                  REGISTRY, VERSION, ALLOWED_HOST,
                                  HTTP_PORT, AGE_KEY_FILE, LOG_LEVEL, etc.
    docker-compose.yml         <- ships with the package

    secrets/                   <- 8 age-encrypted blobs, mode 400 each
      POSTGRES_PASSWORD.enc    <- ~217 bytes of ciphertext
      REDIS_PASSWORD.enc
      SECRET_KEY.enc
      VAULT_KEY.enc
      VAULT_ROOT_TOKEN.enc
      INTERNAL_API_SECRET.enc
      FIRST_ADMIN_PASSWORD.enc
      GRAFANA_ADMIN_PASSWORD.enc

    entrypoints/
      decrypt-and-exec.sh      <- shared boot wrapper for api/worker/pinger
                                  (bind-mounted read-only into each)

    postgres/init/01_schema.sql  <- first-boot schema bootstrap

  (Data volumes — separate, survive container rebuilds)
    postgres_data, redis_data, vault_data, ca_data, grafana_data,
    caddy_data, caddy_config, floor_maps_data, backup_data,
    loki_data, prometheus_data
```

### Container view

```
Inside each service container at boot:

  /encrypted/                    <- bind-mount of host ./secrets/ (read-only)
    POSTGRES_PASSWORD.enc
    ...8 files

  /age-key                       <- bind-mount of the age private key (read-only)

  /usr/local/bin/decrypt-and-exec.sh   <- bind-mount, shared wrapper
                                       (for api/worker/worker-beat/pinger)

  /dev/shm/invenzo-secrets/      <- container-local TMPFS (memory only)
    POSTGRES_PASSWORD            <- decrypted plaintext, mode 400
    ...8 files                     CREATED AT BOOT, DESTROYED AT CONTAINER STOP

  Environment:
    SECRETS_DIR=/dev/shm/invenzo-secrets
```

**Where plaintext lives at any given moment:**

| Location | State |
|---|---|
| Host disk `./secrets/*.enc` | Always ciphertext |
| Host disk `/etc/invenzo/age.key` | Private key (the "decryption authority") |
| Host disk `.env` | Non-secret config only |
| Running container `/dev/shm/invenzo-secrets/` | Plaintext, but tmpfs = RAM only, never flushes to disk |
| `docker inspect` output | Nothing — env vars are non-secret only |
| Application logs | Nothing — only `NAME=file` provenance lines, no values |
| Database | `credentials.secret_encrypted` column = AES(value, VAULT_KEY) — but this is wrapped twice (AES + age-encrypted VAULT_KEY) |

---

## 4. Installation flow

### What the customer types

```bash
# On a fresh Linux host, as root:
git clone https://github.com/raguyazhin/invenzo-package.git
cd invenzo-package
sudo bash install.sh
```

### What `install.sh` actually does (step-by-step)

```
Step 1: Preflight
  - Check: running as root (needed for chmod 400 root:root)
  - Check: Docker installed and reachable
  - Check: CPU/memory/disk meet minimums
  - Check: openssl present (for random-value generation)

Step 2: Install Docker if missing (optional)

Step 3: Copy package files to INSTALL_DIR (default /opt/invenzo/)
  - docker-compose.yml
  - entrypoints/decrypt-and-exec.sh
  - caddy/Caddyfile.prod
  - postgres/init/01_schema.sql
  - loki, promtail, prometheus, grafana configs
  - install.sh / update.sh / uninstall.sh / recover-lost-age-key.sh
  - Creates: secrets/ (mode 700), backups/, agent/dist/

Step 4: Prompt the admin for:
  - Admin password (min 12 chars, confirmed)
  - Admin email
  - Server hostname or IP (auto-detects, accepts override)

Step 5: Generate the age keypair (ONLY if /etc/invenzo/age.key doesn't exist)
  - Runs: `docker run --rm alpine:3.20 sh -c 'apk add -q age && age-keygen'`
  - Parses output, extracts the AGE-SECRET-KEY-* private line and the
    public identity (age1xxx...)
  - Writes private to /etc/invenzo/age.key (chmod 400, root:root)
  - Prints the public identity to stdout:
      "Public identity: age1uxhp6ehchvhqt292tzn5t75yarmzp46a6..."
  - Logs instruction: "Back this up OFFLINE, separate from /opt/invenzo/"

  If age.key already exists (re-running install), REUSES it. This is
  critical — regenerating would break decryption of existing .enc files.

Step 6: Derive the public identity from the private key (again, for encryption)
  - Runs: `docker run --rm -v /etc/invenzo/age.key:/k:ro alpine:3.20
                sh -c 'apk add -q age && age-keygen -y /k'`
  - Stores in shell variable AGE_PUB

Step 7: Generate 8 fresh random secret values in shell variables
  (NEVER written to disk as plaintext — only live in memory)
  - pg_pass=$(openssl rand -hex 16)
  - redis_pass=$(openssl rand -hex 16)
  - secret_key=$(openssl rand -hex 64)
  - vault_key=$(openssl rand -hex 32)            # exactly 64 hex chars
  - vault_root_token="invenzo-$(openssl rand -hex 16)"
  - internal_api_secret=$(openssl rand -hex 32)
  - grafana_admin_pass=$(openssl rand -hex 16)
  - admin_pass=<what user typed>

Step 8: Encrypt each value directly to ./secrets/<NAME>.enc
  Runs ONE docker container that:
  - Receives each value as a docker -e NAME=value environment variable
    (plaintext lives briefly in the container's env for ~100ms)
  - Loops through the 8 variable names, pipes each to:
      printf '%s' "$VALUE" | age --encrypt --recipient "$AGE_PUB" \
                              --output "/out/${NAME}.enc"
  - The ciphertext is written to /opt/invenzo/secrets/<NAME>.enc
  - chmod 400 on each, chown root:root

  At no point does plaintext touch the host's persistent disk. The values
  only exist in:
    (a) The install.sh process's shell variables
    (b) The docker-run child process's environment (memory, destroyed
        when the child exits ~2-3 seconds later)
    (c) The age stdin pipe (fd, in-memory, destroyed at EOF)

Step 9: Scrub shell variables
  - pg_pass="scrubbed"
  - (same for all others)

  Best-effort — they already went into the docker env but overwriting
  now means later shell traces or crash dumps won't contain them.

Step 10: Write .env with non-secret config only
  - REGISTRY=raguyazhin
  - VERSION=<install.sh's INVENZO_VERSION constant>
  - AGE_KEY_FILE=/etc/invenzo/age.key
  - ALLOWED_HOST=<user's hostname>
  - HTTP_PORT=80, HTTPS_PORT=443
  - AGENT_CALLBACK_URL=http://<hostname>:80
  - FIRST_ADMIN_USERNAME=admin
  - FIRST_ADMIN_EMAIL=<user's email>
  - GRAFANA_ADMIN_USER=admin
  - LOG_LEVEL=INFO, LOG_FORMAT=json
  - WORKER_CONCURRENCY=24, PING_INTERVAL_SECONDS=30
  - API_BASE_URL=http://api:8000
  Writes with chmod 600 root:root.

Step 11: Pull Docker images from registry
  - docker compose pull
  (Pre-built images include `age` — baked in at image build time)

Step 12: docker compose up -d
  - All 13 services start
  - Each container does its own decryption at boot (see Runtime Flow below)

Step 13: Wait for API to respond healthy
  - Poll http://localhost:<HTTP_PORT>/api/v1/health

Step 14: Print summary
  - URL to open, initial admin credentials, backup location reminder
```

### After install, the customer's disk contains

```
/etc/invenzo/
  age.key                                    <- CRITICAL, back this up offline

/opt/invenzo/
  .env                                       <- safe to include in support bundles
  docker-compose.yml, Caddyfile.prod, etc.
  secrets/                                   <- ciphertext only
    8 × *.enc files
  entrypoints/
    decrypt-and-exec.sh
  ...
```

Zero plaintext secrets on the host filesystem.

---

## 5. Runtime flow

### What `docker compose up -d` triggers

Each service starts in dependency order (`depends_on` with
`condition: service_healthy`). For each service that consumes secrets, the
flow is:

#### A. API / worker / worker-beat / pinger (Python services)

```
Container starts → entrypoint = /usr/local/bin/decrypt-and-exec.sh

  decrypt-and-exec.sh:
    1. Create /dev/shm/invenzo-secrets/ (tmpfs, chmod 700)
    2. For each /encrypted/*.enc:
         age --decrypt --identity /age-key --output /dev/shm/invenzo-secrets/<NAME>
         chmod 400 <output>
    3. Export SECRETS_DIR=/dev/shm/invenzo-secrets
    4. exec "$@"  (uvicorn, celery, python pinger.py, etc.)

  Python app starts:
    - Pydantic Settings reads /dev/shm/invenzo-secrets/SECRET_KEY,
      /dev/shm/invenzo-secrets/VAULT_KEY, etc. (via secrets_dir)
    - Any code path that calls read_secret("POSTGRES_PASSWORD") gets
      the plaintext from /dev/shm/invenzo-secrets/POSTGRES_PASSWORD
    - Preflight logs: "secret sources: POSTGRES_PASSWORD=file,
      REDIS_PASSWORD=file, SECRET_KEY=file, ..."
    - Preflight logs: "0 warnings — secret hygiene OK" (or warnings)

  API is now serving on :8000
```

`age` is pre-baked into the api/discovery images via
`apt-get install -y age` in Dockerfile.prod — the line:
```Dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 age \
    && rm -rf /var/lib/apt/lists/* ...
```

#### B. PostgreSQL

```
Container starts → entrypoint is overridden in compose to:

  sh -c "
    set -e
    apk add --quiet --no-cache age          # ~2 sec
    mkdir -p /dev/shm/invenzo-secrets && chmod 700 /dev/shm/invenzo-secrets
    age --decrypt --identity /age-key \\
        --output /dev/shm/invenzo-secrets/POSTGRES_PASSWORD \\
        /encrypted/POSTGRES_PASSWORD.enc
    chmod 400 /dev/shm/invenzo-secrets/POSTGRES_PASSWORD
    exec docker-entrypoint.sh postgres       # stock postgres entrypoint
  "

  Postgres then honors POSTGRES_PASSWORD_FILE=/dev/shm/invenzo-secrets/POSTGRES_PASSWORD
  (native support built into the postgres:16-alpine image).
```

Postgres' `POSTGRES_PASSWORD_FILE` env var is a standard feature of the
official image — it reads the file and uses its contents as the password
on first init, and doesn't care after that (the password lives in
`pg_authid` from then on).

#### C. Redis

```
Container starts → entrypoint overridden to:

  sh -c "
    apk add --quiet --no-cache age
    age --decrypt --identity /age-key --output /dev/shm/invenzo-secrets/REDIS_PASSWORD ...
    exec redis-server --requirepass \"\$(cat /dev/shm/invenzo-secrets/REDIS_PASSWORD)\" \\
                      --maxmemory 256mb --maxmemory-policy allkeys-lru --appendonly yes
  "
```

The redis image doesn't support `_FILE` env vars for `--requirepass`, so
we shell-expand the file into the argument list.

#### D. OpenBao (vault)

```
Container starts → entrypoint overridden to:

  sh -c "
    apk add --quiet --no-cache age
    age --decrypt --identity /age-key --output /dev/shm/invenzo-secrets/VAULT_ROOT_TOKEN ...
    export BAO_DEV_ROOT_TOKEN_ID=\"\$(cat /dev/shm/invenzo-secrets/VAULT_ROOT_TOKEN)\"
    exec bao server -dev
  "
```

#### E. Grafana

```
Container starts → entrypoint overridden to:

  sh -c "
    apk add --quiet --no-cache age
    age --decrypt --identity /age-key --output /dev/shm/invenzo-secrets/GRAFANA_ADMIN_PASSWORD ...
    chmod 444  # grafana user (uid 472) needs read
    exec /run.sh        # stock grafana init, drops to grafana user internally
  "

  Grafana reads GF_SECURITY_ADMIN_PASSWORD__FILE natively.
```

### Verification the decryption actually happened

```bash
# Inside any service container
docker compose exec api ls -la /dev/shm/invenzo-secrets/
#  -r--------    1 invenzo  invenzo       128 Apr 22 17:26 SECRET_KEY
#  -r--------    1 invenzo  invenzo        64 Apr 22 17:26 VAULT_KEY
#  ... 8 files total, mode 400

docker compose logs api | grep preflight
#  preflight: secret sources: POSTGRES_PASSWORD=file, REDIS_PASSWORD=file, ...
#  preflight: 0 warnings — secret hygiene OK

# `docker inspect` shows no secret values
docker inspect invenzo-api-1 --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | grep -E "POSTGRES_PASSWORD|VAULT_KEY|SECRET_KEY|REDIS_PASSWORD"
# (no output — secrets are NOT in env)

# Host-side: prove .enc files are real ciphertext
file /opt/invenzo/secrets/SECRET_KEY.enc
#  SECRET_KEY.enc: age encrypted file

# Host-side: prove no plaintext leaked
grep -r "<an-actual-password-fragment>" /opt/invenzo/secrets/
# (no match)
```

---

## 6. Operations

### Rotation — single secret (same age key)

```bash
# Generate a fresh value and encrypt it with the existing age public key
AGE_PUB=$(sudo docker run --rm -v /etc/invenzo/age.key:/k:ro \
            alpine:3.20 sh -c 'apk add -q age && age-keygen -y /k')

openssl rand -hex 32 | sudo docker run --rm -i \
    -v /opt/invenzo/secrets:/out \
    -e AGE_PUB="$AGE_PUB" \
    alpine:3.20 sh -c 'apk add -q age && age --encrypt --recipient "$AGE_PUB" --output /out/VAULT_KEY.enc'

# Restart the services that use it
sudo docker compose -p invenzo restart api worker worker-beat pinger
```

**VAULT_KEY rotation specifically** needs a dual-key window so existing
DB-stored credentials re-encrypt transparently. Use `VAULT_KEY_OLD` as
documented in [api/app/services/credential_service.py](../api/app/services/credential_service.py).

### Rotation — the age key itself

```bash
# 1. Generate new keypair, save pub
docker run --rm alpine:3.20 sh -c 'apk add -q age && age-keygen' > /tmp/new
NEW_PUB=$(grep '^# public key:' /tmp/new | awk '{print $NF}')

# 2. For each existing blob: decrypt with OLD key, re-encrypt with NEW recipient
for enc in /opt/invenzo/secrets/*.enc; do
    docker run --rm -v /etc/invenzo/age.key:/oldkey:ro \
               -v "$(dirname "$enc"):/work" \
               -e NEW_PUB="$NEW_PUB" \
               alpine:3.20 sh -c "
        apk add -q age
        age -d -i /oldkey /work/$(basename "$enc") \
          | age -e -r \$NEW_PUB -o /work/$(basename "$enc").new
        mv /work/$(basename "$enc").new /work/$(basename "$enc")
    "
done

# 3. Atomically swap the private key
sudo grep '^AGE-SECRET-KEY' /tmp/new > /etc/invenzo/age.key.new
sudo mv /etc/invenzo/age.key.new /etc/invenzo/age.key
sudo chmod 400 /etc/invenzo/age.key
sudo rm /tmp/new

# 4. Restart
sudo docker compose -p invenzo restart
```

### Backup rules

| Artifact | Back up? | Where |
|---|---|---|
| `/etc/invenzo/age.key` | **Yes, critical** | OFFLINE, separate from /opt/invenzo tarballs. Admin USB, 1Password/Bitwarden note, hardware key escrow, printed + safe. |
| `/opt/invenzo/secrets/*.enc` | Yes | Included in normal `/opt/invenzo/` backups. Safe to transmit — ciphertext only. |
| `/opt/invenzo/.env` | Yes | Normal backup — no secrets in it, only config. |
| `postgres_data` volume | Yes | Via `backup_service.py` which runs `pg_dump` into `/opt/invenzo/backups/`. |
| `grafana_data`, `vault_data`, `caddy_data` | Optional | Non-critical — regenerable. |

**The inviolable rule:** age.key and secrets/*.enc must never travel in
the same channel. If both are in the same tarball, encryption is defeated.

### Preflight verification

On every API container start, preflight logs run and cache findings. Access them:

```bash
# As admin via API
curl -H "Authorization: Bearer <admin-token>" \
     http://<host>/api/v1/internal/secrets/health | jq

# Response shape:
{
  "provenance": {
    "POSTGRES_PASSWORD": "file",
    "SECRET_KEY": "file",
    ...
  },
  "findings": [],
  "total": 0,
  "severity_counts": {}
}
```

Preflight checks, each with a specific code:

| Code | Trigger |
|---|---|
| `PREFLIGHT_ENV_PERMISSIVE` | `.env` on host is group- or world-readable |
| `PREFLIGHT_DEFAULT_VALUE` | A secret equals a known dev default (`changeme`, `invenzo_dev_2026`, the 64-char dev VAULT_KEY, etc.) |
| `PREFLIGHT_DEV_DEFAULT_IN_PROD` | Same as above but at ERROR severity when production=true |
| `PREFLIGHT_SECRET_SHORT` | A secret is shorter than its minimum length |
| `PREFLIGHT_VAULT_KEY_FORMAT` | VAULT_KEY isn't exactly 64 hex chars |
| `PREFLIGHT_FILES_MODE_MISSING` | SECRETS_MODE=files set but files not mounted |

Every finding reports the **name and length** only — never the value. This
is enforced by constant-time comparisons (`hmac.compare_digest`) and by
the value never being interpolated into log messages.

---

## 7. Recovery from a lost age key

**Scenario:** `/etc/invenzo/age.key` has been deleted, corrupted, or its
backup was lost, and `./secrets/*.enc` cannot be decrypted.

### Fast path — run the recovery script

```bash
cd /opt/invenzo
sudo bash recover-lost-age-key.sh
```

The script walks through every step automatically. It pauses for two
explicit confirmations (`RECOVER` and, if an age.key already exists,
`OVERWRITE`), then:

1. Stops the stack (volumes preserved)
2. Generates a new age keypair at `/etc/invenzo/age.key`
3. Resets the postgres `invenzo` user's password in-place via a
   trust-auth dance (detailed below)
4. Generates 8 new random secrets
5. Encrypts each to `./secrets/*.enc` under the new key
6. Updates `AGE_KEY_FILE` in `.env` if needed
7. Starts the stack
8. Waits for API health
9. Prints the post-recovery checklist

Run time: typically under 2 minutes on a stock install.

### What survives vs what you must re-enter

**Survives (zero data loss):**

- Every row in postgres — assets, software, users, locations, schedules,
  saved views, discovery history, dashboards, audit logs, backup files
- All named Docker volumes (postgres_data, redis_data, vault_data,
  grafana_data, caddy_data, ca_data, backup_data, floor_maps_data)
- **Admin login** — the admin's password hash is bcrypt-stored in the
  `users` table. Bcrypt is not encrypted by VAULT_KEY. The admin logs in
  with the same password they've always used.
- Agent registrations, mTLS certs, RBAC permissions, tag categories,
  asset templates, dropdown lists, SNMP OID registry

**Lost and must be re-entered via UI:**

- **Stored scan credentials only.** The `credentials` table has rows
  like "prod-linux-ssh", "exchange-server-wmi", "core-switch-snmp". Each
  row has:
  - name, type, username, target subnet → preserved
  - secret/password/community/private-key → lost (encrypted with old VAULT_KEY)

  After recovery: open **Credentials** in the sidebar, edit each row,
  re-paste the SSH/WMI/SNMP/cloud password/key. Most customers have
  5-20 credential rows — ten minutes of work.

### Manual procedure (if you prefer to understand each step)

```bash
set -euo pipefail
INSTALL_DIR=/opt/invenzo
COMPOSE_PROJECT=invenzo

# 1. Stop the stack (volumes preserved)
cd "$INSTALL_DIR"
sudo docker compose -p "$COMPOSE_PROJECT" down

# 2. Generate a new age keypair
sudo mkdir -p /etc/invenzo && sudo chmod 700 /etc/invenzo
docker run --rm alpine:3.20 sh -c 'apk add -q age && age-keygen' > /tmp/newkey
sudo sh -c "grep '^AGE-SECRET-KEY' /tmp/newkey > /etc/invenzo/age.key"
sudo chmod 400 /etc/invenzo/age.key
AGE_PUB=$(grep '^# public key:' /tmp/newkey | awk '{print $NF}')
echo "New public identity: $AGE_PUB"
rm /tmp/newkey

# 3. Generate fresh secret values (kept in memory)
PG_PASS=$(openssl rand -hex 16)
REDIS_PASS=$(openssl rand -hex 16)
SECRET_KEY=$(openssl rand -hex 64)
VAULT_KEY=$(openssl rand -hex 32)
VAULT_ROOT="invenzo-$(openssl rand -hex 16)"
INTERNAL=$(openssl rand -hex 32)
GRAFANA=$(openssl rand -hex 16)
FIRST_ADMIN=$(openssl rand -hex 16)   # inert post-first-boot

# 4. Reset postgres 'invenzo' user password via trust-auth dance.
#
# Why this works: pg_hba.conf controls how connections are authenticated.
# By temporarily setting `local all all trust`, any unix-socket connection
# succeeds without a password. We then ALTER USER to set a new password
# (which replaces the password hash in pg_authid — the only place postgres
# actually stores the password). After ALTER USER, we restore pg_hba.conf
# so the next postgres start enforces real auth again.
#
# Nothing in the data volume other than pg_authid (password hash) and
# pg_hba.conf (restored) is touched. All other tables and rows untouched.
docker run --rm \
    -v invenzo_postgres_data:/var/lib/postgresql/data \
    --user postgres \
    -e PG_NEWPASS="$PG_PASS" \
    postgres:16-alpine \
    sh -c '
        set -e
        DATA=/var/lib/postgresql/data
        cp "$DATA/pg_hba.conf" /tmp/pg_hba.conf.bak
        printf "local all all trust\nhost all all 127.0.0.1/32 trust\nhost all all ::1/128 trust\n" > "$DATA/pg_hba.conf"
        pg_ctl start -D "$DATA" -o "-c listen_addresses=" -w -l /tmp/pg.log
        psql -U invenzo -d invenzo -v ON_ERROR_STOP=1 \
            -c "ALTER USER invenzo PASSWORD '"'"'${PG_NEWPASS}'"'"';"
        pg_ctl stop -D "$DATA" -m fast -w
        mv /tmp/pg_hba.conf.bak "$DATA/pg_hba.conf"
    '

# 5. Encrypt each new secret directly to ./secrets/*.enc
for entry in "POSTGRES_PASSWORD=$PG_PASS" "REDIS_PASSWORD=$REDIS_PASS" \
             "SECRET_KEY=$SECRET_KEY" "VAULT_KEY=$VAULT_KEY" \
             "VAULT_ROOT_TOKEN=$VAULT_ROOT" "INTERNAL_API_SECRET=$INTERNAL" \
             "FIRST_ADMIN_PASSWORD=$FIRST_ADMIN" "GRAFANA_ADMIN_PASSWORD=$GRAFANA"; do
    name="${entry%%=*}"
    val="${entry#*=}"
    printf '%s' "$val" | docker run --rm -i \
        -v "$INSTALL_DIR/secrets:/out" \
        -e AGE_PUB="$AGE_PUB" alpine:3.20 \
        sh -c "apk add -q age && age --encrypt --recipient \"\$AGE_PUB\" --output /out/${name}.enc"
    sudo chmod 400 "$INSTALL_DIR/secrets/${name}.enc"
done

# 6. Scrub sensitive shell vars (best-effort)
unset PG_PASS REDIS_PASS SECRET_KEY VAULT_KEY VAULT_ROOT INTERNAL GRAFANA FIRST_ADMIN

# 7. Ensure .env points at the new key
if grep -q '^AGE_KEY_FILE=' "$INSTALL_DIR/.env"; then
    sudo sed -i 's|^AGE_KEY_FILE=.*|AGE_KEY_FILE=/etc/invenzo/age.key|' "$INSTALL_DIR/.env"
else
    echo "AGE_KEY_FILE=/etc/invenzo/age.key" | sudo tee -a "$INSTALL_DIR/.env" >/dev/null
fi

# 8. Start the stack. Redis, Grafana, and OpenBao pick up their new
# passwords on first start — none of them encrypt their persisted data
# using the password.
sudo docker compose -p "$COMPOSE_PROJECT" up -d

# 9. Wait for API health
until curl -sfo /dev/null "http://localhost:${HTTP_PORT:-3080}/api/v1/health"; do
    sleep 2
done
echo "API healthy."

# 10. Back up the new age key OFFLINE (do this now, before anything else)
# Example: scp /etc/invenzo/age.key operator@backup-vault:/secure/invenzo/
```

### Post-recovery checklist

1. **Back up `/etc/invenzo/age.key`** offline, separate from `/opt/invenzo/`.
2. **Log in** at `http://<host>:<port>/` with your EXISTING admin password
   (the one you've been using — the `FIRST_ADMIN_PASSWORD` variable is
   regenerated but only matters on first-ever boot when the users table
   is empty; post-recovery, the admin row already exists and its password
   is the one you already know).
3. **Go to Credentials** in the sidebar.
4. For each credential row: click edit, paste the SSH/WMI/SNMP/cloud
   password in the Secret field, save. The "name" column will tell you
   which credential is for which target.
5. **Run a test discovery scan** against a known device to verify the
   re-entered credentials actually authenticate.

### Why this works — the technical reasoning

| Concern | Answer |
|---|---|
| "If age.key is lost, how do we decrypt the OLD secrets?" | **We don't.** We generate NEW random secret values and encrypt those instead. The old `.enc` files become useless. |
| "But the postgres DB was initialized with the OLD password — how do we get back in?" | postgres stores passwords as hashes in its `pg_authid` table. The hash is created on first init using `POSTGRES_PASSWORD_FILE`, but after that postgres only cares what's in `pg_authid`. We can rewrite `pg_authid` by running `ALTER USER invenzo PASSWORD` as a postgres superuser. And we can connect as that superuser without a password by temporarily setting `pg_hba.conf` to `trust` auth on local connections. Once the password is reset, we restore pg_hba. |
| "What about redis, grafana, openbao? They were started with the old passwords." | None of them encrypt their persisted data using the password — the password is only an auth check. Redis AOF/RDB files don't care. Grafana's dashboards in `grafana_data` don't care. OpenBao in dev-mode keeps data in memory, so it gets re-initialized fresh anyway. Restart with the new password, and they all work. |
| "Can the admin still log in?" | Yes. User passwords in the `users` table are bcrypt-hashed. Bcrypt is not encrypted by VAULT_KEY. The admin's password is whatever they set it to (possibly via install.sh's `FIRST_ADMIN_PASSWORD`) and it's preserved across recovery. |
| "What exactly is lost?" | Only the `credentials.secret_encrypted` column in the DB — AES-encrypted with the old VAULT_KEY. New VAULT_KEY can't decrypt those bytes. The credential NAMES, TYPES, USERNAMES, and CONFIGS (non-secret stuff) are all preserved because those columns aren't encrypted. |

---

## 8. Migration from legacy modes

Invenzo's encryption story evolved across three tiers:

| Tier | Characteristic |
|---|---|
| **Tier 1 — env mode** | Plaintext secrets in `.env` on host. Default before v1.9. |
| **Tier 2 — files mode** | Plaintext secrets in `./secrets/<NAME>` on host, mounted as tmpfs into containers. Intermediate, opt-in. |
| **Tier 3 — encrypted mode** | age-encrypted `./secrets/<NAME>.enc`, decrypted at container boot. **Default from v1.10+**. |

### Upgrading from Tier 1 (env mode) to Tier 3

Paste the migration block at the top of
[UPGRADE.md](UPGRADE.md) into a root shell. It reads the existing
`.env` secrets, generates an age keypair, encrypts each to `.enc`, strips
the secret lines from `.env`, and adds `AGE_KEY_FILE=/etc/invenzo/age.key`.
Then run `update.sh <version>` as normal.

### Upgrading from Tier 2 (files mode) to Tier 3

Same migration block works — it reads `./secrets/<NAME>` plaintext files,
encrypts them, removes the plaintext files afterward. The logic is:
if the plaintext file exists, encrypt it; if it doesn't, fall back to
reading from `.env`.

### Downgrading

Not supported. Once you're in encrypted mode, you stay in encrypted mode.
The cost of a downgrade (re-exposing plaintext) is higher than any
benefit. If a customer genuinely needs Tier 1 (some legacy tooling that
can't handle encrypted secrets), they should do a fresh install of an
older Invenzo version.

---

## 9. SaaS path

When Invenzo becomes a SaaS product, secrets move to a remote provider
(HashiCorp Vault, AWS Secrets Manager, Azure Key Vault). The
application-side code is already ready — stubs exist in
[api/app/secret_provider.py](../api/app/secret_provider.py):

```python
def _read_from_vault(name: str) -> str | None:
    raise NotImplementedError(...)

def _read_from_aws_sm(name: str) -> str | None:
    raise NotImplementedError(...)

def _read_from_azure_kv(name: str) -> str | None:
    raise NotImplementedError(...)
```

SaaS day-1 work:

1. Implement the three stubs (~30 lines each using native SDK)
2. Add `SECRETS_PROVIDER=vault|aws_sm|azure_kv` env var dispatch in `read_secret()`
3. Add a startup hook for provider authentication (AppRole for Vault,
   IAM role for AWS, Managed Identity for Azure)
4. Add in-memory TTL cache (~30s) so each request doesn't hit the
   provider's API
5. Optional: multi-tenancy — add `tenant_id` contextvar + per-tenant secret paths

Every existing call site in the app (`backup_service.py`,
`vault_service.py`, all discovery task files, `config.py`, `celery_app.py`,
`pinger.py`) continues to work without changes — `read_secret()` is the
single integration point.

---

## 10. What was NOT built and why

Explicit non-goals, each with reasoning:

### No TPM binding for `/etc/invenzo/age.key`

**What it would be:** seal the age private key to the host TPM so a
disk clone of the key file by itself can't be used — the TPM has to be
physically present and in the expected state (PCR measurements).

**Why not:** portability. Invenzo ships to customer-managed hosts with
wildly inconsistent TPM support. VMs often don't have a usable TPM;
cloud-provider TPMs have different programming models. A host-file key
works uniformly on every target environment.

**Chosen alternative:** strict permission (mode 400, root:root) on the
key file + operational guidance to back it up offline. Acceptable
because the threat model is "offline disk copy / backup tarball leaks",
not "root-level host compromise".

**If a customer needs TPM** they write a thin Dockerfile on top of
`invenzo-api` that sets up a `systemd-creds encrypt --tpm2` path and
points `AGE_KEY_FILE` at the decrypted output. No app changes required.

### No auto-rotation schedule

**What it would be:** a Celery beat job that rotates the age key or
individual secrets every N days.

**Why not:** rotation carries operational risk (a mid-rotation crash
leaving half-old/half-new state requires manual repair). Compliance
regimes (HIPAA, SOC 2, ISO 27001) require manual, logged rotation —
not silent automation. Admins should rotate on their policy's schedule,
with an audit trail.

**Chosen alternative:** documented one-shot rotation commands in
[Section 6](#6-operations). Admins invoke them from change-management.

**VAULT_KEY** specifically has a different rotation story — the
`VAULT_KEY_OLD` dual-key window in `credential_service.py` rotates
stored credentials transparently as they're accessed.

### No standalone `migrate-to-encrypted.sh` script

**What it would be:** dedicated script customers run during upgrade to
convert env-mode `.env` secrets to `./secrets/*.enc`.

**Why not:** an inline 30-line bash block in
[UPGRADE.md](UPGRADE.md) does the job with less surface area.
A standalone script would need its own error handling, logging,
rollback, `--dry-run` support — hundreds of lines for an operation each
customer runs once. The inline block is easier to audit (customers
paste it into a shell and read what it does) and easier to adapt to
non-standard install paths.

**When this changes:** if >50 support tickets accumulate about migration
failures, a formal script becomes worth the surface area.

### No CI test coverage for the install-script flow

**What it would be:** GitHub Actions job that spins up a Linux container,
runs `bash install.sh`, confirms the stack reaches healthy, reports
pass/fail.

**Why not:** the script is heavily interactive (prompts for admin
password, hostname, confirmations), assumes root + Docker on the host,
and needs a realistic matrix (Ubuntu 22.04 / Debian 12 / RHEL 9, with
Docker 24 / 25, Podman / Docker). This isn't well-served by standard
GitHub Actions runners. The 26 in-container tests already cover the
`read_secret()` + preflight contract — which is where real bugs hide.
The install script itself is bash glue; `bash -n install.sh` +
`docker compose config --quiet` + one manual test run before each
release is proportional coverage.

**When this changes:** if the project gains a dedicated CI fleet with
reliable privileged-container support, a smoke test becomes worthwhile.

### Other explicit non-goals

- **No secret-access audit log.** No "who read VAULT_KEY at timestamp X"
  trail. OpenBao in production mode fills that gap — see the SaaS path.
- **No Windows dev parity for the master-key path.** Dev on Windows
  keeps the age key in the repo root (`.age-key`, gitignored). Linux
  prod uses `/etc/invenzo/age.key`. The `AGE_KEY_FILE` env var bridges
  both.
- **No backup-bundle encryption.** `backup_service.py` pg_dumps are NOT
  age-encrypted by Invenzo. That's the admin's choice — pipe through
  `age -e` or store on encrypted external media.

---

## 11. FAQ

**Q: Do I need to commit `./secrets/*.enc` to git?**
A: No. They're already in `.gitignore`. Each install generates its own
blobs tied to its own age key — committing them would be useless anyway
since the key is per-install.

**Q: What happens if I re-run `install.sh` on an existing install?**
A: `install.sh` checks if `/etc/invenzo/age.key` exists. If yes, it
REUSES it (doesn't regenerate). If no, it generates a fresh keypair.
This is critical for idempotent installs.

**Q: Can I use my own age keypair instead of install.sh's generated one?**
A: Yes. Before running install.sh, put your private key at
`/etc/invenzo/age.key` (mode 400, root:root). install.sh detects it and
skips key generation. Your public identity is derived automatically.

**Q: What if I lose both `/etc/invenzo/age.key` AND all my backups?**
A: Run `recover-lost-age-key.sh` (see [Section 7](#7-recovery-from-a-lost-age-key)).
It regenerates everything and preserves DB data. You lose only the
*stored scan credential passwords*, which you re-enter via the UI.

**Q: Does encryption slow down boot?**
A: Decryption of 8 small files takes ~50ms total per service. The `apk
add age` step in vendor-image entrypoints (postgres, redis, etc.) takes
~2 seconds on first boot per container, then the package cache survives
until the image is rebuilt. End-to-end, boot is ~3-5 seconds slower
than env mode, which is invisible next to postgres and Alembic startup.

**Q: What goes into `docker inspect` output now?**
A: Environment variables — but only non-secret ones. `INVENZO_VERSION`,
`POSTGRES_USER`, `REDIS_HOST`, `VAULT_ADDR`, etc. No secret values, no
passwords, no keys. Anyone with access to the Docker socket (i.e. root
on the host) could always see everything anyway.

**Q: Can I run encrypted mode in dev on Windows?**
A: Yes. The dev `docker-compose.yml` at the repo root uses the same
architecture as prod. The age key lives at `.age-key` in the repo root
(gitignored). `AGE_KEY_FILE` env var defaults to `./.age-key`.

**Q: What if my admin runs `docker compose down -v`?**
A: That wipes named volumes, so postgres/redis/grafana/vault data is
gone. Recovery from an old age key is impossible in that case — new
install only. This is the user's responsibility; age encryption doesn't
protect against voluntary `rm -rf`.

**Q: Does the application have to know anything about encryption?**
A: No. The app only calls `read_secret("NAME")` which reads from
`$SECRETS_DIR` (the tmpfs path populated by the entrypoint wrapper at
boot). The app is oblivious to whether the secret came from a file, env
var, age-decrypted blob, or a future remote provider. This is the whole
point of the abstraction.

---

*Last updated: v1.10.0 release cycle.*
