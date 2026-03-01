# ☁️ Cloud Cost & Security Skills

> **64 production-ready AI skills** for cloud cost optimization and security — built for [OpenClaw.ai](https://openclaw.ai) + [Claude](https://anthropic.com), designed to use and sell.

---

## 🗂️ What's in This Repo

```
cloud-skills/
├── README.md                        ← You are here
├── skills.md                        ← Master index of all 64 skills
│
├── examples/                        ← Sample outputs (what customers will see)
│   ├── aws-cost.md
│   ├── azure-cost.md
│   ├── gcp-cost.md
│   ├── aws-security.md
│   ├── azure-security.md
│   ├── gcp-security.md
│   ├── kubernetes-cost.md
│   └── kubernetes-security.md
│
├── skills/                          ← Skill design specs (human-readable)
│   ├── aws-cost.md                  ← 8 AWS cost optimization skills
│   ├── azure-cost.md                ← 8 Azure cost optimization skills
│   ├── gcp-cost.md                  ← 8 GCP cost optimization skills
│   ├── aws-security.md              ← 8 AWS security skills
│   ├── azure-security.md            ← 8 Azure security skills
│   ├── gcp-security.md              ← 8 GCP security skills
│   ├── kubernetes-cost.md           ← 8 Kubernetes cost skills
│   └── kubernetes-security.md       ← 8 Kubernetes security skills
│
└── openclaw/                        ← Ready-to-deploy OpenClaw SKILL.md files
    ├── aws-cost/
    │   ├── spend-analyzer/SKILL.md
    │   ├── ri-savings-advisor/SKILL.md
    │   ├── idle-resource-detector/SKILL.md
    │   ├── spot-strategy/SKILL.md
    │   ├── tagging-auditor/SKILL.md
    │   ├── finops-report/SKILL.md
    │   ├── anomaly-explainer/SKILL.md
    │   └── data-transfer-optimizer/SKILL.md
    ├── azure-cost/          (8 skills)
    ├── gcp-cost/            (8 skills)
    ├── aws-security/        (8 skills)
    ├── azure-security/      (8 skills)
    ├── gcp-security/        (8 skills)
    ├── kubernetes-cost/     (8 skills)
    └── kubernetes-security/ (8 skills)
```

---

## 🧠 The Idea

Each skill is an **AI agent** that:
1. Takes cloud data as input (billing exports, IAM policies, audit logs, Terraform files, etc.)
2. Runs Claude as the reasoning engine
3. Outputs structured findings, remediation steps, and plain-English narratives
4. Can be sold as a standalone product or bundled into packs

**Stack**: OpenClaw.ai (skill runtime) + Claude (claude-3-5-sonnet / claude-3-opus)

---

## 📦 Skill Packs

| Pack | Skills | What It Does | Example | Sell For |
|---|---|---|---|---|
| **AWS Cost** | 8 | Spend analysis, RI/SP advisor, idle detector, Spot strategy, tagging, FinOps reports | [→](examples/aws-cost.md) | $29–$79/mo |
| **Azure Cost** | 8 | Spend analysis, Reservations+Hybrid Benefit, dev/test optimizer, bandwidth optimizer | [→](examples/azure-cost.md) | $29–$79/mo |
| **GCP Cost** | 8 | Spend analysis, CUD advisor, BigQuery optimizer, Spot VMs, networking optimizer | [→](examples/gcp-cost.md) | $29–$79/mo |
| **AWS Security** | 8 | IAM auditor, GuardDuty explainer, CloudTrail detector, S3 exposure, secrets scanner | [→](examples/aws-security.md) | $49–$199/mo |
| **Azure Security** | 8 | Entra ID auditor, NSG auditor, Defender reviewer, Key Vault auditor, Sentinel detector | [→](examples/azure-security.md) | $49–$199/mo |
| **GCP Security** | 8 | IAM auditor, VPC firewall, GCS exposure, SCC triage, service account auditor | [→](examples/gcp-security.md) | $49–$199/mo |
| **Kubernetes Cost** | 8 | Cluster cost breakdown, right-sizer, node pool optimizer, Spot nodes, chargeback reports | [→](examples/kubernetes-cost.md) | $29–$79/mo |
| **Kubernetes Security** | 8 | RBAC auditor, pod security, network policies, image scanner, Falco detector, CIS benchmark | [→](examples/kubernetes-security.md) | $49–$199/mo |

**Total: 64 skills across 8 packs**

---

## 🚀 Quick Start — Deploy on OpenClaw.ai

### 1. Install OpenClaw

```sh
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

### 2. Deploy a single skill

```sh
# Copy skill to your OpenClaw skills directory
cp -r openclaw/aws-security/iam-policy-auditor ~/.openclaw/skills/

# Restart OpenClaw to load the skill
openclaw restart
```

### 3. Deploy an entire pack

```sh
# Deploy all AWS Security skills at once
cp -r openclaw/aws-security/* ~/.openclaw/skills/

# Or deploy everything
cp -r openclaw/*/* ~/.openclaw/skills/
```

### 4. Use the skill

In any OpenClaw-connected channel (Slack, Discord, Terminal):

```
/skill aws-iam-policy-auditor

Paste your IAM policy JSON here and I'll audit it for security risks.
```

### 5. Publish to ClawHub (to sell)

```sh
# Publish a single skill to the marketplace
clawhub publish aws-iam-policy-auditor

# Publish an entire pack
for skill in openclaw/aws-security/*/; do
  clawhub publish $(basename $skill)
done
```

---

## 🏗️ SKILL.md Structure

Every skill follows this format:

```markdown
---
name: skill-name           # unique slug for OpenClaw
description: one-liner     # shown in ClawHub marketplace
tools: claude, bash        # tools the agent can use
version: 1.0.0
pack: aws-cost             # which pack this belongs to
tier: pro                  # free | pro | security | business | enterprise
price: 29/mo               # suggested selling price
---

# Skill Title

You are a [domain] expert. [Core mission statement.]

## Steps
[Numbered instructions for the agent]

## Output Format
[Exact structure the agent must follow]

## Rules
[Guardrails, edge cases, cloud-specific nuances]
```

---

## 💡 How to Customize a Skill

1. Open any `openclaw/<pack>/<skill>/SKILL.md`
2. Edit the instructions, output format, or rules to match your customers' needs
3. Update the `name` and `version` fields
4. Re-deploy with `cp` or `clawhub publish`

**Example customization** — add a customer's internal Jira project to the output:

```markdown
## Rules
- Always generate Jira ticket with project key: CLOUD-INFRA
- Use priority: P1 for Critical findings, P2 for High
```

---

## 💰 Monetization Strategy

### Pricing Tiers

| Tier | Price | Skills |
|---|---|---|
| **Free** | $0 | 1 skill per pack, 3–5 scans/mo |
| **Pro** | $29/mo | All cost optimization skills |
| **Security** | $49/mo | All security skills |
| **Business** | $79/mo | Cost + Security + FinOps reports |
| **Enterprise** | $199/mo | All skills + Compliance + API access |

### Sell as Packs on ClawHub

Each `openclaw/` subdirectory = one sellable product:
- **"AWS Cost Pack"** → `aws-cost/` → 8 skills
- **"Cloud Security Suite"** → `aws-security/` + `azure-security/` + `gcp-security/` → 24 skills
- **"K8s Full Bundle"** → `kubernetes-cost/` + `kubernetes-security/` → 16 skills

### Recommended Launch Order

1. `aws-security/iam-policy-auditor` — highest demand, great demo
2. `aws-cost/idle-resource-detector` — immediate ROI, easy to prove value
3. `kubernetes-security/rbac-auditor` — every K8s team needs this
4. `kubernetes-cost/resource-rightsizer` — 30–50% cost reclaim, tangible savings
5. `gcp-cost/bigquery-optimizer` — unique GCP skill, no close competitor

---

## 🔍 Skill Design Reference (`skills/`)

The `skills/` folder contains full human-readable design specs for each pack. Use these when:
- Customizing skills for specific customers
- Understanding what each skill does before deploying
- Building new skills using existing ones as templates
- Writing sales copy or documentation for ClawHub listings

Each spec includes: **Input → Output → Claude's Role → Cloud-specific nuances → Competitive edge → Pricing**

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Skill Runtime | OpenClaw.ai |
| AI Model | Claude (claude-3-5-sonnet / claude-3-opus) |
| Cloud Coverage | AWS · Azure · GCP · Kubernetes (EKS/AKS/GKE) |
| Output Formats | Markdown, JSON, YAML patches, PDF |
| Integrations | Slack, Teams, Jira, PagerDuty, GitHub, BigQuery, Grafana |
| Compliance Frameworks | CIS, SOC 2, HIPAA, PCI-DSS v4.0, MITRE ATT&CK Cloud |

---

## 📋 All 64 Skills

<details>
<summary><strong>AWS Cost (8 skills)</strong></summary>

| Skill | Description | Tier |
|---|---|---|
| `aws-spend-analyzer` | Top cost drivers, MoM anomalies, waste across all accounts | Pro |
| `aws-ri-savings-advisor` | Optimal RI/Savings Plan portfolio with break-even analysis | Pro |
| `aws-idle-resource-detector` | Zombie EC2, unattached EBS, idle RDS, orphaned IPs | Pro |
| `aws-spot-strategy` | Interruption-resilient Spot fleet design with Karpenter config | Pro |
| `aws-tagging-auditor` | Tag compliance score, unallocatable spend, AWS Config rules | Pro |
| `aws-finops-report` | Executive monthly report with team chargeback and savings | Business |
| `aws-anomaly-explainer` | Root cause diagnosis when Cost Anomaly Detection fires | Pro |
| `aws-data-transfer-optimizer` | Inter-AZ, NAT Gateway, egress cost reduction with VPC Endpoints | Business |

</details>

<details>
<summary><strong>Azure Cost (8 skills)</strong></summary>

| Skill | Description | Tier |
|---|---|---|
| `azure-spend-analyzer` | Top cost drivers across subscriptions with meter translation | Pro |
| `azure-reservations-hybrid-advisor` | Reservations + Hybrid Benefit stacking up to 80% savings | Pro |
| `azure-idle-resource-detector` | Deallocated VMs, unattached disks, idle App Gateways | Pro |
| `azure-devtest-optimizer` | Auto-shutdown schedules, Dev/Test pricing enrollment | Pro |
| `azure-tagging-auditor` | Tag compliance, unallocatable spend, Azure Policy enforcement | Pro |
| `azure-finops-report` | Monthly FinOps report with subscription chargeback | Business |
| `azure-anomaly-explainer` | Root cause when Azure Cost Management alerts fire | Pro |
| `azure-bandwidth-optimizer` | Egress reduction with CDN, Front Door, Private Endpoints | Business |

</details>

<details>
<summary><strong>GCP Cost (8 skills)</strong></summary>

| Skill | Description | Tier |
|---|---|---|
| `gcp-spend-analyzer` | SKU translation, top cost drivers, label coverage analysis | Pro |
| `gcp-cud-advisor` | Spend-based vs resource-based CUD recommendation with risk | Pro |
| `gcp-idle-resource-detector` | Idle Compute, unattached disks, unused static IPs | Pro |
| `gcp-spot-vm-strategy` | Preemptible/Spot VM strategy with MIG auto-restart config | Pro |
| `gcp-bigquery-optimizer` | Partition pruning, slot reservation, materialized views | Business |
| `gcp-networking-optimizer` | Egress reduction with Private Google Access, CDN, Interconnect | Business |
| `gcp-finops-report` | Monthly FinOps report across projects and folders | Business |
| `gcp-anomaly-explainer` | Root cause when GCP Budget Alerts fire | Pro |

</details>

<details>
<summary><strong>AWS Security (8 skills)</strong></summary>

| Skill | Description | Tier |
|---|---|---|
| `aws-iam-policy-auditor` | Wildcard permissions, admin-equivalent roles, MITRE ATT&CK mapping | Security |
| `aws-security-group-auditor` | 0.0.0.0/0 exposure, database ports, blast radius analysis | Security |
| `aws-s3-exposure-auditor` | Public buckets, dangerous ACLs, missing encryption/versioning | Security |
| `aws-cloudtrail-threat-detector` | Suspicious patterns, attack timeline, MITRE ATT&CK mapping | Security |
| `aws-secrets-scanner` | Hardcoded credentials in IaC, Lambda env vars, git history | Security |
| `aws-terraform-security-reviewer` | Pre-deployment IaC security review with CIS mapping | Security |
| `aws-compliance-analyzer` | CIS AWS / SOC 2 / HIPAA / PCI-DSS gap analysis + roadmap | Enterprise |
| `aws-guardduty-explainer` | Plain-English GuardDuty incident response playbook | Security |

</details>

<details>
<summary><strong>Azure Security (8 skills)</strong></summary>

| Skill | Description | Tier |
|---|---|---|
| `azure-entra-id-auditor` | Permanent admin roles, MFA gaps, legacy auth, PIM adoption | Security |
| `azure-nsg-firewall-auditor` | Internet-exposed ports, missing NSG, JIT VM Access | Security |
| `azure-storage-exposure-auditor` | Public blob containers, shared key rotation, missing protections | Security |
| `azure-defender-posture-reviewer` | Secure Score analysis, attack path, CISO narrative | Security |
| `azure-key-vault-auditor` | Public access, secret rotation, purge protection, RBAC | Security |
| `azure-terraform-security-reviewer` | azurerm + Bicep IaC security review with CIS mapping | Security |
| `azure-activity-log-detector` | Suspicious admin changes, brute force, Sentinel KQL rules | Security |
| `azure-compliance-analyzer` | CIS Azure / SOC 2 / ISO 27001 / HIPAA / PCI-DSS gap analysis | Enterprise |

</details>

<details>
<summary><strong>GCP Security (8 skills)</strong></summary>

| Skill | Description | Tier |
|---|---|---|
| `gcp-iam-policy-auditor` | Primitive roles, allUsers bindings, SA key risks, MITRE mapping | Security |
| `gcp-vpc-firewall-auditor` | Default VPC risks, open ports, missing logging, IAP recommendation | Security |
| `gcp-gcs-exposure-auditor` | Public buckets, uniform access, org-level prevention policy | Security |
| `gcp-scc-triage` | SCC finding prioritization, attack path analysis, Mandiant intel | Security |
| `gcp-service-account-auditor` | SA key age, primitive roles, Workload Identity migration | Security |
| `gcp-terraform-security-reviewer` | google provider IaC review with CIS GCP mapping | Security |
| `gcp-audit-log-detector` | Suspicious IAM changes, SA key creation, exfiltration patterns | Security |
| `gcp-compliance-analyzer` | CIS GCP / SOC 2 / HIPAA / PCI-DSS gap analysis + org policies | Enterprise |

</details>

<details>
<summary><strong>Kubernetes Cost (8 skills)</strong></summary>

| Skill | Description | Tier |
|---|---|---|
| `k8s-cluster-cost-analyzer` | Namespace/workload cost breakdown, idle %, chargeback table | Pro |
| `k8s-resource-rightsizer` | Requests vs actual usage, VPA config, 30–50% cost reclaim | Pro |
| `k8s-nodepool-optimizer` | Karpenter/Cluster Autoscaler tuning, bin-packing efficiency | Pro |
| `k8s-spot-node-strategy` | Spot/Preemptible node strategy for EKS/AKS/GKE | Pro |
| `k8s-idle-workload-detector` | Zombie Deployments, unbound PVCs, orphaned resources | Pro |
| `k8s-namespace-chargeback` | Monthly per-team cost statements with shared cost allocation | Business |
| `k8s-multi-cluster-comparator` | EKS vs AKS vs GKE true TCO comparison at scale | Business |
| `k8s-finops-report` | Monthly K8s FinOps report across all clusters | Business |

</details>

<details>
<summary><strong>Kubernetes Security (8 skills)</strong></summary>

| Skill | Description | Tier |
|---|---|---|
| `k8s-rbac-auditor` | cluster-admin, wildcard verbs, pods/exec, MITRE ATT&CK Containers | Security |
| `k8s-pod-security-auditor` | Privileged containers, hostPID/hostNetwork, PSA enforcement | Security |
| `k8s-network-policy-auditor` | Default-deny gaps, open east-west traffic, egress restrictions | Security |
| `k8s-image-scanner` | CVEs, EOL base images, mutable tags, admission controller policy | Security |
| `k8s-secrets-auditor` | Env var secrets, etcd encryption, External Secrets migration | Security |
| `k8s-iac-reviewer` | Terraform EKS/AKS/GKE + Helm chart security review | Security |
| `k8s-runtime-detector` | Falco alert triage, shell-in-container, crypto-mining, C2 patterns | Security |
| `k8s-cis-benchmark` | CIS EKS/AKS/GKE Benchmark compliance with kube-bench | Enterprise |

</details>

---

## ⚙️ CI/CD — Auto-Publish to ClawHub

Every push to `main` that changes a `SKILL.md` file automatically runs `clawhub publish` for only the skills that changed.

### Setup (one-time)

1. Add your ClawHub API token as a GitHub Actions secret:
   - Go to **Settings → Secrets → Actions → New repository secret**
   - Name: `CLAWHUB_TOKEN`
   - Value: your token from [clawhub.io/settings/tokens](https://clawhub.io/settings/tokens)

2. Push to `main` — the workflow triggers automatically.

### How it works

```
git push (changes openclaw/aws-security/iam-policy-auditor/SKILL.md)
    ↓
GitHub Actions detects changed SKILL.md files
    ↓
Extracts slug: iam-policy-auditor
    ↓
clawhub publish iam-policy-auditor
    ↓
✅ Live on ClawHub marketplace
```

Only changed skills are republished — no redundant publishes on all 64 skills.

---

## 🤝 Contributing

1. Use `skills/<pack>.md` as the spec format for new skill ideas
2. Convert to `openclaw/<pack>/<skill-name>/SKILL.md` for deployment
3. Follow the YAML frontmatter schema
4. Test the skill with real input before publishing to ClawHub

---

## 📄 License

Build with it. Sell with it. Just don't resell the raw skill files themselves.
