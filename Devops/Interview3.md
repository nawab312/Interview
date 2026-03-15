# DevOps / SRE / Cloud Interview Questions (Q91–Q135)

> Each question includes: Key Points to Cover + Interview Tip
> Format mirrors Q64 reference style.

---

## Table of Contents

| # | Topic | Type | Difficulty |
|---|---|---|---|
| 91 | Kubernetes | Conceptual | Advanced |
| 92 | AWS | Scenario-Based | Advanced |
| 93 | Linux / Bash | Conceptual | Advanced |
| 94 | Terraform | Scenario-Based | Advanced |
| 95 | Prometheus | Conceptual | Advanced |
| 96 | Kubernetes | Troubleshooting | Advanced |
| 97 | Git | Conceptual | Advanced |
| 98 | AWS | Conceptual | Advanced |
| 99 | Jenkins | Scenario-Based | Advanced |
| 100 | ELK Stack | Conceptual | Advanced |
| 101 | Kubernetes | Scenario-Based | Advanced |
| 102 | Linux / Bash | Troubleshooting | Advanced |
| 103 | Terraform | Conceptual | Advanced |
| 104 | GitHub Actions | Scenario-Based | Advanced |
| 105 | AWS | Troubleshooting | Advanced |
| 106 | Prometheus | Scenario-Based | Advanced |
| 107 | Kubernetes | Conceptual | Advanced |
| 108 | ArgoCD | Troubleshooting | Advanced |
| 109 | Linux / Bash | Scenario-Based | Advanced |
| 110 | AWS | Scenario-Based | Advanced |
| 111 | Kubernetes | Troubleshooting | Advanced |
| 112 | Grafana | Conceptual | Advanced |
| 113 | Terraform | Troubleshooting | Advanced |
| 114 | Python | Scenario-Based | Advanced |
| 115 | AWS | Conceptual | Advanced |
| 116 | Kubernetes | Scenario-Based | Advanced |
| 117 | ELK Stack | Troubleshooting | Advanced |
| 118 | Jenkins | Conceptual | Advanced |
| 119 | Linux / Bash | Conceptual | Advanced |
| 120 | AWS | Scenario-Based | Advanced |
| 121 | Kubernetes | Conceptual | Advanced |
| 122 | GitHub Actions | Conceptual | Advanced |
| 123 | Terraform | Scenario-Based | Advanced |
| 124 | Prometheus | Troubleshooting | Advanced |
| 125 | AWS | Conceptual | Advanced |
| 126 | Kubernetes | Scenario-Based | Advanced |
| 127 | ArgoCD | Scenario-Based | Advanced |
| 128 | Linux / Bash | Scenario-Based | Advanced |
| 129 | AWS | Scenario-Based | Advanced |
| 130 | Kubernetes | Conceptual | Advanced |
| 131 | Grafana | Troubleshooting | Advanced |
| 132 | Terraform | Conceptual | Advanced |
| 133 | Git | Scenario-Based | Advanced |
| 134 | AWS + Kubernetes | Scenario-Based | Advanced |
| 135 | All Topics | System Design | Advanced |

---

### Q91 — Kubernetes | Conceptual | Advanced

> Explain the **Kubernetes Admission Control** mechanism. What is the difference between a **Validating Admission Webhook** and a **Mutating Admission Webhook**? Give a real-world example of each and explain the **order in which they are called**.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md`, `10_CRDs_EXTENSIBILITY.md`

```
Admission Control Pipeline:
  kubectl apply → API Server → Authn → Authz → Admission → etcd

  Order:
    1. Mutating Admission Webhooks   (can MODIFY the object)
    2. Schema Validation             (check object structure)
    3. Validating Admission Webhooks (can only ALLOW or DENY)

  Mutating runs BEFORE validating so validators see the final object.

Mutating Webhook — modifies the incoming object:
  Real-world example: Istio sidecar injection
    Pod submitted without envoy container
    Mutating webhook adds envoy sidecar automatically
    Pod lands in etcd with the sidecar already injected

  Other uses:
    → Add default resource limits if missing
    → Inject environment variables
    → Normalize image tags (:latest → :sha256)

Validating Webhook — approves or rejects, cannot change:
  Real-world example: OPA/Gatekeeper policy enforcement
    Policy: all Deployments must have a "team" label
    Validating webhook checks incoming Deployment
    Missing label → returns allowed: false → request rejected

  Other uses:
    → Block images from unapproved registries
    → Enforce naming conventions on namespaces
    → Reject pods requesting too many resources

Webhook configuration:
  failurePolicy: Fail    → webhook down = all requests blocked
  failurePolicy: Ignore  → webhook down = requests proceed (less safe)

  namespaceSelector:     → exclude kube-system to avoid circular dependency
  timeoutSeconds: 10     → max 30s hard limit by API server

Key difference:
  Mutating  → can patch the object (JSON Patch response)
  Validating → can only return allowed: true/false
```

💡 **Interview tip:** The order is critical — mutating always runs before validating. This is intentional: you want defaults injected (mutating) before you validate the final object against policy. A common mistake is writing a validating webhook that rejects pods for missing a securityContext, not realising a mutating webhook would have added it. Also always mention `failurePolicy` — a webhook outage with `Fail` policy has taken down entire clusters in production.

---

### Q92 — AWS | Scenario-Based | Advanced

> Your S3 → Lambda → RDS → SNS pipeline is **failing silently** — files are uploaded but sometimes no notification is sent and no record appears in RDS. There are no errors in CloudWatch Logs.

📁 **Reference:** `nawab312/AWS` — Lambda, S3 events, SNS, RDS, CloudWatch, X-Ray

```
Step 1 — Check if Lambda is even being triggered:
  CloudWatch Metrics → Lambda → Invocations
  → If invocations = 0 after S3 upload → S3 event trigger misconfigured
  → Check S3 Event Notifications → does it point to correct Lambda ARN?
  → Check Lambda resource policy allows s3.amazonaws.com to invoke it

Step 2 — Check Lambda execution:
  CloudWatch Logs → /aws/lambda/<function-name>
  → No logs at all → Lambda not invoked (S3 trigger issue)
  → Logs exist but no error → silent failure in code (exception swallowed)
  → Check for: try/except blocks that catch and ignore exceptions

Step 3 — Enable AWS X-Ray on Lambda:
  → Traces show exact duration of each segment
  → Which segment is failing: S3 read? RDS write? SNS publish?
  → Identify the exact line where execution stops

Step 4 — Check Lambda timeout and memory:
  → Default timeout: 3 seconds (too low for RDS connection + query)
  → Check CloudWatch: Duration metric vs configured timeout
  → If duration ≈ timeout → Lambda is timing out silently
  → Fix: increase timeout to 30-60 seconds for DB workloads

Step 5 — Check RDS connectivity from Lambda:
  → Lambda must be in same VPC as RDS
  → Security Group: RDS must allow inbound from Lambda SG on port 5432
  → Lambda cold start + VPC ENI attachment adds ~1-2 seconds

Step 6 — Check Lambda concurrency and throttling:
  CloudWatch Metrics → Throttles
  → Throttled invocations = events dropped silently (no retry by default for S3)
  → S3 event notifications are async — if Lambda throttled, event is lost
  → Fix: configure Dead Letter Queue (DLQ) on Lambda → SQS/SNS

Step 7 — Enable Dead Letter Queue:
  aws lambda update-function-configuration \
    --function-name my-pipeline \
    --dead-letter-config TargetArn=arn:aws:sqs:...:dlq-pipeline
  → Failed/throttled invocations go to DLQ
  → Alert on DLQ depth > 0

Common root causes for silent failure:
  → Lambda timeout (no error logged — just stops)
  → Exception swallowed in try/except
  → S3 trigger not configured for all file types (only .jpg not .png)
  → Lambda not in VPC → can't reach RDS (connection timeout)
  → RDS connection pool exhausted → Lambda waits → times out
```

💡 **Interview tip:** "No errors in CloudWatch Logs" is the key clue — it means either Lambda is not being invoked at all (check invocation metrics, not just logs) or the error is being swallowed in code. Always check metrics AND logs. The most common real cause: Lambda timeout — it just stops execution with no error message. Always set up a DLQ from day one for async Lambda triggers — S3 event notifications do not retry on Lambda failure.

---

### Q93 — Linux / Bash | Conceptual | Advanced

> Explain **`iptables`** and **`nftables`**. Walk through the **5 built-in chains** for an incoming HTTP request to a web server.

📁 **Reference:** `nawab312/DSA` → `Linux` — networking, iptables

```
iptables — legacy packet filtering framework (still widely used):
  → Operates on tables: filter, nat, mangle, raw, security
  → Rules evaluated top to bottom, first match wins
  → Default policy: ACCEPT or DROP

nftables — modern replacement (default on newer distros):
  → Single framework replaces iptables/ip6tables/arptables
  → Better performance (JIT compilation)
  → Cleaner syntax
  → nft command instead of iptables

5 Built-in Chains (Netfilter hooks):

  PREROUTING   → packet arrives, before routing decision
  INPUT        → packet destined for THIS machine
  FORWARD      → packet passing THROUGH this machine
  OUTPUT       → packet originating FROM this machine
  POSTROUTING  → packet leaving, after routing decision

Incoming HTTP request (external client → web server on port 80):

  1. Packet arrives at network interface
          │
          ▼
  2. PREROUTING chain (nat table)
     → DNAT rules applied here
     → Example: redirect port 80 → 8080 for non-root app
     → kube-proxy uses PREROUTING to redirect Service IPs
          │
          ▼
  3. Routing decision
     → Is destination IP this machine? YES → INPUT
     → Is it for another machine?      YES → FORWARD
          │
          ▼
  4. INPUT chain (filter table)
     → Main firewall rules checked here
     → ACCEPT: port 80 allowed → packet delivered to web server
     → DROP:   port 80 blocked → packet silently discarded
     → REJECT: port 80 blocked → client gets connection refused
          │
          ▼
  5. Web server process receives the packet

Response path (web server → client):
  OUTPUT chain → POSTROUTING chain → out to network

iptables vs nftables:
  iptables:  separate tools per protocol, linear rule matching
  nftables:  unified, sets for efficient multi-IP matching, maps
  Kubernetes: still uses iptables heavily (kube-proxy iptables mode)
```

💡 **Interview tip:** Mention that Kubernetes heavily uses iptables — kube-proxy writes PREROUTING/POSTROUTING rules to implement Service ClusterIP DNAT. When a pod calls a Service IP, iptables redirects it to a real pod IP. This is why switching to IPVS mode improves performance at scale — IPVS uses hash tables (O(1) lookup) vs iptables linear rules (O(n) lookup). In security group troubleshooting, knowing PREROUTING vs INPUT matters — NACLs are stateless and apply before routing, Security Groups are stateful and apply at the INPUT/OUTPUT level.

---

### Q94 — Terraform | Scenario-Based | Advanced

> You need to manage **Terraform at scale** across **50 microservices**. How would you design a Terraform platform to standardize infrastructure?

📁 **Reference:** `nawab312/Terraform` — modules, Terraform Cloud, module registry

```
Problem with 50 microservices without a platform:
  → 50 copies of VPC code (drift between teams)
  → Different Terraform versions per service
  → No visibility into what's deployed where
  → No guardrails — teams can create anything

Solution: Internal Module Registry + Standardized Patterns

Layer 1 — Shared Module Library (private registry):
  modules/
  ├── vpc/          → standard VPC + subnets + NAT
  ├── ecs-service/  → ECS task + service + ALB + autoscaling
  ├── rds/          → RDS + parameter group + subnet group
  ├── eks-cluster/  → EKS + node groups + IRSA
  └── s3-bucket/    → S3 + encryption + lifecycle + access logging

  Teams consume modules with pinned versions:
  module "api_service" {
    source  = "git::https://github.com/company/tf-modules//ecs-service?ref=v2.1.0"
    name    = "payment-api"
    cpu     = 512
    memory  = 1024
  }

Layer 2 — Terragrunt for DRY configuration:
  environments/
  ├── prod/
  │   ├── terragrunt.hcl      → backend config, provider version
  │   └── payment-api/
  │       └── terragrunt.hcl  → just the inputs, no boilerplate
  └── staging/
      └── payment-api/
          └── terragrunt.hcl

  → Terragrunt generates backend config automatically per service
  → No copy-pasting provider blocks
  → Single command: terragrunt run-all apply

Layer 3 — Remote State per service:
  S3 bucket: company-terraform-state
  Key pattern: <env>/<service>/terraform.tfstate
  → prod/payment-api/terraform.tfstate
  → prod/user-service/terraform.tfstate
  DynamoDB: terraform-state-locks (prevent concurrent applies)

Layer 4 — CI/CD integration:
  PR opened  → terraform plan (post as PR comment)
  PR merged  → terraform apply (auto or manual approval gate)
  Tools: Atlantis, Terraform Cloud, or custom GitHub Actions

Layer 5 — Policy as Code (Sentinel / OPA):
  → Enforce: all S3 buckets must be encrypted
  → Enforce: no EC2 instance type above m5.2xlarge in dev
  → Enforce: all resources must have cost-center tag

Versioning strategy:
  Major version (v2.0.0) → breaking change, teams migrate when ready
  Minor version (v1.1.0) → new optional feature, backwards compatible
  Patch version (v1.0.1) → bug fix, teams should update
```

💡 **Interview tip:** The answer interviewers want to hear is: modules + versioning + Terragrunt (or similar) + remote state isolation per service + CI/CD automation. The biggest mistake at scale is a monolithic state file — if one team's apply corrupts state, everyone is blocked. Per-service state files are non-negotiable. Mention Atlantis as the open-source alternative to Terraform Cloud — it runs in your own infrastructure and auto-comments plan output on PRs, which teams love.

---

### Q95 — Prometheus | Conceptual | Advanced

> Explain **Prometheus federation** and **Thanos**. When would you use each? Explain Thanos components.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`

```
Problem Prometheus federation and Thanos both solve:
  Single Prometheus: limited to one server's disk (typically 30-90 days)
  Large clusters: too many metrics for one Prometheus to scrape
  Multi-cluster: no single view across all clusters

Federation — hierarchical Prometheus scraping Prometheus:
  Cluster Prometheus A  ──┐
  Cluster Prometheus B  ──┼──► Global Prometheus (scrapes /federate endpoint)
  Cluster Prometheus C  ──┘

  Global scrapes aggregated metrics from cluster Prometheuses
  Cluster Prometheuses keep full detail
  Global keeps only high-level summaries

  Limitations:
  → Global Prometheus is a single point of failure
  → Limited retention (still bounded by local disk)
  → Complex to manage as clusters grow
  → Not true long-term storage

Thanos — production-grade long-term storage:
  Thanos components:

  Sidecar (runs next to each Prometheus):
    → Uploads Prometheus blocks to S3/GCS/Azure Blob every 2 hours
    → Exposes Prometheus data to Thanos Query via gRPC
    → Allows Prometheus to stay stateless (data in object storage)

  Store Gateway:
    → Reads historical blocks from object storage (S3)
    → Exposes them to Thanos Query
    → Caches index information for performance

  Querier (Query):
    → Single global query endpoint (PromQL)
    → Fans out queries to: Sidecars + Store Gateways
    → Deduplicates results (same metric from 2 Prometheus = 1 result)
    → Grafana points here instead of individual Prometheus instances

  Compactor:
    → Runs as a batch job against object storage
    → Merges small 2-hour blocks into larger blocks (2h → 24h → 2w)
    → Applies downsampling (1h resolution for old data saves space)
    → Enforces retention policies (delete data older than N days)

  Ruler (optional):
    → Evaluates alerting and recording rules against global data
    → Needed for cross-cluster alerts

Federation vs Thanos:
  Federation:  simple, no extra components, short-term, small scale
  Thanos:      complex, unlimited retention, multi-cluster, production
```

💡 **Interview tip:** Thanos is the standard answer for long-term Prometheus storage in production. The key insight is that Prometheus remains unchanged — the Thanos sidecar sits next to it and handles upload to S3. Querier provides a single PromQL endpoint that looks like one Prometheus but queries all clusters. Mention that Cortex and Mimir are alternatives to Thanos — they take a different approach (push-based ingestion) but solve the same problem. For retention: Thanos Compactor downsampling means you can keep years of data affordably — 5-minute resolution for recent data, 1-hour resolution for data older than 30 days.

---

### Q96 — Kubernetes | Troubleshooting | Advanced

> Your **Kubernetes API server** is responding very slowly — `kubectl` commands take 30+ seconds. The cluster has 500 nodes and 10,000 pods.

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md`, `09_CLUSTER_OPERATIONS.md`

```
Step 1 — Identify the bottleneck layer:
  # Is it API server or etcd?
  time kubectl get nodes          → measures full round trip
  time kubectl get nodes --watch  → tests watch connection
  curl -s http://localhost:2379/health  → check etcd directly

Step 2 — Check API server metrics:
  kubectl get --raw /metrics | grep apiserver_request_duration
  → apiserver_request_duration_seconds{verb="LIST"} p99 > 5s → LIST calls slow
  → Check: which resources are being listed most frequently?

Step 3 — Check etcd health:
  etcdctl endpoint status --write-out=table
  etcdctl endpoint health
  → DB size approaching quota (default 2GB) → etcd goes read-only
  → High leader elections → network or disk issue
  → Watch latency > 100ms → disk too slow (needs SSD)

  etcdctl defrag             → reclaim space
  etcdctl compact <revision> → compact old revisions

Step 4 — Check for expensive LIST operations:
  # Too many controllers or informers doing full LIST repeatedly
  kubectl get --raw /metrics | grep apiserver_current_inflight_requests
  → List of all pods across 10,000 pods is very expensive
  → Check: any custom controllers doing LIST without watch?
  → Check: kubectl get pods -A in scripts run in a tight loop?

Step 5 — Check watch cache:
  # API server has an in-memory watch cache
  # If cache is disabled or too small → every LIST goes to etcd
  kube-apiserver flags:
    --watch-cache=true           (must be true)
    --watch-cache-sizes=pods#500 (increase cache for large resources)

Step 6 — Check API priority and fairness:
  kubectl get flowschemas
  kubectl get prioritylevelconfigurations
  → Low-priority requests (CI scripts) starving high-priority (controllers)
  → Adjust priority levels for critical controllers

Step 7 — Scale API server:
  → Run 3+ API server replicas behind load balancer
  → Each API server has its own watch cache (reduces etcd load)
  → Use separate etcd cluster for events (very high write volume)
    --etcd-servers-overrides=events#https://etcd-events:2379

Common root causes at 500 nodes / 10,000 pods:
  → etcd disk too slow (magnetic disk instead of SSD)
  → etcd DB size near quota
  → Too many objects in single namespace (split across namespaces)
  → Custom controllers doing LIST without pagination
  → No resource version in LIST (forces full etcd scan)
```

💡 **Interview tip:** The most common cause at scale is etcd disk I/O — etcd is extremely sensitive to fsync latency and must run on dedicated SSDs. The second most common is object count — 10,000 pods means LIST pods is expensive. Always mention separating the etcd cluster for events — Kubernetes events are high-volume writes that pollute the main etcd cluster and cause latency spikes for everything else. The API priority and fairness feature (APF) is a newer answer that shows depth — it prevents runaway LIST calls from starving critical controllers.

---

### Q97 — Git | Conceptual | Advanced

> Explain **`git cherry-pick`**, **`git bisect`**, and **`git stash`** with real-world DevOps scenarios.

📁 **Reference:** `nawab312/CI_CD` → `Git`

```
git cherry-pick — apply a specific commit to another branch:

  Scenario: hotfix committed to develop, needs to go to main NOW
  
  develop:  A → B → C → D(hotfix) → E → F
  main:     A → B → G → H

  git checkout main
  git cherry-pick D          # apply only commit D to main
  main: A → B → G → H → D'  # D' is copy of D

  Use when:
  → Apply a bug fix from develop to an older release branch
  → Pull a single feature commit into a release without the rest
  → Backport a security patch to multiple versions

  Warning: creates a NEW commit with different SHA
           if D is later merged via PR, you get duplicate changes
           always prefer proper branching over repeated cherry-picks

git bisect — binary search through commits to find a regression:

  Scenario: "it worked 3 months ago, now it doesn't"
            500 commits between working and broken state

  git bisect start
  git bisect bad HEAD              # current commit is broken
  git bisect good v1.2.0           # this version was working
  → Git checks out commit in the middle automatically
  → You run your test:
      if broken: git bisect bad
      if working: git bisect good
  → Git narrows down: 500 commits → 250 → 125 → 63... → 1 commit
  → After ~9 steps: "commit abc123 is the first bad commit"

  CI/CD use: git bisect run ./test.sh
  → Automated bisect — runs your test script at each step
  → Finds the breaking commit without manual intervention
  → Extremely useful after a large merge or dependency upgrade

git stash — temporarily shelve uncommitted work:

  Scenario: you are mid-feature, urgent hotfix needed

  # Working on feature, files modified but not committed
  git stash                       # shelve current work
  git checkout main
  git checkout -b hotfix/critical
  # fix the bug, commit, merge
  git checkout feature/my-work
  git stash pop                   # restore your work

  Useful commands:
  git stash list                  # see all stashes
  git stash push -m "WIP: auth"  # named stash
  git stash pop                   # restore + delete stash
  git stash apply stash@{2}      # apply specific stash, keep it
  git stash drop stash@{0}       # delete a stash
```

💡 **Interview tip:** `git bisect run` is the answer that impresses interviewers — automated binary search through commits to find a regression is a genuine production skill. Mention a scenario where a dependency was upgraded and something broke — bisect found the exact commit in minutes across 300 commits. For cherry-pick, always mention the risk: if you cherry-pick a commit and then merge the branch later, you get a duplicate. The clean solution is to use proper feature branches and merge them, not cherry-pick repeatedly.

---

### Q98 — AWS | Conceptual | Advanced

> Explain the difference between **EKS** and **ECS with Fargate**. What are the networking differences? When would you choose EKS?

📁 **Reference:** `nawab312/AWS` — EKS, ECS, Fargate, VPC CNI

```
ECS with Fargate — serverless containers, AWS-managed:
  → No nodes to manage (AWS provisions compute)
  → Task definition = your container spec
  → Service = desired count + placement
  → Networking: awsvpc mode → each task gets its own ENI + private IP
  → Simpler: no Kubernetes knowledge required
  → Limited extensibility (no custom schedulers, no CRDs)

EKS — Kubernetes on AWS, managed control plane:
  → AWS manages control plane (API server, etcd, scheduler)
  → You manage worker nodes (EC2 or Fargate for pods)
  → Full Kubernetes API — CRDs, operators, Helm, custom schedulers
  → Networking: VPC CNI → each pod gets its own VPC IP

Networking deep dive:

  ECS awsvpc mode:
    Each ECS task → own ENI → own VPC IP
    Security groups applied at task level (not host level)
    Direct task-to-task communication via VPC IPs
    Limited by ENIs per EC2 instance type

  EKS VPC CNI:
    Each pod → own VPC IP (same subnet as node)
    Pod IPs routable within VPC without NAT
    Security Groups for Pods: optional per-pod SG
    VPC CNI prefix delegation: more IPs per node
    Max pods limited by instance ENI × IPs per ENI

  Key difference:
    ECS: security at task level via SG, simpler
    EKS: security via NetworkPolicy (Calico/Cilium) OR pod SGs
         more flexible, more complex

When to choose EKS:
  ✓ Team already knows Kubernetes
  ✓ Need custom operators (database operators, etc.)
  ✓ Multi-cloud portability (same manifests on GKE/AKS)
  ✓ Complex networking requirements (NetworkPolicy, service mesh)
  ✓ Large scale with many workload types
  ✓ Need Helm, ArgoCD, Kyverno, Falco

When to choose ECS Fargate:
  ✓ Small team, no Kubernetes expertise
  ✓ Simple microservices, no custom operators needed
  ✓ Want zero node management
  ✓ Cost: Fargate pricing often cheaper for variable workloads
  ✓ AWS-native: tight integration with CodePipeline, App Mesh

Cost comparison:
  ECS Fargate: pay per vCPU/memory/second — no idle node cost
  EKS + EC2:   pay for nodes even when underutilised
  EKS + Fargate: per pod, but EKS control plane = $0.10/hr fixed
```

💡 **Interview tip:** The honest answer is: ECS Fargate is simpler and often cheaper for straightforward workloads; EKS is better when you need the full Kubernetes ecosystem. In interviews, acknowledge both have valid use cases — don't just say "EKS is better." The key networking difference is that both give each container/pod its own VPC IP (awsvpc and VPC CNI respectively), which means you can apply security groups at the container level — this is a major security advantage over the old bridge networking model.

---

### Q99 — Jenkins | Scenario-Based | Advanced

> Implement a **Jenkins pipeline** for a monorepo with 10 microservices — only trigger changed services, run tests in parallel, single deployment gate.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins`

```groovy
pipeline {
  agent any

  environment {
    SERVICES = 'auth,payment,user,order,notification,inventory,gateway,search,analytics,reporting'
  }

  stages {
    stage('Detect Changes') {
      steps {
        script {
          // Get list of changed files since last successful build
          def changedFiles = sh(
            script: "git diff --name-only HEAD~1 HEAD",
            returnStdout: true
          ).trim().split('\n')

          // Map changed files to service names
          env.CHANGED_SERVICES = SERVICES.split(',').findAll { service ->
            changedFiles.any { it.startsWith("services/${service}/") }
          }.join(',')

          echo "Changed services: ${env.CHANGED_SERVICES}"

          if (!env.CHANGED_SERVICES) {
            currentBuild.result = 'SUCCESS'
            error('No service changes detected — skipping pipeline')
          }
        }
      }
    }

    stage('Parallel Test') {
      steps {
        script {
          def parallelStages = [:]

          env.CHANGED_SERVICES.split(',').each { service ->
            parallelStages["Test: ${service}"] = {
              stage("Test ${service}") {
                dir("services/${service}") {
                  sh 'make test'
                  junit 'test-results/*.xml'
                }
              }
            }
          }

          // failFast: true — if one fails, cancel all others
          parallelStages.failFast = true
          parallel parallelStages
        }
      }
    }

    stage('Parallel Build') {
      steps {
        script {
          def parallelBuilds = [:]

          env.CHANGED_SERVICES.split(',').each { service ->
            parallelBuilds["Build: ${service}"] = {
              dir("services/${service}") {
                sh "docker build -t ${service}:${GIT_COMMIT} ."
                sh "docker push company/${service}:${GIT_COMMIT}"
              }
            }
          }

          parallel parallelBuilds
        }
      }
    }

    // Single deployment gate — all services must pass before any deploys
    stage('Deployment Approval') {
      when { branch 'main' }
      steps {
        input message: "Deploy ${env.CHANGED_SERVICES} to production?",
              ok: "Deploy"
      }
    }

    stage('Deploy All') {
      when { branch 'main' }
      steps {
        script {
          env.CHANGED_SERVICES.split(',').each { service ->
            sh "kubectl set image deployment/${service} ${service}=company/${service}:${GIT_COMMIT}"
            sh "kubectl rollout status deployment/${service} --timeout=5m"
          }
        }
      }
    }
  }
}
```

💡 **Interview tip:** The key insight is detecting changed services using `git diff` — this is the foundation of monorepo CI efficiency. `failFast: true` in the parallel map is important — if auth tests fail, you don't want to waste 10 minutes waiting for all other services to finish before you know the build is broken. The single deployment gate (input step after all tests pass) ensures no partially-broken state reaches production. In real production, replace the `input` step with an automated integration test gate.

---

### Q100 — ELK Stack | Conceptual | Advanced

> Explain **Elasticsearch ILM** — 4 phases. Write a policy keeping hot data 7 days, warm after 7 days, delete after 30.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack`

```
ILM — Index Lifecycle Management:
  Automatically transitions indices through phases based on age or size
  Prevents disk from filling up, optimises performance per data age

4 Phases:

  HOT phase (active writes + reads):
    → Index receiving new data right now
    → On fast SSD storage
    → Rollover when: size > 50GB OR age > 1 day OR docs > 100M
    → Primary + replica shards for write performance

  WARM phase (read-only, less frequent queries):
    → No new data, historical queries only
    → Can move to cheaper/slower storage (warm nodes)
    → Force merge: reduces segment count → faster reads
    → Shrink: reduce shard count (from 5 primaries → 1)
    → Saves resources: no replica needed on single shard

  COLD phase (infrequent access, archival):
    → Rarely queried (compliance, audits)
    → Freeze index: unloads from memory, reloads on query
    → Move to cheapest storage tier (frozen nodes or S3-backed)
    → Searchable snapshots: data on S3, query without restoring

  DELETE phase:
    → Index permanently deleted
    → Or: snapshot to S3 before deleting (retain for compliance)

ILM policy for application logs (7-day hot, warm until 30 days, delete):

PUT _ilm/policy/app-logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_size": "50gb"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": { "max_num_segments": 1 },
          "shrink": { "number_of_shards": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}

Apply to index template so new indices automatically use the policy:
PUT _index_template/app-logs-template
{
  "index_patterns": ["app-logs-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "app-logs-policy",
      "index.lifecycle.rollover_alias": "app-logs"
    }
  }
}
```

💡 **Interview tip:** ILM is essential for production Elasticsearch — without it, indices grow forever and disk fills up (which causes RED cluster status). The rollover action in hot phase is the key concept — instead of one giant index, you get small daily indices that are easy to manage and delete. Mention that `min_age` is measured from rollover time, not index creation time. Common mistake: forgetting to set `rollover_alias` — without it, the rollover action doesn't know which index to roll over to.

---

### Q101 — Kubernetes | Scenario-Based | Advanced

> Your pod drops in-flight requests during rolling updates. Explain the **Pod termination lifecycle** and configure **zero-downtime shutdown**.

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`

```
Pod Termination Lifecycle (what happens when pod is deleted):

  T+0:   kubectl delete pod / rolling update triggers
  T+0:   Pod status → Terminating
  T+0:   Endpoint controller removes pod IP from Service EndpointSlice
         BUT: kube-proxy on all nodes takes time to update iptables rules
         GAP: new connections still routed to terminating pod for ~2-5 seconds

  T+0:   preStop hook executes (if configured)
         → pod waits here before SIGTERM is sent

  T+preStop: SIGTERM sent to container
             → app should start graceful shutdown
             → finish in-flight requests
             → stop accepting new connections

  T+terminationGracePeriodSeconds: SIGKILL sent (force kill)
             → default: 30 seconds

The problem — iptables propagation delay:
  Pod removed from endpoints → kube-proxy updates iptables
  This update takes 1-5 seconds across the cluster
  During this gap, new requests still reach the terminating pod
  App receives SIGTERM → closes listener → new requests get 502

Solution — preStop sleep bridge:

spec:
  terminationGracePeriodSeconds: 60   # must be > preStop + drain time
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 15"]
    # sleep 15 = wait for iptables propagation before SIGTERM

  SIGTERM handler in app:
    → Stop accepting NEW connections
    → Finish all IN-FLIGHT requests
    → Close DB connections
    → Exit with code 0

Complete zero-downtime configuration:

apiVersion: apps/v1
kind: Deployment
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # one extra pod during update
      maxUnavailable: 0    # never remove old pod before new one is ready
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: app
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 15"]

Full timeline with fix:
  T+0:   Pod marked Terminating, removed from endpoints
  T+0:   preStop starts (sleep 15)
  T+15:  iptables propagated on all nodes (no new traffic reaching pod)
  T+15:  SIGTERM sent to app
  T+15:  App finishes in-flight requests gracefully
  T+25:  App exits cleanly (well within 60s grace period)
  T+25:  Pod terminated — zero dropped requests
```

💡 **Interview tip:** The `sleep` in preStop is the non-obvious trick that most engineers miss — without it, there is a race condition between iptables propagation and SIGTERM. `maxUnavailable: 0` combined with `maxSurge: 1` ensures old pods are never removed before new ones pass readiness — this is the other half of the solution. Also mention that the readiness probe must return not-ready when the app is shutting down — some apps need explicit code to fail the readiness probe on SIGTERM.

---

### Q102 — Linux / Bash | Troubleshooting | Advanced

> A server has **load average 45.0 on an 8-core machine** but **CPU usage is only 15%**. What does this mean and how do you investigate?

📁 **Reference:** `nawab312/DSA` → `Linux` — load average, I/O wait

```
What load average 45 on 8 cores means:
  Load average = number of processes in RUNNING or UNINTERRUPTIBLE state
  8 cores can run 8 processes simultaneously
  Load 45 = 45 processes waiting to run, only 8 running at a time
  CPU at 15% means processes are NOT waiting for CPU

  Key insight: load average includes I/O wait (D state processes)
  High load + low CPU = processes blocked on I/O, not CPU

  Process states:
    R = Running or runnable (on CPU or waiting for CPU)
    D = Uninterruptible sleep (waiting for I/O — DISK or NETWORK)
    S = Interruptible sleep (waiting for event)
    Z = Zombie

Step 1 — Confirm it is I/O wait:
  top                          → check %wa (I/O wait column)
  → wa > 30% confirms disk I/O is the bottleneck
  → wa near 0% → check for D state processes on network I/O or locks

  iostat -x 1                  → disk I/O statistics
  → %util near 100% → disk saturated
  → await > 20ms → disk latency high
  → r/s and w/s → read/write operations per second

Step 2 — Find which processes are in D state:
  ps aux | awk '$8 == "D"'     → list all uninterruptible processes
  → Which processes? MySQL? Java app? System processes?

Step 3 — Find which disk is the bottleneck:
  iostat -x 1 5
  iotop -o                     → show only processes doing I/O right now
  → Which process is doing the most I/O?
  → Which disk device?

Step 4 — Determine the I/O type:
  # Is it reads or writes?
  iostat -x → rkB/s vs wkB/s
  → High writes → write-heavy workload (logging, DB writes)
  → High reads  → missing cache, cold data, no buffering

Step 5 — Common root causes:
  → Database running on slow EBS (gp2 instead of gp3/io1)
  → Disk full → writes blocked → D state processes pile up
  → NFS mount unresponsive → all NFS I/O hangs in D state
  → Swap thrashing → lots of page faults → high I/O
  → Log rotation writing huge files synchronously
  → MySQL without InnoDB buffer pool → every query hits disk

Step 6 — Quick checks:
  df -h              → is disk full?
  free -h            → is swap being used heavily?
  dmesg | tail -20   → any kernel errors? (disk errors, NFS issues)
  cat /proc/mdstat   → any RAID degraded?
```

💡 **Interview tip:** High load with low CPU is always I/O — this is a classic interview question. The key concept is that Linux load average counts D-state processes, not just CPU-bound ones. In production, the most common cause is database I/O — either slow EBS volumes or missing indexes causing full table scans. On AWS, switching from gp2 to gp3 EBS volumes (and provisioning IOPS) immediately fixes most disk I/O bottlenecks. Also mention: NFS hangs are notorious for causing D-state pile-ups because NFS operations are uninterruptible by default.

---

### Q103 — Terraform | Conceptual | Advanced

> Explain **`for_each`** vs **`count`**. When does `count` cause resource replacement that `for_each` avoids?

📁 **Reference:** `nawab312/Terraform` — meta-arguments

```
count — index-based resource creation:

  variable "buckets" {
    default = ["logs", "backups", "artifacts"]
  }

  resource "aws_s3_bucket" "this" {
    count  = length(var.buckets)
    bucket = var.buckets[count.index]
  }

  Creates:
    aws_s3_bucket.this[0] → "logs"
    aws_s3_bucket.this[1] → "backups"
    aws_s3_bucket.this[2] → "artifacts"

  THE PROBLEM — remove "backups" from the middle:
  variable "buckets" {
    default = ["logs", "artifacts"]  # removed "backups"
  }

  Terraform plan:
    aws_s3_bucket.this[0]  → no change ("logs")
    aws_s3_bucket.this[1]  → DESTROY "backups" + CREATE "artifacts"  ← !!
    aws_s3_bucket.this[2]  → DESTROY "artifacts"

  Index shifted! Terraform destroys and recreates "artifacts"
  even though you only wanted to remove "backups"
  In production: destroys S3 bucket with real data!

for_each — key-based resource creation:

  resource "aws_s3_bucket" "this" {
    for_each = toset(["logs", "backups", "artifacts"])
    bucket   = each.key
  }

  Creates:
    aws_s3_bucket.this["logs"]      → "logs"
    aws_s3_bucket.this["backups"]   → "backups"
    aws_s3_bucket.this["artifacts"] → "artifacts"

  Remove "backups":
  for_each = toset(["logs", "artifacts"])

  Terraform plan:
    aws_s3_bucket.this["logs"]      → no change
    aws_s3_bucket.this["backups"]   → DESTROY (only this one)
    aws_s3_bucket.this["artifacts"] → no change

  Correct! Only "backups" is destroyed.

for_each with maps (richer configuration):

  variable "buckets" {
    default = {
      logs      = { versioning = true }
      backups   = { versioning = true }
      artifacts = { versioning = false }
    }
  }

  resource "aws_s3_bucket" "this" {
    for_each = var.buckets
    bucket   = each.key
  }

  resource "aws_s3_bucket_versioning" "this" {
    for_each = var.buckets
    bucket   = aws_s3_bucket.this[each.key].id
    versioning_configuration {
      status = each.value.versioning ? "Enabled" : "Disabled"
    }
  }

Rule of thumb:
  count     → use ONLY for boolean (create this resource: yes/no)
  for_each  → use for all lists and maps of resources
```

💡 **Interview tip:** The index-shifting problem with `count` is a real production risk — removing an item from the middle of a list can trigger cascading destroys and recreates. This has caused production outages. The rule in practice: only use `count` for `count = var.create_resource ? 1 : 0` (conditional creation). For everything else, use `for_each`. Also mention: `for_each` cannot use values that are not known at plan time (e.g., computed values from other resources) — this is a limitation where `count` might be necessary.

---

### Q104 — GitHub Actions | Scenario-Based | Advanced

> Build a **matrix workflow**: 3 Node versions × 3 OS types, exclude Node 18 + macOS, fail fast, upload artifacts for failures only.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions`

```yaml
name: Matrix Build

on: [push, pull_request]

jobs:
  test:
    runs-on: ${{ matrix.os }}

    strategy:
      fail-fast: true        # cancel all jobs if any fails
      matrix:
        node: [18, 20, 22]
        os: [ubuntu-latest, windows-latest, macos-latest]
        exclude:
          - node: 18
            os: macos-latest  # exclude Node 18 + macOS combination

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        id: tests
        run: npm test
        # continue-on-error not set — failure propagates to fail-fast

      - name: Upload test artifacts (failed only)
        if: failure()           # only runs if previous step failed
        uses: actions/upload-artifact@v4
        with:
          name: test-results-node${{ matrix.node }}-${{ matrix.os }}
          path: |
            test-results/
            coverage/
          retention-days: 7

  # Summary job — runs after all matrix jobs complete
  summary:
    needs: test
    runs-on: ubuntu-latest
    if: always()              # run even if matrix jobs failed
    steps:
      - name: Check matrix result
        run: |
          if [ "${{ needs.test.result }}" = "success" ]; then
            echo "All matrix combinations passed"
          else
            echo "Some combinations failed"
            exit 1
          fi
```

```
Matrix expansion (9 combinations - 1 exclude = 8 jobs):

  node-18 + ubuntu    ✓
  node-18 + windows   ✓
  node-18 + macos     ✗ excluded
  node-20 + ubuntu    ✓
  node-20 + windows   ✓
  node-20 + macos     ✓
  node-22 + ubuntu    ✓
  node-22 + windows   ✓
  node-22 + macos     ✓

fail-fast: true means:
  If node-20 + windows fails → all other running jobs are cancelled immediately
  Saves CI minutes, gives faster feedback
```

💡 **Interview tip:** `if: failure()` is the key pattern for conditional artifact upload — only upload when the job actually fails to avoid storing gigabytes of artifacts for every successful run. `fail-fast: true` is the default in GitHub Actions matrix — mention that you might want `fail-fast: false` if you need to see ALL failure combinations (e.g., checking compatibility across all platforms even when one fails). The `retention-days` on artifacts is important — default is 90 days which accumulates storage costs quickly.

---

### Q105 — AWS | Troubleshooting | Advanced

> Your RDS PostgreSQL is getting `FATAL: remaining connection slots are reserved for non-replication superuser connections`. Immediate fix and long-term solution?

📁 **Reference:** `nawab312/AWS` — RDS, connection pooling, RDS Proxy

```
What the error means:
  PostgreSQL has a max_connections limit (default: varies by instance size)
  db.t3.medium  → max_connections ≈ 420
  db.r5.large   → max_connections ≈ 1700
  Reserved slots for superuser: 3 connections always reserved
  Error = all 420 slots taken, only superuser reserve left

Why this happens in microservices:
  10 microservices × 20 pods each × 5 connections per pod = 1,000 connections
  Exceeds max_connections immediately
  Each app server opens its own connection pool to RDS
  Connection pool never shrinks (keeps min connections open permanently)

Immediate fix — kill idle connections:
  # Connect as superuser via RDS Query Editor or psql:
  SELECT pg_terminate_backend(pid)
  FROM pg_stat_activity
  WHERE state = 'idle'
    AND state_change < NOW() - INTERVAL '10 minutes'
    AND pid <> pg_backend_pid();

  # Check who is using connections:
  SELECT client_addr, count(*), state
  FROM pg_stat_activity
  GROUP BY client_addr, state
  ORDER BY count DESC;

  # Increase max_connections temporarily (requires reboot):
  aws rds modify-db-instance \
    --db-instance-identifier mydb \
    --db-parameter-group-name custom-params
  # In parameter group: max_connections = 1000

Long-term solution — RDS Proxy:

  Without RDS Proxy:
    100 pods × 10 connections = 1,000 connections to RDS directly

  With RDS Proxy:
    100 pods → RDS Proxy (multiplexes connections)
    RDS Proxy → RDS (maintains pool of 50-100 connections)
    Pods see: 1,000 connections available
    RDS sees: 50-100 connections (connection multiplexing)

  aws rds create-db-proxy \
    --db-proxy-name my-proxy \
    --engine-family POSTGRESQL \
    --auth '[{"AuthScheme":"SECRETS","SecretArn":"arn:...","IAMAuth":"REQUIRED"}]' \
    --role-arn arn:aws:iam::123:role/rds-proxy-role \
    --vpc-subnet-ids subnet-123 subnet-456

  Application connects to proxy endpoint instead of RDS endpoint:
  postgres://my-proxy.proxy-xyz.us-east-1.rds.amazonaws.com:5432/mydb

Application-side fixes:
  → Set connection pool max size per pod: pool_max = 5 (not 20)
  → Set connection pool min size: pool_min = 1 (release idle connections)
  → Set idle connection timeout: pool_timeout = 300s
  → PgBouncer as sidecar: lightweight connection pooler

Formula for connection sizing:
  max_connections = (pods × pool_max) + buffer
  db.t3.medium handles ~400 connections
  With 20 pods: pool_max = 400 / 20 = 20 per pod maximum
```

💡 **Interview tip:** RDS Proxy is the AWS-native long-term solution — it multiplexes thousands of application connections into a small pool of actual database connections. It also adds IAM authentication, automatic failover (50% faster than direct connection on Multi-AZ failover), and connection pinning control. Mention PgBouncer as the self-managed alternative. The root cause is almost always microservices each maintaining their own connection pool without coordination — the total connection count grows linearly with pods and services.

---

### Q106 — Prometheus | Scenario-Based | Advanced

> Set up **Prometheus monitoring for Kafka on Kubernetes** — broker metrics, consumer lag, throughput.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`

```
Step 1 — Deploy JMX Exporter as Kafka sidecar:

  Kafka exposes metrics via JMX (Java Management Extensions)
  JMX Exporter converts JMX metrics to Prometheus format

  # In Kafka StatefulSet:
  containers:
  - name: kafka
    image: confluentinc/cp-kafka:7.5.0
    env:
    - name: KAFKA_JMX_PORT
      value: "9999"
    - name: EXTRA_ARGS
      value: "-javaagent:/opt/jmx-exporter/jmx_prometheus_javaagent.jar=9404:/opt/jmx-exporter/kafka.yml"
    ports:
    - name: metrics
      containerPort: 9404

Step 2 — Deploy Kafka Exporter (consumer lag):

  JMX Exporter covers broker metrics
  kafka-exporter covers consumer group lag (needs Kafka API access)

  helm install kafka-exporter prometheus-community/prometheus-kafka-exporter \
    --set kafkaServer[0]=kafka-broker-0.kafka-headless:9092 \
    --set kafkaServer[1]=kafka-broker-1.kafka-headless:9092

Step 3 — ServiceMonitors:

  # Broker metrics (JMX Exporter on port 9404)
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: kafka-broker
    labels:
      release: prometheus
  spec:
    selector:
      matchLabels:
        app: kafka
    endpoints:
    - port: metrics
      interval: 30s
      path: /metrics

  # Consumer lag (kafka-exporter on port 9308)
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: kafka-consumer-lag
    labels:
      release: prometheus
  spec:
    selector:
      matchLabels:
        app: kafka-exporter
    endpoints:
    - port: metrics
      interval: 30s

Step 4 — Key metrics to monitor:

  Broker health:
    kafka_server_replicamanager_underreplicatedpartitions > 0  → ALERT
    kafka_controller_activecontrollercount != 1               → ALERT
    kafka_server_replicamanager_isrshrinks_total              → ISR shrinks

  Consumer lag:
    kafka_consumergroup_lag > 10000   → consumer falling behind
    kafka_consumergroup_lag_sum       → total lag across all partitions

  Throughput:
    kafka_server_brokertopicmetrics_messagesin_total
    kafka_server_brokertopicmetrics_bytesin_total
    kafka_server_brokertopicmetrics_bytesout_total

Step 5 — PrometheusRule alerts:

  groups:
  - name: kafka_alerts
    rules:
    - alert: KafkaUnderReplicatedPartitions
      expr: kafka_server_replicamanager_underreplicatedpartitions > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Kafka has under-replicated partitions on {{ $labels.instance }}"

    - alert: KafkaConsumerGroupLag
      expr: kafka_consumergroup_lag_sum{consumergroup="payment-processor"} > 50000
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Consumer group {{ $labels.consumergroup }} lag is {{ $value }}"
```

💡 **Interview tip:** Under-replicated partitions is the most critical Kafka alert — it means data is not being replicated and a broker failure would cause data loss. Consumer group lag is the most operationally important metric — it tells you if consumers are keeping up with producers. Mention that kafka-exporter and JMX Exporter serve different purposes: JMX covers broker internals, kafka-exporter covers the consumer group API (which requires connecting to Kafka as a client). The Grafana dashboard ID 7589 (Kafka Exporter Overview) is a ready-made dashboard most teams use.

---

### Q107 — Kubernetes | Conceptual | Advanced

> Explain **`nodeAffinity`**, **`podAffinity`**, **`podAntiAffinity`**, and **required vs preferred** scheduling rules. Real-world `podAntiAffinity` example.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

```
nodeAffinity — attract pod to specific NODES:
  Based on node labels
  "Schedule this pod on nodes in zone us-east-1a"
  "Schedule on nodes with label disktype=ssd"

podAffinity — attract pod to nodes WHERE specific pods already run:
  Based on other pods' labels
  "Schedule this cache pod on the same node as the app pod it serves"
  Reduces network latency between co-located pods

podAntiAffinity — repel pod from nodes WHERE specific pods already run:
  "Do NOT schedule two replicas of this pod on the same node"
  "Do NOT schedule two replicas on the same AZ"
  Ensures high availability by spreading pods

required vs preferred:

  requiredDuringSchedulingIgnoredDuringExecution:
    → HARD rule — pod will NOT be scheduled if not satisfied
    → Pod stays Pending if no suitable node exists
    → "IgnoredDuringExecution" = if node labels change AFTER scheduling,
       pod is NOT evicted (only enforced at scheduling time)

  preferredDuringSchedulingIgnoredDuringExecution:
    → SOFT rule — scheduler TRIES to satisfy but will schedule anyway
    → Pod always gets scheduled (worst case: ignores preference)
    → Has a weight (1-100) for when multiple preferences exist

Real-world podAntiAffinity — spread Deployment replicas across nodes:

spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          # HARD: never 2 replicas on same node
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: payment-api       # match other replicas of THIS app
            topologyKey: kubernetes.io/hostname   # "same node" definition

          # SOFT: try to spread across AZs too
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: payment-api
              topologyKey: topology.kubernetes.io/zone  # "same AZ" definition

Result:
  3 replicas → guaranteed on 3 different nodes
  Soft preference → spread across different AZs
  If only 2 AZs available → still schedules (soft rule)
  If only 2 nodes available → 3rd replica stays Pending (hard rule)

topologyKey options:
  kubernetes.io/hostname           → spread across nodes
  topology.kubernetes.io/zone      → spread across AZs
  topology.kubernetes.io/region    → spread across regions
```

💡 **Interview tip:** `podAntiAffinity` with `topologyKey: kubernetes.io/hostname` is something every production deployment should have for stateless services with multiple replicas — it prevents two replicas landing on the same node, so a node failure only takes one replica down. The `IgnoredDuringExecution` suffix is a common interview question — it means the rule is only enforced at scheduling time, not continuously. A future `RequiredDuringExecution` mode would evict pods when rules are no longer satisfied, but this doesn't exist yet in stable Kubernetes.

---

### Q108 — ArgoCD | Troubleshooting | Advanced

> ArgoCD shows all apps as **`Unknown`** health, UI is very slow, application controller is high CPU/memory.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD`

```
Step 1 — Check application controller logs:
  kubectl logs -n argocd deployment/argocd-application-controller \
    --tail=100 | grep -E "ERROR|WARN|slow|timeout"

  Common errors:
  → "Failed to get cluster info" → can't reach target cluster API server
  → "context deadline exceeded"  → reconciliation timing out
  → "rate limit exceeded"        → too many API server requests

Step 2 — Check resource usage:
  kubectl top pods -n argocd
  → argocd-application-controller using > 2 CPU or > 2Gi memory
  → This controller reconciles ALL applications — single-threaded by default

Step 3 — Check number of applications and resources:
  argocd app list | wc -l         → how many apps total?
  argocd app get <app> --hard-refresh  → force refresh one app

  If 100+ apps or 10,000+ K8s resources being tracked:
  → Controller overwhelmed — needs sharding

Step 4 — Enable application controller sharding:
  # argocd-cm ConfigMap:
  data:
    application.instanceLabelKey: argocd.argoproj.io/app-name

  # argocd-application-controller StatefulSet:
  # Scale replicas and add shard count:
  kubectl patch statefulset argocd-application-controller \
    -n argocd \
    --type json \
    -p='[{"op":"replace","path":"/spec/replicas","value":3}]'

  env:
  - name: ARGOCD_CONTROLLER_REPLICAS
    value: "3"
  # Each replica handles 1/3 of applications

Step 5 — Check cluster connectivity:
  kubectl get secrets -n argocd | grep cluster
  argocd cluster list
  → Is any cluster showing as "Unknown"?
  → Check: can ArgoCD reach the target cluster API server?
  → Network policy blocking argocd-application-controller → target cluster

Step 6 — Tune reconciliation settings:
  # argocd-cm ConfigMap:
  data:
    timeout.reconciliation: 180s       # default: 180s, reduce for faster sync
    resource.customizations: |
      # Add custom health checks for CRDs

  # Reduce sync frequency for stable apps:
  argocd app set my-app --sync-policy automated
  # Or set longer refresh interval per app

Step 7 — Resource exclusions (reduce tracked resources):
  # argocd-cm:
  data:
    resource.exclusions: |
      - apiGroups:
        - "*"
        kinds:
        - "Event"          # Events are high-volume, not useful to track
        clusters:
        - "*"
```

💡 **Interview tip:** The most common cause of ArgoCD controller overload is a large number of applications or resources without sharding enabled. ArgoCD application controller is single-threaded by default — it reconciles one app at a time. Sharding splits the application set across multiple controller replicas. Also mention excluding Kubernetes Events from resource tracking — Events are extremely high-volume and ArgoCD tracking them wastes significant memory without adding value. The `Unknown` health status often means ArgoCD can't reach the target cluster, not necessarily an application problem.

---

### Q109 — Linux / Bash | Scenario-Based | Advanced

> Write a Bash script parsing **Nginx access logs** — top 10 IPs, top 10 endpoints, status distribution, average response time per endpoint.

📁 **Reference:** `nawab312/DSA` → `Linux`

```bash
#!/bin/bash

LOG_FILE="${1:-/var/log/nginx/access.log}"

if [ ! -f "$LOG_FILE" ]; then
  echo "Error: log file not found: $LOG_FILE"
  exit 1
fi

echo "========================================"
echo "  Nginx Access Log Report"
echo "  File: $LOG_FILE"
echo "  Lines: $(wc -l < "$LOG_FILE")"
echo "========================================"

echo ""
echo "── Top 10 IPs by Request Count ─────────"
awk '{print $1}' "$LOG_FILE" \
  | sort | uniq -c | sort -rn | head -10 \
  | awk '{printf "  %-8s  %s\n", $1, $2}'

echo ""
echo "── Top 10 Endpoints by Request Count ───"
awk '{print $7}' "$LOG_FILE" \
  | sort | uniq -c | sort -rn | head -10 \
  | awk '{printf "  %-8s  %s\n", $1, $2}'

echo ""
echo "── HTTP Status Code Distribution ───────"
awk '{print $9}' "$LOG_FILE" \
  | sort | uniq -c | sort -rn \
  | awk '{printf "  HTTP %-6s  %s requests\n", $2, $1}'

echo ""
echo "── Average Response Time per Endpoint ──"
# Nginx log format: $remote_addr - $remote_user [$time_local]
# "$request" $status $body_bytes_sent "$http_referer"
# "$http_user_agent" $request_time
# Field $7 = endpoint, $NF = response time (last field)
awk '{
  endpoint = $7
  time = $NF
  sum[endpoint] += time
  count[endpoint]++
}
END {
  for (ep in sum) {
    avg = sum[ep] / count[ep]
    printf "%.4f  %s  (%d requests)\n", avg, ep, count[ep]
  }
}' "$LOG_FILE" \
  | sort -rn | head -15 \
  | awk '{printf "  %-8s  %-40s  %s\n", $1"s", $2, $3" "$4}'

echo ""
echo "========================================"
```

**Example output:**
```
========================================
  Nginx Access Log Report
  File: /var/log/nginx/access.log
  Lines: 142831
========================================

── Top 10 IPs by Request Count ─────────
  8421      192.168.1.105
  6234      10.0.0.52

── HTTP Status Code Distribution ───────
  HTTP 200      98432 requests
  HTTP 404      3821 requests
  HTTP 500      241 requests

── Average Response Time per Endpoint ──
  2.4120s   /api/reports/generate        (142 requests)
  0.8234s   /api/users/search            (1823 requests)
```

💡 **Interview tip:** `awk` is the tool for structured log parsing — it splits each line by whitespace into fields ($1, $2, ... $NF). Knowing your Nginx log format is key — field positions depend on the `log_format` directive. Mention that for production at scale, you would use this script for quick ad-hoc analysis but use the ELK stack or CloudWatch Logs Insights for ongoing monitoring. `sort | uniq -c | sort -rn` is the classic pattern for frequency counting in Bash — memorise it.

---

### Q110 — AWS | Scenario-Based | Advanced

> Implement a **Zero Trust security model** on AWS — never trust any request by default, even within the VPC.

📁 **Reference:** `nawab312/AWS` — IAM, mTLS, App Mesh, CloudTrail, VPC

```
Zero Trust principles:
  → Never trust, always verify (even internal VPC traffic)
  → Least privilege access for every service
  → Assume breach — log and monitor everything
  → Verify explicitly — every request authenticated + authorized

Layer 1 — Network: eliminate implicit VPC trust:

  Traditional: anything in VPC trusted each other
  Zero Trust: every connection requires authentication

  → Remove broad security group rules (no 0.0.0.0/0 within VPC)
  → Service-specific SG: payment-sg allows ONLY from api-sg on port 8080
  → VPC flow logs: log all rejected AND accepted traffic

Layer 2 — Service Identity: mTLS for service-to-service:

  Every service has a TLS certificate (X.509)
  Both sides present certs — mutual TLS (mTLS)
  "I am the payment service and I can prove it"

  Options:
  → AWS App Mesh + Envoy sidecar (auto-injects mTLS)
  → AWS Private Certificate Authority (issues internal certs)
  → Istio service mesh on EKS

  aws appmesh create-virtual-service \
    --mesh-name production \
    --virtual-service-name payment.local \
    --spec '{"provider":{"virtualRouter":{"virtualRouterName":"payment-router"}}}'

Layer 3 — Authorization: IRSA for AWS API calls:

  Every pod has a ServiceAccount → IAM role (IRSA)
  payment-service SA → can only call S3 bucket payment-receipts
  user-service SA    → can only call DynamoDB users table
  No EC2 instance profile with broad permissions

Layer 4 — Authentication: fine-grained API authorization:

  API Gateway + Lambda Authorizer:
    Every API call → Lambda validates JWT token
    Token contains: service identity, permissions
    Lambda returns: IAM policy (allow/deny specific APIs)

  OR: Amazon Verified Permissions (Cedar policy language)
    Centralized authorization decisions
    "Can payment-service call user-service /users/{id}?"

Layer 5 — Audit: log everything:

  CloudTrail: all AWS API calls with caller identity
  VPC Flow Logs: all network connections (accepted + rejected)
  API Gateway access logs: every API call with latency
  AWS Config: all resource configuration changes
  GuardDuty: anomaly detection (unusual API calls, crypto mining)

  Central security lake: all logs → S3 → Athena queries

Layer 6 — Secrets: never static credentials:

  No hardcoded credentials anywhere
  Secrets Manager: rotate DB passwords automatically every 30 days
  IRSA: no AWS access keys (token exchange via OIDC)
  SSM Parameter Store: encrypted config injection at startup
```

💡 **Interview tip:** Zero Trust is a philosophy, not a single product — the answer should cover all layers: network (SGs), identity (mTLS/IRSA), authorization (IAM + API Gateway authorizers), and audit (CloudTrail + GuardDuty). The key phrase is "verify explicitly, use least privilege, assume breach." AWS App Mesh or Istio is the standard way to implement mTLS between microservices without changing application code — the service mesh sidecar handles certificate management and mutual authentication transparently.

---

### Q111 — Kubernetes | Troubleshooting | Advanced

> `kubectl apply -f deployment.yaml` **hangs indefinitely** with no error. What are the possible causes?

📁 **Reference:** `nawab312/Kubernetes` → `14_TROUBLESHOOTING.md`

```
Step 1 — Check if API server is reachable:
  kubectl cluster-info                  → basic connectivity
  kubectl get nodes --request-timeout=5s  → quick timeout test
  → Hangs here → API server unreachable (network, cert issue)

Step 2 — Check for admission webhook timeout:
  # Most common cause of kubectl apply hanging
  kubectl get mutatingwebhookconfigurations
  kubectl get validatingwebhookconfigurations

  → A webhook with failurePolicy: Fail is unreachable
  → API server waits for webhook response up to timeoutSeconds
  → Default webhook timeout: 10-30 seconds
  → If webhook pod is down + failurePolicy=Fail → hangs then fails

  Check webhook endpoint:
  kubectl get endpoints <webhook-service> -n <webhook-namespace>
  → Empty endpoints → webhook pod is down → that is your problem

  Quick fix (emergency):
  kubectl delete mutatingwebhookconfiguration <name>    # removes webhook
  kubectl delete validatingwebhookconfiguration <name>  # removes webhook
  # WARNING: removes ALL policy enforcement for that webhook

Step 3 — Check if namespace is terminating:
  kubectl get namespace <namespace>
  → STATUS: Terminating → namespace stuck in deletion
  → Resources in terminating namespace cannot be created
  → Apply hangs waiting for namespace

  Fix stuck namespace:
  kubectl get namespace <ns> -o json | \
    jq '.spec.finalizers = []' | \
    kubectl replace --raw /api/v1/namespaces/<ns>/finalize -f -

Step 4 — Check etcd health:
  # On control plane node:
  etcdctl endpoint health
  → etcd unresponsive → API server can't persist → hangs

Step 5 — Check API server audit logs:
  # Look for the request that is stuck:
  grep "deployment.yaml-resource" /var/log/kubernetes/audit.log
  → Find the requestURI for your apply
  → Check if it reached ResponseComplete stage
  → If only RequestReceived stage → stuck in admission

Step 6 — Check for resource quota:
  kubectl describe quota -n <namespace>
  → ResourceQuota exceeded → create rejected but kubectl waits (bug in some versions)

Step 7 — Run with verbose flag:
  kubectl apply -f deployment.yaml -v=8
  → Shows each HTTP request and response
  → Identifies exactly which API call is hanging
  → "POST .../deployments" → "waiting for response" → webhook issue
```

💡 **Interview tip:** The most common real cause of `kubectl apply` hanging is a broken admission webhook — especially Istio's mutating webhook or OPA/Gatekeeper's validating webhook. When their pods are down and `failurePolicy: Fail`, every apply to matching resources hangs until the 30-second webhook timeout. The immediate fix is to scale up the webhook deployment. Always run `kubectl apply -v=8` first when debugging — it shows exactly which API call is stuck. Mention that this is why webhook HA (at least 2 replicas) and a `PodDisruptionBudget` are critical in production.

---

### Q112 — Grafana | Conceptual | Advanced

> Explain **Grafana Loki** vs **Elasticsearch** for logs. What is **LogQL**? Give LogQL query examples.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana`

```
Elasticsearch for logs (ELK stack):
  → Full-text indexing of every log field
  → Every word in every log line is indexed
  → Fast full-text search across any field
  → High storage cost: index = 2-3× raw log size
  → Complex to operate (cluster management, shard management, ILM)

Grafana Loki for logs:
  → Indexes ONLY labels (not log content)
  → Labels: {app="payment", env="prod", pod="payment-xyz"}
  → Log content stored compressed, not indexed
  → Low cost: ~6× cheaper than Elasticsearch for same data
  → Simple to operate (like Prometheus but for logs)
  → Tight Grafana integration (same UI for metrics + logs)

Key difference:
  Elasticsearch: "find all logs containing 'NullPointerException'"
    → Fast: full-text index
  Loki: "find all logs from app=payment containing 'NullPointerException'"
    → Must specify labels first, then filter content
    → Slower for content search without labels, but much cheaper

When to use Loki:
  → Already using Prometheus/Grafana (unified observability)
  → Cost is a concern (6× cheaper storage)
  → Kubernetes workloads (natural label alignment with pod labels)
  → Don't need complex full-text search

When to use Elasticsearch:
  → Need full-text search without label filtering first
  → Complex log analytics and aggregations
  → Already have ELK stack
  → Need KQL (Kibana Query Language) features

LogQL — Loki's query language:

  Basic log stream selector:
  {app="payment-service", env="prod"}
  → Returns all logs from payment-service in prod

  Filter by content (pipe):
  {app="payment-service"} |= "ERROR"
  → Only lines containing "ERROR"

  Regex filter:
  {app="payment-service"} |~ "timeout|connection refused"
  → Lines matching regex pattern

  Error rate over time (metric query):
  rate({app="payment-service"} |= "ERROR" [5m])
  → Error log lines per second over 5 minute window

  Extract field from log line:
  {app="payment-service"}
    | logfmt                           # parse logfmt format
    | line_format "{{.level}} {{.msg}}"  # reformat output

  Pattern extraction:
  {app="nginx"}
    | pattern `<ip> - - [<_>] "<method> <path> <_>" <status> <_>`
    | status >= 500
  → Extract fields by pattern, then filter by extracted field
```

💡 **Interview tip:** The key Loki vs Elasticsearch trade-off is: Loki is cheap and simple but requires label-first querying; Elasticsearch is powerful but expensive and complex. Loki follows the Prometheus model — labels are the primary filter, content is secondary. In Kubernetes, this works well because pod labels (app, namespace, env) are natural log labels via Promtail or the Grafana Agent. Mention that Loki 3.x has added bloom filters which dramatically improve content search performance without full indexing.

---

### Q113 — Terraform | Troubleshooting | Advanced

> **Terraform plan takes 45 minutes** on 500+ resources. How do you optimize it?

📁 **Reference:** `nawab312/Terraform` — performance optimization

```
Why Terraform plan is slow:

  For each resource in state:
    1. Read current state from state file
    2. Call AWS API: describe/get the real resource
    3. Compare API response with state
    4. Show diff if different

  500 resources × 1 API call each = 500 AWS API calls
  Sequential by default = 500 × 200ms avg = 100 seconds minimum
  Plus: API throttling adds retries = much longer

Step 1 — Increase parallelism:
  # Default: 10 concurrent operations
  terraform plan -parallelism=50
  → 50 resources refreshed simultaneously instead of 10
  → Can hit AWS API rate limits — tune carefully

  # Set permanently in .terraformrc:
  provider_installation {
    direct {
      include = ["registry.terraform.io/*/*"]
    }
  }

Step 2 — Skip refresh for known-stable resources:
  # Refresh reads current state from AWS on every plan
  # For stable resources (S3 buckets, IAM roles) — skip refresh:
  terraform plan -refresh=false
  → Uses state file values without calling AWS API
  → Much faster but misses out-of-band changes
  → Good for CI/CD when you trust your state is accurate

Step 3 — Target specific resources:
  # Plan only the resources you are changing:
  terraform plan -target=aws_lambda_function.my_function
  terraform plan -target=module.api_service

  → Only refreshes and plans the targeted resource + dependencies
  → 1 resource instead of 500 = seconds not minutes

Step 4 — Split the monolith state:
  # Root cause: all 500 resources in ONE state file
  # Solution: split by service/layer

  Before:
    one big state → 500 resources → 45 minute plan

  After:
    networking/ state  → 50 resources  → 2 min plan
    databases/ state   → 30 resources  → 1 min plan
    services/ state    → 420 resources → split further by service

  Each team/service has their own state — plans are fast and isolated

Step 5 — Use data sources instead of resources for shared infra:
  # Instead of managing VPC in same state as services:
  # Read VPC outputs as data source (no refresh needed):
  data "terraform_remote_state" "networking" {
    backend = "s3"
    config  = { bucket = "tf-state", key = "networking/terraform.tfstate" }
  }
  → No AWS API call for VPC — just reads remote state output

Step 6 — Provider-level optimisations:
  provider "aws" {
    skip_metadata_api_check     = true  # skip EC2 metadata
    skip_region_validation      = true  # skip region validation
    skip_credentials_validation = true  # skip credential check
    # These skip 3 API calls per provider init
  }

Step 7 — Use Terraform Cloud / remote backend with caching:
  → Remote plan runs on Terraform Cloud infrastructure
  → Plan results cached
  → Concurrent plan/apply possible
```

💡 **Interview tip:** The real fix for slow plans is architectural — splitting one large state into multiple smaller states. Optimising `-parallelism` and `-refresh=false` are short-term fixes. Long-term, the rule is: state files should be scoped to a single team or service so plans only touch relevant resources. Also mention `terraform plan -refresh=false` is standard in CI/CD pipelines where the state is trusted — the full refresh is reserved for periodic drift detection runs.

---

### Q114 — Python | Scenario-Based | Advanced

> Write a **Python Kubernetes health checker** — list Deployments, check ready vs desired replicas, Slack alert for unhealthy.

📁 **Reference:** `nawab312/DSA` → `Python`

```python
#!/usr/bin/env python3

import json
import os
import sys
from kubernetes import client, config
import urllib.request
import urllib.error

SLACK_WEBHOOK = os.environ.get("SLACK_WEBHOOK_URL", "")


def load_kube_config():
    try:
        config.load_incluster_config()      # running inside a pod
    except config.ConfigException:
        config.load_kube_config()           # running locally


def get_all_deployments():
    apps_v1 = client.AppsV1Api()
    return apps_v1.list_deployment_for_all_namespaces(watch=False).items


def check_deployment_health(deployment):
    name      = deployment.metadata.name
    namespace = deployment.metadata.namespace
    desired   = deployment.spec.replicas or 0
    ready     = deployment.status.ready_replicas or 0
    available = deployment.status.available_replicas or 0

    healthy = (ready == desired and desired > 0)

    return {
        "name":      name,
        "namespace": namespace,
        "desired":   desired,
        "ready":     ready,
        "available": available,
        "healthy":   healthy,
    }


def send_slack_alert(unhealthy):
    if not SLACK_WEBHOOK:
        print("  WARNING: SLACK_WEBHOOK_URL not set — skipping Slack alert")
        return

    lines = ["*Kubernetes Unhealthy Deployments Alert*\n"]
    for d in unhealthy:
        lines.append(
            f"• `{d['namespace']}/{d['name']}` — "
            f"ready: {d['ready']}/{d['desired']}"
        )

    payload = json.dumps({"text": "\n".join(lines)}).encode("utf-8")
    req = urllib.request.Request(
        SLACK_WEBHOOK,
        data=payload,
        headers={"Content-Type": "application/json"},
        method="POST",
    )

    try:
        with urllib.request.urlopen(req, timeout=10) as resp:
            print(f"  Slack alert sent — HTTP {resp.status}")
    except urllib.error.URLError as e:
        print(f"  ERROR sending Slack alert: {e}")


def main():
    load_kube_config()
    deployments = get_all_deployments()

    healthy_list   = []
    unhealthy_list = []

    for dep in deployments:
        result = check_deployment_health(dep)
        if result["healthy"]:
            healthy_list.append(result)
        else:
            unhealthy_list.append(result)

    print("\n========================================")
    print("  Kubernetes Deployment Health Report")
    print(f"  Total: {len(deployments)}  |  "
          f"Healthy: {len(healthy_list)}  |  "
          f"Unhealthy: {len(unhealthy_list)}")
    print("========================================\n")

    if healthy_list:
        print("HEALTHY Deployments:")
        for d in healthy_list:
            print(f"  OK  {d['namespace']:<20} {d['name']:<40} "
                  f"{d['ready']}/{d['desired']} ready")

    if unhealthy_list:
        print("\nUNHEALTHY Deployments:")
        for d in unhealthy_list:
            print(f"  !!  {d['namespace']:<20} {d['name']:<40} "
                  f"{d['ready']}/{d['desired']} ready")
        print()
        send_slack_alert(unhealthy_list)

    print("\n========================================")
    sys.exit(1 if unhealthy_list else 0)


if __name__ == "__main__":
    main()
```

💡 **Interview tip:** `load_incluster_config()` with fallback to `load_kube_config()` is the production pattern — the script works both locally (for development) and inside a Kubernetes pod (for production CronJob usage). Mention that this script can be deployed as a Kubernetes CronJob running every 5 minutes with a ServiceAccount that has `get/list` permissions on deployments. `sys.exit(1)` on unhealthy deployments makes the script useful in CI pipelines — a non-zero exit code fails the pipeline.

---

### Q115 — AWS | Conceptual | Advanced

> Explain **VPC Peering** vs **Transit Gateway** vs **PrivateLink**. Hub-and-spoke example with Transit Gateway.

📁 **Reference:** `nawab312/AWS` — VPC Peering, Transit Gateway, PrivateLink

```
VPC Peering — direct encrypted tunnel between 2 VPCs:
  → 1-to-1 connection: VPC A ↔ VPC B
  → Routes must be manually added to route tables
  → Non-transitive: A↔B and B↔C does NOT mean A↔C
  → No bandwidth limits (uses AWS backbone)
  → Cost: data transfer charges only (no hourly fee)

  Limitation — N VPCs needs N×(N-1)/2 peerings:
  10 VPCs = 45 peering connections to fully connect
  Each needs route table entries on both sides
  At scale: unmanageable

Transit Gateway — centralized routing hub:
  → All VPCs attach to one TGW
  → TGW routes between all attached VPCs
  → Transitive: A → TGW → C works automatically
  → Cost: $0.05/hour per attachment + $0.02/GB data processed
  → Supports: VPCs + VPNs + Direct Connect + other TGWs (peered)

  Hub-and-spoke with Transit Gateway:

    prod-vpc      ─┐
    staging-vpc   ─┤
    dev-vpc       ─┼──► Transit Gateway ──► shared-services-vpc
    data-vpc      ─┤                         (DNS, monitoring, tools)
    security-vpc  ─┘

    All VPCs can reach shared-services
    prod-vpc isolated from dev-vpc (route table controls)
    One TGW route table per environment tier

  Route table control (isolation):
    prod TGW route table:    allow → shared-services only
    staging TGW route table: allow → shared-services + dev
    → prod cannot reach dev (security isolation)

AWS PrivateLink — expose a service privately without VPC peering:
  → Service provider creates VPC Endpoint Service
  → Consumer creates Interface VPC Endpoint (ENI with private IP)
  → Traffic never leaves AWS backbone
  → No VPC CIDR overlap issues (works even with same IP ranges)
  → Direction: one-way only (consumer → provider)

  Use cases:
  → SaaS vendor exposing API privately to your VPC
  → AWS services: S3 Gateway endpoint, ECR interface endpoint
  → Internal: platform team exposes logging API privately to all teams

When to use each:

  VPC Peering:
    ✓ 2-5 VPCs
    ✓ Cost-sensitive (no hourly fee)
    ✓ Simple, low-traffic connections

  Transit Gateway:
    ✓ 5+ VPCs
    ✓ Need hub-and-spoke topology
    ✓ Multi-region (TGW peering)
    ✓ Need transitive routing

  PrivateLink:
    ✓ Expose a service to many consumers (one-way)
    ✓ CIDR overlap between VPCs
    ✓ SaaS/multi-tenant service exposure
    ✓ Don't want full VPC-to-VPC access
```

💡 **Interview tip:** The non-transitive limitation of VPC Peering is the key reason Transit Gateway exists — at 10+ VPCs, peering becomes unmanageable. Transit Gateway is the standard for enterprise AWS architectures. PrivateLink is fundamentally different — it exposes a single service endpoint, not full VPC connectivity. It solves the problem where you want a vendor or platform team service accessible privately without giving them access to your entire VPC. Mention TGW inter-region peering for global architectures — you can connect TGWs across regions for global hub-and-spoke.

---

### Q116 — Kubernetes | Scenario-Based | Advanced

> Implement **canary deployments** on Kubernetes without a service mesh — route 10% to v2, 90% to v1.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md`, `11_ISTIO_SERVICE_MESH.md`

```
Native Kubernetes canary — using replica ratio:

  How it works:
  Service routes to pods by label selector
  10 pods total: 9 labeled v1, 1 labeled v2 = 10% canary traffic

  Step 1 — Stable deployment (v1):
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: myapp-stable
  spec:
    replicas: 9                    # 90% of traffic
    selector:
      matchLabels:
        app: myapp
        version: v1
    template:
      metadata:
        labels:
          app: myapp               # Service selects on this
          version: v1
      spec:
        containers:
        - image: myapp:v1

  Step 2 — Canary deployment (v2):
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: myapp-canary
  spec:
    replicas: 1                    # 10% of traffic
    selector:
      matchLabels:
        app: myapp
        version: v2
    template:
      metadata:
        labels:
          app: myapp               # Same label — Service routes here too
          version: v2
      spec:
        containers:
        - image: myapp:v2

  Step 3 — Service (selects BOTH deployments):
  apiVersion: v1
  kind: Service
  spec:
    selector:
      app: myapp                   # matches ALL pods with app=myapp
                                   # does NOT filter by version label

  Result: 10 pods total, 1 v2 = ~10% traffic to canary

  Limitations of native canary:
  ✗ Traffic split is approximate (replica-based, not exact percentage)
  ✗ Scaling v1 changes the ratio (9→18 pods = canary drops to 5%)
  ✗ Cannot do header-based routing (all users get random split)
  ✗ No gradual automated progression
  ✗ Cannot do sticky sessions for canary

How Istio improves canary:
  VirtualService: exact weight-based routing (no replica math)
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  spec:
    http:
    - route:
      - destination:
          host: myapp
          subset: v1
        weight: 90
      - destination:
          host: myapp
          subset: v2
        weight: 10
  → Exactly 10% regardless of replica count
  → Header-based: route users with header X-Canary: true to v2 always

How Argo Rollouts improves canary:
  apiVersion: argoproj.io/v1alpha1
  kind: Rollout
  spec:
    strategy:
      canary:
        steps:
        - setWeight: 10    # 10% canary
        - pause: {duration: 10m}
        - setWeight: 30    # 30% canary
        - pause: {}        # manual approval
        - setWeight: 100   # full rollout
        analysis:          # auto-rollback if error rate > 1%
          templates:
          - templateName: success-rate
```

💡 **Interview tip:** Native Kubernetes canary is a valid interview answer — explain the replica-ratio trick and its limitations. The follow-up is always "how would Istio or Argo Rollouts improve this?" Argo Rollouts is the most complete answer — it has native Kubernetes integration, supports both replica-based and Istio-based traffic splitting, and adds automated analysis for rollback. Mention that Argo Rollouts integrates with Prometheus to automatically roll back if error rate or latency exceeds a threshold — this is the production standard.

---

### Q117 — ELK Stack | Troubleshooting | Advanced

> **Logstash pipeline falling behind** — 30-minute log delay, Logstash CPU at 100%. How do you tune it?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack`

```
Step 1 — Identify the bottleneck stage:
  Logstash pipeline: Input → Filter → Output
  CPU at 100% → Filter stage is bottleneck (grok parsing is expensive)
  OR: Output stage slow → Elasticsearch can't keep up → back-pressure

  Check pipeline stats API:
  curl http://localhost:9600/_node/stats/pipelines?pretty
  → events.filtered vs events.out delta shows queue buildup
  → duration_in_millis per plugin shows which is slowest

Step 2 — Tune pipeline workers:
  # logstash.yml or pipeline config:
  pipeline.workers: 8        # default: number of CPU cores
  pipeline.batch.size: 500   # default: 125 events per batch
  pipeline.batch.delay: 50   # default: 50ms wait for batch fill

  # For I/O bound (slow Elasticsearch output):
  pipeline.workers: 16       # more workers to overlap I/O wait
  pipeline.batch.size: 1000  # larger batches = fewer Elasticsearch requests

  # For CPU bound (heavy grok parsing):
  pipeline.workers: 8        # match CPU cores
  pipeline.batch.size: 250   # smaller batches for faster per-event processing

Step 3 — Optimise Grok patterns (most common bottleneck):
  # Grok parses every log line with regex — expensive
  # BAD: complex nested grok:
  filter {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
  }

  # BETTER: dissect for structured logs (10-50× faster than grok):
  filter {
    dissect {
      mapping => { "message" => "%{ip} - %{user} [%{timestamp}] \"%{request}\" %{status} %{bytes}" }
    }
  }
  # Dissect uses simple string splitting, no regex engine

  # If must use grok: tag_on_failure to skip failed patterns quickly:
  grok {
    match => { "message" => "..." }
    tag_on_failure => ["_grokparsefailure"]
  }

Step 4 — Enable persistent queue (prevent data loss on lag):
  # logstash.yml:
  queue.type: persisted        # default: memory (data lost on crash)
  queue.max_bytes: 10gb        # disk queue size
  queue.checkpoint.acks: 1024  # checkpoint every N events
  # Persistent queue acts as buffer — input doesn't block when output is slow

Step 5 — Tune Elasticsearch output:
  output {
    elasticsearch {
      hosts    => ["elasticsearch:9200"]
      workers  => 4        # parallel output workers
      bulk_max_size => 500 # events per bulk request (default: 125)
      flush_size => 500    # trigger bulk at this count
      idle_flush_time => 5 # flush after 5s even if not full
    }
  }

Step 6 — Scale Logstash horizontally:
  # Single Logstash → bottleneck for entire pipeline
  # Multiple Logstash instances with shared input (Kafka/Redis):

  Beats → Kafka → Logstash-1 (partition 0,1)
                → Logstash-2 (partition 2,3)
                → Logstash-3 (partition 4,5)
               → Elasticsearch

  # Kafka acts as buffer + load balancer
  # Each Logstash consumer group partition assignment
```

💡 **Interview tip:** Grok is almost always the bottleneck — it runs a regex engine against every single log line. Switching to `dissect` for structured logs is the single biggest performance win (10-50× improvement). The architectural fix for sustained lag is adding Kafka between Beats and Logstash — it decouples ingestion from processing and allows horizontal scaling of Logstash independently. Mention monitoring `logstash_events_filtered_total` rate in Prometheus/Grafana to detect lag before it becomes a user-visible 30-minute delay.

---

### Q118 — Jenkins | Conceptual | Advanced

> Explain **Blue Ocean vs Classic UI**, **Multibranch Pipeline**, and how to organize **100+ Jenkins jobs**.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins`

```
Jenkins Classic UI:
  → Original Jenkins interface
  → Job-centric view
  → Build history as list
  → Configuration via forms (not code)
  → Powerful but dense and complex

Jenkins Blue Ocean:
  → Modern visual pipeline view
  → Pipeline stages shown as swimlanes
  → Clear pass/fail per stage with timing
  → Easy to see which step in parallel failed
  → Built for Declarative Pipeline (Jenkinsfile)
  → Read-only view for most operations
  → Still requires Classic for deep configuration

When to use Blue Ocean:
  → Viewing pipeline execution and stage results
  → Sharing pipeline status with non-Jenkins users
  → Debugging which specific parallel stage failed

Multibranch Pipeline:
  → Scans a Git repository for branches and PRs
  → Automatically creates a pipeline job for EACH branch
  → Each branch runs its own Jenkinsfile
  → When branch is deleted → Jenkins job automatically deleted
  → PR pipelines: auto-created when PR opened, deleted when closed

  vs Regular Pipeline job:
    Pipeline:            single job, single branch, manually configured
    Multibranch Pipeline: one job per branch, auto-discovered, self-managing

  Configuration:
    Jenkins → New Item → Multibranch Pipeline
    → Branch Sources: GitHub/GitLab
    → Scan interval: every 5 minutes (detects new branches)
    → Jenkinsfile path: Jenkinsfile (default)

Organizing 100+ Jenkins jobs:

  Option 1 — Folders (recommended):
    Jenkins Folders plugin
    tree structure like a filesystem

    Platform/
    ├── Infrastructure/
    │   ├── terraform-networking
    │   ├── terraform-databases
    │   └── terraform-services
    ├── Services/
    │   ├── payment-service (Multibranch)
    │   ├── user-service (Multibranch)
    │   └── auth-service (Multibranch)
    └── Platform-Tools/
        ├── update-base-images
        └── security-scans

    Benefits:
    → RBAC per folder (team A can't see team B's jobs)
    → Credentials scoped to folder
    → Clean URL structure: /job/Platform/job/Services/job/payment

  Option 2 — Views (tags/filter):
    Create views that filter jobs by name pattern
    "Production" view: jobs matching *-prod*
    "Failing" view: jobs with current build failing
    → Views don't scope permissions (just filters)
    → Less powerful than Folders for large teams

  Option 3 — Jenkins Organization Folders:
    GitHub Organisation plugin
    Scans entire GitHub Org
    Auto-creates Multibranch Pipeline per repo
    100 repos → 100 Multibranch Pipelines automatically
```

💡 **Interview tip:** Multibranch Pipeline is the standard for modern Jenkins — it eliminates the need to manually create jobs for feature branches and PR builds. Combined with GitHub webhooks, a new PR triggers a pipeline in seconds automatically. For organizing 100+ jobs, Folders with RBAC is the only scalable answer — flat job lists are unusable at scale. Mention that at 100+ jobs you should also consider migrating to GitHub Actions or GitLab CI where pipelines are defined in the repo and require no Jenkins job management at all.

---

### Q119 — Linux / Bash | Conceptual | Advanced

> Explain **Linux namespaces** and **cgroups**. List the **7 namespaces** and explain how Docker uses them.

📁 **Reference:** `nawab312/DSA` → `Linux`, `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md`

```
Containers = Linux processes with namespaces + cgroups applied.
No magic. No VM. Just Linux kernel features.

NAMESPACES — provide isolation (what the process can SEE):

  1. PID namespace:
     Process sees its own PID tree starting from PID 1
     Cannot see host processes or other container processes
     docker run → container process is PID 1 inside its namespace
     but host sees it as PID 47382

  2. Network namespace:
     Own network interfaces, IP addresses, routing tables, iptables rules
     Container gets eth0 (connected via veth pair to host)
     Container cannot see host's physical NICs

  3. Mount (mnt) namespace:
     Own filesystem view
     Container sees / as its root (from the container image)
     Cannot see host filesystem (unless explicitly volume-mounted)

  4. UTS namespace:
     Own hostname and domain name
     Container can set its hostname without affecting host
     docker run → container hostname = container ID by default

  5. IPC namespace:
     Own System V IPC objects, POSIX message queues
     Containers cannot see each other's shared memory segments

  6. User namespace:
     Map container UIDs to different host UIDs
     Container root (UID 0) → host non-privileged user (UID 65534)
     Enables rootless containers (user namespace remapping)
     NOT enabled by default in Docker (security vs compatibility tradeoff)

  7. Cgroup namespace (added Linux 4.6):
     Container has its own view of cgroup hierarchy
     Cannot see host's cgroup structure

CGROUPS — enforce resource limits (what the process can USE):

  CPU cgroup:
    cpu.shares: relative CPU weight (1024 = default, 512 = half)
    cpu.cfs_quota_us: hard limit (100000 = 1 full CPU)
    docker run --cpus=0.5 → sets cfs_quota_us = 50000

  Memory cgroup:
    memory.limit_in_bytes: hard memory limit
    docker run --memory=512m → limit_in_bytes = 536870912
    Exceeding limit → OOM killer kills the process (container exits)

  Block I/O cgroup:
    blkio.weight: relative I/O weight
    blkio.throttle.read_bps_device: max read bytes/sec

How docker run uses them:
  docker run --cpus=0.5 --memory=512m nginx

  1. Create namespaces (pid, net, mnt, uts, ipc, cgroup)
  2. Set up cgroups (cpu quota = 0.5, memory limit = 512m)
  3. Set up veth pair (container eth0 ↔ host docker0 bridge)
  4. Mount container image layers (overlay filesystem)
  5. chroot into container rootfs
  6. Start process (nginx) in the new namespace context

  Result: nginx sees isolated PID tree, own network, own filesystem
          host kernel enforces: max 0.5 CPU and 512m memory
```

💡 **Interview tip:** "Containers are just processes" is the key statement — they use Linux kernel features available since the 2.6/3.x era. No hypervisor, no separate kernel. The practical implication: a container on Linux and the host share the same kernel version — a kernel vulnerability affects both. This is why Kata Containers (VM-based) or gVisor exist for multi-tenant security requirements. The 7 namespaces list is a common interview question — PID, Network, Mount, UTS, IPC, User, Cgroup.

---

### Q120 — AWS | Scenario-Based | Advanced

> Design a **multi-region active-active application** — nearest region routing, 60-second failover, data sync, split-brain handling.

📁 **Reference:** `nawab312/AWS` — Route53, Global Accelerator, DynamoDB Global Tables

```
Architecture overview:

  Users globally → AWS Global Accelerator → nearest region
  us-east-1 (primary) ↔ eu-west-1 (secondary) — both active

Layer 1 — Global Traffic Routing:

  Option A: Route53 Latency-Based Routing:
    → Routes users to region with lowest latency
    → DNS TTL = 60 seconds (fast failover)
    → Health checks on ALBs → unhealthy → failover in <60s

  Option B: AWS Global Accelerator (better for <60s failover):
    → Static anycast IPs (no DNS propagation delay)
    → Continuously health checks all endpoints
    → Failover in 30 seconds (not DNS TTL dependent)
    → Routes to healthy endpoint automatically

  Global Accelerator is the better answer for 60s RTO requirement
  DNS TTL caching can cause >60s failover in practice

Layer 2 — Data Synchronization:

  DynamoDB Global Tables:
    → Active-active multi-region replication
    → Write to any region → replicates to all others in <1 second
    → Each region has a full copy of all data
    → Last-writer-wins conflict resolution

  RDS Aurora Global Database:
    → One primary region (writes) + up to 5 read regions
    → NOT active-active for writes — primary region only
    → Failover: promote secondary region to primary (~60 seconds)
    → Replication lag: typically <1 second

  S3 Cross-Region Replication:
    → Async replication (~15 minute SLA)
    → Good for static assets, backups, logs
    → NOT suitable for real-time data

Layer 3 — Application Layer:

  Each region runs full stack:
    Global Accelerator → ALB → EKS cluster → services → DynamoDB

  Stateless services: no issue (any region handles any request)
  Stateful operations: route to owning region via consistent hashing
    → User sessions: use DynamoDB (replicated, consistent)
    → File uploads: write to S3 in local region, replicate async

Layer 4 — Split-brain handling:

  Split-brain: two regions lose connectivity but both think they are primary

  DynamoDB Global Tables: last-writer-wins (eventual consistency)
    → Application must handle: read-your-own-writes may not work across regions
    → Use conditional writes for critical operations

  Aurora Global Database: only ONE primary → no write conflict
    → But: secondary region goes read-only during network partition
    → After partition heals: data reconciled automatically

  Application-level:
    → Idempotency keys: operations safe to retry from any region
    → Vector clocks or CRDTs: for operations requiring merge semantics
    → Circuit breaker: stop cross-region calls during partition

Failover runbook (60-second target):
  T+0:  Region failure detected by Global Accelerator health check
  T+10: Global Accelerator stops routing to failed region
  T+30: All user traffic now hitting healthy region only
  T+45: DynamoDB Global Tables: healthy region already has all data
  T+60: Full recovery — all requests served from healthy region
```

💡 **Interview tip:** Global Accelerator vs Route53 for failover speed is a key distinction — Route53 DNS TTL means clients can cache the old record for up to 60 seconds after failover, while Global Accelerator uses anycast IPs so failover is immediate from the network layer. For data, DynamoDB Global Tables is the cleanest active-active solution — mention the last-writer-wins conflict model and its implications. Aurora Global Database is active-passive for writes — be clear about this distinction, it's a common interview confusion.

---

### Q121 — Kubernetes | Conceptual | Advanced

> Explain **ConfigMap vs Secret** — differences, limitations, security problems, and better production alternatives.

📁 **Reference:** `nawab312/Kubernetes` → `05_CONFIGURATION_SECRETS.md`

```
ConfigMap:
  → Non-sensitive configuration data
  → Stored in etcd as plain text
  → Used for: app config, feature flags, environment settings
  → Max size: 1MB per ConfigMap

Secret:
  → Sensitive data (passwords, tokens, TLS certs)
  → Stored in etcd as base64-encoded (NOT encrypted by default)
  → Types: Opaque, kubernetes.io/tls, kubernetes.io/dockerconfigjson
  → Max size: 1MB per Secret

Base64 is NOT encryption:
  echo "cGFzc3dvcmQxMjM=" | base64 -d
  → password123
  Anyone with etcd access can decode all Secrets instantly

Security problems with Kubernetes Secrets:

  Problem 1: etcd stores secrets unencrypted by default
    → etcd backup = all secrets exposed
    → etcd access = all secrets readable
    Fix: enable EncryptionConfiguration (AES or KMS provider)

  Problem 2: Secrets stored in Git (GitOps)
    → You need secret values in the cluster
    → GitOps means everything in Git
    → Can't store raw secrets in Git (base64 is not protection)

  Problem 3: No secret rotation
    → App restarts needed to pick up rotated secrets
    → No audit trail of who read which secret

  Problem 4: RBAC on Secrets is all-or-nothing
    → get secrets = read ALL secrets in namespace
    → No per-secret access control in standard RBAC

Better alternatives:

  Option 1 — Sealed Secrets (GitOps friendly):
    kubeseal CLI encrypts Secret using cluster's public key
    Encrypted SealedSecret stored in Git (safe)
    Controller in cluster decrypts → creates real Secret
    Only the cluster can decrypt (private key stays in cluster)

  Option 2 — External Secrets Operator (ESO):
    Reads secrets from external stores: AWS Secrets Manager, Vault, GCP
    Creates Kubernetes Secrets automatically in the cluster
    Secret values never stored in Git
    ESO syncs on schedule → supports rotation

    apiVersion: external-secrets.io/v1beta1
    kind: ExternalSecret
    spec:
      refreshInterval: 1h
      secretStoreRef:
        name: aws-secrets-manager
        kind: SecretStore
      target:
        name: my-app-secret
      data:
      - secretKey: DB_PASSWORD
        remoteRef:
          key: prod/myapp/db-password

  Option 3 — HashiCorp Vault + Vault Agent Sidecar:
    Vault stores secrets centrally
    Vault Agent sidecar injects secrets as files at pod startup
    Dynamic secrets: Vault generates DB credentials per pod
    Full audit trail: who accessed which secret when

  Option 4 — IRSA + AWS Secrets Manager directly:
    App uses AWS SDK to read secrets from Secrets Manager
    No Kubernetes Secret created at all
    App has IAM role (IRSA) allowing GetSecretValue
    Secrets Manager handles rotation natively

Enabling etcd encryption at rest (minimum for production):
  EncryptionConfiguration:
    resources:
    - resources: ["secrets"]
      providers:
      - aescbc:
          keys:
          - name: key1
            secret: <32-byte-base64-key>
      - identity: {}   # fallback for unencrypted existing secrets
```

💡 **Interview tip:** The most important thing to say: Kubernetes Secrets are NOT secure by default — base64 is encoding, not encryption. For production, always recommend External Secrets Operator with AWS Secrets Manager or Vault — this keeps secret values out of the cluster entirely and enables proper rotation and audit trails. Sealed Secrets is a good answer for teams that want GitOps but aren't ready for a full external secret store. Mention EncryptionConfiguration as the minimum baseline — at least encrypt secrets at rest in etcd.

---

### Q122 — GitHub Actions | Conceptual | Advanced

> Explain **secrets vs vars vs environments** in GitHub Actions. Environment protection rules. Repo vs environment secrets. Write a staging vs prod workflow.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions`

```
secrets — encrypted, write-only values:
  → Encrypted at rest by GitHub
  → Never appear in logs (masked as ***)
  → Cannot be read back after setting (write-only)
  → Access: ${{ secrets.MY_SECRET }}
  → Used for: passwords, API keys, tokens, TLS certs

vars (Variables) — plaintext configuration values:
  → NOT encrypted — visible in UI and logs
  → Can be read and updated easily
  → Access: ${{ vars.MY_VAR }}
  → Used for: non-sensitive config (region, URLs, feature flags)

environments — deployment target with protection rules:
  → Named: staging, production
  → Can have own secrets/vars that override repo-level
  → Protection rules: required reviewers, wait timer, branch filter
  → Creates audit trail: who deployed what to where

Protection rules for production environment:
  Required reviewers: 2 approvers must review before job runs
  Wait timer: 10 minute delay before deployment proceeds
  Deployment branches: only "main" branch can deploy to production
  → Combined: prevents accidental/unauthorized production deployments

Repo secrets vs Environment secrets:
  Repository secrets:
    → Available to all workflows in the repo
    → All environments can use them
    → Good for: shared credentials (Docker Hub, NPM token)

  Environment secrets:
    → Only available when job specifies that environment
    → Override repo-level secret of same name
    → Good for: prod uses different AWS account than staging

Complete staging vs production workflow:

name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.company.com     # shown in deployment UI
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to staging
        env:
          AWS_ACCOUNT: ${{ vars.AWS_ACCOUNT_ID }}       # staging account
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}       # staging DB password
        run: |
          aws ecs update-service \
            --cluster staging \
            --service myapp \
            --force-new-deployment

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging           # only after staging succeeds
    environment:
      name: production              # triggers protection rules
      url: https://app.company.com
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        env:
          AWS_ACCOUNT: ${{ vars.AWS_ACCOUNT_ID }}       # prod account (different var)
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}       # prod DB password (different secret)
        run: |
          aws ecs update-service \
            --cluster production \
            --service myapp \
            --force-new-deployment

# Result:
# staging:    deploys automatically on push to main
# production: requires 2 reviewers to approve (from environment protection)
#             + 10-minute wait timer
#             + only runs after staging passes
```

💡 **Interview tip:** Environment protection rules are the key security feature — without them, anyone who can push to main can deploy to production. Required reviewers + branch protection on main = production deployments require a PR review AND a deployment approval. Mention that environment variables and secrets are scoped — the same secret name `DB_PASSWORD` can have different values for staging and production environments, and GitHub automatically injects the right value based on which environment the job specifies.

---

### Q123 — Terraform | Scenario-Based | Advanced

> Provision **ephemeral environments per Pull Request** — each PR gets its own VPC/ECS/RDS, destroyed when PR closes.

📁 **Reference:** `nawab312/Terraform`, `nawab312/CI_CD` → `GithubActions`

```
Architecture:
  PR opened  → GitHub Actions → terraform apply  → isolated environment
  PR updated → GitHub Actions → terraform apply  → update environment
  PR closed  → GitHub Actions → terraform destroy → clean up everything

Step 1 — Workspace per PR:
  # Each PR gets its own Terraform workspace = own state file
  # Workspace name = pr-<PR_NUMBER>

  # In GitHub Actions workflow:
  - name: Create/select workspace
    run: |
      terraform workspace new pr-${{ github.event.number }} 2>/dev/null || \
      terraform workspace select pr-${{ github.event.number }}

Step 2 — PR-specific variables:
  # Environment name derived from PR number
  variable "environment" {
    type = string  # "pr-123"
  }

  # All resources namespaced to avoid conflicts:
  resource "aws_ecs_cluster" "this" {
    name = "myapp-${var.environment}"  # myapp-pr-123
  }

  resource "aws_db_instance" "this" {
    identifier = "myapp-${var.environment}"  # myapp-pr-123
    instance_class = "db.t3.micro"           # cheap for PR envs
    skip_final_snapshot = true               # must be true for destroy
  }

Step 3 — Complete GitHub Actions workflow:

name: Ephemeral Environment

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

env:
  TF_VAR_environment: pr-${{ github.event.number }}

jobs:
  deploy:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Terraform init
        run: terraform init
        env:
          TF_BACKEND_KEY: pr-${{ github.event.number }}/terraform.tfstate

      - name: Terraform workspace
        run: |
          terraform workspace new pr-${{ github.event.number }} 2>/dev/null || \
          terraform workspace select pr-${{ github.event.number }}

      - name: Terraform apply
        run: terraform apply -auto-approve
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Comment PR with environment URL
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              body: `Environment ready: https://pr-${{ github.event.number }}.staging.company.com`
            })

  destroy:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Terraform destroy
        run: |
          terraform workspace select pr-${{ github.event.number }}
          terraform destroy -auto-approve
          terraform workspace select default
          terraform workspace delete pr-${{ github.event.number }}

Challenges and solutions:

  Challenge 1: RDS takes 10 minutes to create
    Solution: use Aurora Serverless v2 (faster spin-up)
    OR: share one RDS, create per-PR database schema only

  Challenge 2: Cost of 20 open PRs × full stack
    Solution: scheduled destroy at night, recreate on demand
    OR: use cheaper instance types for PR envs

  Challenge 3: State file left if workflow fails
    Solution: S3 lifecycle rule: delete state files older than 30 days
              with prefix pr-*/

  Challenge 4: Database with skip_final_snapshot = false blocks destroy
    Solution: always set skip_final_snapshot = true for ephemeral envs
```

💡 **Interview tip:** The `skip_final_snapshot = true` on RDS is critical — forgetting this means `terraform destroy` hangs waiting for a final snapshot that you don't need for a PR environment. The Terraform workspace per PR approach is clean but has a scale issue — 50 open PRs = 50 state files, 50 sets of AWS resources. Mention the cost optimization: for PR environments, use the smallest possible instance sizes and consider destroying overnight with a scheduled action to avoid paying for idle resources.

---

### Q124 — Prometheus | Troubleshooting | Advanced

> **Alert flapping** — alerts firing and resolving every few minutes. On-call team getting spammed. How to fix?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`

```
What is flapping:
  Alert fires   → notification sent
  Alert resolves → resolved notification sent
  Alert fires   → notification sent again
  Repeat every 2-3 minutes indefinitely

  Root cause: metric oscillates around the alert threshold

  Example:
    Alert: CPU > 80% for 5m
    CPU: 82% → 78% → 83% → 77% → 81%...
    Never stable above OR below 80% for 5 full minutes
    Alert fires on the way up, resolves on the way down, repeats

Fix 1 — Increase the `for` duration in alert rules:
  # Too sensitive:
  - alert: HighCPU
    expr: cpu_usage > 80
    for: 1m       # fires after 1 minute above threshold
                  # resolves almost immediately when dips below

  # Better — must be elevated for longer to fire:
  - alert: HighCPU
    expr: cpu_usage > 80
    for: 10m      # must stay above 80% for 10 full minutes
                  # brief spikes don't trigger it
                  # reduces false positives

Fix 2 — Add hysteresis (different thresholds for fire vs resolve):
  Prometheus doesn't have native hysteresis
  Workaround: use two rules

  - alert: HighCPUFiring
    expr: cpu_usage > 85      # fires at 85%
    for: 5m

  - alert: HighCPUWarning
    expr: cpu_usage > 75      # resolves when drops below 75%
    for: 5m

  # 10% buffer between fire and resolve threshold
  # Metric must drop significantly before resolving

Fix 3 — Smooth the metric with rate/avg_over_time:
  # Raw metric is noisy — use averages:
  expr: avg_over_time(cpu_usage[15m]) > 80
  # 15-minute rolling average — doesn't spike on brief bursts

  # Or use rate smoothing:
  expr: rate(http_errors_total[10m]) > 0.05
  # 10-minute rate — stable, not per-second noise

Fix 4 — Alertmanager: group_wait and repeat_interval:
  # alertmanager.yml:
  route:
    group_wait: 30s          # wait 30s before sending first notification
                             # other alerts in group collected during this time
    group_interval: 5m       # wait 5m before sending update for ongoing alert
    repeat_interval: 4h      # re-notify only every 4 hours for ongoing alert

  # repeat_interval = 4h means: alert fires, notified, if still firing
  #                   4 hours later you get another notification
  #                   Not every 2 minutes

Fix 5 — Alertmanager inhibition rules:
  # If higher severity alert is firing, silence lower severity:
  inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['service', 'instance']
  # High CPU (critical) silences Slow Response (warning) on same instance
  # Reduces notification flood when multiple alerts trigger together

Fix 6 — Dead man's switch pattern:
  # Alert that fires when Prometheus STOPS sending (opposite of normal):
  - alert: PrometheusDown
    expr: absent(up{job="prometheus"})
    for: 5m
  # Useful for confirming monitoring is working
```

💡 **Interview tip:** Increasing `for` duration is the first and most effective fix — it filters out brief transient spikes. The key insight about `repeat_interval` in Alertmanager is often missed — it controls how often you get re-notified about an ongoing alert, not how often the alert is evaluated. Setting `repeat_interval: 4h` means a production alert that stays firing won't spam the on-call every 2 minutes. For metrics that are genuinely noisy by nature, smoothing with `avg_over_time` is more principled than just increasing thresholds.

---

### Q125 — AWS | Conceptual | Advanced

> Explain **ASG scaling policies** in detail — Target Tracking, Step Scaling, Simple Scaling, Scheduled. Cooldown periods and instance warm-up.

📁 **Reference:** `nawab312/AWS` — Auto Scaling Groups

```
ASG Scaling Policies:

1. Target Tracking Scaling (recommended — simplest):
   → Define a target metric value, ASG maintains it automatically
   → Similar to a thermostat: "keep CPU at 50%"
   → AWS calculates required instance count automatically
   → Works bidirectionally (scale out AND in)

   aws autoscaling put-scaling-policy \
     --policy-name cpu-target-tracking \
     --auto-scaling-group-name my-asg \
     --policy-type TargetTrackingScaling \
     --target-tracking-configuration '{
       "PredefinedMetricSpecification": {
         "PredefinedMetricType": "ASGAverageCPUUtilization"
       },
       "TargetValue": 50.0,
       "ScaleInCooldown": 300,
       "ScaleOutCooldown": 60
     }'

   Built-in metrics: CPU, NetworkIn/Out, ALB RequestCount per target

2. Step Scaling:
   → Different scaling amounts based on how far metric exceeds threshold
   → "At 70% CPU: add 1 instance. At 90% CPU: add 3 instances."
   → More granular than simple scaling

   Steps:
     CPU 70-80%: add 1 instance
     CPU 80-90%: add 2 instances
     CPU 90%+:   add 4 instances

   Use when: you know different breach levels need different responses

3. Simple Scaling (legacy — avoid for new workloads):
   → Single scaling action when alarm breaches
   → Waits for cooldown period before evaluating again
   → Simple but slow to respond to rapidly changing load
   → Replaced by Target Tracking for most use cases

4. Scheduled Scaling:
   → Pre-configured scaling at specific times
   → "Add 10 instances at 8 AM Monday, remove at 8 PM"
   → Useful for predictable traffic patterns

   aws autoscaling put-scheduled-update-group-action \
     --auto-scaling-group-name my-asg \
     --scheduled-action-name morning-scale-out \
     --recurrence "0 8 * * MON-FRI" \
     --min-size 10 --desired-capacity 10

Cooldown periods:
  → Prevent scaling from running before previous scaling completes
  → Scale-out cooldown (default 300s):
     After adding instances, wait 300s before scaling out again
     Lower value = react faster to continued load spike
     Typical: 60-90s for scale-out

  → Scale-in cooldown (default 300s):
     After removing instances, wait 300s before removing more
     Higher value = more conservative scale-in (avoid removing too fast)
     Typical: 300-600s for scale-in

Instance warm-up:
  → New EC2 instance takes time to be ready (bootstrap, health check)
  → Without warm-up: metrics from new instance (0% CPU, not ready)
     pollutes average → ASG thinks load is lower than it is → stops scaling
  → With warm-up: new instances excluded from metrics for N seconds
     ASG continues scaling until warm-up period completes
     Then new instances counted in the average

  instance_warmup: 120    # 120 seconds before new instance counted in avg
  → Set to: time from launch to when instance is actually serving traffic
```

💡 **Interview tip:** Target Tracking is the answer for 95% of use cases — don't over-engineer with Step Scaling unless you have a specific reason. The key concept around cooldown: scale-out cooldown should be short (60s) because you want to add more capacity quickly during a spike; scale-in cooldown should be long (300-600s) because you want to be conservative about removing capacity. Instance warm-up is the often-forgotten setting that prevents premature scale-in right after scale-out.

---

### Q126 — Kubernetes | Scenario-Based | Advanced

> Enforce pod security hardening across the entire cluster — no root, no privilege escalation, read-only filesystem, no host networking.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md`

```
Option 1 — Pod Security Admission (built-in, simplest):

  Apply Restricted profile to all namespaces:
  kubectl label namespace production \
    pod-security.kubernetes.io/enforce=restricted \
    pod-security.kubernetes.io/enforce-version=v1.28

  Restricted profile enforces:
    ✓ runAsNonRoot: true
    ✓ allowPrivilegeEscalation: false
    ✓ capabilities.drop: ["ALL"]
    ✓ seccompProfile: RuntimeDefault

  kube-system must remain Privileged (CNI, CSI need host access):
  kubectl label namespace kube-system \
    pod-security.kubernetes.io/enforce=privileged

Option 2 — OPA/Gatekeeper (more granular control):

  ConstraintTemplate for no-root:
  apiVersion: templates.gatekeeper.sh/v1
  kind: ConstraintTemplate
  metadata:
    name: k8snoroot
  spec:
    crd:
      spec:
        names:
          kind: K8sNoRoot
    targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snoroot
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Container '%v' must set runAsNonRoot: true", [container.name])
        }

Pod securityContext for compliant pods:

apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault

  containers:
  - name: app
    image: myapp:v1
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    # hostNetwork: false (default — no need to set)
    # hostPID: false (default)
    # hostIPC: false (default)
    volumeMounts:
    - name: tmp
      mountPath: /tmp              # writable temp dir for readOnlyRootFilesystem

  volumes:
  - name: tmp
    emptyDir: {}

Enforcement strategy (gradual rollout):

  Phase 1 — audit mode (discover violations):
  kubectl label namespace production \
    pod-security.kubernetes.io/audit=restricted

  Phase 2 — warn mode (developers see warnings):
  kubectl label namespace production \
    pod-security.kubernetes.io/warn=restricted

  Phase 3 — enforce mode (reject violations):
  kubectl label namespace production \
    pod-security.kubernetes.io/enforce=restricted

  Important: enforce does NOT evict existing running pods
             Only new/updated pods are subject to enforcement
             Must restart existing pods to apply enforcement to them

Common issues when enforcing:
  → Many official images run as root (nginx:latest, etc.)
    Fix: use non-root variants (nginxinc/nginx-unprivileged)
  → Apps write to /tmp or /var/cache
    Fix: add emptyDir volumes for writable paths
  → Java apps need CAP_NET_BIND_SERVICE for port < 1024
    Fix: run on port > 1024 (8080 not 80)
```

💡 **Interview tip:** Pod Security Admission is the built-in answer — no additional components needed. The three-phase rollout (audit → warn → enforce) is the production approach because jumping straight to enforce will break existing workloads. The `readOnlyRootFilesystem: true` setting requires explicit emptyDir volumes for any path the app writes to — forgetting this is the most common reason apps crash after hardening. Mention that `kube-system` must stay Privileged — applying Restricted to system namespaces will break your CNI and CSI drivers.

---

### Q127 — ArgoCD | Scenario-Based | Advanced

> Explain **3 approaches to secrets management with ArgoCD/GitOps** and their trade-offs.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD`

```
The GitOps problem:
  Everything should be in Git (ArgoCD syncs from Git)
  Secrets cannot be in Git (plaintext or base64 is not secure)
  How do you manage secrets that ArgoCD needs to deploy?

Approach 1 — Sealed Secrets:

  How it works:
  kubeseal encrypts Secret using cluster's public key
  Encrypted SealedSecret object stored safely in Git
  Sealed Secrets controller in cluster decrypts → creates real Secret

  kubectl create secret generic db-password \
    --dry-run=client -o yaml | \
    kubeseal --controller-namespace kube-system \
    > sealed-db-password.yaml
  # sealed-db-password.yaml goes into Git safely

  Pros:
  ✓ Fully GitOps — everything in Git
  ✓ Simple to understand and operate
  ✓ No external dependency at runtime

  Cons:
  ✗ Cluster-specific: sealed secret encrypted for ONE cluster
    → Can't use same sealed secret in staging and prod
  ✗ Key rotation is complex (must re-seal ALL secrets)
  ✗ No secret rotation (static values)
  ✗ No audit trail of secret access

Approach 2 — External Secrets Operator (ESO):

  How it works:
  ExternalSecret CR in Git (no actual secret values)
  ESO controller reads values from AWS Secrets Manager/Vault
  ESO creates Kubernetes Secret automatically

  # In Git (safe — no secret values):
  apiVersion: external-secrets.io/v1beta1
  kind: ExternalSecret
  metadata:
    name: db-credentials
  spec:
    refreshInterval: 1h
    secretStoreRef:
      name: aws-secrets-manager
      kind: ClusterSecretStore
    target:
      name: db-credentials
    data:
    - secretKey: password
      remoteRef:
        key: prod/myapp/db-password

  Pros:
  ✓ Secret values never in Git or cluster state
  ✓ Supports rotation (refreshInterval syncs updated values)
  ✓ Works across environments (same ESO, different Secrets Manager paths)
  ✓ Audit trail in AWS CloudTrail (who accessed which secret)

  Cons:
  ✗ Dependency on external service (Secrets Manager must be available)
  ✗ More complex setup (ESO + SecretStore + IAM/IRSA)
  ✗ Slight delay: secret updates propagate on refresh interval

Approach 3 — Vault + Vault Agent / Vault Secrets Operator:

  How it works:
  HashiCorp Vault stores secrets centrally
  Option A: Vault Agent sidecar injects secrets as files at pod startup
  Option B: Vault Secrets Operator syncs Vault secrets to K8s Secrets

  # Vault Agent annotation approach:
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp"
    vault.hashicorp.com/agent-inject-secret-config.txt: "secret/myapp/config"
    vault.hashicorp.com/agent-inject-template-config.txt: |
      {{- with secret "secret/myapp/config" -}}
      DB_PASSWORD={{ .Data.data.password }}
      {{- end }}

  Pros:
  ✓ Most powerful: dynamic secrets (Vault generates DB credentials per pod)
  ✓ Fine-grained policies per service
  ✓ Complete audit trail in Vault
  ✓ Automatic secret lease renewal

  Cons:
  ✗ Vault is complex to operate (HA Vault cluster needed)
  ✗ Vault becomes a critical dependency (it goes down → new pods can't start)
  ✗ Steeper learning curve

Summary:

  Small team, simple needs:       Sealed Secrets
  AWS-native, moderate scale:     External Secrets Operator + Secrets Manager
  Enterprise, multi-cloud, audit: HashiCorp Vault
```

💡 **Interview tip:** External Secrets Operator is the most commonly recommended answer for AWS environments — it integrates naturally with IAM/IRSA, AWS Secrets Manager handles rotation natively, and CloudTrail provides audit logs. Sealed Secrets is the simplest GitOps answer but the cluster-specific limitation is a real operational pain. Vault is the enterprise answer — mention that Vault's dynamic secrets (it generates temporary DB credentials per pod, valid for 1 hour) is a fundamentally better security model than static passwords.

---

### Q128 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash deployment rollback system** — last 5 versions, deploy/rollback/list commands, logging, Slack notification.

📁 **Reference:** `nawab312/DSA` → `Linux`

```bash
#!/bin/bash

DEPLOY_DIR="/opt/deployments"
VERSIONS_DIR="${DEPLOY_DIR}/versions"
CURRENT_LINK="${DEPLOY_DIR}/current"
LOG_FILE="${DEPLOY_DIR}/deploy.log"
SLACK_WEBHOOK="${SLACK_WEBHOOK_URL:-}"
MAX_VERSIONS=5

mkdir -p "$VERSIONS_DIR"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

notify_slack() {
  local message="$1"
  [ -z "$SLACK_WEBHOOK" ] && return
  curl -s -X POST "$SLACK_WEBHOOK" \
    -H 'Content-Type: application/json' \
    -d "{\"text\": \"${message}\"}" > /dev/null
}

get_current_version() {
  [ -L "$CURRENT_LINK" ] && basename "$(readlink "$CURRENT_LINK")" || echo "none"
}

list_versions() {
  echo ""
  echo "Available versions (newest first):"
  echo "──────────────────────────────────"
  local current
  current=$(get_current_version)

  ls -t "$VERSIONS_DIR" 2>/dev/null | while read -r version; do
    local ts
    ts=$(stat -c %y "${VERSIONS_DIR}/${version}" 2>/dev/null | cut -d'.' -f1)
    if [ "$version" = "$current" ]; then
      echo "  * ${version}  [CURRENT]  ${ts}"
    else
      echo "    ${version}             ${ts}"
    fi
  done
  echo ""
}

deploy() {
  local version="$1"

  if [ -z "$version" ]; then
    echo "Usage: $0 deploy <version>"
    exit 1
  fi

  local version_path="${VERSIONS_DIR}/${version}"

  if [ ! -d "$version_path" ]; then
    # Simulate downloading/extracting the version
    log "Fetching version: ${version}"
    mkdir -p "$version_path"
    echo "$version" > "${version_path}/version.txt"
    echo "$(date)" > "${version_path}/deployed_at.txt"
  fi

  local previous
  previous=$(get_current_version)

  log "Deploying version: ${version} (was: ${previous})"

  # Update symlink atomically
  ln -sfn "$version_path" "$CURRENT_LINK"

  log "Deployment complete: ${version}"
  notify_slack ":rocket: *Deployed* \`${version}\` to production (was: \`${previous}\`)"

  # Keep only last MAX_VERSIONS versions
  local count
  count=$(ls "$VERSIONS_DIR" | wc -l)
  if [ "$count" -gt "$MAX_VERSIONS" ]; then
    ls -t "$VERSIONS_DIR" | tail -n +"$((MAX_VERSIONS + 1))" | while read -r old; do
      log "Pruning old version: ${old}"
      rm -rf "${VERSIONS_DIR:?}/${old}"
    done
  fi
}

rollback() {
  local current
  current=$(get_current_version)

  if [ "$current" = "none" ]; then
    echo "No current deployment found."
    exit 1
  fi

  # Get the version before current
  local previous
  previous=$(ls -t "$VERSIONS_DIR" | grep -v "^${current}$" | head -1)

  if [ -z "$previous" ]; then
    echo "No previous version available to roll back to."
    exit 1
  fi

  log "Rolling back from ${current} to ${previous}"
  ln -sfn "${VERSIONS_DIR}/${previous}" "$CURRENT_LINK"
  log "Rollback complete: now on ${previous}"
  notify_slack ":rewind: *Rolled back* from \`${current}\` to \`${previous}\`"
}

# Main command dispatcher
case "${1:-}" in
  deploy)   deploy "$2" ;;
  rollback) rollback ;;
  list)     list_versions ;;
  *)
    echo "Usage: $0 {deploy <version>|rollback|list}"
    echo ""
    echo "  deploy <version>  Deploy a specific version"
    echo "  rollback          Roll back to the previous version"
    echo "  list              Show all available versions"
    exit 1
    ;;
esac
```

💡 **Interview tip:** The `ln -sfn` (symlink swap) is the key deployment primitive — it changes the `current` pointer atomically, so there is no window where `current` points to nothing. The `grep -v "^${current}$"` in rollback filters out the current version to find the previous one. Mention that in real production, the "deploy" function would run the actual deployment commands (pull Docker image, restart service, run health checks) before updating the symlink — only update the pointer after the deployment succeeds.

---

### Q129 — AWS | Scenario-Based | Advanced

> Implement **PII security on AWS** — encryption at rest/transit, data masking in logs, access audit trail, 2-year auto-delete.

📁 **Reference:** `nawab312/AWS` — KMS, DynamoDB, S3, CloudTrail

```
Encryption at rest:

  DynamoDB:
    → Enable at creation with CMK (Customer Managed Key):
    aws dynamodb create-table \
      --table-name users \
      --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=arn:aws:kms:...
    → CMK gives you: key rotation control, cross-account access, key policy
    → Audit: every KMS decrypt call logged in CloudTrail

  S3:
    → Bucket-level default encryption:
    aws s3api put-bucket-encryption \
      --bucket pii-data-bucket \
      --server-side-encryption-configuration '{
        "Rules": [{
          "ApplyServerSideEncryptionByDefault": {
            "SSEAlgorithm": "aws:kms",
            "KMSMasterKeyId": "arn:aws:kms:..."
          },
          "BucketKeyEnabled": true
        }]
      }'
    → Bucket policy: deny PutObject without encryption header

Encryption in transit:
  → S3: bucket policy denying non-HTTPS requests:
  {"Condition": {"Bool": {"aws:SecureTransport": "false"}}, "Effect": "Deny"}

  → DynamoDB: HTTPS only (AWS enforces this — no config needed)
  → RDS: require SSL in parameter group + enforce_ssl = 1
  → ALB: redirect HTTP → HTTPS, TLS 1.2 minimum policy

Data masking in logs:
  → Lambda: never log raw PII — mask before logging:
  def mask_pii(data):
      return {
          'email': data['email'][:3] + '***@' + data['email'].split('@')[1],
          'ssn': '***-**-' + data['ssn'][-4:],
          'name': data['name'][0] + '***'
      }

  → CloudWatch Logs Data Protection:
  aws logs create-data-protection-policy \
    --log-group-name /aws/lambda/user-service \
    --policy-document '{
      "DataIdentifiers": ["arn:aws:dataprotection::aws:data-identifier/EmailAddress",
                          "arn:aws:dataprotection::aws:data-identifier/SocialSecurityNumber"],
      "Operation": {"Deidentify": {"MaskConfig": {}}}
    }'
  → Auto-masks SSN and email patterns in CloudWatch logs

Access audit trail:
  → CloudTrail: log all S3 GetObject and DynamoDB GetItem calls:
    Enable Data Events for S3 and DynamoDB in CloudTrail
    Cost: $0.10 per 100,000 events

  → S3 Server Access Logs: who accessed which object, when
  → DynamoDB Streams: all write operations with before/after values
  → AWS Macie: scan S3 buckets for PII, alert on unexpected access

2-year auto-delete:
  S3 Lifecycle Policy:
    aws s3api put-bucket-lifecycle-configuration \
      --bucket pii-data-bucket \
      --lifecycle-configuration '{
        "Rules": [{
          "ID": "delete-pii-after-2-years",
          "Status": "Enabled",
          "Expiration": {"Days": 730},
          "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
        }]
      }'

  DynamoDB TTL:
    → Add ttl attribute to each item (Unix timestamp 2 years ahead)
    → Enable TTL on the attribute:
    aws dynamodb update-time-to-live \
      --table-name users \
      --time-to-live-specification Enabled=true,AttributeName=ttl
    → DynamoDB automatically deletes items when TTL expires (within 48h)

  GDPR "right to be forgotten":
    → User deletion request → delete specific items
    → S3: use object tagging to mark user's data → lifecycle rule deletes tagged objects
```

💡 **Interview tip:** CloudWatch Logs Data Protection is a newer feature many engineers don't know about — it automatically detects and masks PII patterns in logs at the CloudWatch level, so even if application code accidentally logs PII, it's masked before anyone can read it. DynamoDB TTL is the cleanest answer for 2-year auto-delete — it's built-in, serverless, and doesn't require batch jobs or Lambda. Mention that enabling DynamoDB data events in CloudTrail is expensive ($0.10/100K events) but required for compliance audit trails — recommend sampling or filtering to specific tables to manage cost.

---

### Q130 — Kubernetes | Conceptual | Advanced

> Explain **ServiceAccount token evolution** — static tokens to bound tokens. What is **IRSA** and why is it better than EC2 instance profiles?

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md`

```
Old tokens (K8s < 1.24):
  → Creating a ServiceAccount auto-created a Secret
  → Secret contained: long-lived JWT that NEVER expires
  → Token mounted at: /var/run/secrets/kubernetes.io/serviceaccount/token
  → Risk: token stolen from pod → attacker has permanent cluster access
           even after pod deleted, token still valid

New bound tokens (K8s 1.22+ default, 1.24+ mandatory):
  → Projected volume injects token directly (no Secret object)
  → Token is:
      - Time-limited: expires after 1 hour (default)
      - Pod-bound: invalid when pod is deleted
      - Audience-bound: only valid for kubernetes API server
  → Kubelet auto-rotates at 80% of lifetime
  → Token stolen: expires in < 1 hour, can't be renewed without the pod

Token format comparison:
  Old: {"sub": "system:serviceaccount:default:myapp", "exp": never}
  New: {"sub": "system:serviceaccount:default:myapp",
        "exp": 1720000000,
        "kubernetes.io": {"pod": {"name": "myapp-xyz", "uid": "..."}}}

IRSA — IAM Roles for Service Accounts (EKS):

  Problem IRSA solves:
    EC2 instance profile: ALL pods on the node share the same AWS permissions
    One compromised pod → attacker has the node's IAM role → too broad

  How IRSA works:
    1. EKS cluster has built-in OIDC provider URL
    2. IAM role trust policy: allow THIS specific ServiceAccount to assume it
    3. ServiceAccount annotated with IAM role ARN
    4. Pod admission webhook injects: AWS_ROLE_ARN + token file path
    5. AWS SDK reads SA token → calls STS AssumeRoleWithWebIdentity
    6. STS validates token against EKS OIDC endpoint
    7. Returns temporary credentials (valid 1 hour)

  Trust policy scoped to specific SA:
  "Condition": {
    "StringEquals": {
      "oidc.eks.us-east-1.amazonaws.com/.../sub":
        "system:serviceaccount:production:payment-sa"
    }
  }

IRSA vs EC2 Instance Profile:

  Instance Profile:
    All pods on node → SAME IAM role
    payment-service can call S3
    user-service can also call S3 (shouldn't be able to)
    Compromise any pod → get all node permissions

  IRSA:
    payment-service SA → IAM role → only S3 payment-receipts bucket
    user-service SA    → IAM role → only DynamoDB users table
    Compromise payment-service → can only access payment-receipts
    No access to user-service permissions

  IRSA benefits:
  ✓ Least privilege: per-pod AWS permissions
  ✓ Automatic rotation: 1-hour credentials, no manual rotation
  ✓ Audit trail: CloudTrail shows which pod assumed which role
  ✓ No AWS credentials in Secrets or environment variables
```

💡 **Interview tip:** IRSA is the Kubernetes security topic most relevant to AWS interviewers — it demonstrates you understand both K8s service accounts AND AWS IAM deeply. The key comparison: instance profiles give all pods on a node the same AWS permissions (blast radius = node), IRSA gives each pod its own IAM role (blast radius = one service). Mention EKS Pod Identity as the newer, simpler alternative to IRSA — it doesn't require annotating the ServiceAccount and is managed through the EKS API, though IRSA is still more widely used.

---

### Q131 — Grafana | Troubleshooting | Advanced

> **Grafana dashboards taking 45 seconds to load** (was 2 seconds). Server looks normal. Investigate and fix.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana`

```
Step 1 — Identify slow panels:
  Grafana built-in: open dashboard → F12 (browser dev tools) → Network tab
  → Which API calls take longest?
  → /api/datasources/proxy/... → slow panel queries

  Grafana UI: enable query inspector per panel
  Panel → Edit → Query Inspector → Show query time
  → Identify which panel(s) are the slowest

Step 2 — Check Prometheus query performance:
  If Prometheus is the datasource:
  → Evaluate the slow PromQL directly in Prometheus UI
  → Add to query: | by (__name__) → see evaluation time

  Common slow queries:
    rate(metric[1m]) over 6 months → scanning 6 months of data
    sum without(instance)(...) with 1M series → high cardinality
    join operations: metric_a * on(pod) metric_b → expensive

  Fix slow PromQL:
    → Use recording rules to pre-compute expensive queries:
    groups:
    - name: precomputed
      interval: 1m
      rules:
      - record: job:http_requests:rate5m
        expr: sum by(job) (rate(http_requests_total[5m]))
    → Dashboard queries job:http_requests:rate5m (instant) instead
       of the expensive sum(rate(...)) (recomputed each load)

Step 3 — Check time range:
  Dashboard time range: "Last 6 months"?
  → Prometheus must scan 6 months of data per panel
  → Fix: change default time range to "Last 24h" or "Last 7d"
  → Use step parameter: longer step = fewer data points = faster

Step 4 — Check panel count:
  Dashboard has 50+ panels?
  → All load simultaneously on dashboard open
  → Each panel = one or more Prometheus queries
  → Fix: use panel lazy loading (Enterprise) or split into multiple dashboards
  → Fix: reduce panel count (combine related panels)

Step 5 — Check Grafana datasource query timeout:
  Grafana config (grafana.ini):
  [dataproxy]
  timeout = 30           # default: 30s
  → If Prometheus takes 45s → Grafana times out → shows error
  → Fix: increase timeout OR fix the slow query

Step 6 — Check Prometheus resource utilisation:
  → Prometheus TSDB head_chunks too large (high cardinality)
  → CPU maxed on query evaluation
  → kubectl top pod prometheus-0 during dashboard load
  → If CPU spikes: reduce query complexity, add recording rules

Step 7 — Enable caching (Grafana 9+):
  [caching]
  enabled = true
  backend = redis
  → Cache query results for N seconds
  → Multiple users loading same dashboard = one Prometheus query
  → Dramatically reduces load on Prometheus

Step 8 — Check variables (dashboard variables as high-cardinality queries):
  Dashboard variable: $namespace = query all namespaces
  Evaluated on every dashboard load
  1000 namespaces = slow variable query
  Fix: static list or limit variable query scope
```

💡 **Interview tip:** Recording rules are the most impactful fix for slow Grafana dashboards — pre-computing expensive queries means dashboard load becomes an instant lookup rather than a multi-second computation. The `step` parameter is often overlooked: Grafana auto-calculates step based on time range and panel width — a 7-day range with a narrow panel = very fine resolution = many data points = slow. Setting `min_step: 5m` forces coarser resolution which loads much faster for overview dashboards.

---

### Q132 — Terraform | Conceptual | Advanced

> Explain **`moved` block**, **`removed` block**, and **`import` block** (Terraform 1.5+). How do they improve on CLI commands?

📁 **Reference:** `nawab312/Terraform` — Terraform 1.x features

```
Old CLI approach (Terraform < 1.5):

  terraform state mv → moves resources in state (CLI only, not tracked in code)
  terraform import  → imports resource into state (CLI only, not in code)
  terraform state rm → removes from state (CLI only)

  Problem: team member runs terraform state mv manually
           No record in Git of why or what moved
           Next developer has no idea why resources are named differently
           CI/CD can't run these automatically

New code-based approach (Terraform 1.5+):

1. moved block — rename resource without destroying/recreating:

  # Before refactor: monolithic module
  resource "aws_s3_bucket" "my_bucket" {}

  # After refactor: moved to a module
  module "s3" {
    source = "./modules/s3"
  }

  # Add moved block to tell Terraform about the rename:
  moved {
    from = aws_s3_bucket.my_bucket
    to   = module.s3.aws_s3_bucket.this
  }

  Terraform plan shows:
    # aws_s3_bucket.my_bucket has moved to module.s3.aws_s3_bucket.this
    # (No changes needed)

  Benefits vs terraform state mv:
  ✓ Tracked in Git — reviewable in PR
  ✓ Works in CI/CD without manual intervention
  ✓ Documents WHY the resource was moved
  ✓ Team members get correct plan without manual steps

2. import block — declare resource import in code (Terraform 1.5+):

  # Instead of: terraform import aws_s3_bucket.existing my-bucket-name
  # Declare in .tf file:

  import {
    to = aws_s3_bucket.existing
    id = "my-existing-bucket-name"
  }

  resource "aws_s3_bucket" "existing" {
    bucket = "my-existing-bucket-name"
    # Terraform 1.5+ can also GENERATE this resource block from the import
  }

  # terraform plan shows the import + any drift
  # terraform apply imports the resource into state

  Benefits:
  ✓ Import is code-reviewed (not a CLI command run by one person)
  ✓ Works in automated pipelines
  ✓ Can be committed alongside the resource definition

  # Terraform 1.5+ generation:
  terraform plan -generate-config-out=generated.tf
  → Auto-generates resource block from real AWS resource

3. removed block — remove from state without destroying (Terraform 1.7+):

  # Remove from state but keep the real AWS resource:
  removed {
    from = aws_s3_bucket.legacy
    lifecycle {
      destroy = false  # keep the real resource, just stop tracking it
    }
  }

  vs terraform state rm:
  ✓ Tracked in Git
  ✓ destroy = false makes intent explicit (don't delete the real resource)
  ✓ Reviewable — team knows this was intentional

Real-world moved block example (module refactor):

  # Old: standalone resources
  resource "aws_lb" "main" {}
  resource "aws_lb_listener" "http" {}

  # New: wrapped in module
  module "alb" {
    source = "./modules/alb"
  }

  # moved blocks for safe migration:
  moved {
    from = aws_lb.main
    to   = module.alb.aws_lb.this
  }
  moved {
    from = aws_lb_listener.http
    to   = module.alb.aws_lb_listener.http
  }

  # terraform plan: 0 changes, 0 destroys (just state moves)
  # after apply: delete the moved blocks from code
```

💡 **Interview tip:** The key selling point of these code-based approaches is GitOps compatibility — state operations become PR-reviewable code changes instead of undocumented CLI commands. This matters in team environments where `terraform state mv` run by one engineer leaves others confused about why the state differs from the code. The `import` block with `-generate-config-out` is the most impressive answer — Terraform can now auto-generate the resource definition from the existing AWS resource, dramatically simplifying the import of complex resources like RDS or EKS clusters.

---

### Q133 — Git | Scenario-Based | Advanced

> **AWS credentials committed and pushed to public GitHub 3 days ago**. Complete incident response — secure, clean history, prevent recurrence.

📁 **Reference:** `nawab312/CI_CD` → `Git`

```
Step 1 — IMMEDIATELY rotate the exposed credentials (do this first):

  # Before touching Git — the credentials are already out there
  # Assume they have been compromised from the moment they were pushed

  aws iam delete-access-key \
    --user-name ci-user \
    --access-key-id AKIAIOSFODNN7EXAMPLE
  aws iam create-access-key --user-name ci-user
  # New key generated

  # Check CloudTrail for any unusual usage of the old key:
  aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=Username,AttributeValue=ci-user \
    --start-time 2024-01-12T00:00:00Z  # 3 days ago

  # Check for: unknown regions, unusual services, large data transfers
  # If suspicious activity: escalate — may have been exploited

Step 2 — Remove from Git history:

  Method A — BFG Repo Cleaner (faster, recommended):
  # Download BFG:
  java -jar bfg.jar \
    --replace-text credentials.txt \  # file with strings to replace
    --no-blob-protection \
    my-repo.git

  # credentials.txt contents:
  AKIAIOSFODNN7EXAMPLE==>REMOVED
  wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY==>REMOVED

  Method B — git filter-repo (modern, built-in alternative to BFG):
  pip install git-filter-repo
  git filter-repo \
    --replace-text credentials.txt \
    --force
  # Rewrites entire history — all SHAs change

  Step 2b — Force push rewritten history:
  git push origin --force --all
  git push origin --force --tags
  # WARNING: this breaks everyone's local clones

  Step 2c — Notify all contributors to re-clone:
  # Email team: "Repository history rewritten — please delete local clone and re-clone"
  # GitHub: any forks also have the history — contact GitHub Support for fork cleanup

Step 3 — GitHub-specific: request secret scanning cleanup:
  GitHub automatically scans public repos for known secret patterns
  GitHub may have already notified AWS (GitHub Secret Scanning → AWS)
  Contact GitHub Support to invalidate any cached versions

Step 4 — Prevent recurrence:

  Pre-commit hooks (local detection):
  pip install pre-commit detect-secrets
  # .pre-commit-config.yaml:
  repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
    - id: detect-secrets
      args: ['--baseline', '.secrets.baseline']
  pre-commit install
  # Now: git commit with AWS key → automatically blocked

  GitHub Actions secret scanning:
  → Enable in repo settings: Security → Secret scanning → Enable
  → GitHub will alert you if secrets are committed

  CI check:
  - name: Scan for secrets
    uses: trufflesecurity/trufflehog@main
    with:
      path: ./
      base: main
      head: HEAD

  Fix root cause — don't use long-lived credentials:
  → Use IRSA (for EKS workloads)
  → Use GitHub Actions OIDC (for CI/CD)
  → Use IAM roles via aws-vault (for local development)
  → Never store credentials in files or environment variables that get committed

  GitHub OIDC for GitHub Actions:
  # No credentials in GitHub Secrets at all:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123:role/github-actions-role
      aws-region: us-east-1
  # GitHub federated identity → STS token → temporary credentials
```

💡 **Interview tip:** The order matters: rotate credentials FIRST before touching Git — the history rewrite doesn't help if the credentials are still active. BFG Repo Cleaner or git filter-repo rewrites all commit SHAs — anyone with a local clone now has diverged history and must re-clone. The long-term prevention answer interviewers want: switch from static IAM credentials entirely by using GitHub Actions OIDC — this eliminates the entire class of "credentials committed to Git" by ensuring credentials are never stored anywhere.

---

### Q134 — AWS + Kubernetes | Scenario-Based | Advanced

> Migrate a **monolithic EC2 application** to microservices on EKS — PostgreSQL 1TB, local disk storage, in-memory sessions, cron jobs.

📁 **Reference:** `nawab312/AWS`, `nawab312/Kubernetes`

```
Component 1 — PostgreSQL 1TB database:

  Strategy: Lift-and-shift first, optimise later
  Target: Amazon RDS PostgreSQL (managed)

  Migration steps:
  1. Create RDS PostgreSQL with Multi-AZ
  2. AWS DMS (Database Migration Service) for initial load + CDC
     → DMS streams changes from source → RDS in real time
     → Zero-downtime migration
  3. Cutover window: stop writes to old DB, flush DMS, update connection string
  4. Verify row counts and spot check data integrity

  RDS sizing for 1TB:
    Storage: 2TB gp3 (room to grow)
    Instance: db.r6g.2xlarge (16GB RAM for 1TB data)
    Multi-AZ: yes (automatic failover)
    Read replicas: 2 (for read-heavy queries)

  Connection management:
    → RDS Proxy: manage connection pool for many EKS pods → RDS

Component 2 — Local disk storage (images, uploads):

  Target: Amazon S3 + CloudFront

  Migration steps:
  1. Audit what's on local disk: user uploads, processed files, temp files
  2. Sync existing files to S3:
     aws s3 sync /var/app/uploads s3://myapp-uploads/
  3. Update application code:
     Before: open('/var/app/uploads/user-photo.jpg')
     After:  s3.get_object(Bucket='myapp-uploads', Key='user-photo.jpg')
  4. CloudFront distribution in front of S3 for fast global delivery
  5. Pre-signed URLs for private file access (no public S3)

  For temporary/scratch files: emptyDir volume in Kubernetes pod
  For shared persistent storage: EFS (NFS) if multiple pods need same files

Component 3 — In-memory sessions:

  Target: Amazon ElastiCache Redis

  Problem: in-memory sessions don't survive pod restarts
           each pod has its own session store
           load balancer routes to different pod → session lost

  Solution:
  1. Deploy ElastiCache Redis cluster (Multi-AZ)
  2. Update session middleware to use Redis:
     # Python Flask:
     app.config['SESSION_TYPE'] = 'redis'
     app.config['SESSION_REDIS'] = redis.from_url('redis://elasticache-endpoint:6379')
  3. All pods share the same Redis → sessions persist across pod restarts and scaling

Component 4 — Cron jobs:

  Target: Kubernetes CronJob

  Before: EC2 crontab
    0 2 * * * /opt/app/scripts/nightly-report.sh
    */5 * * * * /opt/app/scripts/health-check.sh

  After: Kubernetes CronJob
  apiVersion: batch/v1
  kind: CronJob
  metadata:
    name: nightly-report
  spec:
    schedule: "0 2 * * *"
    jobTemplate:
      spec:
        template:
          spec:
            containers:
            - name: report
              image: myapp:v1
              command: ["python", "scripts/nightly_report.py"]
            restartPolicy: OnFailure
    successfulJobsHistoryLimit: 3
    failedJobsHistoryLimit: 3
    concurrencyPolicy: Forbid    # don't run if previous still running

Migration approach (strangler fig pattern):

  Phase 1: Extract stateless services first (easier, lower risk)
    → API gateway → EKS (stateless, easy to containerize)
    → Static asset serving → S3 + CloudFront
    → Background report jobs → Kubernetes CronJobs

  Phase 2: Migrate stateful components (harder, higher risk)
    → Sessions → ElastiCache (update app code)
    → File storage → S3 (update app code)
    → Database → RDS via DMS (careful migration with cutover window)

  Phase 3: Decompose monolith into microservices
    → Identify bounded contexts
    → Extract services one at a time
    → Use AWS API Gateway as facade during transition
```

💡 **Interview tip:** The strangler fig pattern is the correct answer for monolith migration — never try to rewrite everything at once. Extract the easiest components first (cron jobs → K8s CronJobs is trivial), build confidence, then tackle the harder stateful components. For the database, AWS DMS with CDC (Change Data Capture) enables near-zero-downtime migration — it syncs the initial 1TB, then continuously applies changes until you're ready for the final cutover. Emphasise RDS Proxy for the EKS → RDS connection problem — many pods each maintaining a connection pool quickly exhausts PostgreSQL's `max_connections`.

---

### Q135 — All Topics | System Design | Advanced

> Design an **observability and reliability platform** for a fintech company: 1M transactions/day, 7-year log retention, 1-second fraud detection, 99.999% uptime, 30 microservices, 6-month capacity planning.

📁 **Reference:** All repositories

```
Requirements breakdown:
  1M transactions/day = ~12 TPS average, ~120 TPS peak (10× average)
  7-year log retention (regulatory: PCI-DSS, SOC2)
  <1 second fraud detection alerts
  99.999% uptime = 5.26 minutes downtime per year
  Full distributed tracing across 30 microservices
  6-month capacity prediction

Platform Architecture:

─── METRICS ────────────────────────────────────────────────

  Collection:
    Prometheus (per cluster) with 15-day local retention
    Thanos Sidecar: upload blocks to S3 every 2 hours
    Thanos Store Gateway: query historical S3 data
    Thanos Querier: single PromQL endpoint for Grafana

  Thanos retention:
    S3 bucket: metrics data
    Thanos Compactor: compact + downsample
      → Raw (15s): keep 30 days
      → 5-minute: keep 1 year
      → 1-hour: keep 3 years
    Cost: ~$50/month for 3 years of metrics

  Critical metrics per service:
    RED: rate, errors, duration
    USE: utilisation, saturation, errors (for infra)
    Transaction-specific: payment_success_rate, fraud_score, settlement_lag

─── LOGGING ─────────────────────────────────────────────────

  7-year retention strategy:
    Hot tier (0-30 days): Elasticsearch on EKS (fast queries, expensive)
    Warm tier (30-180 days): Elasticsearch with cold nodes (slower, cheaper)
    Cold tier (6 months - 7 years): S3 + Athena (near-free storage, query on demand)

  Pipeline:
    App → Fluent Bit (on each node) → Kafka → Logstash → Elasticsearch + S3

  Kafka as buffer:
    → Decouples ingestion from processing
    → Handles burst: 10× normal TPS during peak
    → Enables replay if Elasticsearch is down

  Log format:
    Structured JSON mandatory: {timestamp, service, trace_id, level, msg, ...}
    PII masking before Kafka: mask card numbers, SSN, email in Fluent Bit
    Immutable audit logs: S3 Object Lock (WORM) for financial records

─── DISTRIBUTED TRACING ─────────────────────────────────────

  Tool: AWS X-Ray or Jaeger (OpenTelemetry instrumentation)

  OpenTelemetry SDK in each of 30 services:
    Auto-instrumentation: HTTP, gRPC, database calls traced automatically
    Custom spans: business logic (payment processing, fraud check)

  Trace collection:
    Service → OpenTelemetry Collector (sidecar or DaemonSet)
    Collector → Jaeger (or X-Ray) + Elasticsearch (trace storage)

  Sampling strategy:
    Head-based: sample 1% of normal transactions (12 TPS × 1% = 0.12 TPS)
    Tail-based: 100% sample all transactions > 1 second (slow path)
    100% sample all errors (never lose an error trace)

─── REAL-TIME FRAUD DETECTION (<1 second) ───────────────────

  Architecture: streaming, not batch

  Flow:
    Transaction event → Kafka → Flink/Kinesis Analytics
    → Real-time rules engine + ML model inference (< 100ms)
    → Fraud score > threshold → SNS → PagerDuty (<1 second total)

  Prometheus rule (metric-based indicator):
    alert: HighFraudScore
    expr: transaction_fraud_score > 0.85
    for: 0s    # fire immediately, no for duration
    → Feeds into AlertManager → PagerDuty in ~5 seconds

  Full <1 second path:
    Transaction occurs → Kinesis Data Streams (< 200ms)
    → Lambda (fraud model inference) (< 300ms)
    → SNS topic → PagerDuty (< 500ms total)

─── SLOs FOR 99.999% UPTIME ─────────────────────────────────

  Error budget: 5.26 minutes downtime per year

  SLIs (what to measure):
    Availability: successful_requests / total_requests > 99.999%
    Latency: p99 payment processing < 500ms
    Throughput: processed_tps > 100 during peak

  Error budget burn rate alerts:
    5% budget consumed in 1 hour → page on-call immediately
    2% budget consumed in 6 hours → ticket for next business day

  Multi-region for 99.999%:
    Active-active: us-east-1 + eu-west-1
    Global Accelerator: 30-second failover
    DynamoDB Global Tables: < 1 second replication
    Without multi-region: planned maintenance alone exceeds 99.999% budget

─── CAPACITY PLANNING (6-month forecast) ────────────────────

  Data collection:
    Prometheus: historical metrics for 6+ months
    Grafana dashboards: growth trends per service

  Forecasting:
    ARIMA or Prophet model on transaction volume
    Input: daily_transactions for past 12 months
    Output: predicted daily_transactions for next 6 months

  Per-service capacity:
    Map transaction volume → CPU/memory usage (linear regression)
    Forecast resource needs per service
    Add 30% buffer for safety margin

  Automated alerts:
    alert: CapacityWarning
    expr: predict_linear(node_memory_used[30d], 180*24*3600) > node_memory_total * 0.8
    → Fires if linear extrapolation shows memory exhaustion in 6 months

  Quarterly review:
    Finance team + Engineering: capacity vs budget
    Pre-provision EKS node groups before predicted need
    RDS instance upgrades scheduled in advance (< 30min downtime)

─── SUMMARY ARCHITECTURE ────────────────────────────────────

  Metrics:      Prometheus + Thanos + S3 (3-year retention, 50$/month)
  Logs:         Fluent Bit → Kafka → Logstash → ES (hot) + S3 (cold, 7 years)
  Traces:       OpenTelemetry → Jaeger (1% sample + 100% errors)
  Fraud alerts: Kinesis → Lambda → SNS < 1 second
  SLOs:         Error budget burn rate alerts, multi-region for 99.999%
  Capacity:     predict_linear in Prometheus + quarterly review
```

💡 **Interview tip:** This question tests breadth across all topics — don't try to go deep on everything. Structure your answer by requirement: metrics → logging → tracing → fraud detection → SLOs → capacity planning. The two differentiating answers are: (1) Thanos for long-term metrics storage with downsampling — most candidates say "Prometheus" and stop; and (2) tail-based sampling for tracing — sampling 100% of slow/error traces while sampling 1% of normal ones. For 99.999% uptime, explicitly state that single-region is impossible — you need multi-region active-active. A fintech interviewer will appreciate mentioning WORM (Write Once Read Many) via S3 Object Lock for immutable audit logs — this is a PCI-DSS requirement.

---

*DevOps/SRE Interview Questions Q91–Q135 — nawab312 GitHub repositories*
