---
layout: default
title: Invenzo ITAM — Documentation
description: Self-hosted IT Asset Management
---

# Invenzo ITAM

**Self-hosted IT Asset Management platform.**
Network discovery · Asset inventory · Software asset management · Compliance · Agent deployment · Change management.

<p>
  <a href="DEPLOYMENT_GUIDE.md" style="background:#1FCC7A;color:white;padding:10px 20px;text-decoration:none;border-radius:6px;font-weight:500;display:inline-block;margin-right:8px;">📖 Full Deployment Guide</a>
  <a href="https://github.com/raguyazhin/invenzo-package" style="background:#333;color:white;padding:10px 20px;text-decoration:none;border-radius:6px;font-weight:500;display:inline-block;">📦 Install Package</a>
</p>

---

## Two repositories, two audiences

Invenzo uses two GitHub repositories to keep source code protected while making deployment open:

| Repo | URL | Visibility | Who uses it | What it contains |
|---|---|---|---|---|
| 📦 **invenzo-package** | [github.com/raguyazhin/invenzo-package](https://github.com/raguyazhin/invenzo-package) | 🌐 Public | Customers installing / upgrading | `install.sh`, `update.sh`, `docker-compose.yml`, Caddyfile — **no source code**. Pulls pre-built Docker images from Docker Hub. |
| 🔒 **Invenzo** | [github.com/raguyazhin/Invenzo](https://github.com/raguyazhin/Invenzo) | 🔐 Private | Developers (you) | Full source code for API, UI, discovery engine, Go agent. Licensed commercial software. |

**Customers never see source code.** They just run `install.sh` from the public `invenzo-package` repo, which pulls compiled Docker images (`raguyazhin/invenzo-api`, `-discovery`, `-ui`) from Docker Hub. Python source is Cython-compiled to `.so` binaries in the production images.

---

## Pick your path

### 🚀 I want to install Invenzo (customer / production)

**New Linux server, installing for the first time:**

```bash
# 1. Clone the public install package (no source code, just deployment scripts)
git clone https://github.com/raguyazhin/invenzo-package.git /tmp/invenzo-package
cd /tmp/invenzo-package

# 2. Run the installer
sudo bash install.sh
```

The installer:
- Installs Docker + openssl if missing
- Generates strong random secrets (POSTGRES_PASSWORD, SECRET_KEY, VAULT_KEY, etc.)
- Prompts for an admin password + hostname
- Pulls the 3 Docker images from Docker Hub
- Starts all services

Open `https://<your-hostname>` in a browser after 5-10 minutes. Full details: [Customer Install](DEPLOYMENT_GUIDE.md#c-customer--fresh-install)

---

### 🔄 I want to upgrade an existing install

```bash
cd /opt/invenzo
sudo bash update.sh 1.8.3
```

Your `.env` secrets stay untouched — only the `VERSION=` line is updated. Full details: [UPGRADE.md](https://github.com/raguyazhin/invenzo-package/blob/main/UPGRADE.md)

---

### 🤖 How do endpoints get the agent?

From the **Agents page** in the Invenzo UI:

1. Log in as admin → navigate to **Agents**
2. Click **"Download Installer"** — downloads `invenzo-installer.exe` with pre-configured server URL + registration token baked in
3. Run the installer on the endpoint (double-click, or silent: `msiexec /i invenzo-agent-setup.msi /qn`)
4. The installer:
   - Installs the Go agent as a Windows service (or systemd / launchd on Linux / macOS)
   - Writes `config.yaml` with the API URL + auth token
   - Starts the service
5. Within seconds, the agent checks in — the endpoint appears on the Assets page

**No manual configuration needed on endpoints.** Everything is baked into the installer when the admin downloads it.

---

### 💻 I'm a developer (with access to the private Invenzo repo)

**First clone on a new laptop:**

```bash
git clone https://github.com/raguyazhin/Invenzo.git
cd Invenzo
bash bootstrap-dev.sh           # generates .env with random secrets + admin password
docker compose up -d --build    # builds images locally from source
```

**Daily update:**

```bash
cd Invenzo
git pull origin master
docker compose up -d --build   # rebuilds from latest source + auto-applies DB migrations
```

**Release a new version:**

```bash
bash scripts/bump-version.sh patch --all   # bump → commit → build → push to Docker Hub
```

Full details: [Developer Setup](DEPLOYMENT_GUIDE.md#a-developer--first-install)

---

## Latest release

**v1.8.3** — Printer MIB / SNMP enhancements.

Docker Hub images (pull with `docker pull raguyazhin/<name>:1.8.3`):

- [`raguyazhin/invenzo-api`](https://hub.docker.com/r/raguyazhin/invenzo-api)
- [`raguyazhin/invenzo-discovery`](https://hub.docker.com/r/raguyazhin/invenzo-discovery)
- [`raguyazhin/invenzo-ui`](https://hub.docker.com/r/raguyazhin/invenzo-ui)

---

## How do I verify what's running?

Open **Settings → Version & Schema** in the UI. It shows:

- Application version (e.g. `1.8.3`)
- DB schema version (e.g. `0055`)
- Sync status: ✅ In Sync / ⚠️ Migration Pending / ❓ Unknown
- Full migration history
- **Compare** widget to verify a customer/staging install against your local

No SSH, no SQL, no guessing.

---

## System requirements

| Resource | Minimum | Recommended |
|---|---|---|
| OS | Ubuntu 22.04 / Debian 12 / RHEL 9 | Ubuntu 22.04 LTS |
| CPU | 4 vCPU | 8 vCPU |
| RAM | 8 GB | 16 GB |
| Disk | 50 GB | 200 GB |
| Network | Outbound HTTPS (Docker Hub) | + inbound TCP 80/443 |

---

## Links

| | |
|---|---|
| 📦 Install package (public) | [github.com/raguyazhin/invenzo-package](https://github.com/raguyazhin/invenzo-package) |
| 🔒 Source code (private) | [github.com/raguyazhin/Invenzo](https://github.com/raguyazhin/Invenzo) |
| 📖 Full deployment guide | [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md) |
| 🛡️ AV / EDR exclusions (hand to customer's AV admin) | [AV_EXCLUSIONS.md](AV_EXCLUSIONS.md) |
| 🎨 UI design reference | [UI_DESIGN_GUIDE.md](UI_DESIGN_GUIDE.md) |
| 🐳 Docker Hub | [hub.docker.com/u/raguyazhin](https://hub.docker.com/u/raguyazhin) |

---

<p style="text-align:center;color:#888;font-size:12px;margin-top:40px">
  Invenzo ITAM v1.8.3 · Self-hosted IT Asset Management · Commercial License
</p>
