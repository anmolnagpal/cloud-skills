# AWS Cost Pack — Example Outputs

Real-world sample outputs from two skills in this pack.

---

## 1. `aws-spend-analyzer`

**Input**: AWS Cost & Usage Report (CUR) for account `123456789012`, March 2025

---

### 📊 AWS Spend Analysis — March 2025

**Total Spend**: $48,312.40 | **vs Feb**: ▲ 12.4% (+$5,340)

---

#### Top 10 Cost Drivers

| Rank | Service | Cost | MoM Change | % of Total |
|------|---------|------|-----------|------------|
| 1 | Amazon EC2 | $18,440 | ▲ 8% | 38.2% |
| 2 | Amazon RDS | $9,120 | ▲ 22% | 18.9% |
| 3 | Amazon S3 | $4,880 | ▲ 3% | 10.1% |
| 4 | AWS Lambda | $3,210 | ▼ 5% | 6.6% |
| 5 | Amazon CloudFront | $2,740 | ▲ 41% | 5.7% |
| 6 | Amazon EKS | $2,310 | ▲ 18% | 4.8% |
| 7 | AWS Glue | $1,980 | ▲ 67% | 4.1% |
| 8 | Data Transfer | $1,840 | ▲ 11% | 3.8% |
| 9 | Amazon ElastiCache | $1,620 | — | 3.4% |
| 10 | Amazon Route 53 | $420 | ▼ 2% | 0.9% |

---

#### 🚨 Anomalies Detected (3)

**1. AWS Glue — 67% spike ($800 increase)**
- Root cause: `etl-pipeline-prod` job running every 15 min (was hourly)
- Scheduled change deployed March 3 by `terraform-ci` role
- **Fix**: Revert schedule or use Glue Triggers instead of time-based — saves ~$900/mo

**2. Amazon RDS — 22% spike (+$1,640)**
- New `db.r6g.2xlarge` instance `analytics-db-prod` provisioned March 8 (untagged, no owner)
- Running at 4% CPU average — severely over-provisioned
- **Fix**: Right-size to `db.r6g.large` or use Aurora Serverless v2 — saves ~$1,200/mo

**3. Amazon CloudFront — 41% spike (+$800)**
- HTTPS request volume up 38% — correlates with `marketing-site` traffic spike
- Legitimate growth, not waste
- **Action**: Enable CloudFront compression + cache-hit optimization (currently 42% hit rate vs 80% benchmark)

---

#### 💡 Top Savings Opportunities

| Opportunity | Estimated Savings/mo | Effort |
|---|---|---|
| Right-size `analytics-db-prod` RDS | $1,200 | Low |
| Purchase EC2 Savings Plans (3-yr, 40% on-demand) | $4,400 | Low |
| Fix Glue ETL schedule | $900 | Low |
| Enable S3 Intelligent-Tiering (82% of objects >30 days old) | $620 | Low |
| Replace NAT Gateway with VPC Endpoints for S3/DynamoDB | $380 | Medium |
| **Total identified savings** | **$7,500/mo** | |

---

#### 📋 Tag Compliance

- **Tagged spend**: 61% | **Untagged**: 39% ($18,841)
- Missing tags: `Environment`, `Owner`, `CostCenter` on 47 resources
- Largest untagged: `analytics-db-prod` ($3,840), `eks-node-group-prod` ($2,310)

---

## 2. `aws-idle-resource-detector`

**Input**: AWS account `123456789012`, all regions, scanned March 15 2025

---

### 🔍 Idle Resource Report — 23 Resources Found

**Total Monthly Waste**: **$3,847/mo** ($46,164 annualized)

---

#### EC2 — 8 idle instances

| Instance ID | Type | Region | CPU 30d avg | Monthly Cost | Status |
|---|---|---|---|---|---|
| i-0a3b2c4d5e | m5.2xlarge | us-east-1 | 1.2% | $277 | ⛔ Zombie — terminate |
| i-0f1e2d3c4b | t3.large | us-east-1 | 2.8% | $61 | ⛔ Zombie — terminate |
| i-0123456789 | c5.xlarge | us-west-2 | 3.1% | $122 | ⚠️ Stop & snapshot |
| i-0987654321 | r5.4xlarge | eu-west-1 | 0.0% | $864 | ⛔ Zombie — stopped 47 days |
| i-0abcdef123 | m5.large | us-east-1 | 4.2% | $70 | ⚠️ Stop & snapshot |
| i-0fedcba987 | t3.medium | ap-southeast-1 | 1.9% | $30 | ⛔ Zombie — terminate |
| i-0111222333 | c5.2xlarge | us-east-1 | 2.0% | $244 | ⚠️ Stop & snapshot |
| i-0444555666 | m5.xlarge | us-east-1 | 3.3% | $138 | ⚠️ Stop & snapshot |

**EC2 waste subtotal**: $1,806/mo

#### EBS Volumes — 9 unattached volumes

| Volume ID | Size | Region | Days detached | Monthly Cost |
|---|---|---|---|---|
| vol-0a1b2c3d4e | 500 GB gp2 | us-east-1 | 63 days | $50 |
| vol-0f1e2d3c4b | 1,000 GB gp2 | us-east-1 | 91 days | $100 |
| vol-0123456789 | 200 GB io1 | us-west-2 | 14 days | $25 |
| vol-0abcdef123 | 2,000 GB gp3 | eu-west-1 | 38 days | $160 |
| *(5 more)* | | | | $215 |

**EBS waste subtotal**: $550/mo | **Upgrade tip**: 4 of these are gp2 — migrate to gp3 for 20% savings if kept

#### RDS — 3 idle databases

| DB Identifier | Instance Class | CPU 30d avg | Monthly Cost | Recommendation |
|---|---|---|---|---|
| analytics-db-dev | db.r5.2xlarge | 0.3% | $912 | ⛔ Delete (dev DB unused 30+ days) |
| reporting-mysql-old | db.m5.large | 0.0% | $138 | ⛔ Delete (replaced by Redshift) |
| legacy-postgres | db.t3.medium | 1.1% | $55 | ⚠️ Stop for 7 days, then delete |

**RDS waste subtotal**: $1,105/mo

#### Elastic IPs — 3 unassociated

Cost: $10.80/mo — release all 3 immediately.

---

#### 🛠️ Remediation Script

```bash
# Terminate confirmed zombie instances
aws ec2 terminate-instances --instance-ids i-0a3b2c4d5e i-0f1e2d3c4b i-0987654321 i-0fedcba987

# Delete unattached EBS volumes (snapshot first)
aws ec2 create-snapshot --volume-id vol-0a1b2c3d4e --description "pre-delete-backup-$(date +%Y%m%d)"
aws ec2 delete-volume --volume-id vol-0a1b2c3d4e

# Release unassociated Elastic IPs
aws ec2 release-address --allocation-id eipalloc-0a1b2c3d4e
```

> ⚠️ Validate with resource owners before terminating. Set `DeleteAfter` tag 7 days out for confirmation workflow.
