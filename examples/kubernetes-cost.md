# Kubernetes Cost Pack — Example Outputs

Real-world sample outputs from two skills in this pack.

---

## 1. `k8s-cluster-cost-analyzer`

**Input**: Kubecost export + kubectl cluster-info for EKS cluster `prod-eks-us-east-1`, March 2025

---

### 📊 Kubernetes Cluster Cost Analysis

**Cluster**: prod-eks-us-east-1 | **Cloud**: AWS EKS | **Period**: March 2025  
**Total cluster cost**: **$28,420/mo**  
**Estimated idle/waste**: **$9,340/mo (32.9%)**

---

#### Cost Breakdown by Category

| Category | Monthly Cost | % of Total |
|---|---|---|
| EC2 Node Instances | $18,840 | 66.3% |
| EBS Persistent Volumes | $3,210 | 11.3% |
| Data Transfer | $1,880 | 6.6% |
| ELB Load Balancers | $1,440 | 5.1% |
| ECR Container Registry | $620 | 2.2% |
| EKS Control Plane | $210 | 0.7% |
| NAT Gateway | $1,980 | 7.0% |
| Misc (Route 53, ACM) | $240 | 0.8% |

---

#### Cost by Namespace (Top 10)

| Namespace | Monthly Cost | CPU Efficiency | Memory Efficiency | Idle % |
|---|---|---|---|---|
| `payments` | $6,840 | 71% | 68% | 29% |
| `data-pipeline` | $5,120 | 38% | 44% | 61% |
| `api-gateway` | $4,280 | 82% | 79% | 19% |
| `machine-learning` | $3,940 | 12% | 31% | 74% |
| `frontend` | $2,180 | 55% | 62% | 41% |
| `monitoring` | $1,840 | 63% | 71% | 32% |
| `auth-service` | $1,420 | 78% | 74% | 23% |
| `notifications` | $980 | 29% | 33% | 67% |
| `staging` | $860 | 8% | 12% | 91% |
| `kube-system` | $620 | 84% | 88% | 14% |

---

#### 🚨 Biggest Waste Sources

**1. `machine-learning` namespace — 74% idle ($2,920/mo wasted)**
- 4 GPU nodes running 24/7 for batch training jobs
- Actual GPU utilization: 2–6 hours/day
- **Fix**: Use Karpenter with GPU node consolidation + spot instances for training jobs

**2. `staging` namespace — 91% idle ($782/mo wasted)**
- 18 deployments running identical replicas to prod, 24/7
- No traffic outside business hours
- **Fix**: KEDA scale-to-zero for all staging workloads + Cluster Autoscaler aggressive scale-down

**3. NAT Gateway — $1,980/mo**
- All pods routing internet traffic through NAT Gateway
- Includes ECR pulls, S3 access, public API calls
- **Fix**: VPC Endpoints for S3 and ECR saves ~$1,200/mo; cache ECR images in internal mirror

---

#### Chargeback Summary

| Team | Namespace | Monthly Cost | QoQ Change |
|---|---|---|---|
| Payments Team | `payments` | $6,840 | ▲ 4% |
| Data Engineering | `data-pipeline`, `machine-learning` | $9,060 | ▲ 31% |
| Platform Team | `api-gateway`, `auth-service` | $5,700 | ▲ 2% |
| Frontend Team | `frontend` | $2,180 | ▼ 8% |
| DevOps (shared) | `monitoring`, `kube-system` | $2,460 | — |
| Unallocated | misc pods | $1,180 | — |

---

## 2. `k8s-resource-rightsizer`

**Input**: 30-day VPA metrics + `kubectl top pods` for cluster `prod-eks-us-east-1`

---

### ⚖️ Resource Right-Sizing Report

**Pods analyzed**: 284 | **Over-provisioned**: 187 (66%) | **Under-provisioned**: 12 (4%)  
**Estimated savings**: **$4,840/mo (17% of cluster cost)**

---

#### Top 10 Over-provisioned Workloads

| Workload | Namespace | CPU Request | CPU Actual (p95) | Memory Request | Memory Actual (p95) | Wasted $/mo |
|---|---|---|---|---|---|---|
| `ml-training-worker` | machine-learning | 8 CPU | 0.3 CPU | 32 GB | 4.2 GB | $1,120 |
| `data-importer` | data-pipeline | 4 CPU | 0.8 CPU | 16 GB | 2.1 GB | $680 |
| `report-generator` | data-pipeline | 2 CPU | 0.1 CPU | 8 GB | 0.8 GB | $420 |
| `notification-sender` | notifications | 1 CPU | 0.05 CPU | 4 GB | 0.3 GB | $240 |
| `staging-api` | staging | 2 CPU | 0.03 CPU | 4 GB | 0.2 GB | $220 |
| `cache-warmer` | payments | 0.5 CPU | 0.02 CPU | 2 GB | 0.1 GB | $140 |
| *(12 more)* | various | — | — | — | — | $2,020 |

---

#### Right-sizing Recommendations: `ml-training-worker`

**Current spec:**
```yaml
resources:
  requests:
    cpu: "8"
    memory: "32Gi"
  limits:
    cpu: "8"
    memory: "32Gi"
```

**Recommended spec (with 40% headroom buffer):**
```yaml
resources:
  requests:
    cpu: "500m"
    memory: "6Gi"
  limits:
    cpu: "2"       # Burst allowance for actual training runs
    memory: "8Gi"
```

**VPA recommendation (auto-apply):**
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: ml-training-worker-vpa
  namespace: machine-learning
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ml-training-worker
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: worker
      minAllowed:
        cpu: 200m
        memory: 1Gi
      maxAllowed:
        cpu: 4
        memory: 16Gi
```

---

#### ⚠️ Under-provisioned Workloads (Needs Urgent Attention)

| Workload | Namespace | Memory Request | Memory Actual (p99) | OOMKill Risk |
|---|---|---|---|---|
| `payments-api` | payments | 512 MB | 498 MB (97% of limit) | 🔴 High |
| `auth-service` | auth-service | 256 MB | 241 MB (94% of limit) | 🟠 Medium |

> ⚠️ `payments-api` is 3MB away from OOMKill. Increase memory limit to 1 GB immediately before making any right-sizing changes elsewhere.

---

#### Projected Savings After Right-sizing

| Action | Saves/mo | Risk |
|---|---|---|
| Right-size top 18 over-provisioned workloads | $4,840 | Low (with VPA) |
| Scale staging to zero overnight (KEDA) | $782 | Low |
| Karpenter bin-packing consolidation | $1,200 | Medium |
| **Total** | **$6,822/mo** | |
