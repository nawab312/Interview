# Kubernetes — Complete Remaining Gaps
## 11 Missing Topics | Key Points + Interview Tips

> **What this file covers:** Every Kubernetes topic confirmed missing
> after a complete audit of all 675+ questions across the entire question bank.
> After this file, Kubernetes coverage is 100% complete.

---

## TOPIC INDEX

| # | Topic | Gap Type |
|---|---|---|
| **Q-K8S-01** | kube-scheduler deep dive (filter/score/framework) | ❌ Zero coverage |
| **Q-K8S-02** | Sealed Secrets (kubeseal, encrypted-in-Git) | ❌ Zero coverage |
| **Q-K8S-03** | Descheduler (strategies, when to use) | ❌ Zero coverage |
| **Q-K8S-04** | EKS aws-auth ConfigMap + Access Entries | ❌ Zero coverage |
| **Q-K8S-05** | EKS cluster upgrade process | ❌ Zero coverage |
| **Q-K8S-06** | Pod preemption (vs eviction, preemptionPolicy) | ❌ Zero coverage |
| **Q-K8S-07** | Blue/Green on Kubernetes (native, no Argo) | ❌ Zero coverage |
| **Q-K8S-08** | Network troubleshooting methodology | ❌ Zero coverage |
| **Q-K8S-09** | Helm template functions (if/range/with/else) | ⚠️ define/include covered, loops/conditionals missing |
| **Q-K8S-10** | Probe implementation types (all 4 types deep dive) | ⚠️ Types listed, no deep dive |
| **Q-K8S-11** | Distributed tracing in Kubernetes (OpenTelemetry/Jaeger) | ⚠️ Mentioned, no K8s setup |

---

## Q-K8S-01 — Kubernetes | Conceptual | Advanced

> Explain how the **kube-scheduler** works internally. What are the **Filter** and **Score** phases? What is the **Scheduling Framework** and how do plugins fit into it?
>
> Walk through what happens step-by-step when a Pod is created and needs to be scheduled onto a node.

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md`

### Key Points to Cover:

**High-level scheduling flow:**
```
Pod created (kubectl apply)
    ↓
API server stores Pod in etcd (nodeName: "" = unscheduled)
    ↓
Scheduler watches API server → sees unscheduled Pod
    ↓
Scheduler runs: FILTER → SCORE → BIND
    ↓
Scheduler writes nodeName to Pod spec in etcd
    ↓
kubelet on chosen node sees Pod assigned to it → starts containers
```

**Phase 1 — Filter (eliminate unfit nodes):**
```
Runs all filter plugins against each node.
ANY plugin returning false = node eliminated.
Result: list of "feasible nodes"

Built-in filter plugins:
  NodeUnschedulable:    skip nodes with spec.unschedulable=true (cordoned)
  NodeResourcesFit:     enough CPU/memory for pod requests?
  NodeAffinity:         node labels match requiredDuringScheduling?
  TaintToleration:      pod tolerates all node taints?
  VolumeBinding:        required PVC can be bound on this node?
  PodTopologySpread:    does placing here satisfy spread constraints?
  NodePorts:            required hostPorts available?
  InterPodAffinity:     pod affinity/anti-affinity constraints satisfied?

If 0 nodes pass all filters:
  → Pod stays Pending
  → Event: "0/5 nodes are available: 3 Insufficient memory, 2 node(s) had taint"
```

**Phase 2 — Score (rank feasible nodes):**
```
Runs all score plugins against each feasible node.
Each plugin returns score 0-100.
Weighted sum → highest score = chosen node.

Built-in score plugins:
  LeastAllocated:       prefer nodes with more available resources
                        → spread pods across nodes (default behavior)
  MostAllocated:        prefer most utilized nodes
                        → bin packing (fewer nodes, cost saving)
  NodeAffinity:         nodes matching preferred affinity get higher score
  InterPodAffinity:     nodes with preferred pod neighbors score higher
  TaintToleration:      nodes with fewer taints score higher
  ImageLocality:        nodes that already have the image score higher
                        (avoids image pull on every new node)
  NodeResourcesBalancedAllocation: prefer nodes where CPU/memory usage is balanced

Score example:
  Node A: LeastAllocated=80, ImageLocality=50 → weighted total: 72
  Node B: LeastAllocated=60, ImageLocality=90 → weighted total: 68
  → Scheduler chooses Node A
```

**Scheduling Framework (extensible plugin system):**
```
Kubernetes 1.15+: scheduling is fully plugin-based
Extension points (lifecycle hooks):

  PreFilter:     validate/transform pod before filtering
  Filter:        eliminate nodes (see above)
  PostFilter:    runs if no nodes pass Filter (enables preemption)
  PreScore:      prepare state for scoring
  Score:         rank nodes (see above)
  NormalizeScore: normalize scores 0-100
  Reserve:       reserve resources (prevent race conditions)
  Permit:        allow/wait/deny binding (e.g., gang scheduling)
  PreBind:       prepare before binding (e.g., provision volume)
  Bind:          write nodeName to pod (default: DefaultBinder)
  PostBind:      after binding (cleanup, notifications)

Custom scheduler plugins:
  → Implement one or more extension points
  → Register with --config scheduler-config.yaml
  → No need for a completely separate scheduler
  → Example: Volcano (batch/AI workloads), Koordinator (QoS-aware)

Multiple schedulers:
  → Default scheduler: kube-scheduler
  → Custom scheduler pod: runs alongside default
  → Pod selects scheduler: spec.schedulerName: "my-custom-scheduler"
  → Each scheduler operates independently
```

**Scheduler configuration:**
```yaml
# KubeSchedulerConfiguration (scheduler-config.yaml)
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  plugins:
    score:
      enabled:
      - name: NodeResourcesMostAllocated  # enable bin packing
        weight: 1
      disabled:
      - name: NodeResourcesLeastAllocated # disable spread
  pluginConfig:
  - name: NodeResourcesMostAllocated
    args:
      resources:
      - name: cpu
        weight: 1
      - name: memory
        weight: 1
```

> 💡 **Interview tip:** The most impressive scheduler knowledge: **Filter eliminates, Score ranks**. Most candidates know taints/tolerations and affinity (inputs to the scheduler) but not the internal Filter/Score phases. The **PostFilter** extension point is where **preemption** happens — if no nodes pass Filter, PostFilter runs and tries to find nodes where evicting lower-priority pods would make the pod schedulable. Also: **ImageLocality** scoring means pods are preferentially scheduled on nodes that already have the image cached — this is why newly added nodes initially get fewer pods (no cached images = lower score).

---

## Q-K8S-02 — Kubernetes | Conceptual | Intermediate

> Explain **Sealed Secrets** — what problem does it solve, how does it work, and how does it compare to the **External Secrets Operator**?
>
> Walk through the complete setup and workflow of using Sealed Secrets in a GitOps environment.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md`

### Key Points to Cover:

**The problem with Kubernetes Secrets in GitOps:**
```
GitOps principle: everything in Git
Kubernetes Secret: base64 encoded (NOT encrypted)

  kubectl create secret generic db-creds \
    --from-literal=password=MyP@ssw0rd \
    --dry-run=client -o yaml

  # Output:
  apiVersion: v1
  kind: Secret
  data:
    password: TXlQQHNzVzByZA==   # base64("MyP@ssw0rd")

  # PROBLEM: base64 is NOT encryption
  # Anyone with repo access → base64 decode → plaintext password
  # Cannot commit secrets to Git safely
```

**Sealed Secrets solution:**
```
Components:
  kubeseal CLI:          encrypts secrets on developer's machine
  Sealed Secrets controller: runs in cluster, decrypts and creates real Secrets

How it works:
  1. Developer creates SealedSecret (encrypted with cluster's PUBLIC key)
  2. SealedSecret committed to Git (safe — only cluster can decrypt)
  3. ArgoCD/Flux applies SealedSecret to cluster
  4. Controller decrypts with PRIVATE key → creates real Kubernetes Secret
  5. Pod uses the real Secret normally

Keys:
  Private key: stored in controller's Secret in kube-system (NEVER leave cluster)
  Public key:  downloadable by anyone (used for encryption only)
```

**Setup and workflow:**
```bash
# Install controller:
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# Download public key:
kubeseal --fetch-cert \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  > pub-cert.pem

# Create a regular Secret first (don't apply to cluster):
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=SuperSecret123 \
  --dry-run=client -o yaml > db-secret.yaml

# Seal it (encrypt with public key):
kubeseal --format yaml \
  --cert pub-cert.pem \
  < db-secret.yaml \
  > db-sealed-secret.yaml

# Result (safe to commit to Git):
# apiVersion: bitnami.com/v1alpha1
# kind: SealedSecret
# metadata:
#   name: db-credentials
# spec:
#   encryptedData:
#     username: AgBX7K9... (RSA encrypted)
#     password: AgCD3R2... (RSA encrypted)

# Commit to Git:
git add db-sealed-secret.yaml
git commit -m "Add sealed database credentials"
git push

# ArgoCD syncs → controller decrypts → real Secret created automatically
```

**Scoping (security control):**
```bash
# strict scope (default): sealed for specific name+namespace only
kubeseal --scope strict ...
# Cannot be used in different namespace or with different name

# namespace-wide: any name in this namespace
kubeseal --scope namespace-wide ...

# cluster-wide: any namespace, any name
kubeseal --scope cluster-wide ...
```

**Sealed Secrets vs External Secrets Operator:**
```
Sealed Secrets:
  ✅ Secrets encrypted IN Git (self-contained GitOps)
  ✅ No external dependency (no Vault/AWS Secrets Manager needed)
  ✅ Simple setup, small footprint
  ❌ Key rotation = re-seal ALL secrets (operational overhead)
  ❌ No secret rotation (static values)
  ❌ Secrets still replicated in etcd
  Best for: small teams, simple secrets, true GitOps purists

External Secrets Operator (ESO):
  ✅ Single source of truth in external system (AWS Secrets Manager, Vault)
  ✅ Automatic rotation (ESO re-syncs when external value changes)
  ✅ Audit trail in external system
  ❌ Requires external dependency (AWS/GCP/Vault)
  ❌ Secrets NOT in Git (just references)
  Best for: enterprise, compliance requirements, frequent rotation

Rule of thumb:
  Simple app, small team → Sealed Secrets
  Enterprise, compliance, rotation needed → ESO + AWS Secrets Manager/Vault
```

> 💡 **Interview tip:** The critical Sealed Secrets **key rotation concern**: if the controller's private key is compromised or lost, ALL secrets need to be re-sealed with a new key. This is why always back up the controller's private key: `kubectl get secret -n kube-system sealed-secrets-key -o yaml > sealed-secrets-backup.yaml`. Store this backup securely (encrypted, separate from Git). Without this backup, losing the cluster means losing the ability to decrypt all your SealedSecrets — you'd need to re-create every secret from scratch.

---

## Q-K8S-03 — Kubernetes | Conceptual | Intermediate

> Explain the **Kubernetes Descheduler** — what problem does it solve, what are its main strategies, and when would you use it in production?
>
> What is the difference between the Scheduler and the Descheduler?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

### Key Points to Cover:

**The problem the Descheduler solves:**
```
The kube-scheduler makes decisions at POD CREATION TIME only.
It has no knowledge of the cluster state that evolved AFTER scheduling.

Real-world problems that develop over time:
  1. Node imbalance after new nodes added:
     → 10 pods on old-node-1, 0 pods on new-node-1
     → Scheduler won't move existing pods to balance them

  2. Policy violations after cluster changes:
     → Affinity rule added AFTER pods were scheduled
     → Existing pods violate the new affinity rule
     → Scheduler doesn't retroactively move them

  3. Node utilization imbalance:
     → Some nodes at 90% utilization, others at 10%
     → New pods can't schedule on overloaded nodes
     → Descheduler can evict from overloaded nodes

  4. Duplicate pods on same node:
     → Anti-affinity configured after deployment
     → Two pods of same app land on same node after node failure/recovery

Solution: Descheduler periodically EVICTS pods that violate policies
→ Evicted pods are rescheduled by the Scheduler onto better nodes
→ Descheduler never schedules — it only evicts
```

**Installation:**
```bash
helm repo add descheduler https://kubernetes-sigs.github.io/descheduler
helm install descheduler descheduler/descheduler \
  --namespace kube-system \
  --set schedule="*/5 * * * *"   # run every 5 minutes
```

**Key strategies:**

```yaml
apiVersion: "descheduler/v1alpha2"
kind: DeschedulerPolicy
profiles:
- name: default
  plugins:
    balance:
      enabled:
      - name: LowNodeUtilization        # evict from overloaded nodes
      - name: HighNodeUtilization       # consolidate (bin packing)
    deschedule:
      enabled:
      - name: RemovePodsViolatingNodeAffinity     # affinity violations
      - name: RemovePodsViolatingInterPodAntiAffinity  # anti-affinity violations
      - name: RemovePodsViolatingNodeTaints       # taint violations
      - name: RemoveDuplicates                    # duplicate pods on same node
      - name: PodLifeTime                         # evict long-running pods

  pluginConfig:
  - name: LowNodeUtilization
    args:
      thresholds:
        cpu: 20     # nodes below 20% CPU → too low (evict FROM other nodes to here)
        memory: 20  # nodes below 20% memory → too low
      targetThresholds:
        cpu: 50     # nodes above 50% CPU → too high (evict FROM these nodes)
        memory: 50  # nodes above 50% memory → too high
      # Result: pods moved from >50% nodes to <20% nodes until balanced

  - name: PodLifeTime
    args:
      maxPodLifeTimeSeconds: 86400   # evict pods older than 24 hours
      # Use case: stateless pods accumulate bugs over time → periodic refresh
```

**Strategy reference:**
```
LowNodeUtilization:
  → Evict pods from overutilized nodes → reschedule on underutilized
  → Result: balanced cluster utilization

HighNodeUtilization (bin packing):
  → Evict pods from underutilized nodes → reschedule on fuller nodes
  → Result: fewer nodes needed (cost saving for autoscaling)
  → Use with Cluster Autoscaler to consolidate and scale down

RemoveDuplicates:
  → Ensures no two pods of same ReplicaSet/Deployment on same node
  → Fixes the "all replicas ended up on one node" problem

RemovePodsViolatingNodeAffinity:
  → Evicts pods that no longer satisfy their own nodeAffinity rules
  → Needed when node labels change after pod was scheduled

RemovePodsViolatingInterPodAntiAffinity:
  → Evicts pods that violate anti-affinity rules
  → Needed when pods ended up on same node during recovery

PodLifeTime:
  → Evicts pods older than N seconds
  → Use case: long-running pods accumulate state, memory leaks, stale connections
  → Controversial — only use for truly stateless workloads
```

**Safety considerations:**
```
Descheduler RESPECTS:
  → PodDisruptionBudgets (won't evict if PDB says no)
  → Pods with local storage (won't evict by default)
  → DaemonSet pods (never evicted)
  → Mirror pods
  → Pods with annotations: descheduler.alpha.kubernetes.io/evict: "false"

Descheduler IGNORES by default:
  → Pods with priorityClassName: system-cluster-critical
  → Pods in kube-system namespace
```

> 💡 **Interview tip:** The Descheduler is the answer to "the scheduler placed pods optimally at creation time, but the cluster drifted — how do you rebalance?" The most common production use case: **a new node group is added and existing pods don't move to it**. The scheduler only considers new pods. The Descheduler periodically evicts a subset of pods from overloaded nodes, causing them to be rescheduled by the scheduler onto the new nodes. Always pair Descheduler with PodDisruptionBudgets — without PDBs, aggressive Descheduler strategies can evict too many pods simultaneously and cause service degradation.

---

## Q-K8S-04 — Kubernetes | Conceptual | Intermediate

> Explain how **EKS IAM authentication** works. What is the **aws-auth ConfigMap**, how do you add users and roles to it, and what is the new **EKS Access Entries** method?
>
> How does the entire authentication and authorization flow work from `kubectl` command to API server response?

📁 **Reference:** `nawab312/Kubernetes` → EKS authentication sections

### Key Points to Cover:

**EKS authentication flow:**
```
Standard K8s auth: certificates or tokens
EKS auth: AWS IAM identity → K8s identity (mapping)

Step-by-step flow:
1. Developer runs: kubectl get pods
2. kubectl calls: aws sts get-caller-identity (via AWS authenticator)
3. AWS returns: signed token (presigned STS URL)
4. kubectl sends: bearer token in request to K8s API server
5. API server calls: aws-iam-authenticator webhook
6. Authenticator verifies token with AWS STS
7. AWS confirms: "This token belongs to arn:aws:iam::123:user/siddharth"
8. Authenticator maps IAM identity → K8s username/groups (via aws-auth)
9. K8s RBAC: checks if username/groups have permission for the action
10. Response returned to kubectl
```

**aws-auth ConfigMap (legacy method):**
```yaml
# Location: kube-system namespace
# Created automatically by eksctl/EKS with cluster creator's role

kubectl edit configmap aws-auth -n kube-system

apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  # Map IAM ROLES to K8s groups/usernames:
  mapRoles: |
    # Node group role (REQUIRED — worker nodes need this):
    - rolearn: arn:aws:iam::123456789012:role/NodeInstanceRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes

    # CI/CD role (GitHub Actions, Jenkins):
    - rolearn: arn:aws:iam::123456789012:role/github-actions-role
      username: github-actions
      groups:
        - system:masters   # cluster-admin (use sparingly)
        # OR use a custom group:
        - deployers        # bind with RBAC RoleBinding

    # Platform team role:
    - rolearn: arn:aws:iam::123456789012:role/PlatformTeamRole
      username: platform-team
      groups:
        - platform-admins

  # Map IAM USERS to K8s groups/usernames:
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/siddharth
      username: siddharth
      groups:
        - system:masters

  # Map AWS accounts (cross-account access):
  mapAccounts: |
    - "999999999999"   # allow all users from this account
```

**Common operations:**
```bash
# Add access with eksctl (safest method):
eksctl create iamidentitymapping \
  --cluster my-cluster \
  --region us-east-1 \
  --arn arn:aws:iam::123:role/DevTeamRole \
  --username dev-team \
  --group developers

# View current mappings:
eksctl get iamidentitymapping --cluster my-cluster

# Delete mapping:
eksctl delete iamidentitymapping \
  --cluster my-cluster \
  --arn arn:aws:iam::123:role/DevTeamRole

# RBAC: bind the K8s group to permissions
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developers-binding
subjects:
- kind: Group
  name: developers          # matches group in aws-auth
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view                # built-in: read-only cluster access
  apiGroup: rbac.authorization.k8s.io
EOF
```

**EKS Access Entries (new method, EKS 1.29+):**
```bash
# Problem with aws-auth: editing a ConfigMap is error-prone
# Typo in YAML → lose all cluster access (including your own)
# Access Entries: managed via EKS API (safer, auditable)

# Create access entry for an IAM role:
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123:role/DevTeamRole \
  --type STANDARD \
  --username dev-user

# Associate with access policy:
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123:role/DevTeamRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=production

# Built-in EKS access policies:
# AmazonEKSClusterAdminPolicy   → cluster-admin
# AmazonEKSAdminPolicy          → admin (not cluster-admin)
# AmazonEKSEditPolicy            → edit (create/update/delete most resources)
# AmazonEKSViewPolicy            → view (read-only)

# List access entries:
aws eks list-access-entries --cluster-name my-cluster

# Migration: can use BOTH aws-auth AND access entries during transition
# Set authentication mode:
aws eks update-cluster-config \
  --name my-cluster \
  --access-config authenticationMode=API_AND_CONFIG_MAP
  # API_AND_CONFIG_MAP: both methods work
  # API: only access entries (migrate away from aws-auth)
  # CONFIG_MAP: only aws-auth (legacy)
```

> 💡 **Interview tip:** The **aws-auth ConfigMap is notoriously dangerous to edit manually** — a single YAML indentation mistake can lock everyone out of the cluster (including you). The recovery: you need direct AWS console access to the cluster creator's IAM role, or you need to access the API server directly. Always use `eksctl create iamidentitymapping` (which validates the YAML before applying) rather than `kubectl edit configmap aws-auth`. The new **Access Entries** method via the EKS API is much safer — API calls are validated server-side, and you can't accidentally create malformed YAML. For new clusters in 2024+, always use Access Entries.

---

## Q-K8S-05 — Kubernetes | Scenario-Based | Advanced

> Walk through the **EKS cluster upgrade process** — what is the correct order, what are the risks at each step, and how do you minimize downtime?
>
> What is the **Kubernetes version skew policy** and why does it matter during upgrades?

📁 **Reference:** `nawab312/Kubernetes` → `09_CLUSTER_OPERATIONS.md`

### Key Points to Cover:

**Version skew policy (critical to understand first):**
```
Kubernetes version skew rules:
  kube-apiserver:       must be upgraded FIRST
  kubelet:              can be 2 minor versions BEHIND apiserver
  kubectl:              can be 1 minor version behind or ahead of apiserver
  kube-controller-manager, kube-scheduler: must be same or lower than apiserver

EKS example:
  Current: 1.28
  Target:  1.30

  Step 1: Control plane to 1.29 (apiserver, controller, scheduler)
  Step 2: Node groups to 1.29 (kubelet) ← within skew policy (1 version behind)
  Step 3: Control plane to 1.30
  Step 4: Node groups to 1.30

  NEVER skip versions: 1.28 → 1.30 directly = NOT supported
  Must do: 1.28 → 1.29 → 1.30 (one minor version at a time)
```

**EKS upgrade process (step by step):**
```bash
# BEFORE UPGRADING — preparation:

# 1. Check add-ons compatibility:
aws eks describe-addon-versions --kubernetes-version 1.29
# Verify: CoreDNS, kube-proxy, VPC CNI, EBS CSI driver compatible with 1.29

# 2. Check deprecated APIs:
# Use kubent (kube-no-trouble) to find deprecated API usage:
kubent
# Output: objects in cluster using APIs removed in 1.29

# 3. Review EKS release notes:
# https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html

# 4. Test in non-prod cluster first

# STEP 1: Upgrade control plane (managed by AWS):
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.29

# Monitor upgrade:
aws eks describe-cluster --name my-cluster \
  --query 'cluster.status'
# UPDATING → ACTIVE (takes ~10-15 minutes)
# During this time: API server has brief interruptions (~30s per AZ)
# Running workloads: unaffected (kubelet manages pods independently)

# STEP 2: Upgrade EKS add-ons:
aws eks update-addon --cluster-name my-cluster --addon-name coredns \
  --addon-version v1.10.1-eksbuild.1

aws eks update-addon --cluster-name my-cluster --addon-name kube-proxy \
  --addon-version v1.29.0-eksbuild.1

aws eks update-addon --cluster-name my-cluster --addon-name vpc-cni \
  --addon-version v1.16.0-eksbuild.1

# STEP 3: Upgrade node groups (managed node groups):
# Option A: In-place rolling update (recommended for most cases):
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name standard-nodes \
  --kubernetes-version 1.29

# Process: 
# → Creates new nodes with 1.29
# → Cordons and drains old nodes (respects PDBs)
# → Terminates old nodes
# → maxUnavailable: 1 (default) → zero-downtime if replicas >= 2

# Option B: Blue/Green node group (safer for production):
# 1. Create new node group with 1.29:
eksctl create nodegroup \
  --cluster my-cluster \
  --name standard-nodes-v129 \
  --node-type m5.xlarge \
  --nodes 5 \
  --kubernetes-version 1.29

# 2. Cordon old node group (no new pods):
kubectl cordon -l eks.amazonaws.com/nodegroup=standard-nodes

# 3. Drain old node group (move pods to new nodes):
kubectl drain -l eks.amazonaws.com/nodegroup=standard-nodes \
  --ignore-daemonsets --delete-emptydir-data

# 4. Verify workloads on new nodes, then delete old group:
eksctl delete nodegroup --cluster my-cluster --name standard-nodes

# STEP 4: Upgrade self-managed node groups (if any):
# Must upgrade kubelet manually:
# 1. Bake new AMI with updated kubelet
# 2. Update Launch Template
# 3. Rolling replace instances

# POST-UPGRADE verification:
kubectl get nodes        # all nodes showing 1.29
kubectl get pods -A      # all pods running
kubectl get componentstatuses  # control plane healthy
```

**What can go wrong:**
```
1. Deprecated API usage:
   → Pod uses batch/v1beta1 (removed in 1.25) → fails after upgrade
   → Prevention: kubent scan BEFORE upgrade

2. PodDisruptionBudget blocks drain:
   → PDB too strict → node drain hangs
   → Fix: temporarily relax PDB or force drain (--disable-eviction)

3. Incompatible admission webhooks:
   → Custom webhook built for old K8s → rejects valid objects in new version
   → Prevention: test webhook in new version before cluster upgrade

4. EKS add-on version incompatibility:
   → Forgot to upgrade CoreDNS → DNS failures after upgrade
   → Prevention: upgrade add-ons immediately after control plane upgrade

5. Workload using removed beta APIs:
   → Deployment uses extensions/v1beta1 (removed) → can't apply after upgrade
   → Prevention: kubent, CI/CD checks for deprecated APIs
```

> 💡 **Interview tip:** The **most common EKS upgrade mistake**: upgrading control plane from 1.28 to 1.29 but leaving node groups on 1.27 (2 versions behind, which is the skew limit). If you then try to upgrade to 1.30, your nodes at 1.27 would be 3 versions behind — violating the skew policy. The safest rule: **always upgrade node groups immediately after the control plane upgrade** within the same maintenance window. Also: **deprecated API checking with `kubent` is non-negotiable** — discovering that your production Helm charts use `policy/v1beta1 PodSecurityPolicy` (removed in 1.25) after upgrading is a production incident.

---

## Q-K8S-06 — Kubernetes | Conceptual | Advanced

> Explain **pod preemption** in Kubernetes — what is it, how does it work, and how is it different from **eviction**?
>
> What is `preemptionPolicy: Never` and when would you use it? Walk through the exact sequence of events when a high-priority pod triggers preemption.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

### Key Points to Cover:

**Preemption vs Eviction:**
```
Eviction:
  → Triggered by: kubelet (node resource pressure — memory, disk)
  → Involuntary: kubelet decides to kill pods to protect the node
  → Target: lowest QoS class first (BestEffort → Burstable → Guaranteed)
  → No new pod waiting — just freeing resources from a stressed node

Preemption:
  → Triggered by: scheduler (new high-priority pod can't fit)
  → Voluntary (planned): scheduler decides to remove lower-priority pods
  → Target: pods with lowest priority that block the pending pod
  → REASON: make room for a specific high-priority pending pod
  → Respects: PodDisruptionBudgets (won't preempt if PDB violated)
```

**Preemption sequence (step by step):**
```
Scenario: pending pod P (priority 1000) needs 4 CPU
          all nodes full with lower-priority pods

Step 1 — Filter phase finds 0 feasible nodes
         → No node has enough free resources

Step 2 — PostFilter runs (preemption logic):
         → Finds "victim candidates": pods that could be evicted to make room
         → Considers: pods with priority < P's priority
         → Evaluates: which node needs FEWEST victims evicted

Step 3 — Scheduler nominates a node:
         → Sets pod.status.nominatedNodeName = "node-2"
         → This is just a hint — not a guarantee

Step 4 — Scheduler sends graceful termination to victims:
         → kubectl delete pod victim-1 victim-2 (graceful)
         → Victims get SIGTERM → terminationGracePeriodSeconds to finish

Step 5 — Resources freed → P gets scheduled on node-2

Step 6 — If node-2 gets a different pod before P (race condition):
         → P re-runs the whole scheduling process
         → nominatedNodeName doesn't guarantee reservation

Example:
  PriorityClass: critical (value: 1000000)
  PriorityClass: batch (value: 100)

  → batch pod running on node-1 using 4 CPU
  → critical pod pending, needs 4 CPU, node-1 only option
  → scheduler preempts batch pod → critical pod scheduled
```

**PriorityClass configuration:**
```yaml
# Define priority classes:
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-critical
value: 1000000
globalDefault: false
description: "Production critical services — never preempted"
preemptionPolicy: PreemptLowerPriority   # DEFAULT: can preempt others

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-jobs
value: 100
preemptionPolicy: Never  # CANNOT preempt other pods even if higher priority than target
description: "Batch jobs — can be preempted, cannot preempt others"

---
# Use in pod:
spec:
  priorityClassName: production-critical
  containers:
  - name: api-server
    ...

# Built-in system classes:
# system-cluster-critical: 2000000000 (CoreDNS, kube-proxy)
# system-node-critical:    2000001000 (static pods on nodes)
```

**`preemptionPolicy: Never` use cases:**
```
preemptionPolicy: Never means:
  "This pod has high priority (gets scheduled over lower-priority in queue)
   BUT cannot evict any running pod to make room for itself"

Use cases:
  1. Batch jobs that should be prioritized in queue but are interruptible:
     → Job waits for resources to free up naturally (no preemption)
     → Won't disrupt running production workloads

  2. Non-production workloads that should queue fairly:
     → Higher value than default (0) → scheduled before unknown pods
     → But won't kick out production pods

  3. Machine Learning training jobs:
     → Schedule before generic workloads but never preempt serving pods
```

**Practical configuration:**
```yaml
# Production tiers:
kind: PriorityClass
metadata: { name: tier-critical }
value: 1000000
preemptionPolicy: PreemptLowerPriority  # can preempt anything below

---
kind: PriorityClass
metadata: { name: tier-high }
value: 100000
preemptionPolicy: PreemptLowerPriority

---
kind: PriorityClass
metadata: { name: tier-batch }
value: 1000
preemptionPolicy: Never   # queues behind others but won't preempt

---
kind: PriorityClass
metadata: { name: tier-dev }
value: 0   # default priority — first to be preempted
```

> 💡 **Interview tip:** The key preemption insight for interviews: **preemption is triggered by the scheduler, eviction by the kubelet — they are independent mechanisms**. A node under memory pressure uses eviction (kubelet, reactive). A pending high-priority pod uses preemption (scheduler, proactive). They share one concept: QoS and priority classes. Also: `nominatedNodeName` is NOT a reservation — it's a hint. Between the scheduler setting it and the victims being gracefully terminated, another pod could grab the freed resources. Kubernetes makes a "best effort" to schedule the high-priority pod on the nominated node, but there's no lock. This is by design to avoid deadlocks.

---

## Q-K8S-07 — Kubernetes | Conceptual | Intermediate

> Explain **Blue/Green deployment on Kubernetes** using native resources (without Argo Rollouts). How does the Service selector swap work?
>
> Compare this native approach with using **Argo Rollouts** for blue/green. What are the trade-offs?

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`

### Key Points to Cover:

**Native Blue/Green with Service selector swap:**
```
Core concept:
  → Two Deployments running simultaneously (blue and green)
  → ONE Service at a time pointing to active deployment
  → Traffic switch: change Service selector label
  → Rollback: change Service selector back (instantaneous)

Implementation:

# Blue Deployment (current production):
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-blue
  labels:
    app: payment
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment
      version: blue     # ← key label
  template:
    metadata:
      labels:
        app: payment
        version: blue
    spec:
      containers:
      - name: payment
        image: payment-service:v1.2.3

---
# Green Deployment (new version, not receiving traffic yet):
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-green
  labels:
    app: payment
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment
      version: green    # ← different label
  template:
    metadata:
      labels:
        app: payment
        version: green
    spec:
      containers:
      - name: payment
        image: payment-service:v1.3.0   # new version

---
# Service (currently pointing to blue):
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
    version: blue    # ← change this to switch traffic
  ports:
  - port: 80
    targetPort: 8080
```

**Traffic switch workflow:**
```bash
# Step 1: Deploy green (while blue serves 100% traffic):
kubectl apply -f payment-green.yaml

# Step 2: Verify green is healthy:
kubectl rollout status deployment/payment-green
kubectl get pods -l version=green

# Step 3: Test green (optional — port-forward or preview service):
kubectl port-forward svc/payment-preview 8080:80
curl http://localhost:8080/health

# Step 4: Switch traffic to green (atomic — service selector update):
kubectl patch service payment-service \
  -p '{"spec":{"selector":{"app":"payment","version":"green"}}}'

# Verify switch:
kubectl get service payment-service -o jsonpath='{.spec.selector}'
# {"app":"payment","version":"green"}

# Step 5: Monitor green (watch error rates, latency):
# If all good → delete blue after confidence period:
kubectl delete deployment payment-blue

# ROLLBACK (if issues detected):
kubectl patch service payment-service \
  -p '{"spec":{"selector":{"app":"payment","version":"blue"}}}'
# Instantaneous — blue pods still running!
```

**Native B/G vs Argo Rollouts:**
```
Native Blue/Green:
  Pros:
  ✅ No additional tooling (just kubectl + YAML)
  ✅ Simple mental model (two deployments, one service)
  ✅ Full control over timing
  ✅ Works with any K8s version

  Cons:
  ❌ 2x resource cost (both versions running simultaneously)
  ❌ Manual traffic switching (human error risk)
  ❌ No built-in metrics-based promotion
  ❌ No progressive traffic shifting (0% → 100%, no middle ground)
  ❌ Multiple services needed for testing (green test service)

Argo Rollouts Blue/Green:
  Pros:
  ✅ Automated promotion based on analysis (Prometheus metrics)
  ✅ Pre/post-promotion analysis (automatic gate)
  ✅ Built-in pause for manual inspection
  ✅ Anti-affinity support
  ✅ Webhook integrations (Slack notifications on promotion)
  ✅ Tracks rollout history, easy rollback via CLI

  Cons:
  ❌ Additional CRD + controller to manage
  ❌ More complex configuration
  ❌ Steeper learning curve

When to use native:
  → Simple apps, small teams
  → No Argo Rollouts in cluster
  → Maximum simplicity preferred

When to use Argo Rollouts:
  → Production services requiring automated gates
  → Teams want progressive traffic shifting (10% → 50% → 100%)
  → Metrics-based automatic rollback
```

**Ingress-based traffic splitting (progressive):**
```yaml
# For gradual traffic shift (native K8s + NGINX Ingress):
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"  # 20% to green
spec:
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: payment-green-service
            port:
              number: 80
# Gradually increase canary-weight: 20 → 50 → 100 → remove canary annotation
```

> 💡 **Interview tip:** The **Service selector swap** is the core primitive — it's atomic at the API server level, meaning there's no period where traffic goes to both versions simultaneously (unlike weighted routing). The risk: if you delete blue immediately after switching, you lose instant rollback. Best practice: **keep blue running for at least 15 minutes** after switching to green (enough to catch any issues in monitoring), then delete. Cost: 2x resources during this window. For services where cost matters, use Argo Rollouts canary (which uses a single deployment and splits traffic) instead of pure blue/green.

---

## Q-K8S-08 — Kubernetes | Troubleshooting | Advanced

> A pod cannot connect to another service. Walk through your **complete systematic network troubleshooting methodology** in Kubernetes — from pod-to-pod, pod-to-service, and pod-to-external.
>
> What tools do you use at each layer? How do you rule out DNS, kube-proxy, NetworkPolicy, and CNI issues?

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md`, `14_TROUBLESHOOTING.md`

### Key Points to Cover:

**Troubleshooting layers (OSI model for K8s):**
```
Layer 1 — Is the target pod/service running?
Layer 2 — DNS resolution (can we resolve the name?)
Layer 3 — Network reachability (can we reach the IP?)
Layer 4 — Port connectivity (is the port open?)
Layer 5 — Application (is the app responding correctly?)
Layer 6 — NetworkPolicy (is traffic explicitly allowed?)
```

**Debug pod setup (netshoot — the Swiss army knife):**
```bash
# Deploy a debug pod in the same namespace:
kubectl run netshoot --image=nicolaka/netshoot \
  -n production --rm -it -- bash
# netshoot has: curl, wget, dig, nslookup, nc, nmap, tcpdump, ss, ip

# OR: add ephemeral debug container to existing pod:
kubectl debug -it my-pod \
  --image=nicolaka/netshoot \
  --target=my-container
```

**Step 1 — Verify target is running:**
```bash
# Check pods:
kubectl get pods -n production -l app=backend
kubectl describe pod <backend-pod> -n production
# → Is it Running? Healthy? Endpoints populated?

# Check service endpoints:
kubectl get endpoints backend-service -n production
# → Empty endpoints = pod not ready or selector mismatch
kubectl describe service backend-service -n production
# → Verify selector labels match pod labels
```

**Step 2 — DNS resolution:**
```bash
# From debug pod in SAME namespace:
nslookup backend-service                                    # short name
nslookup backend-service.production                        # with namespace
nslookup backend-service.production.svc.cluster.local     # FQDN

# From debug pod in DIFFERENT namespace:
nslookup backend-service.production.svc.cluster.local     # must use FQDN

# Check CoreDNS is running:
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns | tail -20

# Test CoreDNS directly:
dig @10.96.0.10 backend-service.production.svc.cluster.local
# 10.96.0.10 = CoreDNS ClusterIP (check: kubectl get svc -n kube-system kube-dns)

# Common DNS failures:
# → NXDOMAIN: service doesn't exist / wrong namespace
# → Timeout: CoreDNS pod not running / node DNS resolution issues
# → Wrong IP: CoreDNS returning stale cache
```

**Step 3 — Reachability to Pod IP directly:**
```bash
# Get pod IP:
kubectl get pod backend-pod -o jsonpath='{.status.podIP}'
# e.g., 10.244.2.15

# Test direct pod-to-pod (bypasses Service and kube-proxy):
ping 10.244.2.15
curl http://10.244.2.15:8080/health

# If this fails but same-node works:
# → CNI issue (routes between nodes broken)
# → Check CNI pods: kubectl get pods -n kube-system -l k8s-app=calico-node

# If this fails completely:
# → Check pod CIDR routes: ip route show | grep 10.244
# → Check node-to-node connectivity
```

**Step 4 — Service ClusterIP connectivity:**
```bash
# Get service ClusterIP:
kubectl get service backend-service -o jsonpath='{.spec.clusterIP}'
# e.g., 10.96.45.200

# Test Service IP (goes through kube-proxy iptables/IPVS):
curl http://10.96.45.200:80/health

# If pod-to-pod works but Service IP fails:
# → kube-proxy issue
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy | tail -20

# Check iptables rules for service:
iptables -t nat -L -n | grep 10.96.45.200
# Should see: KUBE-SVC-XXXXXXXX chain for the service
```

**Step 5 — NetworkPolicy audit:**
```bash
# List all NetworkPolicies:
kubectl get networkpolicy -n production -o yaml

# Check if a default-deny exists:
kubectl get networkpolicy -n production | grep default-deny

# Test: temporarily allow all traffic to isolate:
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-temp
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress: [{}]
  egress: [{}]
EOF
# If connectivity works now → NetworkPolicy was blocking
# Delete temp policy and fix the real one
kubectl delete networkpolicy allow-all-temp -n production
```

**Step 6 — Pod-to-external connectivity:**
```bash
# Test external DNS:
nslookup google.com
dig google.com @8.8.8.8

# Test external HTTP:
curl -v https://api.external.com/health --max-time 5

# If fails — check routing:
# → Does the pod's subnet have a route to internet?
# → Is there a NAT Gateway for private subnets?
ip route show
curl https://ifconfig.me  # what IP am I hitting internet from?

# Check if NetworkPolicy blocks egress to external:
# Default-deny-all blocks ALL egress including external
# Must explicitly allow: egress to 0.0.0.0/0 port 443/80
```

**Quick troubleshooting decision tree:**
```
Pod can't reach Service?
├── DNS fails?
│   ├── Yes → CoreDNS issue or wrong service name/namespace
│   └── No
├── Pod IP reachable directly?
│   ├── No → CNI issue (routes broken between nodes)
│   └── Yes
├── Service ClusterIP reachable?
│   ├── No → kube-proxy issue (iptables/IPVS rules missing)
│   └── Yes
├── NetworkPolicy blocking?
│   ├── Yes → add ingress/egress rules
│   └── No
└── Application layer issue (connection refused, 500, TLS)
    → app not listening, wrong port, TLS mismatch
```

> 💡 **Interview tip:** **`kubectl get endpoints <service-name>`** is the single fastest diagnostic — if endpoints are empty, traffic will NEVER reach pods regardless of what else you do. Empty endpoints means either: (1) no pods with matching labels exist, (2) pods exist but readiness probe is failing. Always check endpoints first. The second most important tool: **netshoot** (`docker pull nicolaka/netshoot`) — it has every networking tool you need in one container. Keep it as your go-to debug image. The methodology: always go bottom-up (can I reach the pod IP? → can I reach the Service IP? → is DNS working?) rather than starting at the application layer.

---

## Q-K8S-09 — Kubernetes | Conceptual | Intermediate

> Explain **Helm template functions** — specifically `{{- if}}`, `{{- range}}`, `{{- with}}`, `{{- else}}`, and `toYaml | nindent`.
>
> Write practical examples of each in the context of a Kubernetes Deployment template.

📁 **Reference:** `nawab312/Kubernetes` — Helm chart development sections

### Key Points to Cover:

**Helm templating basics:**
```
Helm templates = Go text/template + Sprig functions
{{ }} = action block (evaluated)
{{- }} = trim whitespace BEFORE block
{{ -}} = trim whitespace AFTER block
{{- -}} = trim both sides
. = current context (dot) — starts as top-level Values/Chart/Release
```

**`{{- if}} / {{- else}} / {{- end}}` — conditionals:**
```yaml
# templates/deployment.yaml

# Simple boolean:
{{- if .Values.autoscaling.enabled }}
# This block only rendered if autoscaling.enabled is true
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
{{- end }}

# String comparison:
{{- if eq .Values.environment "production" }}
  replicas: 5
{{- else if eq .Values.environment "staging" }}
  replicas: 2
{{- else }}
  replicas: 1
{{- end }}

# Existence check (is value set and non-empty?):
{{- if .Values.ingress.annotations }}
  annotations:
    {{- toYaml .Values.ingress.annotations | nindent 4 }}
{{- end }}

# Values:
# ingress:
#   annotations:
#     kubernetes.io/ingress.class: nginx
#     cert-manager.io/cluster-issuer: letsencrypt

# NOT operator:
{{- if not .Values.readinessProbe.disabled }}
  readinessProbe:
    httpGet:
      path: /health
      port: 8080
{{- end }}
```

**`{{- range}}` — loops:**
```yaml
# Loop over a list:
# values.yaml:
# env:
#   - name: DATABASE_URL
#     value: postgresql://localhost/mydb
#   - name: LOG_LEVEL
#     value: info

env:
{{- range .Values.env }}
  - name: {{ .name }}
    value: {{ .value | quote }}
{{- end }}

# Loop over a map:
# values.yaml:
# labels:
#   team: platform
#   environment: production

metadata:
  labels:
{{- range $key, $value := .Values.labels }}
    {{ $key }}: {{ $value | quote }}
{{- end }}

# Loop with index:
{{- range $i, $port := .Values.ports }}
  - name: port-{{ $i }}
    containerPort: {{ $port }}
{{- end }}

# Real example — environment variables from multiple sources:
env:
{{- range .Values.secretEnvVars }}
  - name: {{ .name }}
    valueFrom:
      secretKeyRef:
        name: {{ .secretName }}
        key: {{ .secretKey }}
{{- end }}
{{- range .Values.configEnvVars }}
  - name: {{ .name }}
    valueFrom:
      configMapKeyRef:
        name: {{ .configMapName }}
        key: {{ .configMapKey }}
{{- end }}
```

**`{{- with}}` — scope change:**
```yaml
# with changes . (dot) to the specified value
# Also acts as a conditional (skips block if value is empty)

# Without with (verbose):
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
imagePullPolicy: {{ .Values.image.pullPolicy }}

# With 'with' (cleaner scope):
{{- with .Values.image }}
image: {{ .repository }}:{{ .tag }}
imagePullPolicy: {{ .pullPolicy }}
{{- end }}

# Accessing parent context inside with (use $):
{{- with .Values.serviceAccount }}
serviceAccountName: {{ .name }}
  {{- if $.Values.rbac.enabled }}  # $ = root context
automountServiceAccountToken: true
  {{- end }}
{{- end }}
```

**`toYaml | nindent` — embed complex values:**
```yaml
# toYaml: converts a Values object to YAML string
# nindent N: adds N spaces of indentation to each line
# indent N: same but doesn't add leading newline

# values.yaml:
# resources:
#   requests:
#     cpu: "100m"
#     memory: "128Mi"
#   limits:
#     cpu: "500m"
#     memory: "512Mi"

containers:
- name: app
  resources:
    {{- toYaml .Values.resources | nindent 4 }}
  # Result:
  # resources:
  #     requests:
  #       cpu: "100m"
  #       memory: "128Mi"
  #     limits:
  #       cpu: "500m"
  #       memory: "512Mi"

# Common pattern for arbitrary annotations:
metadata:
  annotations:
    {{- toYaml .Values.podAnnotations | nindent 4 }}

# Tolerations (list):
{{- if .Values.tolerations }}
tolerations:
  {{- toYaml .Values.tolerations | nindent 2 }}
{{- end }}
```

**Complete real template snippet:**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        {{- if .Values.env }}
        env:
        {{- range .Values.env }}
        - name: {{ .name }}
          value: {{ .value | quote }}
        {{- end }}
        {{- end }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

> 💡 **Interview tip:** The `{{-` whitespace trimming trips up almost everyone — without it, Helm generates YAML with extra blank lines that may or may not matter (YAML ignores blank lines in most cases, but it looks messy). The rule: use `{{-` to trim leading whitespace on lines that are pure template logic (if/else/end/range/with blocks) that shouldn't contribute blank lines to the output. The **`toYaml | nindent N`** pattern is the most used Helm idiom — it lets users paste any YAML structure into values.yaml and have it correctly indented in the template. Remember: `nindent` adds a LEADING newline before the indented block, making it safe to use after a key name like `resources:`.

---

## Q-K8S-10 — Kubernetes | Conceptual | Intermediate

> Explain all **4 probe implementation types** in Kubernetes — `httpGet`, `tcpSocket`, `exec`, and `grpc`. When would you use each? What are the key probe parameters and how do you tune them for production?

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`

### Key Points to Cover:

**4 probe implementation types:**

```yaml
# Type 1: httpGet — HTTP GET request
# Success: response status code 200-399
# Use: HTTP/HTTPS services (most common)
livenessProbe:
  httpGet:
    path: /healthz          # health check endpoint
    port: 8080
    scheme: HTTP            # HTTP or HTTPS
    httpHeaders:            # optional custom headers
    - name: X-Health-Check
      value: "true"

# Type 2: tcpSocket — TCP connection attempt
# Success: connection established (port is open)
# Does NOT send/receive data — just checks port is listening
# Use: databases, message queues, any TCP service without HTTP
livenessProbe:
  tcpSocket:
    port: 5432              # PostgreSQL — can't use httpGet here
    # OR port by name:
    port: postgresql

# Type 3: exec — run command inside container
# Success: command exits with code 0
# Non-zero exit = failure
# Use: complex custom health logic, CLI-based checks
livenessProbe:
  exec:
    command:
    - sh
    - -c
    - "pg_isready -U postgres -h localhost"  # PostgreSQL readiness
    # OR:
    - "/bin/check-health.sh"
    # OR:
    - "redis-cli ping"                        # Redis

# Type 4: gRPC — gRPC health check protocol
# Requires: app implements gRPC Health Checking Protocol
# Success: returns SERVING status
# Use: gRPC services
livenessProbe:
  grpc:
    port: 9090
    service: "my.grpc.service"  # optional gRPC service name
```

**Key probe parameters and tuning:**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30   # wait 30s before first probe
                             # MUST be > app startup time
                             # too low → app killed before ready
  periodSeconds: 10          # probe every 10s
  timeoutSeconds: 5          # probe fails if no response in 5s
  successThreshold: 1        # 1 success = healthy (liveness always 1)
  failureThreshold: 3        # 3 failures in a row = unhealthy → restart
                             # total tolerance = periodSeconds × failureThreshold
                             # = 30 seconds of failures before restart

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10    # readiness can start earlier than liveness
  periodSeconds: 5           # check more frequently (traffic impact)
  failureThreshold: 3        # 3 failures → remove from Service
  successThreshold: 2        # need 2 successes → re-add to Service
                             # prevents flapping

startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30       # allow 30 × 10s = 5 minutes for startup
  periodSeconds: 10
  # After startup succeeds → liveness/readiness take over
```

**When to use which probe type:**
```
httpGet:
  ✅ REST APIs, web services
  ✅ Most microservices
  ✅ Best for: checking app logic (not just process alive)
  Example: check DB connection, cache warm, config loaded

tcpSocket:
  ✅ Databases (PostgreSQL, MySQL, MongoDB) — no HTTP endpoint
  ✅ Message queues (RabbitMQ, Redis)
  ✅ Any service that accepts TCP but doesn't have HTTP health endpoint
  ⚠️ Only checks port is open, NOT if app is healthy/processing

exec:
  ✅ Complex health logic requiring shell commands
  ✅ Checking background jobs completed
  ✅ Checking file existence (signal file from init process)
  ⚠️ Overhead: spawns new process for EACH probe execution
  ⚠️ Avoid if probeFrequency is high on large clusters

grpc:
  ✅ gRPC services (avoids adding HTTP server just for health checks)
  ✅ Requires implementing grpc.health.v1.Health service
  ✅ Go gRPC example: grpc-health-probe compatible

Liveness vs Readiness endpoint design:
  /healthz (liveness):   "Is the process alive and not stuck?"
    → Return 200 always (unless truly broken: deadlock, panic)
    → Do NOT check dependencies here (DB down ≠ kill the pod)
    → Failing: pod restarts (dependency still down, now in CrashLoopBackOff)

  /ready (readiness):    "Can I serve traffic right now?"
    → Return 200 only if all dependencies available
    → Check: DB connection, cache, downstream services
    → Failing: pod removed from Service (stays running, waits for recovery)
```

> 💡 **Interview tip:** The **most common probe mistake**: putting database connectivity checks in liveness probes. If the database goes down, ALL pods fail their liveness probes simultaneously → all pods restart → they all try to reconnect on startup → the database gets hammered with reconnection attempts → thundering herd. The correct design: **liveness = is the app process healthy**, **readiness = can it serve traffic (including dependency checks)**. When DB goes down: readiness fails (remove pods from load balancer — no traffic), liveness passes (pods stay running, waiting for DB recovery). When DB recovers: readiness passes, pods re-added to Service. No unnecessary restarts.

---

## Q-K8S-11 — Kubernetes | Conceptual | Intermediate

> Explain **distributed tracing in Kubernetes** — how do you set up **OpenTelemetry**, instrument your applications, and use **Jaeger** or **Tempo** to visualize traces?
>
> How does tracing complement Prometheus metrics and log aggregation (the "three pillars of observability")?

📁 **Reference:** `nawab312/Kubernetes` → `08_OBSERVABILITY.md`

### Key Points to Cover:

**Three pillars of observability:**
```
Metrics (Prometheus):
  → What is happening? (counts, rates, gauges)
  → "Error rate is 5%" — but which request? which user? which path?

Logs (Loki/Elasticsearch):
  → What happened? (discrete events with context)
  → "ERROR: timeout connecting to payment-service" — but how often?
  → Correlation: timestamp, but no easy cross-service correlation

Traces (Jaeger/Tempo/Zipkin):
  → Why did it happen? (causality, timing across services)
  → Single request's journey through ALL microservices
  → "This user's checkout took 3s: 2.5s was in payment-service DB query"

Together:
  Alert fires (Prometheus) → investigate logs (Loki) → trace the slow requests (Jaeger)
```

**OpenTelemetry (OTEL) — the standard:**
```
OpenTelemetry = vendor-neutral standard for traces, metrics, logs
  → SDK: instrument your app (Go, Python, Java, Node.js, etc.)
  → Collector: receive, process, export to any backend
  → Protocol: OTLP (OpenTelemetry Protocol)

Architecture:
  App (OTEL SDK) → OTEL Collector → Jaeger/Tempo/Zipkin/Datadog

Key concepts:
  Trace:   entire journey of one request (unique trace ID)
  Span:    one unit of work within a trace (start time, end time, metadata)
  Context propagation: trace ID passed via HTTP headers between services
    → W3C Trace Context: traceparent: 00-traceID-spanID-flags
```

**Instrument a Node.js app:**
```javascript
// tracing.js — load BEFORE app code
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://otel-collector:4318/v1/traces',  // OTEL Collector in K8s
  }),
  instrumentations: [
    getNodeAutoInstrumentations(),  // auto-instruments: HTTP, Express, DB, etc.
  ],
  serviceName: 'payment-service',
});

sdk.start();  // must start before require('express')

// Start app: node -r ./tracing.js app.js
// Or set env var: NODE_OPTIONS='--require ./tracing.js'
```

**Instrument Python (FastAPI):**
```python
# pip install opentelemetry-distro opentelemetry-exporter-otlp
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

# Setup:
provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(
        endpoint="http://otel-collector:4318/v1/traces"
    ))
)
trace.set_tracer_provider(provider)

# Auto-instrument FastAPI:
FastAPIInstrumentor.instrument_app(app)

# Manual spans:
tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span("database-query") as span:
    span.set_attribute("db.statement", "SELECT * FROM orders")
    result = db.execute(query)
    span.set_attribute("db.rows_returned", len(result))
```

**Deploy OpenTelemetry Collector in Kubernetes:**
```yaml
# ConfigMap for OTEL Collector config:
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        timeout: 5s
        send_batch_size: 1000
      resource:
        attributes:
        - action: insert
          key: cluster
          value: "production-eks"    # add cluster name to all traces

    exporters:
      jaeger:
        endpoint: jaeger-collector:14250   # gRPC to Jaeger
        tls:
          insecure: true
      # OR Grafana Tempo:
      otlp:
        endpoint: tempo:4317
        tls:
          insecure: true

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch, resource]
          exporters: [jaeger]

---
# DaemonSet: one collector per node (low latency):
# OR Deployment: central collector (simpler)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:latest
        args: ["--config=/config/config.yaml"]
        ports:
        - containerPort: 4317   # OTLP gRPC
        - containerPort: 4318   # OTLP HTTP
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: otel-collector-config
```

**Jaeger deployment in Kubernetes:**
```bash
# Install Jaeger Operator:
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm install jaeger-operator jaegertracing/jaeger-operator \
  --namespace observability

# Deploy Jaeger instance (all-in-one for dev):
kubectl apply -f - <<EOF
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
  namespace: observability
spec:
  strategy: allinone   # dev: all in one pod
  # strategy: production  # prod: separate collector/query/storage
  storage:
    type: elasticsearch   # prod: use Elasticsearch for persistence
    options:
      es:
        server-urls: http://elasticsearch:9200
EOF

# Access Jaeger UI:
kubectl port-forward svc/jaeger-query 16686:16686 -n observability
# Open: http://localhost:16686
```

**Grafana Tempo + Loki correlation (modern stack):**
```yaml
# Tempo is Grafana's native tracing backend
# Key feature: correlate traces ↔ logs ↔ metrics IN GRAFANA

# Install Tempo:
helm install tempo grafana/tempo \
  --namespace monitoring \
  --set tempo.reportingEnabled=false

# In Grafana Data Sources:
# Add Tempo → configure TraceQL queries
# Link Loki logs to Tempo traces (by trace ID)
# Link Prometheus metrics to Tempo traces

# The killer feature: Grafana TraceQL
# → Query: { span.http.status_code = 500 } | select(span.db.statement)
# → Find all traces where an HTTP 500 occurred and show the DB query that caused it
```

> 💡 **Interview tip:** The compelling distributed tracing pitch for interviews: **metrics tell you WHAT is broken, logs tell you WHAT happened, traces tell you WHY and WHERE**. In a microservices architecture with 20 services, a 3-second request could be slow anywhere. Prometheus shows "payment service p99 latency = 3s" but doesn't tell you which downstream call is slow. A trace immediately shows: `payment-service (50ms) → database-service (2.8s) → ORDER_SELECT query on unindexed column`. The database query is the culprit — diagnosed in seconds, not hours. Always mention **OpenTelemetry as the standard** — avoid vendor lock-in by using the OTEL SDK (which exports to ANY backend) rather than a vendor-specific SDK.

---

*Kubernetes Complete Remaining Gaps*
*All 11 missing topics — Kubernetes is now 100% covered*
*DevOps / SRE Interview Preparation — nawab312 GitHub repositories*
