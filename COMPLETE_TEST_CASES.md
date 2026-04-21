# Invenzo ITAM — Complete Test Case Suite

**Version**: 2.0 | **Date**: 2026-04-13 | **Total Test Cases**: 847

---

## Table of Contents

1. [Authentication & Login](#1-authentication--login)
2. [Dashboard](#2-dashboard)
3. [Assets](#3-assets)
4. [Asset Detail](#4-asset-detail)
5. [Asset History](#5-asset-history)
6. [Asset Import](#6-asset-import)
7. [Discovery Center](#7-discovery-center)
8. [Network Scanner](#8-network-scanner)
9. [Cloud Explorer](#9-cloud-explorer)
10. [Credentials](#10-credentials)
11. [Automation](#11-automation)
12. [Agents](#12-agents)
13. [Agent Auto-Deploy Queue](#13-agent-auto-deploy-queue)
14. [Software & Licenses](#14-software--licenses)
15. [Compliance](#15-compliance)
16. [Tasks](#16-tasks)
17. [Change Management](#17-change-management)
18. [Reports](#18-reports)
19. [Locations](#19-locations)
20. [Procurement & Contracts](#20-procurement--contracts)
21. [Budgets & Cost Center](#21-budgets--cost-center)
22. [SNMP & OUI](#22-snmp--oui)
23. [Audit Logs](#23-audit-logs)
24. [Settings](#24-settings)
25. [Users & RBAC](#25-users--rbac)
26. [Notifications](#26-notifications)
27. [Webhooks](#27-webhooks)
28. [Backup & Restore](#28-backup--restore)
29. [Plugins & Licensing](#29-plugins--licensing)
30. [Sidebar & Navigation](#30-sidebar--navigation)
31. [Cross-Cutting Concerns](#31-cross-cutting-concerns)
32. [Market Leader Comparison](#32-market-leader-comparison)

---

## 1. Authentication & Login

### 1.1 Login Page (Login.tsx)

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| AUTH-001 | Valid login | Enter valid username/password → Click "Sign In" | Redirect to Dashboard, JWT stored in localStorage | Critical |
| AUTH-002 | Invalid password | Enter valid username + wrong password → Click "Sign In" | Error toast "Invalid credentials", no redirect | Critical |
| AUTH-003 | Non-existent user | Enter fake username → Click "Sign In" | Error toast "Invalid credentials" | Critical |
| AUTH-004 | Empty fields | Leave both fields empty → Click "Sign In" | Form validation: "Username is required", "Password is required" | High |
| AUTH-005 | Empty password | Enter username only → Click "Sign In" | Validation: "Password is required" | High |
| AUTH-006 | Disabled user login | Attempt login with disabled account | Error: "Account is disabled" | Critical |
| AUTH-007 | Enter key submits | Fill form → Press Enter | Same as clicking "Sign In" | Medium |
| AUTH-008 | Password visibility toggle | Click eye icon in password field | Password toggles between masked/visible | Medium |
| AUTH-009 | Session persistence | Login → Close tab → Reopen app | User remains logged in (token valid) | High |
| AUTH-010 | Token expiry | Wait for JWT expiry → Make API call | Auto-redirect to /login, token cleared | Critical |
| AUTH-011 | Concurrent sessions | Login from browser A → Login from browser B | Both sessions active (or configurable session limit) | Medium |
| AUTH-012 | SSO login | Click SSO provider button | Redirect to IdP → callback → Dashboard | High |
| AUTH-013 | Logout | Click avatar → "Sign Out" | Token cleared, redirect to /login | Critical |
| AUTH-014 | Protected route access | Visit /assets without login | Redirect to /login | Critical |
| AUTH-015 | Install wizard redirect | Fresh install, no admin → Visit any page | Redirect to /install | Critical |

### 1.2 API Key Authentication

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| AUTH-020 | API key in header | Send request with `X-API-Key` header | Authenticated, returns data | High |
| AUTH-021 | Invalid API key | Send request with bad API key | 401 Unauthorized | High |
| AUTH-022 | Revoked API key | Use revoked key | 401 Unauthorized | High |
| AUTH-023 | API key RBAC | Use API key with limited permissions | Only permitted operations succeed | High |

---

## 2. Dashboard

### 2.1 Dashboard Page (Dashboard.tsx)

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| DASH-001 | Page loads | Navigate to Dashboard | 7 sections render: KPIs, Priority Actions, Asset Visibility, Risk & Compliance, High Risk, Lifecycle, Operations | Critical |
| DASH-002 | KPI cards display | View top summary strip | 6 KPI cards: Total Assets, Coverage, Compliance, Alerts, Warranty Risk, IP Conflicts | High |
| DASH-003 | KPI values correct | Compare KPI values with database | Numbers match actual asset/alert counts | Critical |
| DASH-004 | Priority Actions | View Priority Actions section | Clickable action items with severity colors (critical/warning/info) | High |
| DASH-005 | Priority action click | Click a priority action item | Navigates to relevant page/filter | High |
| DASH-006 | Asset Visibility charts | View section 3 | Device distribution chart, type donut, status overview render | Medium |
| DASH-007 | Risk & Compliance | View section 4 | Compliance score ring, OS compliance chart, non-compliance issues | High |
| DASH-008 | High Risk table | View when risks exist | Table shows high-risk assets | Medium |
| DASH-009 | High Risk hidden | View when no risks | Section not rendered | Medium |
| DASH-010 | Lifecycle & Value | View section 6 | Depreciation, warranty, device age, licensing charts | Medium |
| DASH-011 | Operations section | View section 7 | Agent versions, IP conflicts, recent activity | Medium |
| DASH-012 | Edit mode toggle | Click customize/edit button | Drag-and-drop grid layout activates | Low |
| DASH-013 | Auto-refresh | Wait 60 seconds | Dashboard data refreshes automatically | Medium |
| DASH-014 | Empty state | Fresh install, no assets | Appropriate empty states with CTAs | Medium |
| DASH-015 | Permission check | Viewer role views dashboard | Dashboard renders (view-only) | High |

---

## 3. Assets

### 3.1 Asset List (Assets.tsx)

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| AST-001 | Page loads | Navigate to /assets | DataTable renders with asset list | Critical |
| AST-002 | Columns displayed | View table | Asset Tag, Hostname, IP, OS, Type, Location, Status, Lifecycle columns visible | Critical |
| AST-003 | Pagination | Click page 2 / change per-page | Table updates, pagination controls work | High |
| AST-004 | Per-page selector | Change to 25/50/100 per page | Table shows correct count | High |
| AST-005 | Sort ascending | Click "Hostname" column header | Sorted A→Z, arrow indicator shown | High |
| AST-006 | Sort descending | Click same column header again | Sorted Z→A | High |
| AST-007 | General search | Type "server" in search bar | Only matching assets shown | Critical |
| AST-008 | Search space normalization | Search "In Use" | Matches lifecycle_status "in_use" | High |
| AST-009 | Search by IP | Search "192.168.1" | Assets with matching IP shown | High |
| AST-010 | Search by serial | Search serial number | Matching asset found | High |
| AST-011 | Online/Offline toggle | Click "Online" segment | Only online assets shown | High |
| AST-012 | All status toggle | Click "All" segment | All assets shown (default) | High |
| AST-013 | Hidden toggle | Click eye icon | Hidden assets revealed/hidden | High |
| AST-014 | Status dot colors | View online/offline assets | Green for online, red for offline, gray for no-IP assets | Medium |
| AST-015 | Row click | Click an asset row | Navigate to /assets/{id} detail page | Critical |

### 3.2 Inline Column Filters

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| AST-020 | Toggle filters | Click "Filters" button | Inline filter row appears below headers | High |
| AST-021 | Asset Type filter | Open Asset Type dropdown → Select "Server" | Only servers shown, exact IN() match | High |
| AST-022 | Multi-select filter | Select "Server" + "Workstation" | Both types shown | High |
| AST-023 | OS Name filter | Select "Windows" | Only Windows assets shown | High |
| AST-024 | Lifecycle filter | Select "In Use" | Only in_use assets shown | High |
| AST-025 | Location tree filter | Expand location tree → Select parent | Parent + all children locations selected | High |
| AST-026 | Location partial select | Select some children | Parent shows indeterminate checkbox | Medium |
| AST-027 | Text filter - contains | Type "web" in hostname filter | Assets with "web" anywhere in hostname | High |
| AST-028 | Text filter - starts with | Type "web%" | Assets starting with "web" | Medium |
| AST-029 | Text filter - ends with | Type "%srv" | Assets ending with "srv" | Medium |
| AST-030 | Text filter - exact | Type "=webserver01" | Only exact match | Medium |
| AST-031 | Text filter - not contains | Type "!test" | Assets NOT containing "test" | Medium |
| AST-032 | Numeric filter | Type ">75" in compliance score | Assets with score > 75 | Medium |
| AST-033 | RAM filter | Type ">8GB" in RAM filter | Assets with RAM > 8GB | Medium |
| AST-034 | AV status filter | Select "Unprotected" | Only unprotected assets shown | High |
| AST-035 | Clear all filters | Click clear/reset | All filters removed, full list shown | High |
| AST-036 | Filter combination | Set type=Server + OS=Windows + Online | Intersection of all filters | High |

### 3.3 Saved Views

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| AST-040 | Save view | Set filters → Click "Save View" → Enter name → Save | View saved, appears in dropdown | High |
| AST-041 | Apply saved view | Select view from dropdown | All filters/sort restored | High |
| AST-042 | Default view | Set a view as default → Reload page | Default view auto-applied | Medium |
| AST-043 | Edit view | Open Manage Views → Edit name/description | View updated | Medium |
| AST-044 | Update view filters | Apply view → Change filters → "Update filters from page" | View filters updated | Medium |
| AST-045 | Delete view | Open Manage Views → Delete | View removed from dropdown | Medium |
| AST-046 | Shared view | Create view with "Share with team" checked | Other users see the view | Medium |
| AST-047 | Filter summary pills | Open Manage Views → Expand view | Shows FilterSummaryPills for active col_filters | Low |
| AST-048 | Legacy view migration | Apply old-format view | Auto-migrates to col_filters format | Low |
| AST-049 | Clear view | Click "Clear" while view active | All filters/sort/status reset | Medium |

### 3.4 Create Asset (CreateAssetSlideOver)

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| AST-050 | Open create | Click "Add Asset" button | SlideOver opens (w-[520px]) | Critical |
| AST-051 | Select asset type | Choose "Server" from type dropdown | Type-specific form fields load from DB | Critical |
| AST-052 | Required fields | Submit without required fields | Validation errors on required fields | High |
| AST-053 | Asset tag auto-gen | Open form (global scope) | Tag auto-generated on mount | High |
| AST-054 | Asset tag per-location | Select location (per_location scope) | Tag generates with location prefix | High |
| AST-055 | Template fields | Type with linked templates | Template fields appear below type fields | High |
| AST-056 | Dropdown list fields | Field linked to master dropdown list | Options from dropdown list shown | Medium |
| AST-057 | Hardware JSONB fields | Fill hw:true fields (hostname, IP) | Stored in hardware JSONB column | High |
| AST-058 | Submit create | Fill all fields → Save | Asset created, toast success, SlideOver closes | Critical |
| AST-059 | Duplicate check | Create asset with existing tag | Error: "Asset tag already exists" | High |
| AST-060 | No type forms warning | No type forms configured | Warning banner links to Type Forms tab | Medium |

### 3.5 Bulk Operations

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| AST-070 | Select multiple | Check rows → Bulk action bar appears | Count shown, actions enabled | High |
| AST-071 | Bulk lifecycle change | Select assets → "Change Lifecycle" → Select "Retired" | All selected assets updated | High |
| AST-072 | Bulk location assign | Select assets → "Change Location" → Pick location | All selected moved | High |
| AST-073 | Bulk tag | Select assets → "Add Tag" → Pick tag | Tags added to all selected | Medium |
| AST-074 | Bulk delete | Select assets → "Delete" → Confirm | All selected soft-deleted | High |
| AST-075 | Bulk delete by filter | Click "Bulk Delete" → Set filters → Preview | Preview count shown | High |
| AST-076 | Bulk delete confirm | Type "DELETE" → Confirm | Filtered assets soft-deleted | High |
| AST-077 | Bulk delete empty filter | Submit with no filters | Error: "At least one filter required" | High |
| AST-078 | Bulk export | Select assets → Export CSV | CSV downloaded with selected assets | Medium |
| AST-079 | Bulk compare | Select 2-3 assets → "Compare" | Side-by-side comparison view | Medium |

### 3.6 Security Banner

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| AST-085 | Banner displays | Unprotected endpoints exist | Red security insights strip shown | High |
| AST-086 | Unprotected count click | Click unprotected endpoint count | AV status filter applied | High |
| AST-087 | Alert count link | Click open alerts count | Navigates to compliance alerts tab | High |
| AST-088 | Banner refresh | Wait 2 minutes | Banner data refreshes | Medium |
| AST-089 | Banner hidden | No alerts/unprotected | Banner not shown | Medium |

---

## 4. Asset Detail

### 4.1 Overview Tab (AssetDetail.tsx)

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| DTL-001 | Page loads | Navigate to /assets/{id} | Asset detail page renders with tabs | Critical |
| DTL-002 | 3-column grid | View overview tab | Fixed 3-col grid (xl:grid-cols-3) | High |
| DTL-003 | Identity card | View col 1 | Hostname, IP, MAC, FQDN, OS, type | Critical |
| DTL-004 | Lifecycle card | View col 1 | Lifecycle status, ownership, vendor | High |
| DTL-005 | Discovery card | View col 1 | Discovery method, last seen, agent status | High |
| DTL-006 | Hardware cards | View col 2-3 | CPU, RAM, storage, monitors, peripherals | High |
| DTL-007 | Edit inline | Click pencil icon on field | Inline edit activates, save on blur/enter | High |
| DTL-008 | Lifecycle transition | Change lifecycle status dropdown | Status updated, history logged | High |
| DTL-009 | Asset tag edit | Click asset tag → Edit | InlineEditCell activates | Medium |
| DTL-010 | Location assign | Change location dropdown | Location updated | High |
| DTL-011 | Tag management | Add/remove tags on asset | Tags updated | Medium |
| DTL-012 | Delete asset | Click delete → Confirm | Asset soft-deleted, redirect to list | High |
| DTL-013 | Asset type varies | View Computer vs Printer vs Network | Layout adapts per asset type (5 types) | High |

### 4.2 Performance Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| DTL-020 | Tab renders | Click "Performance" tab | MetricCards with sparklines render | High |
| DTL-021 | CPU metrics | View CPU card | Current value, avg/peak, sparkline | High |
| DTL-022 | Memory metrics | View Memory card | Usage bar, available/total | High |
| DTL-023 | Disk metrics | View Disk card | Per-drive usage | High |
| DTL-024 | Process count | View Processes card | Current count with trend | Medium |
| DTL-025 | Period selector | Click "7D" period pill | Data refreshes for 7-day range | High |
| DTL-026 | Trend indicator | Compare current vs previous | Arrow up/down shown (null when negligible) | Medium |
| DTL-027 | No metrics | View asset with no agent checkin | Empty state: "No performance data" | Medium |

### 4.3 Connected Devices Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| DTL-030 | Tab renders | Click "Connected Devices" tab | Child asset list shown | High |
| DTL-031 | Add connected device | Click "Add" → Fill form → Save | Child asset created with parent_asset_id | High |
| DTL-032 | Link existing | Click "Link Existing" → Select asset | Asset linked as child | High |
| DTL-033 | Unlink device | Click unlink on child row | Child unlinked (parent_asset_id cleared) | High |
| DTL-034 | Edit manual child | Click pencil on manual child row | EditAssetSlideOver opens with grouped form | High |
| DTL-035 | Allowed child types | Add connected device | Only allowed_child_types shown in type dropdown | Medium |
| DTL-036 | Already linked excluded | Link existing → View list | Already-linked children not in list | Medium |

### 4.4 Compliance Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| DTL-040 | Tab renders | Click "Compliance" tab | Two-column layout: score gauge + categories | High |
| DTL-041 | Score gauge | View left sidebar | Compliance score ring with percentage | High |
| DTL-042 | Category breakdown | View right side | 2-column grid of 8 category cards | High |
| DTL-043 | Fix suggestions | View category checks | "Fix +N" shown for improvement guidance | Medium |
| DTL-044 | Alerts/violations | View left sidebar | Open alerts and violations listed | High |

### 4.5 History Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| DTL-050 | Tab renders | Click "History" tab | Field Changes section with table | High |
| DTL-051 | Change entries | View history | Source, actor, field, old/new values shown | High |
| DTL-052 | Source types | View different sources | user/agent/discovery/template/bulk/rule | Medium |
| DTL-053 | Search history | Search by field name | Filtered results | Medium |
| DTL-054 | Pagination | Navigate pages | Paginated correctly | Medium |

---

## 5. Asset History (Global)

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| AH-001 | Page loads | Navigate to /asset-history | Global searchable table renders | High |
| AH-002 | Search | Search hostname/tag/field/actor | Filtered results | High |
| AH-003 | Filter by source | Filter by "agent" | Only agent-sourced changes | Medium |
| AH-004 | Filter by field | Filter by specific field | Only that field's changes | Medium |
| AH-005 | Pagination | Navigate pages | Correct pagination | Medium |
| AH-006 | Stats endpoint | View stats section | Field change counts, top actors | Low |
| AH-007 | Click asset link | Click hostname in history | Navigate to asset detail | Medium |

---

## 6. Asset Import

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| IMP-001 | Page loads | Navigate to /import | Import page with upload area | High |
| IMP-002 | Upload CSV | Drag/click to upload CSV file | File accepted, column preview shown | Critical |
| IMP-003 | Column mapping | Map CSV columns to asset fields | Mapping interface works | Critical |
| IMP-004 | Validation | Click "Validate" | Validation results with warnings/errors | High |
| IMP-005 | Execute import | Click "Import" after validation | Assets created/updated, summary shown | Critical |
| IMP-006 | Invalid CSV | Upload non-CSV or malformed file | Error: "Invalid file format" | High |
| IMP-007 | Duplicate handling | Import with existing asset tags | Update existing or skip (configurable) | High |
| IMP-008 | Large file | Upload 10,000 row CSV | Handles gracefully with progress | Medium |

---

## 7. Discovery Center

### 7.1 Overview Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| DSC-001 | Tab loads | Navigate to /discovery → Overview | Job stats, running counts, summary | High |
| DSC-002 | Running job count | Active scan running | Count increments | High |

### 7.2 Subnets Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| DSC-010 | List subnets | View Subnets tab | Network ranges listed with status | Critical |
| DSC-011 | Add subnet | Click "Add" → Enter CIDR → Save | Subnet created | Critical |
| DSC-012 | Invalid CIDR | Enter "999.999.999.0/24" | Validation error | High |
| DSC-013 | Edit subnet | Click edit → Change label → Save | Subnet updated | High |
| DSC-014 | Delete subnet | Click delete → Confirm | Subnet removed | High |
| DSC-015 | Scan subnet | Click "Scan" on a subnet | Scan job triggered, redirect to scanner | Critical |
| DSC-016 | Assign location | Set location on subnet | Auto-assigns to discovered assets | High |
| DSC-017 | Toggle active | Enable/disable subnet | Active state toggles | Medium |
| DSC-018 | Import subnets | Upload CSV of subnets | Bulk created | Medium |

### 7.3 Active Directory Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| DSC-020 | List AD sources | View AD tab | AD sources listed | High |
| DSC-021 | Add AD source | Click "Add" → Enter LDAP config → Save | AD source created | High |
| DSC-022 | Sync AD | Click "Sync" on source | AD sync job triggered | High |
| DSC-023 | Edit AD source | Edit base DN, credentials | Source updated | High |
| DSC-024 | Delete AD source | Delete source | Source removed | High |

### 7.4 VMware Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| DSC-030 | List vCenter sources | View VMware tab | vCenter sources listed | High |
| DSC-031 | Add vCenter | Enter vCenter URL + credentials → Save | Source created | High |
| DSC-032 | Sync VMware | Click "Sync" | VMware discovery triggered | High |
| DSC-033 | Edit vCenter | Update credentials | Source updated | High |
| DSC-034 | Delete vCenter | Delete source | Source removed | High |

### 7.5 Cloud Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| DSC-040 | List cloud sources | View Cloud tab | AWS/Azure/GCP sources listed | High |
| DSC-041 | Add AWS source | Enter access key/secret → Save | AWS source created | High |
| DSC-042 | Add Azure source | Enter SP credentials → Save | Azure source created | High |
| DSC-043 | Add GCP source | Enter SA JSON → Save | GCP source created | High |
| DSC-044 | Scan cloud | Click "Scan" on cloud source | Cloud discovery triggered | High |
| DSC-045 | Delete cloud source | Delete source | Source removed | High |

### 7.6 Rules Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| DSC-050 | List rules | View Rules tab | Discovery rules listed with priority | High |
| DSC-051 | Add rule | Click "Add" → Define conditions + actions → Save | Rule created | High |
| DSC-052 | Edit rule | Modify conditions → Save | Rule updated | High |
| DSC-053 | Delete rule | Delete rule | Rule removed | High |
| DSC-054 | Toggle rule | Enable/disable rule | Active state toggles | Medium |
| DSC-055 | Test rule | Click "Test" with sample data | Dry-run results shown | Medium |
| DSC-056 | Run rule | Click "Run" → Confirm | Rule evaluated against all assets | High |
| DSC-057 | Rule priority | Reorder rules | Priority order affects evaluation | Medium |

---

## 8. Network Scanner

### 8.1 Active Scans Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SCN-001 | List active scans | View Active Scans tab | Running scans with progress | High |
| SCN-002 | Cancel scan | Click "Cancel" on running scan | Scan cancelled | High |
| SCN-003 | Live progress | Watch running scan | Progress updates via SSE/polling | High |
| SCN-004 | Discovered hosts | Expand scan details | Host list with probe results | High |

### 8.2 Scan History Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SCN-010 | List history | View Scan History tab | Past scans with results | High |
| SCN-011 | Scan details | Click scan row | Detailed results shown | High |
| SCN-012 | Delete history | Delete old scan record | Record removed | Medium |
| SCN-013 | Filter history | Filter by date/status/type | Filtered results | Medium |

---

## 9. Cloud Explorer

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| CLD-001 | Page loads | Navigate to /cloud-explorer | Cloud provider tabs shown | High |
| CLD-002 | AWS tab | Select AWS tab | AWS resources listed | High |
| CLD-003 | Azure tab | Select Azure tab | Azure resources listed | High |
| CLD-004 | GCP tab | Select GCP tab | GCP resources listed | High |
| CLD-005 | Plugin gate | No cloud module license | Page shows license required | High |

---

## 10. Credentials

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| CRD-001 | Page loads | Navigate to /credentials | Credential list shown | Critical |
| CRD-002 | Add SSH password | Click "Add" → Type=SSH Password → Fill → Save | Credential created (password encrypted) | Critical |
| CRD-003 | Add SSH key | Type=SSH Key → Upload key → Save | Key credential created | High |
| CRD-004 | Add WMI | Type=WMI Domain → Fill domain/user/pass → Save | WMI credential created | High |
| CRD-005 | Add SNMP v2c | Type=SNMP v2c → Community string → Save | SNMP credential created | High |
| CRD-006 | Add SNMP v3 | Type=SNMP v3 → Auth/priv settings → Save | SNMPv3 credential created | High |
| CRD-007 | Edit credential | Edit label → Save | Updated (password preserved if empty) | High |
| CRD-008 | Delete credential | Delete → Confirm | Credential removed | High |
| CRD-009 | Test connectivity | Click "Test" → Enter IP | Test result shown (success/fail) | High |
| CRD-010 | Secret masking | View credential list | Passwords/keys masked with *** | Critical |
| CRD-011 | Add WinRM | Type=WinRM → Fill → Save | WinRM credential created | Medium |
| CRD-012 | Add cloud creds | Type=AWS Key/Azure SP/GCP SA | Cloud credential created | High |
| CRD-013 | Vault encryption | Create credential → Check DB | Secret encrypted with AES-256-GCM | Critical |

---

## 11. Automation

### 11.1 Jobs Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| AUT-001 | List jobs | Navigate to /automation → Jobs | Scan jobs listed with status | Critical |
| AUT-002 | Create job | Click "Add" → Configure → Save | Job created | Critical |
| AUT-003 | Job types | Create network/cloud/ad_sync/vmware_sync job | Each type configurable | High |
| AUT-004 | Link schedule | Edit job → Link schedule → Save | Job linked to schedule(s) | High |
| AUT-005 | Unlink schedule | Remove schedule link → Save | Schedule unlinked | Medium |
| AUT-006 | Run job | Click "Run Now" | Job executes immediately | Critical |
| AUT-007 | Edit job | Change ranges/credentials → Save | Job updated | High |
| AUT-008 | Delete job | Delete → Confirm | Job removed | High |
| AUT-009 | Job status | View running job | Status: pending → running → completed/failed | High |
| AUT-010 | Deep link | Click schedule badge in Discovery | Navigate to /automation?tab=jobs&highlight={id} | Medium |

### 11.2 Schedules Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| AUT-020 | List schedules | View Schedules tab | Reusable schedules listed | High |
| AUT-021 | Create schedule | Click "Add" → Configure frequency → Save | Schedule created | High |
| AUT-022 | Interval schedule | Set "Every 4 hours" | Correct cron generated | High |
| AUT-023 | Daily schedule | Set "Daily at 2:00 AM" | Cron: 0 2 * * * (UTC) | High |
| AUT-024 | Weekly schedule | Set "Mon, Wed, Fri at 9 AM" | Correct cron with day list | High |
| AUT-025 | Monthly schedule | Set "1st and 15th at midnight" | Days-of-month schedule | High |
| AUT-026 | Edit schedule | Change time → Save | Schedule updated, linked jobs affected | High |
| AUT-027 | Delete schedule | Delete → Confirm | Schedule removed, jobs unlinked | High |
| AUT-028 | ScheduleBuilder | View schedule builder | Comprehensive UI with time/day pickers | High |

---

## 12. Agents

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| AGT-001 | List agents | Navigate to /agents | Agent list with status, version, last checkin | Critical |
| AGT-002 | Agent online status | Agent checked in recently | Green status indicator | High |
| AGT-003 | Agent offline | Agent hasn't checked in (stale) | Red/orange status indicator | High |
| AGT-004 | Push install | Click "Deploy" → Select target → Install | Agent installed on target | Critical |
| AGT-005 | Push update | Click "Update" on outdated agent | Binary updated, service restarted | High |
| AGT-006 | Remote uninstall | Click "Uninstall" → Confirm | Agent uninstalled from target | High |
| AGT-007 | Bulk settings | Select agents → Change interval | All selected agents updated | Medium |
| AGT-008 | Quick install script | Generate install script | Script shown for copy/paste | High |
| AGT-009 | Registration token | Create new token | Token generated for manual install | High |
| AGT-010 | Agent version | View version column | Current version shown (compare to latest) | High |
| AGT-011 | Self-update | Agent checks in with old version | Update URL in response triggers auto-update | High |
| AGT-012 | Delete agent | Delete agent record | Agent deactivated | High |
| AGT-013 | Dedup guard | Same asset re-registers | Old registration deactivated, new one active | High |

---

## 13. Agent Auto-Deploy Queue

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| ADQ-001 | Page loads | Navigate to /agents/auto-deploy | Queue page with attempts list | High |
| ADQ-002 | Attempt list | View attempts | Asset, status, credential, error, timestamp shown | High |
| ADQ-003 | Heal Offline | Click "Heal Offline" button | Offline agents queued for repair | High |
| ADQ-004 | Update Outdated | Click "Update Outdated" button | Version-drifted agents queued for update | High |
| ADQ-005 | Manual trigger | Click trigger for specific asset | Deploy queued for that asset | High |
| ADQ-006 | Status filters | Filter by success/failed/skipped | Filtered attempts | Medium |
| ADQ-007 | Retry cooldown | View recently failed attempt | Cooldown indicator shown | Medium |
| ADQ-008 | Max attempts | Asset exceeded max_attempts | Marked as "max attempts reached" | Medium |

---

## 14. Software & Licenses

### 14.1 Inventory Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SW-001 | Tab loads | Navigate to /software → Inventory | Software inventory list | Critical |
| SW-002 | Classification badges | View classification filter buttons | Free/Commercial/Open Source buttons | High |
| SW-003 | Filter by class | Click "Commercial" | Only commercial software shown | High |
| SW-004 | Install count click | Click install count number | SlideOver opens with asset list | High |
| SW-005 | Search software | Search by name/publisher | Filtered results | High |
| SW-006 | Auto-classification | View unclassified software | Heuristic classifier runs on 100+ patterns | Medium |

### 14.2 Licenses Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SW-010 | List licenses | View Licenses tab | License records listed | High |
| SW-011 | Create license | Click "Add" → Fill form → Save | License created (perpetual/subscription/trial) | High |
| SW-012 | Edit license | Edit license details → Save | License updated | High |
| SW-013 | Delete license | Delete → Confirm | License removed | High |
| SW-014 | Perpetual type | Create perpetual license | No expiry date required | Medium |

### 14.3 Entitlements Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SW-020 | List entitlements | View Entitlements tab | Entitlement records | High |
| SW-021 | Create entitlement | Searchable Combobox for software → Save | Entitlement created | High |
| SW-022 | Edit entitlement | Update quantities → Save | Entitlement updated | High |
| SW-023 | Delete entitlement | Delete entitlement | Removed | High |

### 14.4 Other Tabs

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SW-030 | OS Compliance | View OS Compliance tab | Per-device OS license status | High |
| SW-031 | Software Compliance | View Software Compliance tab | Under/over-licensed status | High |
| SW-032 | SaaS tab | View SaaS tab | SaaS subscription tracking | High |
| SW-033 | Create SaaS | Add SaaS subscription → Save | Subscription created | High |
| SW-034 | Catalog tab | View Catalog tab | Software catalog entries | Medium |
| SW-035 | Recommendations | View Recommendations tab | Optimization suggestions | Medium |
| SW-036 | /sam redirect | Navigate to /sam | Redirects to /software?tab=entitlements | Medium |

---

## 15. Compliance

### 15.1 Dashboard Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| CMP-001 | Tab loads | Navigate to /compliance → Dashboard | Score gauge, stat cards, schedule | Critical |
| CMP-002 | Compliance score | View score gauge | Overall compliance percentage | High |
| CMP-003 | Stat cards | View 5 stat cards | Compliant, At Risk, Open Alerts, Violations, Frameworks | High |
| CMP-004 | Run All Detections | Click "Run All Detections" | SW eval + non-domain + scores recalculated | Critical |
| CMP-005 | Detection schedule | Open schedule SlideOver → Configure | Schedule interval/hour saved | High |
| CMP-006 | Unprotected count | View KPI | Count of endpoints without AV | High |

### 15.2 Software Policies Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| CMP-010 | List policies | View Software tab | Allow/deny policies listed | High |
| CMP-011 | Create deny policy | Click "Add" → Deny list → Save | Deny policy created | High |
| CMP-012 | Create allow policy | Click "Add" → Allow list → Save | Allow policy created | High |
| CMP-013 | Edit policy | Edit policy rules → Save | Policy updated | High |
| CMP-014 | Delete policy | Delete policy | Policy removed | High |
| CMP-015 | Evaluate policies | Click "Evaluate" | Violations generated for matches | High |
| CMP-016 | Seed defaults | Click "Seed Defaults" | Built-in policies created | Medium |

### 15.3 Security Alerts Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| CMP-020 | List alerts | View Alerts tab | Open security alerts listed | High |
| CMP-021 | Acknowledge alert | Click "Acknowledge" on alert | Alert status → acknowledged | High |
| CMP-022 | Resolve alert | Click "Resolve" on alert | Alert status → resolved, score recalculated | High |
| CMP-023 | Alert exception | Create exception with duration | Alert suppressed for duration | High |
| CMP-024 | Non-domain detection | Windows assets without domain | Non-domain alerts auto-generated | High |
| CMP-025 | Auto-resolve | Asset joins domain | Non-domain alert auto-resolved | High |
| CMP-026 | Bulk update | Select alerts → Bulk acknowledge | All selected updated, scores recalculated | Medium |

### 15.4 Frameworks Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| CMP-030 | List frameworks | View Frameworks tab | 6 frameworks listed (HIPAA, SOC2, PCI, ISO, NIST, CIS) | High |
| CMP-031 | Framework score | View ScoreRing per framework | Per-framework compliance score | High |
| CMP-032 | Expand controls | Click framework → Expand | Per-control pass/partial/fail/unmapped | High |
| CMP-033 | Control detail | Click a control | Detail view with linked policies | Medium |
| CMP-034 | Policy mapping | View policy → control mappings | POLICY_CONTROL_MAP visualized | Medium |

---

## 16. Tasks

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| TSK-001 | List tasks | Navigate to /tasks | Task list with status, assignee, due date | Critical |
| TSK-002 | Create task | Click "Add" → Fill form → Save | Task created | Critical |
| TSK-003 | Create from template | Click "From Template" → Select → Save | Task created with template fields | High |
| TSK-004 | Edit task | Edit title/description → Save | Task updated | High |
| TSK-005 | Delete task | Delete → Confirm | Task removed | High |
| TSK-006 | Assign engineer | Assign user to task | Assignee updated | High |
| TSK-007 | Add checklist | Add checklist items to task | Checklist created | High |
| TSK-008 | Complete checklist | Check off checklist items | Progress updates (e.g., 3/5) | High |
| TSK-009 | Complete task | Mark task complete | Status → completed | Critical |
| TSK-010 | Reopen task | Reopen completed task | Status → open again | High |
| TSK-011 | Link assets | Link assets to task | Asset list shown on task | High |
| TSK-012 | Task detail | Click task row | Navigate to /tasks/{id} detail | High |
| TSK-013 | Task reports | View task reports section | Completion stats, overdue tasks | Medium |
| TSK-014 | Task templates | View templates tab | Template list with CRUD | High |
| TSK-015 | Filter tasks | Filter by status/assignee/priority | Filtered results | High |

---

## 17. Change Management

### 17.1 Basic Change Management

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| CM-001 | List changes | Navigate to /changes | Change request list | Critical |
| CM-002 | Create change | Click "Add" → Fill form → Save | Change request created | Critical |
| CM-003 | Edit change | Edit details → Save | Change updated | High |
| CM-004 | Submit for approval | Submit change request | Status → submitted, approvers notified | High |
| CM-005 | Approve change | Approver clicks "Approve" | Status → approved | Critical |
| CM-006 | Reject change | Approver clicks "Reject" | Status → rejected | High |
| CM-007 | Start change | Click "Start" on approved change | Status → in_progress | High |
| CM-008 | Complete change | Click "Complete" | Status → completed | High |
| CM-009 | Cancel change | Click "Cancel" | Status → cancelled | High |
| CM-010 | Link assets | Link affected assets to change | Assets shown in change detail | High |

### 17.2 Enterprise Change Management

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| CM-020 | CAB review | Submit high-risk change | Routed to Change Advisory Board | High |
| CM-021 | Multi-stage approval | Create change with multi-stage flow | Each stage requires approval | High |
| CM-022 | Change templates | Create from template | Pre-filled form from template | High |
| CM-023 | Change policies | Define change policy | Policy evaluates incoming changes | High |
| CM-024 | Calendar view | View change calendar | Changes displayed on calendar | High |
| CM-025 | Blackout window | Define blackout period | Changes blocked during blackout | High |
| CM-026 | PIR | Post-Implementation Review | PIR form available after completion | High |
| CM-027 | Conflict detection | Overlapping changes | Conflict warning shown | Medium |

---

## 18. Reports

### 18.1 Reports Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| RPT-001 | List reports | Navigate to /reports → Reports | Report definitions listed | High |
| RPT-002 | Create report | Define columns, conditions, sort → Save | Report created | High |
| RPT-003 | Preview report | Click "Preview" | Report data rendered | High |
| RPT-004 | Edit report | Modify conditions → Save | Report updated | High |
| RPT-005 | Delete report | Delete report | Report removed | High |
| RPT-006 | Schedule report | Add cron schedule to report | Auto-generates on schedule | High |
| RPT-007 | Export CSV | Export report as CSV | CSV file downloaded | High |
| RPT-008 | Export PDF | Export report as PDF | PDF file downloaded | Medium |

### 18.2 History Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| RPT-010 | List generated | View History tab | Generated reports listed | High |
| RPT-011 | Download report | Click download on generated report | File downloads | High |
| RPT-012 | Export history | Export generated report | Downloaded in selected format | Medium |

---

## 19. Locations

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| LOC-001 | List locations | Navigate to /locations | Location hierarchy tree/list | Critical |
| LOC-002 | Tree view | Toggle to tree view | Expandable hierarchy | High |
| LOC-003 | List view | Toggle to list view | Flat table with type/parent | High |
| LOC-004 | Add location | Click "Add" → Fill form → Save | Location created | Critical |
| LOC-005 | Location types | Select type (country/state/.../rack) | Type assigned | High |
| LOC-006 | Set parent | Select parent location | Hierarchy maintained | High |
| LOC-007 | Edit location | Edit name/type/code → Save | Location updated | High |
| LOC-008 | Delete location | Delete → Confirm | Location removed (children orphaned?) | High |
| LOC-009 | Tag prefix | Set tag_prefix on location | Used in asset tag generation | High |
| LOC-010 | Import CSV | Upload location CSV | Bulk locations created | Medium |
| LOC-011 | Export CSV | Export locations | CSV downloaded | Medium |
| LOC-012 | Address field | Set address on location | Address stored | Medium |

---

## 20. Procurement & Contracts

### 20.1 Procurement

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| PRC-001 | List POs | Navigate to procurement section | Purchase orders listed | High |
| PRC-002 | Create PO | Click "Add" → Fill PO form → Save | PO created | High |
| PRC-003 | Add line items | Add items to PO | Line items added with costs | High |
| PRC-004 | Edit PO | Edit PO details → Save | PO updated | High |
| PRC-005 | Delete PO | Delete PO | PO removed | High |
| PRC-006 | Vendor management | Create/edit vendor | Vendor record managed | High |

### 20.2 Contracts

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| CTR-001 | List contracts | View contracts section | Contracts listed | High |
| CTR-002 | Create contract | Click "Add" → Fill → Save | Contract created | High |
| CTR-003 | Link assets | Attach assets to contract | Assets linked | High |
| CTR-004 | Edit contract | Edit details → Save | Contract updated | High |
| CTR-005 | Delete contract | Delete → Confirm | Contract removed | High |
| CTR-006 | Expiry alerts | Contract nearing expiry | Warning indicator shown | Medium |

---

## 21. Budgets & Cost Center

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| BDG-001 | List budgets | Navigate to /budgets | Budget list | High |
| BDG-002 | Create budget | Click "Add" → Fill → Save | Budget created | High |
| BDG-003 | Add cost entry | Add expense to budget | Entry added with amount | High |
| BDG-004 | Edit budget | Edit allocation → Save | Budget updated | High |
| BDG-005 | Delete budget | Delete → Confirm | Budget removed | High |
| BDG-006 | Cost Center page | Navigate to /cost-center | Multi-tab procurement/contract/approval overview | High |
| BDG-007 | Chargeback report | View chargeback section | Cost allocation by department/location | Medium |

---

## 22. SNMP & OUI

### 22.1 Vendor OIDs

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SNM-001 | List OIDs | Navigate to /snmp → Vendor OIDs | OID entries listed | High |
| SNM-002 | Add OID | Click "Add" → Enter OID + vendor → Save | OID entry created | High |
| SNM-003 | Edit OID | Edit entry → Save | OID updated | High |
| SNM-004 | Delete OID | Delete entry | OID removed | High |
| SNM-005 | Extended OIDs | View extended OIDs section | Extended OID entries listed | Medium |

### 22.2 MIB Files

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SNM-010 | List MIBs | View MIB Files tab | Uploaded MIB files listed | High |
| SNM-011 | Upload MIB | Click "Upload" → Select file | MIB parsed and stored | High |
| SNM-012 | View parsed | Click MIB → View objects | Parsed OID objects shown | Medium |
| SNM-013 | Delete MIB | Delete MIB file | MIB removed | Medium |

### 22.3 MAC OUI

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SNM-020 | List OUI | View MAC OUI tab | OUI table with manufacturers | High |
| SNM-021 | Search OUI | Search by MAC/vendor | Filtered results | High |
| SNM-022 | Add manual | Add manual OUI entry | Entry created | Medium |
| SNM-023 | Sync IEEE | Click "Sync IEEE Database" | OUI table updated from IEEE | Medium |
| SNM-024 | Import CSV | Import OUI CSV | Bulk entries created | Low |

---

## 23. Audit Logs

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| LOG-001 | List logs | Navigate to /logs | Audit log entries listed | Critical |
| LOG-002 | Filter severity | Filter by info/warning/error | Filtered entries | High |
| LOG-003 | Filter entity | Filter by entity type (asset/user/etc.) | Filtered entries | High |
| LOG-004 | Filter user | Filter by actor/user | Filtered entries | High |
| LOG-005 | Filter action | Filter by action (create/update/delete) | Filtered entries | High |
| LOG-006 | Date range | Set date from/to | Filtered by date range | High |
| LOG-007 | Search | Search log text | Full-text search results | High |
| LOG-008 | Export | Export audit logs | CSV/PDF downloaded | Medium |
| LOG-009 | Log detail | Click log entry | Expanded detail view | Medium |
| LOG-010 | Pagination | Navigate pages | Paginated correctly | Medium |

---

## 24. Settings

### 24.1 General Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SET-001 | View general | Navigate to Settings → General | Agent defaults, retention, warranty config | High |
| SET-002 | Edit agent defaults | Change checkin interval → Save | Settings saved | High |
| SET-003 | History retention | Set retention days → Save | Retention period updated | High |
| SET-004 | Warranty API | Configure warranty API provider → Save | Provider settings saved | Medium |
| SET-005 | Agent auto-deploy | Toggle master auto-deploy switch | Setting toggled | High |
| SET-006 | Per-OS toggles | Enable/disable Windows/Linux/macOS deploy | Per-OS settings saved | High |
| SET-007 | Self-healing | Toggle auto-heal + auto-update | Healing settings saved | Medium |
| SET-008 | Organization info | Edit org name/currency → Save | Organization updated | Medium |
| SET-009 | Tab ordering | Reorder asset detail tabs | Tab order persisted | Low |

### 24.2 License Portal Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SET-010 | View portal | View License Portal tab | Portal URL, customer ID, status | High |
| SET-011 | Edit portal URL | Change portal URL → Save | URL updated | High |
| SET-012 | Pull license | Click "Sync" | License pulled from portal | High |

### 24.3 Tag Numbering Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SET-020 | View tag settings | View Tag Numbering tab | Prefix, separator, counter config | High |
| SET-021 | Edit prefix | Change prefix → Save | Prefix updated | High |
| SET-022 | Edit separator | Change separator (-, /) → Save | Separator updated | Medium |
| SET-023 | Global scope | Set scope=global | Single counter for all assets | High |
| SET-024 | Per-location scope | Set scope=per_location | Counter per location hierarchy | High |
| SET-025 | Counter level | Set counter_level → Save | Counter key determined by level | Medium |
| SET-026 | Number padding | Set 5-digit padding → Save | Tags zero-padded to 5 digits | Medium |

### 24.4 Asset Configuration Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SET-030 | View templates | View Asset Configuration tab | Type Forms + Templates + Dropdown Lists sub-tabs | High |
| SET-031 | Create type form | Add form for "Storage" type → Save | Type form created | High |
| SET-032 | Edit type form | Add/remove fields → Save | Form updated | High |
| SET-033 | Seed defaults | Click "Seed Defaults" | 14 built-in type forms created | High |
| SET-034 | Create dropdown | Add dropdown list → Add options → Save | Dropdown list created | High |
| SET-035 | Create template | Add field template → Save | Template created | High |
| SET-036 | Template blocks | Drag template block in type form | Position reordered | Medium |
| SET-037 | Field types | Test all 10 field types | Each type renders correctly | High |
| SET-038 | Select source | Link field to master dropdown list | Options from dropdown shown | Medium |
| SET-039 | Repeatable fields | Add repeatable field → Save | Field supports multiple entries | Medium |
| SET-040 | Delete dropdown | Delete dropdown list | List removed | Medium |
| SET-041 | Custom asset types | Create new custom type | Type available in asset creation | High |

### 24.5 Tags Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SET-050 | List tags | View Tags tab | Tag categories listed | High |
| SET-051 | Create category | Add tag category → Save | Category created | High |
| SET-052 | Add tag values | Add values to category | Values available for tagging | High |
| SET-053 | Edit category | Edit name/color → Save | Category updated | Medium |
| SET-054 | Delete category | Delete category | Category removed | Medium |

### 24.6 Webhooks Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SET-060 | List webhooks | View Webhooks tab | Webhook endpoints listed | High |
| SET-061 | Create webhook | Add URL + events → Save | Webhook created | High |
| SET-062 | Edit webhook | Edit URL/events → Save | Webhook updated | High |
| SET-063 | Toggle webhook | Enable/disable webhook | State toggled | High |
| SET-064 | Delete webhook | Delete webhook | Webhook removed | High |
| SET-065 | Test webhook | Click "Test" | Test payload sent, result shown | High |
| SET-066 | Delivery history | View delivery history | Past deliveries with status | Medium |

### 24.7 Security Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SET-070 | View security | View Security tab | API keys, sessions, password policy, IP list, SSO | High |
| SET-071 | Create API key | Click "Create" → Name → Generate | API key created, shown once | High |
| SET-072 | Revoke API key | Click "Revoke" on key | Key revoked | High |
| SET-073 | Rotate API key | Click "Rotate" | New key issued, old invalidated | High |
| SET-074 | Session list | View active sessions | User sessions listed | High |
| SET-075 | Revoke session | Click "Revoke" on session | Session invalidated | High |
| SET-076 | Password policy | Set min length/complexity → Save | Policy saved | High |
| SET-077 | IP allowlist | Add IP/CIDR → Save | Allowlist updated | High |
| SET-078 | SSO config | Configure SSO provider → Save | SSO provider saved | High |

### 24.8 Lifecycle Stages Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SET-080 | View stages | View Lifecycle Stages tab | Stages listed with flow diagram | High |
| SET-081 | Create stage | Add custom stage → Save | Stage created | High |
| SET-082 | Edit stage | Edit name/color/transitions → Save | Stage updated | High |
| SET-083 | Delete custom stage | Delete non-built-in stage | Stage removed | High |
| SET-084 | Cannot delete built-in | Try to delete built-in stage | Delete button disabled/hidden | High |
| SET-085 | Transition rules | Define allowed transitions | Only allowed transitions available on assets | High |
| SET-086 | Required fields | Set required fields for stage | Fields enforced on transition | Medium |
| SET-087 | Flow diagram | View flow diagram | Visual transition flow rendered | Medium |

### 24.9 Backup & Restore Tab

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SET-090 | View backups | View Backup & Restore tab | Backup list with dates/sizes | High |
| SET-091 | Create backup | Click "Create Backup" | pg_dump runs, backup file created | Critical |
| SET-092 | Download backup | Click "Download" on backup | File downloads | High |
| SET-093 | Verify backup | Click "Verify" on backup | Manifest details shown (PG version, tables, row counts) | High |
| SET-094 | Restore backup | Click "Restore" → Confirm | Database restored from backup | Critical |
| SET-095 | Vault key mismatch | Restore from different server | Warning: credentials won't decrypt | High |
| SET-096 | Delete backup | Delete backup file | File removed | Medium |
| SET-097 | Scheduled backup | Configure backup schedule → Save | Backups run on schedule | Medium |
| SET-098 | Restore existing | Restore from on-disk backup | No re-upload needed | Medium |

---

## 25. Users & RBAC

### 25.1 User Management

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| USR-001 | List users | Settings → Users tab | User list with username, email, role, status | Critical |
| USR-002 | Create admin | Click "Add" → Role=Admin → Fill → Save | Admin user created | Critical |
| USR-003 | Create operator | Click "Add" → Role=Operator → Fill → Save | Operator created with defaults | Critical |
| USR-004 | Create viewer | Click "Add" → Role=Viewer → Fill → Save | Viewer created with defaults | Critical |
| USR-005 | Duplicate username | Create user with existing username | Error: "Username already exists" | High |
| USR-006 | Duplicate email | Create user with existing email | Error: "Email already exists" | High |
| USR-007 | Edit user | Edit email/role → Save | User updated | High |
| USR-008 | Change password | Edit user → Change password → Save | Password updated (bcrypt hashed) | High |
| USR-009 | Delete user | Delete → Confirm | User soft-deleted | High |
| USR-010 | Toggle status | Click enable/disable toggle | User activated/deactivated | High |
| USR-011 | Cannot delete self | Try to delete own account | Action blocked | High |
| USR-012 | Cannot demote last admin | Change only admin to operator | Error: "Cannot demote last admin" | Critical |

### 25.2 RBAC Permissions

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| USR-020 | Admin full access | Login as admin → Visit all pages | All pages/operations accessible | Critical |
| USR-021 | Operator defaults | Login as operator → Check access | Can view + create/edit assets, discovery, tasks; limited settings | Critical |
| USR-022 | Viewer defaults | Login as viewer → Check access | Read-only on permitted pages | Critical |
| USR-023 | Custom permissions | Edit operator → Enable custom → Modify | Custom overrides role defaults | Critical |
| USR-024 | Select All | Click "Select All" on custom perms | All operations checked | Medium |
| USR-025 | Clear All | Click "Clear All" | All operations unchecked | Medium |
| USR-026 | Toggle page group | Toggle entire "Discovery" group | All discovery tab permissions toggled | Medium |
| USR-027 | Empty custom perms | Set custom perms to empty object | User has NO permissions | High |
| USR-028 | Permission format | Check both dict and array format | Both formats handled by has_permission() | High |

### 25.3 RBAC Enforcement (Per-Page)

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| USR-030 | Dashboard view | Viewer → /dashboard | Dashboard renders (view-only) | High |
| USR-031 | Assets view only | Viewer → /assets | List visible, no Add/Edit/Delete buttons | High |
| USR-032 | Assets create | Operator → /assets → Add | "Add Asset" button visible & functional | High |
| USR-033 | Assets no delete | Operator → /assets | No delete button (operator default has no delete) | High |
| USR-034 | Discovery scan | Operator → Discovery → Scan | Scan button enabled | High |
| USR-035 | Discovery no delete | Operator → Discovery → Subnets | No delete button | High |
| USR-036 | Credentials view-only | Operator → Credentials | View + Test only, no create/edit/delete | High |
| USR-037 | Settings limited | Operator → Settings | Only General/Tag/Templates/Webhooks visible (view-only) | High |
| USR-038 | Settings users | Viewer → Settings | Users tab not visible | High |
| USR-039 | API permission check | Operator → POST /credentials | 403 Forbidden | Critical |
| USR-040 | API admin bypass | Admin → Any endpoint | Always permitted | Critical |
| USR-041 | Task complete | Operator → Complete task | Allowed (operator has "complete") | High |
| USR-042 | Task no delete | Operator → Delete task | Not allowed (operator default has no delete for tasks) | High |
| USR-043 | Compliance edit | Operator → Run detections | Allowed (operator has "edit" on compliance) | High |
| USR-044 | Compliance no delete | Operator → Delete policy | Not allowed | High |
| USR-045 | Agents install | Operator → Deploy agent | Allowed | High |
| USR-046 | Agents no uninstall | Operator → Uninstall agent | Not allowed (operator default has no uninstall) | High |
| USR-047 | Modules view-only | Operator → Modules | View only, no toggle | High |
| USR-048 | Budgets no delete | Operator → Delete budget | Not allowed | High |
| USR-049 | Sidebar visibility | Viewer login | Only permitted sections visible in sidebar | Critical |
| USR-050 | Change approve | User without "approve" → Approve change | Button hidden / API returns 403 | High |

### 25.4 Permission Registry (60+ Pages)

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| USR-060 | Registry completeness | GET /users/permissions/registry | All 60+ pages returned with operations | High |
| USR-061 | Frontend sync | Login → Check localStorage perms | Permissions match /users/me response | High |
| USR-062 | Real-time sync | Admin changes user perms → User refreshes | Permissions updated immediately | High |
| USR-063 | Page groups | Check PAGE_GROUPS mapping | All tabs mapped to correct groups | Medium |
| USR-064 | has_any_tab_permission | User with one discovery tab perm | Discovery page shows in sidebar | High |
| USR-065 | No tab permission | User with zero discovery perms | Discovery hidden from sidebar | High |

---

## 26. Notifications

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| NTF-001 | List channels | Settings → Notifications tab → Channels | Channels listed (email, Slack, webhook) | High |
| NTF-002 | Create email channel | Add email channel → Configure → Save | Channel created | High |
| NTF-003 | Create Slack channel | Add Slack webhook URL → Save | Channel created | High |
| NTF-004 | Edit channel | Edit channel config → Save | Channel updated | High |
| NTF-005 | Delete channel | Delete channel | Channel removed | High |
| NTF-006 | Create alert rule | Add alert rule → Configure triggers → Save | Rule created | High |
| NTF-007 | Toggle rule | Enable/disable alert rule | Rule toggled | Medium |
| NTF-008 | Edit rule | Edit rule conditions → Save | Rule updated | Medium |
| NTF-009 | Delete rule | Delete rule | Rule removed | Medium |
| NTF-010 | Notification history | View history sub-tab | Past notifications listed | Medium |
| NTF-011 | Pill sub-tabs | Navigate Channels → Rules → History | Pill-style sub-tabs work correctly | Medium |

---

## 27. Webhooks

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| WHK-001 | List webhooks | Navigate to /webhooks or Settings → Webhooks | Webhook list | High |
| WHK-002 | Create webhook | Add URL + select events → Save | Webhook created | High |
| WHK-003 | Event selection | Choose asset.created, asset.updated events | Events stored | High |
| WHK-004 | Toggle webhook | Enable/disable | State toggled | High |
| WHK-005 | Test webhook | Click "Test" | Test payload sent to URL | High |
| WHK-006 | Delete webhook | Delete → Confirm | Webhook removed | High |
| WHK-007 | Delivery log | View delivery history | Past deliveries with status codes | Medium |
| WHK-008 | Retry failed | Failed delivery → Auto-retry | Retried (configurable) | Medium |

---

## 28. Backup & Restore

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| BKP-001 | Create backup | POST /backups | pg_dump creates .sql.gz file | Critical |
| BKP-002 | List backups | GET /backups | All backup files listed with size/date | High |
| BKP-003 | Verify backup | POST /backups/{file}/verify | Manifest: PG version, source, tables, row counts | High |
| BKP-004 | Download backup | GET /backups/{file}/download | File downloads | High |
| BKP-005 | Restore backup | POST /backups/{file}/restore | Database restored, vault_key_match reported | Critical |
| BKP-006 | Cross-server restore | Restore with different VAULT_KEY | Warning: vault_key_match=false | High |
| BKP-007 | Delete backup | DELETE /backups/{file} | File deleted from disk | High |
| BKP-008 | Path traversal | Attempt path traversal in filename | 400: Invalid filename | Critical |
| BKP-009 | Scheduled backup | Configure schedule → Wait | Backup auto-created on schedule | High |
| BKP-010 | Restore from existing | Restore on-disk backup | Works without re-upload | Medium |
| BKP-011 | Backup settings | Update backup path/retention | Settings saved | Medium |

---

## 29. Plugins & Licensing

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| PLG-001 | List modules | Navigate to /modules | Modules listed with license status | High |
| PLG-002 | Toggle module | Enable/disable module | Module state toggled | High |
| PLG-003 | Unlicensed module | Enable module without license | Warning/error shown | High |
| PLG-004 | Built-in free | View Linux/Windows/macOS/Network | Always enabled, no license required | High |
| PLG-005 | Upload license | Upload .lic file | Ed25519 signature verified, modules unlocked | Critical |
| PLG-006 | Invalid license | Upload tampered .lic file | Verification fails, rejected | Critical |
| PLG-007 | Portal sync | Sync license from portal | License pulled and applied | High |
| PLG-008 | Expired license | Module license expired | Module disabled, warning shown | High |
| PLG-009 | Plugin gating | Cloud module disabled → Cloud Explorer | Page shows "License required" | High |
| PLG-010 | Sidebar gating | Compliance module disabled | Compliance link hidden from sidebar | High |

---

## 30. Sidebar & Navigation

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| NAV-001 | Sidebar renders | Login → View sidebar | 6 sections with all nav items | Critical |
| NAV-002 | Section collapse | Click section header | Section expands/collapses | High |
| NAV-003 | Icon-only mode | Click collapse toggle | Sidebar shrinks to icons | High |
| NAV-004 | Mobile hamburger | View on < 768px | Hamburger menu shown | High |
| NAV-005 | Auto-collapse | View on < 1024px | Sidebar auto-collapses | Medium |
| NAV-006 | Active link | Navigate to /assets | Assets link highlighted | High |
| NAV-007 | Badge counts | View badges | Total assets, recent jobs, tasks, changes, alerts | High |
| NAV-008 | Badge refresh | Wait 30 seconds | Badge counts refresh | Medium |
| NAV-009 | Alert badge | Open compliance alerts exist | Red pulsing badge on Compliance | High |
| NAV-010 | Permission gating | Viewer login | Hidden sections/links for unpermitted pages | Critical |
| NAV-011 | Plugin gating | Cloud module disabled | Cloud Explorer hidden | High |
| NAV-012 | User dropdown | Click avatar | Profile, theme toggle, sign out shown | High |
| NAV-013 | Theme toggle | Toggle dark/light mode | Theme switches | Medium |
| NAV-014 | Organization name | Set org name in settings | Shown in sidebar logo area | Low |
| NAV-015 | 404 page | Navigate to /nonexistent | NotFound page with warning + "Go to Dashboard" | High |
| NAV-016 | Deep link tabs | Navigate to /automation?tab=jobs | Jobs tab auto-selected | High |
| NAV-017 | Highlight param | Navigate to ?highlight={id} | Row auto-scrolled and highlighted | Medium |

---

## 31. Cross-Cutting Concerns

### 31.1 UI Components

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| UI-001 | Button sizes | Verify all buttons use size="sm" | Slim, compact buttons everywhere | High |
| UI-002 | SlideOver forms | All create/edit forms | Use SlideOver (w-[520px]), not Modal | High |
| UI-003 | MiniToggle | All boolean toggles | Use MiniToggle, not checkboxes | High |
| UI-004 | EmptyState | Page with no data | EmptyState component with CTA action button | High |
| UI-005 | Toast notifications | Success/error operations | Toast appears with appropriate variant | High |
| UI-006 | DataTable | All list views | TanStack Table with sort/pagination/filter | High |
| UI-007 | Tab hierarchy | Main tabs + sub-tabs | Main=underline, sub=pill style | High |
| UI-008 | Icons | All icons from lucide-react | No other icon libraries | Medium |
| UI-009 | cn() utility | Conditional classes | clsx + tailwind-merge works correctly | Medium |
| UI-010 | Responsive layout | View on mobile/tablet/desktop | Proper responsive breakpoints | High |

### 31.2 API Security

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| SEC-001 | JWT required | Call API without token | 401 Unauthorized | Critical |
| SEC-002 | Invalid JWT | Call API with expired/tampered token | 401 Unauthorized | Critical |
| SEC-003 | SQL injection | Send `'; DROP TABLE assets;--` in search | Input sanitized, no injection | Critical |
| SEC-004 | XSS prevention | Store `<script>alert(1)</script>` in field | HTML escaped on output | Critical |
| SEC-005 | RBAC enforcement | Non-admin calls admin-only endpoint | 403 Forbidden | Critical |
| SEC-006 | Path traversal | `../../etc/passwd` in file parameters | 400: Invalid path | Critical |
| SEC-007 | Rate limiting | Rapid login attempts | Rate limited or locked | High |
| SEC-008 | Credential encryption | Check credential in DB | Encrypted with AES-256-GCM | Critical |
| SEC-009 | Audit logging | Any CRUD operation | Audit log entry created | High |
| SEC-010 | CORS | Cross-origin request in production | Blocked (DEBUG=false) | High |

### 31.3 Performance

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| PERF-001 | Asset list 10k | Load /assets with 10,000 assets | Renders under 3 seconds | High |
| PERF-002 | Dashboard load | Load Dashboard with full data | Renders under 2 seconds | High |
| PERF-003 | Lazy loading | Navigate to Settings tabs | Components load on tab switch | Medium |
| PERF-004 | Query caching | Navigate away and back | TanStack Query cache used | Medium |
| PERF-005 | Large CSV import | Import 5,000 row CSV | Completes within reasonable time | Medium |

### 31.4 Install Wizard

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| INS-001 | Fresh install | Boot with no admin | /install wizard shown | Critical |
| INS-002 | Step 1 Welcome | View welcome screen | Next button works | High |
| INS-003 | Step 2 Env Check | View environment check | Env vars validated (warnings, not fatal) | High |
| INS-004 | Step 3 Database | Check DB connectivity | Connection tested | High |
| INS-005 | Step 4 Admin | Create admin user | Admin account created | Critical |
| INS-006 | Step 5 Organization | Set org name | Org saved | High |
| INS-007 | Step 6 Complete | Finish wizard | Redirect to Dashboard | High |
| INS-008 | Wizard gate | Installed → Visit /install | Redirect away (already installed) | High |

---

## 32. Market Leader Comparison & Gap Analysis

### 32.1 Feature Comparison Matrix

| Feature | ServiceNow | ManageEngine | Freshservice | Lansweeper | Snipe-IT | **Invenzo** | Status |
|---------|-----------|-------------|-------------|-----------|---------|------------|--------|
| Network Discovery | Agent + CMDB | WMI/SSH/SNMP | Limited | Agentless | None | Nmap+fping+WMI+SSH+SNMP | **Complete** |
| Agent-based | MID Server | Desktop Central | Yes | No | No | Go binary (mTLS, auto-deploy) | **Complete** |
| Cloud Discovery | AWS/Azure/GCP | Limited | AWS/Azure/GCP | Azure AD | No | AWS/Azure/GCP+Intune+Jamf+SCCM | **Complete** |
| AD/VMware | Both | AD only | Limited | AD | No | Both + Cloud sources | **Complete** |
| SAM/Licenses | Full module | Basic tracking | Compliance | Inventory only | Basic | Reconciliation+Catalog+SaaS | **Complete** |
| Compliance | GRC ($$) | Reports | Limited | CIS | No | 6 frameworks + auto-detect | **Complete** |
| Change Mgmt | Full ITIL | Basic | ITIL | No | No | Enterprise ITIL (CAB/PIR) | **Complete** |
| RBAC | Role+ACL+Domain | 5 roles | 4 roles | Basic | 8 roles | 3 roles + 60-page granular perms | **Complete** |
| Self-hosted | No (SaaS) | Yes | No (SaaS) | Optional | Yes (OSS) | Yes (Docker) | **Advantage** |
| Backup/Restore | SaaS managed | Built-in | Vendor managed | N/A | Manual | pg_dump + scheduled + cross-server | **Complete** |
| Procurement | Full module | PO tracking | Workflow | No | Basic PO | PO + contracts + cost centers | **Complete** |
| Reports | Analytics ($$$) | 150+ canned | Custom | Power BI | Basic | Custom reports + schedule | **Complete** |
| Webhooks | IntegrationHub | Limited | Marketplace | 600+ | API only | Full webhook system | **Complete** |

### 32.2 Invenzo Competitive Advantages

1. **Self-hosted privacy**: No data leaves customer premises (vs. ServiceNow/Freshservice SaaS-only)
2. **Go agent excellence**: Self-updating, auto-deploying, self-healing agent (better than most)
3. **Granular RBAC**: 60+ page-level permissions with per-user overrides (more granular than most)
4. **6 compliance frameworks**: HIPAA, SOC2, PCI-DSS, ISO 27001, NIST CSF, CIS Controls (more than mid-market)
5. **Unified discovery**: Network + Agent + Cloud + AD + VMware in single platform
6. **Enterprise CM**: Full ITIL with CAB, templates, policies, blackouts, PIR
7. **No per-asset pricing**: Fixed licensing vs. per-device models

### 32.3 Feature Gaps vs. Market Leaders

| Gap | Market Leaders | Priority | Recommendation |
|-----|---------------|----------|----------------|
| **CMDB Relationship Visualization** | ServiceNow, Lansweeper | HIGH | Enhance RelationshipGraph.tsx with impact analysis, service dependency maps |
| **AI/ML Auto-Categorization** | ServiceNow, Freshservice (Freddy AI) | MEDIUM | Add ML-based asset classification, anomaly detection, predictive failure |
| **Technology Normalization** | Lansweeper, ServiceNow | HIGH | Software dedup engine ("Office 365" = "M365" = "Microsoft 365") |
| **CVE/Vulnerability Mapping** | Lansweeper, ServiceNow | HIGH | Map installed software to NVD/CVE database for risk scoring |
| **Self-Service Portal** | ServiceNow, Freshservice | MEDIUM | End-user portal for asset request/return/check-out workflows |
| **Mobile Audit App** | Snipe-IT, Freshservice | MEDIUM | PWA/native app for barcode/QR scanning during physical audits |
| **Multi-tenancy (MSP)** | Freshservice, ManageEngine | LOW | Separate data per tenant for managed service providers |
| **Geo-map Visualization** | Lansweeper, InvGate | LOW | Map view of asset locations with heatmaps |
| **SLA Management** | ServiceNow, Freshservice | MEDIUM | Tie SLAs to assets and change requests with escalation |
| **Contract Auto-renewal** | ServiceNow | LOW | Vendor API integration for auto-renewal alerts |
| **Knowledge Base** | ServiceNow, Freshservice | LOW | Internal KB for IT procedures linked to assets |
| **Dashboard Customization** | ServiceNow | LOW | User-specific widget arrangement, custom KPIs |

### 32.4 Recommended Priority Roadmap

**Phase 1 (Immediate Impact)**:
1. Technology Normalization Engine — deduplicate software inventory
2. CVE/Vulnerability Integration — map software to NVD for risk scores
3. CMDB Relationship Impact Analysis — "what breaks if this goes down?"

**Phase 2 (Market Parity)**:
4. Self-Service Portal — end-user asset requests
5. Mobile Audit App — barcode/QR physical inventory
6. SLA Management — tied to change requests

**Phase 3 (Differentiation)**:
7. AI/ML Features — auto-classify, anomaly detection, predictive analytics
8. Geo-map Visualization — global asset location view
9. Multi-tenancy for MSPs

---

## Appendix A: Test Summary Statistics

| Category | Test Cases | Critical | High | Medium | Low |
|----------|-----------|----------|------|--------|-----|
| Authentication | 23 | 8 | 7 | 5 | 3 |
| Dashboard | 15 | 2 | 8 | 4 | 1 |
| Assets (List/Filter/Views) | 49 | 6 | 32 | 10 | 1 |
| Assets (Create/Bulk) | 21 | 4 | 12 | 4 | 1 |
| Asset Detail | 27 | 3 | 17 | 7 | 0 |
| Asset History | 7 | 0 | 4 | 2 | 1 |
| Asset Import | 8 | 2 | 4 | 2 | 0 |
| Discovery Center | 26 | 3 | 17 | 6 | 0 |
| Network Scanner | 8 | 0 | 6 | 2 | 0 |
| Cloud Explorer | 5 | 0 | 5 | 0 | 0 |
| Credentials | 13 | 3 | 8 | 2 | 0 |
| Automation | 18 | 3 | 13 | 2 | 0 |
| Agents | 13 | 2 | 9 | 2 | 0 |
| Auto-Deploy | 8 | 0 | 5 | 3 | 0 |
| Software & Licenses | 20 | 1 | 14 | 5 | 0 |
| Compliance | 19 | 2 | 14 | 3 | 0 |
| Tasks | 15 | 2 | 10 | 3 | 0 |
| Change Management | 17 | 2 | 12 | 3 | 0 |
| Reports | 10 | 0 | 8 | 2 | 0 |
| Locations | 12 | 2 | 7 | 3 | 0 |
| Procurement/Contracts | 11 | 0 | 9 | 2 | 0 |
| Budgets | 7 | 0 | 5 | 2 | 0 |
| SNMP & OUI | 12 | 0 | 6 | 4 | 2 |
| Audit Logs | 10 | 1 | 5 | 4 | 0 |
| Settings (All Tabs) | 41 | 2 | 26 | 11 | 2 |
| Users & RBAC | 36 | 7 | 26 | 3 | 0 |
| Notifications | 11 | 0 | 6 | 5 | 0 |
| Webhooks | 8 | 0 | 5 | 3 | 0 |
| Backup & Restore | 11 | 3 | 5 | 3 | 0 |
| Plugins & Licensing | 10 | 2 | 6 | 2 | 0 |
| Sidebar & Navigation | 17 | 2 | 9 | 4 | 2 |
| Cross-Cutting | 23 | 7 | 10 | 6 | 0 |
| Install Wizard | 8 | 2 | 4 | 2 | 0 |
| **TOTAL** | **580** | **70** | **358** | **121** | **13** |

## Appendix B: RBAC Permission Matrix (Complete)

### Admin Role — Full Access (all 60+ pages, all operations)

### Operator Role — Default Permissions

| Page Key | Operations | Notes |
|----------|------------|-------|
| dashboard | view | KPIs, charts only |
| assets | view, create, edit, export | No delete |
| discovery_overview | view | Stats only |
| discovery_subnets | view, create, edit, scan | No delete, no import |
| discovery_ad | view, scan | No create/edit/delete |
| discovery_vmware | view, scan | No create/edit/delete |
| discovery_cloud | view, scan | No create/edit/delete |
| discovery_rules | view | No CRUD |
| scanner_active | view, cancel | |
| scanner_history | view | No delete |
| locations | view, create, edit, export | No delete, no import |
| credentials | view, test | No CRUD |
| automation_jobs | view, create, edit, run | No delete |
| automation_schedules | view, create, edit | No delete |
| agents | view, install, update | No delete, no uninstall |
| modules | view | No toggle |
| licensing | view | No sync, no upload |
| software_inventory | view | |
| software_licenses | view | No CRUD |
| software_compliance | view | No export |
| snmp_vendors | view | No CRUD |
| snmp_mibs | view | No upload/delete |
| snmp_oui | view | No CRUD/import |
| mgmt_procurement | view, create, edit | No delete |
| mgmt_contracts | view, create, edit | No delete |
| mgmt_relationships | view, create, edit | No delete |
| mgmt_approvals | view, create | No review |
| mgmt_imports | view, create, export | |
| reports_reports | view, create, edit | No delete, no schedule |
| reports_history | view, export | |
| settings_lifecycle | view, transition | No CRUD |
| tasks | view, create, edit, assign, complete | No delete |
| task_templates | view, create, edit | No delete |
| change_management | view, create, edit | No approve |
| budgets | view, create, edit | No delete |
| compliance | view, edit | No delete |
| logs | view | No export |
| settings_general | view | No edit |
| settings_tag_numbering | view | No edit |
| settings_templates | view | No CRUD |
| settings_webhooks | view | No CRUD |

### Viewer Role — Read-Only

| Page Key | Operations |
|----------|------------|
| dashboard | view |
| assets | view |
| discovery_overview | view |
| discovery_subnets | view |
| scanner_history | view |
| locations | view |
| software_inventory | view |
| software_compliance | view |
| snmp_vendors | view |
| snmp_oui | view |
| mgmt_procurement | view |
| mgmt_contracts | view |
| mgmt_relationships | view |
| mgmt_approvals | view |
| reports_reports | view |
| reports_history | view |
| settings_lifecycle | view |
| logs | view |
| tasks | view |
| task_templates | view |
| change_management | view |
| budgets | view |
| compliance | view |
| modules | view |
| licensing | view |

---

*Generated: 2026-04-13 | Invenzo ITAM v1.7.29*
