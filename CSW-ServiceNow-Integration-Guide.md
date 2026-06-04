# Cisco Secure Workload — ServiceNow CMDB Integration Guide

> **Disclaimer:** Community reference guide by Cisco Solutions Engineering. Always consult [official Cisco Secure Workload documentation](https://www.cisco.com/c/en/us/products/security/tetration/index.html) for authoritative guidance.

## Table of Contents
1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [What CSW Gets from ServiceNow](#3-what-csw-gets-from-servicenow)
4. [Use Cases](#4-use-cases)
5. [Prerequisites](#5-prerequisites)
6. [Step A — Prepare ServiceNow](#6-step-a--prepare-servicenow)
7. [Step B — Configure the ServiceNow Connector on CSW](#7-step-b--configure-the-servicenow-connector-on-csw)
8. [Verification](#8-verification)
9. [Sync Interval and Data Refresh](#9-sync-interval-and-data-refresh)
10. [Limits](#10-limits)
11. [Troubleshooting](#11-troubleshooting)
12. [Related Resources](#12-related-resources)

---

## 1. Overview

The **ServiceNow connector** in Cisco Secure Workload connects to a **ServiceNow instance** to pull **CMDB (Configuration Management Database) labels** for endpoints and enrich CSW workload inventory.

This integration bridges the gap between ITSM-managed asset data (application owner, environment, business unit, CI class, etc.) and CSW workload policy — enabling **CMDB-driven microsegmentation** without manual label maintenance in CSW.

### Why it matters
- CSW scopes stay **accurate automatically** as CMDB records are updated — no manual label tagging
- Enables policy based on **business-meaningful attributes** (app owner, tier, environment, compliance class)
- Connects infrastructure segmentation to **ITSM change management workflows**
- Critical for financial-services environments where CMDB is the authoritative source of asset truth

### Supported ServiceNow tables
Any ServiceNow table that has an **IP address field** can be used. Common examples:
- `cmdb_ci_server` — Physical/virtual server CIs
- `cmdb_ci_linux_server` — Linux servers
- `cmdb_ci_win_server` — Windows servers
- `cmdb_ci_app_server` — Application servers
- Custom CMDB tables with IP address fields
- Scripted REST APIs with IP address support

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Enterprise Environment                     │
│                                                                   │
│  ┌─────────────────────────┐    HTTPS REST      ┌─────────────┐  │
│  │    ServiceNow Instance   │◄───────────────────│ CSW Edge    │  │
│  │                          │   (Table API)      │ Appliance   │  │
│  │  CMDB Tables:            │                    │             │  │
│  │  • cmdb_ci_server        │                    │ ServiceNow  │  │
│  │  • cmdb_ci_linux_server  │                    │ Connector   │  │
│  │  • cmdb_ci_win_server    │                    └──────┬──────┘  │
│  │  • Custom CI tables      │                           │         │
│  └─────────────────────────┘                           │         │
│                                                         │ Labels  │
│  ┌──────────────────────────────────────────────────────▼──────┐  │
│  │              Cisco Secure Workload Cluster                   │  │
│  │                                                              │  │
│  │  Inventory enrichment:                                       │  │
│  │    10.0.0.5 → {app_name: "PaymentAPI", owner: "FinanceIT",  │  │
│  │                environment: "Production", tier: "App"}       │  │
│  │                                                              │  │
│  │  Scopes (CMDB-driven):                                       │  │
│  │    "Production-Finance-App" = environment=Production AND     │  │
│  │                               business_unit=Finance          │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. What CSW Gets from ServiceNow

Any attribute from a configured ServiceNow table can be used as a CSW workload label. Common CMDB attributes:

| ServiceNow Field | CSW Label Example | Use in Policy |
|------------------|-------------------|--------------|
| `name` | `cmdb/name = PaymentAPI` | Identify workloads by CI name |
| `environment` | `cmdb/environment = Production` | Scope by env (Prod/Dev/Test) |
| `u_business_unit` | `cmdb/business_unit = Finance` | Business-unit–based scopes |
| `u_application_owner` | `cmdb/owner = FinanceIT-Team` | Policy review routing |
| `u_tier` | `cmdb/tier = App` | 3-tier app segmentation (Web/App/DB) |
| `u_compliance_class` | `cmdb/compliance = PCI` | Compliance-based policy tightening |
| `operational_status` | `cmdb/status = 1` | Exclude decommissioned CIs |
| `support_group` | `cmdb/support_group = Infra-Ops` | Ops-team ownership labels |

Up to **10 attributes per table** can be configured. Multiple tables can be added.

---

## 4. Use Cases

### Use Case 1 — CMDB-Driven Scope Automation
**Scenario:** Financial services has 2,000 servers. Manually labeling each in CSW is impractical.

**With ServiceNow connector:**
- CSW scope `PCI-Servers` = `{ cmdb/compliance_class == 'PCI' }` — automatically includes all CIs tagged as PCI in ServiceNow
- As new servers are added to ServiceNow and tagged PCI, they automatically join the scope and inherit the correct policies

### Use Case 2 — Environment-Based Policy
**Scenario:** Ensure Production servers are never directly reachable from Development servers.

```
Policy:
  Consumer: Dev-Servers  (scope: cmdb/environment = Development)
  Provider: Prod-Servers (scope: cmdb/environment = Production)
  Action:   DENY ALL
```
CMDB is the authoritative source — policy enforces itself as the CMDB changes.

### Use Case 3 — 3-Tier Application Isolation
**Scenario:** Enforce proper tier-to-tier communication for multi-tier apps.

With `cmdb/tier` labels (Web, App, DB), CSW policies enforce:
- External → Web tier only
- Web tier → App tier only
- App tier → DB tier only
- No cross-tier or tier-skipping allowed

### Use Case 4 — Application Owner Accountability
**Scenario:** Show application owners their application's segmentation posture.

Filter CSW **Policy Analysis** by `cmdb/owner` to give each team a view of only their application's policies — scoped access and accountability without admin privileges.

---

## 5. Prerequisites

### ServiceNow side
- [ ] ServiceNow instance accessible via HTTPS from CSW Edge appliance
- [ ] ServiceNow user account with roles:
  - `cmdb_read` — for standard CMDB tables
  - `web_service_admin` — for Scripted REST APIs (if used)
- [ ] Tables to integrate have an **IP address field** (`ip_address` or equivalent)
- [ ] (If using Scripted REST APIs) APIs must support `sysparm_limit`, `sysparm_fields`, `sysparm_offset` query parameters; no path parameters

### CSW side
- [ ] **CSW Edge appliance** deployed and registered
- [ ] Edge appliance has network access to ServiceNow instance HTTPS endpoint
- [ ] Labels / annotations enabled on the target VRF/tenant

---

## 6. Step A — Prepare ServiceNow

### A1 — Create or identify a service account

Create a dedicated ServiceNow user for the CSW connector:
1. Navigate to **User Administration > Users > New**
2. Set username (e.g., `csw_connector_svc`)
3. Assign roles:
   - `cmdb_read`
   - `web_service_admin` (if using Scripted REST APIs)
4. Set a strong password

### A2 — Identify the tables to integrate

Common tables for server inventory:

| Table Name | Description |
|------------|-------------|
| `cmdb_ci_server` | Base server CI class |
| `cmdb_ci_linux_server` | Linux servers |
| `cmdb_ci_win_server` | Windows servers |
| `cmdb_ci_app_server` | Application servers |
| `cmdb_ci_db_instance` | Database instances |

Verify each table has an IP address field:
```
GET https://<instance>.service-now.com/api/now/table/cmdb_ci_server?sysparm_fields=ip_address,name&sysparm_limit=1
```

### A3 — (Optional) Create a Scripted REST API

For custom data not in standard tables, create a ServiceNow Scripted REST API:
1. Navigate to **System Web Services > Scripted REST APIs > New**
2. Create an API with a `GET` method that returns records with `ip_address`, plus your custom fields
3. Ensure it supports `sysparm_limit`, `sysparm_fields`, `sysparm_offset`
4. Note the API namespace and path for use in CSW connector configuration

---

## 7. Step B — Configure the ServiceNow Connector on CSW

### B1 — Navigate to connector configuration

1. CSW UI: **Manage > Virtual Appliances**
2. Select **Edge appliance**
3. **Connectors** tab → **+ Add Connector** → **ServiceNow**

### B2 — ServiceNow instance credentials

| Field | Description | Example |
|-------|-------------|---------|
| **ServiceNow Instance URL** | Full URL of the ServiceNow instance | `https://mycompany.service-now.com` |
| **Username** | Service account username | `csw_connector_svc` |
| **Password** | Service account password | (encrypted by CSW) |
| **Include Scripted APIs** | Enable if using Scripted REST APIs | Checkbox |

### B3 — Table discovery and selection

After entering credentials, click **Discover Tables**. CSW will:
1. Query ServiceNow for available tables (and Scripted REST APIs if enabled)
2. Present a list of discovered tables

For each table you want to integrate:
1. Select the table from the list
2. Choose the **IP address field** as the primary key (typically `ip_address`)
3. Select **up to 10 additional attributes** to import as CSW labels

### B4 — Attribute mapping example

For `cmdb_ci_server`:

| ServiceNow Attribute | Map to CSW Label |
|---------------------|-----------------|
| `ip_address` | (primary key — not a label) |
| `name` | `cmdb/name` |
| `u_environment` | `cmdb/environment` |
| `u_business_unit` | `cmdb/business_unit` |
| `u_tier` | `cmdb/tier` |
| `u_compliance_class` | `cmdb/compliance_class` |
| `operational_status` | `cmdb/operational_status` |

### B5 — Set sync interval

Default sync interval: **60 minutes**. Adjust based on how frequently your CMDB changes:
- High-change environments: 30 minutes
- Stable environments: 120–240 minutes

### B6 — Apply configuration

Click **Test and Apply**. The connector will:
1. Perform an initial full sync of the configured tables
2. Annotate matching IP addresses in CSW inventory with CMDB labels
3. Schedule periodic syncs at the configured interval

---

## 8. Verification

### Check connector status
**Manage > Virtual Appliances > [Edge] > Connectors**
ServiceNow connector should show **Status: Active**

### Check label enrichment in inventory
1. Navigate to **Inventory > Workloads**
2. Filter by a server you know is in ServiceNow (search by IP)
3. Confirm `cmdb/` labels are populated

### Verify in CSW Explore
Run an Explore query to list all IPs enriched with CMDB labels:
```
label cmdb/environment exists
```

### Delete stale labels (if needed)
If you need to remove labels added by the ServiceNow connector (e.g., after a table reconfiguration):
```
# From Explore command UI in CSW:
labels delete --connector servicenow --vrf <VRF_ID>
```

---

## 9. Sync Interval and Data Refresh

| Task | Frequency | Description |
|------|-----------|-------------|
| Initial sync | On connector start | Full pull of all configured tables |
| Periodic sync | Configurable (default 60 min) | Incremental update of changed records |
| Label cleanup | On sync | IPs no longer in ServiceNow lose CMDB labels |

---

## 10. Limits

| Metric | Limit |
|--------|-------|
| Attributes per table | 10 |
| Tables per ServiceNow instance config | Multiple (no hard limit) |
| Scripted REST API path parameters | Not supported |
| ServiceNow connector per Edge appliance | 1 recommended |
| IP field requirement | Table must have an IP address field |

---

## 11. Troubleshooting

| Symptom | Check |
|---------|-------|
| Connector fails to authenticate | Verify username/password; confirm account is active in ServiceNow |
| No tables discovered | Check `cmdb_read` role assigned; verify HTTPS connectivity from Edge to ServiceNow |
| IP labels not appearing | Confirm the IP address field name in the table is correctly mapped; verify IPs in ServiceNow match IPs visible to CSW |
| Stale labels after CI deletion | Labels are removed on next sync cycle; force sync by restarting connector |
| Scripted REST API not listed | Confirm `web_service_admin` role; verify no path parameters in the API |

---

## 12. Related Resources

| Repository | Description | Best for |
|------------|-------------|---------|
| [CSW-Agent-Installation-Guide](https://github.com/chandrapati/CSW-Agent-Installation-Guide) | Deploy CSW agents on Linux/Windows workloads | Server-side agent deployment |
| [CSW-Policy-Lifecycle](https://github.com/chandrapati/CSW-Policy-Lifecycle) | Policy discovery → enforcement workflow | Policy management |
| [csw-ise-integration](https://github.com/chandrapati/csw-ise-integration) | ISE/pxGrid: user-identity–aware policy | Identity-aware policy |
| [CSW-Compliance-Mapping](https://github.com/chandrapati/CSW-Compliance-Mapping) | Map CSW to NIST, PCI-DSS, HIPAA, CIS | Compliance reporting |
| [CSW-Operations-Toolkit](https://github.com/chandrapati/CSW-Operations-Toolkit) | Day-2 ops scripts: health, reporting, policy | Ongoing operations |

> **Best practice:** Combine ServiceNow CMDB labels (environment, tier, compliance class) with ISE SGT labels (user identity) for the richest policy context in your tenant.

---
*Community reference — Cisco Solutions Engineering. Not an official Cisco product document.*
