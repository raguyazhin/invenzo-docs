---
layout: default
title: Invenzo Agent — Antivirus / EDR Exclusions
description: Exception list for AV / EDR administrators
---

# Invenzo Agent — Antivirus / EDR Exclusions

**Audience:** Security / antivirus administrators at organisations deploying the Invenzo ITAM agent.
**Purpose:** One-page reference listing the paths, processes, and services that must be excluded from real-time scanning and behaviour-detection heuristics so the Invenzo agent can install and run.

---

## Why exclusions are needed

The Invenzo agent is a legitimate IT Asset Management tool that inventories hardware, software, and security posture on the endpoint. Its Windows installer runs a short process chain during install:

```
msiexec.exe  →  cmd.exe  →  install.bat  →  invenzo-agent.exe install-service
```

This process lineage (system-level `LocalSystem` spawning `cmd → powershell → service-install`) is also used by malware droppers, so **generic behaviour-detection heuristics** in many AV / EDR products flag it. Examples:

| Product | Signature reported |
|---|---|
| Seqrite Endpoint Protection | `Behavior Detection System Detected Malicious Activity : 944` |
| Microsoft Defender | `Behavior:Win32/Generic` / SmartScreen warning |
| CrowdStrike Falcon | `Suspicious Process Lineage` IOA |
| SentinelOne | `Suspicious Script Execution` behaviour rule |
| Trend Micro Apex One | `Suspicious Connection / Process Chain` |

**None of these are signatures on the Invenzo binary itself** — they are heuristic rules matching the install *pattern*. The binary is an unmodified Go 1.22 static executable with no packing, no obfuscation, no injection.

Adding the exclusions below silences these heuristics for the specific Invenzo install paths and processes, without weakening protection elsewhere on the endpoint.

---

## Which agent variant am I whitelisting?

Invenzo ships two agent variants. Both install to the **same path** (`C:\ProgramData\InvenzoAgent\`) and run as the **same service** (`InvenzoAgent`) — so **the exclusion list below works for both**. The only difference is the binary filename you'll see on the wire during push-deploy, and the SHA256 hash.

| Variant | Installs on | Binary filename (before install) | Purpose |
|---|---|---|---|
| **Standard agent** | Regular endpoints — laptops, desktops, servers, VMs | `invenzo-agent-windows-amd64.exe` | Hardware / software / security inventory |
| **Session Host agent** | VDI host servers only — NComputing vSpace, Microsoft RDS, Citrix XenApp / Virtual Apps, VMware Horizon | `invenzo-agent-sessionhost-windows-amd64.exe` | Everything the standard agent does **plus** enumerates active VDI sessions and reports connected thin clients as child assets |

**Thin clients themselves do NOT run an Invenzo agent.** They have no local inventory collector. Their existence is discovered indirectly by the Session Host agent on the VDI host they connect to. So no AV exclusions are required on thin-client devices — only on the VDI host servers.

Once either variant is installed, the on-disk binary path is always `C:\ProgramData\InvenzoAgent\invenzo-agent.exe`. The exclusions in the next section cover both.

---

## Full exclusion list

Apply all of the following on every Windows endpoint that will receive the Invenzo agent.

### 1. Folder exclusions (real-time scan)

| Path | Purpose |
|---|---|
| `C:\ProgramData\InvenzoAgent\` | Agent install root — binary, config, install scripts |
| `C:\ProgramData\InvenzoAgent\logs\` | Agent runtime logs |
| `C:\Windows\Installer\` | MSI cache (often already system-excluded) |

### 2. Process exclusions (behaviour detection / HIPS)

| Process | Purpose |
|---|---|
| `C:\ProgramData\InvenzoAgent\invenzo-agent.exe` | The agent service binary (runs continuously as `LocalSystem`) |
| `C:\ProgramData\InvenzoAgent\install.bat` | First-time install orchestrator (run once per endpoint) |
| `C:\ProgramData\InvenzoAgent\uninstall.bat` | Uninstall orchestrator |
| `C:\Windows\System32\msiexec.exe` | Windows Installer (only during the install window) |

### 3. File-name exclusions (for MSI-extracted temp copies)

| File | Purpose |
|---|---|
| `invenzo-agent-setup.msi` | The installer package (extracted to `%TEMP%` during install) |
| `invenzo-agent.exe` | The agent binary (may be scanned in `%TEMP%` during MSI extraction) |
| `install.bat` | Install script |

### 4. Service exclusion

| Service name | Purpose |
|---|---|
| `InvenzoAgent` | The running Windows service |

### 5. Network exclusion (firewall / web-control)

| Destination | Port | Purpose |
|---|---|---|
| Your Invenzo server URL (e.g. `invenzo.company.com`) | 443 (HTTPS) | Agent check-in, data upload, self-update downloads |

The agent never makes outbound connections to any other destination. All traffic goes to the customer's self-hosted Invenzo server — no calls to vendor-hosted cloud services.

---

## Vendor-specific steps

---

## 🛡️ Hand-off: Seqrite Endpoint Protection (Quick Heal)

**For the Seqrite administrator — this section is self-contained, you don't need to read the rest of the document.**

### Where to add the exception

**Seqrite EPP Server Console → Clients → Manage Policies → [Your Policy] → Edit**

Add the entries below in **two** places inside the policy (both are required — one stops real-time scan from quarantining the binary, the other stops behavior-detection from flagging the install):

1. **Advanced DNAScan → Exclude Files & Folders**
2. **Behavior Detection System → Exclusions** ← this is what fires signature **944**

### Folders to exclude

```
C:\ProgramData\InvenzoAgent\
C:\ProgramData\InvenzoAgent\logs\
```

### Processes to exclude

```
C:\ProgramData\InvenzoAgent\invenzo-agent.exe
C:\ProgramData\InvenzoAgent\install.bat
C:\ProgramData\InvenzoAgent\uninstall.bat
C:\Windows\System32\msiexec.exe
```

### Files to exclude (by name — for MSI temp extraction)

```
invenzo-agent-setup.msi
invenzo-agent.exe
install.bat
```

### Network / firewall

Allow outbound **HTTPS (port 443)** from endpoints to the Invenzo server URL (the IT team running Invenzo will give you this — e.g. `invenzo.company.com`). No other outbound destinations are needed.

### Why Seqrite flags it

Seqrite's **Behavior Detection System signature 944** is a heuristic rule that fires when `NT AUTHORITY\SYSTEM` spawns this process chain:

```
powershell.exe  →  cmd.exe  →  install.bat  →  invenzo-agent.exe install-service
```

This is the same pattern malware droppers use, which is why the heuristic is sensitive — but it's not matching a signature on the Invenzo binary. It's matching the **install behavior**. The exclusions above tell Seqrite to skip the heuristic for these specific files only; everything else on the endpoint remains fully protected.

### Deploy the policy

1. Save the policy in the EPP console.
2. Push / apply to all client groups that will receive the Invenzo agent.
3. Wait ~5 minutes for endpoints to pull the updated policy (check **Client Status** in the console to confirm the policy version has reached each endpoint).
4. Ask the Invenzo administrator to retry the agent install on one endpoint as a test before full rollout.

### (Optional) Submit as false positive to Quick Heal

To get a permanent global whitelist for this binary so future Invenzo releases don't need re-whitelisting:

- Upload `invenzo-agent-windows-amd64.exe` and `invenzo-agent-setup.msi` to **https://submit.quickheal.com/**
- Category: **False Positive**
- Ask the Invenzo admin for the current release's SHA256 hash to include in the submission.

---

## 🛡️ Hand-off: Microsoft Defender Antivirus

**For the Defender / Intune / Group Policy administrator — this section is self-contained.**

Pick the deployment method that matches your environment:

### Option A — Group Policy (AD-joined machines)

1. Open **Group Policy Management Editor** → select the GPO applied to your endpoint OU.
2. Navigate: **Computer Configuration → Administrative Templates → Windows Components → Microsoft Defender Antivirus → Exclusions**.
3. Enable and populate each of the three exclusion policies:

**Path Exclusions** — add these entries (value column = `0` for each):
```
C:\ProgramData\InvenzoAgent
C:\ProgramData\InvenzoAgent\logs
```

**Process Exclusions** — add these entries (value column = `0` for each):
```
invenzo-agent.exe
install.bat
uninstall.bat
msiexec.exe
```

**Extension Exclusions** — not required (leave alone).

4. Run `gpupdate /force` on one target endpoint to pull the policy immediately, then retry the agent install as a test.

### Option B — Microsoft Intune (MDM-managed machines)

1. **Microsoft Intune admin center → Endpoint security → Antivirus → Create Policy**.
2. Platform: **Windows 10 and later** → Profile: **Microsoft Defender Antivirus Exclusions**.
3. Add the same values as the Group Policy option above under **Defender Processes to exclude** and **Paths to exclude**.
4. Assign the profile to the device group that will receive the Invenzo agent.
5. On a test endpoint, run `Company Portal app → Sync` (or wait ~8 hours for automatic sync), then retry the install.

### Option C — PowerShell (single-machine test / ad-hoc)

Run as Administrator on the target endpoint:

```powershell
Add-MpPreference -ExclusionPath 'C:\ProgramData\InvenzoAgent'
Add-MpPreference -ExclusionPath 'C:\ProgramData\InvenzoAgent\logs'
Add-MpPreference -ExclusionProcess 'invenzo-agent.exe','install.bat','uninstall.bat','msiexec.exe'

# Verify exclusions are applied:
Get-MpPreference | Select-Object -ExpandProperty ExclusionPath
Get-MpPreference | Select-Object -ExpandProperty ExclusionProcess
```

Note: local PowerShell exclusions are **overridden by Group Policy / Intune** if those are also in effect — for production rollout use Option A or B, not this.

### Option D — Microsoft Defender for Endpoint (EDR — separate from Defender AV)

If MDE is active (visible in **security.microsoft.com**):

1. **Endpoints → Configuration management → Endpoint security policies → Attack surface reduction rules**.
2. Add **File / folder exclusions** for the paths above.
3. Under **Device groups**, ensure the policy applies to endpoints receiving Invenzo.

### Network / firewall

Allow outbound **HTTPS (443)** to the Invenzo server URL. If Defender SmartScreen is active, add the URL to the trusted sites list in **Windows Security → App & browser control → Reputation-based protection settings**.

### Why Defender flags it

Unsigned binaries executing a `LocalSystem → cmd → powershell → service-install` chain match Defender's generic **Behavior:Win32/Generic** heuristic. Defender also shows a SmartScreen warning on first execution of any unsigned binary. The exclusions above disable both for the Invenzo install paths specifically. If the Invenzo team code-signs the binary in a future release (with a valid Authenticode certificate), these exclusions are no longer required — Defender auto-trusts signed binaries with established reputation.

### Submit false positive to Microsoft

For permanent global whitelisting so all your endpoints stop flagging future releases:
- https://www.microsoft.com/wdsi/filesubmission
- Submission type: **Software developer**
- Select **Incorrect detection** and upload both `invenzo-agent-windows-amd64.exe` and `invenzo-agent-setup.msi`.

---

## Other AV / EDR vendors

The sections below are condensed — the folder / process / file exclusion values are the same as listed in [Full exclusion list](#full-exclusion-list) above; only the console navigation differs.

### CrowdStrike Falcon

1. **Falcon Console → Configuration → Prevention Policies → [Policy] → Exclusions**.
2. Add **Sensor Visibility Exclusions** for the folder paths (section 1).
3. Add **Machine Learning Exclusions** for the process paths (section 2).
4. For persistent IOA flags, add an **IOA Exclusion** with pattern matching the process command line (`invenzo-agent.exe install-service`).

### SentinelOne

1. **Sentinels → Exclusions → Add Exclusion**.
2. Type: **Path** — add folder `C:\ProgramData\InvenzoAgent\` with scope "Subfolders".
3. Type: **Path** — add processes from section 2.
4. Optionally add **Hash** exclusions by uploading the SHA256 of the agent binary — ask your Invenzo administrator for the current release hash.

### Sophos Central

1. **Global Settings → Global Exclusions → Add**.
2. Type: **Folder** — add `C:\ProgramData\InvenzoAgent\`.
3. Type: **Process** — add processes from section 2.

### Trend Micro Apex One

1. **Agent Management → Policies → [Policy] → Scan Exclusions**.
2. Add real-time scan exclusions (section 1).
3. Under **Behavior Monitoring → Exception List**, add process exclusions (section 2).

### Kaspersky Endpoint Security

1. **Policy → General protection settings → Exclusions and trusted zone**.
2. Add the folder from section 1 as a **Trusted zone** entry.
3. Add processes from section 2 as **Trusted applications** with all monitoring checkboxes disabled for those paths.

### Bitdefender GravityZone

1. **Policies → [Policy] → Antimalware → Settings → Exclusions**.
2. Add **Folder** exclusions (section 1) and **Process** exclusions (section 2).
3. Under **Advanced Anti-Exploit**, add the same processes to the exclusion list.

### McAfee Trellix ENS

1. **ePO Console → Policy Catalog → On-Access Scan → Exclusions**.
2. Add folder and process exclusions from sections 1–2.
3. Under **Threat Prevention → Exploit Prevention → Exclusions**, add the agent binary process.

---

## Verifying exclusions are active

After applying exclusions, verify on one endpoint before rolling out:

```powershell
# 1. Check Defender exclusions (if Defender is active)
Get-MpPreference | Select ExclusionPath, ExclusionProcess

# 2. Run the agent installer
# Right-click install.bat → Run as administrator

# 3. Confirm the service is running
Get-Service InvenzoAgent

# 4. Confirm the agent binary is not quarantined
Test-Path 'C:\ProgramData\InvenzoAgent\invenzo-agent.exe'
# Should return: True

# 5. Confirm outbound connectivity to Invenzo server
Test-NetConnection -ComputerName invenzo.company.com -Port 443
```

If any of the above fail, re-check the AV console event log for the specific block event — the product name and rule ID in the log tell you exactly which exclusion is missing.

---

## For the Invenzo administrator

If a customer's AV team asks for the SHA256 hashes of the agent binaries to whitelist by hash (strongest exclusion — survives folder moves):

1. On the Invenzo server: `docker compose exec api sha256sum /app/agent-binaries/invenzo-agent-windows-amd64.exe`
2. Or locally on a Windows endpoint that already has the agent installed:
   ```powershell
   Get-FileHash 'C:\ProgramData\InvenzoAgent\invenzo-agent.exe' -Algorithm SHA256
   ```

Hashes change on every release. If the customer uses hash-based exclusions, you'll need to publish the new SHA256 on each version bump and they'll need to update their policy — path + process exclusions (recommended) don't need maintenance across releases.

---

## Security assurance for the AV team

The following may help your AV administrator approve the exclusions:

- **Source**: the Invenzo agent is a standalone compiled Go binary. No interpreter, no scripting engine, no plugin system — the attack surface is the code that ships in the binary.
- **Privileges**: runs as `LocalSystem` Windows service for hardware inventory access (read-only WMI, read-only file system traversal). Does not execute remote commands or accept inbound connections.
- **Network**: outbound HTTPS to the customer's self-hosted Invenzo server *only*. No telemetry to vendor-hosted services.
- **Data collected**: hardware specs, installed software list, running services, security posture (AV product name + signature date + firewall state), performance counters. No file contents, no keystrokes, no screenshots, no browser history.
- **Code signing** (newer releases): signed with a valid Authenticode certificate from {{ certificate authority }}. Verify with `Get-AuthenticodeSignature C:\ProgramData\InvenzoAgent\invenzo-agent.exe`.
- **Licence**: Commercial software governed by the Invenzo End-User License Agreement.

---

## Getting help

- **Invenzo admin** (the person who gave you this document): ask them to submit the binary to your AV vendor as a false positive — this whitelists by hash globally so future endpoints at other customers with the same AV don't hit the same issue.
- **Documentation**: https://raguyazhin.github.io/invenzo-docs/
- **Agent deployment guide**: [DEPLOYMENT_GUIDE.md — Agent Deployment](DEPLOYMENT_GUIDE.md#g-agent-deployment)

---

*This document targets Invenzo Agent v1.9.x. Paths and process names are stable across releases — only SHA256 hashes change on version bumps.*
