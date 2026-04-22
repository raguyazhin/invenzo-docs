# Invenzo ITAM — Deployment Guide

Everything you need to install, update, or develop Invenzo — on any machine.

**Public docs**: https://raguyazhin.github.io/invenzo-docs/

---

## Two-repository model

| Purpose | Repo | URL | Who |
|---|---|---|---|
| 📦 **Install / upgrade package** | `invenzo-package` (public) | https://github.com/raguyazhin/invenzo-package | Customers |
| 🔒 **Source code** | `Invenzo` (private) | https://github.com/raguyazhin/Invenzo | Developers |
| 📖 **Documentation** | `invenzo-docs` (public, GitHub Pages) | https://raguyazhin.github.io/invenzo-docs/ | Everyone |

Customers **never** need access to the private source repo — they pull pre-built, Cython-compiled Docker images directly from Docker Hub (`raguyazhin/invenzo-api`, `-discovery`, `-ui`).

---

## Quick decision tree

| I want to... | See section |
|---|---|
| Install Invenzo on a new server (customer / production) | [C. Customer — Fresh Install](#c-customer--fresh-install) |
| Upgrade an existing install | [D. Customer — Update Existing Install](#d-customer--update-existing-install) |
| Deploy agents to endpoints | [G. Agent Deployment](#g-agent-deployment) |
| Develop — first clone on a new laptop | [A. Developer — First Install](#a-developer--first-install) |
| Develop — pull latest code onto an existing dev setup | [B. Developer — Update to Latest](#b-developer--update-to-latest) |
| Publish a new version | [E. Releasing a New Version](#e-releasing-a-new-version) |
| Fix something | [F. Troubleshooting](#f-troubleshooting) |

---

## A. Developer — First Install

**Prerequisites**: Docker Desktop 4.26+ (or Docker Engine 24+), git, openssl, bash, **access to private `Invenzo` repo**.

```bash
# 1. Clone the private source repo
git clone https://github.com/raguyazhin/Invenzo.git d:/Invenzo
cd d:/Invenzo

# 2. Bootstrap: generates age keypair (./.age-key), encrypts 8 secrets
#    to ./secrets/*.enc, writes .env with non-secret config only
bash bootstrap-dev.sh

# 3. Build and start the full stack (3-8 minutes first time)
#    Each service decrypts its own secrets into /dev/shm at container boot
docker compose up -d --build

# 4. Open http://localhost:3080, log in as admin
```

**Verify**: `docker compose ps` — all services should show `Up (healthy)`.

---

## B. Developer — Update to Latest

```bash
cd d:/Invenzo
git pull origin master
docker compose up -d --build    # auto-runs Alembic migrations
```

Hard-refresh browser (**Ctrl+Shift+R**) to clear the PWA service worker cache.

**Verify**: Open **Settings → Version & Schema** — should show ✅ **In Sync** banner.

If version shows `dev` instead of a number, sync `.env`:

```bash
CURRENT=$(grep -oP '"version":\s*"\K[^"]+' ui/package.json)
bash scripts/bump-version.sh $CURRENT
docker compose up -d --build api
```

---

## C. Customer — Fresh Install

**Prerequisites**: Fresh Linux host (Ubuntu 22.04 LTS / Debian 12 / RHEL 9 / Rocky 9), root or sudo access, outbound internet (Docker Hub).

### Online install (recommended)

```bash
# 1. Clone the PUBLIC install package (no source code, just deployment scripts)
cd /tmp
git clone https://github.com/raguyazhin/invenzo-package.git
cd invenzo-package

# 2. Run the installer
sudo bash install.sh
```

The installer does 10 steps automatically:

1. Pre-flight checks (root, OS, RAM, disk)
2. Installs `curl`, `openssl`, `ca-certificates` if missing
3. Installs Docker Engine + compose plugin if missing
4. Verifies ports 80/443 are free
5. Copies config to `/opt/invenzo/`
6. Generates random secrets + prompts for admin password, email, hostname
7. Writes `/opt/invenzo/.env` (mode 600)
8. Pulls all Docker images from Docker Hub
9. Runs `docker compose up -d`
10. Waits for health checks

Open `https://<your-hostname>` and log in as the admin you created.

### Offline / air-gapped install

On an internet-connected machine:

```bash
# Pull and save images
for img in invenzo-api invenzo-discovery invenzo-ui; do
  docker pull raguyazhin/$img:1.8.3
  docker save raguyazhin/$img:1.8.3 | gzip > $img-1.8.3.tar.gz
done

# Download the package
git clone https://github.com/raguyazhin/invenzo-package.git
tar czf invenzo-package.tar.gz invenzo-package/
```

Transfer to air-gapped server, then:

```bash
tar xzf invenzo-package.tar.gz
cd invenzo-package

# Load images
docker load < /path/to/invenzo-api-1.8.3.tar.gz
docker load < /path/to/invenzo-discovery-1.8.3.tar.gz
docker load < /path/to/invenzo-ui-1.8.3.tar.gz

# Install in offline mode
sudo bash install.sh --offline
```

---

## D. Customer — Update Existing Install

```bash
cd /opt/invenzo
sudo bash update.sh 1.8.3
```

**What `update.sh` does**:

1. Creates pre-upgrade backup in `/opt/invenzo/backups/`
2. Pulls new Docker images
3. Drains Celery worker (waits for active tasks)
4. Stops services
5. Updates `VERSION=` in `.env` (**only that line**)
6. Starts services → API auto-applies Alembic migrations
7. Health check

**Your secrets and keys are never touched**:

| Artifact | Modified by `update.sh`? |
|---|---|
| `.env` `VERSION=` line | ✅ Yes |
| `.env` non-secret config (REGISTRY, ALLOWED_HOST, ports, AGE_KEY_FILE, …) | ❌ Never |
| `./secrets/*.enc` (age-encrypted secret values) | ❌ Never |
| `/etc/invenzo/age.key` (master private key) | ❌ Never |
| `./secrets/` file permissions (mode 400, root:root) | ❌ Never |

**Encrypted-secrets mode (v1.10+):** secrets live as age-encrypted `.enc`
files in `./secrets/` — not as plaintext in `.env`. `update.sh` swaps the
Docker image version but leaves secret blobs, the age private key, and
all non-secret `.env` config alone. See
[SECRETS.md](SECRETS.md) for the full architecture.

**Upgrading from an older install still using plaintext `.env` secrets?**
Run the one-shot migration block at the top of
[UPGRADE.md](https://github.com/raguyazhin/invenzo-package/blob/main/UPGRADE.md)
BEFORE `update.sh`. It converts each `.env` secret to `./secrets/*.enc`
and generates the age keypair at `/etc/invenzo/age.key`.

### Post-upgrade verification

1. Hard-refresh browser (**Ctrl+Shift+R**)
2. **Settings → Version & Schema** — confirm ✅ **In Sync**
3. Spot-check dashboard, assets list

If ⚠️ **Migration Pending** appears: `docker compose -p invenzo restart api`

### Rollback

```bash
cd /opt/invenzo
docker compose -p invenzo stop api worker worker-beat pinger

# Restore pre-upgrade backup
gunzip -c backups/invenzo_pre_update_YYYYMMDD_HHMMSS.sql.gz \
  | docker exec -i $(docker compose -p invenzo ps -q postgres) \
    psql -U invenzo -d invenzo

sudo sed -i 's/^VERSION=.*/VERSION=<previous-version>/' .env
docker compose -p invenzo pull && docker compose -p invenzo up -d
```

---

## E. Releasing a New Version

**One-command release:**

```bash
cd d:/Invenzo
bash scripts/bump-version.sh patch --all
```

This does 4 things:

1. Bumps version in `ui/package.json`, `.env`, `invenzo-package/.env.example`, `install.sh`
2. Git commits the bump
3. Builds all 3 Docker images with `--build-arg INVENZO_VERSION=<new>`
4. Pushes `<new>` + `latest` tags to Docker Hub

### Other bump modes

```bash
bash scripts/bump-version.sh 1.9.0            # Explicit version
bash scripts/bump-version.sh minor            # 1.8.3 → 1.9.0
bash scripts/bump-version.sh major            # 1.8.3 → 2.0.0
bash scripts/bump-version.sh 2.0.0-rc1        # Pre-release
```

### After releasing

1. **Tag the private repo**:
   ```bash
   git tag -a v1.8.4 -m "Release 1.8.4"
   git push origin v1.8.4
   ```

2. **Sync the public `invenzo-package` repo**:
   ```bash
   # In a separate checkout of invenzo-package:
   cd /tmp/invenzo-package
   cp /path/to/Invenzo/invenzo-package/.env.example .
   cp /path/to/Invenzo/invenzo-package/install.sh .
   cp /path/to/Invenzo/invenzo-package/update.sh .
   cp /path/to/Invenzo/invenzo-package/docker-compose.yml .
   cp /path/to/Invenzo/invenzo-package/UPGRADE.md .
   git add -A && git commit -m "Release 1.8.4"
   git push
   git tag -a v1.8.4 -m "Release 1.8.4"
   git push origin v1.8.4
   ```

3. **Create a GitHub Release** on the public `invenzo-package` repo with release notes.

### Code-signing the Windows agent (recommended per release)

Signing the agent binary + MSI drastically reduces antivirus false-positives. The Makefile has signing wired in — enable it by passing `SIGN_CERT`.

**One-time setup**:

- Buy an OV or EV code-signing certificate (DigiCert, Sectigo, SSL.com ~$200-400/yr). EV certs get instant SmartScreen reputation; OV certs build reputation over ~1000 installs.
- Receive `.pfx` file + password (OV) or USB token / HSM slot + SHA1 thumbprint (EV).
- Install Windows SDK (includes `signtool.exe`) on the release machine. On Linux/macOS, `apt install osslsigncode` works for `.pfx` files but not EV tokens.

**Per release** — build signed agent binaries:

```bash
cd agent

# Option A: .pfx file (OV certs)
make build-all-standard SIGN_CERT=/path/to/codesign.pfx SIGN_CERT_PASSWORD=xxx
make build-all-sessionhost SIGN_CERT=/path/to/codesign.pfx SIGN_CERT_PASSWORD=xxx

# Option B: certificate thumbprint from Windows cert store (EV tokens)
make build-all-standard SIGN_CERT=A1B2C3D4E5F6...
make build-all-sessionhost SIGN_CERT=A1B2C3D4E5F6...

# Copy signed binaries into the release directory
cp agent/dist/invenzo-agent-*-amd64.exe agent-binaries/
cp agent/dist/invenzo-agent-*-arm64.exe agent-binaries/ 2>/dev/null || true
```

**Also sign the MSI** (after rebuilding with WiX):

```bash
# From the agent directory, after light.exe produces invenzo-agent-setup.msi
make sign-windows TARGET=../agent-binaries/invenzo-agent-setup.msi \
    SIGN_CERT=/path/to/codesign.pfx SIGN_CERT_PASSWORD=xxx
```

**Verify the signature**:

```powershell
# On any Windows box:
Get-AuthenticodeSignature .\invenzo-agent-windows-amd64.exe | Format-List
# Status should be "Valid", SignerCertificate populated, TimeStamperCertificate populated
```

**Submit new signed builds to AV vendors** (once per release) so customers don't have to whitelist on every version bump:

- Microsoft Defender: https://www.microsoft.com/wdsi/filesubmission (select "Developer" → "Incorrect detection")
- CrowdStrike / SentinelOne / Sophos / Bitdefender / Kaspersky / McAfee: open a false-positive ticket via the customer's vendor support portal with the SHA256 of the new binary.

---

## F. Troubleshooting

### Migration Pending after update

```bash
docker compose -p invenzo restart api
# OR force:
docker compose -p invenzo exec api alembic upgrade head
```

### Sidebar/login shows old version

PWA service worker caching. Fix: hard-refresh (Ctrl+Shift+R), or DevTools → Application → Service Workers → Unregister.

### API container keeps restarting

```bash
docker compose -p invenzo logs --tail 50 api
```

Common causes: missing env var, DB not healthy, migration error.

### "dev" instead of version

`.env` missing `VERSION=` line. Fix:

```bash
echo "VERSION=1.8.3" >> .env
docker compose up -d --build api
```

### Can't pull images (customer install)

```bash
curl -sI https://hub.docker.com/ | head -1   # expect HTTP/2 200
docker pull raguyazhin/invenzo-api:1.8.3
```

### Reset everything (DEV ONLY — destroys data)

```bash
docker compose down -v
docker compose up -d --build
```

---

## G. Agent Deployment

**Zero-config for end users — admin generates a pre-configured bundle per endpoint.**

### Windows — Ready-to-Run Bundle (recommended)

**Admin side** (once per target machine):

1. Invenzo UI → **Agents** → click **Generate Install Command**
2. Enter server URL → server issues a single-use registration token
3. Click **Download Ready-to-Run ZIP**
4. Send the ZIP (`invenzo-agent-installer.zip`, ~10 MB) to the end user via email / USB / shared drive

**End-user side** (zero typing):

1. Extract the ZIP anywhere (desktop, temp folder, etc.)
2. Right-click **`install.bat`** → **Run as administrator**
3. Wait ~10 seconds — the agent registers, installs as a Windows service, starts reporting
4. Done. The endpoint appears on the Assets page of the Invenzo UI within a few seconds.

**What the ZIP contains**:

| File | Purpose |
|---|---|
| `invenzo-agent-setup.msi` | The Windows installer package |
| `install.bat` | Elevates to Administrator + runs msiexec with baked-in URL and token |
| `install.ps1` | Same workflow in PowerShell (alternative) |
| `README.txt` | Plain-text instructions for the end user |

**Security**: each bundle contains a single-use registration token. Once the token is consumed during install, it's marked invalid on the server. Admin must generate a new bundle for each additional endpoint.

### Windows — Advanced (raw MSI + manual command)

For scripted / MDM deployment where you want to run the MSI directly:

1. UI → **Agents** → **Generate Install Command** → expand **"Advanced"** section
2. Download `invenzo-agent-setup.msi`
3. Run silently on the endpoint:
   ```cmd
   msiexec /i invenzo-agent-setup.msi APIURL=https://invenzo.company.com TOKEN=<TOKEN> /qn
   ```

### Linux

1. UI → **Agents** → **Deploy Agent** → enter target IP + SSH credentials → Deploy
2. Invenzo pushes the agent binary over SSH, registers it as a `systemd` service
3. OR manually: download `invenzo-agent-linux-amd64` from the UI, copy to endpoint, run:
   ```bash
   sudo ./invenzo-agent install --server https://<invenzo-url> --token <TOKEN>
   sudo systemctl start invenzo-agent
   ```

### macOS

Same as Linux but uses `launchd`. Binary is `invenzo-agent-darwin-amd64` / `invenzo-agent-darwin-arm64`.

### Linux

1. **Agents page** → **Deploy Agent** → enter target IP + SSH credentials → Deploy
2. Invenzo pushes the agent binary over SSH, registers it as a `systemd` service
3. OR manually: download `invenzo-agent-linux-amd64` from the UI, copy to endpoint, run:
   ```bash
   sudo ./invenzo-agent install --server https://<invenzo-url> --token <TOKEN>
   sudo systemctl start invenzo-agent
   ```

### macOS

Same as Linux but uses `launchd`. The binary is `invenzo-agent-darwin-amd64` / `invenzo-agent-darwin-arm64`.

### What the agent collects

- Hardware (CPU, RAM, disk, network adapters, serials)
- Installed software (OS packages + applications)
- Running services / processes
- Security posture (AV status, firewall, encryption)
- Performance metrics (CPU / memory / disk usage)

All data pushed to Invenzo server on configurable interval (default: 15 minutes).

### Agent self-update

Agents check their own version on every check-in. If the server has pushed a newer binary, the agent downloads it, restarts the service, and resumes reporting — no admin intervention needed.

### Antivirus / EDR exclusions

> **Hand-off doc for customer AV teams**: [AV_EXCLUSIONS.md](AV_EXCLUSIONS.md) — standalone page with full vendor-by-vendor exception instructions. Send this link directly to the customer's AV / security administrator.

Some antivirus and EDR products (Defender, CrowdStrike, SentinelOne, Sophos, Bitdefender, Kaspersky, Trend Micro, McAfee, Seqrite) quarantine unsigned agent binaries or flag the MSI install process. If `install.bat` fails with "access denied" / "operation was cancelled" or the `InvenzoAgent` service disappears minutes after install, the endpoint's AV is blocking it.

**Recommended path**: the shipped agent MSI is code-signed on every release (see [Releasing a New Version](#e-releasing-a-new-version)). Signed binaries build reputation automatically with most AVs, and Defender's SmartScreen honours EV certs on the first install.

**Fallback: pre-configure exclusions before deploying**. The agent installs into `C:\ProgramData\InvenzoAgent\` (not Program Files) because the MSI places it under `CommonAppDataFolder` so `LocalSystem` can write `config.yaml` and logs without UAC prompts. Paths/processes/services that must be excluded on every Windows endpoint:

| Type | Value |
|---|---|
| Path (folder) | `C:\ProgramData\InvenzoAgent\` |
| Path (folder) | `C:\ProgramData\InvenzoAgent\logs\` |
| Path (folder) | `C:\Windows\Installer\` (MSI temp files — often already allowed) |
| Path (file) | `%TEMP%\invenzo-agent-install.log` |
| Process | `C:\ProgramData\InvenzoAgent\invenzo-agent.exe` |
| Process | `C:\ProgramData\InvenzoAgent\install.bat` |
| Process | `C:\ProgramData\InvenzoAgent\uninstall.bat` |
| Process | `C:\Windows\System32\msiexec.exe` (install window only) |
| Service | `InvenzoAgent` |
| Network | Outbound HTTPS (443) to your Invenzo server URL |

**Why the behavior-detection rule fires**: AVs like Seqrite / Defender flag the chain `powershell.exe → cmd.exe → install.bat → invenzo-agent.exe install-service` as a generic heuristic ("Behavior Detection System Detected Malicious Activity" / "Suspicious process lineage"). It's not matching a signature on the binary — it's matching the *install pattern*. Exclusions on the paths + processes above disable the heuristic for those specific files.

**Pushing exclusions at scale**:

- **Microsoft Defender via Group Policy**: `Computer Configuration → Admin Templates → Windows Components → Microsoft Defender Antivirus → Exclusions` — add the paths/processes above.
- **Microsoft Defender via Intune**: Endpoint security → Antivirus → Windows Defender Antivirus exclusions profile.
- **CrowdStrike Falcon**: Configuration → Prevention Policies → ML exclusions / IOA exclusions with the paths above.
- **SentinelOne**: Sentinels → Exclusions → Path/Hash → add `C:\ProgramData\InvenzoAgent\invenzo-agent.exe`.
- **Sophos Central**: Global Settings → Global Exclusions → Path + Process.
- **Seqrite Endpoint Protection (Quick Heal)**: Seqrite EPP Server Console → Clients → Manage Policies → [policy] → **Advanced DNAScan** + **Behavior Detection System** → Exclusions. Add folder `C:\ProgramData\InvenzoAgent\` and processes `invenzo-agent.exe`, `install.bat`. The heuristic signature is usually reported as `"Behavior Detection System Detected Malicious Activity : 944"` — a generic process-chain rule that auto-skips signed binaries but needs path exclusions for unsigned builds.
- **Trend Micro Apex One**: Agent Management → Exclusions → Real-time Scan + Behavior Monitoring → add folder + processes.
- **Kaspersky Endpoint Security**: Policy → General protection settings → Exclusions and trusted zone → add folder as "Trusted".

**Submit false positives upstream** (do this once per release, per vendor the customer uses):

- Microsoft Defender: https://www.microsoft.com/wdsi/filesubmission
- CrowdStrike / SentinelOne / Sophos / Trend / McAfee / Bitdefender / Kaspersky: open a false-positive ticket through the customer's vendor support portal with the SHA256 of the offending binary.

**If the agent is already quarantined**: restore the binary from the AV's quarantine console (don't re-run `install.bat` — the MSI's "won't overwrite unversioned file" policy leaves a zombie install). Once restored, re-run `install.bat` as Administrator.

---

## Quick reference card

| Task | Command |
|---|---|
| Developer first install | `git clone <private> && bash bootstrap-dev.sh && docker compose up -d --build` |
| Developer daily update | `git pull && docker compose up -d --build` |
| Customer fresh install | `git clone https://github.com/raguyazhin/invenzo-package.git && cd invenzo-package && sudo bash install.sh` |
| Customer upgrade | `cd /opt/invenzo && sudo bash update.sh <version>` |
| Release new version | `bash scripts/bump-version.sh patch --all` |
| Verify version / schema | Open **Settings → Version & Schema** tab |
| Tail logs | `docker compose logs -f api` |
| Download agent installer | UI → Agents → **Download Installer** |

---

## Links

- **Install package (public)**: https://github.com/raguyazhin/invenzo-package
- **Source code (private)**: https://github.com/raguyazhin/Invenzo
- **Documentation (this site)**: https://raguyazhin.github.io/invenzo-docs/
- **Docker Hub**: https://hub.docker.com/u/raguyazhin

---

*Last updated: 2026-04-15 for Invenzo v1.8.3*
