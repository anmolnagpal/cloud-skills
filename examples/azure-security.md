# Azure Security Pack — Example Outputs

Real-world sample outputs from two skills in this pack.

---

## 1. `azure-entra-id-auditor`

**Input**: Entra ID tenant `contoso.onmicrosoft.com`, exported March 2025

---

### 🔐 Entra ID Security Audit

**Tenant**: contoso.onmicrosoft.com | **Users**: 842 | **Scan date**: 2025-03-15  
**Risk Score**: 🔴 **79/100 (High)**

---

#### Executive Summary

| Severity | Findings |
|---|---|
| 🔴 Critical | 3 |
| 🟠 High | 7 |
| 🟡 Medium | 11 |

**Most urgent**: 4 Global Administrators (best practice: 2–4, but 3 have no MFA), legacy auth protocols still enabled, 14 service principals with Client Secret credentials older than 365 days.

---

#### 🔴 Critical Findings

**C-1: 3 Global Administrators without MFA**

| UPN | Last Sign-in | MFA Status | Risk |
|---|---|---|---|
| `john.smith@contoso.com` | 2 days ago | ❌ Disabled | Critical |
| `sarah.jones@contoso.com` | 12 days ago | ❌ Not registered | Critical |
| `it-admin@contoso.com` | 34 days ago | ❌ Disabled | Critical |

- **Fix**: Enforce MFA via Conditional Access Policy targeting `Directory Roles = Global Administrator`
- **MITRE ATT&CK**: T1078.004 — Valid Accounts

```json
{
  "displayName": "Require MFA for Admins",
  "state": "enabled",
  "conditions": {
    "users": { "includeRoles": ["62e90394-69f5-4237-9190-012177145e10"] }
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["mfa"]
  }
}
```

---

**C-2: Legacy Authentication Not Blocked**

- SMTP Auth, Basic Auth, and ActiveSync still enabled tenant-wide
- 23 sign-ins via legacy auth in last 30 days (7 users)
- Legacy auth bypasses MFA — this is the #1 account takeover vector
- **Fix**: Deploy "Block Legacy Authentication" Conditional Access policy immediately

---

**C-3: 14 Service Principals with Expired/Stale Secrets**

| App Name | Secret Age | Last Used | Risk |
|---|---|---|---|
| `legacy-erp-integration` | 847 days | 6 months ago | 🔴 Critical |
| `reporting-app-prod` | 412 days | Yesterday | 🟠 High — rotate now |
| `backup-automation` | 398 days | 3 days ago | 🟠 High — rotate now |
| *(11 more)* | 367–720 days | Various | 🟠 High |

- **Fix**: Implement secret rotation via Azure Key Vault + Managed Identity where possible

---

#### 🟠 High Findings (summary)

| ID | Issue | Count | Recommendation |
|---|---|---|---|
| H-1 | Privileged roles assigned directly (not via PIM) | 18 users | Migrate to PIM with approval workflow |
| H-2 | Guest users with Member permissions | 7 guests | Review and downgrade to Guest |
| H-3 | No Conditional Access for risky sign-ins | — | Enable Identity Protection + CA policy |
| H-4 | Break-glass accounts not excluded from CA policies | 2 accounts | Exclude from all CA, monitor with alerts |

---

## 2. `azure-nsg-firewall-auditor`

**Input**: All NSG rules across subscription `prod-workloads`, March 2025

---

### 🔥 NSG & Firewall Audit Report

**Subscription**: prod-workloads | **NSGs scanned**: 23 | **Total rules**: 418  
**Risk Score**: 🟠 **71/100 (High)**

---

#### Critical Exposure: Internet-Facing Rules

| NSG | Rule Name | Port | Source | Destination | Severity |
|---|---|---|---|---|---|
| `nsg-jumpbox` | `Allow-RDP-All` | 3389/TCP | `0.0.0.0/0` | VirtualNetwork | 🔴 Critical |
| `nsg-db-prod` | `Allow-SQL-Any` | 1433/TCP | `0.0.0.0/0` | `10.0.2.0/24` | 🔴 Critical |
| `nsg-app-prod` | `AllowAll-Inbound` | `*` | `0.0.0.0/0` | `*` | 🔴 Critical |
| `nsg-mgmt` | `Allow-SSH-All` | 22/TCP | `0.0.0.0/0` | VirtualNetwork | 🟠 High |
| `nsg-web` | `Allow-HTTP` | 80/TCP | `0.0.0.0/0` | `10.0.1.0/24` | 🟡 Medium |

---

#### 🔴 Critical: RDP (3389) open to internet on `nsg-jumpbox`

- **Resource**: `vm-jumpbox-prod` in `subnet-management`
- **Exposure**: Full RDP brute-force attack surface — average internet scanner hits open RDP within 2 minutes
- **MITRE ATT&CK**: T1110 — Brute Force, T1021.001 — Remote Services: RDP
- **Fix Option A** (Recommended): Enable Just-in-Time VM Access via Microsoft Defender for Cloud
- **Fix Option B**: Restrict source to corporate IP range only:

```bicep
resource nsgRule 'Microsoft.Network/networkSecurityGroups/securityRules@2023-04-01' = {
  name: 'Allow-RDP-CorpOnly'
  properties: {
    protocol: 'Tcp'
    sourcePortRange: '*'
    destinationPortRange: '3389'
    sourceAddressPrefix: '203.0.113.0/24'  // Replace with your corp IP
    destinationAddressPrefix: '*'
    access: 'Allow'
    priority: 100
    direction: 'Inbound'
  }
}
```

---

#### 🔴 Critical: SQL Server (1433) open to internet on `nsg-db-prod`

- **Resource**: `sql-server-prod` hosting 3 databases
- **Exposure**: Credential stuffing + SQL injection attempts visible in Azure Monitor logs (847 blocked attempts in last 30 days)
- **Fix**: Remove rule entirely. Use Azure Private Endpoint for SQL + restrict to app subnet `10.0.1.0/24` only

---

#### Subnet Coverage Gaps

| Subnet | VMs | NSG Assigned | Risk |
|---|---|---|---|
| `subnet-data` | 4 | ❌ None | 🔴 Critical |
| `subnet-monitoring` | 2 | ❌ None | 🟠 High |
| `subnet-app` | 8 | ✅ `nsg-app-prod` | — |

> **2 subnets have no NSG** — any traffic is allowed by default. Assign NSGs immediately.
