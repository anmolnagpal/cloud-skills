# AWS Security Pack — Example Outputs

Real-world sample outputs from two skills in this pack.

---

## 1. `aws-iam-policy-auditor`

**Input**: IAM policies and roles for account `123456789012`, exported March 2025

---

### 🔐 IAM Policy Audit Report

**Account**: 123456789012 | **Scan date**: 2025-03-15  
**Risk Score**: 🔴 **87/100 (Critical)**

---

#### Executive Summary

| Severity | Findings |
|---|---|
| 🔴 Critical | 4 |
| 🟠 High | 9 |
| 🟡 Medium | 14 |
| 🟢 Low | 7 |

**Most urgent**: 2 roles with effective `AdministratorAccess`, 3 roles allowing `iam:PassRole` to `*`, 1 human user with programmatic access + no MFA.

---

#### 🔴 Critical Findings

**C-1: `DataPipelineRole` has effective AdministratorAccess**

```json
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```

- **MITRE ATT&CK**: T1078.004 — Valid Accounts: Cloud Accounts
- **Blast radius**: Full account compromise. Can create IAM users, disable CloudTrail, access all S3 buckets.
- **Why it exists**: Copied from a dev policy, never tightened for prod.
- **Fix**: Replace `*` actions with `s3:GetObject`, `s3:PutObject`, `glue:StartJobRun` (actual permissions needed)

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject",
    "glue:StartJobRun",
    "glue:GetJobRun"
  ],
  "Resource": [
    "arn:aws:s3:::data-pipeline-prod/*",
    "arn:aws:glue:us-east-1:123456789012:job/etl-*"
  ]
}
```

---

**C-2: `DeploymentRole` allows `iam:PassRole` to `*`**

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "*"
}
```

- **MITRE ATT&CK**: T1548.005 — Abuse Elevation Control Mechanism: Temporary Elevated Cloud Access
- **Blast radius**: Can escalate to any role in the account, including admin roles.
- **Fix**: Scope to specific roles:

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::123456789012:role/deploy-*"
}
```

---

**C-3: User `ci-bot` has access keys + no MFA, active 487 days**

- Last key rotation: Never
- Last used: 3 hours ago (us-east-1, `s3:PutObject`)
- **Fix**: Rotate key immediately, enforce MFA via SCP, or replace with IAM role + OIDC (GitHub Actions / Jenkins)

---

**C-4: `LambdaExecRole-legacy` allows `s3:*` on `*`**

- 14 Lambda functions use this role; 11 only need `s3:GetObject` on a single bucket
- **Fix**: Create per-function roles with least-privilege S3 access

---

#### 🟠 High Findings (summary)

| ID | Role/User | Issue | Fix |
|---|---|---|---|
| H-1 | `EC2InstanceProfile` | `ec2:*` on `*` | Scope to instance operations only |
| H-2 | `BackupRole` | No resource constraint on `kms:Decrypt` | Scope to backup KMS key ARN |
| H-3 | `developer-group` | `iam:CreateUser` allowed | Remove — use SSO |
| H-4 | `MonitoringRole` | `logs:*` includes `logs:DeleteLogGroup` | Remove delete permissions |
| H-5 | 7 roles | No permission boundary | Attach `DeveloperBoundary` policy |

---

#### Remediation Priority Order

1. `DataPipelineRole` — fix `Action: *` → 30 min effort, critical risk
2. `DeploymentRole` — scope `iam:PassRole` → 15 min effort, critical risk
3. Rotate `ci-bot` access key → 10 min effort, critical risk
4. `LambdaExecRole-legacy` — split into per-function roles → 2hr effort, high risk

---

## 2. `aws-guardduty-explainer`

**Input**: GuardDuty finding `a1b2c3d4e5f67890a1b2c3d4e5f67890` exported as JSON

---

### 🚨 GuardDuty Incident Response — Finding Explained

**Finding**: `UnauthorizedAccess:IAMUser/MaliciousIPCaller.Custom`  
**Severity**: 🔴 **8.3 / 10 (High)**  
**Detected**: 2025-03-14 02:17:43 UTC

---

#### What Happened (Plain English)

Your IAM user `deploy-prod` made AWS API calls from IP address `185.220.101.47` at 2:17 AM UTC. This IP is on a **known Tor exit node list** — commonly used to anonymize attacker activity.

The calls made were:
- `iam:ListUsers` — enumerated all IAM users in the account
- `iam:GetUser` on 3 users
- `s3:ListBuckets` — listed all S3 buckets
- `ec2:DescribeInstances` — listed all EC2 instances in all regions

This is a **reconnaissance pattern** — an attacker with stolen credentials mapping your environment before taking action.

---

#### Timeline

| Time (UTC) | Action | Source IP | Region |
|---|---|---|---|
| 02:17:43 | `iam:ListUsers` | 185.220.101.47 | us-east-1 |
| 02:17:44 | `iam:GetUser` (x3) | 185.220.101.47 | us-east-1 |
| 02:17:46 | `s3:ListBuckets` | 185.220.101.47 | us-east-1 |
| 02:17:49 | `ec2:DescribeInstances` | 185.220.101.47 | us-east-1 |
| 02:17:51 | `ec2:DescribeInstances` | 185.220.101.47 | eu-west-1 |
| 02:17:53 | `ec2:DescribeInstances` | 185.220.101.47 | ap-southeast-1 |

---

#### Immediate Response Steps

```bash
# Step 1: Disable the compromised access key immediately
aws iam update-access-key \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --status Inactive \
  --user-name deploy-prod

# Step 2: Attach an explicit deny policy to stop all actions while investigating
aws iam put-user-policy \
  --user-name deploy-prod \
  --policy-name EmergencyDenyAll \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Deny","Action":"*","Resource":"*"}]}'

# Step 3: Check for any resources created by this user in the last 24h
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=deploy-prod \
  --start-time 2025-03-13T00:00:00Z \
  --end-time 2025-03-14T06:00:00Z \
  --query 'Events[?EventName!=`DescribeInstances` && EventName!=`ListBuckets` && EventName!=`ListUsers`]'
```

---

#### What To Check Next

- [ ] Were any IAM users or access keys **created** after 02:17? (check CloudTrail for `CreateUser`, `CreateAccessKey`)
- [ ] Were any S3 bucket policies **modified** or objects downloaded?
- [ ] Any EC2 instances or Lambda functions **launched** in regions you don't normally use?
- [ ] Where was this access key last legitimately used? (check `last_used_date` — if CI/CD, rotate the secret in your pipeline)

---

#### MITRE ATT&CK Mapping

| Technique | ID | Description |
|---|---|---|
| Valid Accounts: Cloud Accounts | T1078.004 | Using stolen IAM credentials |
| Account Discovery | T1087.004 | `iam:ListUsers` enumeration |
| Cloud Infrastructure Discovery | T1580 | `ec2:DescribeInstances` across regions |
| Cloud Service Discovery | T1526 | `s3:ListBuckets` |

---

#### Prevention

1. **Never use IAM users for CI/CD** — replace `deploy-prod` with IAM Roles + OIDC (GitHub Actions, Jenkins)
2. **Enable GuardDuty Malware Protection** and S3 Protection if not already on
3. **Create a GuardDuty → EventBridge → Lambda** auto-remediation rule to disable keys automatically on High severity findings
