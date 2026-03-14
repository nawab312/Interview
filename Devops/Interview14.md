# DevOps / SRE Interview Questions — File 1: Core Fundamentals (Q576–Q625)
### 50 Questions — Foundation Topics Every Senior Engineer Must Know

> These are the **building-block questions** asked at the START of every interview.
> If you cannot answer these, interviewers do not move to advanced topics.
> All are **100% unique** — zero overlap with Q1–Q575.
> Every question includes **Key Points to Cover** + **💡 Interview Tips**.

---

## Coverage Map

| Topic | Questions |
|---|---|
| **Kubernetes Architecture** | Q576–Q588 |
| **AWS Fundamentals** | Q589–Q601 |
| **Docker / Container Fundamentals** | Q602–Q608 |
| **Linux Fundamentals** | Q609–Q617 |
| **Terraform Fundamentals** | Q618–Q621 |
| **Prometheus / SRE Fundamentals** | Q622–Q625 |

---

## SECTION 1 — Kubernetes Architecture & Fundamentals

---

### Q576 — Kubernetes | Conceptual | Intermediate

> Explain the **complete Kubernetes architecture**. What are all the components in the **control plane** and the **worker node**? What does each component do?
>
> If you had to draw a diagram, what connects to what? What is the difference between the control plane and the data plane?

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md`

#### Key Points to Cover in Your Answer:

**Control Plane Components:**
```
kube-apiserver:
  → The FRONT DOOR of Kubernetes — all communication goes through it
  → REST API server — kubectl, controllers, kubelets all talk to it
  → Validates and processes all API requests
  → The ONLY component that reads/writes to etcd

etcd:
  → Distributed key-value store — the cluster's brain/memory
  → Stores ALL cluster state (pods, services, configs, secrets)
  → Highly available (usually 3 or 5 nodes for quorum)
  → ONLY the apiserver talks to etcd directly

kube-scheduler:
  → Watches for Pods with no assigned node (Pending state)
  → Finds the best node based on: resources, taints, affinity, policies
  → Writes the node assignment back to apiserver (NOT to the node directly)
  → Does NOT start the pod — just assigns it

kube-controller-manager:
  → Runs all built-in controllers in one process
  → Each controller watches the apiserver for its resource type
  → Reconciles desired state → actual state
  → Examples: Deployment controller, ReplicaSet controller, Node controller

cloud-controller-manager:
  → AWS/GCP/Azure-specific controllers
  → Creates Load Balancers, manages node lifecycle
  → Separates cloud-specific logic from core Kubernetes
```

**Worker Node Components:**
```
kubelet:
  → The agent on every node — talks to the apiserver
  → Watches for Pods assigned to its node
  → Tells the container runtime to start/stop containers
  → Reports node and pod status back to apiserver
  → Does NOT run containers itself — delegates to CRI

kube-proxy:
  → Runs on every node
  → Maintains network rules (iptables or IPVS) for Services
  → Enables pods on one node to reach services on other nodes
  → Gets Service/Endpoint info from apiserver → updates iptables

Container Runtime (containerd/CRI-O):
  → Actually runs the containers
  → kubelet talks to it via CRI (Container Runtime Interface)
  → Pulls images, creates containers, manages their lifecycle

Pods:
  → The actual workloads running on the node
```

**Architecture flow:**
```
Developer → kubectl → kube-apiserver → etcd (stores state)
                                    → kube-scheduler (assigns node)
                                    → controller-manager (reconciles)
                                    → kubelet (on node) → containerd → container
```

> 💡 **Interview tip:** The single most important thing to say: **kube-apiserver is the ONLY component that reads and writes etcd**. Everything else (scheduler, controllers, kubelet, kubectl) communicates through the apiserver. This is why the apiserver is the center of everything. Also: the scheduler does NOT start pods — it ONLY writes the node assignment. The kubelet on that node sees the assignment and starts the pod.

---

### Q577 — Kubernetes | Conceptual | Intermediate

> Explain the **Kubernetes reconciliation loop** — what is it and why is it the core design pattern of Kubernetes?
>
> How does a Deployment controller use the reconciliation loop to maintain the desired number of replicas? What is the difference between **desired state** and **actual state**?

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md` — controller pattern

#### Key Points to Cover in Your Answer:

**The core Kubernetes design pattern:**
```
Kubernetes is DECLARATIVE, not imperative.

Imperative: "start 3 pods" (you tell it WHAT TO DO)
Declarative: "I want 3 pods" (you tell it WHAT YOU WANT)

Kubernetes constantly asks:
  "What is the desired state?" (from etcd)
  "What is the actual state?" (from observing the cluster)
  "How do I make actual = desired?"
  → This loop = RECONCILIATION LOOP
```

**How it works:**
```
1. WATCH: Controller watches kube-apiserver for changes
   → Uses informers (efficient watch mechanism, not polling)
   → "Tell me when any Deployment changes"

2. COMPARE: Compare desired vs actual
   → Deployment says: replicas: 3
   → Actual running pods: 2 (one crashed)
   → Difference: need 1 more pod

3. ACT: Take action to close the gap
   → Create 1 new Pod
   → Write back to apiserver

4. REPEAT: Loop forever
   → Next reconciliation: 3 pods running, desired 3 → no action needed
```

**Deployment controller example:**
```
You create: Deployment(replicas=3, image=nginx:1.19)

Deployment Controller sees new Deployment →
  Creates ReplicaSet(replicas=3, image=nginx:1.19)

ReplicaSet Controller sees new ReplicaSet →
  Creates 3 Pods

kube-scheduler sees 3 unscheduled Pods →
  Assigns each to a node

kubelet on each node sees assigned Pods →
  Tells containerd to pull image + run container

--- 1 pod crashes ---

ReplicaSet Controller:
  desired=3, actual=2 → creates 1 new Pod → actual=3
```

**Why this is powerful:**
```
Self-healing:    pod crashes → controller notices → creates new one
Auto-scaling:    HPA changes replicas → controller reconciles
Rolling update:  update image → controller creates new pods, removes old
Node failure:    node gone → pods evicted → controllers reschedule
```

> 💡 **Interview tip:** "Kubernetes is a reconciliation engine" — this one sentence impresses interviewers. Every component (scheduler, controllers, kubelet) is just running reconciliation loops at different scopes. Understanding this pattern explains why Kubernetes is self-healing: any deviation from desired state is automatically corrected. Contrast with the old way: scripts that manually restart failed services (fragile, not scalable).

---

### Q578 — Kubernetes | Conceptual | Intermediate

> Explain **what happens when you run `kubectl apply -f deployment.yaml`** — trace the request through every Kubernetes component from your laptop to the container running.

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md` — request flow

#### Key Points to Cover in Your Answer:

**Complete request flow:**
```
Step 1: kubectl reads your kubeconfig
  ~/.kube/config → gets apiserver URL + credentials
  kubectl serializes deployment.yaml to JSON

Step 2: kubectl → kube-apiserver (HTTPS)
  Authentication: is this a valid user? (cert, token, OIDC)
  Authorization: is this user allowed to create Deployments? (RBAC)
  Admission Controllers: mutate/validate the request
    → MutatingAdmissionWebhook: add default values, inject sidecars
    → ValidatingAdmissionWebhook: reject if policy violated
    → LimitRanger: set default resource limits
    → ResourceQuota: check namespace quota not exceeded

Step 3: apiserver writes to etcd
  Deployment object stored in etcd

Step 4: Deployment Controller (in controller-manager) is notified
  Watches apiserver for Deployment changes (via informer/watch)
  Creates a ReplicaSet object → writes to apiserver → etcd

Step 5: ReplicaSet Controller is notified
  Creates N Pod objects (replicas=3 → creates 3 Pods)
  Pods written to etcd with nodeName="" (unscheduled)

Step 6: kube-scheduler is notified (watches for unscheduled Pods)
  For each unscheduled Pod:
    - Filtering: find nodes that CAN run this pod (resources, taints, affinity)
    - Scoring: rank remaining nodes (least loaded, spread, etc.)
    - Binding: write nodeName=worker-2 back to apiserver

Step 7: kubelet on worker-2 is notified
  Sees a Pod assigned to its node
  Calls containerd (via CRI) to:
    - Pull the image (from ECR/Docker Hub)
    - Create the container
    - Set up volumes, environment variables, secrets
    - Start the container

Step 8: kubelet reports status
  Updates Pod status back to apiserver
  Pod.Status.Phase = Running
  kubectl get pods → shows Running
```

> 💡 **Interview tip:** The most impressive addition is explaining **Admission Controllers** between authentication/authorization and etcd write. Most candidates skip this step. Admission controllers are where: sidecar injection happens (Istio injects Envoy proxy here), default resource limits are added, image policy checks run, and security policies are enforced. This step happens BEFORE the object is persisted to etcd.

---

### Q579 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes liveness, readiness, and startup probes**. What is the purpose of each? What happens if each one fails?
>
> Give a real example of when you would use all three on the same container. What are the 3 probe mechanisms (httpGet, exec, tcpSocket)?

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md` — probes

#### Key Points to Cover in Your Answer:

**3 probe types and consequences of failure:**

```
Liveness Probe:
  Question: "Is this container alive (not in a broken state)?"
  Failure action: KILL the container and restart it
  Use case: detect deadlocks, infinite loops, hung processes
  Example: app is running but stuck in deadlock, not processing requests
  WITHOUT liveness: container runs forever in broken state, never auto-healed

Readiness Probe:
  Question: "Is this container ready to RECEIVE traffic?"
  Failure action: REMOVE from Service endpoints (no traffic sent)
  Container stays running — just removed from load balancer rotation
  Use case: app startup (takes 30s to warm up), temporary overload
  WITHOUT readiness: traffic sent to pods that aren't ready → errors

Startup Probe:
  Question: "Has this slow-starting container finished initializing?"
  Failure action: KILL the container (same as liveness)
  Purpose: gives slow-starting apps time before liveness kicks in
  Use case: legacy Java app takes 3 minutes to start
  WITHOUT startup: liveness probe kills app before it finishes starting
```

**Example — all three on same container:**
```yaml
containers:
- name: payment-service
  image: payment-service:v1.2.3

  # Startup probe: give app up to 5 minutes to start
  # (30 failures × 10s interval = 300s max)
  startupProbe:
    httpGet:
      path: /healthz
      port: 8080
    failureThreshold: 30
    periodSeconds: 10

  # Liveness probe: if /healthz returns 500 → restart container
  # (only runs AFTER startupProbe succeeds)
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 0    # startup probe handles initial delay
    periodSeconds: 10
    failureThreshold: 3       # 3 consecutive failures → restart

  # Readiness probe: only send traffic if /readyz returns 200
  # (stricter than liveness — checks dependencies too)
  readinessProbe:
    httpGet:
      path: /readyz           # checks DB connection, cache, etc.
      port: 8080
    periodSeconds: 5
    failureThreshold: 1       # even 1 failure → remove from Service
    successThreshold: 2       # need 2 consecutive passes to add back
```

**3 probe mechanisms:**
```
httpGet:   HTTP GET request → success if 200-399
           Best for: HTTP services
exec:      Run command inside container → success if exit code 0
           Best for: databases, custom checks
           Example: exec: command: ["pg_isready", "-U", "postgres"]
tcpSocket: TCP connection attempt → success if connection opens
           Best for: non-HTTP services (databases, message queues)
```

> 💡 **Interview tip:** The most important distinction: **liveness failure = RESTART, readiness failure = REMOVE FROM TRAFFIC**. A common mistake is using liveness for everything — if your liveness probe is too aggressive, you create restart loops that make things worse. Readiness is safer: the pod stays alive, just gets no traffic until it recovers. Use liveness only for truly unrecoverable states (deadlock, OOM).

---

### Q580 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes Service types** — ClusterIP, NodePort, LoadBalancer, ExternalName, and Headless.
>
> How does each one expose your application? In what scenario would you use each? How does a ClusterIP Service actually route traffic to pods?

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — Service types

#### Key Points to Cover in Your Answer:

**5 Service types:**

```
ClusterIP (default):
  → Virtual IP reachable ONLY inside the cluster
  → kube-proxy creates iptables/IPVS rules for this IP
  → Traffic to ClusterIP → load balanced across healthy pods
  → Use: internal service-to-service communication
  → Cannot be reached from outside cluster

NodePort:
  → Opens same port on EVERY node (30000-32767)
  → External traffic → any node IP:nodePort → routes to pods
  → Also creates ClusterIP internally
  → Use: development/testing, when no cloud load balancer available
  → Problem: exposes ports on every node, clients must know node IPs

LoadBalancer:
  → Creates cloud provider load balancer (AWS ALB/NLB, GCP LB)
  → Also creates NodePort and ClusterIP
  → External clients → cloud LB → NodePort → pods
  → Use: production internet-facing services
  → Problem: one LB per service = expensive (use Ingress instead for HTTP)

ExternalName:
  → CNAME alias for external DNS name
  → spec.externalName: "my-database.company.com"
  → Use: migrate external service to in-cluster gradually
  → Pods use Service name, Service resolves to external DNS

Headless (ClusterIP: None):
  → No virtual IP — DNS returns pod IPs directly
  → DNS query → gets list of all pod IPs (not a single VIP)
  → Use: StatefulSets (each pod has stable DNS), service discovery
  → Use: when clients need to connect to ALL pods (Kafka, Cassandra)
```

**How ClusterIP routing works:**
```
Service: payment-service → ClusterIP: 10.96.100.50
Pods behind service: 10.0.1.5, 10.0.1.6, 10.0.1.7

kube-proxy creates iptables rules on every node:
  PREROUTING: if dst=10.96.100.50 → randomly pick one of:
    → DNAT to 10.0.1.5  (33%)
    → DNAT to 10.0.1.6  (33%)
    → DNAT to 10.0.1.7  (33%)

Pod calls: curl http://payment-service:8080
  1. DNS: payment-service → 10.96.100.50 (from CoreDNS)
  2. Packet sent to 10.96.100.50
  3. iptables DNAT → one of the pod IPs
  4. Packet reaches pod directly
```

> 💡 **Interview tip:** The most important thing to understand about ClusterIP: **it is a virtual IP that does not exist on any network interface**. It is purely an iptables/IPVS construct. When kube-proxy sees a new Service, it writes iptables rules that intercept packets destined for the ClusterIP and redirect them to actual pod IPs. This is why Services still work when pods move to different nodes — iptables rules are updated automatically.

---

### Q581 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes resource requests and limits** — what is the difference and why do both matter?
>
> What are **QoS classes** (Guaranteed, Burstable, BestEffort)? What happens during **node memory pressure** — which pods get evicted first? What is CPU throttling?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

#### Key Points to Cover in Your Answer:

**Requests vs Limits:**
```
Request:
  → "I need at least this much"
  → Used by SCHEDULER to decide which node to place pod on
  → Guaranteed allocation — node will always have this available for the pod
  → Sum of all pod requests ≤ node allocatable capacity

Limit:
  → "I cannot use more than this"
  → Enforced by KUBELET/kernel via cgroups
  → CPU limit: container THROTTLED when exceeded (slowed down, not killed)
  → Memory limit: container KILLED (OOMKilled) when exceeded
```

**CPU vs Memory — different enforcement:**
```
CPU (compressible resource):
  → Throttled: process slows down but keeps running
  → Process can't use more CPU cycles than its limit
  → No crash — just performance degradation
  → Check throttling: kubectl top pod or prometheus container_cpu_cfs_throttled

Memory (incompressible resource):
  → OOMKilled: process killed when it exceeds memory limit
  → kernel sends SIGKILL
  → Pod restarts → CrashLoopBackOff if keeps happening
  → Check: kubectl describe pod → "OOMKilled" in events
```

**3 QoS Classes:**
```
Guaranteed (highest priority, evicted LAST):
  → requests == limits for ALL containers
  → Both CPU and memory have equal request and limit
  resources:
    requests: {cpu: "500m", memory: "512Mi"}
    limits:   {cpu: "500m", memory: "512Mi"}  ← same!

Burstable (medium priority):
  → requests < limits (can burst above request)
  → OR: only some containers have limits set
  resources:
    requests: {cpu: "100m", memory: "128Mi"}
    limits:   {cpu: "1",    memory: "512Mi"}  ← higher than request

BestEffort (lowest priority, evicted FIRST):
  → NO requests OR limits set at all
  → No resource guarantee whatsoever
  resources: {}   ← nothing set
```

**Eviction order during node memory pressure:**
```
Node memory runs low → kubelet evicts pods in this order:
1. BestEffort pods (no resources set) — evicted FIRST
2. Burstable pods that exceeded their requests
3. Burstable pods that are below their requests
4. Guaranteed pods — evicted LAST (only if no other option)

Rule: ALWAYS set requests and limits in production
      BestEffort pods are evicted at the first sign of pressure
      Guaranteed pods are essentially protected
```

> 💡 **Interview tip:** The most commonly asked follow-up: "What should you set requests and limits to?" Best practice: (1) set requests based on average usage observed in production, (2) set limits 2-3× the request, (3) for Java apps — NEVER set CPU limit (JVM GC is bursty — throttling causes latency spikes). For memory: always set both request and limit to avoid OOMKills from surprise memory growth.

---

### Q582 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes ConfigMaps and Secrets** — what are they, what is the difference, and what are the different ways to use them in Pods?
>
> What are the **types of Kubernetes Secrets**? What is the security concern with Secrets and how should you handle it properly?

📁 **Reference:** `nawab312/Kubernetes` → `05_CONFIGURATION_SECRETS.md`

#### Key Points to Cover in Your Answer:

**ConfigMap vs Secret:**
```
ConfigMap:
  → Non-sensitive configuration data
  → Stored in etcd as plaintext
  → Examples: app.properties, database host, feature flags, config files
  → Max size: 1MB

Secret:
  → Sensitive data: passwords, tokens, certificates, keys
  → Stored in etcd BASE64 ENCODED (NOT encrypted by default!)
  → Kubernetes does NOT encrypt secrets at rest by default
  → Max size: 1MB
```

**3 ways to use ConfigMap/Secret in Pod:**
```yaml
# Method 1: Environment variable (single key)
env:
- name: DB_HOST
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: database.host
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password

# Method 2: All keys as env vars
envFrom:
- configMapRef:
    name: app-config    # ALL keys become env vars
- secretRef:
    name: db-secret

# Method 3: Mount as files (best for large configs, certificates)
volumes:
- name: config-volume
  configMap:
    name: app-config
- name: secret-volume
  secret:
    secretName: db-secret
    defaultMode: 0400   # read-only, owner only (important for security)
volumeMounts:
- name: config-volume
  mountPath: /etc/config
- name: secret-volume
  mountPath: /etc/secrets
  readOnly: true
```

**Secret types:**
```
Opaque (default):    arbitrary key-value data
kubernetes.io/tls:   TLS certificate + private key
kubernetes.io/dockerconfigjson:  Docker registry credentials
kubernetes.io/service-account-token:  SA token (auto-created)
kubernetes.io/ssh-auth:  SSH credentials
bootstrap.kubernetes.io/token:  bootstrap tokens
```

**The Security Problem — and real fix:**
```
Problem: kubectl get secret db-secret -o yaml → shows base64 data
         base64 decode → plaintext password
         Secrets stored in etcd as base64 (not encrypted!)
         Anyone with etcd read access = all secrets exposed

Partial fix (built-in):
  Enable encryption at rest: EncryptionConfiguration + KMS provider
  → Encrypts secrets in etcd using AWS KMS

Real production fix:
  Use External Secrets Operator + AWS Secrets Manager
  → Secrets NEVER stored in K8s etcd
  → ExternalSecret CRD → fetches from Secrets Manager at runtime
  → Rotation handled by Secrets Manager
```

> 💡 **Interview tip:** Always mention that **Kubernetes Secrets are NOT secret by default** — they are just base64 encoded. This is one of the most important security points. In banking/fintech, the answer is always: use External Secrets Operator with AWS Secrets Manager or HashiCorp Vault. Never commit Secret YAML files to Git (even though the values are base64). Use Sealed Secrets or External Secrets for GitOps workflows.

---

### Q583 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes namespaces** — what are they, what do they provide, and what do they NOT provide?
>
> What is the difference between **namespace-scoped** and **cluster-scoped** resources? How do `ResourceQuota` and `LimitRange` work at the namespace level?

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md` — namespaces

#### Key Points to Cover in Your Answer:

**What namespaces provide:**
```
Logical isolation within a cluster:
  → Separate "virtual clusters" within one physical cluster
  → Resource names unique within namespace (not across cluster)
  → RBAC can be scoped to namespace
  → Network Policies can isolate by namespace
  → Resource quotas per namespace

Default namespaces:
  default:         where your resources go without specifying namespace
  kube-system:     Kubernetes system components (CoreDNS, kube-proxy)
  kube-public:     publicly readable resources (cluster-info)
  kube-node-lease: node heartbeat Lease objects
```

**What namespaces do NOT provide:**
```
❌ NOT strong security isolation (containers can still be privileged)
❌ NOT network isolation by default (pods can reach across namespaces)
   → Need NetworkPolicy for actual network isolation
❌ NOT node isolation (pods from different namespaces run on same nodes)

For true isolation: use separate clusters (for different customers/teams)
For soft isolation (same company, different teams): namespaces are fine
```

**Namespace-scoped vs Cluster-scoped:**
```
Namespace-scoped (belong to a namespace):
  Pods, Deployments, Services, ConfigMaps, Secrets, PVCs
  Roles, RoleBindings, ServiceAccounts, Ingress

Cluster-scoped (exist at cluster level, no namespace):
  Nodes, PersistentVolumes, StorageClasses
  ClusterRoles, ClusterRoleBindings
  Namespaces themselves, CustomResourceDefinitions
```

**ResourceQuota — limit total resources in namespace:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"           # total CPU requests ≤ 10 cores
    requests.memory: "20Gi"      # total memory requests ≤ 20GB
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"                   # max 50 pods
    count/deployments.apps: "20" # max 20 deployments
    persistentvolumeclaims: "10"
```

**LimitRange — default + enforce per-pod limits:**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:             # applied if container has NO limits set
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:      # applied if container has NO requests set
      cpu: "100m"
      memory: "128Mi"
    max:                 # container CANNOT exceed these
      cpu: "2"
      memory: "2Gi"
    min:                 # container MUST request at least this
      cpu: "50m"
      memory: "64Mi"
```

> 💡 **Interview tip:** LimitRange and ResourceQuota work together as a two-layer system: LimitRange ensures EVERY container has requests/limits set (prevents BestEffort QoS pods if you require minimum resources), while ResourceQuota caps the TOTAL consumption of the namespace. Without LimitRange, a developer who forgets to set resource limits creates BestEffort pods that get evicted first — which is surprising to the developer.

---

### Q584 — Kubernetes | Conceptual | Intermediate

> Explain the difference between **Deployment**, **ReplicaSet**, **StatefulSet**, and **DaemonSet**. When would you use each?
>
> What is the relationship between a Deployment and ReplicaSet? Why does Kubernetes have both?

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`

#### Key Points to Cover in Your Answer:

**Deployment vs ReplicaSet relationship:**
```
Deployment manages ReplicaSets, ReplicaSet manages Pods

Deployment → creates → ReplicaSet v1 (3 pods)
  → rolling update → creates → ReplicaSet v2 (3 pods)
                   → scales down → ReplicaSet v1 (0 pods, kept for rollback)

Why both exist:
  ReplicaSet: knows how to maintain N copies of a pod template
  Deployment: knows how to update pods (rolling/recreate) and rollback
  Separation of concerns: scaling logic ≠ update logic
```

**4 workload types:**
```
Deployment (stateless apps):
  → Pods are identical, interchangeable
  → Pod names random: web-7d8f9b-xkz4p
  → Volumes shared or none
  → Use: web servers, APIs, microservices
  → Supports: rolling update, rollback, scaling

StatefulSet (stateful apps):
  → Pods have stable identity (web-0, web-1, web-2)
  → Stable DNS: web-0.my-service.namespace.svc.cluster.local
  → Each pod gets its OWN PVC (not shared)
  → Ordered start/stop (web-0 before web-1)
  → Use: databases (PostgreSQL, MongoDB), Kafka, Elasticsearch
  → Supports: rolling update but ordered

DaemonSet (node-level agents):
  → Runs exactly ONE pod on EVERY node (or selected nodes)
  → Auto-adds pod when new node joins cluster
  → Auto-removes pod when node is removed
  → Use: log collectors (Fluentbit), monitoring agents (node-exporter),
         network plugins (Cilium), storage agents
  → NOT for application workloads

Job/CronJob (batch):
  → Job: runs to completion (exit 0), then done
  → CronJob: runs Job on a schedule
  → Pods not restarted after successful completion
  → Use: database migrations, report generation, data processing
```

**Comparison table:**
```
                Deployment   StatefulSet   DaemonSet    Job
Pod identity:   Random       Stable(0,1,2) Per-node     Temporary
Pod DNS:        Shared svc   Unique each   N/A          N/A
PVC:            Shared/none  One per pod   Usually none Usually none
Start order:    Any          Ordered       N/A          N/A
Runs forever:   Yes          Yes           Yes          No (to completion)
```

> 💡 **Interview tip:** The most important distinction in interviews: **Deployment for stateless, StatefulSet for stateful**. The key question to ask: "Do these pods need their own persistent storage?" If yes → StatefulSet. "Does each pod need to know who it is (pod-0, pod-1)?" If yes → StatefulSet. For DaemonSet, the question is: "Should this run on every node?" Node monitoring agents, log shippers, and CNI plugins are classic DaemonSet use cases.

---

### Q585 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes labels, annotations, and selectors**. What is the difference between labels and annotations?
>
> How do Services use selectors to find pods? What happens if the Service selector does not match any pods?

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md`

#### Key Points to Cover in Your Answer:

**Labels vs Annotations:**
```
Labels:
  → Key-value pairs for IDENTIFYING and SELECTING resources
  → Used by: Services (pod selection), Deployments (pod ownership)
  → Used by: kubectl get pods -l app=nginx
  → Used by: Node selectors, affinity rules
  → Keep them: concise, meaningful, indexable
  → Limited length: 63 char value, 253 char key

Annotations:
  → Key-value pairs for METADATA / NON-IDENTIFYING info
  → NOT used for selection
  → Can be large (e.g., last deployment info, tool config)
  → Used by: tools, operators (Prometheus scraping config, Ingress config)
  → Examples:
    prometheus.io/scrape: "true"
    kubernetes.io/ingress.class: "nginx"
    deployment-tool: "argocd"
    last-applied-at: "2024-01-15T14:05:00Z"
```

**Service selector — how it finds pods:**
```yaml
# Service with selector
apiVersion: v1
kind: Service
metadata:
  name: payment-svc
spec:
  selector:
    app: payment          # match pods with BOTH labels
    version: v2
  ports:
  - port: 80
    targetPort: 8080

# Pod that MATCHES the selector
metadata:
  labels:
    app: payment          # ✅ matches
    version: v2           # ✅ matches
    team: payments        # extra label — doesn't matter

# Pod that DOES NOT match
metadata:
  labels:
    app: payment          # ✅
    version: v1           # ❌ wrong version — not selected
```

**What happens when selector matches no pods:**
```
Service exists but Endpoints object has no addresses:
  kubectl get endpoints payment-svc → NONE

Any request to Service ClusterIP → connection refused
  (iptables has no backend to route to)

Debug:
  kubectl get endpoints payment-svc → see if any IPs listed
  kubectl get pods -l app=payment,version=v2 → see if pods exist
  kubectl describe pod <pod> → check labels match exactly
```

**Selector types:**
```yaml
# Equality-based (Service, ReplicationController)
selector:
  app: nginx       # app == nginx

# Set-based (Deployment, DaemonSet, Job)
selector:
  matchLabels:
    app: nginx
  matchExpressions:
  - key: environment
    operator: In
    values: [production, staging]
  - key: tier
    operator: NotIn
    values: [frontend]
  - key: beta
    operator: DoesNotExist  # key must NOT exist
```

> 💡 **Interview tip:** Label selector mismatch is one of the most common Kubernetes configuration bugs — Service creates successfully but never routes traffic because the selector doesn't match the pods. The debug command: `kubectl get endpoints <service-name>`. If it shows `<none>`, the selector is wrong. Compare `kubectl describe service` (selector field) with `kubectl describe pod` (labels field) side by side.

---

### Q586 — Kubernetes | Conceptual | Intermediate

> Explain **how Kubernetes Ingress works** — what is the Ingress resource vs Ingress Controller?
>
> What is the flow from a user's browser to a Pod when Ingress is configured? What is TLS termination at the Ingress level?

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — Ingress

#### Key Points to Cover in Your Answer:

**Ingress resource vs Ingress Controller:**
```
Ingress resource:
  → Kubernetes API object — defines routing RULES
  → "Route traffic for api.company.com/payments to payment-service:80"
  → Just configuration — does nothing by itself

Ingress Controller:
  → The actual software that READS Ingress resources and implements them
  → Runs as pods in the cluster (usually a DaemonSet or Deployment)
  → Examples: Nginx Ingress Controller, Traefik, AWS ALB Ingress Controller
  → Watches for Ingress resources → reconfigures itself (nginx.conf, etc.)
  → Without a controller, Ingress resources are ignored
```

**Complete traffic flow:**
```
1. DNS: api.company.com → Load Balancer IP (AWS NLB/ALB)

2. Load Balancer → Ingress Controller pods
   (NodePort or direct to Ingress Controller Service)

3. Ingress Controller reads Ingress rules:
   Host: api.company.com
   Path: /payments → payment-service:80
   Path: /auth     → auth-service:80

4. Ingress Controller → Service → Pods

For Nginx Ingress Controller specifically:
  Nginx runs IN the pod, reads Ingress objects, updates nginx.conf
  Traffic: Client → Nginx pod → Service → App Pod
```

**Ingress resource example:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - api.company.com
    secretName: api-tls-secret    # TLS cert stored as Secret
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /payments
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 80
      - path: /auth
        pathType: Prefix
        backend:
          service:
            name: auth-service
            port:
              number: 80
```

**TLS termination:**
```
Client → HTTPS → Ingress Controller (decrypts TLS here)
                → HTTP → Backend pods (unencrypted inside cluster)

Benefits:
  → Backend pods don't need SSL certificates
  → Centralized certificate management
  → cert-manager auto-renews certificates, updates the Secret

For end-to-end encryption (mTLS to pods):
  → Ingress Controller → HTTPS → backend pods
  → Or use service mesh (Istio) for pod-level mTLS
```

> 💡 **Interview tip:** The most important thing to emphasize: Ingress is just a **configuration object** — it does nothing without an Ingress Controller. This confuses many beginners who create an Ingress and wonder why it doesn't work. Also: one Ingress resource can route many domains/paths, which is why Ingress is cheaper than one LoadBalancer Service per microservice (Ingress = one cloud load balancer for all services vs N cloud load balancers).

---

### Q587 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes volume types** — emptyDir, hostPath, ConfigMap, Secret, PVC. When would you use each?
>
> What is the lifecycle of each volume type relative to the Pod?

📁 **Reference:** `nawab312/Kubernetes` → `04_STORAGE.md`

#### Key Points to Cover in Your Answer:

**Volume types and lifecycles:**
```
emptyDir:
  → Temporary directory created when Pod starts
  → Lives as long as the Pod (deleted when Pod dies)
  → Shared between ALL containers in the same Pod
  → Use: sharing files between init container and main container
  → Use: temporary scratch space (sorting large datasets)
  → Storage: memory (medium: Memory) or disk (default)
  → NOT for persistent data

hostPath:
  → Mounts a file or directory from the HOST NODE's filesystem
  → Persists even when Pod is deleted (data on host)
  → Security risk: pod can access host files
  → Use: node-level agents (Fluentbit reading /var/log/containers)
  → Use: DaemonSets that need host filesystem access
  → NOT for portable apps (different nodes have different files)

ConfigMap/Secret:
  → Mounts ConfigMap or Secret keys as files
  → Auto-updates when ConfigMap/Secret changes (after ~1 minute)
  → Read-only by default
  → Use: config files, certificates, credentials

PersistentVolumeClaim (PVC):
  → Requests persistent storage from the cluster
  → Independent of Pod lifecycle — survives pod deletion
  → Storage provisioned from StorageClass (EBS, EFS, etc.)
  → Use: databases, stateful applications
```

**Volume lifecycle comparison:**
```
emptyDir:     Created with pod → DELETED with pod
hostPath:     Exists before pod → PERSISTS after pod deleted
ConfigMap:    Exists independently → updated when CM changes
Secret:       Exists independently → updated when Secret changes
PVC:          Exists independently → persists after pod deleted
              (unless storageClass has Delete reclaim policy)
```

**emptyDir shared between containers:**
```yaml
spec:
  volumes:
  - name: shared-data
    emptyDir: {}

  initContainers:
  - name: data-loader
    image: busybox
    command: ["sh", "-c", "echo 'config' > /data/config.json"]
    volumeMounts:
    - name: shared-data
      mountPath: /data

  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: shared-data
      mountPath: /data    # reads config.json written by initContainer
```

> 💡 **Interview tip:** The most practical volume question in interviews: "How do you share data between two containers in the same pod?" The answer is **emptyDir** — it is created fresh when the pod starts and shared by all containers in that pod. Classic use case: an init container downloads or transforms data into an emptyDir, then the main container reads it. This is the canonical Kubernetes pattern for container initialization.

---

### Q588 — Kubernetes | Conceptual | Intermediate

> Explain **Kubernetes kube-scheduler internals** — how does the scheduler decide which node to place a Pod on?
>
> What are **Filtering** and **Scoring** phases? What factors affect node selection? What happens if NO node passes filtering?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — scheduler

#### Key Points to Cover in Your Answer:

**2-phase scheduling:**
```
Phase 1: FILTERING (Predicates)
  → Eliminate nodes that CANNOT run this pod
  → Result: list of feasible nodes

Phase 2: SCORING (Priorities)
  → Rank remaining feasible nodes
  → Highest score wins
  → Pod bound to winning node
```

**Filtering checks (eliminates nodes):**
```
Resource fit:
  → Node has enough CPU/memory for pod requests
  → Node.allocatable - running_pods_requests >= pod_requests

Node selector / Node affinity:
  → Pod spec.nodeSelector matches node labels
  → spec.affinity.nodeAffinity rules satisfied

Taints and tolerations:
  → Node has taint → pod must have matching toleration
  → If no toleration: node filtered out

Volume constraints:
  → Node supports the volume type required
  → Max volumes per node not exceeded (EBS: 25 per node)

Pod anti-affinity:
  → Filters out nodes where pods with conflicting labels already run

HostPort conflicts:
  → If pod uses hostPort, must find node where that port is free
```

**Scoring (ranks remaining nodes):**
```
LeastAllocated: prefer nodes with most free resources
  → Spread pods evenly across cluster

MostAllocated: prefer nodes with least free resources
  → Pack pods tightly (cost optimization)

NodeAffinity: higher score if preferred affinity matches

ImageLocality: prefer nodes that already have the container image
  → Avoids image pull delay
  → Image already on node = higher score

InterPodAffinity: prefer nodes near pods we want to be close to
```

**When NO node passes filtering — Pending Pod:**
```
Pod stays in Pending state
Event: "0/10 nodes are available: 5 Insufficient cpu, 5 node(s) had taint"

Common causes:
  → Not enough CPU/memory on any node → add nodes or reduce requests
  → Node selector/affinity doesn't match any node → check labels
  → Taint with no toleration → add toleration or remove taint
  → All nodes have DiskPressure/MemoryPressure → fix node health

Debug:
  kubectl describe pod <pending-pod>
  → Look at Events section for scheduling failure reason
```

> 💡 **Interview tip:** The most important debugging tip for scheduling: **`kubectl describe pod <pod-name>` → look at Events**. The scheduler writes exactly WHY it couldn't schedule the pod. "0/5 nodes are available: 3 Insufficient memory, 2 node(s) had taint" tells you precisely: 3 nodes are full, 2 nodes have taints your pod doesn't tolerate. You never need to guess — the scheduler tells you.

---

## SECTION 2 — AWS Fundamentals

---

### Q589 — AWS | Conceptual | Beginner-Intermediate

> Explain **AWS global infrastructure** — what are **Regions**, **Availability Zones**, and **Edge Locations**?
>
> Why does high availability require multiple AZs? What is the difference between **Multi-AZ** and **Multi-Region**? What services are Global vs Regional vs AZ-specific?

📁 **Reference:** `nawab312/AWS` — Global infrastructure, HA architecture

#### Key Points to Cover in Your Answer:

```
Region:
  → Geographic area with multiple isolated data centers
  → Examples: us-east-1 (N. Virginia), eu-west-1 (Ireland)
  → Each region is INDEPENDENT — isolated from failures in other regions
  → Resources stay in region unless you explicitly replicate
  → Choose region: latency to users, compliance (data sovereignty), services available

Availability Zone (AZ):
  → One or more physical data centers within a Region
  → Isolated from failures in other AZs (separate power, network, cooling)
  → Connected to other AZs in same region via low-latency fiber
  → Examples: us-east-1a, us-east-1b, us-east-1c (3 AZs)
  → HA requires deploying across MULTIPLE AZs
  → One AZ fails → other AZs handle traffic

Edge Locations:
  → CDN points of presence worldwide (400+ locations)
  → Used by: CloudFront, Route53, AWS Shield
  → Cache content closer to users → lower latency
  → NOT for compute/storage workloads

Multi-AZ vs Multi-Region:
  Multi-AZ:     same region, different physical facilities
                → protects against: datacenter fire, power failure
                → RTO: seconds-minutes (automatic failover)
                → RPO: near-zero (synchronous replication usually)
                → Cost: moderate

  Multi-Region: different geographic regions
                → protects against: natural disaster, regional outage
                → RTO: minutes-hours (requires DNS failover, data sync)
                → RPO: seconds-minutes (asynchronous replication)
                → Cost: high (full duplicate infrastructure)
```

**Global vs Regional vs AZ services:**
```
Global services (not tied to region):
  IAM, Route53, CloudFront, WAF (some), Organizations

Regional services (exist per region, span AZs):
  VPC, S3, Lambda, DynamoDB, SQS, SNS, ECR

AZ-specific resources (exist in one AZ):
  EC2 instances, EBS volumes, Subnets
  (attach EBS to EC2 in SAME AZ — different AZ = cannot attach)
```

> 💡 **Interview tip:** The most important rule: **EBS volumes are AZ-specific**. A common mistake: create EBS in us-east-1a, try to attach to EC2 in us-east-1b → fails. Always create EBS in the same AZ as the EC2 instance. For HA databases: use RDS Multi-AZ (synchronous replication between AZs in same region) — not EBS snapshots across AZs.

---

### Q590 — AWS | Conceptual | Intermediate

> Explain **AWS ALB vs NLB** — what is the fundamental difference and when would you use each?
>
> What OSI layer does each operate at? What are the key features unique to each? When would NLB be the wrong choice and vice versa?

📁 **Reference:** `nawab312/AWS` — ALB, NLB, load balancing

#### Key Points to Cover in Your Answer:

**Fundamental difference:**
```
ALB (Application Load Balancer):
  → Operates at Layer 7 (Application layer — HTTP/HTTPS)
  → Understands HTTP: headers, paths, methods, query strings
  → Can route based on URL path, host header, HTTP method
  → Use: web applications, microservices, HTTP-based APIs

NLB (Network Load Balancer):
  → Operates at Layer 4 (Transport layer — TCP/UDP/TLS)
  → Does NOT understand HTTP — just raw TCP/UDP connections
  → Extremely high performance: millions of requests/second
  → Preserves client IP address (ALB replaces with its own IP)
  → Use: high-performance TCP, gaming, IoT, non-HTTP protocols
```

**ALB-only features:**
```
✅ Host-based routing: api.company.com → api-service
✅ Path-based routing: /api → api-service, /static → cdn
✅ HTTP header routing: route by User-Agent, custom header
✅ Query string routing: ?version=v2 → v2 service
✅ Weighted routing (canary deployments)
✅ Redirect HTTP → HTTPS
✅ Fixed response (return 200 without backend)
✅ WAF integration (Layer 7 protection)
✅ Lambda as target
✅ gRPC support
```

**NLB-only features:**
```
✅ Static IP per AZ (same IP always — good for whitelisting)
✅ Ultra-low latency (~100 microseconds vs ALB ~1ms)
✅ Millions of requests per second
✅ Client IP preservation (backend sees real client IP)
✅ TCP passthrough (end-to-end TLS without termination)
✅ UDP support (ALB is TCP only)
✅ Non-HTTP protocols: SMTP, MQTT, custom TCP
```

**Decision guide:**
```
Use ALB when:
  → HTTP/HTTPS application (99% of web apps)
  → Need path/host-based routing
  → Need WAF, Lambda targets
  → Canary deployments (weighted routing)

Use NLB when:
  → Need static IP for firewall whitelisting
  → Need client IP preservation
  → Non-HTTP protocol (game server, MQTT, custom TCP)
  → Ultra-high throughput / ultra-low latency
  → TLS passthrough to backend

When NLB is WRONG:
  → You need WAF (NLB has no WAF)
  → You need path-based routing (NLB can't read URLs)
  → Lambda targets (NLB doesn't support Lambda)

When ALB is WRONG:
  → You need static IP (ALB IPs change)
  → You need raw TCP passthrough
  → You need UDP (ALB is TCP only)
```

> 💡 **Interview tip:** The **static IP** requirement is the #1 reason companies use NLB even for HTTP workloads. Enterprise security teams often require that external partners whitelist specific IPs. ALB IPs change (they are managed by AWS). NLB gives you Elastic IP addresses per AZ that never change. The workaround for ALB: put NLB in front of ALB (NLB has static IPs, ALB handles routing logic).

---

### Q591 — AWS | Conceptual | Intermediate

> Explain **AWS EBS volume types** — what are gp2, gp3, io1, io2, st1, and sc1? How do you choose the right one?
>
> What is the difference between gp2 and gp3? What is **IOPS burst** and why did AWS introduce gp3?

📁 **Reference:** `nawab312/AWS` — EBS volume types, storage

#### Key Points to Cover in Your Answer:

**EBS volume types:**

| Type | Category | IOPS | Throughput | Use case |
|---|---|---|---|---|
| gp3 | SSD General | 3,000-16,000 | 125-1,000 MB/s | Default for most workloads |
| gp2 | SSD General | 100-16,000 (burst) | 128-250 MB/s | Older, avoid for new |
| io2 Block Express | SSD Provisioned | Up to 256,000 | 4,000 MB/s | Critical DBs, SAP HANA |
| io1 | SSD Provisioned | Up to 64,000 | 1,000 MB/s | I/O-intensive databases |
| st1 | HDD Throughput | 500 | 500 MB/s | Big data, log processing |
| sc1 | HDD Cold | 250 | 250 MB/s | Infrequent access, cheapest |

**gp2 vs gp3 — the most important difference:**
```
gp2 (old):
  → IOPS: 3 IOPS/GB (1TB = 3,000 IOPS baseline)
  → Burst: up to 3,000 IOPS using burst credits
  → When burst credits depleted: drops to baseline (3 IOPS/GB)
  → To get more IOPS: must buy BIGGER volume (waste money)
  → 1,000 IOPS: need 333GB volume minimum
  → Problem: volume size and IOPS are COUPLED

gp3 (new — default since 2021):
  → Baseline: 3,000 IOPS regardless of volume size
  → IOPS and throughput: independently configurable
  → 1,000 IOPS: can use 10GB volume (no need to over-provision size)
  → 20% CHEAPER than gp2
  → No burst credits — consistent performance always
  → Up to 16,000 IOPS, 1,000 MB/s throughput (add-on cost)

Rule: ALWAYS use gp3 for new volumes. Migrate existing gp2 to gp3.
```

**When to use io1/io2:**
```
Use when gp3 max (16,000 IOPS) is not enough
io1: up to 64,000 IOPS (for i3en.x instances only: 256,000)
io2 Block Express: up to 256,000 IOPS, sub-millisecond latency
Use for: Oracle RAC, SAP HANA, high-frequency trading databases

io2 vs io1:
  io2: 99.999% durability (vs io1 99.8-99.9%)
  io2: more IOPS per GB (500 IOPS/GB vs 50 IOPS/GB for io1)
  io2: same price as io1 — always choose io2 over io1
```

> 💡 **Interview tip:** The **gp2 burst depletion** problem is a very common production issue — database performance mysteriously degrades after sustained I/O load. The root cause: gp2 burst credits exhausted, IOPS dropped from 3,000 to 300 (for 100GB volume). The fix: migrate to gp3 (no burst concept, consistent 3,000 IOPS always). Monitor: `BurstBalance` CloudWatch metric for gp2 — alert at <20% before it reaches 0.

---

### Q592 — AWS | Conceptual | Intermediate

> Explain **AWS DynamoDB** — what is it, when would you use it over RDS? What are **partition keys**, **sort keys**, **GSI**, and **LSI**?
>
> What is the difference between **Provisioned** and **On-Demand** capacity modes? What causes **ProvisionedThroughputExceededException**?

📁 **Reference:** `nawab312/AWS` — DynamoDB, NoSQL, capacity planning

#### Key Points to Cover in Your Answer:

**DynamoDB basics:**
```
Fully managed NoSQL database (key-value + document)
  → No schema enforcement (each item can have different attributes)
  → Single-digit millisecond latency at ANY scale
  → Automatic sharding across partitions
  → Multi-AZ by default (synchronous replication across 3 AZs)
  → No JOIN operations (not relational)

When DynamoDB over RDS:
  ✅ Need single-digit ms latency at scale
  ✅ Unpredictable/massive scale (millions of requests/sec)
  ✅ Simple access patterns (get by key, range queries)
  ✅ Serverless / event-driven architectures
  ✅ Session storage, user profiles, IoT data, gaming leaderboards
  ❌ Complex queries, JOINs, transactions → use RDS
  ❌ Reporting / analytics → use Athena/Redshift
```

**Primary key types:**
```
Partition key only (simple PK):
  → Single attribute uniquely identifies item
  → Example: userId (each user is one item)
  → Table: Users{userId(PK), name, email, created_at}

Partition key + Sort key (composite PK):
  → Two attributes together uniquely identify item
  → Partition key: groups related items together
  → Sort key: orders items within a partition + enables range queries
  → Example: OrdersByUser{userId(PK), orderId(SK), amount, status}
  → Query: "all orders for user 123" or "orders for user 123 after Jan 1"
```

**Indexes:**
```
GSI (Global Secondary Index):
  → Different partition key + optional sort key
  → Query data in a completely different access pattern
  → Own read/write capacity (separate from table)
  → Example: query by email (not userId)
  → GSI: email(PK), createdAt(SK)
  → Can add GSI after table creation
  → Up to 20 GSIs per table

LSI (Local Secondary Index):
  → Same partition key, DIFFERENT sort key
  → Query data in same partition by different sort key
  → Example: query orders by status within same user
  → LSI: userId(PK), status(SK)
  → MUST be created at table creation time
  → Limited to 10GB per partition key
  → Up to 5 LSIs per table
```

**Capacity modes:**
```
Provisioned:
  → You specify Read Capacity Units (RCU) and Write Capacity Units (WCU)
  → 1 RCU = 1 strongly consistent read of 4KB/second
  → 1 WCU = 1 write of 1KB/second
  → If you exceed → ProvisionedThroughputExceededException (ThrottlingException)
  → Cheaper for predictable, steady traffic
  → Use Auto Scaling to adjust capacity

On-Demand:
  → Pay per request (no capacity planning)
  → Scales automatically to any traffic level
  → 2-3x more expensive per request than provisioned
  → Use for: unpredictable traffic, new tables, dev/staging
```

> 💡 **Interview tip:** The most important DynamoDB design principle: **design your access patterns FIRST, then design your table**. DynamoDB is the opposite of SQL — you cannot query arbitrary columns efficiently. You must know in advance: "I will query by userId", "I will query by email", "I will query by orderId within a user". Each access pattern needs a key or index. A common mistake: create a DynamoDB table like a SQL table, then struggle because you need complex queries.

---

### Q593 — AWS | Conceptual | Intermediate

> Explain **AWS Lambda** — what is it, how does it work, and what are **cold starts**?
>
> What is the difference between **Reserved Concurrency**, **Provisioned Concurrency**, and the account-level concurrency limit? How do Lambda **layers** work?

📁 **Reference:** `nawab312/AWS` — Lambda, serverless, concurrency

#### Key Points to Cover in Your Answer:

**How Lambda works:**
```
Event → Lambda Service → execution environment → your code → response

Execution environment:
  → Micro-VM with your code + runtime (Python, Node, Java, etc.)
  → Created fresh when no warm environment available
  → Reused for subsequent invocations (warm invocation)

Pricing:
  → $0.20 per 1M requests + $0.0000166667 per GB-second
  → Pay only when running (not when idle)
  → Free tier: 1M requests + 400,000 GB-seconds/month
```

**Cold start problem:**
```
Cold start: time to create new execution environment
  → Download code/layer zips from S3
  → Start the runtime (JVM, Python interpreter)
  → Run your initialization code (outside handler)
  → THEN handle the request

Cold start duration by runtime:
  Python/Node.js: 100-500ms
  Java (JVM):     1-10 seconds (JVM startup is slow!)
  .NET, Go:       50-200ms

When cold starts happen:
  → First invocation of a function
  → After function is idle (no invocations for ~15 minutes)
  → When scaling up (more concurrent executions needed)
  → After deployment (new code version)

Mitigation:
  1. Choose fast runtime (Node.js/Python over Java)
  2. Reduce package size (smaller zip = faster init)
  3. Keep initialization code minimal (outside handler)
  4. Provisioned Concurrency (pre-warm environments)
  5. SnapStart (Java only) — snapshot after init
```

**Concurrency types:**
```
Account-level default:
  → 1,000 concurrent executions per region (soft limit)
  → All functions share this pool

Reserved Concurrency:
  → Guarantee N executions FOR a specific function
  → ALSO CAPS the function to N (throttles above N)
  → Use: protect other functions from one function consuming all concurrency
  → Use: hard cap on expensive functions

Provisioned Concurrency:
  → Pre-initialize N execution environments
  → These environments stay warm (no cold starts for these N invocations)
  → Costs money even when idle
  → Use: latency-critical functions where cold starts are unacceptable

Lambda Layers:
  → Zip files shared across multiple functions
  → Contains: shared libraries, custom runtimes, data
  → Max 5 layers per function
  → Max 250MB total (function + all layers)
  → Layer stored in Lambda service, not in your function zip
  → Use: shared code, SDK packages, binary dependencies
```

> 💡 **Interview tip:** The **Java cold start** is the most common Lambda performance problem — JVM startup takes 5-10 seconds which makes the function unusable for synchronous API calls. Solutions in order of preference: (1) switch to Python/Node.js for new functions, (2) use Lambda SnapStart (JVM checkpoint after init), (3) use Provisioned Concurrency (expensive). For existing Java functions: Lambda SnapStart is the best option — it checkpoints the JVM after initialization and restores from the snapshot instead of re-initializing.

---

### Q594 — AWS | Conceptual | Intermediate

> Explain **AWS IAM** — what are the components: **Users**, **Groups**, **Roles**, and **Policies**? What is the difference between an IAM User and an IAM Role?
>
> What are **inline policies** vs **managed policies** vs **customer managed policies**? When would you use each?

📁 **Reference:** `nawab312/AWS` — IAM fundamentals

#### Key Points to Cover in Your Answer:

**IAM Components:**
```
User:
  → Long-term identity for a person or application
  → Has permanent credentials: password (console) + access keys (API)
  → Belongs to one AWS account
  → Use: human users who need long-term access (developers, admins)
  → Best practice: use SSO/Identity Center instead of IAM users for humans

Group:
  → Collection of IAM users
  → Attach policies to group → all users in group inherit policies
  → Cannot nest groups
  → Use: organize users by team/function (Developers group, Admins group)

Role:
  → Temporary identity that can be ASSUMED by: EC2, Lambda, ECS, other AWS services, users
  → NO permanent credentials — generates temporary tokens via STS
  → Best practice for applications (EC2 Instance Profile, Lambda execution role)
  → Use: service-to-service access (EC2 reading S3, Lambda writing DynamoDB)
  → Cross-account access

Policy:
  → JSON document defining permissions (Allow/Deny actions on resources)
  → Attached to users, groups, or roles
```

**User vs Role — the key distinction:**
```
IAM User:
  → For HUMANS (or legacy applications with long-term access keys)
  → Access keys are long-lived (never expire unless rotated)
  → Access key leaked → permanent security risk until manually rotated
  → Problem: developers commit access keys to GitHub accidentally

IAM Role:
  → For AWS SERVICES and cross-account access
  → Generates temporary credentials (expire in 1-12 hours)
  → No credentials to store → nothing to leak from application code
  → EC2 instance assumes role → gets temp credentials from instance metadata
  → Never hardcode credentials when a role can be used instead
```

**3 policy types:**
```
AWS Managed Policies:
  → Created and maintained by AWS
  → Examples: ReadOnlyAccess, AdministratorAccess, AmazonS3ReadOnlyAccess
  → Updated by AWS when new services/actions added
  → Cannot be modified
  → Use: standard permissions (quick to start, may be overly permissive)

Customer Managed Policies:
  → Created by you, stored in your account
  → Reusable across multiple users/roles
  → Versioned (up to 5 versions)
  → Use: custom permissions that multiple principals need
  → Best practice: create least-privilege customer managed policies

Inline Policies:
  → Directly embedded in ONE user/role/group
  → Cannot be reused
  → Deleted when the user/role is deleted
  → Use: strict one-to-one permission that should never be shared
  → Generally avoid — prefer customer managed for maintainability
```

> 💡 **Interview tip:** "**Always use roles for applications, never access keys**" — this is the single most important IAM best practice. When asked "how does your EC2 application access S3?", the correct answer is always: IAM Role attached as Instance Profile. Never: hardcoded access keys in config files or environment variables. The second best practice: for human access, use AWS IAM Identity Center (SSO) instead of IAM users, so credentials are tied to your corporate identity provider and automatically revoked when employees leave.

---

### Q595 — AWS | Conceptual | Intermediate

> Explain **AWS S3 storage classes** — what are Standard, Intelligent-Tiering, Standard-IA, One Zone-IA, Glacier Instant Retrieval, Glacier Flexible Retrieval, and Glacier Deep Archive?
>
> How do **S3 Lifecycle Policies** automate moving objects between storage classes? What is the minimum storage duration fee for each class?

📁 **Reference:** `nawab312/AWS` — S3 storage classes, lifecycle

#### Key Points to Cover in Your Answer:

**S3 storage classes:**
```
Standard:
  → Frequent access
  → 99.999999999% (11 9s) durability, 99.99% availability
  → No retrieval fee, no minimum storage duration
  → Cost: $0.023/GB/month
  → Use: actively used data, web content, databases

Intelligent-Tiering:
  → Unknown or changing access patterns
  → Automatically moves objects between tiers based on access
  → Small monitoring fee per object ($0.0025/1000 objects)
  → No retrieval fees, no minimum duration
  → Use: data lakes, backups with unpredictable access

Standard-IA (Infrequent Access):
  → Accessed less than once per month
  → Lower storage cost than Standard
  → Per-GB retrieval fee
  → Minimum 30-day storage (charged even if deleted earlier)
  → Use: disaster recovery, backups

One Zone-IA:
  → Same as Standard-IA but stored in only 1 AZ
  → 20% cheaper than Standard-IA
  → Risk: if that AZ fails, data lost
  → Use: easily reproducible data, secondary backups

Glacier Instant Retrieval:
  → Archive data accessed quarterly
  → Millisecond retrieval (like Standard)
  → Minimum 90-day storage
  → Use: medical images, news media archives

Glacier Flexible Retrieval:
  → Archive data rarely accessed
  → Retrieval: Expedited (1-5 min), Standard (3-5 hours), Bulk (5-12 hours)
  → Minimum 90-day storage
  → Use: compliance archives, backups

Glacier Deep Archive:
  → Cheapest ($0.00099/GB — 23x cheaper than Standard!)
  → Retrieval: Standard (12 hours), Bulk (48 hours)
  → Minimum 180-day storage
  → Use: regulatory compliance, 7-year audit log retention
```

**S3 Lifecycle Policy example:**
```xml
<!-- Automatically move logs through tiers -->
<LifecycleConfiguration>
  <Rule>
    <ID>log-lifecycle</ID>
    <Status>Enabled</Status>
    <Prefix>logs/</Prefix>

    <!-- After 30 days: move to Standard-IA -->
    <Transition>
      <Days>30</Days>
      <StorageClass>STANDARD_IA</StorageClass>
    </Transition>

    <!-- After 90 days: move to Glacier Flexible -->
    <Transition>
      <Days>90</Days>
      <StorageClass>GLACIER</StorageClass>
    </Transition>

    <!-- After 7 years: delete (compliance period over) -->
    <Expiration>
      <Days>2555</Days>
    </Expiration>
  </Rule>
</LifecycleConfiguration>
```

> 💡 **Interview tip:** **Intelligent-Tiering** is the safest default for any data with unknown access patterns — it automatically optimizes costs without retrieval fee surprises. The only downside is the per-object monitoring fee, which becomes negligible for large objects but can be expensive for millions of tiny files. For compliance archives (bank statements, audit logs): Glacier Deep Archive + Object Lock (Compliance mode) = cheapest compliant storage available anywhere.

---

### Q596 — AWS | Conceptual | Intermediate

> Explain **AWS RDS Multi-AZ vs Read Replica** — what is the fundamental difference in purpose?
>
> What exactly happens during a Multi-AZ failover? How long does it take? What is the failover mechanism? When would you have BOTH Multi-AZ AND Read Replicas?

📁 **Reference:** `nawab312/AWS` — RDS Multi-AZ, Read Replicas, HA

#### Key Points to Cover in Your Answer:

**Purpose difference:**
```
Multi-AZ:
  → Purpose: HIGH AVAILABILITY (HA) / disaster recovery
  → Synchronous replication to standby in different AZ
  → Standby is NOT accessible (no reads, no writes)
  → Automatic failover when primary fails
  → Same endpoint URL (DNS switches automatically)
  → NOT for performance — standby just sits there waiting

Read Replica:
  → Purpose: PERFORMANCE / read scaling
  → Asynchronous replication from primary
  → Replica IS accessible (read-only queries)
  → No automatic failover (must manually promote)
  → Different endpoint URL (you route reads to it)
  → Can be in different AZ, region, or same AZ
```

**Multi-AZ failover mechanics:**
```
Normal operation:
  Primary (us-east-1a) → synchronous write → Standby (us-east-1b)
  Your app → endpoint: mydb.cluster.us-east-1.rds.amazonaws.com → Primary

Primary fails (hardware, OS crash, AZ failure):
  1. RDS detects failure (via heartbeat monitor)
  2. RDS promotes standby to become new primary
  3. RDS updates DNS record to point to new primary
  4. Total time: 60-120 seconds (DNS propagation included)

Your app:
  → Short connection failures during 60-120 second window
  → After DNS TTL expires: app automatically reconnects to new primary
  → Same endpoint URL — no code change needed

Key: use cluster endpoint (not instance endpoint)
     cluster endpoint always points to current primary
```

**When to have BOTH:**
```
High traffic read-heavy application:
  Primary (write) → Multi-AZ Standby (HA, no reads)
  Primary (write) → Read Replicas (distribute reads, improve performance)

Read Replicas add:
  → Offload SELECT queries from primary
  → Analytics queries don't slow down OLTP
  → Can promote to primary if needed (manual)
  → Can be cross-region (disaster recovery + low-latency reads globally)

Example banking application:
  Primary: handles all writes + critical reads
  Multi-AZ standby: automatic failover in 60-120s if primary fails
  Read Replica 1: reporting queries (don't slow primary)
  Read Replica 2: read-heavy microservice queries
```

> 💡 **Interview tip:** The biggest Multi-AZ misconception: **standby is NOT for reads**. It exists purely as a hot spare for failover. Many people think Multi-AZ improves read performance — it does not. For read performance, use Read Replicas. For HA, use Multi-AZ. For both: use both. Also: Multi-AZ failover is AUTOMATIC (RDS does it), Read Replica promotion is MANUAL (you have to call an API). Aurora changes this — Aurora replicas can serve reads AND be promoted automatically.

---

### Q597 — AWS | Conceptual | Intermediate

> Explain **AWS VPC** — what is it and what are its core components? Walk through how you would design a **3-tier VPC architecture** for a production web application.
>
> What is an **Internet Gateway**, **NAT Gateway**, and **Route Table**? Why do private subnets need a NAT Gateway?

📁 **Reference:** `nawab312/AWS` — VPC, networking fundamentals

#### Key Points to Cover in Your Answer:

**VPC basics:**
```
Virtual Private Cloud:
  → Logically isolated network in AWS cloud
  → You control: IP range (CIDR), subnets, routing, security
  → Default VPC created per region (10.0.0.0/16 or similar)
  → Resources in VPC isolated from other AWS customers
```

**Core VPC components:**
```
CIDR Block:
  → IP address range for your VPC (e.g., 10.0.0.0/16 = 65,536 IPs)

Subnet:
  → Subdivision of VPC CIDR, exists in ONE AZ
  → Public subnet: has route to Internet Gateway
  → Private subnet: no direct internet route

Internet Gateway (IGW):
  → Allows PUBLIC subnet resources to communicate with internet
  → Two-way: inbound AND outbound internet
  → One per VPC, attached to VPC
  → Public EC2 gets public IP + route through IGW

NAT Gateway:
  → Allows PRIVATE subnet resources to reach internet (outbound only)
  → Inbound connections from internet BLOCKED (one-way)
  → Lives in public subnet, has Elastic IP
  → Private EC2 → NAT Gateway → Internet Gateway → Internet
  → Use: downloading packages, calling external APIs from private resources
  → Cost: ~$0.045/hour + $0.045/GB data processed

Route Table:
  → Rules that direct network traffic
  → Public subnet route table: 0.0.0.0/0 → Internet Gateway
  → Private subnet route table: 0.0.0.0/0 → NAT Gateway
  → Local route: 10.0.0.0/16 → local (within VPC, always)
```

**3-tier VPC design:**
```
VPC: 10.0.0.0/16

Public subnets (DMZ): us-east-1a: 10.0.1.0/24, us-east-1b: 10.0.2.0/24
  → Route: 0.0.0.0/0 → Internet Gateway
  → Hosts: ALB, NAT Gateways, Bastion hosts

Private (app) subnets: us-east-1a: 10.0.10.0/24, us-east-1b: 10.0.11.0/24
  → Route: 0.0.0.0/0 → NAT Gateway (in public subnet)
  → Hosts: EC2 app servers, ECS tasks, EKS workers

Private (data) subnets: us-east-1a: 10.0.20.0/24, us-east-1b: 10.0.21.0/24
  → Route: NO route to 0.0.0.0/0 (completely private)
  → Hosts: RDS, ElastiCache, internal databases

Traffic flow:
  Internet → IGW → ALB (public subnet) → App servers (private) → RDS (data)
  App servers → NAT Gateway (public) → IGW → Internet (for outbound)
```

> 💡 **Interview tip:** The most important VPC security principle: **databases should be in data subnets with NO internet route** — not even outbound via NAT. If your RDS can reach the internet (even outbound), that is a security risk (data exfiltration). Data subnets should only have a local VPC route. The app servers reach the database via the VPC's local network — no internet involved.

---

### Q598 — AWS | Conceptual | Intermediate

> Explain **AWS EC2** — what happens when you stop vs terminate an instance? What data is preserved?
>
> What is the difference between **Instance Store** and **EBS-backed** storage? What is an **AMI** and how do you create a golden AMI?

📁 **Reference:** `nawab312/AWS` — EC2 basics, AMI

#### Key Points to Cover in Your Answer:

**Stop vs Terminate:**
```
Stop (only for EBS-backed instances):
  → Instance shuts down (OS shutdown)
  → EBS volumes: PRESERVED (data kept, you pay for storage)
  → Elastic IP: PRESERVED (stays assigned)
  → Public IP: LOST (gets new IP when started again)
  → RAM: LOST
  → Billing: EC2 charge STOPS (still pay for EBS and Elastic IP)
  → Can restart: YES

Terminate:
  → Instance permanently deleted
  → Root EBS volume: DELETED (by default, deleteOnTermination=true)
  → Additional EBS volumes: PRESERVED (deleteOnTermination=false by default)
  → Elastic IP: RELEASED (unless separately associated)
  → Billing: EC2 charge STOPS
  → Can restart: NO — instance is gone forever
```

**Instance Store vs EBS-backed:**
```
Instance Store (ephemeral):
  → Physical disk attached to the HOST machine
  → Extremely fast (NVMe, 1-4 million IOPS)
  → Data LOST when: instance stops, terminates, or host fails
  → Cannot detach and reattach to another instance
  → Not all instance types have it (i3, d3 families)
  → Use: temporary files, buffers, caches, swap

EBS-backed (persistent):
  → Network-attached block storage
  → Survives instance stop/start
  → Can detach from one instance, attach to another
  → Backed by redundant storage (within AZ)
  → Default for most instances (root volume)
  → Use: OS, databases, application data
```

**AMI (Amazon Machine Image):**
```
What it is:
  → Template for creating EC2 instances
  → Contains: OS + software + configuration
  → Includes: EBS snapshot for root volume + permissions + launch permissions

Golden AMI approach:
  1. Start with base AWS AMI (Amazon Linux 2023, Ubuntu 22.04)
  2. Launch EC2 instance
  3. Install + configure: OS patches, agents (CloudWatch, SSM), security hardening,
     application runtime (Java, Python), common tools
  4. Bake AMI: Actions → Image and Templates → Create Image
  5. AMI stored in account → use for all new EC2 launches

Benefits of golden AMI:
  → Consistent, pre-configured instances (no config drift)
  → Fast launch (all software pre-installed, no bootstrap time)
  → Security baseline applied at image level
  → Immutable infrastructure: new AMI for each change (don't patch running instances)

Tools for AMI automation:
  → Packer (HashiCorp): define AMI as code
  → EC2 Image Builder: AWS-native AMI pipeline
```

> 💡 **Interview tip:** **Instance Store data is permanently lost when the instance stops or terminates** — this catches many people off-guard. Even a controlled stop (not a crash) causes data loss on instance store. This is why instance store is ONLY appropriate for data you can reconstruct (caches, temp files, buffers) — never for data you need to keep. For Kubernetes nodes using instance store (i3 family for high IOPS), ensure no critical pod data is written to local storage.

---

### Q599 — AWS | Conceptual | Intermediate

> Explain **AWS CloudWatch** — what can it monitor and what are the main features?
>
> What is the difference between a **CloudWatch Metric**, **Metric Filter**, **Log Group**, **Dashboard**, and **Alarm**? What is a **Dimension** in CloudWatch metrics?

📁 **Reference:** `nawab312/AWS` — CloudWatch fundamentals

#### Key Points to Cover in Your Answer:

**CloudWatch main capabilities:**
```
Metrics:    Numerical time-series data (CPU%, request count, error rate)
Logs:       Text log data from any source (EC2, Lambda, ECS, apps)
Alarms:     Trigger actions when metrics cross thresholds
Dashboards: Visualize metrics + alarms on a custom dashboard
Events:     EventBridge (schedule/respond to events in AWS)
Insights:   Query and analyze logs (CloudWatch Logs Insights)
Synthetics: Canary scripts for synthetic monitoring
Container Insights: K8s/ECS container-level metrics
X-Ray:      Distributed tracing (technically separate service)
```

**Core concepts:**
```
Metric:
  → Time-series data point
  → Each metric: Namespace + Metric Name + Dimensions + Timestamp + Value
  → Example: AWS/EC2 | CPUUtilization | InstanceId=i-123 | 14:05:00 | 45.3%
  → Free tier: basic metrics every 5 minutes
  → Detailed monitoring: every 1 minute (additional cost)

Dimension:
  → Key-value pair that identifies a specific metric instance
  → CPUUtilization for WHICH instance? Dimension: InstanceId=i-12345
  → CPUUtilization for WHICH auto-scaling group? Dimension: AutoScalingGroupName=web-asg
  → Same metric, different dimensions = different time series

Namespace:
  → Container for related metrics
  → AWS services use: AWS/EC2, AWS/RDS, AWS/Lambda
  → Custom metrics: use your own namespace (MyApp/Payments)

Log Group:
  → Container for log streams from same source
  → /aws/lambda/payment-function → all logs from that Lambda
  → /aws/ecs/payment-service → all logs from ECS tasks
  → Retention policy set per log group (1 day to 10 years)

Log Stream:
  → Sequence of log events from ONE source
  → One Lambda execution environment = one log stream
  → Multiple log streams in a log group

Metric Filter:
  → Extracts metric data FROM log text
  → Pattern: "[status=5*]" → count 500 errors per minute
  → Creates CloudWatch metric from log data
  → Example: count "ERROR" occurrences per minute from application logs
  → Alert on that metric → paged when error rate spikes
```

**CloudWatch Alarm states:**
```
OK:                   metric within threshold
ALARM:                metric breached threshold → action triggered
INSUFFICIENT_DATA:    not enough data points (new resource, first minutes)

Alarm actions:
  → SNS notification (email, SMS, PagerDuty)
  → EC2 action (stop, terminate, reboot, recover)
  → Auto Scaling action (scale up/down)
  → Systems Manager OpsItem
```

> 💡 **Interview tip:** **Metric Filters** are one of the most underused CloudWatch features — they let you create custom metrics from existing logs without any application changes. Common use case: count HTTP 5xx errors from Nginx access logs, count exceptions from application logs, count failed login attempts from auth logs. The metric costs nothing to create (you only pay for custom metric data points), and you can then alarm on it and use it in dashboards. This is often the fastest way to add monitoring to an existing application.

---

### Q600 — AWS | Conceptual | Intermediate

> Explain **AWS Kinesis** — what are the differences between **Kinesis Data Streams**, **Kinesis Data Firehose**, and **Kinesis Data Analytics**?
>
> What is a **shard** in Kinesis Data Streams? What happens when a shard is overwhelmed? What is the difference between **Enhanced Fan-Out** and standard consumer?

📁 **Reference:** `nawab312/AWS` — Kinesis, streaming data

#### Key Points to Cover in Your Answer:

**3 Kinesis services:**
```
Kinesis Data Streams:
  → Real-time streaming data ingestion
  → YOU manage: shards, consumers, retention (1-365 days)
  → Multiple consumers can read same data simultaneously
  → Sub-second latency
  → Use: real-time analytics, event-driven processing, custom consumers

Kinesis Data Firehose:
  → Fully managed — no shard management
  → Delivers data to: S3, Redshift, OpenSearch, HTTP endpoint
  → Near-real-time (60-second buffer minimum)
  → Can transform data with Lambda before delivery
  → Use: log ingestion to S3/OpenSearch, simple ETL pipelines
  → Simplest option when you just need to store streaming data

Kinesis Data Analytics:
  → Run SQL or Apache Flink queries on streaming data
  → Processes data from Kinesis Data Streams or Firehose
  → Use: real-time aggregations, anomaly detection, alerts
```

**Kinesis Data Streams — Shards:**
```
Shard:
  → Basic unit of capacity in Kinesis Data Streams
  → One shard: 1 MB/s ingest, 2 MB/s read, 1,000 records/sec
  → Partition key hashed → determines which shard receives record
  → Hot shard: one partition key routes too much data to one shard
    → Exceeds 1MB/s → ProvisionedThroughputExceededException
    → Fix: use more unique partition keys (userId, not state)

Scaling:
  → Add shards (shard splitting): double capacity
  → Remove shards (shard merging): halve capacity
  → On-Demand mode (2023+): auto-scales like Firehose
```

**Standard vs Enhanced Fan-Out:**
```
Standard consumer (GetRecords API):
  → Polling: consumer asks "give me records" every 200ms
  → 2 MB/s SHARED across all standard consumers
  → If 5 consumers: each gets 400 KB/s
  → Latency: ~200ms

Enhanced Fan-Out (SubscribeToShard API):
  → Push model: Kinesis PUSHES to consumer as soon as available
  → 2 MB/s PER consumer (dedicated throughput)
  → Up to 20 consumers with dedicated throughput
  → Latency: ~70ms
  → Additional cost: $0.015/shard-hour
  → Use: multiple consumers needing full throughput independently
```

> 💡 **Interview tip:** The **hot shard problem** is the most common Kinesis production issue. If you use the same partition key (e.g., partition key = "all") for all records, every record goes to the same shard, and you hit the 1MB/s limit almost immediately. The fix: use a high-cardinality partition key (userId, deviceId, transactionId). A good partition key distributes load evenly across all shards.

---

### Q601 — AWS | Conceptual | Intermediate

> Explain **AWS Route53 DNS record types** — what are A, AAAA, CNAME, ALIAS, MX, TXT, and NS records? What is the difference between CNAME and ALIAS?
>
> What is **Route53 Resolver** and what problem does it solve for hybrid architectures?

📁 **Reference:** `nawab312/AWS` — Route53, DNS record types

#### Key Points to Cover in Your Answer:

**DNS record types:**
```
A record:
  → Maps hostname → IPv4 address
  → api.company.com → 54.23.45.67
  → Most common record type

AAAA record:
  → Maps hostname → IPv6 address
  → api.company.com → 2001:db8::1

CNAME (Canonical Name):
  → Maps hostname → another hostname (alias)
  → www.company.com → company.com
  → CANNOT be used at zone apex (company.com itself)
  → CANNOT point to an IP address
  → Each DNS lookup requires extra resolution (slight latency)

ALIAS:
  → AWS-specific record type (not standard DNS)
  → Maps hostname → AWS resource DNS name
  → CAN be used at zone apex (company.com → alb.amazonaws.com)
  → No extra DNS lookup (resolved inside Route53 to actual IPs)
  → FREE (no charge for alias queries unlike CNAME)
  → Works with: ALB, NLB, CloudFront, S3 website, API Gateway

MX record:
  → Mail exchange → specifies email servers
  → Used for email routing (not relevant for most DevOps)

TXT record:
  → Text records for domain verification, SPF, DKIM
  → Google/AWS verify domain ownership via TXT records
  → "_amazonses.company.com" → verify SES domain

NS record:
  → Name Server records → which servers are authoritative for domain
  → company.com → ns1.route53.amazonaws.com, ns2.route53.amazonaws.com
  → Set at domain registrar to point to Route53

SOA record:
  → Start of Authority → admin info about the zone
  → Created automatically by Route53
```

**CNAME vs ALIAS — the critical difference:**
```
CNAME:
  company.com → CNAME → www.company.com  ← INVALID (zone apex restriction)
  api.company.com → CNAME → my-alb.us-east-1.elb.amazonaws.com  ← valid but charged

ALIAS:
  company.com → ALIAS → my-alb.us-east-1.elb.amazonaws.com  ← VALID (apex works!)
  api.company.com → ALIAS → my-alb.us-east-1.elb.amazonaws.com  ← free, faster

Use ALIAS when pointing to AWS resources — always.
Use CNAME when pointing to non-AWS external hostnames.
```

**Route53 Resolver:**
```
Problem: EC2 in VPC needs to resolve on-premises hostnames
         (internal company domains like corp.company.internal)
         Also: on-premises servers need to resolve AWS resource names

Route53 Resolver:
  → DNS service at VPC+2 IP address (e.g., 10.0.0.2 for 10.0.0.0/16 VPC)
  → Resolver Inbound Endpoints: on-prem DNS → AWS (query AWS names from on-prem)
  → Resolver Outbound Endpoints: AWS → on-prem DNS (resolve internal names from AWS)
  → Forwarding rules: company.internal → forward to on-prem DNS 192.168.1.53

Use case:
  EC2 queries corp.company.internal (internal HR system)
  → Resolver outbound endpoint → on-premises DNS → gets IP → EC2 connects
```

> 💡 **Interview tip:** The **ALIAS vs CNAME** distinction is frequently tested in AWS certification-style questions. Remember: ALIAS is **AWS-only**, **free**, **works at zone apex**, and **automatically tracks AWS resource IP changes**. CNAME is **standard DNS**, **charged per query**, **cannot be at zone apex**, and requires an extra DNS lookup. For any AWS resource (ALB, CloudFront, S3 website), always use ALIAS.

---

## SECTION 3 — Docker Fundamentals

---

### Q602 — Docker | Conceptual | Beginner-Intermediate

> Explain the difference between **Containers and Virtual Machines**. How do containers achieve isolation without a full OS?
>
> What Linux kernel features do containers use? What is the **Docker daemon** and how does it work?

📁 **Reference:** `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md`

#### Key Points to Cover in Your Answer:

**VM vs Container:**
```
Virtual Machine:
  → Full OS inside a VM (kernel + libraries + app)
  → Hypervisor: Type 1 (bare metal) or Type 2 (on OS)
  → Hardware emulated for each VM
  → Strong isolation (separate kernel)
  → Size: GBs (full OS)
  → Startup: minutes (boot full OS)
  → Use: strong isolation, different OS, legacy apps

Container:
  → Shares HOST kernel — no separate OS
  → Just: app + libraries + dependencies
  → Isolation via Linux kernel features (namespaces + cgroups)
  → NOT true isolation — if kernel has vuln, all containers affected
  → Size: MBs (no OS)
  → Startup: milliseconds (no OS boot)
  → Use: microservices, CI/CD, cloud-native apps

Key: containers share the HOST kernel
     VMs have their OWN kernel
```

**Linux kernel features containers use:**
```
Namespaces (isolation — "what can it see"):
  pid:  container has its own process tree (PID 1 = app, not init)
  net:  container has its own network interfaces, routing table
  mnt:  container has its own filesystem view (different root /)
  uts:  container has its own hostname
  ipc:  container has its own inter-process communication
  user: container has its own user/group IDs (user namespace)

cgroups (resource control — "how much can it use"):
  cpu:    limit CPU percentage/shares
  memory: limit RAM and swap usage
  blkio:  limit disk I/O bandwidth
  network: limit network bandwidth

OverlayFS (union filesystem):
  → Container filesystem built from image layers
  → Read-only base layers + read-write container layer
  → Changes stored in top writable layer only
```

**Docker daemon architecture:**
```
Docker Client (docker CLI):
  → You run: docker run, docker build, docker pull
  → Sends commands via REST API to Docker daemon

Docker Daemon (dockerd):
  → Background service on host
  → Manages: images, containers, networks, volumes
  → Talks to container runtime (containerd) via gRPC

containerd:
  → Container runtime (OCI-compliant)
  → Actually creates/manages containers
  → Talks to runc (low-level container runtime)

runc:
  → OCI runtime spec implementation
  → Calls kernel APIs (namespaces, cgroups)
  → Creates actual container processes

Flow: docker CLI → REST API → dockerd → containerd → runc → kernel namespaces/cgroups
```

> 💡 **Interview tip:** The most important container security implication: **containers share the host kernel**. If a container escapes its namespace isolation (container escape vulnerability), it can potentially access the host system. This is why "containers are not VMs" from a security perspective — privileged containers essentially have root access to the host. For stronger isolation: use gVisor (user-space kernel), Kata containers (lightweight VMs), or Firecracker (AWS Lambda's approach).

---

### Q603 — Docker | Conceptual | Beginner-Intermediate

> Walk through the key **Dockerfile instructions** — FROM, RUN, COPY, ADD, ENV, ARG, WORKDIR, EXPOSE, USER, VOLUME, ENTRYPOINT, CMD.
>
> What is the difference between COPY and ADD? What is the difference between ENV and ARG? Write a well-structured production Dockerfile.

📁 **Reference:** `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md`

#### Key Points to Cover in Your Answer:

**Key instructions:**
```
FROM base_image:tag
  → Starting point — which base image to build on
  → Always specify exact tag (not latest)
  → In multi-stage: FROM image AS stage-name

RUN command
  → Execute command during BUILD time
  → Creates a new layer
  → Chain with && to reduce layers and clean up:
    RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

COPY src dest
  → Copy files FROM build context (your machine) TO image
  → Preferred over ADD for simple file copies

ADD src dest
  → Like COPY but also:
  → Extracts tar files automatically (COPY does not)
  → Can fetch URLs (discouraged — prefer RUN curl instead)
  → Use COPY unless you need tar extraction

ENV KEY=VALUE
  → Set environment variable available at BUILD time AND RUNTIME
  → Visible in docker inspect → avoid for secrets

ARG NAME=default
  → Build-time variable only (NOT available at runtime)
  → Pass with: docker build --build-arg NAME=value
  → Use for: build-time customization (versions, config)
  → Safer than ENV for build-time secrets (not in final image metadata)

WORKDIR /path
  → Set working directory for subsequent instructions
  → Creates directory if doesn't exist
  → Use instead of: RUN cd /path

EXPOSE port
  → Documents which port the container listens on
  → Does NOT actually publish the port (just documentation)
  → Publish with: docker run -p 8080:8080

USER username
  → Switch to non-root user for subsequent instructions and container runtime
  → Security best practice: never run as root

VOLUME /path
  → Creates a mount point for external volumes
  → Data at this path persists beyond container lifecycle

ENTRYPOINT ["executable", "arg"]
  → Main process — always runs (hard to override)
  → Use exec form (array) not shell form (string)

CMD ["arg1", "arg2"]
  → Default arguments to ENTRYPOINT or default command
  → Easily overridden at runtime
```

**Production Dockerfile example:**
```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /build
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Stage 2: Runtime
FROM node:20-alpine AS runtime

# Install security updates
RUN apk upgrade --no-cache

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy only what's needed from builder
COPY --from=builder /build/node_modules ./node_modules
COPY --chown=appuser:appgroup src/ ./src/

# Runtime configuration via env
ENV NODE_ENV=production \
    PORT=3000

# Switch to non-root user
USER appuser

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -q -O /dev/null http://localhost:3000/health || exit 1

# Exec form (not shell form) — required for proper signal handling
ENTRYPOINT ["node"]
CMD ["src/server.js"]
```

> 💡 **Interview tip:** **COPY vs ADD** — always prefer COPY unless you specifically need tar extraction or URL fetching. ADD with URLs is non-deterministic (content can change), making builds non-reproducible. **ENV vs ARG** — ARG values are NOT persisted in the final image (not visible in docker inspect), making ARG safer for build-time secrets. However, ARG values ARE visible in the build history (`docker history`) — for true secrets, use Docker BuildKit secrets instead.

---

### Q604 — Docker | Conceptual | Beginner-Intermediate

> Explain **Docker image layers** and the **union filesystem**. How are layers created and how does layer caching work?
>
> What is the difference between an **image** and a **container**? What is the **container layer**?

📁 **Reference:** `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md`

#### Key Points to Cover in Your Answer:

**Image vs Container:**
```
Image:
  → Read-only template — blueprint for containers
  → Stack of immutable layers
  → Stored on disk/registry
  → Like a class definition in OOP

Container:
  → Running instance of an image
  → Image layers (read-only) + thin writable container layer on top
  → Like an object instantiated from a class
  → Multiple containers can run from same image
  → Isolated from each other (separate namespaces)
```

**Layer architecture:**
```
ubuntu:22.04 base layer         [read-only]  100MB
+ apt-get install nginx         [read-only]  45MB
+ COPY nginx.conf               [read-only]  1KB
+ EXPOSE 80                     [read-only]  0B (metadata only)
= nginx:custom IMAGE

Run container from image:
ubuntu:22.04 base layer         [read-only]  shared with all containers
nginx packages layer            [read-only]  shared with all containers
nginx.conf layer                [read-only]  shared with all containers
Container writable layer        [READ/WRITE] unique per container (thin)

Container writes a file:
  → Copy-on-Write: file copied from read-only layer to writable layer
  → Modification stored in writable layer only
  → Original image layer unchanged

Container deleted:
  → Writable layer deleted (all changes lost)
  → Base image layers remain (reused by other containers)
```

**Layer caching:**
```
Docker checks cache for each instruction:
  → Same base image + same instruction + same context → use cache
  → Any change invalidates cache for that instruction AND ALL SUBSEQUENT

Order matters — stable → frequently changing:
FROM node:20-alpine              # layer 1: rarely changes, always cached
WORKDIR /app                     # layer 2: rarely changes
COPY package*.json ./            # layer 3: changes only when deps change
RUN npm ci                       # layer 4: heavy! only runs when layer 3 changes
COPY src/ ./                     # layer 5: changes with every code edit
CMD ["node", "src/server.js"]    # layer 6: rarely changes

Bad order (cache busted on every code change):
COPY . .                         # copies everything including src
RUN npm ci                       # ALWAYS runs (COPY changed)
```

> 💡 **Interview tip:** **OverlayFS (overlay2)** is the default Docker storage driver on modern Linux systems. It implements the union filesystem using three directories: lower (image layers, read-only), upper (container layer, read-write), and merged (what the container sees — the union of lower + upper). When you write a file that exists in a lower layer, the file is first copied to upper (copy-on-write), then modified. This is why container storage isn't suitable for databases — every write involves a copy overhead.

---

## SECTION 4 — Linux Fundamentals

---

### Q605 — Linux | Conceptual | Beginner-Intermediate

> Explain **Linux file permissions** — what do the read, write, execute bits mean for files vs directories? How does `chmod` work with octal notation?
>
> What are **special permission bits** (setuid, setgid, sticky bit)? What is `umask`?

📁 **Reference:** `nawab312/DSA` → `Linux` — file permissions

#### Key Points to Cover in Your Answer:

**Permission structure:**
```
-rwxr-xr-- 1 user group 1234 Jan 15 file.sh
│└──┬──┘└──┬──┘└──┬──┘
│   │      │      └── other permissions (r--)
│   │      └────────── group permissions (r-x)
│   └───────────────── user/owner permissions (rwx)
└───────────────────── file type (- file, d dir, l symlink)

r = read    (4)   = can read file / list directory contents
w = write   (2)   = can modify file / create/delete files in dir
x = execute (1)   = can run file / traverse directory (cd into it)
- = no permission (0)
```

**chmod — change permissions:**
```
Octal notation (most common):
  chmod 755 file.sh
  7 = rwx (4+2+1) = owner can read+write+execute
  5 = r-x (4+0+1) = group can read+execute
  5 = r-x (4+0+1) = others can read+execute

Common octal patterns:
  777 = rwxrwxrwx (everyone has full access — dangerous!)
  755 = rwxr-xr-x (scripts/programs)
  644 = rw-r--r-- (regular files)
  600 = rw------- (private files, SSH private key)
  400 = r-------- (read-only by owner, no modification)

Symbolic notation:
  chmod u+x file     # add execute for user
  chmod g-w file     # remove write for group
  chmod o=r file     # set others to read only
  chmod a+r file     # add read for all (a = all)
  chmod +x script.sh # add execute for all
```

**Special bits:**
```
SetUID (4000):
  chmod 4755 /usr/bin/passwd
  → When executed: runs with FILE OWNER's permissions (not runner's)
  → passwd runs as root (even when regular user runs it)
  → How: regular user can change their own password (writes to /etc/shadow)
  → Security risk: SetUID on malicious binary → root access

SetGID (2000):
  chmod 2755 /shared-dir
  → Directory: files created inside inherit directory's GROUP
  → Team shares directory → all files owned by team group → correct permissions

Sticky bit (1000):
  chmod 1777 /tmp
  → Directory: only file OWNER (or root) can delete their own files
  → /tmp is world-writable but you can't delete others' files
  → Use: shared directories (tmp, uploads)

umask:
  → Default permission mask applied when creating files/dirs
  → umask 022: new files get 644 (666-022), new dirs get 755 (777-022)
  → umask 077: private — files get 600, dirs get 700
  → View: umask; Set: umask 027
```

> 💡 **Interview tip:** The **execute bit on directories** is the most commonly misunderstood permission. Execute on a directory means "can traverse" (can `cd` into it or access files inside). Without execute, you cannot enter a directory even if you have read permission (can list contents but cannot access files). This is why directories usually need 755 — group/others need `x` to use the directory even if they should not write to it.

---

### Q606 — Linux | Conceptual | Beginner-Intermediate

> Explain **Linux processes** — what is a process, what is a thread? What are the key process states?
>
> Explain the **fork-exec model** — how are new processes created? What is a **zombie process** and an **orphan process**?

📁 **Reference:** `nawab312/DSA` → `Linux` — process management

#### Key Points to Cover in Your Answer:

**Process basics:**
```
Process: program in execution
  → Has its own: PID, memory space, file descriptors, environment
  → Created by fork() system call
  → Every process has a parent (except PID 1 = init/systemd)

Thread: lightweight process within a process
  → Shares: memory space, file descriptors, code with other threads
  → Has its own: stack, registers, program counter
  → Cheaper to create than processes
  → Java/Go/Python create threads for concurrent execution
```

**Process states:**
```
R (Running/Runnable):
  → Currently executing on CPU or waiting in run queue
  → "I'm ready to run"

S (Sleeping — interruptible):
  → Waiting for event (I/O, timer, signal)
  → Can be woken by signal
  → Most processes in this state (waiting for I/O)

D (Sleeping — uninterruptible):
  → Waiting for I/O that CANNOT be interrupted
  → Even SIGKILL won't kill it!
  → Usually: waiting for disk I/O, NFS, kernel operations
  → Danger: D state too long = system hung

Z (Zombie):
  → Process finished but parent hasn't read exit status (via wait())
  → Process table entry still exists, no resources consumed
  → Just an entry in process table
  → Parent must call wait() to reap zombie
  → Many zombies = parent bug (not calling wait)

T (Stopped):
  → Stopped by signal (SIGSTOP, SIGTSTP from Ctrl+Z)
  → Can be resumed with SIGCONT

I (Idle kernel thread):
  → Idle kernel threads (Linux 4.14+)
```

**Fork-exec model:**
```
How shell runs "ls" command:

1. fork():
   → Creates COPY of parent process
   → Child is exact duplicate: same code, data, environment, file descriptors
   → fork() returns 0 in child, child's PID in parent
   → Now two identical processes running

2. exec() in child:
   → Replaces child process memory with new program (ls)
   → New code/data loaded from disk
   → Execution starts at new program's main()
   → PID stays the same (still the child process)

Parent:
   → Can continue running OR wait() for child to finish
   → wait() = blocking until child exits, gets exit code

Why fork then exec?
   → Between fork() and exec(): can set up file descriptors, env vars
   → Shell does: fd[1] = pipe before exec → new process inherits pipe
   → Foundation of Unix pipes: ls | grep foo
```

**Zombie and orphan:**
```
Zombie process:
  → Child finishes, parent doesn't call wait()
  → Zombie entry stays in process table
  → Shows as "Z" or "<defunct>" in ps
  → Fix: fix parent to call wait() OR kill parent (init reaps zombies)

Orphan process:
  → Parent dies before child
  → Child is "adopted" by init (PID 1)
  → init calls wait() properly → no zombie problem
  → Intentional orphans: daemonize (double fork trick)
```

> 💡 **Interview tip:** Zombie processes don't consume CPU or memory — they are just a process table entry. The only risk: if thousands accumulate, process table fills up and no new processes can be created. The fix is not to kill zombies (they have no resources) but to fix or kill their parent. In containers: this is why `tini` or `--init` is used — PID 1 in a container must properly reap zombie children, which normal application processes don't do.

---

### Q607 — Linux | Conceptual | Beginner-Intermediate

> Explain **grep, sed, and awk** — when would you use each? Provide practical examples for each tool.
>
> Write commands to: extract all IP addresses from a log file, replace all occurrences of a string in a file, calculate average response time from access logs.

📁 **Reference:** `nawab312/DSA` → `Linux` — text processing

#### Key Points to Cover in Your Answer:

**grep — search and filter:**
```bash
# What it does: search text for patterns, output matching lines

# Basic search
grep "ERROR" application.log               # find lines with ERROR
grep -i "error" application.log            # case insensitive
grep -v "INFO" application.log             # invert: lines NOT containing INFO
grep -n "ERROR" application.log            # show line numbers
grep -c "ERROR" application.log            # count matching lines
grep -r "TODO" /src/                       # recursive directory search

# Extended regex
grep -E "ERROR|WARN" application.log       # multiple patterns (OR)
grep -E "^[0-9]{4}-[0-9]{2}-[0-9]{2}" log # starts with date

# Extract IPs from log file
grep -oE "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" access.log | sort | uniq -c | sort -rn
# -o: print only matching part (not whole line)
# sort | uniq -c: count occurrences
# sort -rn: sort by count descending
```

**sed — stream editor, transform text:**
```bash
# What it does: edit text streams (substitute, delete, insert)

# Substitute (most common use)
sed 's/old/new/' file               # replace first occurrence per line
sed 's/old/new/g' file              # replace ALL occurrences (global)
sed 's/localhost/prod-db.com/g' config.yaml  # update config

# In-place editing
sed -i 's/localhost/prod-db.com/g' config.yaml  # edit file directly
sed -i.bak 's/old/new/g' file       # edit file, keep backup as file.bak

# Delete lines
sed '/^#/d' config.conf             # delete comment lines
sed '/^$/d' file                    # delete empty lines

# Print specific lines
sed -n '10,20p' file                # print lines 10-20

# Multiple expressions
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file
```

**awk — column-based text processing:**
```bash
# What it does: process structured text field by field

# Fields: $1=first field, $2=second, $NF=last field
# Default separator: whitespace

# Print specific column
awk '{print $1}' file               # print first column
awk '{print $1, $3}' file           # print columns 1 and 3

# Custom separator
awk -F: '{print $1}' /etc/passwd    # colon-separated, print usernames
awk -F, '{print $2}' data.csv       # CSV, print second column

# Filter and process
awk '$3 > 500 {print $1, $3}' access.log   # lines where col3 > 500

# Calculate average response time from access log
# Log format: ip - - [date] "method path" status size response_time
awk '{sum+=$NF; count++} END {print "avg:", sum/count "ms"}' access.log

# Count by status code
awk '{print $9}' access.log | sort | uniq -c | sort -rn
# $9 = HTTP status code in common log format

# Sum a column
awk '{sum+=$5} END {print "Total bytes:", sum}' access.log
```

> 💡 **Interview tip:** Rule of thumb: **grep for filtering, sed for substitution, awk for structured data processing**. 90% of text processing needs are: `grep pattern file` (find it), `sed 's/old/new/g' file` (change it), `awk '{print $N}' file` (extract column N). For complex processing: awk is essentially a programming language with loops and math. Combining these with pipes is the Linux power user's toolkit.

---

### Q608 — Linux | Conceptual | Beginner-Intermediate

> Explain **SSH key-based authentication** — how does it work? What files are involved?
>
> How do you set up **passwordless SSH**? What is `ssh-agent` and `ssh-keyscan`? What should be in a production `/etc/ssh/sshd_config`?

📁 **Reference:** `nawab312/DSA` → `Linux` — SSH configuration

#### Key Points to Cover in Your Answer:

**SSH key-based authentication flow:**
```
Key pair:
  Private key: ~/.ssh/id_rsa (NEVER share, stay on your machine)
  Public key:  ~/.ssh/id_rsa.pub (place on servers you want to access)

Authentication flow:
  1. Client: "I want to connect as user ec2-user"
  2. Server: checks ~/.ssh/authorized_keys for public key
  3. Server: generates random challenge, encrypts with public key
  4. Server: sends encrypted challenge to client
  5. Client: decrypts challenge with PRIVATE key
  6. Client: sends response to server
  7. Server: verifies response → authentication success
  → Private key never leaves the client — challenge-response proves possession
```

**Setup passwordless SSH:**
```bash
# Step 1: Generate key pair (once per machine)
ssh-keygen -t ed25519 -C "siddharth@company.com"
# ed25519: modern, more secure than RSA (but RSA 4096 also fine)
# -C: comment for identification
# Files: ~/.ssh/id_ed25519 (private) + ~/.ssh/id_ed25519.pub (public)

# Step 2: Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
# OR manually: cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# Step 3: Set correct permissions (SSH is strict about this)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519      # private key MUST be 600

# Step 4: Test
ssh -i ~/.ssh/id_ed25519 user@server
```

**ssh-agent:**
```bash
# Problem: passphrase protected key prompts every connection
# ssh-agent: holds decrypted keys in memory for the session

eval $(ssh-agent)               # start agent, export env vars
ssh-add ~/.ssh/id_ed25519       # add key (prompts for passphrase once)
ssh user@server                 # no passphrase prompt (agent handles it)
ssh-add -l                      # list loaded keys
ssh-add -D                      # remove all keys from agent
```

**Production sshd_config hardening:**
```bash
# /etc/ssh/sshd_config — server configuration
PermitRootLogin no              # never allow root SSH login
PasswordAuthentication no       # require key-based auth only
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PermitEmptyPasswords no
ChallengeResponseAuthentication no
X11Forwarding no                # disable GUI forwarding
MaxAuthTries 3                  # limit brute force attempts
ClientAliveInterval 300         # disconnect idle sessions after 5 min
ClientAliveCountMax 2
AllowUsers ec2-user deploy      # whitelist which users can SSH
Protocol 2                      # only SSHv2 (v1 is insecure)

# After changing:
sudo systemctl reload sshd      # reload without interrupting connections
```

> 💡 **Interview tip:** **Disabling PasswordAuthentication is the single most impactful SSH security measure**. SSH brute force attacks are constant on any internet-facing server — with password auth enabled, attackers can try millions of passwords. With key-only auth, there is nothing to brute force. In AWS, all EC2 instances use key-based auth by default. For bastion hosts: also restrict with `AllowUsers` and use IP-based security groups to limit who can even attempt connections.

---

## SECTION 5 — Terraform Fundamentals

---

### Q609 — Terraform | Conceptual | Beginner-Intermediate

> Explain the **Terraform state file** — what is it, what does it contain, and why does Terraform need it?
>
> What are the risks of the state file? What should you NEVER do with state?

📁 **Reference:** `nawab312/Terraform` — state fundamentals

#### Key Points to Cover in Your Answer:

**What the state file is:**
```
terraform.tfstate: JSON file mapping your Terraform resources
                   to real infrastructure objects in your cloud

Contains:
  → All managed resources with their attributes (IDs, ARNs, IPs)
  → Dependencies between resources
  → Provider metadata
  → Output values

Example state entry:
{
  "type": "aws_instance",
  "name": "web",
  "instances": [{
    "attributes": {
      "id": "i-0a1b2c3d4e5f",
      "instance_type": "t3.medium",
      "public_ip": "54.23.45.67",
      "tags": {"Name": "web-server"}
    }
  }]
}
```

**Why Terraform needs state:**
```
Problem: Terraform CANNOT query all attributes from cloud API
  → AMI used to create EC2 is not returned by describe-instances
  → Output values of one resource needed by another resource
  → Performance: can't check 500 resources on every plan (too slow)

State solves this:
  → Maps config resource names → real resource IDs
  → "aws_instance.web" → "i-0a1b2c3d4e5f"
  → terraform plan reads state → compares to .tf files → finds differences
  → Without state: Terraform doesn't know what it already created

State is also a lock:
  → Prevents two people running apply simultaneously (corrupting resources)
```

**Risks and rules:**
```
Risk 1: State contains sensitive data (passwords, private IPs)
  → Encrypt state in S3 with SSE-KMS
  → Never commit state to Git (use .gitignore for *.tfstate)

Risk 2: State corruption from concurrent runs
  → Use DynamoDB locking with S3 backend
  → Never run terraform apply from multiple terminals simultaneously

Risk 3: State diverges from reality (drift)
  → Manual changes in console → state out of sync
  → Fix: terraform refresh (read actual state) or terraform import

NEVER do:
  ❌ Never manually edit tfstate (use terraform state commands)
  ❌ Never commit tfstate to Git
  ❌ Never delete tfstate (you lose tracking of all resources)
  ❌ Never share tfstate files via email/Slack (sensitive data)
```

> 💡 **Interview tip:** "State is Terraform's source of truth about what it manages." Without state, Terraform cannot determine what already exists vs what needs to be created. This is why remote state (S3 + DynamoDB) is non-negotiable for teams — local state files cause conflicts when multiple engineers run Terraform. The most common interview follow-up: "What happens if you lose the state file?" Answer: you don't lose the infrastructure, but Terraform no longer knows it manages it. Fix: `terraform import` each resource back into state.

---

### Q610 — Terraform | Conceptual | Intermediate

> Explain **`count` vs `for_each`** in Terraform — what is the difference and when should you use each?
>
> What is the famous **count index shifting problem** and how does `for_each` solve it?

📁 **Reference:** `nawab312/Terraform` — count, for_each, loops

#### Key Points to Cover in Your Answer:

**count — create N copies:**
```hcl
# Create 3 identical EC2 instances
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-12345"
  instance_type = "t3.micro"
  tags = {
    Name = "web-${count.index}"   # web-0, web-1, web-2
  }
}

# Reference individual instance
aws_instance.web[0].id   # first instance
aws_instance.web[*].id   # all instance IDs
```

**for_each — create from map or set:**
```hcl
# Create instances from a map (better for named resources)
locals {
  environments = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.medium"
  }
}

resource "aws_instance" "env" {
  for_each      = local.environments
  ami           = "ami-12345"
  instance_type = each.value    # t3.micro, t3.small, t3.medium
  tags = {
    Name = "web-${each.key}"    # web-dev, web-staging, web-prod
  }
}

# Reference individual instance
aws_instance.env["dev"].id       # dev instance
aws_instance.env["prod"].id      # prod instance
```

**The count index shifting problem:**
```hcl
# With count - using a list:
variable "names" {
  default = ["alice", "bob", "charlie"]
}
resource "aws_iam_user" "user" {
  count = length(var.names)
  name  = var.names[count.index]
}
# Creates: user[0]=alice, user[1]=bob, user[2]=charlie

# PROBLEM: Remove "bob" from the middle:
variable "names" {
  default = ["alice", "charlie"]   # bob removed
}
# Terraform now sees:
# user[0]=alice (no change)
# user[1]=charlie (was bob → terraform wants to UPDATE bob to charlie!)
# user[2] DESTROY (charlie moved to index 1)
# Result: bob deleted AND charlie renamed AND then alice renamed!
# → Destroying and recreating resources you didn't want to change!

# SOLUTION: for_each with a set or map
resource "aws_iam_user" "user" {
  for_each = toset(["alice", "bob", "charlie"])
  name     = each.key
}
# Creates: user["alice"], user["bob"], user["charlie"]
# Remove "bob":
for_each = toset(["alice", "charlie"])
# Terraform: ONLY deletes user["bob"] — alice and charlie untouched! ✅
```

**When to use each:**
```
count:    when creating N identical resources, no names needed
          count = var.enable_monitoring ? 1 : 0  (conditional creation)

for_each: when creating resources from a map/list with distinct identities
          when you might add/remove items (avoids index shift problem)
          PREFERRED for most resource creation scenarios
```

> 💡 **Interview tip:** The **count index shifting problem** is one of the most asked Terraform intermediate questions. In production, accidentally destroying and recreating databases or load balancers (because you removed an item from the middle of a count list) is catastrophic. Always use `for_each` with a map or set when the resources have identities (names, environments, regions). Use `count` only for truly identical resources or conditional creation (`count = var.enabled ? 1 : 0`).

---

## SECTION 6 — Prometheus & SRE Fundamentals

---

### Q611 — Prometheus | Conceptual | Beginner-Intermediate

> Explain **PromQL basics** — what is an **instant vector**, **range vector**, and **scalar**? What are **selectors** and **matchers**?
>
> Write basic PromQL queries for: current CPU usage per instance, total HTTP requests per second, memory usage percentage.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — PromQL basics

#### Key Points to Cover in Your Answer:

**PromQL data types:**
```
Instant Vector:
  → Set of time series, ONE sample per series at current time
  → Most common type
  → Example: node_cpu_seconds_total
  → Returns: one value per {instance, cpu, mode} label combination

Range Vector:
  → Set of time series, MULTIPLE samples over a time range
  → Used with functions like rate(), increase(), avg_over_time()
  → Syntax: metric[5m] = last 5 minutes of data
  → Example: node_cpu_seconds_total[5m]

Scalar:
  → Single numeric value (no labels)
  → Example: 1024 * 1024 (1MB in bytes)
  → Result of operations that reduce to single number
```

**Selectors and matchers:**
```promql
# Exact match (=)
http_requests_total{job="api-server"}

# Regex match (=~)
http_requests_total{method=~"GET|POST"}    # GET or POST

# Not equal (!=)
http_requests_total{status!="200"}         # non-200 responses

# Not regex match (!~)
http_requests_total{instance!~".*staging.*"}  # exclude staging

# Multiple matchers (AND)
http_requests_total{job="api", status="200"}

# Wildcard (all time series for metric)
http_requests_total
```

**Common PromQL queries:**
```promql
# CPU usage percentage per instance (excluding idle)
100 - (avg by (instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
) * 100)

# HTTP requests per second (rate over 5 min)
rate(http_requests_total[5m])

# HTTP requests per second by service (sum)
sum by (service) (rate(http_requests_total[5m]))

# Memory usage percentage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# 5xx error rate
rate(http_requests_total{status=~"5.."}[5m])
/ rate(http_requests_total[5m]) * 100

# Pod restart count
increase(kube_pod_container_status_restarts_total[1h])

# Disk space remaining percentage
(node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100

# Top 5 pods by memory
topk(5, container_memory_usage_bytes{container!=""})
```

**Key PromQL functions:**
```promql
rate(metric[5m])          # per-second rate of increase (counters)
irate(metric[5m])         # instantaneous rate (last 2 points)
increase(metric[5m])      # total increase over window
sum(metric)               # sum all time series
avg(metric)               # average across time series
min(metric) / max(metric) # min/max across time series
sum by(label)(metric)     # sum, grouped by label
count(metric)             # count of time series
histogram_quantile(0.99, rate(metric_bucket[5m]))  # p99 latency
```

> 💡 **Interview tip:** The most important PromQL concept for beginners: **`rate()` requires a Counter metric and a range vector**. `rate(http_requests_total[5m])` means "per-second rate over the last 5 minutes". `rate()` handles counter resets (process restarts) correctly. A common mistake is using `rate()` on a Gauge metric (like memory) — use `avg_over_time(memory[5m])` for gauges instead.

---

### Q612 — SRE | Conceptual | Beginner-Intermediate

> Explain **SLI, SLO, SLA, and Error Budget** — what is each, how do they relate, and how do you use them in practice?
>
> Define SLOs for a payment API. How does an error budget change how engineering teams behave?

📁 **Reference:** SRE practices — SLO/SLI/SLA fundamentals

#### Key Points to Cover in Your Answer:

**Definitions:**
```
SLI (Service Level Indicator):
  → A METRIC that measures a dimension of service quality
  → Answers: "How are we doing right now?"
  → Examples:
    Request success rate = successful_requests / total_requests × 100
    p99 latency = 99th percentile response time
    Availability = uptime / total_time × 100

SLO (Service Level Objective):
  → A TARGET for an SLI
  → Internal commitment (engineering to business)
  → Answers: "What performance do we promise to ourselves?"
  → Examples:
    Success rate ≥ 99.9%
    p99 latency < 200ms
    Availability ≥ 99.95%

SLA (Service Level Agreement):
  → Legal/contractual commitment to CUSTOMERS
  → Consequences if violated (credits, refunds, penalties)
  → SLA should be LESS strict than SLO (buffer for unknown failures)
  → Examples:
    99.9% uptime per month (or 10% credit)
  → SLO: 99.95% → SLA: 99.9% (internal buffer)

Error Budget:
  → How much "bad time" is allowed in the SLO period
  → Error Budget = 100% - SLO%
  → 99.9% SLO → 0.1% error budget = 43.8 minutes/month of downtime allowed
  → 99.99% SLO → 0.01% = 4.38 minutes/month
```

**SLOs for payment API:**
```
SLI 1: Availability
  Metric: % of successful payment requests (non-500 status)
  SLO: 99.95% → Error budget: 21.9 minutes/month

SLI 2: Latency
  Metric: p99 response time for payment processing
  SLO: p99 < 500ms → measured as % of requests under 500ms ≥ 99%

SLI 3: Freshness (for fraud detection)
  Metric: % of fraud checks completed within 200ms
  SLO: 99.9% of fraud checks under 200ms
```

**How error budget changes behavior:**
```
Error budget > 50%: Normal operations
  → Engineers can take risks: deploy often, try new things
  → "We have budget to spend on experiments"

Error budget < 25%: Caution mode
  → Slow down risky changes
  → Focus on reliability improvements

Error budget = 0%: Freeze mode
  → STOP all feature work
  → All engineers focus on reliability only
  → Root cause analysis of budget burn
  → Resume only when budget replenishes

Why this is powerful:
  → Business understands reliability as a resource (budget)
  → Engineers own their reliability (they lose deployment freedom)
  → Natural tension: "go fast" vs "be reliable" resolved by data
  → No more vague debates about "is it reliable enough"
```

> 💡 **Interview tip:** The **most common SLO mistake**: setting SLO = SLA. You need a buffer — if your SLA is 99.9%, your SLO should be 99.95%. If your SLO burns, you have time to fix it before breaching the SLA and facing penalties. The **second common mistake**: setting SLO too high (99.999%). Every 9 added multiplies the engineering investment needed by 10x. Start with a reasonable SLO (99.9%), meet it consistently, then improve. A 99.9% SLO with consistent execution beats a 99.99% SLO that is regularly violated.

---

### Q613 — SRE | Conceptual | Beginner-Intermediate

> Explain **GitOps** — what is it, what are its core principles, and how does it differ from traditional CI/CD?
>
> What is the difference between **push-based** and **pull-based** deployments? What tools implement GitOps?

📁 **Reference:** `nawab312/CI_CD` — GitOps principles, ArgoCD

#### Key Points to Cover in Your Answer:

**What GitOps is:**
```
GitOps = using Git as the single source of truth for infrastructure
         and application deployment

Core principles (Weaveworks definition):
  1. Declarative: entire system described in declarative config (YAML)
  2. Versioned and immutable: config stored in Git (history, audit trail)
  3. Pulled automatically: software agents reconcile actual vs desired state
  4. Continuously reconciled: agents detect and correct drift automatically

In practice:
  → All Kubernetes manifests, Helm charts, Kustomize in Git
  → Every production change = a Git commit (PR → merge → deploy)
  → No kubectl apply from developer laptops
  → ArgoCD/Flux watches Git → applies changes automatically
```

**Push-based vs Pull-based:**
```
Push-based (traditional CI/CD):
  Jenkins/GitHub Actions pipeline → kubectl apply → cluster
  
  ✅ Simple to understand
  ✅ Familiar to most teams
  ❌ Pipeline has cluster credentials (security risk)
  ❌ Drift: someone applies manifest outside pipeline → not detected
  ❌ No reconciliation: if cluster diverges, pipeline won't fix it
  
  Examples: Jenkins deploying to K8s, GitHub Actions with kubectl

Pull-based (GitOps):
  ArgoCD/Flux running IN cluster → watches Git repo → pulls changes
  
  ✅ Cluster credentials never leave cluster (more secure)
  ✅ Auto-detects and fixes drift (continuous reconciliation)
  ✅ Git is source of truth (every state is in Git history)
  ✅ Audit trail: every change is a Git commit
  ❌ Slightly more complex setup
  ❌ Need to manage Git repo structure carefully
  
  Examples: ArgoCD, Flux
```

**GitOps workflow:**
```
Developer:
  1. Makes code change
  2. CI pipeline: test → build image → push to ECR with SHA tag
  3. CI updates Git repo: deployment.yaml image.tag = abc1234
  4. Creates PR to config repo

GitOps operator (ArgoCD):
  5. Detects PR merged to main
  6. Compares desired state (Git) vs actual state (cluster)
  7. Applies differences: kubectl apply
  8. Reports sync status

Benefits:
  → Full audit trail: who deployed what, when, why (Git blame)
  → Easy rollback: git revert + merge
  → Disaster recovery: rebuild cluster from Git = exact state restored
```

**Tools:**
```
ArgoCD:   Web UI, Helm/Kustomize, ApplicationSet, multi-cluster
Flux:     Lighter weight, native Kubernetes (no UI), GitOps toolkit
Jenkins X: Jenkins-native GitOps (less popular now)
Rancher Fleet: GitOps for large-scale multi-cluster
```

> 💡 **Interview tip:** The **most important GitOps benefit** for security teams: **no credentials leave the cluster**. In push-based CI/CD, your Jenkins/GitHub Actions needs cluster admin credentials (stored as secrets in CI system). If the CI system is compromised, attackers get cluster access. In pull-based GitOps, ArgoCD runs inside the cluster and pulls from Git — no cluster credentials anywhere outside. This is especially important for regulated industries.

---

### Q614 — CI/CD | Conceptual | Beginner-Intermediate

> Explain **Jenkins architecture** — what are the master/controller, agents/workers, executors, and build queues?
>
> What is the difference between **declarative** and **scripted** pipelines? What is a `Jenkinsfile`?

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — architecture, pipeline types

#### Key Points to Cover in Your Answer:

**Jenkins architecture:**
```
Jenkins Controller (Master):
  → Central server running the Jenkins web UI
  → Stores: jobs, configurations, build history, plugins
  → Schedules builds, assigns to agents
  → Does NOT run builds directly (delegates to agents)
  → Single point of management

Jenkins Agent (Worker/Slave):
  → Machine that runs the actual build
  → Connects to controller via: SSH or JNLP (Java Web Start)
  → Can be: permanent agents or ephemeral (Docker, Kubernetes)
  → Each agent has one or more EXECUTORS

Executor:
  → A slot for running builds on an agent
  → Agent with 4 executors can run 4 builds simultaneously
  → Think of executor as a thread

Build Queue:
  → Builds waiting for an available executor
  → Job triggered → added to queue → assigned when executor free
  → High queue depth = need more agents/executors

Kubernetes agents (dynamic):
  → Controller creates agent pods on-demand per build
  → Pod terminated after build completes
  → Clean environment for every build
  → Scale: unlimited agents (Kubernetes scales pods)
```

**Declarative vs Scripted pipeline:**
```
Declarative (recommended, easier):
  → Structured, opinionated syntax
  → Limited to pipeline DSL (good for consistency)
  → Better error messages, easier to read
  → Has: pipeline{}, agent{}, stages{}, steps{}, post{}

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    }
    post {
        success { echo 'Pipeline succeeded!' }
        failure { echo 'Pipeline failed!' }
    }
}

Scripted (legacy, more powerful):
  → Groovy code directly
  → Maximum flexibility (can do any Groovy)
  → Harder to read, no guardrails
  → Use only when declarative is insufficient

node {
    stage('Build') {
        sh 'mvn package'
    }
    stage('Test') {
        sh 'mvn test'
    }
}
```

**Jenkinsfile:**
```
→ Text file containing pipeline definition
→ Stored IN the application's Git repository
→ Version-controlled with the code
→ Loaded automatically when Jenkins scans the repo
→ "Pipeline as Code" — pipeline definition alongside code

Benefits:
  → Pipeline changes reviewed like code (PR process)
  → Rollback pipeline = git revert
  → Audit trail for pipeline changes
  → No Jenkins UI changes needed for pipeline updates
```

> 💡 **Interview tip:** The **most important Jenkins architecture point** for scaling: **never run builds on the controller**. The controller should only orchestrate — actual build work goes to agents. A common beginner mistake is using `agent any` on the controller itself, which causes: build artifacts consuming controller disk, builds consuming controller CPU/memory, potential controller instability during heavy builds. In production: use `agent { kubernetes { ... } }` for dynamic ephemeral agents that spin up for each build and terminate after.

---

### Q615 — Linux | Conceptual | Beginner-Intermediate

> Explain **Linux cron** and **crontab** — what is cron, what is crontab syntax, and what are the differences between system-level and user-level cron?
>
> What are common cron problems? What is `anacron` and when would you use it over cron?

📁 **Reference:** `nawab312/DSA` → `Linux` — cron, scheduling

#### Key Points to Cover in Your Answer:

**Cron basics:**
```
cron: daemon that runs scheduled commands
crontab: table of cron jobs (per user)

Crontab syntax: 5 fields + command
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-7, 0 and 7 = Sunday)
│ │ │ │ │
* * * * * command

Examples:
0 2 * * *    /backup.sh          # 2am every day
*/15 * * * * /health-check.sh    # every 15 minutes
0 9 * * 1    /weekly-report.sh   # 9am every Monday
0 0 1 * *    /monthly-cleanup.sh # midnight, 1st of each month
@reboot      /startup-script.sh  # once on system boot
@daily       /daily-job.sh       # shortcut for "0 0 * * *"
@hourly      /hourly-job.sh      # shortcut for "0 * * * *"
```

**crontab management:**
```bash
crontab -e          # edit current user's crontab (opens in $EDITOR)
crontab -l          # list current user's crontab
crontab -r          # REMOVE current user's crontab (be careful!)
crontab -u user -e  # edit another user's crontab (root only)

# System cron (run as root, for system tasks)
/etc/crontab        # system-wide crontab (has USERNAME field)
/etc/cron.d/        # drop-in crontab files
/etc/cron.hourly/   # scripts run hourly
/etc/cron.daily/    # scripts run daily
/etc/cron.weekly/   # scripts run weekly
/etc/cron.monthly/  # scripts run monthly
```

**Common cron problems:**
```
1. Wrong PATH: cron runs with minimal PATH (/usr/bin:/bin)
   Fix: use full paths in cron scripts
   or: PATH=/usr/local/bin:/usr/bin:/bin in crontab header

2. No output captured: by default output goes to email (may be lost)
   Fix: redirect output:
   0 2 * * * /backup.sh >> /var/log/backup.log 2>&1

3. Job runs but nothing happens: permissions wrong, wrong user
   Fix: crontab -u root -e for system jobs

4. Time zone confusion: cron uses server's local timezone
   Fix: set TZ=UTC in crontab or use absolute times in UTC
   TZ=UTC
   0 14 * * * /job.sh    # 2pm UTC every day

5. Missed jobs (server was down): cron skips jobs if server off
   Fix: use anacron for non-time-critical jobs
```

**anacron vs cron:**
```
cron:    runs at EXACT scheduled time (misses if server off)
         requires server always on
         best for: real-time operations (minute/hour precision)

anacron: runs jobs when they should have run, even after downtime
         doesn't require server always running
         minimum granularity: DAILY (not minute-level)
         best for: maintenance tasks on servers that shut down
         (laptops, desktops, servers with maintenance windows)

Example: daily backup scheduled but server was down for 2 days
  cron: backup missed for 2 days ← data loss risk
  anacron: runs backup as soon as server comes back online ← safe
```

> 💡 **Interview tip:** The **output redirection** is the most commonly forgotten cron configuration. Without `>> /var/log/job.log 2>&1`, all output (including errors) goes to the user's mail (often /var/mail/root which nobody reads). When a cron job fails silently, the first debug step is: add `2>&1 | tee /var/log/job.log` to capture both stdout and stderr, then check the log file. Also: always test cron scripts manually as the cron user (`su -s /bin/bash -c '/path/to/script.sh' cronuser`) before relying on them in production.

---

### Q616 — Linux | Conceptual | Beginner-Intermediate

> Explain **inodes and links** in Linux — what is an inode, what does it store, and what is the relationship between a filename and an inode?
>
> What is the difference between a **hard link** and a **soft link (symbolic link)**? What are the limitations of each?

📁 **Reference:** `nawab312/DSA` → `Linux` — filesystem, inodes

#### Key Points to Cover in Your Answer:

**Inode:**
```
inode (index node): data structure storing file metadata
  → Every file and directory has exactly ONE inode
  → inode stores: file type, permissions, owner, size, timestamps,
                  block locations (where data is on disk)
  → inode does NOT store: filename

How files work:
  Filename → inode number → inode → data blocks on disk

  Directory: maps filenames → inode numbers
  /home/user/file.txt → inode 12345 → data blocks

View inode number:
  ls -i file.txt           → shows inode number
  stat file.txt            → shows all inode attributes
  df -i                    → show inode usage per filesystem
  find / -inum 12345       → find file by inode number

Inode exhaustion:
  Filesystem can run out of inodes (fixed at format time)
  "No space left on device" but df shows space → inode exhaustion!
  Common cause: millions of tiny files
  Check: df -i
```

**Hard links:**
```
Hard link: another directory entry pointing to the SAME inode
  → Multiple names for the same file
  → inode reference count increases (link count)
  → File exists as long as ONE hard link exists
  → Deleting one link: just removes that directory entry, inode stays

Create:  ln original.txt hardlink.txt
View:    ls -la  → shows link count (column 2)

Properties:
  ✅ Same inode: changes to one = changes to all (same data)
  ✅ Deleting "original" doesn't delete data (hardlink still exists)
  ✅ No dangling links (inode exists as long as a link exists)
  ❌ Cannot span filesystems (inode numbers only unique within filesystem)
  ❌ Cannot link directories (prevents filesystem loops)
  ❌ Both files appear to be in same location (confusing)
```

**Soft links (symbolic links):**
```
Soft link: special file containing a PATH to another file
  → New inode created for the symlink itself
  → Points to: path string (not inode number)
  → Like a shortcut/alias

Create:  ln -s /original/path /link/path
View:    ls -la  → shows l and → target
         lrwxrwxrwx 1 user group 15 Jan 15 mylink -> /target/file

Properties:
  ✅ Can span filesystems (stores path, not inode)
  ✅ Can link directories
  ✅ Clearly shows it's a link (ls -la shows arrow)
  ✅ Use: /usr/bin/python → /usr/bin/python3.10 (version management)
  ❌ Dangling links: if target deleted, symlink is broken
  ❌ Different inode than target
  ❌ Following symlinks has slight overhead
```

**Comparison:**
```
Feature          Hard Link      Soft Link
Same inode?      YES            No (own inode)
Cross filesystem? No            YES
Link directories? No            YES (with care)
Broken link?      Never         Possible
Shows target?     No            Yes (ls -la)
Common use:       Rarely needed  Version mgmt, shortcuts
```

> 💡 **Interview tip:** Soft links are used everywhere in Linux for **version management** — `/usr/bin/python` → `/usr/bin/python3.10`, `/usr/bin/java` → `/usr/lib/jvm/java-17/bin/java`. Update-alternatives manages these symlinks when you install new versions. The most practical inode interview question: "Why does `df` show space available but file creation fails?" Answer: inode table is full. Fix: either delete many small files or reformat with more inodes (using `mkfs.ext4 -N <count>`).

---

### Q617 — Linux | Conceptual | Beginner-Intermediate

> Explain **Linux package management** — what is the difference between **apt/dpkg** (Debian/Ubuntu) and **yum/dnf/rpm** (RHEL/CentOS/Amazon Linux)?
>
> What is a **package repository**? How do you add a new repository? What is the difference between installing from a repository vs installing from a .deb/.rpm file?

📁 **Reference:** `nawab312/DSA` → `Linux` — package management

#### Key Points to Cover in Your Answer:

**Package management systems:**
```
Debian/Ubuntu family:
  Low-level:  dpkg  → installs .deb packages, no dependency resolution
  High-level: apt   → uses dpkg, resolves dependencies, downloads from repos
  Config:     /etc/apt/sources.list + /etc/apt/sources.list.d/

RHEL/CentOS/Amazon Linux family:
  Low-level:  rpm   → installs .rpm packages, no dependency resolution
  High-level: yum   → uses rpm, resolves dependencies (older)
  High-level: dnf   → modern replacement for yum (faster, better)
  Config:     /etc/yum.repos.d/ or /etc/dnf/dnf.conf

Amazon Linux 2023:
  Uses dnf (same as RHEL/CentOS)
```

**Common commands:**
```bash
# Debian/Ubuntu (apt):
apt update                        # refresh package list from repos
apt upgrade                       # upgrade all installed packages
apt install nginx                 # install package
apt remove nginx                  # remove package (keep config)
apt purge nginx                   # remove package AND config files
apt search "web server"           # search available packages
apt show nginx                    # show package info
apt list --installed              # list installed packages
dpkg -l nginx                     # check if nginx installed
dpkg -L nginx                     # list files installed by package

# RHEL/CentOS/Amazon Linux (dnf/yum):
dnf check-update                  # check for updates
dnf update                        # update all packages
dnf install nginx                 # install package
dnf remove nginx                  # remove package
dnf search "web server"           # search packages
dnf info nginx                    # show package info
rpm -qa nginx                     # check if nginx installed
rpm -ql nginx                     # list files in package
rpm -qi nginx                     # package info
```

**Repository vs direct .deb/.rpm:**
```
From repository (recommended):
  ✅ Dependency resolution (installs all required packages)
  ✅ Updates: apt upgrade picks up new versions
  ✅ Verified: packages signed by maintainer
  ✅ Clean uninstall (package manager tracks all files)

From file (apt install ./package.deb or rpm -ivh package.rpm):
  ✅ Specific version not in public repo
  ✅ Private packages
  ❌ No automatic dependency resolution (must install deps manually)
  ❌ No automatic updates
  → For dependencies: apt install -f (fix broken) or dnf install package.rpm (dnf resolves)
```

**Adding a new repository:**
```bash
# Debian/Ubuntu - add repository:
# Method 1: add-apt-repository (for PPAs)
add-apt-repository ppa:nginx/stable
apt update && apt install nginx

# Method 2: manual key + source
curl -fsSL https://nginx.org/keys/nginx_signing.key | apt-key add -
echo "deb http://nginx.org/packages/ubuntu focal nginx" > /etc/apt/sources.list.d/nginx.list
apt update && apt install nginx

# RHEL/CentOS - add repository:
# Method 1: dnf config-manager
dnf config-manager --add-repo https://nginx.org/packages/centos/nginx.repo

# Method 2: .repo file
cat > /etc/yum.repos.d/nginx.repo << EOF
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
enabled=1
gpgcheck=1
gpgkey=https://nginx.org/keys/nginx_signing.key
EOF
dnf install nginx
```

> 💡 **Interview tip:** On **Amazon Linux 2023** (which uses dnf), the most important thing: **never install packages from third-party repos without checking compatibility**. Amazon Linux packages are maintained by AWS and are optimized for EC2. For production servers: use the AWS package repositories, avoid PPAs/third-party repos, and use `yum install` (which maps to dnf on AL2023) for anything in standard repos. For custom software, prefer Docker containers over system packages — isolates the software and its dependencies from the host OS.

---

## Final Coverage Summary — Q576–Q625

| Topic | Questions Covered |
|---|---|
| **K8s: Complete Architecture** | Q576 |
| **K8s: Reconciliation Loop** | Q577 |
| **K8s: kubectl apply flow** | Q578 |
| **K8s: Probes** | Q579 |
| **K8s: Service Types** | Q580 |
| **K8s: Resource Requests/Limits/QoS** | Q581 |
| **K8s: ConfigMaps & Secrets** | Q582 |
| **K8s: Namespaces** | Q583 |
| **K8s: Workload Types** | Q584 |
| **K8s: Labels, Annotations, Selectors** | Q585 |
| **K8s: Ingress** | Q586 |
| **K8s: Volume Types** | Q587 |
| **K8s: Scheduler Internals** | Q588 |
| **AWS: Global Infrastructure** | Q589 |
| **AWS: ALB vs NLB** | Q590 |
| **AWS: EBS Volume Types** | Q591 |
| **AWS: DynamoDB** | Q592 |
| **AWS: Lambda** | Q593 |
| **AWS: IAM Fundamentals** | Q594 |
| **AWS: S3 Storage Classes** | Q595 |
| **AWS: RDS Multi-AZ vs Read Replica** | Q596 |
| **AWS: VPC Design** | Q597 |
| **AWS: EC2 Basics** | Q598 |
| **AWS: CloudWatch** | Q599 |
| **AWS: Kinesis** | Q600 |
| **AWS: Route53 Record Types** | Q601 |
| **Docker: Container vs VM** | Q602 |
| **Docker: Dockerfile Instructions** | Q603 |
| **Docker: Image Layers** | Q604 |
| **Linux: File Permissions** | Q605 |
| **Linux: Process States** | Q606 |
| **Linux: grep/sed/awk** | Q607 |
| **Linux: SSH** | Q608 |
| **Terraform: State** | Q609 |
| **Terraform: count vs for_each** | Q610 |
| **Prometheus: PromQL Basics** | Q611 |
| **SRE: SLO/SLI/SLA/Error Budget** | Q612 |
| **SRE: GitOps** | Q613 |
| **CI/CD: Jenkins Architecture** | Q614 |
| **Linux: Cron** | Q615 |
| **Linux: Inodes/Links** | Q616 |
| **Linux: Package Management** | Q617 |

---

*File 1 of 2 — Core Fundamentals*
*DevOps / SRE Interview Preparation — Q576–Q625*
