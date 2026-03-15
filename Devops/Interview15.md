# DevOps / SRE Interview Questions — File 2: Remaining Fundamentals + Advanced Gaps
## Q626–Q675 | Kubernetes Deep Dives + AWS + Linux + Docker + Git + Prometheus + SRE

> **Purpose:** Covers ALL remaining important gaps not in Q1–Q625.
> Zero overlap with any previous question.
> Every question includes **Key Points to Cover** + **💡 Interview Tips**.

---

## Topic Coverage

| Area | Questions |
|---|---|
| Kubernetes — Deep Dives | Q626–Q638 |
| AWS — Remaining Gaps | Q639–Q648 |
| Linux — Remaining Fundamentals | Q649–Q655 |
| Docker — Remaining Gaps | Q656–Q659 |
| Git — Fundamentals | Q660–Q663 |
| Prometheus — Remaining | Q664–Q667 |
| Terraform — Remaining | Q668–Q670 |
| CI/CD — Remaining | Q671–Q673 |
| SRE — Remaining | Q674–Q675 |

---

## SECTION 1 — KUBERNETES DEEP DIVES (Q626–Q638)

---

### Q626 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes Pod networking** end-to-end — how does a Pod get an IP address? How does one Pod communicate with another Pod on a different node?
>
> What is the **CNI (Container Network Interface)** and what role does it play? What are the Kubernetes networking requirements every CNI must satisfy?

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md`

#### Key Points to Cover:

**Kubernetes networking requirements (the 3 rules):**
```
1. Every Pod gets its own unique IP address
2. Pods on the same node can communicate without NAT
3. Pods on different nodes can communicate without NAT
   (Pod IP is routable across the cluster)

These are requirements — NOT an implementation.
CNI plugins implement these requirements differently.
```

**How a Pod gets an IP:**
```
1. kubelet creates the Pod sandbox (pause container)
2. kubelet calls CNI plugin via /opt/cni/bin/<plugin>
3. CNI plugin:
   a. Creates a virtual ethernet pair (veth pair)
   b. One end stays in Pod network namespace
   c. Other end connects to node's bridge (cbr0/cni0)
   d. Assigns IP from CIDR allocated to this node
   e. Sets up routing rules
4. Pod can now send/receive traffic

Pause container:
  → "infrastructure container" that holds the network namespace
  → All containers in the Pod SHARE this network namespace
  → That is why containers in same Pod communicate via localhost
```

**Pod-to-Pod communication (different nodes):**
```
Pod A (10.244.1.5) on Node 1 → Pod B (10.244.2.8) on Node 2

Option 1: Overlay network (Flannel VXLAN)
  → Packet from Pod A
  → Hits node 1's bridge (cni0)
  → Flannel encapsulates in VXLAN UDP packet
  → Sends to Node 2 via physical network
  → Node 2 Flannel decapsulates
  → Delivers to Pod B
  → Extra encapsulation overhead

Option 2: Underlay (Calico BGP, AWS VPC CNI)
  → Pod IPs are real IPs on the network
  → No encapsulation — native routing
  → AWS VPC CNI: Pod IPs are actual ENI secondary IPs
  → Packet routes directly via VPC routing table
  → Lower latency, higher performance
```

**Popular CNI plugins:**
```
Flannel:   Simple, overlay (VXLAN), no NetworkPolicy
Calico:    BGP routing or overlay, full NetworkPolicy, network security
Cilium:    eBPF-based, L7 NetworkPolicy, observability, performance
AWS VPC CNI: Pod gets real VPC IP, native routing, no overlay
Weave:     Simple, overlay, NetworkPolicy support
```

> 💡 **Interview tip:** On AWS EKS, **AWS VPC CNI** is the default — each Pod gets a secondary IP from the node's ENI. This is why EKS pod density is limited by instance type (each EC2 instance type has a max number of ENIs × max IPs per ENI). A `t3.medium` supports max 17 pods. Karpenter solves this with prefix delegation — each ENI can get a /28 (16 IPs) instead of individual IPs, dramatically increasing pod density.

---

### Q627 — Kubernetes | Conceptual | Intermediate

> Explain the **kube-apiserver** in depth — what is its role, why is it called the "center of Kubernetes," and what are the key responsibilities it has?
>
> What is the **API request pipeline** — what happens when a request hits the API server? What are **authentication**, **authorization**, **admission control**, and **validation**?

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md`

#### Key Points to Cover:

**kube-apiserver role:**
```
The ONLY component that talks to etcd.
ALL other components communicate ONLY through the API server.
It is the single source of truth for cluster state.

Responsibilities:
1. Serves the Kubernetes REST API (kubectl, controllers, kubelet all call it)
2. Authenticates requests (who are you?)
3. Authorizes requests (are you allowed to do this?)
4. Runs admission controllers (should this be allowed/modified?)
5. Validates resources (is this a valid Deployment spec?)
6. Reads/writes to etcd
7. Serves as notification hub (watch mechanism for controllers)
```

**API Request Pipeline:**
```
kubectl apply -f deployment.yaml
           ↓
1. AUTHENTICATION — Who are you?
   → Client certificate (kubeconfig)
   → Bearer token (ServiceAccount JWT)
   → OIDC token (external IdP)
   → Result: identity (user/serviceaccount) or 401

2. AUTHORIZATION — Are you allowed?
   → RBAC check: does this identity have permission?
   → Role/ClusterRole + RoleBinding/ClusterRoleBinding
   → Result: allow or 403

3. ADMISSION CONTROLLERS — Should this be modified/allowed?
   → MutatingAdmissionWebhook: can MODIFY the request
      (e.g., inject sidecar, add default labels)
   → ValidatingAdmissionWebhook: can REJECT the request
      (e.g., OPA Gatekeeper policy violation)
   → Built-in: LimitRanger, ResourceQuota, PodSecurity
   → Result: modified request, or 400/403

4. VALIDATION — Is the spec valid?
   → Check against OpenAPI schema
   → Required fields present?
   → Valid values for enum fields?
   → Result: valid object or 422

5. PERSIST to etcd
   → Writes to etcd
   → Returns 200/201 to client

6. WATCHERS NOTIFIED
   → Controller-manager watching Deployments sees new object
   → Begins reconciliation
```

> 💡 **Interview tip:** The **admission controller** stage is where most security and policy enforcement happens. OPA Gatekeeper, Kyverno, and Pod Security Admission all work as admission webhooks. The key distinction: mutating webhooks run BEFORE validating webhooks. This allows mutating webhooks to inject defaults/sidecars, then validating webhooks check the final object. If you reject at validation, no resource is created. If the webhook itself is unreachable and `failurePolicy: Fail`, all requests are rejected — a common cause of cluster-wide outages.

---

### Q628 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes Resource QoS classes** — what are **Guaranteed**, **Burstable**, and **BestEffort**? How does Kubernetes decide which class a Pod belongs to?
>
> When nodes run out of memory, in what order does Kubernetes evict Pods? What is the **OOM killer** and how does it interact with QoS classes?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

#### Key Points to Cover:

**3 QoS Classes:**
```
Guaranteed (highest priority):
  Conditions:
    - EVERY container has CPU limit = CPU request
    - EVERY container has memory limit = memory request
  Result: pod gets exactly what it requests, no more, no less
  Eviction order: LAST to be evicted

  Example:
    resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
      limits:
        cpu: "500m"      # must equal request
        memory: "256Mi"  # must equal request

Burstable (medium priority):
  Conditions:
    - At least one container has a request OR limit
    - Does NOT meet Guaranteed conditions
  Result: can burst above request (up to limit) when node has spare resources
  Eviction order: second to be evicted

  Example:
    resources:
      requests:
        cpu: "250m"
        memory: "128Mi"
      limits:
        cpu: "1"          # limit > request → Burstable
        memory: "512Mi"

BestEffort (lowest priority):
  Conditions:
    - NO containers have any requests OR limits
  Result: gets whatever is left over, first to be evicted
  Eviction order: FIRST to be evicted
  Use case: non-critical dev workloads only

  Example:
    resources: {}   # no requests or limits = BestEffort
```

**Eviction and OOM:**
```
Node Memory Pressure:
1. kubelet notices memory below eviction threshold
2. Eviction order:
   a. BestEffort pods first
   b. Burstable pods using most above their request
   c. Guaranteed pods (last resort)

OOM Killer:
  → Linux kernel kills process when memory exhausted
  → Before kubelet eviction can act
  → oom_score_adj determines priority:
     BestEffort: oom_score_adj = 1000 (killed first)
     Burstable:  oom_score_adj = 2-999 (proportional to usage)
     Guaranteed: oom_score_adj = -997 (protected, killed last)

If a Guaranteed pod is OOMKilled:
  → It exceeded its memory LIMIT
  → Even Guaranteed pods can be OOMKilled if they hit their limit
  → Fix: increase memory limit or fix memory leak
```

> 💡 **Interview tip:** The most common mistake is not setting resource requests AND limits — without requests, pods are BestEffort and first to be evicted during memory pressure. Without limits, a single memory-leaking pod can consume all node memory and cause cascading evictions. **Best practice: always set both requests AND limits.** For production critical services, set them equal (Guaranteed class) to prevent eviction entirely.

---

### Q629 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes RBAC** in depth — what are **Roles**, **ClusterRoles**, **RoleBindings**, and **ClusterRoleBindings**?
>
> What is the difference between a Role and a ClusterRole? When would you use a RoleBinding vs ClusterRoleBinding? Write RBAC rules that give a developer read-only access to Pods in the `production` namespace but allow them to exec into Pods.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md`

#### Key Points to Cover:

**4 RBAC objects:**
```
Role:               defines permissions WITHIN a namespace
ClusterRole:        defines permissions cluster-wide OR for cluster-scoped resources
RoleBinding:        grants a Role/ClusterRole to a subject WITHIN a namespace
ClusterRoleBinding: grants a ClusterRole to a subject cluster-wide
```

**Role vs ClusterRole:**
```
Use Role when:
  → Permissions limited to ONE specific namespace
  → e.g., "read pods in production namespace"

Use ClusterRole when:
  → Permissions across ALL namespaces
  → Cluster-scoped resources (nodes, PVs, namespaces themselves)
  → e.g., "read nodes anywhere", "read pods in all namespaces"

ClusterRole + RoleBinding (powerful combination):
  → Reuse same ClusterRole in different namespaces
  → e.g., create "developer" ClusterRole once
  → Bind it per-namespace via RoleBinding
  → Team A gets "developer" access to namespace A
  → Team B gets "developer" access to namespace B
```

**Developer read-only + exec access:**
```yaml
# Step 1: ClusterRole defining the permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader-exec
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods/exec"]         # allow kubectl exec
  verbs: ["create"]                # exec uses POST = create verb
- apiGroups: [""]
  resources: ["pods/portforward"]
  verbs: ["create"]
---
# Step 2: RoleBinding — grant ONLY in production namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-pod-access
  namespace: production              # scoped to this namespace only
subjects:
- kind: User
  name: siddharth@company.com
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: dev-team                     # entire group
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole                  # reference the ClusterRole
  name: pod-reader-exec
  apiGroup: rbac.authorization.k8s.io
```

**Verify RBAC:**
```bash
# Can Siddharth list pods in production?
kubectl auth can-i list pods -n production --as siddharth@company.com
# yes

# Can Siddharth delete pods in production?
kubectl auth can-i delete pods -n production --as siddharth@company.com
# no

# Full permissions check
kubectl auth can-i --list -n production --as siddharth@company.com
```

> 💡 **Interview tip:** `pods/exec` is one of the most commonly misunderstood RBAC rules. Exec is a **subresource** — it needs `pods/exec: create` permission, not `pods: exec`. Also important: anyone with `pods/exec` access can effectively escape RBAC for many operations (they can exec into a pod and do anything the pod's service account can do). In production, treat `pods/exec` access as highly privileged.

---

### Q630 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes Pod Security** — what was **PodSecurityPolicy (PSP)** and why was it removed? What replaced it?
>
> What is **Pod Security Admission (PSA)** and what are the 3 policy levels? What is **`runAsNonRoot`**, **`privileged`**, **`allowPrivilegeEscalation`**, and **`readOnlyRootFilesystem`**?

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md`

#### Key Points to Cover:

**PSP removal:**
```
PodSecurityPolicy (PSP): deprecated K8s 1.21, removed K8s 1.25
Problems with PSP:
  → Complex: had to grant RBAC permission to USE the policy
  → Confusing: if ANY policy allowed pod, pod was admitted
  → No dry-run mode to test policies
  → Hard to migrate between policies
```

**Pod Security Admission (PSA) — replacement:**
```
Built into kube-apiserver (no webhook needed)
Applied at NAMESPACE level via labels

3 policy levels:
  privileged: no restrictions (for system namespaces like kube-system)
  baseline:   prevents known privilege escalations (blocks most dangerous settings)
  restricted: heavily restricted, follows security best practices

3 enforcement modes per level:
  enforce: reject violating pods
  warn:    allow but warn (for migration)
  audit:   allow but log to audit (for testing)

Apply via namespace labels:
  kubectl label namespace production \
    pod-security.kubernetes.io/enforce=restricted \
    pod-security.kubernetes.io/warn=restricted \
    pod-security.kubernetes.io/audit=restricted
```

**Key security context fields:**
```yaml
spec:
  securityContext:              # Pod-level
    runAsNonRoot: true          # Pod MUST run as non-root (UID != 0)
    runAsUser: 1000             # specific UID to run as
    fsGroup: 2000               # group for mounted volumes
    seccompProfile:
      type: RuntimeDefault      # seccomp filtering (restricted policy requires this)

  containers:
  - name: app
    securityContext:            # Container-level (overrides Pod-level)
      privileged: false         # NOT root on the HOST (very dangerous if true)
      allowPrivilegeEscalation: false  # prevent sudo, setuid binaries
      readOnlyRootFilesystem: true     # no writes to container filesystem
      capabilities:
        drop: ["ALL"]           # drop all Linux capabilities
        add: ["NET_BIND_SERVICE"]  # only add what's needed
```

**What each setting prevents:**
```
privileged: true      → container IS root on the host → full node access
                         Use only for: node agents (Fluentbit, cni plugins)

allowPrivilegeEscalation: → prevents setuid/setgid programs from gaining
                           more privileges (sudo, ping in some distros)

readOnlyRootFilesystem:  → if container is compromised, attacker cannot
                           write new executables or scripts
                           requires: tmpfs for /tmp, emptyDir for writable paths

runAsNonRoot:          → even if someone breaks out of app, UID ≠ 0
                         reduces blast radius
```

> 💡 **Interview tip:** The **`restricted` policy level** in PSA is what most production workloads should target. It requires: `runAsNonRoot`, `allowPrivilegeEscalation: false`, `drop ALL capabilities`, `seccompProfile: RuntimeDefault`. The migration path: start with `audit` mode to find violations → fix them → move to `warn` → finally `enforce`. Never jump straight to `enforce` on existing namespaces — it will break running workloads.

---

### Q631 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes Jobs and CronJobs** — what are they and when would you use them instead of a Deployment?
>
> What is the `completions`, `parallelism`, `backoffLimit`, and `activeDeadlineSeconds` in a Job? What is `concurrencyPolicy` in CronJob and what happens when a CronJob misses its schedule?

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`

#### Key Points to Cover:

**Job vs Deployment:**
```
Deployment: runs pods that should run FOREVER (web servers, APIs)
  → restartPolicy: Always (always restart on failure)
  → desired state: N replicas always running

Job: runs pods to COMPLETION (one-time task)
  → restartPolicy: OnFailure or Never
  → desired state: task completed successfully

Use Job for:
  → Database migrations
  → Batch data processing
  → Sending bulk emails
  → One-time setup/cleanup tasks
  → Image processing jobs
```

**Key Job fields:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  completions: 5          # total successful completions needed
  parallelism: 2          # max pods running simultaneously
  backoffLimit: 3         # retry up to 3 times before marking failed
  activeDeadlineSeconds: 3600  # kill job if running > 1 hour
  ttlSecondsAfterFinished: 300 # auto-delete job 5 min after completion
  template:
    spec:
      restartPolicy: OnFailure  # restart container (not pod) on failure
      containers:
      - name: migration
        image: my-app:v2
        command: ["./migrate.sh"]
```

**CronJob:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"   # 2am every day (cron syntax)
  concurrencyPolicy: Forbid  # options: Allow, Forbid, Replace

  # Allow:   allow concurrent runs (previous still running = new one starts)
  # Forbid:  skip new run if previous still running (safe for most batch jobs)
  # Replace: cancel previous run and start new one

  startingDeadlineSeconds: 3600  # if missed schedule, can start up to 1hr late
  successfulJobsHistoryLimit: 3  # keep last 3 successful job records
  failedJobsHistoryLimit: 1      # keep last 1 failed job record
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:latest
```

**Missed schedule:**
```
If K8s was down during scheduled time:
  startingDeadlineSeconds: 3600
  → If missed by < 1 hour: run it now (late)
  → If missed by > 1 hour: skip this run, wait for next schedule

If missed > 100 schedules: CronJob is suspended automatically
  (prevents thousands of jobs queuing up after long downtime)
```

> 💡 **Interview tip:** `concurrencyPolicy: Forbid` is the safest default for most batch jobs — it prevents the "duplicate processing" problem where a slow job from previous run is still running when the next schedule fires. Without this, you get two jobs processing the same data simultaneously. The `ttlSecondsAfterFinished` field is critical for cluster hygiene — without it, completed Jobs and their Pods accumulate forever, cluttering `kubectl get pods` output.

---

### Q632 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes multi-container Pod patterns** — Sidecar, Init Container, Ambassador, and Adapter. What problem does each solve?
>
> When would you use an **init container** vs a **sidecar**? Write a Pod spec that uses both an init container (to wait for a database) and a sidecar (for log shipping).

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`

#### Key Points to Cover:

**4 Multi-container patterns:**

```
Init Container:
  → Runs BEFORE main containers start
  → Must complete successfully before main containers begin
  → Run sequentially (one at a time)
  → Use: pre-conditions check, data seeding, wait for dependencies
  → Example: wait for DB to be ready, clone git repo, set up permissions

Sidecar:
  → Runs ALONGSIDE main container (same lifecycle)
  → Enhances main container's functionality
  → Shares network and storage namespaces
  → Use: log shipping (Fluentbit), service mesh proxy (Envoy), metrics exporter

Ambassador:
  → Proxy for network traffic
  → Main container connects to localhost → ambassador proxies to real service
  → Use: connection pooling, circuit breaker, local Redis proxy

Adapter:
  → Transforms output of main container
  → Normalizes metrics/logs into standard format
  → Use: legacy app produces non-Prometheus metrics → adapter converts them
```

**Pod spec with init container + sidecar:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payment-service
spec:
  # Init container runs FIRST — waits for DB
  initContainers:
  - name: wait-for-db
    image: busybox:1.35
    command:
    - sh
    - -c
    - |
      until nc -z postgres-service 5432; do
        echo "Waiting for PostgreSQL...";
        sleep 2;
      done;
      echo "PostgreSQL is ready!"
    # Main containers do NOT start until this exits with code 0

  - name: run-migrations
    image: payment-app:v2
    command: ["./migrate.sh"]
    # Second init container runs after first completes

  # Main containers run together after all init containers complete
  containers:
  - name: payment-app           # main application
    image: payment-app:v2
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: logs
      mountPath: /var/log/app

  - name: log-shipper           # sidecar — shares log volume
    image: fluent/fluent-bit:2.0
    volumeMounts:
    - name: logs
      mountPath: /var/log/app   # reads same log directory
      readOnly: true
    - name: fluentbit-config
      mountPath: /fluent-bit/etc

  volumes:
  - name: logs
    emptyDir: {}                # shared between app and sidecar
  - name: fluentbit-config
    configMap:
      name: fluentbit-config
```

> 💡 **Interview tip:** In **Kubernetes 1.29+**, sidecar containers have a dedicated `initContainers` field with `restartPolicy: Always` — this is the "native sidecar" feature. Native sidecars start before main containers (like init containers) but keep running throughout the Pod's lifecycle (like regular sidecars). This solves the previous problem where a sidecar defined as a regular container could start AFTER the main container, causing the log shipper to miss early startup logs.

---

### Q633 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes HPA (Horizontal Pod Autoscaler)** in depth — how does it work, what metrics can it scale on, and what is the scaling algorithm?
>
> What is the difference between HPA, VPA, and KEDA? What is **scale-to-zero** and which tool enables it?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

#### Key Points to Cover:

**HPA scaling algorithm:**
```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / desiredMetricValue))

Example:
  currentReplicas: 3
  targetCPU: 50%
  currentCPU: 90%
  → desiredReplicas = ceil(3 × 90/50) = ceil(5.4) = 6

Stabilization window:
  Scale up:   default 0s (scale up immediately)
  Scale down: default 300s (wait 5 min before scaling down)
  Prevents thrashing when metrics fluctuate
```

**HPA metric types:**
```yaml
spec:
  metrics:
  # 1. Resource metrics (CPU/memory)
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60  # target 60% CPU

  # 2. Custom metrics (from Prometheus via custom-metrics-apiserver)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"     # 100 req/s per pod

  # 3. External metrics (SQS queue depth, etc.)
  - type: External
    external:
      metric:
        name: sqs_queue_depth
      target:
        type: Value
        value: "30"             # scale when queue > 30 messages
```

**HPA vs VPA vs KEDA:**
```
HPA (Horizontal Pod Autoscaler):
  → Adds/removes PODS (horizontal scaling)
  → Built into Kubernetes
  → Cannot scale below 1 replica (no scale-to-zero)
  → Best for: stateless services with variable traffic

VPA (Vertical Pod Autoscaler):
  → Changes CPU/memory REQUESTS of existing pods (vertical scaling)
  → Requires pod restart to apply (disruptive)
  → Best for: stateful workloads, batch jobs, right-sizing
  → Do NOT use HPA + VPA on CPU simultaneously (conflicts)

KEDA (Kubernetes Event Driven Autoscaler):
  → Extends HPA with 60+ event sources
  → CAN scale to ZERO (no pods when no events)
  → Sources: SQS, Kafka, RabbitMQ, Prometheus, Cron, GitHub, etc.
  → Best for: event-driven, batch, queue consumers

Scale-to-zero with KEDA:
  → 0 messages in SQS → scale to 0 pods (save cost)
  → New message arrives → KEDA scales from 0 to 1 pod
  → HPA cannot do this (minimum 1 replica)
```

> 💡 **Interview tip:** The most common HPA mistake is **scaling on CPU without setting CPU requests** — HPA cannot calculate CPU utilization percentage if there is no request to compare against. `<unknown>/50%` in `kubectl get hpa` means metrics are not available, almost always because CPU requests are not set on the pods. Always set CPU requests before enabling HPA. Also mention that HPA requires the **metrics-server** to be installed — it is not installed by default on all Kubernetes distributions.

---

### Q634 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes taints and tolerations** — what problem do they solve and how do they work?
>
> What are the 3 taint effects — `NoSchedule`, `PreferNoSchedule`, `NoExecute`? Give real-world examples where you would use each. What is the difference between taints+tolerations and node affinity?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

#### Key Points to Cover:

**What taints solve:**
```
Without taints: scheduler places pods on ANY node
Problem: GPU nodes getting general-purpose workloads
         Tainted nodes (in maintenance) getting new pods

Taint = "I repel pods by default"
Toleration = "I can tolerate this node's taint"
```

**3 Taint effects:**
```
NoSchedule:
  → New pods WITHOUT matching toleration are NOT scheduled here
  → Existing pods on the node: NOT affected
  → Use: GPU nodes (only GPU pods), dedicated nodes
  kubectl taint nodes gpu-node-1 dedicated=gpu:NoSchedule

PreferNoSchedule:
  → Scheduler tries to AVOID placing pods here
  → If no other options: will still schedule here
  → Soft version of NoSchedule
  → Use: "prefer not to use this node" (degraded but still usable)

NoExecute:
  → New pods: not scheduled (like NoSchedule)
  → Existing pods WITHOUT matching toleration: EVICTED
  → Use: node maintenance, removing node from rotation
  kubectl taint nodes node-1 maintenance=true:NoExecute
  # All pods without this toleration immediately evicted
```

**Toleration to run on tainted node:**
```yaml
spec:
  tolerations:
  # Exact match: tolerate the specific taint
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule

  # Tolerate ANY taint with this key (any value)
  - key: dedicated
    operator: Exists
    effect: NoSchedule

  # Tolerate NoExecute with timeout (stay for 60s before being evicted)
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 60   # stay 60 seconds, then evict
```

**Taints vs Node Affinity:**
```
Taints+Tolerations: REPEL (node repels pods)
  → Node says: "I don't want most pods"
  → Pod says: "I can handle this node"
  → Use: dedicated nodes, GPU, maintenance

Node Affinity: ATTRACT (pod is attracted to specific nodes)
  → Pod says: "I want to run on nodes with label X"
  → Node has no say
  → Use: pod placement preference, AZ spread

Real production setup for GPU nodes:
  1. Taint GPU nodes (repel non-GPU pods)
     kubectl taint node gpu-1 nvidia.com/gpu=true:NoSchedule
  2. Node affinity for GPU pods (attract to GPU nodes)
     nodeAffinity: requiredDuringScheduling → label gpu=true
  3. GPU pods tolerate the taint AND request GPU resource
  → Result: only GPU pods on GPU nodes, no general pods
```

> 💡 **Interview tip:** The `node.kubernetes.io/not-ready:NoExecute` and `node.kubernetes.io/unreachable:NoExecute` taints are **automatically added by Kubernetes** when a node becomes not-ready. By default, pods have a built-in toleration of 300 seconds for these taints — this is why pods stay on NotReady nodes for 5 minutes before being evicted. You can override this with custom tolerationSeconds. This explains why after a node failure, your pods don't immediately reschedule — Kubernetes is waiting to see if the node recovers.

---

### Q635 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes PersistentVolumes (PV)**, **PersistentVolumeClaims (PVC)**, and **StorageClasses** — how does the dynamic provisioning flow work?
>
> What is the difference between **static provisioning** and **dynamic provisioning**? What happens to the PV when the PVC is deleted? What are the reclaim policies?

📁 **Reference:** `nawab312/Kubernetes` → `04_STORAGE.md`

#### Key Points to Cover:

**Static vs Dynamic provisioning:**
```
Static provisioning (manual):
  Admin creates PV → Developer creates PVC → K8s binds them
  → Admin must know storage requirements in advance
  → More operational overhead

Dynamic provisioning (automatic):
  Developer creates PVC → StorageClass auto-provisions PV → K8s binds
  → No admin intervention needed
  → StorageClass defines the provisioner (EBS, EFS, etc.)
  → Modern approach, used in almost all cloud deployments
```

**Dynamic provisioning flow:**
```
1. Developer creates PVC:
   spec:
     storageClassName: gp3          # which StorageClass to use
     accessModes: [ReadWriteOnce]
     resources:
       requests:
         storage: 20Gi

2. PVC created (status: Pending)

3. StorageClass provisioner sees new PVC
   → Calls cloud API: create 20GB EBS gp3 volume
   → EBS volume created: vol-12345

4. PV automatically created for this EBS volume

5. PV bound to PVC (both show status: Bound)

6. Pod can now use the PVC as a volume:
   volumes:
   - name: data
     persistentVolumeClaim:
       claimName: my-pvc
```

**StorageClass example:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com        # AWS EBS CSI driver
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete                # what happens when PVC deleted
volumeBindingMode: WaitForFirstConsumer  # create EBS in same AZ as Pod
allowVolumeExpansion: true           # allow PVC resize
```

**Reclaim policies:**
```
Delete (default for dynamic):
  → PVC deleted → PV deleted → EBS volume DELETED
  → Data is gone permanently
  → Use: ephemeral data, test environments

Retain:
  → PVC deleted → PV stays (status: Released)
  → EBS volume NOT deleted — data preserved
  → Admin must manually reclaim/delete PV and EBS
  → Use: production databases, important data

Recycle (deprecated):
  → Basic scrub (rm -rf /volume/*) and make available again
  → Deprecated — use dynamic provisioning instead
```

> 💡 **Interview tip:** `volumeBindingMode: WaitForFirstConsumer` is critical for multi-AZ clusters — without it, the EBS volume is created in a random AZ immediately when the PVC is created. If the Pod is then scheduled to a different AZ, it cannot mount the EBS volume (EBS is AZ-specific). With `WaitForFirstConsumer`, EBS creation is delayed until the Pod is scheduled, ensuring the volume is created in the same AZ as the Pod.

---

### Q636 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes Service Accounts** — what are they, why do pods need them, and how do they work?
>
> What is the difference between the **default service account** and a custom one? What is a **token projection** and why is it more secure than the old mounted secret tokens?

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md`

#### Key Points to Cover:

**What Service Accounts are:**
```
Every Pod runs as a ServiceAccount (like a user identity for pods)
Default: pods use the "default" ServiceAccount in their namespace

Why pods need identity:
  → Pod calls kube-apiserver → must authenticate → SA provides identity
  → RBAC grants permissions to ServiceAccount
  → AWS IRSA: SA maps to IAM role for AWS API access
```

**Old approach (token secret) vs New (projected token):**
```
Old approach (< K8s 1.22):
  → Static token stored in a Secret
  → Automatically mounted at /var/run/secrets/kubernetes.io/serviceaccount/token
  → Token NEVER expires
  → Security risk: if container compromised → token stolen → used forever

New approach (projected token):
  → Short-lived token (1 hour expiry by default)
  → Automatically rotated by kubelet before expiry
  → Audience-bound (only valid for specific API server)
  → kubelet mounts fresh token before old one expires
  → Token stolen → useless after 1 hour
```

**Custom ServiceAccount with RBAC:**
```yaml
# Step 1: Create ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-service-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456:role/payment-role  # IRSA

---
# Step 2: Grant permissions via RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payment-service-permissions
  namespace: production
subjects:
- kind: ServiceAccount
  name: payment-service-sa
  namespace: production
roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# Step 3: Use in Pod
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: payment-service-sa   # use custom SA
  automountServiceAccountToken: true        # default true

  # Or DISABLE if pod doesn't need K8s API access:
  automountServiceAccountToken: false       # security best practice
```

**Best practices:**
```
1. Never use the "default" ServiceAccount for production pods
   → default SA often has accumulated permissions
   → Create dedicated SA per workload

2. Set automountServiceAccountToken: false if pod doesn't call K8s API
   → Most application pods don't need it
   → Reduces attack surface

3. Use IRSA (IAM Roles for Service Accounts) for AWS API access
   → Pod's SA token → AWS STS AssumeRoleWithWebIdentity
   → Temporary AWS credentials (no long-lived keys needed)
```

> 💡 **Interview tip:** **`automountServiceAccountToken: false`** is a security hardening step most teams forget. By default, EVERY pod gets a token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token` — even if the pod never calls the Kubernetes API. If an attacker compromises the pod, they have a token that can be used to interact with the Kubernetes API. Setting this to false eliminates this attack vector for pods that don't need Kubernetes API access.

---

### Q637 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes ConfigMap and Secret management best practices**. What are the risks of secrets being stored in etcd?
>
> What is the difference between mounting a ConfigMap as a **volume** vs injecting as **environment variables**? What is **hot reload** of ConfigMaps and when does it work?

📁 **Reference:** `nawab312/Kubernetes` → `05_CONFIGURATION_SECRETS.md`

#### Key Points to Cover:

**ConfigMap vs Secret:**
```
ConfigMap:
  → Non-sensitive configuration (app config, feature flags, connection strings)
  → Stored as plaintext in etcd
  → Max size: 1MB

Secret:
  → Sensitive data (passwords, tokens, certificates)
  → Stored base64 encoded in etcd (NOT encrypted by default!)
  → base64 is NOT encryption — anyone with etcd access can decode
  → Real encryption: enable etcd encryption at rest (KMS provider)
  → Max size: 1MB
```

**Volume mount vs Environment variable:**
```
Environment variable injection:
  env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password

  Problems:
  → Visible in process list (ps aux)
  → Visible in docker inspect
  → Child processes inherit (may be logged)
  → NO automatic update — pod must restart to get new value
  → Logs might accidentally capture env vars

Volume mount:
  volumeMounts:
  - name: db-secret
    mountPath: /etc/secrets
    readOnly: true
  volumes:
  - name: db-secret
    secret:
      secretName: db-secret

  Benefits:
  → File permissions can restrict access (chmod 400)
  → HOT RELOAD: kubelet updates mounted files when Secret changes
  → Not in process list
  → Better for certificates (file-based)
```

**Hot reload behavior:**
```
Volume-mounted ConfigMaps/Secrets:
  → kubelet syncs every 60s (default kubelet sync period)
  → Updated value appears in /etc/config/ after up to 60s
  → Application must watch file for changes to pick up update
  → Flask, Node.js apps typically need reload logic

Environment variables:
  → NEVER updated without pod restart
  → kubectl rollout restart deployment/myapp to pick up new secret

When hot reload works automatically:
  → Nginx: watches config files, reload signal sent
  → Most apps: need code to watch config files (inotify)
  → Kubernetes secret rotation: works if app reads file on each request

When hot reload does NOT work:
  → App reads config once on startup (most Java apps)
  → Solution: use a sidecar watcher that sends SIGHUP to main process
```

> 💡 **Interview tip:** The most important security point about Kubernetes Secrets: **base64 is not encryption**. `kubectl get secret mysecret -o yaml` shows the base64-encoded value which anyone can decode. Real security requires **etcd encryption at rest** using a KMS provider (AWS KMS for EKS). On EKS, enable this with `--encryption-provider-config`. Also mention **External Secrets Operator** as the production approach — secrets stay in AWS Secrets Manager (encrypted, audited, rotatable) and are synced into Kubernetes Secrets just-in-time.

---

### Q638 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes networking with Ingress** — what is the complete flow from a user's browser request to a Pod?
>
> What is **TLS termination** at the Ingress? What is **path-based routing** vs **host-based routing**? What are the common **Nginx Ingress annotations** every DevOps engineer should know?

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md`

#### Key Points to Cover:

**Complete request flow:**
```
User browser
    ↓
DNS: api.company.com → ALB IP (AWS) or LoadBalancer IP
    ↓
AWS ALB (port 443 → 80)
    ↓
Kubernetes Service (ingress-nginx NodePort)
    ↓
Nginx Ingress Controller Pod
    ↓ (matches rules in Ingress resources)
Kubernetes Service (payment-service ClusterIP)
    ↓
Pod (any healthy pod)
```

**Host-based vs Path-based routing:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - api.company.com
    - app.company.com
    secretName: company-tls-cert    # TLS terminated at Ingress

  rules:
  # HOST-based routing (different subdomains → different services)
  - host: api.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

  - host: app.company.com           # different host → different service
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80

  # PATH-based routing (same host, different paths)
  - host: company.com
    http:
      paths:
      - path: /api                  # company.com/api → api service
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /                     # company.com/* → frontend
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

**Essential Nginx Ingress annotations:**
```yaml
annotations:
  # Rate limiting
  nginx.ingress.kubernetes.io/limit-rps: "100"
  nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"

  # Timeouts
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "120"

  # SSL redirect
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

  # Request body size (for file uploads)
  nginx.ingress.kubernetes.io/proxy-body-size: "50m"

  # CORS
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.company.com"

  # Authentication
  nginx.ingress.kubernetes.io/auth-url: "https://auth.company.com/check"
  nginx.ingress.kubernetes.io/auth-signin: "https://auth.company.com/login"

  # Canary routing
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% to canary
```

> 💡 **Interview tip:** **`pathType`** matters significantly: `Prefix` means `/api` matches `/api`, `/api/users`, `/api/payments`. `Exact` means `/api` matches ONLY `/api`. `ImplementationSpecific` is controller-dependent. The most common mistake is using `Exact` when `Prefix` is needed, causing all sub-paths to return 404. Also: order matters for path rules — more specific paths should come BEFORE less specific ones (longest match wins in Nginx, but Kubernetes processes rules in order).

---

## SECTION 2 — AWS REMAINING GAPS (Q639–Q648)

---

### Q639 — AWS | Conceptual | Intermediate

> Explain **AWS Lambda** in depth — how does it work internally? What is a **cold start**, what causes it, and how do you mitigate it?
>
> What are Lambda **concurrency limits**, **reserved concurrency**, and **provisioned concurrency**? What are Lambda **layers** and Lambda **destinations**?

📁 **Reference:** `nawab312/AWS` — Lambda, cold starts, concurrency sections

#### Key Points to Cover:

**How Lambda works internally:**
```
1. Event triggers Lambda (API Gateway, S3, SQS, etc.)
2. Lambda service checks for a warm execution environment
3. If WARM: reuse existing environment (fast, milliseconds)
4. If COLD: create new environment:
   a. Download your deployment package (ZIP or container)
   b. Start execution environment (MicroVM via Firecracker)
   c. Initialize runtime (Python/Node/Java/etc.)
   d. Run initialization code (outside handler)
   e. Run your handler function
5. After handler returns: environment kept warm ~15 min
6. If another request comes: reuse (warm start)
```

**Cold start duration:**
```
Python/Node.js: 100-500ms
Java/C#:        500ms-2s (JVM startup is slow)
Container image: 1-5s (larger image = slower)

Cold start triggers:
  → First invocation ever
  → After 15+ min of inactivity
  → New concurrent invocations (when all environments busy)
  → Deployment update (all environments recycled)
  → VPC configuration change

Mitigation:
  → Provisioned Concurrency (keep N environments warm)
  → Keep Lambda warm with scheduled ping (workaround)
  → Use Python/Node (faster cold start than Java)
  → Reduce deployment package size
  → SnapStart (for Java Lambda — snapshot after init)
```

**Concurrency:**
```
Account default: 1000 concurrent executions per region
Reserved concurrency:
  → Set max concurrent for THIS function
  → Protects backend (throttles Lambda before overwhelming DB)
  → Reserves capacity from account pool

Provisioned concurrency:
  → Pre-initializes N execution environments
  → Eliminates cold starts for those N concurrent requests
  → Costs money even when idle (pay for warm environments)
  → Use for: latency-critical APIs, user-facing functions
```

**Lambda layers:**
```
Layer = ZIP file with libraries/dependencies
  → Shared across multiple functions
  → Max 5 layers per function
  → Separates dependencies from function code
  → Deploy function without re-uploading dependencies
  → Example: pandas layer shared by 20 data processing functions

Lambda destinations (async invocations):
  → On success: send result to SQS/SNS/EventBridge/Lambda
  → On failure: send to DLQ or SQS/SNS
  → Better than DLQ alone — captures success events too
  → Use for: async workflow chaining
```

> 💡 **Interview tip:** **Provisioned Concurrency costs money whether or not requests are made** — this is often forgotten. For APIs with predictable traffic (business hours only), combine Provisioned Concurrency with **Application Auto Scaling** to scale it up before peak hours and down overnight. This gives cold-start-free performance during peak with lower cost during off-hours. Also mention Lambda **SnapStart** for Java — it snapshots the initialized state and restores it for new executions, reducing Java cold starts from 2+ seconds to under 100ms.

---

### Q640 — AWS | Conceptual | Intermediate

> Explain **AWS CloudWatch Metric Filters** — what are they and how do they work?
>
> Write a metric filter that counts the number of ERROR log lines per minute in a log group and creates an alarm when errors exceed 10 per minute. What is the difference between `filterPattern` and a CloudWatch Logs Insights query?

📁 **Reference:** `nawab312/AWS` — CloudWatch Metric Filters, log metrics sections

#### Key Points to Cover:

**What Metric Filters do:**
```
CloudWatch Logs → (metric filter) → CloudWatch Metric

You define a pattern to match in log lines
CloudWatch counts matches and publishes as a custom metric
This metric can then trigger alarms, dashboards, anomaly detection

Use cases:
  → Count ERROR lines → alert on error rate
  → Count 5xx HTTP responses → alert on error %
  → Track specific business events (payments processed)
  → Monitor security events (failed logins)
```

**Create metric filter + alarm:**
```bash
# Step 1: Create metric filter
aws logs put-metric-filter \
  --log-group-name /app/payment-service \
  --filter-name ErrorCount \
  --filter-pattern "ERROR" \
  --metric-transformations \
    metricName=ErrorCount,\
    metricNamespace=PaymentService,\
    metricValue=1,\
    defaultValue=0,\
    unit=Count

# Filter patterns:
# "ERROR"              → lines containing ERROR
# "ERROR" "timeout"   → lines with BOTH words
# "[level=ERROR]"      → JSON: level field equals ERROR
# "[timestamp, level=ERROR, ...]" → space-delimited fields
# "?ERROR ?WARN"       → lines with ERROR or WARN
```

**Create alarm on metric filter:**
```bash
# Step 2: Create alarm
aws cloudwatch put-metric-alarm \
  --alarm-name payment-high-errors \
  --metric-name ErrorCount \
  --namespace PaymentService \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --statistic Sum \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-1:123456:alerts
```

**Metric Filter vs Logs Insights:**
```
Metric Filter:
  → Real-time (processes logs as they arrive)
  → Creates persistent CloudWatch metric
  → Can trigger alarms
  → Simple pattern matching only
  → Cost: charged per metric filter (free tier available)

CloudWatch Logs Insights:
  → Ad-hoc queries on historical log data
  → SQL-like query language with aggregations
  → NOT real-time (query runs on demand)
  → Cannot trigger alarms directly
  → Cost: charged per GB scanned
  → Use for: investigation, debugging, analysis
```

> 💡 **Interview tip:** **`defaultValue=0`** in metric transformations is critical for alarms. Without it, when no logs arrive in a period (no errors), the metric publishes NO DATA POINT — CloudWatch alarm goes to `INSUFFICIENT_DATA` state, not `OK`. With `defaultValue=0`, a value of 0 is published even when no matching log lines exist, keeping the alarm in `OK` state properly. Always set `defaultValue=0` for error count metrics.

---

### Q641 — AWS | Conceptual | Intermediate

> Explain **AWS Auto Scaling Groups (ASG)** — what is the difference between **Launch Template** and **Launch Configuration**? What is a **mixed instances policy** and why does it matter for cost optimization?
>
> Explain the difference between **Target Tracking**, **Step Scaling**, **Simple Scaling**, and **Scheduled scaling** policies.

📁 **Reference:** `nawab312/AWS` — ASG, Launch Templates, scaling policies sections

#### Key Points to Cover:

**Launch Configuration vs Launch Template:**
```
Launch Configuration (old, avoid):
  → Immutable — cannot be modified (must create new one)
  → No versioning
  → Cannot use T2/T3 Unlimited
  → Will be deprecated by AWS

Launch Template (modern, use this):
  → Versioned (v1, v2, v3...)
  → Can inherit from another template
  → Supports mixed instances (Spot + On-Demand)
  → Full EC2 features (metadata v2, Nitro, etc.)
  → Always use Launch Template for new ASGs
```

**Mixed instances policy:**
```
ASG with mixed instances = Spot + On-Demand in same group
Benefits: 70-80% cost savings vs all On-Demand

Configuration:
  Base capacity: 2 On-Demand (always reliable)
  Beyond base: 80% Spot, 20% On-Demand

Multiple Spot pools (diversification):
  ["m5.xlarge", "m5a.xlarge", "m6i.xlarge", "c5.xlarge", "c6i.xlarge"]
  → If one pool interrupted, capacity from others
  → Lower interruption probability with more pools

Allocation strategy:
  capacity-optimized: launch from most available Spot pool
                      (lowest interruption probability)
  lowest-price:       cheapest Spot across specified pools
  → Use capacity-optimized for production workloads
```

**4 Scaling Policy types:**
```
1. Target Tracking (simplest, recommended):
   → Set target metric value
   → AWS calculates and creates scale-out/scale-in alarms automatically
   → Example: keep average CPU at 60%
   → Pro: fully automated, adapts automatically
   → Con: less control over scaling increments

2. Step Scaling:
   → Different step adjustments based on alarm breach magnitude
   → CPU 70-80%: add 1 instance
   → CPU 80-90%: add 3 instances
   → CPU > 90%: add 5 instances
   → Pro: fine-grained control
   → Con: more configuration required

3. Simple Scaling:
   → Alarm fires → add N instances → wait cooldown period → check again
   → Legacy — use Step or Target Tracking instead
   → Con: cooldown period wastes time during rapid traffic growth

4. Scheduled Scaling:
   → Set min/max/desired at specific times
   → "Every Friday 5pm: set min=10 (weekend traffic)"
   → "Every Monday 8am: set min=5"
   → Pro: predictable traffic patterns
   → Con: doesn't adapt to unexpected traffic
```

> 💡 **Interview tip:** Always combine **Scheduled Scaling** with **Target Tracking** for best results. Scheduled Scaling pre-warms capacity before known traffic peaks (morning ramp-up, lunch rush, weekends). Target Tracking handles unexpected peaks beyond the pre-warmed capacity. Using only Target Tracking means the cluster starts scaling AFTER the traffic spike hits — by then, existing instances may already be overloaded. Pre-warming eliminates this lag.

---

### Q642 — AWS | Conceptual | Intermediate

> Explain **AWS ECS (Elastic Container Service)** in depth — what is the difference between **EC2 launch type**, **Fargate launch type**, and **ECS on Outposts**?
>
> What is a **Task Definition** vs a **Service** vs a **Task**? How does ECS service discovery work with **AWS Cloud Map**?

📁 **Reference:** `nawab312/AWS` — ECS, Fargate, task definitions sections

#### Key Points to Cover:

**EC2 vs Fargate:**
```
EC2 launch type:
  → You manage the EC2 instances (patching, scaling, capacity)
  → Containers run on your instances
  → More control over instance type, placement, networking
  → Can use Spot instances (significant cost savings)
  → Better for: high throughput, custom AMIs, GPU workloads

Fargate:
  → Serverless containers — AWS manages the infrastructure
  → No EC2 instances to manage
  → Pay per vCPU + memory per second
  → Slightly higher cost per unit than EC2
  → Slower startup (5-30s vs 1-5s for EC2)
  → Better for: variable workloads, simpler operations, less ops overhead

Cost comparison:
  Fargate: ~$0.04/vCPU/hour, ~$0.004/GB-memory/hour
  EC2: depends on instance type + Spot usage
  At scale: EC2 with Spot is cheaper; small teams: Fargate simpler
```

**Task Definition vs Service vs Task:**
```
Task Definition:
  → Blueprint/template for how to run containers
  → Defines: image, CPU, memory, ports, env vars, volumes, IAM role
  → Versioned (family:revision — e.g., payment-service:7)
  → Immutable once created

Task:
  → ONE running instance of a Task Definition
  → Like a Pod in Kubernetes
  → Can be standalone (one-off run) or managed by a Service

Service:
  → Manages running N tasks from a Task Definition
  → Maintains desired count (like a Deployment in Kubernetes)
  → Handles rolling updates, health checks, load balancer registration
  → Like a Kubernetes Deployment + Service combined
```

**ECS Service Discovery with Cloud Map:**
```
Problem: Service A needs to call Service B
         Service B's tasks have dynamic IPs (change on restart)

Cloud Map solution:
  → ECS registers each task in Cloud Map (DNS service)
  → Service B registered as: payment-service.production.local
  → DNS returns current task IPs
  → Service A calls: http://payment-service.production.local:8080

Configuration in ECS Service:
  serviceRegistries:
  - registryArn: arn:aws:servicediscovery:...:service/srv-xyz
    port: 8080
    containerName: payment-app
    containerPort: 8080
```

> 💡 **Interview tip:** The key ECS concept that trips up Kubernetes engineers: **ECS has no equivalent of Kubernetes Namespace-level isolation**. All ECS services in the same cluster can potentially communicate. Security boundaries are enforced via Security Groups on the tasks (awsvpc mode gives each task its own ENI) and IAM Task Roles (each task can have different AWS permissions). This is different from Kubernetes where NetworkPolicy provides in-cluster pod isolation.

---

### Q643 — AWS | Conceptual | Intermediate

> Explain **AWS S3** advanced features — what is **S3 versioning**, **S3 replication**, **S3 event notifications**, and **S3 presigned URLs**?
>
> What is the S3 **request rate limit** and how do you work around it for high-throughput applications?

📁 **Reference:** `nawab312/AWS` — S3 advanced features sections

#### Key Points to Cover:

**S3 Versioning:**
```
Versioning: keep ALL versions of every object
  → Delete = add delete marker (not actually deleted)
  → Can restore previous version by specifying version ID
  → Protects against: accidental delete, accidental overwrite
  → Increases storage cost (all versions stored)

Enabling:
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled

MFA Delete: require MFA to delete versions or disable versioning
  → Extra protection for compliance (WORM-like)
```

**S3 Replication:**
```
Cross-Region Replication (CRR):
  → Copy objects to bucket in DIFFERENT region
  → Use: DR, latency reduction for global users, compliance
  → Requires: versioning enabled on source AND destination

Same-Region Replication (SRR):
  → Copy objects to bucket in SAME region
  → Use: log aggregation from multiple accounts, testing

Important:
  → Only NEW objects after replication enabled (not existing)
  → For existing objects: use S3 Batch Replication
  → Replicated objects have SAME storage class unless overridden
```

**S3 Event Notifications:**
```
Trigger: object created, deleted, restored from Glacier, replicated
Destination: SQS, SNS, Lambda, EventBridge

Example: image upload → Lambda → resize → store thumbnail
aws s3api put-bucket-notification-configuration \
  --bucket upload-bucket \
  --notification-configuration '{
    "LambdaFunctionConfigurations": [{
      "LambdaFunctionArn": "arn:aws:lambda:...:function:resize",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {"FilterRules": [{"Name": "suffix", "Value": ".jpg"}]}
      }
    }]
  }'
```

**S3 Presigned URLs:**
```
Presigned URL: time-limited URL granting temporary access to private object

Use cases:
  → Allow user to download private file without making bucket public
  → Allow direct upload from browser (avoids proxying through server)
  → Share file for limited time (expires after N hours)

Generate:
aws s3 presign s3://my-bucket/report.pdf --expires-in 3600

Or via SDK:
url = s3_client.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'report.pdf'},
    ExpiresIn=3600
)
```

**Request rate limits:**
```
S3 can handle:
  5,500 GET/HEAD requests/second per prefix
  3,500 PUT/DELETE requests/second per prefix

"prefix" = the part of the key before the last /
  s3://bucket/2024/01/15/file.jpg → prefix = 2024/01/15/

High throughput tip:
  → Use multiple prefixes to distribute load
  → DO NOT use sequential keys (date-based) for high-write workloads
  → Randomize key prefix: hash/uuid at beginning
     BAD:  2024-01-15-user-001.jpg (sequential → same prefix)
     GOOD: a8f3/user-001-2024-01-15.jpg (random → distributed)
```

> 💡 **Interview tip:** **S3 Transfer Acceleration** and **Multipart Upload** are the two performance features most commonly missed. Transfer Acceleration routes uploads through CloudFront edge locations (faster for cross-continental uploads). Multipart Upload is mandatory for files > 5GB (S3 single PUT limit) and recommended for files > 100MB (parallel parts = faster upload, resume on failure). Always use multipart upload for large objects in production.

---

### Q644 — AWS | Conceptual | Intermediate

> Explain **AWS IAM cross-account access** — how does a user or service in Account A access resources in Account B?
>
> What is the **`sts:AssumeRole`** flow? Write the trust policy and permission policy required for cross-account EC2 access to S3. What is the difference between **resource-based policies** and **identity-based policies** for cross-account?

📁 **Reference:** `nawab312/AWS` — IAM cross-account, STS, trust policies sections

#### Key Points to Cover:

**Cross-account assume role flow:**
```
Account A (source):           Account B (target):
  EC2 instance                  S3 bucket
  IAM role: "ec2-role"          IAM role: "cross-account-s3-role"
  
Flow:
1. EC2 calls STS: AssumeRole → "cross-account-s3-role" ARN in Account B
2. STS checks:
   a. Does "ec2-role" have permission to call sts:AssumeRole? (identity policy in A)
   b. Does "cross-account-s3-role" trust Account A? (trust policy in B)
3. If both yes: STS returns temporary credentials (15min-1hour)
4. EC2 uses temp credentials to access S3 in Account B
```

**Trust policy (in Account B — who can assume this role):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::111111111111:role/ec2-role"  // Account A role
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "unique-external-id-12345"   // prevents confused deputy
      }
    }
  }]
}
```

**Permission policy (in Account B — what this role can do):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::account-b-bucket/*"
  }]
}
```

**Identity policy in Account A (ec2-role can assume cross-account role):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::222222222222:role/cross-account-s3-role"
  }]
}
```

**Identity-based vs Resource-based for cross-account:**
```
Cross-account via assume role:
  → BOTH sides must allow: identity policy in source + trust policy in target
  → Temporary credentials, audited in CloudTrail

Cross-account via resource-based policy (S3, SQS, etc.):
  → Only the resource policy needed (bucket policy can grant access)
  → No assume role needed
  → Example: S3 bucket policy allowing another account's root or role
  → Simpler but less audited, less granular
```

> 💡 **Interview tip:** The **ExternalId** in trust policies prevents the **confused deputy problem** — if Account A's role can assume ANY cross-account role that trusts Account A, a malicious actor could trick Account A into assuming a role in their account. ExternalId is a secret shared between the two accounts, making it impossible for a third party to exploit. This is mandatory for SaaS products that integrate with customer AWS accounts.

---

### Q645 — AWS | Conceptual | Intermediate

> Explain **AWS DynamoDB** capacity modes — **On-Demand** vs **Provisioned** capacity. What is **auto-scaling** for DynamoDB?
>
> What is a **DynamoDB Global Table**? What are **DynamoDB Streams** and what can you do with them?

📁 **Reference:** `nawab312/AWS` — DynamoDB capacity, streams, global tables sections

#### Key Points to Cover:

**On-Demand vs Provisioned:**
```
On-Demand:
  → No capacity planning — scales instantly to any throughput
  → Pay per request: ~$1.25/million writes, ~$0.25/million reads
  → Best for: unpredictable traffic, new tables, dev/staging
  → More expensive at consistent high throughput

Provisioned:
  → Set Read Capacity Units (RCU) and Write Capacity Units (WCU)
  → 1 RCU = 1 strongly consistent read/second for up to 4KB
  → 1 WCU = 1 write/second for up to 1KB
  → Pay per hour for provisioned capacity (whether used or not)
  → Best for: predictable traffic, cost optimization at scale

Auto-scaling (Provisioned):
  → Set target utilization (e.g., 70%)
  → DynamoDB auto-adjusts RCU/WCU based on actual usage
  → Scale up: fast (seconds)
  → Scale down: slow (minutes, only if utilization low for sustained period)
  → Best of both: no over-provisioning, handles spikes

Decision:
  Steady high traffic → Provisioned + Auto-scaling (cheaper)
  Unpredictable traffic → On-Demand (simpler, no throttling)
```

**DynamoDB Streams:**
```
Stream: ordered sequence of item-level changes (insert/update/delete)
  → Real-time (milliseconds after change)
  → 24-hour retention
  → Exactly-once delivery to stream consumers

What you can build:
  → Audit trail: every change recorded
  → Event sourcing: stream → Lambda → downstream systems
  → Cache invalidation: item changed → Lambda → invalidate Redis
  → Cross-region replication: stream → Lambda → write to other region
  → Global Tables uses streams internally for replication

Views available:
  KEYS_ONLY:     only key attributes
  NEW_IMAGE:     only new item state
  OLD_IMAGE:     only previous item state
  NEW_AND_OLD_IMAGES: both (most complete, most data)
```

**Global Tables:**
```
Multi-region, multi-active DynamoDB
  → Tables replicated across multiple regions
  → Reads and WRITES accepted in all regions
  → Conflict resolution: last-writer-wins (by timestamp)
  → ~1 second replication lag between regions

Use cases:
  → Global applications needing low write latency everywhere
  → Active-active multi-region DR
  → Data sovereignty: EU data written in EU, US in US

Requires:
  → DynamoDB Streams enabled
  → On-Demand capacity OR Provisioned with auto-scaling
```

> 💡 **Interview tip:** DynamoDB **hot partition** is the most common production performance problem. If your partition key has low cardinality (e.g., status = "pending"/"complete"), most traffic goes to 2 partitions while others sit idle. The solution is high-cardinality partition keys (user_id, order_id). If you need to query by a low-cardinality field (status), add a Global Secondary Index (GSI) with a better partition key, or use a "write sharding" pattern (status + random suffix as partition key).

---

### Q646 — AWS | Conceptual | Intermediate

> Explain **AWS CloudTrail** in depth — what does it log, what does it NOT log, and what is the difference between **Management events** and **Data events**?
>
> How do you use CloudTrail to investigate a security incident? What is **CloudTrail Lake** and how does it differ from standard CloudTrail?

📁 **Reference:** `nawab312/AWS` — CloudTrail, audit logging, security investigation sections

#### Key Points to Cover:

**What CloudTrail logs:**
```
CloudTrail records: API calls to AWS services

Management Events (default, free for first trail):
  → Creating, modifying, deleting AWS resources
  → Examples: EC2 RunInstances, S3 CreateBucket, IAM CreateUser
  → Who made the call, when, from where, with what parameters
  → Stored in S3

Data Events (extra cost, disabled by default):
  → Operations ON data: S3 object-level (GetObject, PutObject)
  → Lambda invocations
  → DynamoDB item-level operations
  → Very high volume (billions/day for busy accounts)
  → Must explicitly enable per resource or all resources

What CloudTrail does NOT log:
  → SSH/RDP sessions (use SSM Session Manager for audit)
  → Application-level logs (use CloudWatch Logs)
  → Queries to RDS/DynamoDB (use database audit logs)
  → Network traffic (use VPC Flow Logs)
```

**Security investigation with CloudTrail:**
```bash
# Find all actions by a specific IAM user in last 24 hours
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=johndoe \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ)

# Find all RunInstances (who launched EC2 instances)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --start-time 2024-01-15T00:00:00Z

# Query via Athena (for bulk analysis of S3-stored events):
SELECT userIdentity.userName, eventName, sourceIPAddress, eventTime
FROM cloudtrail_logs
WHERE eventTime > '2024-01-15'
  AND errorCode IS NOT NULL  -- find failed API calls
ORDER BY eventTime DESC
LIMIT 100;
```

**CloudTrail Lake:**
```
Standard CloudTrail: stores events in S3, query via Athena
CloudTrail Lake: managed event data store with built-in SQL queries
  → 7 year retention (vs 90 days for standard)
  → No S3 or Athena setup needed
  → Direct SQL queries in console
  → Can query across multiple accounts and regions from one place
  → Higher cost but much simpler
  → Best for: compliance, long-term audit retention, security investigations
```

> 💡 **Interview tip:** CloudTrail has a **15-minute delivery delay** to S3 — it is NOT real-time. For real-time alerting on suspicious activity (like root account usage), use CloudTrail → EventBridge rule → Lambda. EventBridge receives CloudTrail events in near-real-time (seconds) while S3 delivery takes 15 minutes. Also: the **root account login** should ALWAYS trigger a PagerDuty alert — there is almost no legitimate reason to log in as root in a well-governed AWS account.

---

### Q647 — AWS | Conceptual | Intermediate

> Explain **AWS EKS** (Elastic Kubernetes Service) — how does EKS differ from self-managed Kubernetes? What does AWS manage and what do YOU manage?
>
> What are **EKS Managed Node Groups** vs **Self-Managed Node Groups** vs **Fargate Profiles**? What is the EKS **cluster endpoint access** (public vs private)?

📁 **Reference:** `nawab312/AWS` — EKS, managed node groups sections

#### Key Points to Cover:

**EKS vs Self-Managed Kubernetes:**
```
Self-managed:
  YOU manage everything:
  → etcd (backup, HA, upgrades)
  → kube-apiserver (HA, certificates)
  → kube-scheduler, controller-manager
  → Worker node OS, kubelet
  → All upgrades (complex, risky)
  → Control plane HA (3 masters needed)

EKS:
  AWS manages:
  → Control plane (apiserver, etcd, scheduler, controller-manager)
  → Control plane HA (runs in 3 AZs automatically)
  → etcd backups
  → Control plane upgrades (one-click)
  → Control plane security patches

  YOU manage:
  → Worker nodes (unless using Fargate)
  → Node upgrades (worker node OS, kubelet version)
  → Add-ons (CoreDNS, kube-proxy, VPC CNI versions)
  → Your application workloads
  → Networking (VPC, subnets, security groups)
```

**3 Node options:**
```
Managed Node Groups:
  → AWS manages node provisioning and lifecycle
  → One-click node upgrades (rolling update)
  → Automatic node health checks and replacement
  → Limited: specific instance types, no custom launch scripts
  → Best for: standard workloads, teams wanting less ops

Self-Managed Node Groups:
  → You fully control the EC2 instances
  → Custom AMI, custom bootstrap scripts
  → Mixed instance types in one group
  → Must manage node upgrades yourself
  → Best for: GPU nodes, custom configurations, compliance requirements

Fargate Profiles:
  → No nodes at all (serverless Kubernetes)
  → AWS runs each pod on isolated compute
  → Select by namespace and labels
  → Cannot run DaemonSets (no node)
  → No privileged containers
  → Higher cost, simpler operations
  → Best for: highly variable workloads, strict isolation requirements
```

**Cluster endpoint access:**
```
Public endpoint (default):
  → kubectl from anywhere on internet
  → Convenient for developers
  → Risk: API server exposed to internet (rate-limited, but surface area)

Private endpoint:
  → kubectl only from within VPC
  → More secure (no internet exposure)
  → Developers need VPN or bastion to connect

Public + Private (recommended):
  → Allow specific CIDRs (office IPs, VPN) for public access
  → All cluster-internal communication uses private endpoint
  → privateAccess: true, publicAccess: true,
    publicAccessCidrs: ["203.0.113.0/24"]  # office CIDR only
```

> 💡 **Interview tip:** The most impactful EKS-specific issue new teams face is **the ENI limit determining pod density**. AWS VPC CNI gives each pod a real VPC IP (secondary IP on the node's ENI). Each EC2 instance type has a maximum number of ENIs × IPs per ENI. A `t3.medium` supports only 3 ENIs × 6 IPs = max 17 pods. With Karpenter and prefix delegation (`ENABLE_PREFIX_DELEGATION=true`), each ENI gets a /28 (16 IPs) instead of individual IPs, supporting up to 110 pods on larger instances.

---

### Q648 — AWS | Conceptual | Intermediate

> Explain **AWS VPC Security Groups vs Network ACLs (NACLs)** in depth — what is the fundamental difference in how they work?
>
> What is **stateful vs stateless** filtering? In what order are they evaluated? Give a scenario where you need BOTH — why would a Security Group alone not be sufficient?

📁 **Reference:** `nawab312/AWS` — VPC, Security Groups, NACLs sections

#### Key Points to Cover:

**Security Groups (SG):**
```
Attached to: EC2, RDS, Lambda, ELB, ECS tasks
Scope: instance-level firewall
Type: STATEFUL — tracks connection state

Stateful means:
  → Allow inbound HTTP (port 80) → RETURN traffic automatically allowed
  → No need to add outbound rule for established connections
  → Tracks connection state (5-tuple: src IP, src port, dst IP, dst port, protocol)

Rules:
  → Only ALLOW rules (no deny rules)
  → Whitelist approach: deny everything except what's allowed
  → Source/destination can be: IP CIDR, another Security Group
  → Referencing another SG: "allow all traffic from the ALB security group"
    → Auto-updates when instances are added to that SG

Evaluation:
  → All rules evaluated together (no order)
  → If ANY rule matches → allow
```

**Network ACLs (NACL):**
```
Attached to: subnet
Scope: subnet-level firewall
Type: STATELESS — does NOT track connection state

Stateless means:
  → Allow inbound port 80 → must ALSO allow outbound ephemeral ports (1024-65535)
  → Both directions must be explicitly configured
  → Higher management overhead

Rules:
  → ALLOW and DENY rules both supported
  → Rules have numbers (evaluated lowest first)
  → First matching rule applied (order matters!)
  → Default NACL: allow all inbound and outbound
  → Custom NACL: deny all by default
```

**Order of evaluation:**
```
Traffic flow (inbound to EC2):
  1. NACL inbound rules on the subnet evaluated
  2. If denied by NACL: dropped
  3. If allowed by NACL: proceeds to SG
  4. Security Group inbound rules evaluated
  5. If no matching allow rule: dropped
  6. Packet reaches EC2

Return traffic:
  1. SG: STATEFUL — return allowed automatically
  2. NACL: STATELESS — return traffic hits NACL outbound rules
             Must have outbound rule allowing ephemeral ports!
```

**When you need BOTH:**
```
Scenario: Block a specific IP that is attacking your application

Security Group: cannot DENY specific IPs (only allow rules)
  → Cannot block 198.51.100.0/24

NACL: CAN explicitly deny specific IP ranges
  → Add: Rule 100 DENY 198.51.100.0/24 ALL ports INBOUND
  → This blocks the attacker at subnet level
  → SG still protects other instances in other subnets

Together:
  NACL: coarse-grained subnet protection, deny specific IPs
  SG:   fine-grained per-instance control, allow specific ports/services
```

> 💡 **Interview tip:** The **ephemeral port** range is the most commonly forgotten NACL rule. When a client connects to your server (e.g., port 443), the return traffic goes back to the CLIENT's ephemeral port (random port in range 1024-65535). If your NACL doesn't have an outbound rule allowing ports 1024-65535 to the internet, clients receive no response even though they can connect. Security Groups handle this automatically (stateful). NACLs require explicit rules for both directions.

---

## SECTION 3 — LINUX REMAINING FUNDAMENTALS (Q649–Q655)

---

### Q649 — Linux | Conceptual | Beginner-Intermediate

> Explain the **Linux boot process** — what happens from pressing the power button to getting a login prompt?
>
> What is **BIOS vs UEFI**? What is **GRUB**? What is the difference between **init**, **upstart**, and **systemd**? What is **PID 1** and why is it special?

📁 **Reference:** `nawab312/DSA` → `Linux` — boot process sections

#### Key Points to Cover:

**Complete boot sequence:**
```
1. BIOS/UEFI:
   → Power on → CPU executes firmware from ROM
   → POST (Power-On Self Test): hardware check
   → Finds bootable device (disk, USB, network)
   → BIOS: MBR (Master Boot Record) — old, 2TB limit, 4 partitions
   → UEFI: GPT (GUID Partition Table) — modern, 9ZB limit, 128 partitions

2. Bootloader (GRUB2):
   → GRUB = GRand Unified Bootloader
   → Loads Linux kernel into memory
   → Passes kernel parameters (root device, init program, etc.)
   → Shows boot menu (can have multiple kernels)
   → Config: /etc/grub2.cfg

3. Kernel initialization:
   → Kernel decompresses itself into memory
   → Initializes CPU, memory management, device drivers
   → Mounts root filesystem (as read-only initially)
   → Starts PID 1 (init system)

4. Init system (PID 1):
   → FIRST user-space process
   → Starts all other processes
   → Old: SysV init (/etc/init.d scripts, sequential)
   → Upstart: event-driven, parallel (Ubuntu 2006-2015)
   → systemd: current standard, parallel, dependency-based

5. systemd startup:
   → Reads unit files (/etc/systemd/system/)
   → Starts services in parallel based on dependencies
   → Reaches target (multi-user.target, graphical.target)
   → Presents login prompt

6. Login:
   → Getty process presents login prompt
   → User enters credentials → PAM authenticates
   → Shell launched
```

**Why PID 1 is special:**
```
PID 1 is the init process:
  → Cannot be killed (SIGKILL to PID 1 = kernel panic)
  → Adopts orphaned processes (child processes whose parent died)
  → Must reap zombie processes (call wait() for dead children)
  → If PID 1 dies: system crashes

In containers:
  → Your app might become PID 1 (especially with exec form)
  → If app doesn't handle signals properly: docker stop waits 10s then SIGKILL
  → If app doesn't reap zombies: PID namespace fills up
  → Solution: use tini as PID 1 init in containers
```

> 💡 **Interview tip:** The **zombie process** question relates to PID 1 — a zombie (defunct) process is one that has exited but whose parent hasn't called `wait()` to collect its exit status. The process table entry stays until parent acknowledges. If PID 1 in a container (your app) doesn't reap zombies, they accumulate. `tini` solves this by being a proper init that reaps all orphaned processes. This is why official Docker images use tini and why Kubernetes uses a pause container as PID 1 in each Pod's network namespace.

---

### Q650 — Linux | Conceptual | Beginner-Intermediate

> Explain **Linux OSI model and networking stack** — how does a packet travel from an application sending data to bits on the wire?
>
> What happens at each layer: Application → Transport → Network → Data Link → Physical? Where do tools like `iptables`, `ss`, `tcpdump`, and `ping` operate?

📁 **Reference:** `nawab312/DSA` → `Linux` — networking, OSI model sections

#### Key Points to Cover:

**7 OSI layers (with TCP/IP mapping):**
```
Layer 7: Application    → HTTP, HTTPS, DNS, FTP, SMTP, SSH
Layer 6: Presentation   → TLS/SSL encryption, encoding
Layer 5: Session        → Session management (merged into L7 in TCP/IP)
Layer 4: Transport      → TCP, UDP — ports, reliability, flow control
Layer 3: Network        → IP — routing between networks
Layer 2: Data Link      → Ethernet — MAC addresses, frames
Layer 1: Physical       → Cables, signals, bits

TCP/IP Model (practical):
  Application (L7+6+5)
  Transport   (L4)
  Internet    (L3)
  Link        (L2+1)
```

**Packet journey (sending HTTP request):**
```
Application (your code):
  → http.get("https://api.example.com/data")
  → Socket API: data → OS kernel

Transport Layer (TCP):
  → Segment: adds TCP header (src port 45123, dst port 443)
  → Sequence numbers, acknowledgment, flow control
  → If TCP: establishes connection first (3-way handshake)

Network Layer (IP):
  → Packet: adds IP header (src 10.0.1.5, dst 93.184.216.34)
  → Routing: which interface to send out? (check routing table)
  → If dst not local: send to default gateway

Data Link (Ethernet):
  → Frame: adds Ethernet header (src MAC, dst MAC)
  → ARP: who has 10.0.1.1? → gateway MAC address
  → CRC checksum added

Physical:
  → Frame converted to electrical signals / optical pulses
  → Transmitted on wire / fiber / WiFi
```

**Where tools operate:**
```
Layer 7: curl, wget, dig, nslookup (application protocols)
Layer 4: ss, netstat (TCP/UDP connections and ports)
Layer 3: ping (ICMP), traceroute, ip route, iptables
Layer 2: arp, ip link, tcpdump (captures L2+)
Layer 1: ethtool (NIC statistics, link speed)

iptables operates at: Layer 3/4 (IP + TCP/UDP)
  → Can filter by IP, port, protocol
  → Cannot filter by hostname (must resolve to IP first)

tcpdump operates at: Layer 2 (raw frames)
  → Captures everything below TLS encryption
  → For HTTPS: captures encrypted payload (not plaintext)
```

> 💡 **Interview tip:** The most practical OSI interview answer demonstrates how troubleshooting maps to layers: (1) `ping` works = Layer 3 OK, (2) `telnet host port` works = Layer 4 OK, (3) `curl` fails = Layer 7 issue (HTTP/TLS problem). This "layer-by-layer elimination" approach shows you understand networking systematically rather than just running random commands. Also: DNS is Layer 7 but affects Layer 3 — DNS failure makes Layer 3 routing fail because you can't resolve the IP.

---

### Q651 — Linux | Conceptual | Beginner-Intermediate

> Explain **Linux file system hierarchy** — what is in `/etc`, `/var`, `/tmp`, `/usr`, `/home`, `/proc`, `/sys`, `/dev`, `/opt`, `/mnt`?
>
> What is the difference between **hard links** and **symbolic links**? What is an **inode**? What happens when you delete a file that is still open by a running process?

📁 **Reference:** `nawab312/DSA` → `Linux` — filesystem, inodes sections

#### Key Points to Cover:

**Key directories:**
```
/etc        → System-wide configuration files
              /etc/passwd (users), /etc/hosts (DNS overrides),
              /etc/fstab (filesystems), /etc/systemd/

/var        → Variable data that changes at runtime
              /var/log (logs), /var/lib (app data),
              /var/spool (print/mail queues), /var/tmp (temp survives reboot)

/tmp        → Temporary files, cleared on reboot
              World-writable (anyone can create files here)

/usr        → User programs and data (read-only in modern systems)
              /usr/bin (commands), /usr/lib (libraries),
              /usr/local (locally installed software)

/proc       → Virtual filesystem — live kernel/process state
              /proc/<PID>/ (per-process info), /proc/meminfo

/sys        → Virtual filesystem — hardware/kernel configuration
              /sys/class/net/ (network interfaces),
              /sys/block/ (disk devices)

/dev        → Device files (hard drives, terminals, random)
              /dev/sda (first disk), /dev/null (discard output),
              /dev/random, /dev/zero

/opt        → Optional/third-party software (manual installs)
              e.g., /opt/myapp/, /opt/google/chrome/

/mnt        → Temporary mount points
/media      → Removable media (USB drives, CD-ROM)
```

**Inodes and links:**
```
Inode: metadata about a file (NOT the filename)
  → File type, permissions, owner, timestamps
  → Pointers to data blocks on disk
  → Does NOT contain the filename

Filename → inode number → inode → data blocks
  (directory stores: filename + inode number pairs)

Hard link:
  → Second directory entry pointing to SAME inode
  → Same file, different names (possibly different directories)
  → Delete one: other still works (inode not freed until count = 0)
  → Cannot span filesystems (inodes are per-filesystem)
  → Cannot link directories (prevents loops)
  ln /etc/passwd /tmp/passwd-backup   # hard link

Symbolic link (symlink):
  → Special file containing PATH to another file
  → If target deleted: symlink is broken (dangling symlink)
  → Can span filesystems
  → Can link directories
  → Has its own inode (with a different inode number)
  ln -s /etc/passwd /tmp/passwd-link  # symlink
```

**File deleted but still open:**
```
process A opens /var/log/app.log (inode 12345)
rm /var/log/app.log
  → directory entry removed (filename gone)
  → but inode 12345 still exists! (reference count = 1, still open by process A)
  → file STILL CONSUMING DISK SPACE (data blocks not freed)
  → process A can still write to it

Why this matters:
  df shows disk full but ls shows nothing in /var/log
  → A process has a deleted log file still open and growing
  → Common cause: log rotation ran but process still writing to old FD

Fix:
  lsof | grep deleted    # find processes with deleted open files
  kill -HUP <pid>        # signal log rotation (process reopens log file)
  # OR restart the process
```

> 💡 **Interview tip:** **Disk full with no files showing** is a classic production troubleshooting scenario that tests inode knowledge. The command `lsof | grep "(deleted)"` reveals the culprit — a process still writing to a deleted log file. The disk space won't be freed until the process closes the file (or is restarted). This is also why log rotation tools send SIGHUP or a custom signal after rotating — to tell the application to close the old log file handle and open the new one.

---

### Q652 — Linux | Conceptual | Beginner-Intermediate

> Explain **Linux systemd** in depth — what is a unit file, what are the main unit types, and what are the most important directives?
>
> How do you create a custom systemd service? What is `systemctl` and what are the key commands? What is `journald` and how do you query logs from it?

📁 **Reference:** `nawab312/DSA` → `Linux` — systemd, service management sections

#### Key Points to Cover:

**Unit types:**
```
.service  → A process/daemon (nginx.service, sshd.service)
.timer    → Scheduled execution (replaces cron for system services)
.socket   → Socket activation (start service when connection arrives)
.mount    → Mount points (managed by systemd)
.target   → Group of units (multi-user.target = runlevel 3)
.path     → Watch files/directories for changes
.slice    → cgroup hierarchy for resource management
```

**Creating a custom service:**
```ini
# /etc/systemd/system/payment-service.service
[Unit]
Description=Payment Microservice
After=network.target postgresql.service    # start after these
Requires=postgresql.service               # fail if postgresql not running
Documentation=https://docs.company.com

[Service]
Type=simple                               # process stays in foreground
User=appuser                              # run as this user
Group=appgroup
WorkingDirectory=/opt/payment-service
ExecStart=/opt/payment-service/bin/server --config /etc/payment/config.yaml
ExecStop=/bin/kill -TERM $MAINPID
ExecReload=/bin/kill -HUP $MAINPID
Restart=always                            # restart if crashes
RestartSec=5s                             # wait 5s before restart
StandardOutput=journal                    # log stdout to journald
StandardError=journal
SyslogIdentifier=payment-service

# Resource limits
LimitNOFILE=65535                         # max open file descriptors
MemoryLimit=512M
CPUQuota=50%

# Security hardening
NoNewPrivileges=true                      # no privilege escalation
PrivateTmp=true                           # isolated /tmp
ReadOnlyPaths=/etc                        # /etc is read-only

[Install]
WantedBy=multi-user.target               # start in multi-user mode
```

**Key systemctl commands:**
```bash
systemctl start payment-service      # start now
systemctl stop payment-service       # stop
systemctl restart payment-service    # stop then start
systemctl reload payment-service     # reload config (SIGHUP)
systemctl status payment-service     # show status, last log lines
systemctl enable payment-service     # start on boot
systemctl disable payment-service    # don't start on boot
systemctl daemon-reload              # reload unit files (after editing)
systemctl list-units --type=service  # list all services
systemctl list-units --failed        # show failed units
```

**journald log querying:**
```bash
journalctl -u payment-service            # all logs for service
journalctl -u payment-service -f         # follow (like tail -f)
journalctl -u payment-service --since "1 hour ago"
journalctl -u payment-service --since "2024-01-15 10:00" --until "2024-01-15 11:00"
journalctl -p err -u payment-service     # only error level and above
journalctl --disk-usage                  # how much space logs use
journalctl --vacuum-size=500M            # delete old logs to stay under 500M
journalctl -b                            # logs from current boot
journalctl -b -1                         # logs from previous boot
```

> 💡 **Interview tip:** `systemctl daemon-reload` is one of the most commonly forgotten commands — after editing a `.service` file, systemd doesn't know about the changes until you run `daemon-reload`. Forgetting this leads to confusion when `systemctl restart` seems to use old settings. Also: `Type=simple` vs `Type=forking` matters — use `simple` for modern applications that stay in the foreground (Node.js, Python), use `forking` only for old-school daemons that fork and exit the parent process (some legacy Apache configs).

---

### Q653 — Linux | Conceptual | Beginner-Intermediate

> Explain **Linux disk management** — what is the difference between **`df`** and **`du`**? What is `lsblk`, `fdisk`, `parted`, and `mount`?
>
> What is **LVM (Logical Volume Manager)** and what problem does it solve? How do you extend a filesystem online (without downtime)?

📁 **Reference:** `nawab312/DSA` → `Linux` — disk management, LVM sections

#### Key Points to Cover:

**df vs du:**
```
df (disk free):
  → Shows filesystem-level usage
  → Reads from filesystem metadata (fast)
  → Shows: total/used/available per MOUNTED FILESYSTEM
  df -h                     # human readable
  df -h /var/log            # specific mount point
  df -i                     # inode usage (full inodes = no new files)

du (disk usage):
  → Walks directory tree and sums file sizes
  → Can be slow on large directories
  → Shows: how much space a DIRECTORY/FILE uses
  du -sh /var/log           # total size of /var/log
  du -sh /var/log/*         # size of each subdirectory
  du -sh * | sort -rh | head -10  # top 10 largest items

Discrepancy between df and du:
  → df shows 90% full but du shows only 50% used
  → Cause: deleted files still open by processes (see Q651)
  → Fix: find with lsof | grep deleted
```

**Disk management commands:**
```bash
lsblk                         # list block devices (disks, partitions, LVM)
lsblk -f                      # show filesystem type and UUID

fdisk /dev/sdb                # partition a disk (MBR, <2TB)
parted /dev/sdb               # partition a disk (GPT, >2TB, preferred)

# Create filesystem
mkfs.ext4 /dev/sdb1           # format as ext4
mkfs.xfs /dev/sdb1            # format as XFS (default on RHEL/CentOS)

# Mount
mount /dev/sdb1 /mnt/data     # temporary mount
# Permanent (survives reboot):
echo "/dev/sdb1 /mnt/data ext4 defaults 0 2" >> /etc/fstab
mount -a                      # mount all entries in fstab
```

**LVM — extending storage online:**
```
Problem without LVM:
  → Disk full → must resize partition → resize filesystem → DOWNTIME
  → Cannot span multiple physical disks with one filesystem

LVM solution:
  → Abstract physical disks into logical volumes
  → Resize logical volumes online (no unmounting needed for ext4/xfs)
  → Add new disk and extend existing volume group → extend logical volume

Concepts:
  PV (Physical Volume) → actual disk or partition (/dev/sdb)
  VG (Volume Group)    → pool of PVs
  LV (Logical Volume)  → "virtual partition" carved from VG

Extend filesystem online (no downtime):
# Step 1: Add new disk to system, identify it
lsblk                                   # see /dev/sdc

# Step 2: Create physical volume
pvcreate /dev/sdc

# Step 3: Add to existing volume group
vgextend ubuntu-vg /dev/sdc

# Step 4: Extend logical volume
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv

# Step 5: Resize filesystem (online, no unmount needed)
resize2fs /dev/ubuntu-vg/ubuntu-lv    # ext4
xfs_growfs /                          # xfs
```

> 💡 **Interview tip:** The **inode exhaustion** problem is tested frequently — you can have plenty of disk space but be unable to create new files because all inodes are used. `df -i` shows inode usage. This commonly happens with applications that create millions of tiny files (email systems, session storage, cache files). The fix is to use fewer, larger files or choose a filesystem with dynamic inode allocation (XFS allocates inodes dynamically, ext4 has fixed inode count at creation).

---

### Q654 — Linux | Conceptual | Beginner-Intermediate

> Explain **Linux networking commands** — what does each command do and when would you use it?
>
> Cover: `ip addr`, `ip route`, `ip link`, `netstat`/`ss`, `dig`/`nslookup`, `curl`, `wget`, `traceroute`/`mtr`, `iptables`, `nmap`.

📁 **Reference:** `nawab312/DSA` → `Linux` — networking commands sections

#### Key Points to Cover:

**Interface management:**
```bash
# ip addr — show/configure IP addresses
ip addr show                    # all interfaces
ip addr show eth0               # specific interface
ip addr add 10.0.1.100/24 dev eth0  # add IP
ip addr del 10.0.1.100/24 dev eth0  # remove IP

# ip link — manage network interfaces
ip link show                    # all interfaces + status
ip link set eth0 up             # bring up
ip link set eth0 down           # bring down
ip link show eth0 | grep "state" # check UP/DOWN state

# ip route — routing table
ip route show                   # show routes
ip route add 192.168.0.0/24 via 10.0.1.1   # add route
ip route del 192.168.0.0/24                 # remove route
ip route get 8.8.8.8            # which route used for this destination?
```

**Connection inspection:**
```bash
# ss (modern netstat)
ss -tlnp    # TCP listening ports with process names
ss -tn state established  # all established TCP connections
ss -s       # connection statistics summary

# netstat (legacy, but still common)
netstat -tlnp   # TCP listening ports
netstat -an | grep :8080    # connections on port 8080
```

**DNS tools:**
```bash
# dig — detailed DNS queries (preferred for debugging)
dig google.com              # A record lookup
dig google.com MX           # MX records
dig google.com @8.8.8.8     # query specific nameserver
dig +trace google.com       # trace full DNS resolution
dig -x 8.8.8.8              # reverse DNS lookup

# nslookup — simpler DNS lookup
nslookup google.com
nslookup google.com 8.8.8.8
```

**Connectivity testing:**
```bash
# ping — ICMP reachability test (Layer 3)
ping -c 4 google.com        # 4 packets only
ping -i 0.2 google.com      # fast ping (200ms interval)

# traceroute / mtr
traceroute google.com       # show hop path
mtr google.com              # continuous traceroute with packet loss per hop
mtr --report google.com     # generate report

# curl — HTTP testing
curl -v https://api.example.com          # verbose (shows headers)
curl -s -o /dev/null -w "%{http_code}" https://api.example.com  # just status code
curl -k https://self-signed.example.com  # ignore TLS cert errors
curl -H "Authorization: Bearer token" https://api.example.com/data
curl --max-time 5 https://api.example.com  # 5 second timeout

# telnet/nc — TCP port testing
nc -zv hostname 443          # test if port 443 is open
nc -l 8080                   # listen on port 8080 (simple server)
```

**iptables (basic):**
```bash
iptables -L -n -v           # list all rules with packet counts
iptables -L INPUT -n        # show INPUT chain
iptables -A INPUT -p tcp --dport 22 -j ACCEPT   # allow SSH
iptables -A INPUT -s 10.0.0.0/8 -j DROP         # block IP range
iptables-save > /etc/iptables/rules.v4           # save rules
```

> 💡 **Interview tip:** `ip route get <destination>` is one of the most useful but least known networking commands — it shows exactly which route would be used and which interface for a specific destination. This instantly tells you if traffic to a particular host would go through your VPN, through the default gateway, or through a specific interface. Much faster than reading the full routing table and figuring it out manually.

---

### Q655 — Linux | Conceptual | Beginner-Intermediate

> Explain **Linux user management** — what is the difference between **`useradd`** and **`adduser`**? What is **`/etc/passwd`**, **`/etc/shadow`**, and **`/etc/group`**?
>
> What is **sudo** and how does `/etc/sudoers` work? What is the difference between `su` and `sudo`? What are **PAM (Pluggable Authentication Modules)**?

📁 **Reference:** `nawab312/DSA` → `Linux` — user management, security sections

#### Key Points to Cover:

**User management:**
```bash
# useradd vs adduser
useradd appuser            # low-level, creates user with minimal defaults
                           # no home dir by default, no password set
adduser appuser            # high-level wrapper (Debian/Ubuntu)
                           # interactive, creates home dir, sets password

# Create system user for a service (no login shell, no home)
useradd -r -s /bin/false -M appservice
  -r: system account (UID < 1000)
  -s /bin/false: no shell (cannot login)
  -M: no home directory

# User commands
passwd appuser             # set password
usermod -aG docker appuser # add to docker group
usermod -s /bin/bash appuser  # change shell
userdel -r appuser         # delete user and home directory
id appuser                 # show UID, GID, groups
```

**Key files:**
```
/etc/passwd: one line per user
  username:x:UID:GID:comment:home:shell
  appuser:x:1001:1001:App User:/home/appuser:/bin/bash
  "x" = password in /etc/shadow (was plaintext historically)

/etc/shadow: hashed passwords (readable only by root)
  username:$hash:lastchange:min:max:warn:inactive:expire:
  appuser:$6$salt$hashedpassword:19500:0:99999:7:::

/etc/group: group memberships
  groupname:x:GID:member1,member2
  docker:x:999:ubuntu,siddharth
```

**sudo vs su:**
```
su (substitute user):
  su -           # become root (need root PASSWORD)
  su - appuser   # become appuser (need appuser's password)
  Creates new shell as that user

sudo (superuser do):
  sudo command        # run one command as root (need YOUR password)
  sudo -u appuser cmd # run as different user
  sudo -i             # interactive root shell
  Does NOT need root's password — uses YOUR password + sudoers policy
  Every command logged (audit trail)
  Better than su: granular permissions, audit trail
```

**`/etc/sudoers` configuration:**
```
# Edit ONLY via: visudo (validates syntax before saving)

# Allow user full sudo
siddharth ALL=(ALL:ALL) ALL
# username host=(run_as_user:run_as_group) commands

# Allow group to use sudo
%developers ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart payment-service
# group     host=(ALL) no-password: specific command only

# Allow without password for specific commands
siddharth ALL=(ALL) NOPASSWD: /usr/bin/kubectl, /usr/bin/helm

# Include directory of separate sudoers files (preferred)
#includedir /etc/sudoers.d/
```

**PAM (Pluggable Authentication Modules):**
```
PAM = framework that separates authentication from applications
Applications use PAM API → PAM modules do the actual authentication

Common PAM modules:
  pam_unix.so     → traditional /etc/shadow authentication
  pam_ldap.so     → LDAP/Active Directory authentication
  pam_google_auth → Google Authenticator 2FA
  pam_limits.so   → enforce ulimits on login

Config: /etc/pam.d/<service>  (sshd, sudo, login)
```

> 💡 **Interview tip:** In production environments, **direct root login should be disabled** (`PermitRootLogin no` in `/etc/ssh/sshd_config`). All privileged operations should go through `sudo` with specific command allowlists in `/etc/sudoers.d/`. This creates an audit trail (every sudo command logged to `/var/log/auth.log`) and enforces least privilege. NEVER put `NOPASSWD: ALL` in sudoers for real users — this gives passwordless root access, completely defeating the purpose of sudo.

---

## SECTION 4 — DOCKER REMAINING GAPS (Q656–Q659)

---

### Q656 — Docker | Conceptual | Beginner-Intermediate

> Explain **Docker architecture** — what are the **Docker daemon**, **Docker client**, **Docker registry**, and **containerd**?
>
> What is the difference between a **Docker image** and a **Docker container**? What is a **container** vs a **virtual machine** at the technical level?

📁 **Reference:** `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md`

#### Key Points to Cover:

**Docker architecture:**
```
Docker Client (docker CLI)
    ↓ REST API (Unix socket or TCP)
Docker Daemon (dockerd)
    ↓ gRPC
containerd (container runtime)
    ↓ OCI runtime spec
runc (creates actual containers)
    ↓
Linux kernel (namespaces + cgroups)

docker run nginx:
  1. docker CLI → REST API → dockerd
  2. dockerd checks: is nginx image local?
  3. If not: dockerd → registry (Docker Hub) → pull image
  4. dockerd → containerd: create container from image
  5. containerd → runc: create Linux namespaces + cgroups
  6. runc: executes nginx process in isolated environment
  7. Container running
```

**Image vs Container:**
```
Image:
  → Read-only template (like a class in OOP)
  → Layers stacked using Union filesystem (OverlayFS)
  → Stored in registry or local cache
  → Identified by name:tag or SHA256 digest
  → docker images → list local images

Container:
  → Running instance of an image (like an object in OOP)
  → Thin read-write layer on top of image layers
  → Has its own PID namespace, network namespace, mount namespace
  → Exists only while process runs (unless --restart flag)
  → docker ps → list running containers
  → docker ps -a → list all containers (including stopped)

Multiple containers from same image:
  → All share the read-only image layers (efficient storage)
  → Each has its own read-write layer
  → Changes in container do NOT affect image or other containers
```

**Container vs Virtual Machine:**
```
Virtual Machine:
  Physical Hardware
    → Hypervisor (VMware, KVM, Hyper-V)
    → VM 1: full OS kernel + userspace + app
    → VM 2: full OS kernel + userspace + app
  Each VM: 512MB-2GB overhead for OS alone
  Startup: minutes
  Strong isolation (separate kernel)
  Can run different OS (Windows VM on Linux host)

Container:
  Physical Hardware
    → Linux Kernel (shared)
    → Container 1: userspace + app (no kernel)
    → Container 2: userspace + app (no kernel)
  Each container: 10-100MB overhead (just the app + dependencies)
  Startup: milliseconds
  Weaker isolation (shared kernel — container escape possible)
  Must use same kernel type (Linux containers need Linux kernel)

Key technology:
  Namespaces: isolation (each container thinks it's alone)
    pid, net, mnt, uts, ipc, user namespaces
  cgroups: resource limits (CPU, memory, I/O per container)
```

> 💡 **Interview tip:** The **shared kernel** is the fundamental security difference between containers and VMs. A compromised VM cannot access the host or other VMs (different kernel). A compromised container that escapes its namespace restrictions CAN access the host (same kernel). This is why container security (non-root user, read-only filesystem, no privileged mode) matters — the blast radius of a container escape is much larger than a VM escape. For high-security workloads, gVisor (runsc) or Kata Containers provide VM-level isolation with container-like experience.

---

### Q657 — Docker | Conceptual | Beginner-Intermediate

> Walk through the key **Dockerfile instructions** — what does each one do and what are the common mistakes with each?
>
> What is the difference between **`ADD`** and **`COPY`**? What is the difference between **`CMD`** and **`ENTRYPOINT`** (again, but from a "which to use" perspective)? What are **build arguments** (`ARG`) vs **environment variables** (`ENV`)?

📁 **Reference:** `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md`

#### Key Points to Cover:

**All Dockerfile instructions:**
```dockerfile
# FROM — base image (always first instruction)
FROM node:20-alpine AS base
# Best practice: use specific version tags, not "latest"
# Use Alpine for smaller images

# RUN — execute command during BUILD (creates new layer)
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
# Chain with && to keep in one layer (reduces image size)
# Clean up caches in same RUN to avoid caching in layer

# COPY — copy files from build context into image
COPY package*.json ./           # copies package.json and package-lock.json
COPY src/ ./src/                # copies entire src directory
# Preferred over ADD for regular files

# ADD — like COPY but with extra features
ADD archive.tar.gz /app/        # auto-extracts archives
ADD https://example.com/file /  # can download from URL (use curl/wget instead)
# Avoid ADD for regular files — use COPY (more predictable)

# WORKDIR — set working directory (creates if doesn't exist)
WORKDIR /app                    # all subsequent commands run here
# Better than: RUN cd /app (cd doesn't persist between RUN instructions)

# ENV — set environment variables (available during build AND runtime)
ENV NODE_ENV=production
ENV PORT=3000
# These are baked into image — visible in docker inspect

# ARG — build-time variables (NOT available at runtime)
ARG BUILD_VERSION=1.0.0
RUN echo "Building version $BUILD_VERSION"
# docker build --build-arg BUILD_VERSION=2.0.0 .
# Useful for: version tags, API URLs specific to build environment

# EXPOSE — document which port the container listens on (informational only)
EXPOSE 3000
# Does NOT actually publish the port (use -p flag at runtime)
# Just documentation — but useful for tooling

# USER — switch to non-root user
USER node                       # run remaining commands as 'node' user
# Security: always run app as non-root

# VOLUME — declare mount points
VOLUME ["/data"]                # document that /data should be mounted
# Creates anonymous volume if not mounted at runtime

# ENTRYPOINT — main executable (hard to override)
ENTRYPOINT ["node"]             # node is always the executable
CMD ["server.js"]               # default argument to node

# CMD — default command (easy to override)
CMD ["node", "server.js"]       # override with: docker run myimage python app.py
```

**ARG vs ENV:**
```
ARG:                          ENV:
  Build time only               Build time + Runtime
  Not visible in running        Visible in: docker inspect,
    container                     ps aux, logs (risk)
  docker build --build-arg      docker run -e or in Dockerfile
  Good for: build configs,      Good for: app config, paths
    CI/CD version tags          Bad for: secrets (visible!)
  Not visible in docker inspect
```

> 💡 **Interview tip:** `COPY package*.json ./` before `COPY . .` and `RUN npm install` between them is the single most important caching optimization — but the `*` glob is often mistyped. `package*.json` matches both `package.json` and `package-lock.json`. Also: `WORKDIR` creates directories automatically — you don't need `RUN mkdir -p /app && cd /app`. And always use `WORKDIR` instead of `RUN cd` because `cd` doesn't persist across `RUN` instructions in separate shell invocations.

---

### Q658 — Docker | Conceptual | Intermediate

> Explain **Docker networking** in depth — when you run `docker network create mynetwork`, what actually happens at the Linux level?
>
> What is a **veth pair**? What is the **docker0 bridge**? How does port publishing (`-p 8080:80`) work at the iptables level?

📁 **Reference:** `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md`

#### Key Points to Cover:

**What happens when container starts:**
```
docker run --network mynet nginx

Linux kernel operations:
1. Docker creates network namespace for container
   → Isolated network stack (interfaces, routing table, iptables)

2. Creates veth pair (virtual ethernet cable)
   → eth0 inside container namespace
   → veth0abc on host (connected to bridge)

3. Creates/uses bridge (docker0 or custom)
   → Linux bridge = virtual network switch
   → All containers on same bridge can communicate

4. Assigns IP from bridge subnet
   → Container: 172.17.0.3/16
   → Gateway: 172.17.0.1 (the bridge itself)

5. Container's eth0 → veth0abc → docker0 bridge
   → Other containers on docker0: directly reachable
   → External traffic: via NAT through host's eth0
```

**How port publishing works (-p 8080:80):**
```
docker run -p 8080:80 nginx

Docker adds iptables rules:
  iptables -t nat -A DOCKER -p tcp --dport 8080 -j DNAT \
    --to-destination 172.17.0.2:80
  # DNAT: Destination NAT — rewrite destination before routing

Traffic flow:
  External → host:8080
  → iptables PREROUTING: DNAT → 172.17.0.2:80
  → Packet forwarded to container
  → Container receives request on port 80
  → Response: SNAT back to original source
  → External client sees response from host:8080

Check iptables rules Docker created:
  iptables -t nat -L -n
  iptables -L DOCKER -n
```

**Bridge vs Host networking:**
```
Bridge (default):
  Container: 172.17.0.2  (private IP)
  Host: 192.168.1.10     (real IP)
  Communication through NAT + bridge
  Port publishing needed to expose services
  Network isolation between containers on different bridges

Host:
  Container shares host's network stack directly
  Container sees all host interfaces
  Port 80 in container = port 80 on host (no mapping needed)
  No isolation — container can see all host ports
  Use case: monitoring agents (node-exporter), performance-critical apps
  Risk: port conflicts with host services
```

> 💡 **Interview tip:** The **DNS behavior difference** between default bridge and custom bridge networks is one of the most practical Docker networking facts: on the default `bridge` network, containers can only communicate via IP address. On a **custom named bridge network**, Docker's embedded DNS server allows communication by container name. This is why Docker Compose always creates a custom bridge network — so your services can call each other by service name (`db`, `redis`) instead of IP addresses that change on restart.

---

### Q659 — Docker | Conceptual | Intermediate

> What is a **Docker registry** — what is the difference between **Docker Hub**, **Amazon ECR**, **GitHub Container Registry**, and a **self-hosted registry**?
>
> What is an **OCI (Open Container Initiative)** image and what are the OCI standards? What is the difference between `docker pull` and `docker push`? What are **image digests** and why are they more reliable than tags?

📁 **Reference:** `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md`

#### Key Points to Cover:

**Registry types:**
```
Docker Hub (hub.docker.com):
  → Default public registry
  → Free: 1 private repo, public unlimited
  → Rate limits for unauthenticated pulls: 100/6h
  → Official images: nginx, postgres, node (trusted, maintained)
  → Use: public open source, development

Amazon ECR:
  → AWS-managed private registry
  → Integrated with IAM (no separate credentials needed from EC2/ECS/EKS)
  → Scanning with Inspector
  → Pull through cache (proxy Docker Hub)
  → No rate limits for same-region pulls from AWS compute
  → Use: production AWS workloads

GitHub Container Registry (ghcr.io):
  → Integrated with GitHub Actions workflows
  → Packages tab in GitHub repo
  → Use: open source projects, GitHub-centric workflows

Self-hosted (registry:2 or Harbor):
  → Full control over storage and access
  → Air-gapped environments
  → Harbor: enterprise registry (scanning, replication, RBAC)
```

**OCI Standards:**
```
OCI = Open Container Initiative (Linux Foundation)
Defines standards so containers are portable:

OCI Image Spec:
  → Standard format for container images
  → Any OCI-compliant runtime can run OCI images
  → Docker images ARE OCI images (Docker donated format)

OCI Runtime Spec:
  → Standard for how to run containers
  → runc, crun, gVisor (runsc) are all OCI runtimes
  → containerd calls OCI runtime to create containers

OCI Distribution Spec:
  → Standard for registries (push/pull API)
  → Any OCI-compliant registry works with any OCI client
```

**Image digests vs tags:**
```
Tag (mutable):
  docker pull nginx:1.25.3
  → "1.25.3" is a tag — can be overwritten by maintainer
  → docker pull nginx:1.25.3 today ≠ nginx:1.25.3 tomorrow (potentially)
  → Not reproducible

Digest (immutable):
  docker pull nginx@sha256:abc123...
  → SHA256 hash of the image manifest
  → Mathematically tied to exact image content
  → Always pulls SAME image regardless of when/where
  → Reproducible builds

Pinning by digest in Dockerfile:
  FROM nginx:1.25.3@sha256:abc123def456...
  → If maintainer changes "1.25.3" tag → you still get original image
  → Best for production (security, reproducibility)
  → Get digest: docker inspect nginx:1.25.3 --format '{{.RepoDigests}}'
```

> 💡 **Interview tip:** **Tag mutation** is a real production risk. If your Kubernetes deployment uses `image: nginx:1.25.3` and the maintainer republishes that tag (possible for non-official images), a new pod might pull a different image than existing pods — causing inconsistency in your cluster. The production solution is: (1) use immutable digests, (2) mirror images to your own ECR repo (you control the tag), (3) use ECR image immutability setting to prevent tag overwriting. At minimum, never use `latest` in production manifests.

---

## SECTION 5 — GIT REMAINING FUNDAMENTALS (Q660–Q663)

---

### Q660 — Git | Conceptual | Beginner-Intermediate

> Explain the **Git data model** — what is a **blob**, **tree**, **commit**, and **tag** object? How does Git store data internally?
>
> What is the **working directory**, **staging area (index)**, and **repository**? What is `HEAD` and what is a **detached HEAD** state?

📁 **Reference:** `nawab312/CI_CD` → `Git` — internals, data model sections

#### Key Points to Cover:

**Git object types:**
```
blob:    stores file content (not filename)
         every unique file content = one blob
         blob hash = SHA1(content)

tree:    stores directory listing
         maps filenames → blob hashes or other tree hashes
         represents one directory at one point in time

commit:  stores a snapshot of the project
         contains: tree hash, parent commit hash(es),
                   author, committer, timestamp, message
         commit hash = SHA1(tree + parent + metadata)

tag:     annotated pointer to a specific commit
         contains: target commit hash, tagger, message, signature
```

**3 areas:**
```
Working Directory:   actual files you see and edit
                     untracked or modified files here

Staging Area (Index): what will go into NEXT commit
                      git add → moves changes here
                      staging allows partial commits

Repository (.git/):  all committed history
                     git commit → moves staging to repository

Flow:
  Edit file → Working Directory (modified)
  git add file → Staging Area (staged)
  git commit → Repository (committed)
  git push → Remote Repository
```

**HEAD:**
```
HEAD = pointer to current commit (or branch)
  → Normally: HEAD → branch (main) → commit hash
  → HEAD is indirect: HEAD → refs/heads/main → abc1234

Detached HEAD:
  git checkout abc1234    # checkout specific commit (not a branch)
  HEAD → abc1234 directly (no branch pointer)

  Danger: commits in detached HEAD state are NOT on any branch
          running git checkout main abandons those commits
          (they become unreachable, garbage collected)

  Fix if you made commits in detached HEAD:
  git branch recovery-branch    # create branch at current position
  git checkout main             # go back to main
  git merge recovery-branch     # merge your work
```

> 💡 **Interview tip:** Understanding the **staging area** is what separates a Git novice from an intermediate user. The staging area lets you craft commits that are logical and focused — you can have 10 changed files but stage only 3 for a focused commit, stage only specific lines of a file (`git add -p`), or create multiple commits from one working directory state. This is the key to good commit history and meaningful code review. Many engineers just `git add .` everything — mention `git add -p` (patch mode) as the proper approach for clean commit history.

---

### Q661 — Git | Conceptual | Beginner-Intermediate

> Explain the difference between **`git fetch`**, **`git pull`**, and **`git push`**. What is an **upstream** branch?
>
> What is the difference between **`git merge`** and **`git rebase`**? What is a **fast-forward merge**? What is **`git cherry-pick`** and when would you use it?

📁 **Reference:** `nawab312/CI_CD` → `Git` — remote operations, merge strategies sections

#### Key Points to Cover:

**fetch vs pull:**
```
git fetch origin:
  → Downloads remote changes to local remote-tracking branches
  → Does NOT modify working directory or current branch
  → Safe: see what's changed without integrating
  → origin/main is updated, but your main is untouched

git pull origin main:
  → git fetch + git merge (or git rebase with --rebase flag)
  → Modifies your current branch
  → May create merge commit if branches diverged
  → git pull --rebase = fetch + rebase (cleaner history)

Best practice:
  git fetch origin         # update remote-tracking branches
  git log origin/main      # review what changed
  git merge origin/main    # merge when ready (explicit control)
  # vs blind: git pull (fetches and merges at once)
```

**git push:**
```
git push origin main      # push local main to remote main
git push origin feature   # push feature branch
git push -u origin feature # push + set upstream (future: just git push)
git push --force-with-lease # safe force push (checks for upstream changes first)
# NEVER git push --force on shared branches (rewrites remote history)
```

**merge vs rebase:**
```
Merge:
  Creates a merge commit
  Preserves complete history (who branched when)
  main:    A - B - C
  feature:         D - E
  result:  A - B - C - M (M = merge commit with parents C and E)
                   ↑
                   D - E

Rebase:
  Rewrites commits on top of another branch
  Linear history (cleaner, easier to bisect)
  main:    A - B - C
  feature: replayed as D' - E' on top of C
  result:  A - B - C - D' - E'

  NEVER rebase shared/public branches (rewrites history others have)
  USE for local feature branches before merging to main

Fast-forward merge:
  When branch has no divergence — just moves pointer forward
  main: A - B
  feature:     C - D (ahead of main)
  ff merge: main pointer just moves to D (no merge commit)
  git merge --no-ff feature  # force merge commit even for ff

Cherry-pick:
  Apply ONE specific commit to current branch
  Use cases:
    → Backport a bugfix from main to release/1.0 branch
    → Apply one commit without merging entire feature branch
  git cherry-pick abc1234   # apply commit abc1234 to current branch
  git cherry-pick main~2..main  # apply last 2 commits from main
```

> 💡 **Interview tip:** The rule for rebase vs merge: **private branches = rebase** (before merging to main, rebase onto latest main for clean linear history), **public/shared branches = merge** (never rewrite history that others have already cloned). A quick memory aid: "rebase before you share, merge after you share." Also: `git merge --no-ff` is the Git Flow way — it always creates a merge commit even for fast-forward merges, making it visible in history that a feature branch was merged.

---

### Q662 — Git | Conceptual | Beginner-Intermediate

> Explain **`git reset`**, **`git revert`**, and **`git stash`** — what does each do and when would you use each?
>
> What are the 3 modes of `git reset` — `--soft`, `--mixed`, `--hard`? What is the danger of `git reset --hard`?

📁 **Reference:** `nawab312/CI_CD` → `Git` — undoing changes sections

#### Key Points to Cover:

**git reset (moves HEAD backward):**
```
Moves current branch pointer to a previous commit

--soft:  moves HEAD, keeps staging area and working directory unchanged
         use: undo last commit but keep changes staged (ready to recommit)
         git reset --soft HEAD~1
         → commit is undone, files still staged

--mixed: moves HEAD, clears staging area, keeps working directory unchanged
(default) use: undo last commit and unstage changes (files still modified)
         git reset HEAD~1  (same as --mixed)
         → commit is undone, files are modified but not staged

--hard:  moves HEAD, clears staging area AND working directory
         DANGER: working directory changes are LOST (cannot recover without reflog)
         use: completely undo commit AND discard all changes
         git reset --hard HEAD~3  (undo 3 commits, discard all changes)

Memory aid:
  --soft:  only HEAD moved
  --mixed: HEAD + index cleared
  --hard:  HEAD + index + working directory wiped
```

**git revert (creates new commit that undoes):**
```
git revert abc1234   → creates new commit that reverses abc1234

vs reset:
  reset: rewrites history (removes commit from log)
  revert: adds new commit (history preserved)

Use revert for:
  → Shared/public branches (history preserved, safe for others)
  → When you need to undo a pushed commit
  → Audit trail (the revert is visible in history)

Use reset for:
  → Local branches before push
  → Cleaning up your own work before sharing
```

**git stash:**
```
Saves uncommitted changes (modified tracked files) temporarily
Lets you switch context without committing work-in-progress

git stash             # save working directory + index
git stash push -m "WIP: payment feature"  # with description
git stash list        # show all stashes
git stash pop         # apply most recent stash AND remove it
git stash apply       # apply most recent stash but KEEP it in stash
git stash drop        # delete most recent stash
git stash apply stash@{2}  # apply specific stash

Use cases:
  Working on feature → urgent hotfix needed → stash feature work
  → fix hotfix → stash pop → continue feature
  Experimental change → want to run tests on clean tree → stash → test → stash pop
```

> 💡 **Interview tip:** The most important safety tip about `git reset --hard`: **use `git stash` instead if you want to temporarily set aside changes but might want them back**. `git reset --hard` destroys uncommitted work permanently (unless `git reflog` saves you within 90 days). The safe default should be `git stash` for "put this aside" and `git reset --soft`/`--mixed` for "undo commits but keep work". Reserve `--hard` only for "I am absolutely certain I want to discard all of this forever."

---

### Q663 — Git | Conceptual | Intermediate

> Explain **Git branching strategies** — what are **GitFlow**, **GitHub Flow**, and **Trunk-Based Development**? What are the pros and cons of each?
>
> What is a **release branch**? What is a **hotfix**? How do you handle a **hotfix** in each branching strategy?

📁 **Reference:** `nawab312/CI_CD` → `Git` — branching strategies sections

#### Key Points to Cover:

**GitFlow:**
```
Branches: main, develop, feature/*, release/*, hotfix/*

Flow:
  feature branches → develop → release/* → main
  Parallel: main always = production, develop = integration

Pros:
  → Clear separation of stable (main) and unstable (develop)
  → Supports multiple versions in parallel
  → Release preparation has dedicated branch

Cons:
  → Complex, many branches to manage
  → Long-lived branches → merge conflicts
  → Slow release cycle (days/weeks per release)
  → Not suitable for continuous deployment

Hotfix:
  → Create hotfix/* from main
  → Fix the bug
  → Merge into BOTH main AND develop
  → Tag on main
```

**GitHub Flow:**
```
Branches: main + short-lived feature branches only

Flow:
  feature branch → PR → code review → merge to main → deploy

Pros:
  → Simple (only main + feature branches)
  → Fast releases (merge to main = deploy)
  → Works well for web applications

Cons:
  → No explicit release branch
  → Harder to support multiple versions
  → Requires good CI/CD and feature flags

Hotfix:
  → Create fix branch from main
  → Fix → PR → merge → deploy immediately
  → Same as any other change
```

**Trunk-Based Development:**
```
Branches: main only (or very short-lived < 1 day branches)

Flow:
  Commit directly to main (small teams)
  OR feature branches that live < 1 day
  Feature flags hide incomplete features in production

Pros:
  → No merge conflicts (no long-lived branches)
  → Continuous integration (always green main)
  → Fastest release cycle
  → Used by Google, Facebook, Netflix

Cons:
  → Requires: excellent test coverage, feature flags, discipline
  → Cultural shift for teams used to GitFlow
  → All code in main at all times (feature flags required)

Hotfix:
  → Just commit to main (it's a normal commit)
  → Deploy immediately (main is always deployable)
```

**Recommendation for Senior DevOps role:**
```
Small teams (<10 devs): GitHub Flow or Trunk-Based
  → Simple, fast, fits CI/CD

Large teams/enterprise: GitFlow or scaled Trunk-Based
  → Multiple supported versions
  → Regulated releases

Modern high-performing teams: Trunk-Based + Feature Flags
  → Deploy daily or multiple times/day
  → Zero long-lived branches = zero merge conflicts
```

> 💡 **Interview tip:** The **DORA research** (which you covered earlier) shows that elite performing teams (deploying multiple times per day with low failure rates) almost universally use trunk-based development. GitFlow is correlated with lower deployment frequency. The question "which branching strategy would you recommend?" should always reference DORA metrics — trunk-based development with feature flags is what enables elite performance. But also acknowledge that GitFlow makes sense for products with multiple supported versions (open-source libraries, enterprise software with long-term support releases).

---

## SECTION 6 — PROMETHEUS REMAINING (Q664–Q667)

---

### Q664 — Prometheus | Conceptual | Beginner-Intermediate

> Explain the **basic PromQL query structure** — what is an **instant vector**, a **range vector**, and a **scalar**?
>
> Write PromQL queries for these tasks: count all HTTP requests in last 5 minutes, get current memory usage per pod, find pods with CPU usage > 80%, and calculate error rate percentage.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`

#### Key Points to Cover:

**3 PromQL data types:**
```
Instant Vector:
  → One value per time series at the CURRENT moment
  → Result of: metric_name{labels}
  → Like a snapshot at "now"
  → Example: up → returns current up/down status of all targets

Range Vector:
  → Multiple values per time series over a TIME RANGE
  → Result of: metric_name{labels}[5m]
  → Needed for: rate(), increase(), avg_over_time()
  → Example: http_requests_total[5m] → last 5 minutes of data points

Scalar:
  → Single numeric value (no labels)
  → Result of: scalar(), mathematical operations
  → Example: scalar(sum(up)) → count of all healthy targets
```

**Essential PromQL queries:**
```promql
# 1. Count all HTTP requests per second over last 5 minutes
rate(http_requests_total[5m])

# 2. Total requests per service (summed across all pods)
sum by(service) (rate(http_requests_total[5m]))

# 3. Current memory usage per pod (MB)
container_memory_usage_bytes{container!="", container!="POD"}
/ 1024 / 1024

# 4. Memory usage as % of limit
container_memory_usage_bytes
/
container_spec_memory_limit_bytes * 100

# 5. CPU usage per pod > 80%
(
  sum by(pod, namespace) (
    rate(container_cpu_usage_seconds_total{container!=""}[5m])
  )
  /
  sum by(pod, namespace) (
    kube_pod_container_resource_limits{resource="cpu"}
  )
) * 100 > 80

# 6. Error rate % for a service
(
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
) * 100

# 7. Number of unhealthy instances
count(up == 0)

# 8. How many pods are not Running
kube_pod_status_phase{phase!="Running"} == 1

# 9. Disk space remaining % per node
(
  node_filesystem_avail_bytes{mountpoint="/"}
  /
  node_filesystem_size_bytes{mountpoint="/"}
) * 100

# 10. 95th percentile request latency
histogram_quantile(0.95, 
  sum by(le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

**Label selectors (matchers):**
```promql
up{job="payment-service"}          # exact match
up{job=~"payment.*"}               # regex match
up{job!="monitoring"}              # not equal
up{job!~"test.*"}                  # not regex match
```

> 💡 **Interview tip:** The most important PromQL concept for interviews is the difference between **instant vector** and **range vector**. A common mistake: `rate(http_requests_total)` — this is WRONG because `rate()` requires a range vector, not an instant vector. The correct form is `rate(http_requests_total[5m])`. If you see this error: `expected type range vector in call to function "rate", got instant vector` — you forgot the `[5m]` range selector.

---

### Q665 — Prometheus | Conceptual | Intermediate

> Explain **Prometheus AlertManager** — what is it, what is its role, and what are **grouping**, **inhibition**, and **silences**?
>
> Write an AlertManager config that groups alerts by cluster and namespace, routes critical alerts to PagerDuty and warnings to Slack, and inhibits warning alerts when a critical alert for the same service is already firing.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — AlertManager sections

#### Key Points to Cover:

**AlertManager role:**
```
Prometheus handles: metric collection, rule evaluation, firing alerts
AlertManager handles: routing, deduplication, grouping, silencing, inhibition

Why separate?
  → Decouple alert firing from notification logic
  → Multiple Prometheus instances can share one AlertManager
  → AlertManager provides HA (multiple instances, gossip protocol)
  → Notification logic is complex (routing trees, retries, dedup)

Flow:
  Prometheus (alert fires) → AlertManager → notifications (Slack/PagerDuty/email)
```

**AlertManager key features:**
```
Grouping:
  → Multiple alerts about same issue → ONE notification
  → Example: 50 pods failing → not 50 Slack messages → 1 grouped message
  → Group by: alertname, cluster, namespace, service

Deduplication:
  → Same alert firing from multiple Prometheus instances → sent only once
  → Uses: alertname + labels to deduplicate
  → Requires HA AlertManager setup (gossip protocol)

Inhibition:
  → If alert A is firing → suppress alert B
  → Example: "cluster down" inhibits all per-pod alerts
  → Reduces noise when root cause alert is already firing

Silences:
  → Manually suppress alerts for a time period
  → Use: scheduled maintenance, known flapping
  → Requires: creator name + justification (audit trail)
  → Expires automatically
```

**Complete AlertManager config:**
```yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/xxx'

route:
  group_by: ['cluster', 'namespace', 'alertname']  # group related alerts
  group_wait: 30s           # wait 30s for more alerts before first notification
  group_interval: 5m        # interval for new alerts in same group
  repeat_interval: 4h       # resend if still firing after 4 hours
  receiver: 'slack-warnings' # default receiver

  routes:
  # Critical → PagerDuty
  - matchers:
    - severity = critical
    receiver: pagerduty-critical
    repeat_interval: 1h      # page again if still critical after 1 hour

  # Warning → Slack
  - matchers:
    - severity = warning
    receiver: slack-warnings

inhibit_rules:
# If service is completely down (critical) → suppress individual component warnings
- source_matchers:
  - severity = critical
  - alertname = ServiceDown
  target_matchers:
  - severity = warning
  equal: ['service', 'namespace']  # same service and namespace

receivers:
- name: pagerduty-critical
  pagerduty_configs:
  - service_key: '<pagerduty-key>'
    description: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
    severity: critical

- name: slack-warnings
  slack_configs:
  - channel: '#alerts'
    title: '{{ .CommonAnnotations.summary }}'
    text: |
      *Severity:* {{ .CommonLabels.severity }}
      *Namespace:* {{ .CommonLabels.namespace }}
      {{ range .Alerts }}• {{ .Annotations.description }}{{ end }}
    send_resolved: true   # notify when alert resolves
```

> 💡 **Interview tip:** **`group_wait`** is the most impactful AlertManager tuning parameter. It determines how long AlertManager waits after the first alert in a group before sending the notification — this window allows related alerts to be batched together. Too short (5s): you get 50 separate messages before grouping. Too long (5m): you're notified 5 minutes after the first alert fires. The default 30s is usually a good starting point. Lower it for P1 alerts (faster notification), keep higher for warnings (more batching).

---

### Q666 — Prometheus | Conceptual | Intermediate

> Explain how **Prometheus service discovery** works in Kubernetes — what is **`kubernetes_sd_configs`** and how does it automatically discover Pods, Services, Nodes, and Endpoints?
>
> What are the `__meta_kubernetes_*` labels and what is their purpose? How do relabeling rules use them?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — service discovery sections

#### Key Points to Cover:

**Kubernetes service discovery:**
```
Without SD: manually list all targets in prometheus.yml
  → New pod added → manually update config → reload Prometheus
  → Doesn't scale

With kubernetes_sd_configs:
  → Prometheus watches Kubernetes API for changes
  → New pod added → automatically scraped
  → Pod deleted → automatically removed from targets
  → No config reload needed
```

**4 Kubernetes SD roles:**
```yaml
scrape_configs:
- job_name: 'kubernetes-pods'
  kubernetes_sd_configs:
  - role: pod           # discover all pods
  # Available roles:
  # pod:       one target per pod (all pods in cluster)
  # service:   one target per service port
  # endpoints: one target per endpoint behind a service
  # node:      one target per node (for node metrics)
  # ingress:   one target per ingress

  # For cluster-scoped access: needs RBAC
  # ServiceAccount with: get/list/watch on pods, endpoints, nodes
```

**`__meta_kubernetes_*` labels (auto-populated by SD):**
```
For pods:
  __meta_kubernetes_pod_name               → pod name
  __meta_kubernetes_pod_namespace          → namespace
  __meta_kubernetes_pod_label_<label>      → each pod label
  __meta_kubernetes_pod_annotation_<ann>   → each annotation
  __meta_kubernetes_pod_ip                 → pod IP
  __meta_kubernetes_pod_container_name     → container name
  __meta_kubernetes_pod_container_port_number → port number

These are DISCOVERY labels — available before relabeling
All start with __ (double underscore) → dropped after relabeling
Must be explicitly relabeled to keep them
```

**Relabeling to use SD labels:**
```yaml
relabel_configs:
# Only scrape pods with annotation: prometheus.io/scrape=true
- source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
  action: keep
  regex: true

# Use annotation for scrape port (if pod has prometheus.io/port annotation)
- source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
  action: replace
  target_label: __address__
  regex: (.+)
  replacement: ${__meta_kubernetes_pod_ip}:${1}

# Add pod name as label on scraped metrics
- source_labels: [__meta_kubernetes_pod_name]
  action: replace
  target_label: pod

# Add namespace as label
- source_labels: [__meta_kubernetes_pod_namespace]
  action: replace
  target_label: namespace

# Add all pod labels as Prometheus labels
- action: labelmap
  regex: __meta_kubernetes_pod_label_(.+)
```

> 💡 **Interview tip:** The annotation-based discovery (`prometheus.io/scrape: "true"`) is the "opt-in" pattern — only pods that explicitly annotate themselves get scraped. The Prometheus Operator's ServiceMonitor is the "opt-in via CRD" pattern. Neither approach auto-scrapes everything. This is intentional — auto-scraping all pods would include test pods, debugging tools, and generate massive cardinality. Always require an explicit opt-in for new scrape targets.

---

### Q667 — Prometheus | Conceptual | Intermediate

> Explain **Prometheus recording rules** and **alerting rules** — what is the syntax and what are the best practices for each?
>
> Why are recording rules important for performance? Write recording rules for a service's RED metrics (Rate, Errors, Duration) and alerting rules based on those recording rules.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — rules sections

#### Key Points to Cover:

**Why recording rules:**
```
Problem:
  Complex PromQL query computed on EVERY dashboard load
  → 100 users viewing dashboard = 100 expensive queries/second
  → Grafana query timeout at high scale

  Especially bad for:
  → Queries over long time ranges (1 day, 1 week)
  → Queries aggregating across many time series (sum over 1000 pods)
  → Frequently-used queries (main dashboard everyone views)

Solution (recording rules):
  → Pre-compute the expensive query periodically
  → Store result as NEW metric with simple name
  → Dashboard queries the simple metric (fast lookup)
  → Computed once per evaluation interval (not per user)
```

**Recording rule syntax:**
```yaml
# /etc/prometheus/rules/payment-service.yml
groups:
- name: payment_service_recording
  interval: 60s   # evaluate every 60s (default: global evaluation_interval)
  rules:

  # RED Metrics: Rate
  - record: job:http_requests:rate5m    # naming convention: level:metric:operation
    expr: |
      sum by(job, service) (
        rate(http_requests_total{job="payment-service"}[5m])
      )

  # RED Metrics: Errors (error rate %)
  - record: job:http_errors:rate5m_percentage
    expr: |
      100 * sum by(job, service) (
        rate(http_requests_total{job="payment-service", status=~"5.."}[5m])
      )
      /
      sum by(job, service) (
        rate(http_requests_total{job="payment-service"}[5m])
      )

  # RED Metrics: Duration (p99 latency)
  - record: job:http_request_duration_seconds:p99
    expr: |
      histogram_quantile(0.99,
        sum by(job, le) (
          rate(http_request_duration_seconds_bucket{job="payment-service"}[5m])
        )
      )
```

**Alerting rules using recording rules:**
```yaml
- name: payment_service_alerts
  rules:

  # Alert on high error rate using pre-computed recording rule (FAST)
  - alert: PaymentHighErrorRate
    expr: job:http_errors:rate5m_percentage > 5
    for: 5m
    labels:
      severity: critical
      service: payment
    annotations:
      summary: "Payment service error rate {{ $value | humanize }}%"
      runbook: "https://wiki.company.com/runbooks/payment-errors"
      dashboard: "https://grafana.company.com/d/payment"

  # Alert on high latency
  - alert: PaymentHighLatency
    expr: job:http_request_duration_seconds:p99 > 0.5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Payment p99 latency {{ $value | humanize }}s"
```

**Naming convention for recording rules:**
```
level:metric:operation
  level:    aggregation level (job, instance, cluster)
  metric:   base metric name (http_requests, cpu_usage)
  operation: what was done (rate5m, sum, p99)

Examples:
  job:http_requests:rate5m       → per-job request rate over 5m
  instance:cpu:avg5m             → per-instance avg CPU over 5m
  cluster:memory_usage:sum       → cluster-total memory sum
```

> 💡 **Interview tip:** Recording rules also significantly improve **alert evaluation performance**. If your alerting rule runs a complex multi-minute query every 15 seconds, it becomes a performance bottleneck. Pre-computing with recording rules makes the alert evaluation trivial (just check if pre-computed value > threshold). Also: always put recording rules in SEPARATE files from alerting rules — this makes it easy to reload just one type when debugging.

---

## SECTION 7 — TERRAFORM REMAINING (Q668–Q670)

---

### Q668 — Terraform | Conceptual | Beginner-Intermediate

> Explain **Terraform modules** — what is a module, what is the module structure, and what makes a good reusable module?
>
> What is the difference between the **root module** and **child modules**? What is a **module source** and what are the different source types? Write a module for an AWS S3 bucket with best practices.

📁 **Reference:** `nawab312/Terraform` — modules, reusability sections

#### Key Points to Cover:

**What is a module:**
```
Module = any directory with .tf files
  → Every Terraform configuration IS a module (the root module)
  → Child modules = reusable components called from root

Benefits:
  → DRY: create S3 bucket pattern once, use in 20 places
  → Consistency: all S3 buckets follow same standards
  → Versioning: pin module version for stability
  → Abstraction: hide complexity (100-line S3 setup → 5-line call)
```

**Module structure:**
```
modules/s3-bucket/
  main.tf        # resources
  variables.tf   # input variables
  outputs.tf     # output values
  versions.tf    # required providers and versions
  README.md      # documentation
```

**Module source types:**
```hcl
# 1. Local path (development)
module "bucket" {
  source = "./modules/s3-bucket"
}

# 2. GitHub (public)
module "bucket" {
  source = "github.com/myorg/terraform-modules//s3-bucket?ref=v1.2.0"
}

# 3. Terraform Registry (public modules)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}

# 4. S3 bucket (private)
module "bucket" {
  source = "s3::https://my-bucket.s3.us-east-1.amazonaws.com/modules/s3.zip"
}
```

**Well-designed S3 module:**
```hcl
# modules/s3-bucket/variables.tf
variable "bucket_name" {
  type        = string
  description = "Name of the S3 bucket"
}

variable "enable_versioning" {
  type        = bool
  default     = true
  description = "Enable S3 versioning (recommended)"
}

variable "lifecycle_days" {
  type        = number
  default     = 90
  description = "Days before transitioning to Glacier"
}

variable "tags" {
  type        = map(string)
  default     = {}
  description = "Tags to apply to all resources"
}

# modules/s3-bucket/main.tf
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
  tags   = var.tags
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Disabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# modules/s3-bucket/outputs.tf
output "bucket_id" {
  value = aws_s3_bucket.this.id
}

output "bucket_arn" {
  value = aws_s3_bucket.this.arn
}
```

**Using the module:**
```hcl
module "app_data" {
  source         = "./modules/s3-bucket"
  bucket_name    = "company-app-data-${var.environment}"
  enable_versioning = true
  tags = {
    Environment = var.environment
    Team        = "platform"
  }
}

output "data_bucket" {
  value = module.app_data.bucket_arn
}
```

> 💡 **Interview tip:** Good module design follows the **principle of least surprise** — a module should do ONE thing well, expose only the necessary variables, and have sensible secure defaults (encryption on, public access blocked, versioning on). Modules that expose 50+ variables with no defaults are anti-patterns — they force every caller to re-specify security settings that should be non-negotiable. The best modules have 3-5 required variables and bake in best practices as non-overridable defaults.

---

### Q669 — Terraform | Conceptual | Intermediate

> Explain **Terraform `depends_on`**, **`lifecycle`**, and **`moved`** meta-arguments. When do you need `depends_on` and when does Terraform figure out dependencies automatically?
>
> What are `create_before_destroy`, `prevent_destroy`, and `ignore_changes` in the lifecycle block?

📁 **Reference:** `nawab312/Terraform` — meta-arguments, lifecycle sections

#### Key Points to Cover:

**Automatic vs explicit dependencies:**
```
Terraform automatically infers dependency from references:
  resource "aws_subnet" "private" {
    vpc_id = aws_vpc.main.id    # implicit dependency: subnet depends on VPC
  }
  → Terraform knows: create VPC first, then subnet

depends_on: only needed for HIDDEN dependencies
  Cases:
  1. Side effects not visible in config
     → Lambda function reads from a specific S3 path that must exist
     → But you reference the bucket ARN somewhere else
     → S3 path creation is a side effect terraform can't see

  2. Module depends on module
     → Module B assumes Module A ran certain scripts
     → No resource reference between them

  resource "aws_instance" "app" {
    depends_on = [aws_db_instance.main]
    # Even though app doesn't reference db directly
    # App's startup script reads DB endpoint from Secrets Manager
    # DB must be created first so the endpoint exists
  }
```

**lifecycle meta-argument:**
```hcl
resource "aws_db_instance" "production" {
  # ...

  lifecycle {
    # create_before_destroy:
    # Default: destroy old → create new (downtime!)
    # With this: create new → verify → destroy old (zero downtime)
    # Use for: resources where downtime is unacceptable
    create_before_destroy = true

    # prevent_destroy:
    # Fails terraform apply if this resource would be destroyed
    # Use for: databases, critical state (accidental destroy protection)
    prevent_destroy = true

    # ignore_changes:
    # Don't update these attributes if they change outside Terraform
    # Use for: auto-scaling (desired count managed externally),
    #           tags managed by AWS (cost allocation tags)
    ignore_changes = [
      tags["LastDeployedBy"],    # tag set by CI/CD pipeline
      engine_version,             # auto-minor-version upgrades
    ]
  }
}
```

**`moved` block (Terraform 1.1+):**
```hcl
# Rename a resource without destroying and recreating it
# Old name was: aws_instance.web
# New name is: aws_instance.web_server

moved {
  from = aws_instance.web
  to   = aws_instance.web_server
}

# Also: move resources into/out of modules
moved {
  from = aws_s3_bucket.data
  to   = module.storage.aws_s3_bucket.data
}
# Without moved: Terraform would destroy and recreate
# With moved: just updates state file mapping, no AWS API calls
```

> 💡 **Interview tip:** `prevent_destroy = true` is the most important lifecycle setting for production databases. Without it, a typo in a variable name, a module rename, or a `count`-to-`for_each` migration can cause Terraform to propose destroying your production database. `prevent_destroy` makes Terraform error out instead: `Error: Instance cannot be destroyed`. The only way to proceed is to explicitly remove the `prevent_destroy` flag — a human speed-bump that forces intentional action before destroying critical infrastructure.

---

### Q670 — Terraform | Conceptual | Intermediate

> Explain **Terraform workspaces** — what are they, what problem do they solve, and when should you use them vs separate directories?
>
> What is the difference between `terraform workspace` and using separate state files per environment via different backends? What are the **limitations** of workspaces?

📁 **Reference:** `nawab312/Terraform` — workspaces, multi-environment sections

#### Key Points to Cover:

**What workspaces are:**
```
Workspace = separate state file for the same configuration

Default workspace: "default" (always exists)
Create new: terraform workspace new staging
Switch:     terraform workspace select production
List:       terraform workspace list
Current:    terraform workspace show

Same .tf files → different state files → different resources
Use workspace name in config:
  resource "aws_s3_bucket" "data" {
    bucket = "company-${terraform.workspace}-data"
    # Creates: company-dev-data, company-staging-data, company-prod-data
  }
```

**Workspaces vs separate directories:**
```
Workspaces: SAME config, different state
  ✅ Easy to sync changes across environments (one set of .tf files)
  ❌ Same config = same resources (cannot have different resource types per env)
  ❌ If you break the config, ALL workspaces are affected
  ❌ Credentials: same provider block (hard to use different AWS accounts)
  Best for: simple cases, dev/staging identical to prod

Separate directories: DIFFERENT config, different state
  ✅ Full independence: each env can have different resources
  ✅ Different AWS accounts per environment (different provider configs)
  ✅ Break dev without affecting prod
  ❌ Duplication: must sync changes across directories manually
  ❌ More overhead
  Best for: production setups with meaningful environment differences

Terragrunt: DRY + separate directories (best of both worlds)
  → Separate state per environment (like directories)
  → DRY configuration (like workspaces)
  → Recommended approach for large organizations
```

**Workspace limitations:**
```
1. Cannot use different backends per workspace
   → All workspaces share same S3 bucket (just different keys)

2. No isolation of provider credentials
   → Using workspace "prod" doesn't automatically use prod AWS account
   → Must manually manage credentials alongside workspaces

3. No built-in access control
   → Anyone who can terraform workspace select production can modify prod
   → No approval gates

4. State key collision risk
   → If workspace name typo: terraform workspace select prodution (typo)
   → Creates NEW workspace + new state instead of using existing prod

Recommendation:
  Use workspaces for: short-lived environments (feature branch envs)
  Use separate state paths for: long-lived environments (dev/staging/prod)
```

> 💡 **Interview tip:** The most important workspace limitation to mention in interviews: **workspaces provide NO additional security or access control**. A developer who can run `terraform apply` in the dev workspace can also run it in the prod workspace unless you add external controls (separate IAM roles per workspace, CI/CD pipeline gates). This is the primary reason most production organizations prefer separate state files in separate backends (different S3 paths or different S3 buckets) rather than workspaces — the separation is more explicit and easier to secure with AWS IAM.

---

## SECTION 8 — CI/CD REMAINING (Q671–Q673)

---

### Q671 — CI/CD | Conceptual | Beginner-Intermediate

> Explain **GitHub Actions** fundamentals — what is a **workflow**, **job**, **step**, and **action**? What are **events** that trigger workflows?
>
> What is the difference between `runs-on: ubuntu-latest` and `runs-on: self-hosted`? What are **environment variables**, **secrets**, and **context objects** in GitHub Actions?

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` sections

#### Key Points to Cover:

**Core concepts:**
```
Workflow:   YAML file in .github/workflows/ — defines automation
Job:        A set of steps that run on ONE runner (machine)
            Jobs run in PARALLEL by default
            use "needs:" to make jobs sequential
Step:       Individual task within a job
            Can be: shell command (run:) or action (uses:)
Action:     Reusable unit of work
            GitHub Marketplace: thousands of pre-built actions
            uses: actions/checkout@v4  (official)
            uses: user/repo@v1          (community)
Runner:     Machine that executes jobs
            GitHub-hosted: ubuntu-latest, windows-latest, macos-latest
            Self-hosted: your own machine/EC2/VM
```

**Complete workflow example:**
```yaml
name: CI Pipeline

on:                              # EVENTS that trigger this workflow
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'         # 2am daily
  workflow_dispatch:             # manual trigger (button in GitHub UI)

env:                             # workflow-level environment variables
  NODE_VERSION: '20'
  REGISTRY: ghcr.io

jobs:
  test:                          # job name
    runs-on: ubuntu-latest       # GitHub-hosted runner
    
    strategy:                    # matrix: run same job with different params
      matrix:
        node: [18, 20]
        os: [ubuntu-latest, windows-latest]
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4  # action

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node }}  # context: matrix variable

    - name: Install and test
      run: |                     # shell command
        npm ci
        npm test
      env:                       # step-level env vars
        DATABASE_URL: ${{ secrets.DATABASE_URL }}  # secret reference

  deploy:
    runs-on: ubuntu-latest
    needs: test                  # wait for test job to succeed
    if: github.ref == 'refs/heads/main'  # only on main branch
    environment: production      # requires environment approval (if configured)
    steps:
    - uses: actions/checkout@v4
    - name: Deploy
      run: ./deploy.sh
```

**Key context objects:**
```yaml
${{ github.sha }}          # commit SHA
${{ github.ref }}          # branch/tag ref
${{ github.event_name }}   # what triggered (push/pull_request/etc)
${{ github.repository }}   # "owner/repo"
${{ github.actor }}        # user who triggered
${{ runner.os }}           # Linux/Windows/macOS
${{ env.MY_VAR }}          # environment variable
${{ secrets.MY_SECRET }}   # repository/org secret (masked in logs)
${{ vars.MY_VAR }}         # repository/org variable (not masked)
${{ job.status }}          # success/failure/cancelled
```

> 💡 **Interview tip:** The most commonly misunderstood GitHub Actions concept is **parallel vs sequential jobs**. By default, ALL jobs in a workflow run in parallel. If job B requires job A's output or should only run after A succeeds, you need `needs: [job-a]`. This is why CI pipelines often have lint and unit-test jobs running in parallel, but the deploy job `needs: [lint, test]`. Also: `if: success()` is the default behavior — steps/jobs only run if previous succeeded. Use `if: always()` to run cleanup steps even after failure.

---

### Q672 — CI/CD | Conceptual | Intermediate

> Explain **Jenkins Declarative Pipeline** syntax in depth — what are the key sections: `agent`, `environment`, `stages`, `steps`, `post`?
>
> What is the difference between `stage` failure and `post` failure handling? What are `when` conditions? Write a complete Jenkinsfile for a microservice that tests, builds, and deploys with proper error handling.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` sections

#### Key Points to Cover:

**Declarative pipeline structure:**
```groovy
pipeline {
  // AGENT: where to run (required)
  agent {
    label 'linux'     // run on node with 'linux' label
    // OR:
    // any              // any available agent
    // none             // no global agent, defined per stage
    // docker { image 'node:20' }  // run in Docker container
  }

  // ENVIRONMENT: variables available throughout pipeline
  environment {
    IMAGE_NAME = "my-app"
    AWS_REGION = "us-east-1"
    ECR_REGISTRY = credentials('ecr-registry-url')  // from Jenkins credentials
    DB_PASSWORD = credentials('db-password')
  }

  // OPTIONS: pipeline-wide settings
  options {
    timeout(time: 30, unit: 'MINUTES')    // kill pipeline after 30min
    buildDiscarder(logRotator(numToKeepStr: '10'))  // keep last 10 builds
    skipDefaultCheckout()                  // don't auto-checkout at start
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm   // explicit checkout
      }
    }

    stage('Test') {
      steps {
        sh 'npm ci && npm test'
        publishTestResults testResultsPattern: '**/test-results/*.xml'
      }
    }

    stage('Build') {
      when {
        anyOf {
          branch 'main'
          branch pattern: 'release/.*', comparator: 'REGEXP'
        }
      }
      steps {
        sh 'docker build -t ${IMAGE_NAME}:${GIT_COMMIT} .'
      }
    }

    stage('Deploy to Staging') {
      when { branch 'main' }
      steps {
        sh 'kubectl set image deployment/${IMAGE_NAME} ${IMAGE_NAME}=${ECR_REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT}'
      }
    }

    stage('Approval') {
      when { branch 'main' }
      steps {
        input message: 'Deploy to production?', ok: 'Deploy'
      }
    }

    stage('Deploy to Production') {
      when { branch 'main' }
      steps {
        sh './scripts/deploy-prod.sh ${GIT_COMMIT}'
      }
    }
  }

  // POST: always runs after stages complete
  post {
    always {              // runs regardless of outcome
      cleanWs()           // clean workspace
    }
    success {
      slackSend color: 'good', message: "✅ ${JOB_NAME} #${BUILD_NUMBER} passed"
    }
    failure {
      slackSend color: 'danger', message: "❌ ${JOB_NAME} #${BUILD_NUMBER} failed"
      emailext to: 'dev-team@company.com',
               subject: "FAILED: ${JOB_NAME}",
               body: "Check ${BUILD_URL}"
    }
    unstable {            // tests passed but some warnings
      slackSend color: 'warning', message: "⚠️ ${JOB_NAME} unstable"
    }
  }
}
```

**`when` conditions:**
```groovy
when {
  branch 'main'                              // only on main branch
  branch pattern: 'release/.*'             // regex branch match
  environment name: 'DEPLOY', value: 'true' // env var check
  expression { return params.DEPLOY == 'yes' }  // custom expression
  not { branch 'develop' }                  // NOT condition
  anyOf { branch 'main'; branch 'master' }  // OR condition
  allOf { branch 'main'; environment name: 'PROD', value: 'true' }  // AND
  changeRequest()                           // only for Pull Requests
}
```

> 💡 **Interview tip:** The most important Jenkins pipeline practice: **always define `timeout()` in options** — without it, a hung step (waiting for input that never comes, network request that never responds) will run forever and consume your executor. Set `timeout(time: 30, unit: 'MINUTES')` at pipeline level and `timeout(time: 10, unit: 'MINUTES')` on individual long steps. Also: `cleanWs()` in `post { always }` prevents disk space issues from build artifact accumulation on Jenkins agents.

---

### Q673 — CI/CD | Conceptual | Intermediate

> Explain **ArgoCD** — what is it, how does it implement GitOps, and how does it differ from push-based CI/CD (like Jenkins deploying to Kubernetes)?
>
> What is the **sync loop** in ArgoCD? What is the difference between **Auto-Sync** and **Manual Sync**? What happens when there is a configuration drift?

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` sections

#### Key Points to Cover:

**ArgoCD core concept:**
```
Traditional CI/CD (push-based):
  Code push → CI builds → CI deploys → kubectl apply → K8s
  Problem: CI server needs cluster credentials (security risk)
  Problem: No way to detect drift (someone kubectl edits directly)
  Problem: No "what's deployed" history tied to Git

ArgoCD (pull-based GitOps):
  Code push → CI builds image → CI updates Git (image tag)
  ArgoCD watches Git → detects change → pulls manifests → applies to K8s
  Benefits:
    → ArgoCD runs INSIDE the cluster (no external credentials needed)
    → Git is source of truth (cluster state = Git state)
    → Drift detection: anything changed outside Git → detected
    → Complete audit trail: every deployment = a Git commit
```

**ArgoCD sync loop:**
```
Every 3 minutes (default) OR on webhook:
1. Fetch latest manifests from Git
2. Compare with live cluster state (desired vs actual)
3. If same: status = Synced ✅
4. If different: status = OutOfSync ⚠️
   → With auto-sync: automatically apply changes
   → Without auto-sync: alert, wait for manual sync

Refresh vs Sync:
  Refresh: re-read from Git (check for new commits)
  Sync:    apply Git state to cluster (make cluster match Git)
  Hard refresh: invalidate cached manifests (force re-render from source)
```

**Configuration Drift:**
```
Drift = live cluster state ≠ Git state
Causes:
  → Someone ran kubectl edit/patch directly
  → AWS/K8s auto-modifies resources (annotations, status)
  → Another tool changed resources (Helm upgrade outside ArgoCD)

ArgoCD detects drift:
  → Status: OutOfSync (with diff showing what changed)
  → Which fields changed and how

Handling drift:
  Auto-sync with self-heal:
    syncPolicy:
      automated:
        selfHeal: true   # revert drift automatically
  Manual: review drift → decide to sync (revert to Git) or ignore

ignoreDifferences: tell ArgoCD to ignore certain fields
  spec:
    ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
      - /spec/replicas    # HPA manages replicas, ignore drift here
```

> 💡 **Interview tip:** **`selfHeal: true` is a double-edged sword** — it automatically reverts any change made outside Git (even legitimate emergency changes). If you need to make an emergency change directly to the cluster during an incident (when there's no time to push to Git), ArgoCD will revert it within 3 minutes. The professional approach: disable selfHeal for production by default, use it for dev/staging. For emergencies: create a GitOps branch → push fix → ArgoCD syncs. This maintains the audit trail even during incidents.

---

## SECTION 9 — SRE REMAINING (Q674–Q675)

---

### Q674 — SRE | Conceptual | Beginner-Intermediate

> Explain **SRE vs DevOps** — what is the difference? What are the core SRE principles from Google's SRE book?
>
> What is the **SRE role** in an organization and how does it relate to the development team? What is the "**ops tax**" and how does SRE avoid it?

📁 **Reference:** SRE principles, Google SRE book sections

#### Key Points to Cover:

**DevOps vs SRE:**
```
DevOps:
  → Philosophy/culture of breaking down Dev-Ops silos
  → Practices: CI/CD, automation, collaboration
  → "You build it, you run it" (Werner Vogels)
  → Not a specific job role — a way of working

SRE (Site Reliability Engineering):
  → Specific job role created by Google (2003)
  → "DevOps is what you do, SRE is how you do it"
  → Software engineers solving operational problems
  → Prescriptive implementation of DevOps philosophy

Key difference:
  DevOps: culture and practices
  SRE: specific implementation with concrete tools:
       SLOs, error budgets, toil elimination, on-call rotation
```

**Core SRE principles (Google SRE book):**
```
1. Embracing Risk:
   → 100% reliability is the wrong target (too expensive)
   → Find the RIGHT level of reliability (SLO)
   → Error budget: trade reliability for feature velocity

2. Service Level Objectives (SLOs):
   → Define what "reliable enough" means
   → Measure against it
   → Error budget = acceptable unreliability

3. Eliminating Toil:
   → Manual operational work scales poorly
   → Automate toil → more time for engineering
   → SRE cap: < 50% of time on operational work

4. Monitoring and Alerting:
   → Every page must be actionable
   → Alert on symptoms, not causes
   → Distinguish pages (urgent) from tickets (not urgent)

5. Automation:
   → If done manually more than twice → automate
   → Software should manage software

6. Release Engineering:
   → Versioning, packaging, deployment automation
   → Reproducible builds, hermetic releases

7. Simplicity:
   → Simple systems are easier to make reliable
   → Resist accidental complexity
```

**Ops tax and how SRE avoids it:**
```
Ops tax: operations work that consumes all engineering capacity
  → New service → more monitoring → more on-call → more toil
  → Team spends 100% on keeping lights on, zero on improvements

SRE avoids it:
  1. < 50% toil rule: if operational work > 50%, push back to dev team
  2. Error budgets: when budget spent, dev team does reliability work (not just SRE)
  3. Automation: every toil task automated = toil doesn't scale
  4. "SRE team refuses to do ops work" clause:
     → If service consistently exceeds ops capacity → hand back to dev team
     → Creates incentive for dev teams to build reliable services
```

> 💡 **Interview tip:** The most important SRE concept to communicate in an interview: **error budgets align incentives**. Without error budgets, dev teams want to move fast (deploy often) and ops teams want stability (deploy rarely). With error budgets: when the budget is healthy, EVERYONE can move fast. When budget is almost exhausted, EVERYONE slows down for reliability. This shared budget creates aligned incentives — the SRE insight that transformed how Google manages reliability at scale.

---

### Q675 — SRE | Conceptual | Intermediate

> Explain **observability vs monitoring** — what is the difference? What are the **three pillars of observability** (metrics, logs, traces) and what does each tell you?
>
> What is the **difference between white-box and black-box monitoring**? What is the **difference between cause-based and symptom-based alerting** and why does Google SRE recommend symptom-based?

📁 **Reference:** SRE observability, monitoring strategy sections

#### Key Points to Cover:

**Monitoring vs Observability:**
```
Monitoring:
  → Watching KNOWN metrics (CPU, memory, error rate)
  → Dashboard shows: "CPU is high"
  → Good for: known failure modes, expected issues
  → Limitation: can't answer questions you didn't anticipate

Observability:
  → Ability to understand INTERNAL STATE from EXTERNAL OUTPUTS
  → Can answer any question about system behavior without redeploying
  → Includes: metrics + logs + traces + profiling
  → Good for: novel failures, complex distributed systems

Key distinction:
  Monitoring: "is the system down?" (binary)
  Observability: "WHY is this specific request slow for this user?" (exploratory)

A well-monitored system: you know WHAT is broken
An observable system: you know WHAT and WHY
```

**3 Pillars:**
```
Metrics (numbers over time):
  → Aggregated, quantified measurements
  → CPU: 75%, requests/sec: 1234, error rate: 0.5%
  → Tools: Prometheus, CloudWatch, Datadog
  → Good for: dashboards, alerting, trends, capacity planning
  → Bad for: debugging specific request failures

Logs (events as text):
  → Discrete events with full context
  → "2024-01-15 14:05:23 ERROR payment.processor: card declined for user 12345"
  → Tools: ELK Stack, Loki, CloudWatch Logs
  → Good for: debugging, audit trails, detailed error context
  → Bad for: aggregation, trends, high-cardinality queries (too slow)

Traces (request journey):
  → Distributed transaction across multiple services
  → "Request abc123: api(5ms) → payment-service(150ms) → db(145ms SLOW)"
  → Tools: Jaeger, Zipkin, AWS X-Ray, Grafana Tempo
  → Good for: finding bottlenecks in distributed calls
  → Bad for: aggregated metrics, log analysis
```

**White-box vs Black-box monitoring:**
```
White-box (internal):
  → Metrics from INSIDE the system (app exposes /metrics)
  → CPU, memory, queue depth, error counts, latency histograms
  → Advantage: can see exactly what's happening internally
  → Disadvantage: blind to issues users see but metrics don't capture

Black-box (external):
  → Tests system from OUTSIDE (like a user would)
  → Synthetic transactions, ping checks, CloudWatch Synthetics
  → Advantage: catches issues users experience regardless of cause
  → Disadvantage: doesn't tell you WHY something is broken

Use BOTH:
  Black-box: immediately tells you user impact ("checkout is down")
  White-box: tells you why ("DB connection pool exhausted")
```

**Cause vs Symptom alerting:**
```
Cause-based alerting (BAD for pages):
  → "CPU > 80%" (cause)
  → "Disk > 90%" (cause)
  → "JVM heap > 75%" (cause)
  → Problem: CPU high doesn't necessarily = user impact
              CPU high at 3am for batch job = nobody cares
              Alert fatigue: many causes, none affecting users

Symptom-based alerting (GOOD for pages):
  → "Error rate > 1% for 5 minutes" (users affected)
  → "p99 latency > 2 seconds" (users experiencing slowness)
  → "Availability < 99.9% over last hour" (SLO breaching)
  → These ALWAYS mean user impact → always actionable
  → Use cause-based for TICKETS (less urgent debugging hints)

Google SRE rule:
  Page on: symptoms (user-facing impact)
  Ticket on: causes (will eventually become symptoms if not fixed)
```

> 💡 **Interview tip:** The **symptom vs cause alerting** distinction is one of the most impressive answers you can give in an SRE interview. Most engineers page on causes (CPU, disk, memory). Senior SREs page on symptoms (error rate, latency, availability). The test: "if this alert fires at 3am and the on-call investigates, is there always something they can do RIGHT NOW?" CPU alerts fail this test — high CPU might be caused by a batch job that's supposed to run, requiring no action. Error rate alerts pass — errors always need investigation.

---

## Summary — Q626–Q675

| Section | Questions | Topics |
|---|---|---|
| Kubernetes Deep Dives | Q626–Q638 | Pod networking, CNI, kube-apiserver, QoS, RBAC, Pod Security, Jobs/CronJobs, Multi-container, HPA, Taints, PV/PVC, Service Accounts, ConfigMaps/Secrets, Ingress |
| AWS Remaining | Q639–Q648 | Lambda cold starts, CloudWatch Metric Filters, ASG/Launch Templates, ECS, S3 advanced, Cross-account IAM, DynamoDB capacity, CloudTrail, EKS, Security Groups vs NACLs |
| Linux Fundamentals | Q649–Q655 | Boot process, OSI model, Filesystem hierarchy, systemd, Disk management, Networking commands, User management |
| Docker Remaining | Q656–Q659 | Docker architecture, Image vs Container vs VM, Dockerfile instructions, Docker networking deep, Registries/OCI |
| Git Fundamentals | Q660–Q663 | Git data model, fetch/pull/push/cherry-pick, reset/revert/stash, branching strategies |
| Prometheus Remaining | Q664–Q667 | PromQL basics, AlertManager, Kubernetes SD, Recording/Alerting rules |
| Terraform Remaining | Q668–Q670 | Modules, depends_on/lifecycle/moved, Workspaces |
| CI/CD Remaining | Q671–Q673 | GitHub Actions fundamentals, Jenkins Declarative Pipeline, ArgoCD sync loop |
| SRE Remaining | Q674–Q675 | SRE vs DevOps, Observability pillars, Symptom vs Cause alerting |

---

## Complete Question Bank: **Q1–Q675 = 675 Questions**

**Full coverage: fundamentals through advanced across all topics.**

---

*File 2: Remaining Fundamentals + Advanced Gaps*
*DevOps / SRE Interview Preparation — nawab312 GitHub repositories*
