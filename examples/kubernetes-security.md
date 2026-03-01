# Kubernetes Security Pack — Example Outputs

Real-world sample outputs from two skills in this pack.

---

## 1. `k8s-rbac-auditor`

**Input**: `kubectl get clusterrolebindings,rolebindings -A -o json` for cluster `prod-eks-us-east-1`

---

### 🔐 Kubernetes RBAC Audit Report

**Cluster**: prod-eks-us-east-1 | **Scan date**: 2025-03-15  
**Risk Score**: 🔴 **84/100 (Critical)**

---

#### Summary

| Severity | Findings |
|---|---|
| 🔴 Critical | 4 |
| 🟠 High | 7 |
| 🟡 Medium | 11 |

---

#### 🔴 Critical Findings

**C-1: ServiceAccount `ci-deploy-sa` has `cluster-admin`**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ci-deploy-sa-admin
subjects:
- kind: ServiceAccount
  name: ci-deploy-sa
  namespace: ci-cd
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

- **MITRE ATT&CK Containers**: T1610 — Deploy Container, T1613 — Container and Resource Discovery
- **Blast radius**: Full cluster compromise. Can create/delete any resource in any namespace, read all secrets, exec into any pod.
- **Why it's dangerous**: CI/CD compromise (supply chain attack, leaked token) = complete cluster takeover
- **Fix**: Replace with a custom role scoped to only the namespaces and verbs needed:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deployer
  namespace: payments
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update", "patch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "update"]
```

---

**C-2: Wildcard verb `*` on `pods` — `monitoring-role`**

```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["*"]  # includes pods/exec, pods/attach
```

- `pods/exec` = shell access to any pod in the namespace = container escape opportunity
- **MITRE ATT&CK**: T1609 — Container Administration Command
- **Fix**: Replace `*` with `["get", "list", "watch"]` — monitoring doesn't need exec

---

**C-3: `default` ServiceAccount has elevated permissions in `data-pipeline` namespace**

- `default` SA auto-mounted to all pods unless `automountServiceAccountToken: false`
- Any compromised pod in `data-pipeline` inherits `get/list/watch` on Secrets
- **Fix**:

```yaml
# Disable auto-mount on the default SA
kubectl patch serviceaccount default -n data-pipeline \
  -p '{"automountServiceAccountToken": false}'

# Then explicitly mount only where needed with dedicated SAs
```

---

**C-4: `system:anonymous` can access API server metrics**

- `ClusterRoleBinding` grants `system:anonymous` access to `/metrics` endpoint
- Leaks pod names, namespaces, resource counts — reconnaissance goldmine
- **MITRE ATT&CK**: T1613 — Container and Resource Discovery
- **Fix**: Delete the binding, restrict `/metrics` to Prometheus SA only

---

#### 🟠 High Findings (summary)

| ID | Subject | Issue | Namespace | Fix |
|---|---|---|---|---|
| H-1 | `developer-group` | Can `exec` into prod pods | `payments`, `auth-service` | Remove pods/exec, use `kubectl debug` workflow |
| H-2 | `logging-sa` | `secrets` get/list in all namespaces | cluster-wide | Scope to `kube-system` only |
| H-3 | 3 unused RoleBindings | Bound to deleted ServiceAccounts | various | Delete orphaned bindings |
| H-4 | `data-team` group | `configmaps` create in prod | `data-pipeline` | Move to separate non-prod cluster |

---

## 2. `k8s-pod-security-auditor`

**Input**: `kubectl get pods -A -o json` + pod specs from cluster `prod-eks-us-east-1`

---

### 🛡️ Pod Security Audit Report

**Pods scanned**: 284 across 14 namespaces  
**Risk Score**: 🟠 **76/100 (High)**

---

#### Pod Security Standards Compliance

| Namespace | Current PSA Mode | CIS Level | Critical Violations |
|---|---|---|---|
| `payments` | `warn` (baseline) | Failed | 2 |
| `data-pipeline` | None (disabled) | Failed | 5 |
| `machine-learning` | None (disabled) | Failed | 3 |
| `monitoring` | `warn` (baseline) | Partial | 1 |
| `api-gateway` | `enforce` (baseline) | Pass | 0 |
| `kube-system` | — (system) | Pass | 0 |

---

#### 🔴 Critical: Privileged Containers

**`data-exporter` in `data-pipeline` namespace**:

```yaml
securityContext:
  privileged: true   # Full host access — equivalent to root on the node
  runAsUser: 0       # Running as root
  allowPrivilegeEscalation: true
```

- **MITRE ATT&CK**: T1611 — Escape to Host
- **Impact**: Container escape to underlying EC2 node, access to all other pods' filesystems, AWS IMDS credentials
- **Fix**:

```yaml
securityContext:
  privileged: false
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
```

---

#### 🔴 Critical: `hostPID: true` on `node-agent` in `monitoring`

```yaml
spec:
  hostPID: true   # Can see all processes on the host
  hostNetwork: true  # Bypasses Kubernetes network policy
```

- Can read memory of all processes, including secrets in env vars of other containers
- **Fix**: If this is a legitimate DaemonSet (e.g., Datadog, Falco), ensure it runs on dedicated nodes with taints. Otherwise remove `hostPID`.

---

#### Remediation: Enforce Pod Security Admission

```yaml
# Label namespaces to enforce restricted policy
kubectl label namespace payments \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.29

# Start with warn mode to discover violations without breaking deployments
kubectl label namespace data-pipeline \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/warn-version=v1.29
```

---

#### Image Security Issues

| Workload | Namespace | Image | Issue |
|---|---|---|---|
| `api-server` | `api-gateway` | `nginx:latest` | Mutable tag — no digest pinning |
| `worker` | `data-pipeline` | `python:3.9` | EOL base image (Python 3.9 EOL Oct 2025) |
| `ml-trainer` | `machine-learning` | `tensorflow:2.12.0` | Known CVEs: CVE-2024-3660 (CVSS 8.1) |
| `legacy-app` | `payments` | `ubuntu:18.04` | EOL OS (bionic) |

**Fix all 4**: Pin images to SHA256 digest, update to current base images:
```yaml
# Before
image: nginx:latest

# After
image: nginx:1.27.0@sha256:67682bda769fae1ccf5183192b8daf37b64cae99c6c3302650f6f8bf5f0f95df
```

---

#### Enforcement Roadmap

| Week | Action | Risk |
|---|---|---|
| Week 1 | Fix all `privileged: true` containers | Medium |
| Week 1 | Enable `warn` PSA on all namespaces | Zero |
| Week 2 | Remove root users, add `runAsNonRoot: true` | Low |
| Week 3 | Enable `enforce` on non-critical namespaces | Low |
| Week 4 | Fix remaining violations, enforce on `payments` | Medium |
| Week 5 | Deploy OPA/Kyverno admission controller | Medium |
