# DevOps / SRE / Cloud Interview Questions (46–90) — Enhanced
## Key Points to Cover + Interview Tips Added to Every Question

---

### Q46 — Kubernetes | Conceptual | Advanced

> Explain what a **Kubernetes Operator** is. How is it different from a regular **Controller**? Give a real-world example of when you would build or use an Operator instead of a standard Kubernetes resource.

📁 **Reference:** `nawab312/Kubernetes` → `10_CRDs_EXTENSIBILITY.md`

#### Key Points to Cover:
```
Controller (built-in):
  → Watches K8s objects, reconciles actual state to desired state
  → Built into kube-controller-manager
  → Manages built-in resources: Deployments, ReplicaSets, Services
  → Generic logic — doesn't know your app's domain

Operator = Controller + CRD (Custom Resource Definition)
  → Encodes HUMAN OPERATOR knowledge into software
  → Manages a CUSTOM resource (your domain object)
  → Knows app-specific lifecycle: backup, upgrade, failover
  → Deployed as a Pod in your cluster

Example CRD + Operator:
  # Without Operator: run PostgreSQL on K8s
  → Need: StatefulSet, Services, PVCs, backup CronJob, init scripts
  → Complex to manage, lots of manual steps for upgrades/failover

  # With Operator (e.g. CloudNativePG):
  apiVersion: postgresql.cnpg.io/v1
  kind: Cluster
  metadata:
    name: postgres-cluster
  spec:
    instances: 3
    storage:
      size: 100Gi
  # Operator handles: primary election, replication,
  # automated backups, minor version upgrades, failover

When to USE an existing Operator:
  → Databases: CloudNativePG, MongoDB, MySQL
  → Kafka: Strimzi Operator
  → Prometheus: Prometheus Operator
  → Elasticsearch: ECK Operator
  → Cert-manager, Vault, Istio all use Operators

When to BUILD a custom Operator:
  → Your app has complex stateful lifecycle management
  → You need automated Day-2 operations (upgrades, backups)
  → Repeated human steps that could be codified
  → Tools: Operator SDK (Go), Kopf (Python), KUDO

Operator Maturity Model:
  Level 1: Basic Install
  Level 2: Seamless Upgrades
  Level 3: Full Lifecycle
  Level 4: Deep Insights
  Level 5: Auto Pilot
```

> 💡 **Interview tip:** The key distinction: **Controllers manage built-in resources, Operators manage custom domain resources with application-specific knowledge.** The classic example: a human PostgreSQL DBA knows to promote a replica when primary fails, run WAL archiving, take consistent backups. An Operator encodes exactly that knowledge — it's a "robotic DBA." When asked "when would you build one?", the answer is: when you have **repetitive Day-2 operational tasks** (upgrades, backups, scaling decisions) that require application-domain knowledge beyond what standard K8s provides.

---

### Q47 — AWS | Scenario-Based | Advanced

> Your company has a microservices architecture on AWS. You are seeing **intermittent timeout errors** between Service A and Service B. Service A calls Service B via HTTP. Sometimes the call succeeds, sometimes it times out with no clear pattern. **How would you systematically investigate?**

📁 **Reference:** `nawab312/AWS` — VPC, Security Groups, Load Balancers, CloudWatch sections

#### Key Points to Cover:
```
Step 1 — Characterize the problem:
  → How often? (5%? 50%? random? time-based?)
  → Which AZ? (check if single AZ correlation)
  → Which instance? (check if specific EC2/pod is slow)
  → What payload size? (large payloads timeout, small succeed?)

Step 2 — Check Service B health:
  aws cloudwatch get-metric-statistics \
    --metric-name TargetResponseTime \
    --namespace AWS/ApplicationELB
  → Is Service B's p99 latency high?
  → ALB 5xx vs 4xx errors?
  → Target group healthy instance count dropping?

  p99 latency means the response time that 99% of requests finish within.
  
  Imagine 100 requests made to your service:
  
    Request 1    →  12ms
    Request 2    →  15ms
    Request 3    →  18ms
    ...
    Request 97   →  100ms
    Request 98   →  120ms
    Request 99   →  180ms   ← p99 is this number
    Request 100  →  4500ms  ← this one outlier (the slowest 1%)
  
    p99 = 180ms
    meaning: 99% of your users got a response in 180ms or faster
             1% of your users waited longer than 180ms

Step 3 — Check connection pool exhaustion:
  → Service B might be running out of connections
  → Check: active threads, connection pool metrics
  → CloudWatch: DatabaseConnections (if calling RDS downstream)

Step 4 — Check network path:
  # From Service A's EC2/pod:
  curl -v --max-time 5 http://service-b-internal/health
  traceroute service-b-internal
  → Is DNS resolving correctly?
  → Are Security Groups allowing traffic on correct port?
  → NACLs blocking return traffic (ephemeral ports)?

Step 5 — Enable AWS X-Ray distributed tracing:
  → Shows exact latency at each service hop
  → Identifies: is slow segment in Service B itself or the network?
  → Service map shows which downstream calls are slow

Step 6 — Check for connection timeout vs read timeout:
  → Connection timeout: can't reach Service B (network/SG)
  → Read timeout: reached but response too slow (app issue)

Common root causes:
  → Service B under load (no capacity)
  → Connection pool exhausted on Service B
  → GC pause in Service B (Java)
  → AZ imbalance (all traffic hitting one instance)
  → Keep-alive connection reuse issues
  → NLB unhealthy target not removed fast enough
  → DNS TTL caching stale IPs
```

> 💡 **Interview tip:** Structure the investigation as **network vs application**. First prove it's not a network issue (Security Groups, NACLs, DNS) — this is fast to check. Then prove it's not infrastructure capacity (CloudWatch ALB metrics, target health). Then look at application behavior (X-Ray traces, connection pools, GC logs). Mention **X-Ray service map** as the single most useful tool — it visually shows which service and which downstream call is the latency source. Without X-Ray, you're guessing; with X-Ray, you're reading a map.

---

### Q48 — Terraform | Conceptual | Advanced

> What is the difference between **`terraform workspace`** and **separate directories per environment**? What are the pros and cons of each, and which would you recommend for production?

📁 **Reference:** `nawab312/Terraform` — Workspaces and environment management sections

#### Key Points to Cover:
```
Terraform Workspaces:
  → Single codebase, multiple state files per workspace
  → terraform workspace new prod
  → terraform workspace select staging
  → State stored at: terraform.tfstate.d/<workspace>/
  → Same .tf files for all environments
  → Access workspace name: ${terraform.workspace}

  resource "aws_instance" "web" {
    instance_type = terraform.workspace == "prod" ? "m5.xlarge" : "t3.medium"
  }

  Pros:
  ✅ Single codebase (DRY)
  ✅ Easy to create new environments
  ✅ Workspace name available in code

  Cons:
  ❌ Same backend bucket, easy to accidentally destroy prod
  ❌ Logic becomes messy (if workspace == prod...)
  ❌ No isolation between environments
  ❌ Hard to have fundamentally different architectures
  ❌ Not recommended for production by HashiCorp

Separate directories (recommended for production):
  environments/
    dev/
      main.tf       ← calls modules with dev variables
      variables.tf
      terraform.tfvars
      backend.tf    ← separate S3 key: "dev/terraform.tfstate"
    staging/
      main.tf
      backend.tf    ← separate S3 key: "staging/terraform.tfstate"
    prod/
      main.tf
      backend.tf    ← separate S3 key: "prod/terraform.tfstate"
  modules/
    vpc/
    eks/
    rds/

  Pros:
  ✅ Complete isolation (separate state, separate backend)
  ✅ Can have different architecture per env (prod has WAF, dev doesn't)
  ✅ Clear separation of who can run what (IAM per env)
  ✅ Accidental prod destroy much harder
  ✅ HashiCorp recommended for production

  Cons:
  ❌ More directories to manage
  ❌ Must navigate to correct dir before running
  ❌ Risk of drift if modules updated in one env but not others

Recommendation: Separate directories + modules
  → Modules: shared reusable code
  → Environments: separate dirs with separate backends
  → Terragrunt: makes this pattern DRY (avoids backend config repetition)
```

> 💡 **Interview tip:** HashiCorp themselves say **workspaces are NOT intended for environment separation** — they're for testing alternate configurations of the same infrastructure. The problem with workspaces in production: all environments share one backend bucket, and a typo (running `terraform destroy` on the wrong workspace) can destroy prod. With separate directories + separate S3 buckets + separate IAM roles, a developer physically cannot run `terraform apply` against prod unless they have prod IAM permissions. **Isolation by IAM** is the production-grade approach.

---

### Q49 — Kubernetes | Scenario-Based | Advanced

> You have a multi-tenant Kubernetes cluster where Team A's application is consuming excessive CPU/memory, causing other teams' resource starvation and evictions. **How would you isolate resources between teams and prevent one team from affecting others?**

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

#### Key Points to Cover:
```
Layer 1 — Namespace isolation:
  kubectl create namespace team-a
  kubectl create namespace team-b
  # Each team gets their own namespace

Layer 2 — ResourceQuota (namespace-level ceiling):
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: team-a-quota
    namespace: team-a
  spec:
    hard:
      requests.cpu: "10"        # total CPU requests in namespace
      requests.memory: 20Gi    # total memory requests
      limits.cpu: "20"          # total CPU limits
      limits.memory: 40Gi
      pods: "50"               # max pods
      services: "10"

Layer 3 — LimitRange (per-container defaults and limits):
  apiVersion: v1
  kind: LimitRange
  metadata:
    name: team-a-limits
    namespace: team-a
  spec:
    limits:
    - type: Container
      default:               # default LIMIT if not specified
        cpu: "500m"
        memory: 512Mi
      defaultRequest:        # default REQUEST if not specified
        cpu: "100m"
        memory: 128Mi
      max:                   # maximum allowed per container
        cpu: "4"
        memory: 8Gi
      min:
        cpu: "50m"
        memory: 64Mi

Layer 4 — Priority Classes:
  # Ensure critical infra pods aren't evicted before team pods
  apiVersion: scheduling.k8s.io/v1
  kind: PriorityClass
  metadata:
    name: team-critical
  value: 1000000
  globalDefault: false
  # Assign to critical pods: priorityClassName: team-critical

Layer 5 — Node taints + team-specific node groups:
  # Dedicated nodes for team isolation (strongest isolation)
  kubectl taint nodes node-group-a team=a:NoSchedule
  # Team A pods get toleration for this taint

Layer 6 — NetworkPolicy (traffic isolation):
  # Team A pods cannot reach Team B pods
  kind: NetworkPolicy
  spec:
    podSelector: {}       # all pods in namespace
    policyTypes: [Ingress, Egress]
    ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            team: a       # only from same team namespace

Immediate fix for existing starvation:
  kubectl top pods -n team-a --sort-by=cpu
  kubectl describe pod <hungry-pod> -n team-a
  # Add resource requests/limits to team-a pods immediately
```

> 💡 **Interview tip:** The **order of enforcement** matters: LimitRange sets per-container defaults and maximums (enforced at pod creation), ResourceQuota sets namespace-level totals (enforced at namespace level). Without LimitRange, a pod with no resource requests is treated as requesting 0 — it gets scheduled everywhere but then steals resources without limits. Always pair ResourceQuota with LimitRange — quota without per-pod limits is incomplete. For **strongest isolation**, use dedicated node groups with taints — this prevents noisy neighbors at the kernel level, not just the K8s scheduling level.

---

### Q50 — Linux / Bash | Conceptual | Advanced

> Explain the difference between a **process** and a **thread**. What are **zombie processes** and **orphan processes** — how are they created, what problems do they cause, and how do you identify and clean them up?

📁 **Reference:** `nawab312/DSA` → `Linux` — process management sections

#### Key Points to Cover:
```
Process vs Thread:
  Process:
    → Independent execution unit with own memory space
    → Own PID, file descriptors, address space
    → Communication via IPC (pipes, sockets, shared memory)
    → Isolation: crash doesn't affect other processes
    → Creation: fork() system call (expensive — copies memory)

  Thread:
    → Lightweight execution unit WITHIN a process
    → Shares: memory, file descriptors, address space with parent
    → Own: stack, registers, thread ID
    → Communication: shared memory (fast but needs synchronization)
    → Creation: pthread_create() (cheap — no memory copy)
    → Crash in one thread can kill entire process

  View threads:
    ps -T -p <PID>        # threads of specific process
    top -H -p <PID>       # top with thread view
    ls /proc/<PID>/task/  # one directory per thread

Zombie processes:
  Definition: child process has EXITED but parent hasn't called wait()
  State: 'Z' in ps output (ps aux shows 'Z' in STAT column)

  How created:
    1. Child process calls exit()
    2. Child enters zombie state (kernel keeps exit status)
    3. Parent should call wait() to collect exit status
    4. If parent never calls wait() → zombie persists

  Problem: zombie consumes PID (limited resource, ~32768 by default)
  Many zombies → cannot create new processes → system unusable

  Identify:
    ps aux | awk '$8 == "Z"'
    ps aux | grep defunct

  Clean up:
    # Option 1: kill the PARENT process (parent dying triggers init to reap)
    kill -9 <PARENT_PID>
    # Option 2: send SIGCHLD to parent (asks parent to call wait())
    kill -SIGCHLD <PARENT_PID>
    # Note: cannot kill zombie itself — it's already dead

Orphan processes:
  Definition: parent process DIES before child process
  Child is "adopted" by init/systemd (PID 1) automatically
  Init calls wait() when orphan exits → no zombie created

  Usually not a problem — init handles them
  Can be a problem if: orphan runs forever consuming resources

  Identify:
    pstree    # shows processes whose parent is systemd/init(PID 1)
    ps -eo pid,ppid,cmd | awk '$2 == 1'  # all children of PID 1
```

> 💡 **Interview tip:** Zombies are a common production issue in containerized environments — if your container's PID 1 doesn't properly handle SIGCHLD and reap children, zombie processes accumulate. This is why Kubernetes uses **`tini`** as PID 1 in containers (or Docker's `--init` flag) — tini is a tiny init process that properly reaps orphan/zombie children. The interview gold: "zombie can't be killed with `kill -9` because it's already dead — you kill the parent, which causes the zombie to be reaped by init."

---

### Q51 — Prometheus | Scenario-Based | Advanced

> You are asked to monitor a Node.js microservice running on Kubernetes with no existing metrics. Walk through: instrumenting the app, exposing metrics, and writing PromQL for request rate, p99 latency for `/api/checkout`, and 5xx error percentage.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`

#### Key Points to Cover:
```
Step 1 — Instrument Node.js app:
  npm install prom-client

  const promClient = require('prom-client');
  const register = new promClient.Registry();
  promClient.collectDefaultMetrics({ register }); // CPU, memory, GC

  // Custom metrics:
  const httpRequestsTotal = new promClient.Counter({
    name: 'http_requests_total',
    help: 'Total HTTP requests',
    labelNames: ['method', 'route', 'status_code'],
    registers: [register],
  });

  const httpRequestDuration = new promClient.Histogram({
    name: 'http_request_duration_seconds',
    help: 'HTTP request duration in seconds',
    labelNames: ['method', 'route', 'status_code'],
    buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
    registers: [register],
  });

  // Middleware to record metrics:
  app.use((req, res, next) => {
    const end = httpRequestDuration.startTimer({
      method: req.method,
      route: req.route?.path || req.path,
    });
    res.on('finish', () => {
      httpRequestsTotal.inc({
        method: req.method,
        route: req.route?.path || req.path,
        status_code: res.statusCode,
      });
      end({ status_code: res.statusCode });
    });
    next();
  });

  // Expose /metrics endpoint:
  app.get('/metrics', async (req, res) => {
    res.set('Content-Type', register.contentType);
    res.end(await register.metrics());
  });

Step 2 — K8s Service with named port:
  spec:
    ports:
    - name: http-metrics
      port: 9090

Step 3 — ServiceMonitor (Prometheus Operator):
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: nodejs-app
    labels:
      release: prometheus
  spec:
    selector:
      matchLabels:
        app: nodejs-app
    endpoints:
    - port: http-metrics
      interval: 30s

Step 4 — PromQL queries:

  # Request rate over last 5 minutes:
  sum(rate(http_requests_total[5m]))

  # Request rate per route:
  sum by(route) (rate(http_requests_total[5m]))

  # p99 latency for /api/checkout:
  histogram_quantile(0.99,
    sum by(le) (
      rate(http_request_duration_seconds_bucket{route="/api/checkout"}[5m])
    )
  )

  # 5xx error percentage:
  sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
  * 100

  # RED dashboard all-in-one:
  # Rate:   sum(rate(http_requests_total[5m]))
  # Errors: sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  # Durat:  histogram_quantile(0.99, sum by(le)(rate(http_request_duration_seconds_bucket[5m])))
```

> 💡 **Interview tip:** Use **Histogram (not Summary)** for latency in Kubernetes — Histograms can be aggregated across multiple pods (`sum by(le)`), Summaries cannot. With 10 pods, `histogram_quantile` correctly calculates p99 across all 10 pods. Summary calculates per-pod and cannot be aggregated. Also mention: always add a `/metrics` endpoint secured with network policy or basic auth — you don't want metrics exposed publicly. The `prom-client` `collectDefaultMetrics()` is often overlooked but provides CPU, memory, event loop lag, GC metrics for free.

---

### Q52 — Kubernetes | Troubleshooting | Advanced

> A Node in your Kubernetes cluster shows status **`NotReady`**. Pods on that node are being evicted. Walk through your complete troubleshooting process.

📁 **Reference:** `nawab312/Kubernetes` → `09_CLUSTER_OPERATIONS.md`, `14_TROUBLESHOOTING.md`

#### Key Points to Cover:
```
Step 1 — Identify the node and get details:
  kubectl get nodes
  kubectl describe node <node-name>
  # Look at: Conditions section (MemoryPressure, DiskPressure, PIDPressure, Ready)
  # Look at: Events section (what happened recently)

Step 2 — Check node conditions:
  Condition      Status  Meaning
  Ready          False   kubelet not communicating
  MemoryPressure True    Node running out of memory
  DiskPressure   True    Node running out of disk
  PIDPressure    True    Too many processes

Step 3 — SSH into the node:
  ssh ec2-user@<node-ip>

  # Check kubelet status:
  systemctl status kubelet
  journalctl -u kubelet -n 100 --no-pager
  # Most common: kubelet stopped, failed to connect to API server

  # Check container runtime:
  systemctl status containerd   # or docker
  crictl ps                     # list running containers

  # Check disk:
  df -h
  du -sh /var/lib/kubelet/*
  du -sh /var/log/containers/*

  # Check memory:
  free -h
  cat /proc/meminfo

  # Check certificates:
  ls /var/lib/kubelet/pki/
  openssl x509 -in /var/lib/kubelet/pki/kubelet.crt -noout -dates
  # Expired cert = kubelet can't auth to API server → NotReady

Step 4 — Common fixes:
  # Restart kubelet:
  systemctl restart kubelet
  systemctl restart containerd

  # Clear disk:
  crictl rmi --prune    # remove unused images
  journalctl --vacuum-size=500M

  # If certificate expired:
  # Rotate node certificates or rejoin node to cluster

Step 5 — If node unrecoverable, drain and replace:
  kubectl cordon <node-name>       # stop new pods scheduling
  kubectl drain <node-name> \      # evict existing pods
    --ignore-daemonsets \
    --delete-emptydir-data
  # Terminate EC2 instance → ASG replaces with fresh node
  kubectl delete node <node-name>  # remove from cluster state

Step 6 — Investigate pod evictions:
  kubectl get events -n <namespace> | grep Evicted
  kubectl describe pod <evicted-pod>
  # Reason: Node pressure, resource limits exceeded
```

> 💡 **Interview tip:** The most common NotReady cause in production: **kubelet stopped** (usually due to expired certificates or disk pressure). The fastest diagnostic: `systemctl status kubelet` — if it's failed, `journalctl -u kubelet -n 50` shows exactly why. Expired certificates are very common in clusters created 1 year ago — kubelet certs default to 1-year expiry. Also mention: when a node is NotReady, Kubernetes waits `node-monitor-grace-period` (default 40s) before marking pods for eviction, then waits `pod-eviction-timeout` (default 5min) before actually evicting — this adds up to ~5.5 minutes of potential impact before pods move.

---

### Q53 — AWS | Conceptual | Advanced

> Explain the difference between **AWS CloudWatch**, **AWS CloudTrail**, and **AWS Config**. Give a real-world example of each and how they complement each other in security and compliance.

📁 **Reference:** `nawab312/AWS` — CloudWatch, CloudTrail, AWS Config sections

#### Key Points to Cover:
```
CloudWatch — METRICS AND MONITORING:
  → Collects metrics, logs, events from AWS services
  → Real-time monitoring of operational health
  → Alarms, dashboards, anomaly detection
  → Use: "Is my EC2 CPU high RIGHT NOW?"
  → Use: "Alert me when RDS connections exceed 100"
  → Stores: numeric metrics + log data
  → Log Insights: query logs with SQL-like syntax

CloudTrail — API AUDIT TRAIL:
  → Records EVERY API call made in your AWS account
  → Who did what, when, from where
  → Answers: "Who deleted that S3 bucket?"
  → Answers: "Who changed this Security Group at 2am?"
  → Stores: JSON records of API calls (90 days default, S3 for longer)
  → Use: security investigations, compliance audits, forensics
  → CloudTrail Insights: detect unusual API activity patterns

AWS Config — RESOURCE CONFIGURATION HISTORY:
  → Continuously records configuration state of AWS resources
  → Answers: "What did this Security Group look like 6 months ago?"
  → Answers: "Which EC2 instances have public IPs?"
  → Config Rules: evaluate compliance (is S3 bucket public? fail)
  → Remediation: auto-fix non-compliant resources
  → Use: compliance, config drift detection, resource inventory

Real-world example — Security incident:
  Scenario: S3 bucket with customer data became publicly accessible

  CloudWatch:  "S3 GetObject requests spiked 10,000% at 3am" (detected)
  CloudTrail:  "PutBucketPolicy API called by user john@company.com
                from IP 203.x.x.x at 2:47am" (who did it)
  AWS Config:  "S3 bucket 'customer-data' changed from PRIVATE to PUBLIC
                at 2:47am. Previous config: BlockPublicAccess=true" (what changed)

Together:
  CloudWatch  → operational health + alerting (real-time)
  CloudTrail  → who/what/when API audit (forensics)
  AWS Config  → resource compliance + config history (governance)
```

> 💡 **Interview tip:** A common interview trick: "Are CloudTrail and CloudWatch the same thing?" They are NOT. CloudWatch = operational metrics (CPU, memory, request count). CloudTrail = API audit log (who called what API). The analogy: CloudWatch is like your server health monitor, CloudTrail is like your security camera footage. For compliance (PCI-DSS, HIPAA, SOC2), **CloudTrail must be enabled in all regions** with log file integrity validation — this proves logs weren't tampered with. AWS Config + Config Rules is the **automated compliance checker** — define rules once, AWS continuously checks all resources and reports non-compliant ones.

---

### Q54 — Git | Scenario-Based | Advanced

> Your team uses **GitFlow**. A critical bug is found in production. Fix it immediately without waiting for the feature branch cycle, deploy to production, and ensure the fix is in the development branch. Walk through the **exact Git commands**.

📁 **Reference:** `nawab312/CI_CD` → `Git` — GitFlow hotfix workflow sections

#### Key Points to Cover:
```
GitFlow hotfix workflow:

# Current state:
# main   → production code (v1.2.0)
# develop → next release work
# feature/new-auth → in-progress feature

# Step 1: Create hotfix branch FROM main (not develop):
git checkout main
git pull origin main
git checkout -b hotfix/fix-payment-null-pointer

# Step 2: Fix the bug:
# ... make code changes ...
git add src/payment/PaymentService.java
git commit -m "fix: resolve null pointer in payment processing"

# Step 3: Bump version (hotfix = patch version):
# Update version: 1.2.0 → 1.2.1
git add version.txt
git commit -m "chore: bump version to 1.2.1"

# Step 4: Merge hotfix into MAIN:
git checkout main
git merge --no-ff hotfix/fix-payment-null-pointer
git tag -a v1.2.1 -m "Hotfix: fix payment null pointer"
git push origin main
git push origin v1.2.1
# → CI/CD pipeline triggers production deployment

# Step 5: Merge hotfix into DEVELOP (critical — don't lose the fix):
git checkout develop
git pull origin develop
git merge --no-ff hotfix/fix-payment-null-pointer
# Resolve conflicts if any (develop may have diverged)
git push origin develop

# Step 6: Delete hotfix branch:
git branch -d hotfix/fix-payment-null-pointer
git push origin --delete hotfix/fix-payment-null-pointer

# Step 7: Verify on develop that fix is there:
git log develop --oneline | head -5
# Should see: "fix: resolve null pointer in payment processing"

# Why hotfix branches from MAIN (not develop):
# develop has unreleased features that aren't ready for production
# hotfix from main = only the fix goes to prod, no half-baked features

# Emergency: if too urgent for CI/CD pipeline review:
# Pair with another engineer, do code review manually
# Deploy directly from hotfix branch
# Then follow merge steps above
```

> 💡 **Interview tip:** The critical step most candidates forget: **merging hotfix into BOTH main AND develop**. If you only merge into main, the fix is in production but the next release (from develop) will re-introduce the bug. The `--no-ff` flag creates a merge commit even when fast-forward is possible — this preserves the history of "this was a hotfix branch" in the log. Also: if develop has conflicts with the hotfix, resolve them carefully — the hotfix must apply cleanly to both branches without breaking develop's in-progress features.

---

### Q55 — Jenkins | Conceptual | Advanced

> Explain the difference between **Declarative** and **Scripted** Jenkins pipelines. Also explain what **Shared Libraries** are and give a real-world example.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — pipeline types and Shared Libraries sections

#### Key Points to Cover:
```
Declarative Pipeline:
  → Structured, opinionated syntax
  → Requires specific blocks: pipeline{}, agent{}, stages{}, steps{}
  → Built-in validation (syntax errors caught before run)
  → Easier to read, recommended for most use cases
  → Limited flexibility (within the Declarative syntax)

  pipeline {
    agent { kubernetes { yaml '...' } }
    environment { IMAGE_TAG = "${GIT_COMMIT[0..7]}" }
    stages {
      stage('Test') {
        steps { sh 'npm test' }
      }
      stage('Build') {
        steps { sh 'docker build .' }
      }
    }
    post {
      failure { slackSend channel: '#alerts', message: "FAILED" }
    }
  }

Scripted Pipeline:
  → Free-form Groovy code
  → Wraps everything in node{} block
  → Full Groovy language available
  → More flexible but harder to read
  → Use when Declarative can't express what you need

  node('agent-label') {
    stage('Test') {
      def result = sh(script: 'npm test', returnStatus: true)
      if (result != 0) {
        currentBuild.result = 'UNSTABLE'
        // complex logic here
      }
    }
  }

When to use Scripted:
  → Complex conditional logic
  → Dynamic stage generation
  → Advanced error handling
  → Most projects use Declarative; Scripted is for edge cases

Shared Libraries:
  Problem: 50 microservices each have their own Jenkinsfile
           All have the same Docker build + ECR push + K8s deploy logic
           Change needed → update 50 Jenkinsfiles

  Solution: Shared Library = reusable Groovy code in a Git repo

  # Structure:
  jenkins-shared-library/
    vars/
      buildAndPush.groovy    ← global function (call like a step)
      deployToK8s.groovy
    src/
      com/company/Utils.groovy ← class-based helpers

  # vars/buildAndPush.groovy:
  def call(Map config) {
    sh "docker build -t ${config.registry}/${config.image}:${config.tag} ."
    sh "docker push ${config.registry}/${config.image}:${config.tag}"
  }

  # In any Jenkinsfile across all repos:
  @Library('jenkins-shared-library') _
  pipeline {
    stages {
      stage('Build') {
        steps {
          buildAndPush(registry: '123.ecr.aws', image: 'myapp', tag: GIT_COMMIT)
        }
      }
    }
  }

  Benefits:
  → One change in library → all pipelines updated
  → Version the library (tag v1.2.3, pin pipelines to version)
  → Unit test Groovy code with JenkinsPipelineUnit
```

> 💡 **Interview tip:** Shared Libraries are the **DRY principle applied to Jenkins** — the most important Jenkins architectural pattern for large teams. The key selling point: security fixes in the build process (e.g., adding a Trivy scan step) get applied to ALL pipelines by updating one library, not 50 Jenkinsfiles. Version pinning (`@Library('lib@v1.2.3')`) prevents accidental breaking changes — test the new library version on one pipeline before rolling out to all. Always mention that vars/ functions are `def call(Map config)` pattern — this is the idiomatic Jenkins shared library entrypoint.

---

### Q56 — Kubernetes | Conceptual | Advanced

> Explain how **Kubernetes DNS** works. What happens step-by-step when a Pod requests `backend.production.svc.cluster.local`? What is **ndots** and how does it affect DNS performance?

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md`

#### Key Points to Cover:
```
Kubernetes DNS architecture:
  → CoreDNS runs as Deployment in kube-system namespace
  → Every Pod's /etc/resolv.conf points to CoreDNS ClusterIP
  → CoreDNS watches K8s API → builds DNS records for Services

DNS record types:
  Service:   backend.production.svc.cluster.local → ClusterIP
  Pod:       1-2-3-4.production.pod.cluster.local → Pod IP
  Headless:  returns individual Pod IPs (StatefulSet use case)

Step-by-step resolution of backend.production.svc.cluster.local:
  1. Pod makes DNS query
  2. Query sent to CoreDNS (IP from /etc/resolv.conf)
  3. CoreDNS receives: backend.production.svc.cluster.local
  4. CoreDNS checks: is this a known Service?
     → Yes: backend Service in production namespace exists
  5. Returns: ClusterIP of the backend Service (e.g., 10.96.100.50)
  6. Pod connects to ClusterIP → kube-proxy routes to Pod endpoint

ndots and the search path problem:
  /etc/resolv.conf in every Pod:
    search production.svc.cluster.local svc.cluster.local cluster.local
    options ndots:5

  ndots:5 means: if the query has fewer than 5 dots → try search domains first

  Example: querying "backend" (0 dots < 5):
    1. Try: backend.production.svc.cluster.local → ✅ found (1 query)
    
  Example: querying "google.com" (1 dot < 5):
    1. Try: google.com.production.svc.cluster.local → NXDOMAIN (fail)
    2. Try: google.com.svc.cluster.local → NXDOMAIN (fail)
    3. Try: google.com.cluster.local → NXDOMAIN (fail)
    4. Try: google.com → ✅ found (4 queries for 1 external lookup!)

  Performance impact:
  → Every external DNS lookup = 3 wasted queries before resolving
  → High-traffic pods making external calls → DNS latency spike

  Fixes:
  # Option 1: Use FQDN (trailing dot):
  curl http://backend.production.svc.cluster.local.  # 5 dots = direct

  # Option 2: Set ndots:1 in dnsConfig:
  spec:
    dnsConfig:
      options:
      - name: ndots
        value: "1"
    # Reduces search attempts for external domains

  # Option 3: Use internal service short names within same namespace
  # (pods in production namespace): curl http://backend  (0 dots, found immediately)
```

> 💡 **Interview tip:** The **ndots performance issue** is a real production problem that most candidates don't know about. In a microservice making 100 external API calls per second, the default ndots:5 means 300 wasted DNS queries per second. The fix for external calls: always use FQDN with trailing dot, or reduce ndots. The fix for internal calls: use short service names within the same namespace (they resolve with the first search domain). Mention **CoreDNS caching** — CoreDNS caches responses, so repeated queries for the same name are fast. The issue is the first lookup for each unique external domain.

---

### Q57 — Linux / Bash | Scenario-Based | Advanced

> Write a Bash script that: takes servers from `servers.txt`, SSHes to each **in parallel**, runs `df -h`, outputs a consolidated report showing servers where disk usage is **above 80%**, and handles SSH failures gracefully.

📁 **Reference:** `nawab312/DSA` → `Linux` — Bash scripting, SSH, parallel execution sections

#### Key Points to Cover:
```bash
#!/usr/bin/env bash
set -euo pipefail

SERVERS_FILE="${1:-servers.txt}"
THRESHOLD=80
RESULTS_DIR=$(mktemp -d)
MAX_PARALLEL=10
SSH_TIMEOUT=10

cleanup() { rm -rf "$RESULTS_DIR"; }
trap cleanup EXIT

check_server() {
    local server="$1"
    local result_file="$RESULTS_DIR/${server//\//_}.txt"

    # SSH with timeout, strict host key checking off for automation:
    if ! ssh -o ConnectTimeout="$SSH_TIMEOUT" \
             -o StrictHostKeyChecking=no \
             -o BatchMode=yes \
             "$server" "df -h" > "$result_file" 2>/dev/null; then
        echo "FAILED:$server:SSH_ERROR" > "$result_file"
        return 0  # don't fail script on individual server error
    fi

    # Parse df output — find partitions above threshold:
    local alerts=""
    while IFS= read -r line; do
        # Skip header line:
        [[ "$line" =~ ^Filesystem ]] && continue
        # Extract usage percentage:
        local usage
        usage=$(echo "$line" | awk '{print $5}' | tr -d '%')
        local mount
        mount=$(echo "$line" | awk '{print $6}')
        if [[ "$usage" =~ ^[0-9]+$ ]] && [ "$usage" -ge "$THRESHOLD" ]; then
            alerts+="  ⚠️  ${mount}: ${usage}%\n"
        fi
    done < "$result_file"

    if [ -n "$alerts" ]; then
        echo "ALERT:$server"
        echo -e "$alerts"
    fi
}

export -f check_server
export RESULTS_DIR THRESHOLD SSH_TIMEOUT

# Validate input file:
if [ ! -f "$SERVERS_FILE" ]; then
    echo "ERROR: $SERVERS_FILE not found" >&2
    exit 1
fi

echo "============================================"
echo "Disk Usage Report — $(date '+%Y-%m-%d %H:%M:%S')"
echo "Threshold: ${THRESHOLD}%"
echo "============================================"
echo ""

# Run in parallel using xargs:
mapfile -t servers < "$SERVERS_FILE"
total="${#servers[@]}"
echo "Checking $total servers (max $MAX_PARALLEL parallel)..."
echo ""

printf '%s\n' "${servers[@]}" | \
    xargs -P "$MAX_PARALLEL" -I{} bash -c 'check_server "$@"' _ {}

echo ""
echo "============================================"
echo "Scan complete. Servers checked: $total"

# Count results:
failed=$(grep -rl "FAILED:" "$RESULTS_DIR" 2>/dev/null | wc -l)
echo "SSH failures: $failed"
echo "============================================"
```

> 💡 **Interview tip:** Three things make this production-quality: (1) **`-o BatchMode=yes`** in SSH prevents password prompts hanging the script when key auth fails — it immediately returns non-zero. (2) **`xargs -P`** for parallelism is cleaner than managing background jobs manually — it handles the concurrency limit automatically. (3) **`mapfile -t`** for reading the servers file is safer than `for server in $(cat file)` — it handles spaces in hostnames and preserves the array structure. Always mention `ConnectTimeout` — without it, an unreachable server hangs for the OS default TCP timeout (90+ seconds).

---

### Q58 — AWS | Troubleshooting | Advanced

> Your application on AWS is experiencing **high latency and increased error rates** during business hours. Infrastructure looks healthy — EC2 running, RDS available, no alarms firing. Users are complaining. Where would you look?

📁 **Reference:** `nawab312/AWS` — CloudWatch, ALB, RDS Performance Insights, X-Ray sections

#### Key Points to Cover:
```
Step 1 — ALB metrics (first stop):
  CloudWatch → ApplicationELB:
  → TargetResponseTime: is p99 high? (shows backend slowness)
  → HTTPCode_Target_5XX_Count: backend errors?
  → HTTPCode_ELB_5XX_Count: ELB itself erroring?
  → ActiveConnectionCount: connection exhaustion?
  → RejectedConnectionCount: hitting surge queue limit?

Step 2 — RDS Performance Insights:
  → Top SQL queries by load (which query is slow?)
  → DBLoad: is DB CPU maxed during business hours?
  → Wait events: "lock" wait? "io" wait?
  → ActiveSessions vs max_connections
  → Slow query log in CloudWatch Logs

Step 3 — EC2 / Application level:
  CloudWatch → EC2:
  → CPUUtilization: maxed out?
  → NetworkIn/Out: bandwidth saturation?
  → StatusCheckFailed: instance health?

  Application logs in CloudWatch Logs:
  → Log Insights query:
    fields @timestamp, @message
    | filter @message like "ERROR"
    | stats count() as errors by bin(5m)
    | sort @timestamp desc
  → Are errors concentrated in specific code paths?

Step 4 — AWS X-Ray (distributed trace):
  → Enable X-Ray on Lambda / EC2 / ECS
  → Service map shows latency per service
  → Trace viewer shows which downstream call is slow
  → "Is slowness in app code, DB query, or external API?"

Step 5 — Time correlation:
  → "During business hours" → likely user load driven
  → Check ASG scaling: did it scale? did scale take too long?
  → Check RDS: read replicas handling read load?
  → Check ElastiCache: cache hit rate dropping under load?

Step 6 — Infrastructure not directly monitored:
  → External API your app calls: their status page?
  → DNS TTL: are clients getting stale IPs?
  → Keep-alive timeout: HTTP connections being dropped?

Common root causes for "healthy infra, unhappy users":
  → RDS connection pool exhaustion (not visible in standard metrics)
  → Slow N+1 DB queries (fast individually, slow at scale)
  → Missing index on RDS (query plan changes under load)
  → Cache stampede (all caches expired simultaneously)
  → Thread pool exhaustion in app (not a CPU issue)
```

> 💡 **Interview tip:** "Infrastructure looks healthy but users are unhappy" is a classic SRE scenario. The key insight: **standard alarms don't catch everything**. CPU at 30% doesn't mean the app is healthy — if the app has 100 threads and 100 are blocked waiting for DB connections, CPU is low but every request times out. This is why **RDS Performance Insights** and **X-Ray** are essential — they show what's happening INSIDE the application layer that CloudWatch metrics can't see. Always end with X-Ray as the definitive answer: it shows the actual request trace and exactly where time is spent.

---

### Q59 — ArgoCD | Scenario-Based | Advanced

> Set up a GitOps workflow with ArgoCD for a microservice: 3 environments (dev/staging/prod), Helm charts, manual approval for prod, different values per environment. How do you structure the Git repo and configure ArgoCD?

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — Helm, multi-environment, sync policies sections

#### Key Points to Cover:
```
Git repository structure (two-repo pattern — recommended):
  app-repo/          ← application source code + Helm chart
    Chart.yaml
    templates/
    values.yaml      ← default values
    values-dev.yaml
    values-staging.yaml
    values-prod.yaml

  gitops-repo/       ← ArgoCD manifests (separate repo)
    apps/
      payment-service-dev.yaml
      payment-service-staging.yaml
      payment-service-prod.yaml

ArgoCD Application for each environment:

# payment-service-dev.yaml:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service-dev
  namespace: argocd
spec:
  project: dev-project
  source:
    repoURL: https://github.com/org/app-repo
    targetRevision: main          # track main branch
    path: .
    helm:
      valueFiles:
      - values.yaml
      - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: payment-dev
  syncPolicy:
    automated:                    # AUTO sync for dev
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true

# payment-service-prod.yaml:
spec:
  source:
    targetRevision: v1.2.3        # PINNED tag for prod (not main)
    helm:
      valueFiles:
      - values.yaml
      - values-prod.yaml
  syncPolicy:
    automated: null               # NO auto-sync for prod (manual only)
    # Prod: manually click Sync in ArgoCD UI after approval

# ArgoCD Project for access control:
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: prod-project
spec:
  destinations:
  - namespace: payment-prod
    server: https://prod-cluster.example.com
  sourceRepos:
  - 'https://github.com/org/app-repo'
  roles:
  - name: deployer
    policies:
    - p, proj:prod-project:deployer, applications, sync, prod-project/*, allow
    groups:
    - platform-team     # only platform-team can sync to prod

# Environment differences via values files:
# values-dev.yaml:    replicas: 1, cpu: 100m, image.tag: main-abc1234
# values-staging.yaml: replicas: 2, cpu: 500m, image.tag: v1.2.3-rc1
# values-prod.yaml:    replicas: 5, cpu: 2000m, image.tag: v1.2.3

# CI pipeline updates image tag in values files:
# dev: auto-commit new SHA to values-dev.yaml → ArgoCD auto-syncs
# prod: PR to update values-prod.yaml → human reviews → merges → manual sync
```

> 💡 **Interview tip:** The **two-repo pattern** (app code in one repo, ArgoCD manifests in another) is the production standard. Benefits: (1) Kubernetes manifests have their own history separate from app code, (2) different teams can have different permissions to each repo, (3) no circular dependency (CI commits to gitops-repo, ArgoCD reads gitops-repo). The **sync policy difference** between environments is critical: dev = `automated: {prune: true, selfHeal: true}` (any merge to main deploys instantly), prod = `automated: null` (human must click Sync after reviewing changes). Use ArgoCD Projects + RBAC to ensure only the platform team can trigger prod syncs.

---

### Q60 — Terraform | Troubleshooting | Advanced

> You run `terraform plan` and see **unexpected resource replacements** — Terraform wants to destroy and recreate resources you didn't change. What are the possible reasons and how do you investigate and prevent them?

📁 **Reference:** `nawab312/Terraform` — state management, lifecycle, `ignore_changes` sections

#### Key Points to Cover:
```
Common causes of unexpected replacements:

1. ForceNew attribute changed:
   → Some attributes are marked "ForceNew" — changing them requires recreation
   → Common: aws_instance ami, availability_zone, subnet_id
   → Plan shows: "forces replacement"
   → Fix: use ignore_changes if the attribute changes outside Terraform

2. Resource drifted (manually changed in AWS):
   → Someone changed a resource in console
   → Terraform state says X, AWS has Y → forces replacement
   → Diagnose: terraform refresh, then terraform plan

3. State file mismatch:
   → State says resource exists with old config
   → .tf file has new config
   → Terraform computes diff → replacement
   → Diagnose: terraform state show aws_instance.web

4. Provider version upgrade:
   → New provider version changes attribute handling
   → Fields that were ignored now cause diffs
   → Diagnose: check provider changelog

5. Random IDs or timestamps in resource names:
   → resource "aws_iam_role" "app" { name = "role-${uuid()}" }
   → uuid() generates new value each plan → forces replacement
   → Fix: use static name or store in local

6. Computed values stored differently:
   → AWS returns value in different format than Terraform expects
   → Example: tags stored as map vs list

Investigation steps:
  # Step 1: Read the plan carefully:
  terraform plan 2>&1 | grep -A5 "forces replacement"

  # Step 2: Check what attribute is causing replacement:
  # Plan shows: "~ subnet_id: "subnet-old" -> "subnet-new" # forces replacement"

  # Step 3: Check current state vs real AWS:
  terraform state show aws_instance.web

  # Step 4: If drift, run refresh:
  terraform refresh
  terraform plan  # see if replacement goes away

Fixes:
  # lifecycle ignore_changes (most common fix):
  resource "aws_instance" "web" {
    lifecycle {
      ignore_changes = [
        ami,          # don't replace when AMI ID changes
        tags["LastUpdated"],  # ignore auto-set tags
      ]
    }
  }

  # For auto-scaling resources, ignore replicas:
  resource "aws_ecs_service" "app" {
    lifecycle {
      ignore_changes = [desired_count]  # HPA manages this
    }
  }
```

> 💡 **Interview tip:** The `lifecycle { ignore_changes }` block is the **most used Terraform fix** in production. The most common scenario: AWS automatically adds tags (like `aws:cloudformation:stack-name`) to resources after creation, Terraform doesn't know about them, sees a diff → wants to recreate. Fix: `ignore_changes = [tags]`. Another classic: **ECS desired_count** — Terraform sets it to 3 in config, but autoscaling changes it to 5 at runtime. Next `terraform plan` sees 3 vs 5 → wants to set back to 3. Fix: `ignore_changes = [desired_count]`. The `# forces replacement` comment in `terraform plan` output is your first signal — always read it before applying.

---

### Q61 — Kubernetes | Scenario-Based | Advanced

> Your team is migrating a **PostgreSQL database** (500GB) from a VM to Kubernetes with **minimal downtime**. Walk through your migration strategy, Kubernetes resources used, and how you minimize downtime.

📁 **Reference:** `nawab312/Kubernetes` → `04_STORAGE.md`, `02_WORKLOADS.md`

#### Key Points to Cover:
```
Strategy: Phased migration with streaming replication to minimize downtime

Phase 1 — Set up K8s PostgreSQL (days before cutover):
  # Use StatefulSet with PVC:
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: postgres
  spec:
    serviceName: "postgres"
    replicas: 1
    volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: gp3-encrypted
        resources:
          requests:
            storage: 600Gi    # > 500GB current
    template:
      spec:
        containers:
        - name: postgres
          image: postgres:15
          resources:
            requests:
              memory: 8Gi
              cpu: "2"
            limits:
              memory: 16Gi
              cpu: "4"

  # StorageClass for EBS gp3:
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: gp3-encrypted
  provisioner: ebs.csi.aws.com
  parameters:
    type: gp3
    encrypted: "true"

Phase 2 — Initial data load (while app still on VM):
  # pg_dump → restore to K8s PostgreSQL
  # For 500GB: use pg_basebackup (faster than pg_dump):
  pg_basebackup -h vm-postgres -U replicator \
    -D /postgres-data -P -Xs -R
  # This creates a base backup AND sets up streaming replication

Phase 3 — Set up streaming replication VM → K8s:
  # K8s PostgreSQL becomes a REPLICA of VM PostgreSQL
  # Continuously applying WAL changes
  # Lag: typically < 1 second

Phase 4 — Verify K8s replica is caught up:
  # Check replication lag:
  SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
  # Should be < 1 second

Phase 5 — Cutover (minimal downtime window):
  # T+0: Stop writes to VM PostgreSQL (maintenance mode in app)
  # T+10s: Verify K8s replica is fully caught up (lag = 0)
  # T+15s: Promote K8s replica to primary:
  SELECT pg_promote();
  # T+20s: Update app connection string to K8s service
  # T+30s: Start writes to K8s PostgreSQL
  # TOTAL DOWNTIME: ~30 seconds

Phase 6 — Validation:
  # Test read/write on K8s PostgreSQL
  # Monitor: connections, query latency, replication (if adding replicas)
  # Keep VM PostgreSQL as emergency fallback for 24 hours

Key K8s resources used:
  StatefulSet:            stable pod identity for postgres-0
  Headless Service:       postgres-0.postgres.namespace stable DNS
  PVC + StorageClass:     persistent encrypted EBS storage
  Secret:                 postgres credentials (or use Vault)
  ConfigMap:              postgresql.conf tuning parameters
```

> 💡 **Interview tip:** The **streaming replication approach** is the correct answer — it reduces downtime to ~30 seconds regardless of database size. The naive approach (dump → restore → switch) takes hours of downtime for 500GB. The key: set up K8s PostgreSQL as a replica WHILE the VM is still the primary, let it catch up fully, then promote — the only downtime is the seconds between stopping writes and promoting. Also mention: use the **CloudNativePG operator** in production rather than managing a raw StatefulSet — it handles replication, failover, backup, and connection pooling automatically.

---

### Q62 — ELK Stack | Scenario-Based | Advanced

> Your **Elasticsearch cluster** is showing **RED health status**. Kibana shows no data and Logstash is reporting indexing failures. Walk through your complete investigation and recovery.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack`

#### Key Points to Cover:
```
Step 1 — Check cluster health:
  curl http://localhost:9200/_cluster/health?pretty
  # RED: at least one primary shard unassigned
  # YELLOW: all primaries assigned, some replicas unassigned
  # GREEN: all shards assigned

  curl http://localhost:9200/_cluster/health?level=indices&pretty
  # Shows health per index → which index is RED?

Step 2 — Find unassigned shards:
  curl "http://localhost:9200/_cat/shards?v&h=index,shard,prirep,state,unassigned.reason"
  # Shows exactly which shards are unassigned and WHY
  # UNASSIGNED reasons: NODE_LEFT, ALLOCATION_FAILED, INDEX_CREATED

Step 3 — Get allocation explanation:
  curl -X GET "localhost:9200/_cluster/allocation/explain?pretty" \
    -H 'Content-Type: application/json' \
    -d '{"index":"my-index","shard":0,"primary":true}'
  # This is the most useful command — explains exactly why shard isn't allocated
  # Common: "no valid shard copy found" → data node lost AND no replica

Step 4 — Check node status:
  curl "http://localhost:9200/_cat/nodes?v"
  # Is a data node missing? (node failed, took shards with it)
  # If no replica exists for those shards → RED

Step 5 — Recovery options:

  Option A: Node comes back (wait):
    → If node just rebooted → wait for it to rejoin
    → Shards will be reassigned automatically

  Option B: Restore from snapshot:
    # Check available snapshots:
    curl "http://localhost:9200/_snapshot/my-repo/_all?pretty"
    # Restore specific index:
    curl -X POST "localhost:9200/_snapshot/my-repo/snapshot-1/_restore" \
      -H 'Content-Type: application/json' \
      -d '{"indices": "my-index-2024.01"}'

  Option C: Force allocate (data loss risk — last resort):
    curl -X POST "localhost:9200/_cluster/reroute" \
      -d '{"commands":[{"allocate_stale_primary":{
        "index":"my-index","shard":0,"node":"node-1",
        "accept_data_loss":true}}]}'
    # WARNING: uses potentially stale shard — some data may be lost

Step 6 — Fix Logstash indexing failures:
  # Check Logstash logs:
  journalctl -u logstash -n 100
  # Common: "index_closed_exception" or "index_not_found_exception"
  # Fix: ensure index template exists, cluster is GREEN before retrying

Step 7 — Prevention:
  → Always configure replicas: number_of_replicas: 1 (minimum)
  → Configure S3 snapshot policy: daily snapshots
  → Monitor: elasticsearch_cluster_health_status != 0 → alert
  → Set up ILM (Index Lifecycle Management) to prevent disk full
```

> 💡 **Interview tip:** RED cluster status = **primary shard unassigned** = that data is currently inaccessible. The fastest diagnostic: `_cluster/allocation/explain` — this single API call tells you exactly why the shard isn't allocating (not enough disk space, node missing, shard corrupted). The most common production RED scenario: a data node was terminated and it held the ONLY copy of some shards (replica count was 0). This is why `number_of_replicas: 1` is non-negotiable for production. Always mention: **configure automatic snapshots to S3 from day one** — they're your safety net when cluster recovery isn't possible.

---

### Q63 — GitHub Actions | Conceptual | Advanced

> Explain the difference between **`jobs`** and **`steps`** in GitHub Actions. When do jobs run in parallel vs sequentially? Explain **reusable workflows** and **composite actions** — what they are and when to use each.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — workflow structure sections

#### Key Points to Cover:
```
Jobs vs Steps:
  Step:
    → Individual task within a job
    → Runs sequentially (one after another in the job)
    → Shares: runner, filesystem, environment variables
    → Types: uses (action) or run (shell command)

  Job:
    → Group of steps running on ONE runner
    → By DEFAULT: all jobs run in PARALLEL
    → Must declare needs: [other-job] to run sequentially
    → Each job gets a FRESH runner (no shared filesystem)
    → Communication between jobs: via outputs + artifacts

  # Parallel (default — both jobs start simultaneously):
  jobs:
    lint:        # starts immediately
      steps: ...
    unit-test:   # starts immediately (parallel with lint)
      steps: ...
    deploy:
      needs: [lint, unit-test]  # waits for both to succeed
      steps: ...

Reusable Workflows:
  → A complete workflow file called from another workflow
  → Trigger: workflow_call
  → Can have: multiple jobs, complex logic, secrets passing
  → Defined in: .github/workflows/reusable-build.yml
  → Called from: any repo with permissions

  # reusable-build.yml (called workflow):
  on:
    workflow_call:
      inputs:
        image-name:
          type: string
          required: true
      secrets:
        ECR_ROLE_ARN:
          required: true
  jobs:
    build-and-push:
      runs-on: ubuntu-latest
      steps:
      - uses: actions/checkout@v4
      - name: Build and push to ECR
        run: docker build -t ${{ inputs.image-name }} .

  # caller-workflow.yml:
  jobs:
    call-build:
      uses: myorg/.github/.github/workflows/reusable-build.yml@main
      with:
        image-name: myapp
      secrets:
        ECR_ROLE_ARN: ${{ secrets.ECR_ROLE_ARN }}

Composite Actions:
  → Custom action made of multiple steps (no jobs)
  → Defined in: action.yml in a repo
  → Called as a single step with uses:
  → Cannot have multiple jobs or complex workflow features

  # .github/actions/setup-node/action.yml:
  name: 'Setup Node'
  inputs:
    node-version:
      default: '20'
  runs:
    using: composite
    steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
    - run: npm ci
      shell: bash
    - run: npm run lint
      shell: bash

  # Used as a single step:
  steps:
  - uses: ./.github/actions/setup-node
    with:
      node-version: '20'

When to use which:
  Composite action:    reuse STEPS across jobs in same/different repos
                       → small, focused, step-level reuse
  Reusable workflow:   reuse entire JOB SEQUENCES including deployment logic
                       → large, complete pipeline reuse with secrets/environments
  Custom Docker action: when you need custom environment not available in runners
```

> 💡 **Interview tip:** The clearest distinction: **composite action = reusable steps**, **reusable workflow = reusable jobs**. If you need to share "checkout + setup node + npm ci" (3 steps) across 20 repos, use a composite action. If you need to share "test → build → push to ECR → deploy to staging" (a full pipeline with multiple jobs, environments, and secrets), use a reusable workflow. The biggest benefit of reusable workflows: **centralized security scanning** — define a `security-scan.yml` reusable workflow, call it from all 50 repos. When you add a new scan (e.g., Trivy), one change propagates everywhere.

---

### Q64 — AWS | Scenario-Based | Advanced

> Your company wants to implement a **disaster recovery strategy** for a critical application in `us-east-1`. **RTO is 1 hour**, **RPO is 15 minutes**. What strategy, services, and testing approach would you use?

📁 **Reference:** `nawab312/AWS` — Disaster Recovery, RDS, Route53, S3 Cross-Region Replication sections

#### Key Points to Cover:
```
DR Strategies (choose based on RTO/RPO):
  Backup & Restore:    RTO hours-days,  RPO hours    — cheapest
  Pilot Light:         RTO 10-60 min,   RPO minutes  — matches requirement
  Warm Standby:        RTO minutes,     RPO seconds  — more expensive
  Multi-Site Active:   RTO seconds,     RPO near-zero — most expensive

RTO=1hr + RPO=15min → Pilot Light strategy:
  → Keep minimal "pilot light" running in us-west-2
  → Scale up on DR event to full capacity

Architecture:

  Primary (us-east-1):          DR (us-west-2):
  ──────────────────            ──────────────────
  ALB                           ALB (ready, no traffic)
  EC2 ASG (full size)           EC2 Launch Template (no instances yet)
  RDS Primary                   RDS Read Replica (auto-syncing)
  ElastiCache                   ElastiCache (not running)
  S3 Bucket                     S3 Bucket (Cross-Region Replication)
  ECR Images                    ECR Replicated

RDS setup (RPO = 15 min):
  → RDS Multi-AZ in us-east-1 (HA within region)
  → RDS Read Replica in us-west-2 (cross-region, lag < 15 min typically)
  → On DR: promote replica to primary (takes ~3 min)

S3 Cross-Region Replication:
  aws s3api put-bucket-replication --bucket primary-bucket \
    --replication-configuration '{
      "Role": "arn:aws:iam::123:role/replication-role",
      "Rules": [{"Status":"Enabled",
        "Destination":{"Bucket":"arn:aws:s3:::dr-bucket",
        "ReplicaKmsKeyID":"arn:aws:kms:us-west-2:..."}}]}'

Route53 failover:
  # Health check on primary ALB:
  Primary record:  app.company.com → us-east-1-alb (Primary routing policy)
  DR record:       app.company.com → us-west-2-alb (Secondary routing policy)
  # When health check fails → Route53 auto-switches to DR

DR Runbook (RTO = 1 hour):
  T+0:    DR declared, start runbook
  T+5:    Promote RDS replica in us-west-2
  T+10:   Scale up EC2 ASG in us-west-2 (Launch Template scales to full size)
  T+20:   Validate app connectivity in DR region
  T+30:   Update Route53 if health check didn't auto-failover
  T+60:   Full DR validation complete (within RTO)

DR Testing:
  → Quarterly: tabletop exercise (walk through runbook)
  → Semi-annual: actual DR drill (fail traffic to us-west-2)
  → Chaos Engineering: AWS Fault Injection Simulator
  → Never test in production business hours
```

> 💡 **Interview tip:** Always **match the DR strategy to the RTO/RPO** — don't over-engineer. RTO=1hr and RPO=15min is achievable with Pilot Light (much cheaper than Warm Standby). The most critical component: **automated Route53 health check failover** — without automation, your 1-hour RTO is at risk of human error and slow response. Also mention **DR testing is as important as the DR plan** — a plan that has never been tested will fail when you need it most. Netflix's approach: run Chaos Monkey continuously so failures are discovered before they become disasters.

---

### Q65 — Prometheus | Troubleshooting | Advanced

> Your Prometheus is consuming excessive memory and crashing with OOM. The `/metrics` endpoint shows extremely high `prometheus_tsdb_head_series` count. What is causing this and how do you fix it?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — TSDB, cardinality sections

#### Key Points to Cover:
```
Root cause: HIGH CARDINALITY

prometheus_tsdb_head_series = number of unique time series in memory
Normal: 10,000 - 100,000 series
Problem: 1,000,000+ series → OOM

Each unique combination of label values = 1 time series
Memory per series: ~3KB average

10,000 series  = ~30MB  (fine)
1,000,000 series = ~3GB (OOM risk)

Common cardinality explosions:
  1. User ID in label:
     http_requests_total{user_id="user-123456"}  ← NEVER DO THIS
     1M users = 1M series for ONE metric

  2. Unbounded dynamic labels:
     http_requests_total{url="/api/users/123/orders"}  ← NEVER
     normalize to: {route="/api/users/:id/orders"}

  3. Request ID / trace ID in label:
     request_latency{request_id="abc-123-def"}  ← NEVER

  4. Too many targets × too many metrics:
     1,000 pods × 500 metrics = 500,000 series (can be legitimate)

Diagnosis:
  # Find top metrics by cardinality:
  curl -s http://prometheus:9090/api/v1/label/__name__/values | \
    jq -r '.data[]' | while read metric; do
      count=$(curl -s "http://prometheus:9090/api/v1/query?query=count($metric)" | \
        jq -r '.data.result[0].value[1] // 0')
      echo "$count $metric"
    done | sort -rn | head -20

  # Or use Prometheus Cardinality Explorer (UI > Status > TSDB Status)
  # Shows: top metrics by series count, top label names by series count

Fixes:
  1. Remove high-cardinality labels:
     # In app code: don't add user_id/request_id to Prometheus labels
     # In relabeling: drop the label before storing

  2. metric_relabel_configs to drop problematic labels:
     scrape_configs:
     - job_name: myapp
       metric_relabel_configs:
       - source_labels: [__name__]
         regex: 'http_requests_total'
         action: keep
       - regex: 'user_id|request_id|trace_id'
         action: labeldrop   # drop these labels before storing

  3. Reduce retention / increase resources:
     --storage.tsdb.retention.time=15d  # shorter retention = less memory
     --storage.tsdb.max-block-duration=2h

  4. Recording rules to pre-aggregate:
     # Instead of querying 1M series per dashboard load:
     - record: job:http_requests:rate5m
       expr: sum by(job)(rate(http_requests_total[5m]))
     # Now dashboard queries 1 series instead of 1M
```

> 💡 **Interview tip:** High cardinality is **the #1 Prometheus scalability problem** and the answer that separates junior from senior knowledge. The key rule: **labels must have bounded cardinality** — if a label value can be one of millions of possibilities (user ID, request ID, URL path), it will OOM your Prometheus. The solution has two parts: (1) fix at the source (don't instrument with high-cardinality labels), (2) fix at ingest with `metric_relabel_configs` `labeldrop`. Always check `TSDB Status` in the Prometheus UI first — it shows exactly which metric and which label is the cardinality bomb.

---

### Q66 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes Service Mesh** and specifically **Istio**. What problems does it solve that standard Kubernetes networking cannot? Explain **mTLS**, **traffic shifting**, and **circuit breaking** in the context of Istio.

📁 **Reference:** `nawab312/Kubernetes` → `11_ISTIO_SERVICE_MESH.md`

#### Key Points to Cover:
```
What standard K8s networking lacks:
  → No encryption between pods (traffic is plaintext)
  → No built-in traffic management (weighted routing)
  → No circuit breaking
  → No observability at L7 (HTTP/gRPC) layer
  → Security requires application-level implementation

Istio architecture:
  Control Plane: istiod (Pilot + Citadel + Galley)
  Data Plane:    Envoy sidecar injected into every pod
  → Sidecar intercepts ALL pod network traffic (transparent)
  → App doesn't know Istio exists — zero code changes

mTLS (Mutual TLS):
  Standard TLS: client verifies server identity (HTTPS)
  mTLS:         BOTH client AND server verify each other
  
  With Istio:
  → Envoy on pod A generates cert, Envoy on pod B generates cert
  → Certs issued by istiod (cluster CA)
  → All pod-to-pod traffic encrypted and mutually authenticated
  → Zero trust: even inside the cluster, identity verified

  PeerAuthentication (enforce mTLS):
  apiVersion: security.istio.io/v1beta1
  kind: PeerAuthentication
  metadata:
    name: default
    namespace: production
  spec:
    mtls:
      mode: STRICT  # reject non-mTLS traffic

Traffic Shifting (canary/blue-green):
  VirtualService routes traffic by weight:
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: payment-service
  spec:
    hosts: [payment-service]
    http:
    - route:
      - destination:
          host: payment-service
          subset: v1
        weight: 90
      - destination:
          host: payment-service
          subset: v2   # new version
        weight: 10   # 10% to canary

Circuit Breaking:
  Stops cascading failures — if Service B is slow, don't queue requests
  apiVersion: networking.istio.io/v1alpha3
  kind: DestinationRule
  spec:
    trafficPolicy:
      connectionPool:
        http:
          http1MaxPendingRequests: 100
          http2MaxRequests: 1000
      outlierDetection:
        consecutive5xxErrors: 5      # after 5 errors
        interval: 30s
        baseEjectionTime: 30s        # eject for 30 seconds
        maxEjectionPercent: 50

Observability (automatic with Istio):
  → L7 metrics: request rate, error rate, latency per service
  → Distributed tracing: Jaeger/Zipkin integration
  → Service graph: visualize all service dependencies
```

> 💡 **Interview tip:** The killer Istio feature for interviews: **zero-code mTLS**. Without Istio, adding mutual TLS between 20 microservices requires modifying all 20 services. With Istio, one `PeerAuthentication` YAML enables mTLS cluster-wide — no code change. This is the "why use a service mesh?" answer. Also mention the trade-off: Istio adds **~7ms latency per hop** and ~128MB memory per pod (Envoy sidecar). For latency-sensitive services, this overhead matters. The modern alternative: **Istio ambient mesh** (no sidecar) reduces resource overhead significantly.

---

### Q67 — Linux / Bash | Scenario-Based | Advanced

> You suspect a **disk I/O bottleneck** causing intermittent latency spikes on a production server. Walk through the Linux commands and tools to confirm and diagnose it.

📁 **Reference:** `nawab312/DSA` → `Linux` — I/O monitoring, iostat, iotop sections

#### Key Points to Cover:
```
Step 1 — Quick check (is I/O the issue?):
  top   # look at: %wa (iowait column)
        # iowait > 20% = CPU waiting for I/O = strong signal

  vmstat 1 5
  # Look at: wa column (iowait), b (blocked processes)
  # wa consistently > 10 = I/O saturation

Step 2 — Identify which disk:
  iostat -xz 1 5
  # Key columns:
  # %util = device utilization (>80% = saturated)
  # await = avg I/O wait time in ms (>10ms = problematic)
  # svctm = service time (hardware speed)
  # r/s, w/s = reads/writes per second
  # rkB/s, wkB/s = throughput

  # Example problematic output:
  # Device  r/s   w/s  rkB/s  wkB/s  await  %util
  # nvme0n1 0.0  250.0   0.0  2000.0  95.3   99.8
  # %util=99.8 = disk completely saturated

Step 3 — Identify which PROCESS is causing I/O:
  iotop -o -P
  # -o = only show processes doing I/O
  # -P = show processes not threads
  # Shows: TID, PRIO, USER, DISK READ, DISK WRITE, SWAPIN, IO%, COMMAND

  # Alternative: pidstat -d 1
  pidstat -d 1 5
  # Shows per-process disk read/write rates

Step 4 — Identify which FILES are being accessed:
  lsof | grep <pid-from-iotop>
  # Shows which files the heavy I/O process has open

  # For real-time file access:
  inotifywait -r /path/to/watch
  strace -p <pid> -e trace=read,write,open -c   # syscall I/O summary

Step 5 — Check for swap I/O (memory pressure causing disk I/O):
  free -h
  vmstat 1 | grep -v 0    # look at si/so columns (swap in/out)
  # If si/so non-zero = system is swapping = memory problem not disk

Step 6 — Check disk queue depth:
  cat /sys/block/nvme0n1/queue/nr_requests  # current queue depth
  iostat -x | grep -E "aqu-sz|avgqu-sz"    # average queue size
  # High queue = disk can't keep up with requests

Common fixes:
  → Write I/O: check for excessive log writes, use async writes
  → Read I/O: add caching layer (Redis, ElastiCache)
  → Swap I/O: increase RAM or reduce memory pressure
  → Both: upgrade to faster disk (gp2 → gp3, io1 for IOPS guarantee)
  → Check: noatime mount option (eliminates atime update writes)
```

> 💡 **Interview tip:** The diagnostic hierarchy: `top %wa` → `vmstat wa` → `iostat %util` → `iotop` → `lsof`. Each step narrows the problem: `%wa` confirms I/O is the issue, `iostat` shows which disk, `iotop` shows which process, `lsof` shows which files. The **`iostat %util`** column is the key saturation indicator — 100% means the disk has zero idle time and cannot keep up with requests. On cloud (EBS), also check the EBS CloudWatch metrics — `BurstBalance` dropping to 0 on `gp2` volumes is a common cause of sudden I/O degradation that `iostat` alone won't explain.

---

### Q68 — Terraform | Scenario-Based | Advanced

> Your company is moving from **manually managed AWS infrastructure** to Terraform. Hundreds of resources exist in AWS created manually via the console. How would you import all existing infrastructure without downtime or disruptions?

📁 **Reference:** `nawab312/Terraform` — `terraform import`, state management sections

#### Key Points to Cover:
```
Strategy: Gradual import with zero risk

Phase 1 — Inventory existing resources:
  # Use AWS Config or Tag Editor to list all resources
  aws resourcegroupstaggingapi get-resources \
    --output json | jq -r '.ResourceTagMappingList[].ResourceARN'

Phase 2 — Write Terraform configuration FIRST:
  # Must write the .tf resource block BEFORE importing
  # Otherwise import has nothing to put state into

  resource "aws_s3_bucket" "logs" {
    bucket = "company-logs-bucket"
    # Don't add attributes yet — keep minimal
    # After import, run plan to see actual values, add to config
  }

Phase 3 — Import resource into state:
  terraform import aws_s3_bucket.logs company-logs-bucket
  # Reads current state from AWS → writes to terraform.tfstate

Phase 4 — Generate config from state (TF 1.5+ import blocks):
  # Modern approach — add import block to .tf file:
  import {
    id = "company-logs-bucket"
    to = aws_s3_bucket.logs
  }
  # Then run:
  terraform plan -generate-config-out=generated.tf
  # Terraform generates the resource block automatically!
  # Review generated.tf, clean up, add to your config

Phase 5 — Run terraform plan — should show NO changes:
  terraform plan
  # Goal: "No changes. Infrastructure is up-to-date."
  # If changes shown: AWS config doesn't match your .tf file
  # Fix .tf to match AWS (not the other way — AWS is the truth)

Phase 6 — Handle computed/ignored attributes:
  resource "aws_security_group" "web" {
    name = "web-sg"
    lifecycle {
      ignore_changes = [description]  # set by AWS, can't change
    }
  }

Prioritize import order:
  1. Foundation: VPCs, Subnets, Route Tables, IGW
  2. Security: Security Groups, IAM Roles
  3. Compute: EC2, ASGs, Launch Templates
  4. Data: RDS, ElastiCache, S3
  5. App: EKS, ECS, Lambda

Tools to speed up import:
  # terraformer: auto-generates .tf + imports (bulk)
  terraformer import aws --resources=s3,ec2 --regions=us-east-1

  # AWS provider import blocks (TF 1.5+): batch imports in .tf file
  # Atlantis or Terraform Cloud: run imports via PR review process

Risks to mitigate:
  → Never run terraform apply mid-import (partial state = dangerous)
  → Test in dev account first before prod
  → Use -refresh=false flag during import runs (performance)
  → Keep existing tagging (Terraform will manage from now on)
```

> 💡 **Interview tip:** The most important point: **write the `.tf` config before running `terraform import`** — import only updates the state file, it does NOT generate config. Without a matching resource block, import fails. Terraform 1.5+ `import blocks` + `plan -generate-config-out` changed this — now Terraform can generate the config automatically. Mention **terraformer** as the bulk import tool — it introspects AWS and generates both the .tf config AND import commands for entire services. For large accounts, this cuts import effort from weeks to hours.

---

### Q69 — AWS | Conceptual | Advanced

> Explain the difference between **AWS ALB** and **AWS NLB**. When would you choose each? Explain **target groups** and how **health checks** work differently.

📁 **Reference:** `nawab312/AWS` — ALB, NLB, Load Balancing sections

#### Key Points to Cover:
```
ALB (Application Load Balancer) — Layer 7:
  → HTTP/HTTPS/gRPC/WebSocket aware
  → Routes based on: URL path, host header, HTTP method, query params
  → Terminates TLS (decrypts HTTPS, inspects headers, re-encrypts)
  → Integrates with: WAF, Cognito auth, Lambda targets
  → Static IP: NOT natively (use Global Accelerator)
  → Preserves: X-Forwarded-For header

  Use ALB for:
  ✅ Web apps, REST APIs, microservices
  ✅ Path-based routing (/api/* → service-a, /* → service-b)
  ✅ Host-based routing (api.company.com vs www.company.com)
  ✅ Authentication (Cognito/OIDC at load balancer level)
  ✅ WebSocket applications

NLB (Network Load Balancer) — Layer 4:
  → TCP/UDP/TLS — no HTTP awareness
  → Extremely high performance: millions of RPS, ultra-low latency
  → Preserves source IP (ALB changes source IP to its own)
  → Static IP per AZ (important for whitelisting)
  → Faster failover: seconds (ALB takes 30-60s to detect failures)
  → Supports: TCP_UDP, TCP passthrough (end-to-end TLS)

  Use NLB for:
  ✅ TCP/UDP protocols (non-HTTP): gaming, IoT, streaming
  ✅ Static IP requirement (firewall whitelisting)
  ✅ Extreme low latency requirements (<1ms overhead)
  ✅ Preserving source IP (NLB passes through, ALB doesn't)
  ✅ VPC endpoint services (PrivateLink — requires NLB)

Target Groups:
  → Define WHERE to route traffic (EC2, IP, Lambda, ALB)
  → ALB listener → rules → target group
  → One TG can have: EC2 + Fargate tasks + Lambda mixed
  → Target type: instance, ip, lambda, alb

Health Check differences:
  ALB health checks:
    → HTTP/HTTPS: sends actual GET request to path
    → Checks HTTP status code (200-399 = healthy)
    → More configurable: path, headers, matcher
    Protocol: HTTP, Path: /health, Matcher: 200-299

  NLB health checks:
    → TCP: just establishes connection (port open = healthy)
    → OR HTTP: similar to ALB
    → TCP health check: faster, less overhead
    → No HTTP inspection in TCP mode

ALB vs NLB quick decision:
  HTTP/HTTPS web app?          → ALB
  Need path-based routing?     → ALB
  Need WAF integration?        → ALB
  TCP/UDP non-HTTP protocol?   → NLB
  Need static IP?              → NLB
  Preserving source IP needed? → NLB
  PrivateLink service?         → NLB (required)
```

> 💡 **Interview tip:** The most commonly confused difference: **source IP preservation**. With ALB, the target instance sees the ALB's IP as the source (original IP in X-Forwarded-For header). With NLB, the target sees the client's actual IP directly. This matters for: IP-based rate limiting (needs real IP), security group rules (restrict by client IP), compliance logging (need real client IP in logs). Also: **NLB is required for AWS PrivateLink** — this is an important EKS pattern where you expose K8s services to other VPCs/accounts via PrivateLink.

---

### Q70 — Kubernetes | Troubleshooting | Advanced

> Pods are being **evicted frequently** even though `kubectl top nodes` shows available resources. What are the possible reasons and how do you investigate and stop them?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

#### Key Points to Cover:
```
Step 1 — Check eviction reason:
  kubectl describe pod <evicted-pod>
  # Look for: "The node was low on resource: memory/disk/inodes/pid"
  # Message shows exactly which resource triggered eviction

  kubectl get events --field-selector reason=Evicted -A
  kubectl get events --sort-by='.lastTimestamp' | grep Evict | tail -20

Step 2 — Understand why "available" resources != true availability:
  kubectl top nodes          # shows requests, not limits
  kubectl describe node      # shows allocatable vs actual usage

  # The gap:
  # Node has 8Gi RAM, 4Gi is "available" per kubectl top
  # But: kubelet has reserved 1Gi (--kube-reserved)
  # OS processes using 1Gi more
  # Real allocatable: 6Gi, real available: 2Gi

Eviction types:
  Soft eviction:
    → Usage above soft threshold for grace period
    → Graceful pod termination
    → Thresholds: memory.available<1Gi (evict if below 1Gi for 2 min)

  Hard eviction (immediate):
    → Usage above hard threshold (critical)
    → Immediate SIGKILL — no grace period
    → Thresholds: memory.available<100Mi (immediate kill)

Common causes of unexpected evictions:

1. Disk pressure (most common surprise):
   df -h on node  # is / or /var/lib/kubelet full?
   → Container logs filling /var/log
   → Old images not cleaned up
   Fix: docker image prune, log rotation, increase disk

2. Inode exhaustion:
   df -i  # disk usage 30% but inodes 100%
   → Many small files (temp files, log fragments)
   Fix: find and delete many small files

3. Memory limits not set (BestEffort pods evicted first):
   # Pods with no requests/limits = BestEffort QoS
   # First to be evicted when node is under pressure
   Fix: add requests and limits to ALL pods

4. Node reserved resources not accounting:
   # kubelet reserves: --kube-reserved, --system-reserved
   # These reduce allocatable memory
   kubectl describe node | grep Allocatable -A 10

5. Memory request vs actual usage mismatch:
   # Pod requests 100Mi but uses 900Mi
   # Scheduler thinks node has capacity → schedules → OOM → eviction
   Fix: set requests accurately based on actual usage (VPA recommendation)

Stop evictions:
  → Set accurate resource requests (match actual p99 usage)
  → Set limits = 2x requests (burst room)
  → Configure LimitRange for namespace defaults
  → Monitor: node memory/disk pressure before evictions start
  → Alert: kube_node_status_condition{condition="MemoryPressure"}==1
```

> 💡 **Interview tip:** The `kubectl top nodes` trap: it shows **requests** scheduled on the node, not **actual usage**. A node can show 60% used by `kubectl top` while being at 99% actual memory usage because pods are using far more than their requests. This is why `kubectl describe node` (which shows actual kubelet-reported usage) and `prometheus node_memory_MemAvailable_bytes` are more accurate than `kubectl top`. The fix: **set accurate resource requests** — this makes the scheduler's decisions accurate, preventing the false sense of available capacity.

---

### Q71 — Grafana | Scenario-Based | Advanced

> Build a Grafana dashboard that shows the **SLO** for your API: 99.9% availability and 95% of requests under 500ms. Write the **PromQL queries** for each SLO and explain how you would set up an **error budget** panel.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana`, `Prometheus`

#### Key Points to Cover:
```
SLO Definitions:
  Availability SLO: 99.9% = max 0.1% error rate = 43.2 min downtime/month
  Latency SLO: 95% of requests complete in under 500ms

PromQL for Availability SLO:
  # Current error rate (last 30 days rolling):
  1 - (
    sum(rate(http_requests_total{status!~"5.."}[30d]))
    / sum(rate(http_requests_total[30d]))
  )

  # Current availability (should be >= 0.999):
  sum(rate(http_requests_total{status!~"5.."}[30d]))
  / sum(rate(http_requests_total[30d]))

PromQL for Latency SLO:
  # % of requests within 500ms (last 1 hour):
  sum(rate(http_request_duration_seconds_bucket{le="0.5"}[1h]))
  / sum(rate(http_request_duration_seconds_count[1h]))
  # Should be >= 0.95

Error Budget Calculation:
  Error budget = allowed errors over time period
  For 99.9% SLO over 30 days:
    Budget = 1 - 0.999 = 0.1% = 43.2 minutes

  Budget consumed:
  1 - (
    (sum(rate(http_requests_total{status!~"5.."}[30d]))
     / sum(rate(http_requests_total[30d])))
    / 0.999  # divide by SLO target
  )
  # Returns: 0.0 = no budget consumed, 1.0 = all budget consumed

  Budget remaining (in minutes for 30 days):
  (
    1 - (
      sum(increase(http_requests_total{status=~"5.."}[30d]))
      / sum(increase(http_requests_total[30d]))
    ) / (1 - 0.999)
  ) * 43.2  # 43.2 minutes = full budget

Grafana Dashboard panels:
  Panel 1 (Stat): Current Availability %
    → threshold: green >= 99.9%, yellow >= 99%, red < 99%
    → value: availability query
  
  Panel 2 (Gauge): Error Budget Remaining
    → 0-100% gauge
    → green > 50%, yellow 10-50%, red < 10%
  
  Panel 3 (Time Series): Availability over 30 days
    → rolling window query
    → reference line at 99.9%
  
  Panel 4 (Stat): Budget Burn Rate
    → How fast is budget being consumed?
    → > 1x = consuming budget at SLO rate (neutral)
    → > 10x = need immediate action

Alert thresholds:
  Fast burn: budget_burn_rate > 14.4 for 1h (will exhaust in ~3 days)
  Slow burn: budget_burn_rate > 3 for 6h (will exhaust in 30 days)
```

> 💡 **Interview tip:** The **error budget concept** is what distinguishes SRE thinking from traditional ops. Error budget = "how much unreliability you're allowed." If your SLO is 99.9% and you've used 50% of your error budget in week 1, you need to slow down deployments or focus on reliability. If you end the month with 100% budget remaining, you might be being too conservative — ship more features. The budget burn rate alert pattern (fast burn + slow burn) catches both "fire right now" (fast burn, 14.4x) and "slow leak that will cause problems" (slow burn, 3x) scenarios.

---

### Q72 — Python | Scenario-Based | Advanced

> Write a Python script that: reads EC2 instance IDs from CSV, uses **boto3** to get state/type/tags for each, outputs a **formatted table** with Instance ID, State, Type, Name tag, Environment tag, and handles **API errors and rate limiting** gracefully.

📁 **Reference:** `nawab312/DSA` → `Python` — boto3, CSV handling, error handling sections

#### Key Points to Cover:
```python
#!/usr/bin/env python3
import boto3
import csv
import sys
import time
from botocore.exceptions import ClientError, EndpointConnectionError
from tabulate import tabulate

def get_instance_info(ec2_client, instance_id, max_retries=3):
    """Get EC2 instance info with retry on rate limiting."""
    for attempt in range(max_retries):
        try:
            response = ec2_client.describe_instances(
                InstanceIds=[instance_id]
            )
            reservations = response.get('Reservations', [])
            if not reservations:
                return None

            instance = reservations[0]['Instances'][0]

            # Extract tags as dict:
            tags = {
                tag['Key']: tag['Value']
                for tag in instance.get('Tags', [])
            }

            return {
                'Instance ID': instance_id,
                'State':       instance['State']['Name'],
                'Type':        instance['InstanceType'],
                'Name':        tags.get('Name', 'N/A'),
                'Environment': tags.get('Environment', 'N/A'),
            }

        except ClientError as e:
            error_code = e.response['Error']['Code']

            if error_code == 'RequestLimitExceeded':
                # Exponential backoff on rate limiting:
                wait_time = (2 ** attempt) + 1
                print(f"Rate limited. Waiting {wait_time}s...",
                      file=sys.stderr)
                time.sleep(wait_time)
                continue

            elif error_code == 'InvalidInstanceID.NotFound':
                print(f"Warning: Instance {instance_id} not found",
                      file=sys.stderr)
                return None

            else:
                print(f"AWS error for {instance_id}: {e}",
                      file=sys.stderr)
                return None

    print(f"Error: Max retries exceeded for {instance_id}",
          file=sys.stderr)
    return None


def read_instance_ids(csv_file):
    """Read instance IDs from CSV file."""
    instance_ids = []
    try:
        with open(csv_file, 'r') as f:
            reader = csv.DictReader(f)
            # Support both 'instance_id' and 'InstanceId' columns:
            for row in reader:
                iid = row.get('instance_id') or row.get('InstanceId')
                if iid and iid.strip():
                    instance_ids.append(iid.strip())
    except FileNotFoundError:
        print(f"Error: File '{csv_file}' not found", file=sys.stderr)
        sys.exit(1)
    return instance_ids


def main():
    csv_file = sys.argv[1] if len(sys.argv) > 1 else 'instances.csv'
    region = sys.argv[2] if len(sys.argv) > 2 else 'us-east-1'

    instance_ids = read_instance_ids(csv_file)
    if not instance_ids:
        print("No instance IDs found in CSV", file=sys.stderr)
        sys.exit(1)

    print(f"Fetching info for {len(instance_ids)} instances in {region}...")

    try:
        ec2 = boto3.client('ec2', region_name=region)
    except EndpointConnectionError as e:
        print(f"Cannot connect to AWS: {e}", file=sys.stderr)
        sys.exit(1)

    results = []
    for iid in instance_ids:
        info = get_instance_info(ec2, iid)
        if info:
            results.append(info)

    if results:
        print("\n" + tabulate(results, headers='keys', tablefmt='grid'))
        print(f"\nTotal: {len(results)} instances")
    else:
        print("No results found")


if __name__ == '__main__':
    main()
```

> 💡 **Interview tip:** Three things elevate this script: (1) **Exponential backoff** for rate limiting — `2^attempt + 1` gives 3s, 5s, 9s waits between retries, which is the AWS-recommended pattern. (2) **Tag extraction as dict** with `{tag['Key']: tag['Value']}` — more Pythonic and handles missing tags gracefully with `.get('Name', 'N/A')`. (3) **Separating concerns** into functions — `read_instance_ids` and `get_instance_info` are independently testable. Also mention using **`boto3` session + batch `describe_instances`** with multiple IDs in one call (up to 100 per call) instead of looping — this is 100x faster for large CSV files.

---

### Q73 — Jenkins | Troubleshooting | Advanced

> Jenkins master is **running out of disk space** on `/var/lib/jenkins`. Pipelines are failing with disk-related errors. What is consuming disk and how do you clean it up and prevent recurrence?

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — maintenance and cleanup sections

#### Key Points to Cover:
```
Step 1 — Find what's consuming space:
  du -sh /var/lib/jenkins/* | sort -rh | head -20
  # Typical offenders:
  # /var/lib/jenkins/jobs         ← build logs and artifacts (usually biggest)
  # /var/lib/jenkins/workspace    ← build workspaces (checked out code)
  # /var/lib/jenkins/plugins      ← Jenkins plugins
  # /var/lib/jenkins/caches       ← Maven/Gradle/npm caches

Step 2 — Find largest individual builds:
  du -sh /var/lib/jenkins/jobs/*/builds/* | sort -rh | head -20
  find /var/lib/jenkins/jobs -name "*.log" -size +100M

Step 3 — Immediate cleanup:
  # Clean workspaces for all jobs (safe, code re-checked-out on next build):
  # In Jenkins UI: Job → Workspace → Wipe out workspace
  # Via script console (Manage Jenkins → Script Console):
  Jenkins.instance.getAllItems(Job.class).each { job ->
    job.builds.each { build -> build.delete() }
  }

  # Clean old build artifacts and logs:
  find /var/lib/jenkins/jobs -name "archive" -type d \
    -mtime +30 -exec rm -rf {} +
  find /var/lib/jenkins/jobs -name "*.log" -mtime +30 -delete

Step 4 — Configure automatic build retention per job:
  # In Jenkinsfile:
  options {
    buildDiscarder(logRotator(
      numToKeepStr: '10',        // keep only last 10 builds
      artifactNumToKeepStr: '5', // keep artifacts from last 5 builds
      daysToKeepStr: '30',       // delete builds older than 30 days
      artifactDaysToKeepStr: '7' // delete artifacts older than 7 days
    ))
  }

  # In job config UI: Discard old builds → Days to keep / Max # of builds

Step 5 — Configure Global Build Discard (Manage Jenkins → System):
  # Global default for all jobs without their own settings

Step 6 — Move artifacts to external storage:
  # Instead of storing in Jenkins: push to S3, Nexus, Artifactory
  archiveArtifacts artifacts: '*.jar'     # stores in Jenkins (bad for disk)
  # Better:
  sh 'aws s3 cp target/*.jar s3://artifacts-bucket/builds/'

Step 7 — Workspace cleanup plugin:
  post {
    always {
      cleanWs()  // clean workspace after every build
    }
  }

Step 8 — Monitor disk:
  # Add CloudWatch alarm on Jenkins EC2:
  # DiskSpaceUtilization > 80% → alert → before it's critical
```

> 💡 **Interview tip:** The **`buildDiscarder` option** in Jenkinsfile is the most important preventive measure — every single pipeline should have it. The default in Jenkins is to keep ALL builds forever, which means disk fills up over months as builds accumulate. `numToKeepStr: '10'` keeps the last 10 builds and their logs. `artifactNumToKeepStr: '5'` keeps artifacts from only the last 5 builds (artifacts are usually much larger than logs). Also mention **moving artifacts out of Jenkins entirely** — Jenkins disk should only store logs, not build artifacts. Nexus/Artifactory/S3 are purpose-built for artifact storage with proper retention policies.

---

### Q74 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes Pod QoS classes** — Guaranteed, Burstable, and BestEffort. How does Kubernetes decide which class a Pod belongs to? What is the **eviction order** when a node runs out of memory?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

#### Key Points to Cover:
```
QoS Classes — determined by resource requests/limits:

Guaranteed (highest priority, never evicted unless OOM):
  Requirements:
    → EVERY container must have memory AND cpu requests AND limits
    → requests MUST EQUAL limits (exact same value)

  containers:
  - resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "512Mi"    # same as request
        cpu: "500m"        # same as request
  → System knows exactly what this pod needs — predictable

Burstable (medium priority, evicted when node under pressure):
  Requirements:
    → At least one container has requests OR limits set
    → requests != limits (can burst above request up to limit)

  containers:
  - resources:
      requests:
        memory: "256Mi"   # can use up to 512Mi
      limits:
        memory: "512Mi"
  → Can use more than requested, but might be evicted

BestEffort (lowest priority, first to be evicted):
  Requirements:
    → NO containers have any requests or limits set

  containers:
  - name: app
    image: myapp
    # No resources section at all
  → Gets whatever is leftover — evicted first

Eviction order (node memory pressure):
  1. BestEffort pods evicted first (they have no guarantees)
  2. Burstable pods using most above their request (sorted by excess)
  3. Burstable pods with less excess
  4. Guaranteed pods (only as last resort — OOM kill)

OOM Score:
  Linux kernel assigns oom_score_adj to each process
  Kubernetes sets:
    BestEffort:   oom_score_adj = 1000 (killed first)
    Burstable:    oom_score_adj = 2-999 (proportional to excess)
    Guaranteed:   oom_score_adj = -997 (killed last)

Practical implications:
  Critical services (databases, API servers): use Guaranteed QoS
  → Set requests = limits
  → Protected from eviction

  Stateless workers, jobs: Burstable is fine
  → Set requests at normal usage, limit at burst

  NEVER run without resource requests in production:
  → BestEffort = first evicted = unreliable
```

> 💡 **Interview tip:** The surprising Guaranteed QoS rule: **requests must EQUAL limits**. Many engineers set `requests: 256Mi, limits: 512Mi` thinking they're being conservative — but this makes the pod Burstable (eviction candidate), not Guaranteed. For critical workloads, setting `requests == limits` is worth the slight resource "waste" — it gives the pod eviction protection. The tradeoff: Guaranteed pods can't burst above their limit, so size them at peak usage, not average usage. Use `kubectl get pod <name> -o yaml | grep qosClass` to verify the QoS class assigned.

---

### Q75 — AWS | Scenario-Based | Advanced

> A security audit finds some S3 buckets are **publicly accessible** and some have **no encryption**. How do you audit all S3 buckets across your account, fix them, and prevent this from happening again using automation?

📁 **Reference:** `nawab312/AWS` — S3 security, AWS Config, IAM policies sections

#### Key Points to Cover:
```
Step 1 — Audit all buckets:
  # List all buckets:
  aws s3api list-buckets --query 'Buckets[].Name' --output text

  # Check each bucket for public access:
  for bucket in $(aws s3api list-buckets --query 'Buckets[].Name' --output text); do
    public=$(aws s3api get-public-access-block \
      --bucket "$bucket" \
      --query 'PublicAccessBlockConfiguration' \
      --output text 2>/dev/null || echo "NO_CONFIG")
    
    acl=$(aws s3api get-bucket-acl \
      --bucket "$bucket" \
      --query 'Grants[?Grantee.URI==`http://acs.amazonaws.com/groups/global/AllUsers`]' \
      --output text)
    
    echo "Bucket: $bucket | PublicBlock: $public | PublicACL: $acl"
  done

  # Check encryption:
  aws s3api get-bucket-encryption --bucket my-bucket 2>&1

Step 2 — Fix: Block public access on all buckets:
  # Account-level block (blocks ALL buckets):
  aws s3control put-public-access-block \
    --account-id $(aws sts get-caller-identity --query Account --output text) \
    --public-access-block-configuration \
      BlockPublicAcls=true,IgnorePublicAcls=true,\
      BlockPublicPolicy=true,RestrictPublicBuckets=true

  # Per-bucket fix:
  aws s3api put-public-access-block \
    --bucket my-bucket \
    --public-access-block-configuration \
      BlockPublicAcls=true,IgnorePublicAcls=true,\
      BlockPublicPolicy=true,RestrictPublicBuckets=true

Step 3 — Fix: Enable encryption on all buckets:
  for bucket in $(aws s3api list-buckets --query 'Buckets[].Name' --output text); do
    aws s3api put-bucket-encryption \
      --bucket "$bucket" \
      --server-side-encryption-configuration '{
        "Rules": [{
          "ApplyServerSideEncryptionByDefault": {
            "SSEAlgorithm": "aws:kms",
            "KMSMasterKeyID": "arn:aws:kms:us-east-1:123:key/xxx"
          },
          "BucketKeyEnabled": true
        }]
      }'
  done

Step 4 — Prevent recurrence:

  a) AWS Config Rules (continuous compliance monitoring):
     → s3-bucket-public-read-prohibited (auto-detect public buckets)
     → s3-bucket-server-side-encryption-enabled
     → Config remediation: auto-fix when rule violated

  b) AWS Organizations Service Control Policy (SCP):
     # Block all S3 public access at organization level:
     {
       "Effect": "Deny",
       "Action": "s3:PutBucketPublicAccessBlock",
       "Resource": "*",
       "Condition": {
         "StringEquals": {
           "s3:PublicAccessBlockConfiguration/BlockPublicAcls": "false"
         }
       }
     }

  c) Terraform enforce in IaC:
     resource "aws_s3_bucket_public_access_block" "all" {
       bucket = aws_s3_bucket.main.id
       block_public_acls       = true
       block_public_policy     = true
       ignore_public_acls      = true
       restrict_public_buckets = true
     }

  d) Checkov in CI/CD: fails if S3 bucket missing public access block
```

> 💡 **Interview tip:** The **account-level S3 Block Public Access** setting is the fastest fix — one command blocks public access for every bucket in the account, including any future buckets created. This is the first thing to check in any AWS account setup. For ongoing prevention, the layered approach is: (1) account-level block (catches everything immediately), (2) AWS Config rules (continuous monitoring + alert if someone disables it), (3) SCP (blocks the API call at the org level — even account admins can't re-enable public access). SCPs are the strongest control — they can't be overridden even by account root.

---

### Q76 — Linux / Bash | Conceptual | Advanced

> Explain the Linux **`/proc` filesystem**. Give **5 practical examples** of how a DevOps/SRE engineer would use `/proc` to diagnose production issues, with exact file paths and what they reveal.

📁 **Reference:** `nawab312/DSA` → `Linux` — `/proc` filesystem sections

#### Key Points to Cover:
```
What /proc is:
  → Virtual filesystem — not on disk, generated by kernel at read time
  → Window into kernel and process information
  → Every read returns current live data
  → Zero disk space used (kernel generates content on access)

5 practical examples:

1. Investigate why a process is slow (open files/connections):
   /proc/<PID>/fd/       → all open file descriptors (symlinks to actual files)
   ls -la /proc/1234/fd/ # shows all open files, sockets, pipes
   /proc/<PID>/fdinfo/   → file descriptor details (position, flags)
   
   "Why is nginx serving stale content?"
   ls -la /proc/$(pgrep nginx)/fd | grep access.log
   # Reveals if nginx has stale file descriptor to rotated log

2. Check memory pressure before OOM:
   /proc/meminfo         → detailed memory breakdown
   cat /proc/meminfo | grep -E "MemAvailable|SwapFree|Dirty|Writeback"
   # MemAvailable: what's actually free (not just "free" - includes reclaimable)
   # Dirty: data written to page cache but not yet to disk
   # High Dirty + low MemAvailable = about to OOM

3. Find what's in a process's memory:
   /proc/<PID>/maps      → virtual memory map (what libraries loaded)
   /proc/<PID>/smaps     → detailed memory per mapping with RSS
   cat /proc/1234/smaps | grep -A 10 "heap"
   # Shows exactly how much heap a Java/Python process is using

4. Diagnose network connection issues:
   /proc/net/tcp         → all TCP connections with hex state codes
   /proc/net/tcp6        → IPv6 TCP connections
   /proc/net/sockstat    → socket usage summary
   cat /proc/net/sockstat
   # TCP: inuse 450 orphan 2 tw 120 alloc 452 mem 85
   # High "tw" = many TIME_WAIT connections
   # High "orphan" = connection leak (sockets not properly closed)

5. Tune kernel parameters without reboot:
   /proc/sys/             → live kernel parameters (same as sysctl)
   /proc/sys/vm/swappiness       → swap aggressiveness (0-100)
   /proc/sys/net/ipv4/ip_local_port_range → ephemeral port range
   
   echo "10" > /proc/sys/vm/swappiness   # immediate effect, no restart
   # Use sysctl for permanent changes, /proc/sys for immediate testing
   
   # Kubernetes requirement:
   cat /proc/sys/net/bridge/bridge-nf-call-iptables  # must be 1

Bonus — process environment variables:
   /proc/<PID>/environ   → null-separated env vars of running process
   cat /proc/$(pgrep java)/environ | tr '\0' '\n' | grep DATABASE
   # "What DATABASE_URL is the running app actually using?"
   # Very useful when you suspect env var injection failed
```

> 💡 **Interview tip:** The **`/proc/<PID>/environ`** trick is one of the most practical SRE debugging tools — when a process is misbehaving and you suspect it's using the wrong environment variable, you can read the exact env vars it was started with. Unlike `env` or `printenv` (which show current shell's vars), `/proc/environ` shows the ACTUAL env vars the process received at startup. This catches issues like: "the process was started before the new secret was injected" or "the K8s secret wasn't updated before rollout." Also: `/proc/sys` changes are immediate but not persistent — always test with `/proc/sys` first, then make permanent with `sysctl -w` and `/etc/sysctl.conf`.

---

### Q77 — ArgoCD | Conceptual | Advanced

> Explain the difference between **ArgoCD** and **Flux** as GitOps tools. Also explain what **ApplicationSets** are in ArgoCD and give a real-world example of when to use them over individual Applications.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — ApplicationSets sections

#### Key Points to Cover:
```
ArgoCD vs Flux:

ArgoCD:
  → GUI-first: excellent web UI for visualization and manual operations
  → App of Apps, ApplicationSets for managing many apps
  → Single repository of Application CRDs
  → Push-based sync model (Argo "pulls" from Git but has a sync engine)
  → Sync waves for ordering deployments
  → Better for: teams needing visibility, manual approvals, multi-cluster UI
  → CNCF graduated project

Flux:
  → CLI-first: designed for fully automated GitOps (less UI)
  → Helm Controller, Kustomize Controller as separate controllers
  → Better Helm OCI support (early adopter)
  → More "GitOps pure" — everything is a K8s controller
  → Notification Controller, Image Automation Controller
  → Better for: platform teams wanting fully automated, minimal UI needed
  → CNCF graduated project

Choose ArgoCD when:
  → Need web UI for visibility + manual sync control
  → Teams need to see deployment status visually
  → Multi-cluster with central management
  → App of Apps / ApplicationSets for many services

Choose Flux when:
  → Fully automated, no human in the loop
  → Already using Helm extensively (Helm Controller is excellent)
  → Want pure K8s controller model
  → Smaller team, less need for UI

ApplicationSets:
  Problem: 20 microservices × 3 environments = 60 ArgoCD Application CRDs
  Managing 60 individual Application YAMLs = toil

  Solution: ApplicationSet = template that generates multiple Applications

  # Generator types:
  1. List generator (explicit list):
  spec:
    generators:
    - list:
        elements:
        - service: payment
          namespace: payment-prod
        - service: auth
          namespace: auth-prod
    template:
      metadata:
        name: '{{service}}-prod'
      spec:
        source:
          path: 'apps/{{service}}'
        destination:
          namespace: '{{namespace}}'

  2. Git generator (discovers from Git directory structure):
  spec:
    generators:
    - git:
        repoURL: https://github.com/org/gitops-repo
        revision: HEAD
        directories:
        - path: 'apps/*'   # discovers all directories under apps/
    template:
      metadata:
        name: '{{path.basename}}'  # directory name = app name
      spec:
        source:
          path: '{{path}}'

  3. Cluster generator (deploy to all/some clusters):
  spec:
    generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
    template:
      metadata:
        name: 'myapp-{{name}}'  # name = cluster name
      spec:
        destination:
          server: '{{server}}'  # cluster API URL

  Real-world use: 
  Deploy same monitoring stack (Prometheus, Grafana, Loki) to 10 clusters
  → One ApplicationSet with cluster generator → 10 Applications created auto
  → New cluster added with label → Application created automatically
```

> 💡 **Interview tip:** **ApplicationSet is the most scalable ArgoCD pattern** — the difference between managing 60 YAML files manually vs one template. The **Git directory generator** is particularly powerful: you add a new directory `apps/new-service/` to your Git repo, and ArgoCD automatically creates a new Application for it. Zero manual Application creation. Also mention that **Flux and ArgoCD can coexist** — some teams use Flux for infrastructure (Terraform controllers, cert-manager) and ArgoCD for application deployments. This isn't unusual in mature organizations.

---

### Q78 — Terraform | Conceptual | Advanced

> What are Terraform **`depends_on`**, **`lifecycle`**, and **`provisioner`** blocks? When would you use each? Are there cases where you should **avoid** them?

📁 **Reference:** `nawab312/Terraform` — resource lifecycle, dependencies, provisioners sections

#### Key Points to Cover:
```
depends_on (explicit dependency):
  → Forces Terraform to create/destroy resources in specific order
  → Normally: Terraform infers dependencies from references
  → Use when: dependency isn't expressed through attribute reference

  resource "aws_iam_role_policy" "app_policy" { ... }

  resource "aws_ecs_service" "app" {
    depends_on = [aws_iam_role_policy.app_policy]
    # ECS service needs the policy to exist first
    # But there's no direct attribute reference → Terraform can't infer
  }

  When to avoid:
  ✅ Use when hidden dependencies exist
  ❌ Don't use to fix errors — usually a sign of missing reference
  ❌ Overuse creates fragile ordering that breaks on changes

lifecycle block:
  → Controls resource creation, update, and destruction behavior

  resource "aws_db_instance" "prod" {
    lifecycle {
      create_before_destroy = true
      # Default: destroy old → create new (downtime)
      # With flag: create new → switch → destroy old (zero downtime)
      # Use for: RDS, ALB target groups, EC2 instances

      prevent_destroy = true
      # terraform plan FAILS if this resource would be destroyed
      # Use for: production databases, critical data stores

      ignore_changes = [
        tags["LastModified"],    # ignore auto-set AWS tags
        engine_version,          # ignore RDS auto minor upgrades
        desired_count,           # ignore ECS autoscaling changes
      ]
      # Use when: external systems modify attributes Terraform manages

      replace_triggered_by = [   # TF 1.2+
        aws_launch_template.app.latest_version
        # Force replace when launch template changes
      ]
    }
  }

  When to avoid lifecycle:
  ❌ ignore_changes = all (hides all drift — very dangerous)
  ❌ prevent_destroy in dev (blocks cleanup)

provisioner block:
  → Runs commands on local machine or remote resource
  → local-exec: runs on machine executing Terraform
  → remote-exec: runs via SSH on created resource

  resource "aws_instance" "web" {
    provisioner "local-exec" {
      command = "ansible-playbook -i '${self.public_ip},' site.yml"
    }
    provisioner "remote-exec" {
      connection { type = "ssh"; host = self.public_ip }
      inline = ["sudo apt-get update", "sudo apt-get install -y nginx"]
    }
  }

  When to AVOID (HashiCorp recommendation):
  ❌ Provisioners are last resort — prefer:
     → cloud-init/user_data for EC2 bootstrapping
     → Custom AMIs with Packer (bake config in)
     → AWS Systems Manager for post-deploy config
     → Ansible run separately after Terraform

  Problems with provisioners:
  → Run only at creation (not on config changes)
  → Failures leave resource in unknown state (tainted)
  → remote-exec requires SSH (security concern)
  → Not idempotent (fails on second run)
```

> 💡 **Interview tip:** The strong interview answer on provisioners: **"Provisioners are a code smell in Terraform — their presence usually means you need Packer, user_data, or a separate configuration management tool."** HashiCorp says this explicitly in their docs. The only valid use cases: calling an external API that has no Terraform provider, running a one-time migration script. For everything else: user_data scripts for EC2 bootstrapping, AMIs built with Packer, AWS SSM for ongoing config management.

---

### Q79 — Kubernetes | Scenario-Based | Advanced

> Your batch processing system on Kubernetes processes millions of records nightly. Currently uses a Deployment but you're experiencing: failed jobs leaving no trace, no retry on failure, can't track completion. **How would you redesign using the right workload type?**

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md` — Jobs and CronJobs sections

#### Key Points to Cover:
```
Problem with Deployment for batch:
  → Deployment designed for long-running services (never "done")
  → Failed pod → restarted forever (no failure tracking)
  → No concept of "job completed successfully"
  → No parallelism control
  → No automatic cleanup

Solution: Kubernetes Job

apiVersion: batch/v1
kind: Job
metadata:
  name: nightly-processor-2024-01-15
spec:
  completions: 1000         # total work units to complete
  parallelism: 50           # run 50 pods simultaneously
  completionMode: Indexed   # each pod gets unique index (0-999)
  backoffLimit: 3           # retry failed pods up to 3 times
  activeDeadlineSeconds: 7200  # kill if running > 2 hours

  ttlSecondsAfterFinished: 86400  # auto-delete job after 24 hours

  template:
    spec:
      restartPolicy: OnFailure  # retry on fail (not Never for batch)
      containers:
      - name: processor
        image: myapp-processor:v1.2.3
        env:
        - name: JOB_COMPLETION_INDEX  # index from completionMode
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
        resources:
          requests: { cpu: "500m", memory: "512Mi" }
          limits:   { cpu: "1000m", memory: "1Gi" }

Schedule with CronJob:
  apiVersion: batch/v1
  kind: CronJob
  metadata:
    name: nightly-processor
  spec:
    schedule: "0 2 * * *"        # 2am every night
    concurrencyPolicy: Forbid    # don't run if previous still running
    successfulJobsHistoryLimit: 3
    failedJobsHistoryLimit: 5
    startingDeadlineSeconds: 3600  # skip if can't start within 1 hour
    jobTemplate:
      spec:
        # ... same as Job spec above ...

Monitor job progress:
  kubectl get jobs
  kubectl describe job nightly-processor-2024-01-15
  # Shows: Succeeded: 750/1000, Failed: 3, Active: 47

Handle large data with indexed completion:
  # Each pod processes its slice:
  # Pod 0 processes records 0-999
  # Pod 1 processes records 1000-1999
  # etc.
  BATCH_START=$((JOB_COMPLETION_INDEX * 1000))
  BATCH_END=$(((JOB_COMPLETION_INDEX + 1) * 1000 - 1))
  process_records $BATCH_START $BATCH_END

Failure handling:
  backoffLimit: 3           # try 3 times per pod
  restartPolicy: OnFailure  # restart same pod on failure
  # OR:
  restartPolicy: Never      # create new pod on failure (for debugging)
  # With Never: failed pods stay around for log inspection
```

> 💡 **Interview tip:** The **Indexed Job** pattern (completionMode: Indexed) is the modern Kubernetes approach for parallel batch processing — each pod knows exactly which chunk of work it should process via `JOB_COMPLETION_INDEX`. This eliminates the need for a work queue (SQS, Redis) for simple parallel jobs. The `backoffLimit` vs `activeDeadlineSeconds` distinction is important: `backoffLimit` controls individual pod retry count, `activeDeadlineSeconds` is the absolute time budget for the entire job (kills everything after N seconds regardless of progress). For nightly batch jobs, always set `activeDeadlineSeconds` — otherwise a stuck job blocks the next night's run.

---

### Q80 — AWS | Troubleshooting | Advanced

> Your **AWS Lambda function** times out in production but works fine in dev. It calls an RDS database and an external API. Timeout is set to 30 seconds but sometimes runs the full 30 seconds. Walk through your investigation.

📁 **Reference:** `nawab312/AWS` — Lambda, RDS, VPC, CloudWatch Logs Insights sections

#### Key Points to Cover:
```
Step 1 — Lambda in VPC vs not in VPC:
  # First question: is the Lambda in a VPC?
  aws lambda get-function-configuration --function-name my-fn \
    --query 'VpcConfig'
  
  Lambda NOT in VPC:
    → Cannot access RDS in private subnet (connection refused immediately)
    → Dev might not have VPC = works, Prod has VPC = connectivity issue
  
  Lambda IN VPC:
    → Needs ENI (Elastic Network Interface) — cold start adds 1-3 seconds
    → Must be in subnets with route to RDS security group
    → Outbound to external API needs: NAT Gateway (private subnet) or public subnet

Step 2 — Check cold start vs warm execution:
  # In CloudWatch Logs, look for "Init Duration" in REPORT line:
  # REPORT Duration: 28500ms  Billed: 28500ms  Init Duration: 2500ms
  # Init Duration present = cold start (Lambda container just created)
  # Cold start in VPC with RDS: ENI creation adds 1-3 seconds extra

Step 3 — Identify where time is spent:
  # Add timing logs to Lambda:
  import time, logging
  logger = logging.getLogger()
  
  def handler(event, context):
      t0 = time.time()
      # DB connection:
      conn = get_db_connection()
      logger.info(f"DB connect time: {time.time()-t0:.3f}s")
      
      t1 = time.time()
      result = conn.execute(query)
      logger.info(f"DB query time: {time.time()-t1:.3f}s")
      
      t2 = time.time()
      api_result = call_external_api()
      logger.info(f"API call time: {time.time()-t2:.3f}s")
  
  # CloudWatch Logs Insights query:
  fields @timestamp, @message
  | filter @message like "time:"
  | parse @message "* time: *s" as operation, duration
  | stats avg(duration) by operation

Step 4 — Common Lambda timeout root causes:
  a) DB connection NOT reused between invocations:
     # Wrong: create new connection in handler (every cold start)
     def handler(event, context):
         conn = psycopg2.connect(...)  # NEW connection every time
     
     # Right: connection outside handler (reused on warm start)
     conn = psycopg2.connect(DATABASE_URL)  # at module level
     def handler(event, context):
         cursor = conn.cursor()  # reuses connection

  b) RDS max connections exhausted:
     # Lambda scales to 1000 concurrent = 1000 DB connections
     # RDS db.t3.micro max_connections = 66
     # Fix: RDS Proxy (connection pooling for Lambda)

  c) External API slow or down:
     # Add timeout to external API calls:
     requests.get(url, timeout=5)  # fail fast, not wait 30 seconds

  d) Lambda function running in wrong subnet:
     # Private subnet without NAT → external API calls hang
     # Check: route table has 0.0.0.0/0 → NAT Gateway

  e) DNS resolution slow in VPC:
     # Use endpoint IPs, not DNS for internal calls
     # Or configure Route53 Resolver

Step 5 — Fixes:
  → RDS Proxy: reduces connections, handles pool
  → Connection reuse: move DB init to module level
  → Timeouts on all external calls
  → Lambda Powertools: structured logging + metrics
  → Increase memory (also increases CPU proportionally)
```

> 💡 **Interview tip:** The **Lambda + RDS connection exhaustion** problem is the most common Lambda production issue. Lambda can scale to thousands of concurrent executions, each trying to open a database connection. RDS only allows a limited number of connections (based on instance size). Solution: **RDS Proxy** — it pools connections between Lambda and RDS, so 1000 Lambda invocations might only need 50 actual RDS connections. This is now the standard architecture for Lambda + RDS. Also: connection reuse at module level is a critical Lambda optimization — cold starts are unavoidable but re-creating DB connections on every warm invocation is wasteful and slow.

---

### Q81 — ELK Stack | Conceptual | Advanced

> Explain how **Elasticsearch indexing** works internally. What are **shards** and **replicas** and how do they affect performance and availability? If you have 5 primary shards and 1 replica on a 3-node cluster, what happens when one node goes down?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack`

#### Key Points to Cover:
```
How Elasticsearch indexing works:
  1. Document received by any node (coordinator)
  2. Coordinator determines which primary shard owns this document
     → Shard = hash(document_id) % number_of_primary_shards
  3. Request routed to node holding that primary shard
  4. Document written to shard (Lucene inverted index)
  5. Changes written to translog (durability before flush)
  6. Replicated to replica shards (async by default)
  7. Acknowledged to client when primary + configured replicas confirm

Shards:
  → Elasticsearch index = collection of shards
  → Shard = self-contained Lucene index
  → Primary shard: accepts writes
  → Replica shard: copy of primary (read load + HA)
  → Shards distributed across nodes automatically

  Setting shards at index creation (can't change without reindex):
  PUT /my-index
  {
    "settings": {
      "number_of_shards": 5,
      "number_of_replicas": 1
    }
  }

Performance impact:
  More shards:
  → Parallelism: searches split across shards (faster)
  → BUT: more overhead per shard (each shard = JVM overhead)
  → Rule of thumb: shard size 10-50GB, max 200 shards per node

  More replicas:
  → Better read throughput (read from any replica)
  → Higher storage cost (each replica = full data copy)
  → Slower writes (must replicate to all replicas)

Availability impact:
  → primary_shards: 5, replicas: 1 → 10 total shards across 3 nodes

  Normal state (3 nodes):
    Node 1: P0, P1, R2, R3
    Node 2: P2, P3, R0, R4
    Node 3: P4, R1, R2, R3
  (Elasticsearch distributes to balance)

  One node goes down (e.g., Node 1 fails):
    → P0 and P1 are GONE (primary shards on Node 1)
    → Elasticsearch PROMOTES R0 and R1 to primary (on Node 2 and 3)
    → Cluster temporarily YELLOW (primaries ok, replicas now missing)
    → Cluster status: YELLOW (not RED — all primaries are assigned)
    → If you had 0 replicas → R0 and R1 don't exist → RED

  Recovery when Node 1 comes back:
    → Shards re-assigned from current primaries
    → Cluster returns to GREEN

Two nodes go down simultaneously:
  → Some primary shards may be lost with no replica
  → Cluster becomes RED (unassigned primaries)
  → Prevention: replicas >= 2 for critical data
```

> 💡 **Interview tip:** The interview question "what happens when a node goes down?" tests whether you understand the **replica promotion** mechanism. The answer: with replicas, the cluster goes YELLOW (degraded but functional), not RED. YELLOW means all primaries are assigned (data accessible) but some replicas are missing. RED means at least one PRIMARY is unassigned (data for that shard is inaccessible). The key operational rule: **never run production Elasticsearch with `number_of_replicas: 0`** — one node failure makes cluster RED and that data is gone until the node recovers.

---

### Q82 — GitHub Actions | Troubleshooting | Advanced

> Your GitHub Actions workflow passes locally (act) but fails in GitHub with `EACCES: permission denied, open '/github/workspace/output.json'`. Another workflow is passing but taking 45 minutes instead of 5 minutes. Diagnose and fix both issues.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — runner permissions, caching sections

#### Key Points to Cover:
```
Issue 1 — Permission denied: /github/workspace/output.json

Root cause investigation:
  # GitHub Actions runners run as non-root user
  # Act locally might run as root (different behavior)

  # Check runner user:
  - run: whoami && id
  # Shows: runner, uid=1001

  # Check workspace permissions:
  - run: ls -la /github/workspace/

Common causes:
  a) Previous step creates file as root (via sudo or privileged container):
     # File owned by root, runner can't write
     # Fix: don't run steps as root, or fix permissions:
     - run: sudo chown -R $USER:$USER /github/workspace/

  b) Docker container in job writes file as root:
     # Inside Docker container, process runs as root by default
     # Files created = owned by root
     # Fix: add user to Dockerfile:
     USER 1001  # match GitHub Actions runner UID

  c) Mount permissions in act:
     # act uses Docker mounts — different UID mapping than GitHub
     # Fix: test with --container-daemon-socket to match behavior

  d) Output file path doesn't exist:
     # Output directory not created before write
     - run: mkdir -p $(dirname output.json) && command > output.json

  Fix:
  - name: Fix permissions
    run: |
      mkdir -p /github/workspace
      # OR use GITHUB_WORKSPACE env var:
      echo "data" > $GITHUB_WORKSPACE/output.json

Issue 2 — Workflow taking 45 minutes instead of 5:

Step 1 — Identify slow step:
  # Look at step timing in GitHub Actions UI (each step shows duration)
  # OR add timing:
  - run: |
      time npm ci          # how long is dependency install?
      time npm run build   # how long is build?

Common causes of slow workflows:

  a) No dependency caching:
     # npm ci downloads all packages every run (5-10 min each run)
     # Fix: add caching:
     - uses: actions/cache@v4
       with:
         path: ~/.npm
         key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
         restore-keys: ${{ runner.os }}-npm-
     - run: npm ci

  b) No Docker layer caching:
     # Docker builds rebuild all layers every run
     # Fix: BuildKit cache:
     - uses: docker/setup-buildx-action@v3
     - uses: docker/build-push-action@v5
       with:
         cache-from: type=gha
         cache-to: type=gha,mode=max

  c) Sequential jobs that could be parallel:
     # Unit tests, lint, security scan all run one after another
     # Fix: run as parallel jobs (default in GitHub Actions)

  d) Test suite running serially:
     # Fix: --runInBand removed, use Jest --maxWorkers=4
     # Or: split tests across matrix

  e) Downloading large artifacts between jobs:
     # Fix: build Docker image once, push to registry
     # Next job pulls from registry (cached layers)

  f) Wrong runner type:
     # ubuntu-latest may be slower than specific version
     # Fix: use ubuntu-22.04 for consistency
```

> 💡 **Interview tip:** The permission issue is almost always **root vs non-root user mismatch**. GitHub Actions runners use UID 1001 (non-root). If any step in your workflow runs as root (via `sudo`, privileged Docker container, or certain actions), files it creates are owned by root and subsequent non-root steps can't write to them. The 45-minute workflow is almost always **missing npm/pip/Docker layer caching** — adding `actions/cache` with the right key transforms a 10-minute dependency install into a 30-second cache restore. The cache key using `hashFiles('package-lock.json')` is critical — it invalidates the cache when dependencies change, ensuring you get fresh packages when needed.

---

### Q83 — Kubernetes | Conceptual | Advanced

> Explain **PodDisruptionBudget (PDB)**. What problem does it solve? Write a PDB for a 5-replica deployment requiring at least 3 running. What happens during a **node drain** if PDB cannot be satisfied?

📁 **Reference:** `nawab312/Kubernetes` → `09_CLUSTER_OPERATIONS.md`

#### Key Points to Cover:
```
What PDB solves:
  Without PDB: cluster operations (node drain, cluster upgrade) can
  evict ALL pods of a deployment simultaneously
  → service downtime during planned maintenance

  With PDB: Kubernetes guarantees minimum availability during
  voluntary disruptions (not crashes — voluntary evictions only)

PDB for 5-replica deployment:
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata:
    name: payment-service-pdb
    namespace: production
  spec:
    minAvailable: 3        # always keep at least 3 running
    selector:
      matchLabels:
        app: payment-service

  # OR equivalently:
  spec:
    maxUnavailable: 2      # at most 2 can be unavailable simultaneously
    # minAvailable: 3  ≡  maxUnavailable: 2 for 5 replicas

  # Use percentage:
  spec:
    minAvailable: "60%"    # 60% of replicas must be available
    # For 5 replicas: 60% = 3 pods minimum

Voluntary vs Involuntary disruptions:
  PDB protects against VOLUNTARY:
  ✅ kubectl drain (node maintenance)
  ✅ Rolling update (new deployment)
  ✅ Cluster autoscaler (node removal)
  ✅ kubectl delete pod

  PDB does NOT protect against INVOLUNTARY:
  ❌ Node crash (hardware failure)
  ❌ OOM kill
  ❌ Kernel panic

Node drain with PDB:
  kubectl drain node-1 --ignore-daemonsets

  Scenario: 5 pods, PDB minAvailable=3, pods on 3 nodes
  Node-1 has 3 pods, Node-2 has 1, Node-3 has 1

  Step 1: Drain starts — tries to evict pods from Node-1
  Step 2: Evict pod-1 from Node-1 → 4 pods running (>= 3 ✅) → allowed
  Step 3: Evict pod-2 from Node-1 → 3 pods running (= 3 ✅) → allowed
  Step 4: Evict pod-3 from Node-1 → 2 pods running (< 3 ❌) → BLOCKED
  → Drain WAITS for pod-3 to be replaced before proceeding
  → Kubernetes schedules pod-3 on another node
  → Once 4th pod running → drain evicts pod-3 from Node-1
  → Drain completes with zero service disruption

When PDB cannot be satisfied (stuck drain):
  Scenario: Only 3 pods exist, PDB minAvailable=3
  → No pods can be evicted (evicting any = drops below minimum)
  → kubectl drain hangs indefinitely
  
  Solutions:
  # Option 1: Override PDB (dangerous — may cause downtime):
  kubectl drain node-1 --ignore-daemonsets --disable-eviction
  # Option 2: Temporarily change PDB:
  kubectl patch pdb payment-pdb --type='json' \
    -p='[{"op":"replace","path":"/spec/minAvailable","value":2}]'
  # Option 3: Scale up first:
  kubectl scale deployment payment-service --replicas=6
  # Then drain → then scale back to 5
```

> 💡 **Interview tip:** PDB is the **safety net for cluster operations** — without it, a `kubectl drain` during node maintenance can take down all replicas if they happen to be on one node. Always create PDBs for stateless services with `minAvailable: n-1` or `maxUnavailable: 1`. The common mistake: setting `minAvailable` equal to the total replicas (`minAvailable: 5` for 5 replicas) — this makes draining impossible (can never evict any pod). The correct pattern: `minAvailable: ⌈replicas * 0.6⌉` — allow 40% to be disrupted at once.

---

### Q84 — Prometheus + Grafana | Scenario-Based | Advanced

> Implement **SLO monitoring** for a payment service: 99.95% availability over 30 days, 99% of requests under 200ms. Alert when error budget is 50% consumed and when burn rate is too fast. Write **Prometheus alerting rules** with **multi-window, multi-burn-rate** approach.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`, `Grafana`

#### Key Points to Cover:
```yaml
# Recording rules (pre-compute for efficiency):
groups:
- name: slo_recording_rules
  rules:
  # Error rate over different windows:
  - record: job:http_errors:rate5m
    expr: sum(rate(http_requests_total{status=~"5..",job="payment"}[5m]))
          / sum(rate(http_requests_total{job="payment"}[5m]))
  
  - record: job:http_errors:rate1h
    expr: sum(rate(http_requests_total{status=~"5..",job="payment"}[1h]))
          / sum(rate(http_requests_total{job="payment"}[1h]))

  - record: job:http_errors:rate6h
    expr: sum(rate(http_requests_total{status=~"5..",job="payment"}[6h]))
          / sum(rate(http_requests_total{job="payment"}[6h]))

  - record: job:http_errors:rate3d
    expr: sum(rate(http_requests_total{status=~"5..",job="payment"}[3d]))
          / sum(rate(http_requests_total{job="payment"}[3d]))

# Multi-window, multi-burn-rate alerts:
- name: slo_alerts
  rules:
  # CRITICAL: Fast burn — 99.95% SLO
  # 14.4x burn rate = uses 1 hour of budget in 5 minutes
  # If this continues, 3-day budget exhausted
  - alert: PaymentSLO_FastBurn_Critical
    expr: |
      (job:http_errors:rate5m > (14.4 * 0.0005))
      and
      (job:http_errors:rate1h > (14.4 * 0.0005))
    for: 2m
    labels:
      severity: critical
      slo: payment_availability
    annotations:
      summary: "Payment SLO fast burn - CRITICAL"
      description: |
        Error rate {{ $value | humanizePercentage }} - burning budget
        14.4x faster than allowed. Action required immediately.

  # HIGH: Slow burn — budget will be exhausted in 3 days
  - alert: PaymentSLO_SlowBurn_High
    expr: |
      (job:http_errors:rate1h > (6 * 0.0005))
      and
      (job:http_errors:rate6h > (6 * 0.0005))
    for: 15m
    labels:
      severity: high
    annotations:
      summary: "Payment SLO slow burn - HIGH"
      description: "Error rate elevated, budget will exhaust in ~5 days"

  # WARNING: 50% of monthly error budget consumed
  - alert: PaymentSLO_BudgetHalfConsumed
    expr: |
      1 - (
        sum(rate(http_requests_total{status!~"5..",job="payment"}[30d]))
        / sum(rate(http_requests_total{job="payment"}[30d]))
      ) > 0.5 * 0.0005
    labels:
      severity: warning
    annotations:
      summary: "50% of monthly error budget consumed"

  # Latency SLO: 99% under 200ms
  - alert: PaymentLatencySLO_Burn
    expr: |
      histogram_quantile(0.99,
        sum by(le)(rate(http_request_duration_seconds_bucket{job="payment"}[5m]))
      ) > 0.200
    for: 5m
    labels:
      severity: high
    annotations:
      summary: "Payment p99 latency SLO breach"
      description: "p99 latency {{ $value }}s > 200ms SLO target"
```

```
Multi-window, multi-burn-rate explained:
  Why two windows (5m AND 1h)?
  → Single short window: too noisy (triggers on brief spikes)
  → Single long window: too slow to alert (major outage runs 30 min)
  → Two windows: short confirms it's happening NOW, long confirms it's sustained

  Burn rate calculation for 99.95% SLO:
  Error budget = 1 - 0.9995 = 0.0005 (0.05%)
  14.4x burn rate means: consuming budget 14.4x faster than allowed
  At this rate, 30-day budget exhausted in 30/14.4 = ~2 days
  → CRITICAL: wake someone up NOW

  6x burn rate: 30/6 = 5 days to exhaust
  → HIGH: fix within hours
```

> 💡 **Interview tip:** The **multi-window multi-burn-rate (MWMBR) approach** is from the Google SRE Workbook and is the production-standard SLO alerting pattern. The key insight: a single error rate threshold generates too many false positives (noisy) or misses real problems (too conservative). By requiring BOTH a short window (5m) AND a longer window (1h) to exceed the threshold, you get: fast detection (5m catches it immediately) + confirmation it's sustained (1h ensures it's not a 10-second blip). The burn rate multiplier is calculated: `(1 - SLO) * burn_rate_multiplier` gives you the error rate threshold. For 99.95% + 14.4x = `0.0005 * 14.4 = 0.0072` = 0.72% error rate.

---

### Q85 — AWS | Conceptual | Advanced

> Explain **AWS IAM roles, policies, and permission boundaries**. What is the difference between **identity-based** and **resource-based policies**? How does **AWS STS cross-account access** work using IAM roles?

📁 **Reference:** `nawab312/AWS` — IAM, STS, cross-account access sections

#### Key Points to Cover:
```
IAM Roles:
  → Identity that can be ASSUMED (not logged into)
  → Used by: EC2, Lambda, ECS tasks, users from other accounts
  → Provides: temporary credentials (15 min to 12 hours)
  → Trust policy: WHO can assume this role
  → Permission policy: WHAT the role can do

Identity-based policies (attached to user/role/group):
  → Define what actions the IDENTITY can perform
  → Attached to: IAM users, groups, roles
  → Effect: Allow or Deny

  {
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-bucket/*"
  }

Resource-based policies (attached to resource):
  → Define who CAN ACCESS the resource
  → Attached to: S3 buckets, SQS queues, SNS topics, KMS keys, Lambda
  → Must specify Principal (who is allowed)

  # S3 bucket policy (resource-based):
  {
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::123456789012:role/MyRole"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }

Key difference:
  Identity policy: "I (the user/role) can do X"
  Resource policy: "This resource allows identity Y to do X"
  
  For cross-account: need BOTH (identity policy in source account
  + resource policy/trust policy in target account)

Permission Boundaries:
  → Sets MAXIMUM permissions a role can ever have
  → Even if policies grant more, boundary limits the effective permissions
  → Use case: allow developers to create roles but prevent privilege escalation

  # Boundary allows only S3 and EC2:
  resource "aws_iam_role" "dev_role" {
    permissions_boundary = "arn:aws:iam::123:policy/DeveloperBoundary"
    # Even if dev_role has AdministratorAccess policy attached,
    # actual permissions limited to what DeveloperBoundary allows
  }

Cross-Account Access with STS:
  Account A (Dev): wants to access Account B (Prod) S3
  
  Step 1: In Account B — create role with trust policy:
  {
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::DEV_ACCOUNT_ID:root"},
    "Action": "sts:AssumeRole"
  }

  Step 2: In Account A — user/role calls AssumeRole:
  aws sts assume-role \
    --role-arn "arn:aws:iam::PROD_ACCOUNT_ID:role/CrossAccountRole" \
    --role-session-name "dev-accessing-prod"
  # Returns: AccessKeyId, SecretAccessKey, SessionToken (12-hour max)

  Step 3: Use temporary credentials:
  export AWS_ACCESS_KEY_ID=ASIA...
  export AWS_SECRET_ACCESS_KEY=...
  export AWS_SESSION_TOKEN=...
  aws s3 ls s3://prod-bucket  # works with prod credentials

  In code (SDK handles automatically):
  sts = boto3.client('sts')
  creds = sts.assume_role(
      RoleArn='arn:aws:iam::PROD:role/CrossAccountRole',
      RoleSessionName='dev-session'
  )['Credentials']
  
  s3 = boto3.client('s3',
      aws_access_key_id=creds['AccessKeyId'],
      aws_secret_access_key=creds['SecretAccessKey'],
      aws_session_token=creds['SessionToken']
  )
```

> 💡 **Interview tip:** The **effective permissions** question: if an IAM user has `s3:*` in identity policy but the S3 bucket policy denies `s3:DeleteObject`, what happens? **The Deny wins** — IAM evaluation logic: explicit Deny always overrides Allow. The evaluation order: Organization SCP → Resource policy → Identity policy → Permission boundary → Session policy. Any explicit Deny at any level = denied. For cross-account: you need both the identity in Account A to have permission to call `sts:AssumeRole` AND the role in Account B must trust Account A. Missing either = access denied.

---

### Q86 — Linux / Bash | Troubleshooting | Advanced

> A production server is experiencing **random kernel OOM kills** even though `free -h` shows available memory. Explain what the **OOM killer** is, how it decides which process to kill, and how to **tune it** and **prevent critical processes from being killed**.

📁 **Reference:** `nawab312/DSA` → `Linux` — memory management, OOM killer sections

#### Key Points to Cover:
```
What OOM killer is:
  → Linux kernel mechanism that kills processes when memory is exhausted
  → Triggered: system runs out of physical + swap memory
  → Goal: free enough memory to continue operating
  → Alternative to panic: killing one process > crashing whole system

Why "free -h shows available memory" but OOM still fires:
  1. Memory fragmentation:
     → Plenty of free pages but not contiguous (huge pages need contiguous)
     → OOM can fire even with ~5% free memory

  2. Memory cgroups (container limits):
     → Container has its own memory limit (512Mi)
     → OOM fires when CONTAINER limit hit, not system limit
     → System might have 8GB free, container OOM at 512Mi

  3. Memory overcommit:
     → Linux default: allows allocating more memory than available
     → /proc/sys/vm/overcommit_memory = 0 (default heuristic overcommit)
     → Process allocates, then actually uses (demand paging)
     → If too many processes try to use allocated memory = OOM

How OOM killer chooses victim (oom_score):
  oom_score = 0-1000 (higher = more likely to be killed)
  
  Based on:
  → Memory usage % of total system memory
  → Process age (newer processes scored higher)
  → Child process memory included in parent's score

  # Check OOM score of process:
  cat /proc/<PID>/oom_score
  
  # OOM score adjustment (oom_score_adj):
  cat /proc/<PID>/oom_score_adj
  # Range: -1000 to 1000
  # -1000 = never kill (OOM killer ignores this process)
  # +1000 = kill first

Protect critical processes:
  # Method 1: oom_score_adj (temporary, for this process run):
  echo -1000 > /proc/$(pgrep postgres)/oom_score_adj
  # postgres will NEVER be killed by OOM

  # Method 2: oom_score_adj persistent (set before process starts):
  # In systemd service:
  [Service]
  OOMScoreAdjust=-900    # -1000 = never kill, use -900 for critical

  # Java app service file:
  [Service]
  ExecStart=/usr/bin/java -jar app.jar
  OOMScoreAdjust=-900

  # Method 3: In Kubernetes (set for pod):
  spec:
    containers:
    - name: critical-app
      resources:
        requests:
          memory: "512Mi"
        limits:
          memory: "512Mi"   # Guaranteed QoS = oom_score_adj=-997

Diagnose OOM events:
  # Check kernel logs for OOM events:
  dmesg | grep -i "oom\|killed process"
  journalctl -k | grep -i "oom\|out of memory"
  
  # dmesg output shows:
  # [123456.789] Out of memory: Kill process 1234 (java) score 842
  # Killed process 1234 (java) total-vm:2048000kB, anon-rss:1024000kB

Prevent OOM:
  1. Set accurate memory limits in K8s (avoid BestEffort QoS)
  2. Monitor memory trends (alert at 80% before OOM)
  3. Tune vm.overcommit_memory=2 (don't overcommit):
     sysctl -w vm.overcommit_memory=2
     sysctl -w vm.overcommit_ratio=80  # only commit 80% of RAM
  4. Add swap as emergency buffer (though K8s wants swap=off)
  5. Memory profiling to fix actual leaks
```

> 💡 **Interview tip:** Setting `oom_score_adj = -1000` for critical processes (databases, API servers) is a **production best practice** but it has a cost: if those processes DO have a memory leak, the OOM killer will kill everything else first — potentially cascading failures. The safer approach: set critical processes to `-900` (very unlikely to be killed) but not `-1000` (never). For Kubernetes: **Guaranteed QoS automatically sets oom_score_adj = -997** — this is why setting `requests == limits` for critical pods is so important. It's not just scheduling — it's also OOM protection.

---

### Q87 — Terraform | Scenario-Based | Advanced

> Deploy infrastructure across **5 AWS regions** using Terraform. The same set of resources (VPC, ECS, RDS) needs to be created in each region with region-specific configurations. How would you structure the code **without massive duplication**?

📁 **Reference:** `nawab312/Terraform` — modules, providers, multi-region deployment sections

#### Key Points to Cover:
```
Approach 1 — Provider aliases (for small number of regions):
  # Configure provider for each region:
  provider "aws" { alias = "us_east_1"; region = "us-east-1" }
  provider "aws" { alias = "eu_west_1"; region = "eu-west-1" }
  provider "aws" { alias = "ap_south_1"; region = "ap-south-1" }

  # Call same module for each region:
  module "us_east_1" {
    source    = "./modules/regional-stack"
    providers = { aws = aws.us_east_1 }
    region    = "us-east-1"
    vpc_cidr  = "10.0.0.0/16"
    db_instance_class = "db.r5.xlarge"  # prod size
  }

  module "eu_west_1" {
    source    = "./modules/regional-stack"
    providers = { aws = aws.eu_west_1 }
    region    = "eu-west-1"
    vpc_cidr  = "10.1.0.0/16"
    db_instance_class = "db.r5.large"   # different size for EU
  }

Approach 2 — Terragrunt (for 5+ regions, DRY config):
  # Structure:
  regions/
    us-east-1/
      vpc/terragrunt.hcl      ← just variables, no provider config
      ecs/terragrunt.hcl
      rds/terragrunt.hcl
    eu-west-1/
      vpc/terragrunt.hcl
    ap-southeast-1/
      vpc/terragrunt.hcl
  terragrunt.hcl              ← root: defines backend + provider for all

  # Root terragrunt.hcl:
  locals {
    region = basename(get_terragrunt_dir())  # reads region from dirname
  }
  generate "provider" {
    path = "provider.tf"
    contents = <<EOF
      provider "aws" {
        region = "${local.region}"
      }
    EOF
  }

  # regions/us-east-1/vpc/terragrunt.hcl:
  terraform {
    source = "../../../modules/vpc"
  }
  inputs = {
    cidr_block = "10.0.0.0/16"
    region     = "us-east-1"
  }

  # Deploy all regions at once:
  terragrunt run-all apply   # deploys all regions in parallel

Approach 3 — for_each with alias map (advanced, all in one file):
  locals {
    regions = {
      "us-east-1" = { cidr = "10.0.0.0/16", azs = 3 }
      "eu-west-1" = { cidr = "10.1.0.0/16", azs = 2 }
      "ap-south-1" = { cidr = "10.2.0.0/16", azs = 2 }
    }
  }

  # NOTE: provider aliases can't be dynamic (limitation)
  # Use separate .tf files per region for provider config
  # Use modules + for_each for resources within each region

Module structure (regional-stack):
  modules/regional-stack/
    main.tf     ← VPC + ECS + RDS resources
    variables.tf ← region, vpc_cidr, instance_sizes
    outputs.tf  ← vpc_id, cluster_arn, db_endpoint

Best practices:
  → Use separate state file per region:
    backend config key: "regions/us-east-1/terraform.tfstate"
  → Separate provider credentials per region (same account, different endpoints)
  → Deploy order: VPC → ECS → RDS (use depends_on or sequential applies)
  → Terragrunt run-all: parallel deployment (5 regions simultaneously)
```

> 💡 **Interview tip:** The **Terragrunt approach** is the production answer for 5+ regions. Without Terragrunt, you have provider alias blocks for every region and module calls that look nearly identical — significant repetition. Terragrunt's `run-all apply` also enables **parallel multi-region deployments**, making the process much faster. The key rule: **separate state file per region** — combining all regions in one state file means one region's failure blocks all others, and the state file becomes huge (slow operations). One state per region = independent, parallelizable, isolated blast radius.

---

### Q88 — Kubernetes | Conceptual | Advanced

> Explain **init containers** — how are they different from regular and sidecar containers? Give **3 real-world use cases** and write a Pod spec using an init container to wait for a database to be ready before starting the main app.

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`

#### Key Points to Cover:
```
Init containers vs regular vs sidecar:

Regular containers:
  → Start simultaneously (parallel) after init containers finish
  → Run for the lifetime of the pod
  → Any failure = pod restart (based on restartPolicy)

Init containers:
  → Run BEFORE app containers start
  → Run SEQUENTIALLY (one at a time, in order)
  → Must complete SUCCESSFULLY before next init (or app) starts
  → Exit code 0 = success, move to next
  → Exit non-zero = retry (pod stays in Init state)
  → Cannot run as daemons — MUST exit

Sidecar containers (K8s 1.29+):
  → Run alongside app for pod lifetime (helper/proxy)
  → New: sidecar type starts before init containers (?!)
  → Actually: now there's a formal sidecar concept in K8s
  → Traditional sidecars: just regular containers (no formal distinction pre-1.29)
  → Examples: log collector, service mesh proxy, metrics exporter

3 real-world use cases:

1. Wait for dependency (DB, API) to be ready:
   → "Don't start app until database is up"
   → Init container: until nc -z postgres-svc 5432; do sleep 1; done

2. Database migration before app starts:
   → "Run schema migrations before new app version starts"
   → Init container: flyway migrate (must complete successfully)
   → Guarantees: app never runs with wrong schema version

3. Clone config/secrets from Git or external source:
   → "Pull config files before app starts"
   → Init container: git clone config-repo /config
   → Main app reads /config (shared emptyDir volume)

Pod spec with init container:

apiVersion: v1
kind: Pod
metadata:
  name: payment-service
spec:
  initContainers:
  - name: wait-for-postgres
    image: busybox:1.35
    command:
    - sh
    - -c
    - |
      echo "Waiting for postgres..."
      until nc -z -w 3 postgres-svc.production.svc.cluster.local 5432; do
        echo "postgres not ready, sleeping 2s"
        sleep 2
      done
      echo "Postgres is ready!"
    resources:
      requests: { cpu: "10m", memory: "16Mi" }
      limits:   { cpu: "50m", memory: "32Mi" }

  - name: run-migrations
    image: myapp:v1.2.3
    command: ["python", "manage.py", "migrate"]
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: database-url

  containers:
  - name: payment-service
    image: myapp:v1.2.3
    ports:
    - containerPort: 8080
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: database-url
```

> 💡 **Interview tip:** The **database migration use case** is the most important init container pattern in production. Running migrations in init containers guarantees: (1) migration completes BEFORE any app pod starts receiving traffic, (2) if migration fails, the pod never starts (preventing app from running against wrong schema), (3) in a rolling deployment with 5 replicas, migration runs ONCE (on the first pod's init container), subsequent pods start faster because migration is already done. Without init containers, you'd need complex application-level "run migrations on startup" logic with distributed locking.

---

### Q89 — AWS + Kubernetes | System Design | Advanced

> Design a **complete CI/CD and infrastructure architecture** for a company with 20 microservices on EKS, needing: blue/green deployments, multiple AWS accounts (dev/staging/prod), secrets management, and audit logging of all deployments.

📁 **Reference:** All repositories — system design question

#### Key Points to Cover:
```
Complete architecture:

1. AWS Account Structure:
   AWS Organization:
   ├── Management Account (billing, SCPs)
   ├── Security Account (GuardDuty, CloudTrail aggregation)
   ├── Shared Services Account (ECR, Nexus, Vault, Jenkins)
   ├── Dev Account (EKS dev cluster)
   ├── Staging Account (EKS staging cluster)
   └── Prod Account (EKS prod cluster, RDS, ElastiCache)

2. Container Registry (Shared Services):
   ECR repository per microservice
   Cross-account access via ECR resource policy
   Image scanning: ECR Enhanced Scanning (Inspector)
   Immutable tags enforced

3. CI/CD Pipeline (GitHub Actions):
   On PR:     tests → SAST (Semgrep) → SCA (Snyk) → lint → plan
   On merge:  build image → Trivy scan → push ECR → sign (Cosign)
              → deploy to dev (auto) → integration tests
              → promote to staging (auto) → E2E tests
              → promote to prod (manual approval via GitHub Environment)

4. GitOps / Deployment (ArgoCD per cluster):
   Dev cluster:    ArgoCD with auto-sync, auto-prune
   Staging:        ArgoCD with auto-sync, no auto-prune
   Prod:           ArgoCD with manual sync only
   ApplicationSet: one definition deploys all 20 services per cluster

5. Blue/Green Deployments:
   Argo Rollouts: manages blue/green at pod level
   ALB ingress: weighted target groups (100% blue → 100% green)
   Rollback: instant (switch weights back)

   apiVersion: argoproj.io/v1alpha1
   kind: Rollout
   spec:
     strategy:
       blueGreen:
         activeService: payment-active
         previewService: payment-preview
         autoPromotionEnabled: false  # manual promotion to prod
         scaleDownDelaySeconds: 60    # keep blue for 60s after green promoted

6. Secrets Management (HashiCorp Vault):
   Vault deployed in Shared Services account
   Vault Agent Injector: sidecar injects secrets into pod
   Secrets never in Git, never in env vars (injected at runtime)
   Or: AWS Secrets Manager + External Secrets Operator

7. Audit Logging of all deployments:
   CloudTrail: every AWS API call (ECR push, EKS deploy)
   ArgoCD: deployment history (who synced, which commit, when)
   GitHub Actions: complete workflow audit (in GitHub audit log)
   ArgoCD Notifications: Slack message per deployment:
     "Payment service v1.2.3 deployed to prod by @siddharth"

8. Observability:
   Prometheus Operator: deployed per cluster
   Thanos: aggregate metrics across clusters
   Grafana: central dashboard (all clusters)
   Loki: log aggregation
   Jaeger: distributed tracing
   PagerDuty: incident alerting

9. Networking:
   VPC per account, Transit Gateway for cross-account connectivity
   NACLs: strict, allowing only required traffic
   Security Groups: least privilege per service
   VPC Endpoints: ECR, S3, Secrets Manager (no internet egress)
```

> 💡 **Interview tip:** System design questions test **architectural breadth + justification**. Don't just list services — explain WHY. "Separate AWS accounts per environment" is the recommendation because it provides the strongest isolation (blast radius, IAM boundary, billing separation). If someone compromises dev, they can't access prod. One account for all environments means dev IAM mistakes can affect prod. Also: **always mention the "Day 2" concerns** (upgrades, monitoring, backup) not just Day 1 setup — this shows operational maturity.

---

### Q90 — All Topics | System Design | Advanced

> Design the **complete SRE platform** to achieve: 99.99% uptime SLO, MTTR under 5 minutes, zero-downtime deployments, full observability (metrics/logs/traces), and automated incident response.

📁 **Reference:** All repositories — comprehensive SRE design

#### Key Points to Cover:
```
1. Zero-downtime deployments (99.99% SLO baseline):
   Argo Rollouts: canary or blue/green
   PodDisruptionBudgets: always minAvailable >= 2
   Rolling update: maxUnavailable: 0, maxSurge: 1
   Readiness probes: gate traffic until ready
   PreStop hooks: drain connections before termination
   terminationGracePeriodSeconds: 60 (enough to finish in-flight requests)

2. Full observability (before you can automate, you must observe):
   Metrics:    Prometheus + Thanos (long-term) + Grafana
   Logs:       Loki or Elasticsearch + Kibana
   Traces:     Jaeger / Tempo (OpenTelemetry SDK in apps)
   SLOs:       Prometheus recording rules + error budget dashboards

3. Alerting strategy (MTTR goal = 5 minutes):
   Multi-burn-rate alerts (Google SRE Workbook)
   Fast burn (14.4x): PagerDuty page → on-call in < 2 minutes
   Slow burn (3x): Slack notification → acknowledge in hours
   No alert if no SLO impact (alert on symptoms, not causes)
   Alert quality: < 1 alert/day average (no noise)

4. Automated incident response (MTTR reduction):
   Runbook automation (Ansible/Python scripts):
     → "High error rate" → auto-rollback if deployed in last 30 min
     → "Pod crash loop" → auto-scale + alert
     → "Disk full" → auto-cleanup + alert
   PagerDuty + Rundeck/AWS Systems Manager: one-click runbook execution

   Slack bot for common actions:
     /rollback payment-service   → triggers ArgoCD rollback
     /scale payment-service 10   → scales deployment

5. Incident management process:
   Severity levels: P1 (SLO breach) → P2 (degraded) → P3 (at risk)
   P1: Automated detection → PagerDuty → on-call in 2 min
   On-call: acknowledge → triage (2 min) → mitigate (3 min) → MTTR < 5 min
   Post-incident: blameless postmortem within 48 hours

6. Chaos engineering (prevent incidents):
   Chaos Monkey: random pod kills in production (Netflix approach)
   AWS Fault Injection Simulator: AZ failures, network latency
   Run chaos during business hours when team is available
   Establish steady state → run experiment → verify steady state returns

7. Capacity planning (prevent future incidents):
   Horizontal Pod Autoscaler + KEDA for event-driven scaling
   Cluster Autoscaler + Karpenter for node scaling
   Forecast: Prometheus query for load growth trend
   Pre-scale before known peak events (Black Friday, product launches)

8. SLO management:
   Each service has: availability SLO + latency SLO
   Error budget: calculated daily
   Budget frozen (< 10% remaining): no feature deploys, only reliability work
   Budget healthy (> 50%): ship features

MTTR breakdown for < 5 minutes:
  Detection:   0-2 minutes (fast burn alert fires)
  Acknowledge: 0-1 minutes (PagerDuty, on-call available)
  Mitigate:    1-3 minutes (rollback command, auto-runbook)
  Resolve:     30 seconds (ArgoCD rollback takes 30s)
  Total:       < 5 minutes
```

> 💡 **Interview tip:** The 99.99% SLO means **52 minutes of downtime per year** (4.4 minutes/month). This sounds aggressive but it's achievable with: (1) **zero-downtime deployments** (eliminate deployment downtime entirely), (2) **fast automated rollback** (reduce incident duration to < 5 minutes), (3) **multi-AZ with auto-failover** (eliminate AZ-level single points of failure). The key insight: at 99.99%, you can't afford to have humans manually diagnose and fix — the **automated rollback on error rate spike** is what enables MTTR < 5 minutes. Humans would take 15-30 minutes; automated response takes 30 seconds.

---

*Q46–Q90 Enhanced with Key Points to Cover + Interview Tips*
*DevOps / SRE Interview Preparation — nawab312 GitHub repositories*
