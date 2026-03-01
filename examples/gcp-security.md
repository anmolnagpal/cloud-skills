# GCP Security Pack — Example Outputs

Real-world sample outputs from two skills in this pack.

---

## 1. `gcp-iam-policy-auditor`

**Input**: IAM policy export for GCP project `my-analytics-prod`, March 2025

---

### 🔐 GCP IAM Policy Audit

**Project**: my-analytics-prod | **Scan date**: 2025-03-15  
**Risk Score**: 🔴 **82/100 (Critical)**

---

#### Executive Summary

| Severity | Findings |
|---|---|
| 🔴 Critical | 5 |
| 🟠 High | 8 |
| 🟡 Medium | 12 |

---

#### 🔴 Critical Findings

**C-1: `allUsers` binding on `roles/storage.objectViewer` — GCS bucket `raw-data-prod`**

```json
{
  "role": "roles/storage.objectViewer",
  "members": ["allUsers"]
}
```

- **Impact**: All objects in `raw-data-prod` are publicly readable by anyone on the internet
- **MITRE ATT&CK**: T1530 — Data from Cloud Storage
- **Bucket contains**: 847 GB of customer event data — PII exposure risk, potential GDPR violation
- **Fix**:

```bash
# Remove allUsers binding immediately
gcloud storage buckets remove-iam-policy-binding gs://raw-data-prod \
  --member="allUsers" \
  --role="roles/storage.objectViewer"

# Enable uniform bucket-level access to prevent ACL bypasses
gcloud storage buckets update gs://raw-data-prod \
  --uniform-bucket-level-access
```

---

**C-2: 3 users with `roles/owner` on project (primitive role)**

| Member | Last Active | Justification |
|---|---|---|
| `ali@company.com` | 8 days ago | Developer — should be `roles/editor` at most |
| `old-contractor@gmail.com` | 247 days ago | Former contractor — should be removed |
| `john@company.com` | 1 day ago | DevOps lead — keep, but downgrade to custom role |

- **MITRE ATT&CK**: T1078.004 — Valid Accounts: Cloud Accounts
- **Fix**: Replace `roles/owner` with specific roles. Remove `old-contractor@gmail.com` immediately.

```bash
# Remove former contractor immediately
gcloud projects remove-iam-policy-binding my-analytics-prod \
  --member="user:old-contractor@gmail.com" \
  --role="roles/owner"
```

---

**C-3: Service account `sa-data-pipeline@` has `roles/editor` (primitive)**

- Used by 3 Cloud Run services and 1 Cloud Function
- `roles/editor` grants read/write to almost all GCP services
- Actual permissions needed: `bigquery.dataEditor` on dataset `raw`, `storage.objectCreator` on bucket `raw-data-prod`
- **MITRE ATT&CK**: T1548 — Abuse Elevation Control Mechanism

```bash
# Create a least-privilege custom role
gcloud iam roles create DataPipelineRole \
  --project=my-analytics-prod \
  --permissions="bigquery.tables.create,bigquery.tables.updateData,storage.objects.create"

# Remove primitive role, assign custom role
gcloud projects remove-iam-policy-binding my-analytics-prod \
  --member="serviceAccount:sa-data-pipeline@my-analytics-prod.iam.gserviceaccount.com" \
  --role="roles/editor"

gcloud projects add-iam-policy-binding my-analytics-prod \
  --member="serviceAccount:sa-data-pipeline@my-analytics-prod.iam.gserviceaccount.com" \
  --role="projects/my-analytics-prod/roles/DataPipelineRole"
```

---

**C-4: Service account key file 718 days old — `sa-reporting@`**

- Key created: 2023-03-12 (never rotated)
- Used in production Cloud Scheduler job
- **Fix**: Migrate to Workload Identity (preferred) or rotate key immediately

```bash
# Check all keys for this SA
gcloud iam service-accounts keys list \
  --iam-account=sa-reporting@my-analytics-prod.iam.gserviceaccount.com

# Disable old key after creating new one
gcloud iam service-accounts keys disable KEY_ID \
  --iam-account=sa-reporting@my-analytics-prod.iam.gserviceaccount.com
```

---

## 2. `gcp-scc-triage`

**Input**: Security Command Center findings export, 47 active findings, March 2025

---

### 🛡️ GCP Security Command Center Triage

**Project**: my-analytics-prod | **Active findings**: 47  
**Prioritized for immediate action**: 6 findings

---

#### Severity Distribution

| Severity | Count | Change since last scan |
|---|---|---|
| 🔴 Critical | 2 | ▲ +1 |
| 🟠 High | 8 | — |
| 🟡 Medium | 23 | ▲ +4 |
| 🔵 Low | 14 | ▼ -2 |

---

#### Top 6 Prioritized Findings

**P1 — Public GCS Bucket with Sensitive Data** (Critical, NEW)
- **Finding ID**: `organizations/123/sources/456/findings/abc123`
- **Asset**: `gs://raw-data-prod`
- **Why P1**: Public bucket + sensitive data classification + GDPR exposure
- **Action**: See IAM audit above — remove `allUsers` binding within the hour

---

**P2 — SSH Exposed to Internet on GCE Instance** (Critical)
- **Asset**: `vm-bastion-prod` in `us-central1-a`
- **Firewall rule**: `default-allow-ssh` — port 22 open to `0.0.0.0/0`
- **MITRE ATT&CK**: T1021.004 — Remote Services: SSH
- **Action**:

```bash
# Delete the overly-permissive rule
gcloud compute firewall-rules delete default-allow-ssh --quiet

# Create restricted rule (IAP source range only)
gcloud compute firewall-rules create allow-ssh-iap \
  --allow tcp:22 \
  --source-ranges 35.235.240.0/20 \
  --description "Allow SSH via IAP only"
```

---

**P3 — Audit Logging Disabled for Cloud SQL** (High)
- **Asset**: `cloudsql-prod-mysql`
- Data access audit logs (DATA_READ, DATA_WRITE) not enabled
- Required for PCI-DSS 10.x compliance
- **Action**: Enable via Cloud Console → IAM & Admin → Audit Logs → Cloud SQL

---

**P4 — GKE Cluster Running End-of-Life Version** (High)
- **Asset**: `gke-prod-cluster` running Kubernetes 1.27.8-gke (EOL)
- Current stable channel: 1.31.x
- **Action**: Upgrade control plane first, then node pools (schedule maintenance window)

---

**P5 — Unused Service Account with Editor Role** (High)
- See C-2 in IAM audit above — `old-contractor@gmail.com`
- SCC cross-correlates IAM findings: this account also has GCS `objectAdmin` on 4 buckets
- **Action**: Remove all bindings for this account immediately

---

**P6 — Container Image Vulnerability: CVE-2025-1234 (CVSS 9.1)** (High)
- **Asset**: Cloud Run service `api-prod` using image `gcr.io/my-analytics-prod/api:latest`
- Vulnerability in `openssl 3.0.7` — arbitrary code execution
- **Action**: Rebuild with `openssl 3.3.1+`, redeploy, enable Binary Authorization to block vulnerable images in future

---

#### SCC Findings Noise Reduction

37 of 47 findings are **informational or low-priority**:
- 14 "Default network exists" — acceptable for this org, suppress with finding mute
- 9 "Web UI enabled on GKE" — intentional, suppress
- 8 "HTTP load balancer without SSL policy" — dev environments, suppress
- 6 others — accepted risk

**After muting**: 10 actionable findings remain (6 above + 4 medium)
