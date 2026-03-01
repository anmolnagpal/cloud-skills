# Azure Cost Pack — Example Outputs

Real-world sample outputs from two skills in this pack.

---

## 1. `azure-spend-analyzer`

**Input**: Azure Cost Management export for subscription `sub-prod-001`, March 2025

---

### 📊 Azure Spend Analysis — March 2025

**Total Spend**: £32,184 | **vs Feb**: ▲ 9.1% (+£2,680)  
**Subscription**: `prod-workloads` | **EA Enrollment**: `EA-99887766`

---

#### Top 10 Cost Drivers

| Rank | Service | Cost | MoM Change | % of Total |
|------|---------|------|-----------|------------|
| 1 | Virtual Machines | £11,840 | ▲ 6% | 36.8% |
| 2 | Azure SQL Database | £5,210 | ▲ 31% | 16.2% |
| 3 | Azure Kubernetes Service | £3,480 | ▲ 14% | 10.8% |
| 4 | Azure Blob Storage | £2,640 | ▲ 4% | 8.2% |
| 5 | Azure App Service | £1,920 | ▼ 2% | 6.0% |
| 6 | Azure Synapse Analytics | £1,740 | ▲ 88% | 5.4% |
| 7 | Azure CDN | £1,210 | ▲ 7% | 3.8% |
| 8 | Bandwidth (Egress) | £1,080 | ▲ 19% | 3.4% |
| 9 | Azure Monitor | £880 | ▲ 33% | 2.7% |
| 10 | Azure Key Vault | £240 | — | 0.7% |

---

#### 🚨 Anomalies Detected (2)

**1. Azure Synapse — 88% spike (+£820)**
- New dedicated SQL pool `syn-analytics-dev` provisioned March 7, DW1000c tier
- Running 24/7 with zero queries in last 5 days
- **Fix**: Pause pool when not in use (saves ~£700/mo) or switch to serverless SQL for ad-hoc queries

**2. Azure SQL Database — 31% spike (+£1,240)**
- `sql-reporting-eastus` migrated from GP_Gen5_4 to BC_Gen5_8 (Business Critical) on March 2
- Query performance unchanged, no SLA requirement for BC tier
- **Fix**: Revert to GP_Gen5_4 or evaluate Hyperscale — saves ~£960/mo

---

#### 💡 Top Savings Opportunities

| Opportunity | Savings/mo | Effort |
|---|---|---|
| 3-year Reserved Instances for 8 stable VMs | £2,840 | Low |
| Azure Hybrid Benefit on 14 Windows VMs (not enrolled) | £1,680 | Low |
| Pause Synapse dedicated pool outside business hours | £700 | Low |
| Revert `sql-reporting-eastus` to GP tier | £960 | Low |
| Dev/Test subscription for 6 non-prod VMs | £1,120 | Medium |
| **Total identified savings** | **£7,300/mo** | |

---

#### Hybrid Benefit Analysis

| Resource | Count | Currently Enrolled | Potential Savings |
|---|---|---|---|
| Windows Server VMs | 14 | 0 | £1,680/mo |
| SQL Server VMs | 3 | 1 | £420/mo |
| Azure SQL Databases | 2 | 2 | — |

> 💡 Enrolling all 14 Windows VMs in Azure Hybrid Benefit saves **£1,680/mo (£20,160/yr)** — requires active Software Assurance.

---

## 2. `azure-reservations-hybrid-advisor`

**Input**: 90-day VM usage data, current reservation portfolio, March 2025

---

### 💰 Reservations + Hybrid Benefit Advisory

**Current on-demand spend**: £11,840/mo  
**Estimated optimized spend**: £4,960/mo  
**Total potential savings**: **£6,880/mo (58%)**

---

#### Reservation Recommendations

| VM Size | Region | Qty | Scope | Term | Upfront/mo | Savings vs On-demand |
|---|---|---|---|---|---|---|
| Standard_D4s_v5 | UK South | 6 | Shared | 3-yr | £312 total | 66% |
| Standard_E8s_v5 | UK South | 2 | Single sub | 3-yr | £284 total | 63% |
| Standard_F8s_v2 | West Europe | 4 | Shared | 1-yr | £198 total | 37% |
| Standard_D2s_v5 | UK South | 8 | Shared | 3-yr | £96 total | 66% |

**Confidence**: High (>95% utilization on all recommended VMs over 90 days)

---

#### Stacking Strategy (Maximum Savings)

For `Standard_D4s_v5` in UK South — **before and after**:

| Layer | Action | Savings |
|---|---|---|
| Baseline (on-demand) | £0.192/hr × 6 VMs | — |
| + Reserved Instance (3-yr) | 66% off on-demand | -£0.127/hr |
| + Azure Hybrid Benefit | Remove Windows license cost | -£0.032/hr |
| **Final effective rate** | **£0.033/hr per VM** | **83% total savings** |

> Stack order matters: Apply Hybrid Benefit *first* in the portal, then purchase reservations — otherwise reservation discount applies to the full price.

---

#### Flexibility Recommendations

- Use **instance size flexibility** on all D-series and E-series reservations (default: on)
- Split 3-yr commitments for `Standard_F8s_v2` into 1-yr — lower confidence score (88%) suggests possible right-sizing within 12 months
- Set up **budget alert at 80% of reservation utilization** to catch scope changes
