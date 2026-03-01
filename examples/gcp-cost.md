# GCP Cost Pack — Example Outputs

Real-world sample outputs from two skills in this pack.

---

## 1. `gcp-bigquery-optimizer`

**Input**: BigQuery billing export for project `my-analytics-prod`, last 30 days

---

### 📊 BigQuery Cost Optimization Report

**Project**: my-analytics-prod | **Period**: Feb 2025  
**Total BigQuery cost**: $14,840 | **vs Jan**: ▲ 23% (+$2,780)

---

#### Cost Breakdown

| Component | Cost | % of Total |
|---|---|---|
| On-demand query processing | $11,220 | 75.6% |
| Active storage | $2,140 | 14.4% |
| Long-term storage | $380 | 2.6% |
| Streaming inserts | $980 | 6.6% |
| BigQuery BI Engine | $120 | 0.8% |

---

#### 🚨 Top Query Offenders

| User / Service Account | Bytes processed (30d) | Cost | Top query |
|---|---|---|---|
| `analyst-team@` (8 users) | 42.3 TB | $5,180 | `SELECT * FROM events` full table scans |
| `reporting-sa@` | 18.7 TB | $2,290 | Daily report job — no partition filter |
| `dbt-prod-sa@` | 14.2 TB | $1,740 | 6 models scanning full `raw.events` table |
| `jupyter-notebook-sa@` | 9.8 TB | $1,200 | Ad-hoc exploration with no LIMIT |

---

#### 🔥 Critical: Full Table Scans on `raw.events`

- **Table size**: 8.2 TB | **Partitioned**: ❌ No | **Clustered**: ❌ No
- This single table accounts for **$6,140/mo** in query costs
- `SELECT * FROM raw.events WHERE date = '2025-02-01'` scans **8.2 TB** — there's no partition to prune

**Fix — Add date partitioning:**
```sql
-- Create partitioned replacement
CREATE TABLE `my-analytics-prod.raw.events_partitioned`
PARTITION BY DATE(event_timestamp)
CLUSTER BY user_id, event_type
AS SELECT * FROM `my-analytics-prod.raw.events`;

-- After validation, swap tables
-- Same query now scans ~27 GB instead of 8.2 TB
-- Savings: ~$5,500/mo (98% reduction on this query pattern)
```

---

#### Partition Pruning Audit

| Table | Size | Partitioned | Queries w/o partition filter (30d) | Wasted cost |
|---|---|---|---|---|
| `raw.events` | 8.2 TB | ❌ | 847 queries | $5,500 |
| `analytics.sessions` | 1.4 TB | ✅ | 214 queries (no filter) | $890 |
| `ml.features` | 2.1 TB | ✅ | 12 queries (no filter) | $110 |

---

#### Slot Reservation Analysis

- **Current**: On-demand (flat $5/TB)
- **Monthly bytes processed**: 85 TB
- **On-demand cost**: $10,625

| Option | Monthly Cost | Break-even | Recommendation |
|---|---|---|---|
| On-demand (current) | $10,625 | — | Baseline |
| 100 slots, 1-yr commitment | $2,000 | 0.2 months | ✅ **Recommended** |
| 200 slots, 1-yr commitment | $4,000 | 0.4 months | If growth >2× expected |
| 500 slots, flex (hourly) | Varies | — | For burst analytics jobs |

> **Recommendation**: Purchase 100 Standard Edition slots ($2,000/mo) — saves **$8,625/mo** vs current on-demand.

---

#### Streaming Inserts → Storage Write API Migration

- Current streaming insert cost: **$980/mo** ($0.01/200MB)
- Equivalent Storage Write API cost: **$98/mo** ($0.025/GB committed, or free default stream)
- **Savings**: $882/mo with zero application logic change

```python
# Before (streaming inserts - $980/mo)
errors = client.insert_rows_json(table_ref, rows)

# After (Storage Write API - ~$98/mo)
from google.cloud.bigquery_storage_v1 import BigQueryWriteClient
write_client = BigQueryWriteClient()
# Use default stream for at-least-once, committed stream for exactly-once
```

---

## 2. `gcp-cud-advisor`

**Input**: GCP billing export, 90-day Compute Engine usage for project `my-analytics-prod`

---

### 💰 Committed Use Discount (CUD) Advisory

**Current Compute spend**: $8,420/mo on-demand  
**Recommended CUD portfolio**: saves **$4,180/mo (50%)**

---

#### Stable Workload Analysis (90-day baseline)

| Machine Family | Region | Stable vCPUs | Stable RAM (GB) | Confidence |
|---|---|---|---|---|
| N2 | us-central1 | 64 | 256 | 97% |
| N2D | us-east1 | 32 | 128 | 94% |
| C2 | us-central1 | 16 | 64 | 91% |
| E2 | us-central1 | 8 | 32 | 88% |

---

#### CUD Recommendations

**Option A: Resource-based CUDs (1-year)** ← Recommended for predictable machine types

| Commitment | vCPUs | RAM | Monthly cost | Savings vs on-demand |
|---|---|---|---|---|
| N2 us-central1 1-yr | 64 | 256 GB | $1,840 | 37% |
| N2D us-east1 1-yr | 32 | 128 GB | $820 | 37% |
| C2 us-central1 1-yr | 16 | 64 GB | $640 | 40% |
| **Total CUD spend** | | | **$3,300/mo** | |

**On-demand equivalent**: $5,240/mo → **Savings: $1,940/mo**

---

**Option B: Spend-based CUDs (1-year)** ← Better for mixed/unpredictable machine types

| Service | Monthly commit | Discount | Net spend |
|---|---|---|---|
| Compute Engine (general) | $5,000 | 20% | $4,000 |

**Spend-based savings**: $1,000/mo

---

#### Recommendation: Stack CUDs + Preemptible/Spot VMs

```
Stable baseline (64 vCPU N2): Resource-based CUD       → 37% off
Variable batch jobs (32 vCPU):  Spot VMs                → 60-91% off
Unpredictable burst:            On-demand               → no discount

Blended effective discount: ~52% on total compute
Total monthly compute: $8,420 → $4,040
```

> ⚠️ **Risk note**: Resource-based CUDs are inflexible — you pay whether you use them or not. At 91–97% confidence scores, this is low risk. Do NOT commit more than 80% of your stable baseline.
