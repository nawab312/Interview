# DevOps / SRE / Cloud Interview Questions (316–360)

> Part 8 of interview preparation questions based on your GitHub repositories.
> All questions are unique — not repeated from Q1–Q315.

---

## Table of Contents

| # | Topic | Type | Difficulty |
|---|---|---|---|
| 316 | Kubernetes | Conceptual | Advanced |
| 317 | AWS | Scenario-Based | Advanced |
| 318 | Linux / Bash | Conceptual | Advanced |
| 319 | Terraform | Scenario-Based | Advanced |
| 320 | Prometheus | Conceptual | Advanced |
| 321 | Kubernetes | Troubleshooting | Advanced |
| 322 | Git | Conceptual | Advanced |
| 323 | AWS | Conceptual | Advanced |
| 324 | Jenkins | Conceptual | Advanced |
| 325 | ELK Stack | Scenario-Based | Advanced |
| 326 | Kubernetes | Scenario-Based | Advanced |
| 327 | Linux / Bash | Troubleshooting | Advanced |
| 328 | Terraform | Conceptual | Advanced |
| 329 | GitHub Actions | Conceptual | Advanced |
| 330 | AWS | Troubleshooting | Advanced |
| 331 | Prometheus | Scenario-Based | Advanced |
| 332 | Kubernetes | Conceptual | Advanced |
| 333 | ArgoCD | Scenario-Based | Advanced |
| 334 | Linux / Bash | Scenario-Based | Advanced |
| 335 | AWS | Scenario-Based | Advanced |
| 336 | Kubernetes | Troubleshooting | Advanced |
| 337 | Grafana | Conceptual | Advanced |
| 338 | Terraform | Troubleshooting | Advanced |
| 339 | Python | Scenario-Based | Advanced |
| 340 | AWS | Conceptual | Advanced |
| 341 | Kubernetes | Scenario-Based | Advanced |
| 342 | ELK Stack | Conceptual | Advanced |
| 343 | Jenkins | Troubleshooting | Advanced |
| 344 | Linux / Bash | Conceptual | Advanced |
| 345 | AWS | Scenario-Based | Advanced |
| 346 | Kubernetes | Conceptual | Advanced |
| 347 | GitHub Actions | Scenario-Based | Advanced |
| 348 | Terraform | Scenario-Based | Advanced |
| 349 | Prometheus | Troubleshooting | Advanced |
| 350 | AWS | Conceptual | Advanced |
| 351 | Kubernetes | Scenario-Based | Advanced |
| 352 | ArgoCD | Conceptual | Advanced |
| 353 | Linux / Bash | Scenario-Based | Advanced |
| 354 | AWS | Scenario-Based | Advanced |
| 355 | Kubernetes | Conceptual | Advanced |
| 356 | Grafana | Scenario-Based | Advanced |
| 357 | Terraform | Conceptual | Advanced |
| 358 | Git | Scenario-Based | Advanced |
| 359 | AWS + Kubernetes | Scenario-Based | Advanced |
| 360 | All Topics | System Design | Advanced |

---

## Questions

---

### Q316 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `Gateway API`** — the next generation replacement for Ingress. What problems with the Ingress resource does Gateway API solve?
>
> What are the 4 core resources in Gateway API (`GatewayClass`, `Gateway`, `HTTPRoute`, `TCPRoute`)? How does the **role separation** between infrastructure provider, cluster operator, and application developer work in Gateway API?

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — Gateway API, Ingress limitations, HTTPRoute sections

---

### Q317 — AWS | Scenario-Based | Advanced

> Your company runs **microservices on ECS Fargate** and the inter-service latency has suddenly increased from 2ms to 200ms. No code changes were deployed. CPU and memory are normal.
>
> Walk me through diagnosing the latency increase — covering: ALB access logs, X-Ray traces, service discovery (Cloud Map), DNS resolution, VPC Flow Logs, and Fargate task networking.

📁 **Reference:** `nawab312/AWS` — ECS, Fargate, X-Ray, ALB, Cloud Map, latency investigation sections

---

### Q318 — Linux / Bash | Conceptual | Advanced

> Explain **Linux `scheduler`** — what is the **CFS (Completely Fair Scheduler)**? How does it decide which process runs next?
>
> What are **`nice` values** and **`ionice`**? What is the difference between **real-time scheduling policies** (`SCHED_FIFO`, `SCHED_RR`) and the normal `SCHED_OTHER`? How does Kubernetes CPU throttling interact with CFS bandwidth control?

📁 **Reference:** `nawab312/DSA` → `Linux` — CFS scheduler, nice values, CPU scheduling sections

#### Key Points to Cover:
```
CFS — Completely Fair Scheduler:
  → Default Linux scheduler since kernel 2.6.23
  → Goal: give every process a "fair" share of CPU proportional to its weight
  → Tracks vruntime (virtual runtime) per process — nanoseconds of CPU used,
    normalized by process weight
  → Always picks the process with the LOWEST vruntime to run next
  → Uses a red-black tree (self-balancing BST) as the run queue
    → leftmost node = lowest vruntime = next to run → O(log N) pick
  → Time slice is dynamic: proportional to weight, shrinks as more processes compete
  → No starvation: sleeping processes don't accumulate vruntime,
    so they don't get a huge burst on wakeup (vruntime is clamped)

nice values:
  → Range: -20 (highest priority) to +19 (lowest priority), default 0
  → Maps to a weight: nice 0 = weight 1024, each step is ~10% more/less CPU
  → nice -20 ≈ 3x more CPU than nice 0; nice +19 ≈ 5x less CPU than nice 0
  → Set with: nice -n 10 command OR renice -n 10 -p <pid>
  → Only root can set negative nice values
  → nice controls CPU scheduling ONLY — not I/O

ionice:
  → Controls I/O scheduling priority (separate from CPU scheduling)
  → Uses CFQ (Complete Fair Queuing) or BFQ I/O scheduler
  → Three classes:
      Class 1 (Realtime):   gets I/O first, can starve others
      Class 2 (Best-effort): default, takes a priority 0–7
      Class 3 (Idle):       only gets I/O when nothing else needs disk
  → Set with: ionice -c 3 -p <pid>   (make process idle I/O class)
  → Use case: backup jobs — nice +19 AND ionice -c 3 = zero impact on system

Scheduling Policies:
  SCHED_OTHER (normal):
    → CFS-based, used by virtually all userspace processes
    → Weight determined by nice value
    → Not real-time — can be preempted by any RT task

  SCHED_FIFO (real-time):
    → First In First Out — runs until it voluntarily yields or blocks
    → NO time slice — can monopolize CPU forever
    → Priority 1–99 (higher = runs first)
    → Higher priority SCHED_FIFO always preempts lower priority
    → Risk: a buggy SCHED_FIFO process at priority 99 can hang the system
    → Use: audio servers (PulseAudio), kernel threads

  SCHED_RR (real-time):
    → Round Robin variant of SCHED_FIFO
    → Same priority model (1–99) but WITH a time slice
    → Processes at same priority share CPU in round-robin fashion
    → More forgiving than FIFO — won't fully monopolize a core

  RT vs normal priority:
    → ANY RT task (SCHED_FIFO or SCHED_RR) preempts ALL SCHED_OTHER tasks
    → RT priority 1 > nice -20
    → Set with: chrt -f 50 <command>   (FIFO at priority 50)

  SCHED_DEADLINE (bonus — advanced):
    → Tasks declare: runtime, deadline, period
    → Kernel guarantees task gets `runtime` CPU within each `period`
    → Used for hard real-time workloads

Kubernetes CPU throttling + CFS Bandwidth Control:
  → Kubernetes CPU limits are implemented via CFS bandwidth control (cgroups)
  → Two cgroup parameters:
      cpu.cfs_period_us:  the period window (default 100ms = 100,000 µs)
      cpu.cfs_quota_us:   how many µs of CPU the cgroup gets per period
  → Example: limits.cpu = 500m (0.5 cores)
      quota  = 0.5 × 100,000 = 50,000 µs
      → container gets 50ms of CPU per 100ms window
  → If container uses its quota before the period ends → THROTTLED
      → all processes in cgroup sleep until next period starts
      → this shows up as: container.cpu.throttled_time in metrics
  → Throttling is NOT visible as high CPU — it looks like latency spikes
  → Common mistake: setting CPU limit = CPU request (too tight)
      → bursty JVM/Go GC gets throttled mid-GC → tail latency spikes
  → cpu.shares implements CPU requests (soft limit, weight-based)
      → maps to CFS weight — only enforced under contention
  → Limit  → hard ceiling via quota
     Request → soft weight via shares
```
> 💡 **Interview tip:** Most candidates describe CFS as "round robin with priorities" — that's wrong and interviewers notice. The key insight is the **red-black tree + vruntime** model: the scheduler always picks the leftmost node (lowest vruntime), which makes the algorithm O(log N) and mathematically fair. For Kubernetes, the killer answer is explaining that **CPU throttling looks like latency, not high CPU usage** — a container pegged at 30% CPU can still be heavily throttled if it bursts above its limit within the 100ms window. Mention `container_cpu_cfs_throttled_periods_total` as the metric to watch. That answer separates SREs who've debugged this in production from those who only read about it.

---

### Q319 — Terraform | Scenario-Based | Advanced

> You need to implement **Terraform `null_resource`** and **`terraform_data`** (Terraform 1.4+) for running local scripts as part of infrastructure provisioning.
>
> Give 3 real-world use cases where `null_resource`/`terraform_data` is the right tool:
> - Running a database migration script after RDS is created
> - Waiting for an application to become healthy after deployment
> - Generating a configuration file locally and uploading to S3
>
> Write the Terraform code for each and explain the `triggers` mechanism.

📁 **Reference:** `nawab312/Terraform` — null_resource, terraform_data, local-exec, triggers sections

---

### Q320 — Prometheus | Conceptual | Advanced

> Explain **Prometheus `AlertManager` routing** in depth. How does the routing tree work — from a firing alert to a notification?
>
> What is the difference between **`group_by`**, **`group_wait`**, **`group_interval`**, and **`repeat_interval`**? Write a routing tree that:
> - Groups alerts by `alertname` and `cluster`
> - Routes `severity=critical` to PagerDuty with 30s group_wait
> - Routes `severity=warning` to Slack with 5m group_wait
> - Suppresses all non-critical alerts during a maintenance window

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — Alertmanager routing tree, grouping, inhibition sections

---

### Q321 — Kubernetes | Troubleshooting | Advanced

> Your Kubernetes cluster shows **`PIDPressure`** condition on several nodes. Pods are being evicted and new ones cannot be scheduled on those nodes.
>
> What causes PID pressure? What is the difference between **PID limits at the node level** vs **PID limits at the pod/container level**? How do you investigate which workload is exhausting PIDs and how do you fix it?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — PID pressure, node conditions, PID limits sections

---

### Q322 — Git | Conceptual | Advanced

> Explain **`git bundle`**. What is it, when would you use it, and how does it work?
>
> Give a real-world DevOps scenario where `git bundle` solves a problem that regular `git clone` or `git push` cannot — specifically for **air-gapped environments** or **offline CI/CD pipelines**. Write the exact commands for creating, transferring, and using a bundle.

📁 **Reference:** `nawab312/CI_CD` → `Git` — git bundle, offline Git, air-gapped sections

---

### Q323 — AWS | Conceptual | Advanced

> Explain **AWS `Cognito`** — User Pools vs Identity Pools. What is the difference between **authentication** (User Pool) and **authorization** (Identity Pool)?
>
> How does Cognito integrate with an **API Gateway** for securing a REST API? Walk through the complete token flow from user login to authenticated API call, including the JWT tokens involved (`id_token`, `access_token`, `refresh_token`).

📁 **Reference:** `nawab312/AWS` — Cognito, User Pools, Identity Pools, API Gateway authorization sections

---

### Q324 — Jenkins | Conceptual | Advanced

> Explain **Jenkins `Pipeline Durability`** settings — what are `PERFORMANCE_OPTIMIZED`, `SURVIVABLE_NONATOMIC`, and `MAX_SURVIVABILITY`?
>
> What is the trade-off between **pipeline durability** and **performance**? When would you lower durability settings and what are the risks? How does pipeline durability relate to Jenkins master restarts mid-build?

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — pipeline durability, performance tuning, restart safety sections

---

### Q325 — ELK Stack | Scenario-Based | Advanced

> You need to implement **ELK Stack security** for a production deployment:
> - Enable **TLS/SSL** between all ELK components (Filebeat→Logstash→Elasticsearch→Kibana)
> - Implement **role-based access** in Elasticsearch — developers can only search logs for their own service
> - Enable **audit logging** — track who searched for what
> - Implement **field-level security** — mask PII fields (email, credit card) for non-privileged users
>
> Walk me through the complete security configuration.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Elastic Security, TLS, RBAC, field-level security sections

---

### Q326 — Kubernetes | Scenario-Based | Advanced

> You need to run a **Windows container workload** alongside Linux containers in the same Kubernetes cluster. The Windows workload is a legacy .NET Framework application that cannot be ported to Linux.
>
> Walk me through setting up **Windows nodes in EKS**, the scheduling constraints required, how Windows containers differ from Linux containers in Kubernetes, and the limitations you will face.

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md` and `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — Windows nodes, mixed OS clusters, node selectors sections

---

### Q327 — Linux / Bash | Troubleshooting | Advanced

> A production application is experiencing **sporadic connection resets**. Wireshark shows `TCP RST` packets being sent. The application is a Java service communicating with a database.
>
> Walk me through diagnosing TCP RST issues on Linux — which tools to use (`tcpdump`, `ss`, `netstat`, `/proc/net/tcp`), what causes RST packets, and how connection timeouts relate to `tcp_keepalive` settings and database connection pool `idle timeout`.

📁 **Reference:** `nawab312/DSA` → `Linux` — TCP, RST packets, keepalive, tcpdump sections

---

### Q328 — Terraform | Conceptual | Advanced

> Explain **Terraform `sensitive` values** — how do you mark a variable or output as sensitive? What does `sensitive = true` actually prevent?
>
> What are the limitations of Terraform's sensitive value handling? Where do sensitive values still appear (state file, plan output, provider logs)? How do you properly handle secrets in Terraform without them leaking into `terraform.tfstate`?

📁 **Reference:** `nawab312/Terraform` — sensitive variables, secret management, state encryption sections

---

### Q329 — GitHub Actions | Conceptual | Advanced

> Explain **GitHub Actions `permissions`** at the workflow, job, and step level. What is the **principle of least privilege** applied to GitHub Actions tokens?
>
> What are the risks of using `permissions: write-all`? List the specific permissions needed for common tasks: creating releases, publishing packages, commenting on PRs, pushing to a branch, calling OIDC endpoints. Write a workflow that uses minimum required permissions for each job.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — permissions, GITHUB_TOKEN, least privilege sections

---

### Q330 — AWS | Troubleshooting | Advanced

> Your **AWS DynamoDB** table is experiencing `ProvisionedThroughputExceededException` errors during peak hours even though you have **auto-scaling enabled**.
>
> What causes throughput exceptions even with auto-scaling? Explain **hot partitions**, **adaptive capacity**, and **burst capacity**. Walk me through diagnosing if you have a hot partition problem and the solutions.

📁 **Reference:** `nawab312/AWS` — DynamoDB, hot partitions, auto-scaling, burst capacity sections

---

### Q331 — Prometheus | Scenario-Based | Advanced

> You need to implement **Prometheus monitoring for Nginx Ingress Controller** in Kubernetes. The monitoring should track:
> - Request rate per ingress rule (not just total)
> - Upstream response time per backend service
> - 4xx and 5xx error rates per ingress
> - Active connections and connection rate
> - SSL certificate expiry for each ingress host
>
> Write the complete setup including ServiceMonitor, recording rules, and key alerting rules.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` and `nawab312/Kubernetes` — Nginx Ingress metrics, monitoring setup sections

---

### Q332 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `Lease` objects**. What are they used for in Kubernetes internals?
>
> How do leader election, node heartbeats, and component health checks use Lease objects? What happens when a node stops updating its Lease — how does the node lifecycle controller detect node failure and how long does it take to mark a node as `NotReady`?

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md` — Lease objects, node lifecycle, heartbeat sections

---

### Q333 — ArgoCD | Scenario-Based | Advanced

> You are implementing **ArgoCD notifications** to keep teams informed about deployment status. Configure notifications that:
> - Send a **Slack message** when any application transitions to `Degraded`
> - Send a **different Slack message** (with more detail) when `sync` fails
> - Post a **GitHub commit status** (pass/fail) on every sync attempt
> - Send a **PagerDuty alert** only when a production application has been `Degraded` for more than 10 minutes
>
> Write the complete ArgoCD notifications configuration.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — ArgoCD notifications, triggers, templates sections

---

### Q334 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** for **automated incident triage** that runs when a high CPU alert fires:
> - Captures a snapshot of: top 10 CPU processes, load average history, memory usage, disk I/O, network connections
> - Checks if the high CPU process is a known application or unknown
> - Looks for recent deployments by checking git logs on the server
> - Checks application logs for errors in the last 5 minutes
> - Bundles all findings into a **structured JSON incident report**
> - Posts the report to a **Slack incident channel** with formatted output
> - Creates a **local archive** of all captured data for post-incident review

📁 **Reference:** `nawab312/DSA` → `Linux` — incident triage, bash scripting, JSON, system diagnostics sections

---

### Q335 — AWS | Scenario-Based | Advanced

> Your company wants to implement **AWS `Network Firewall`** to inspect and filter traffic between VPCs and to the internet. Design a **hub-and-spoke network architecture** that:
> - Centrally inspects all **egress traffic** to the internet
> - Blocks traffic to known **malicious domains** (threat intelligence feeds)
> - Allows **east-west traffic** between spoke VPCs only on approved ports
> - Logs all **denied traffic** to S3 for security analysis
>
> Show the VPC, Transit Gateway, and Network Firewall configuration.

📁 **Reference:** `nawab312/AWS` — Network Firewall, Transit Gateway, hub-and-spoke, egress inspection sections

---

### Q336 — Kubernetes | Troubleshooting | Advanced

> Your **Kubernetes `coredns`** pods are running but DNS resolution is intermittently slow — sometimes taking 5+ seconds. This is causing application timeouts.
>
> Walk me through diagnosing CoreDNS performance issues — covering: CoreDNS metrics, `ndots` configuration impact, DNS caching, NodeLocal DNSCache, and how to properly tune DNS for high-throughput microservices environments.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — CoreDNS tuning, ndots, NodeLocal DNSCache sections

---

### Q337 — Grafana | Conceptual | Advanced

> Explain **Grafana `Variables`** in depth — what are the different variable types (`query`, `custom`, `constant`, `datasource`, `interval`, `textbox`, `adhoc`)?
>
> Write a dashboard with chained variables where:
> - First variable queries all available `cluster` labels from Prometheus
> - Second variable dynamically queries `namespace` labels **filtered by selected cluster**
> - Third variable queries `deployment` names **filtered by selected namespace**
> - All panels filter data using all three selected variables

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — dashboard variables, chained queries, templating sections

---

### Q338 — Terraform | Troubleshooting | Advanced

> Your Terraform configuration uses a **`count`-based** resource to create 5 S3 buckets. Someone adds a new bucket in the middle of the list (position 2). After running `terraform plan` you see Terraform wants to **destroy and recreate buckets 3, 4, and 5**.
>
> Explain exactly why this happens, demonstrate the problem with a code example, show the `for_each` solution, and explain how to **migrate existing `count` resources to `for_each`** using `moved` blocks without destroying infrastructure.

📁 **Reference:** `nawab312/Terraform` — count vs for_each, index shifting, moved block migration sections

---

### Q339 — Python | Scenario-Based | Advanced

> Write a **Python script** that implements a **log anomaly detector**:
> - Reads application logs in real-time from a file (tail -f style)
> - Maintains a **sliding window** of error counts (last 5 minutes)
> - Detects **sudden spikes** in error rate using a simple Z-score algorithm
> - Detects **new error patterns** not seen in the last 24 hours
> - When an anomaly is detected, extracts **sample log lines** showing the new pattern
> - Sends alerts to Slack with anomaly details and sample logs
> - Handles **log rotation** gracefully (file truncation or replacement)

📁 **Reference:** `nawab312/DSA` → `Python` — file tailing, sliding window, anomaly detection, regex sections

---

### Q340 — AWS | Conceptual | Advanced

> Explain **AWS `SQS` dead-letter queues (DLQ)** in depth. What is the `maxReceiveCount` setting and how does it work?
>
> Design a **complete error handling strategy** for an SQS-based processing system that:
> - Moves failed messages to DLQ after 3 attempts
> - Alerts when DLQ depth exceeds 10 messages
> - Has a **DLQ replay mechanism** to reprocess failed messages after fixing the bug
> - Handles **poison pill messages** (messages that always fail) separately

📁 **Reference:** `nawab312/AWS` — SQS DLQ, message retention, replay strategy sections

---

### Q341 — Kubernetes | Scenario-Based | Advanced

> You need to implement **horizontal pod autoscaling based on a custom business metric** — specifically, the number of **pending orders in an SQS queue**. When the queue depth exceeds 100 messages per pod, scale up.
>
> Walk me through the complete setup using **KEDA with SQS trigger** on EKS, including the KEDA installation, ScaledObject configuration, IAM permissions via IRSA, and testing the scaling behavior.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` and `nawab312/AWS` — KEDA, SQS scaler, IRSA sections

---

### Q342 — ELK Stack | Conceptual | Advanced

> Explain **Elasticsearch `cross-cluster search (CCS)`** and **`cross-cluster replication (CCR)`**.
>
> What is the difference between CCS and CCR? When would you use each? Design an architecture where:
> - Logs from 3 regions (US, EU, AP) are stored in regional Elasticsearch clusters
> - Ops team needs to **search across all regions** from a single Kibana instance
> - EU cluster needs a **read-only replica** in US for disaster recovery

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — cross-cluster search, cross-cluster replication sections

---

### Q343 — Jenkins | Troubleshooting | Advanced

> Your Jenkins is experiencing **build queue buildup** — there are 150 jobs waiting in the queue. Executors are all busy. Adding more executors doesn't help because the builds are stuck waiting for a **shared resource lock**.
>
> Explain **Jenkins `Lockable Resources` plugin** — how does it work, what causes deadlocks, and how do you resolve a deadlock situation where two pipelines are holding resources that each other needs?

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — Lockable Resources, build queue, deadlock resolution sections

---

### Q344 — Linux / Bash | Conceptual | Advanced

> Explain **Linux `shared libraries`** — what are `.so` files, how does **dynamic linking** work, and what is `LD_LIBRARY_PATH`?
>
> What is the difference between `ldd` and `ldconfig`? What causes `error while loading shared libraries: libxxx.so.x: cannot open shared object file`? How does this relate to **container image layers** and why do containers avoid this problem?

📁 **Reference:** `nawab312/DSA` → `Linux` and `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md` — shared libraries, dynamic linking, container isolation sections

---

### Q345 — AWS | Scenario-Based | Advanced

> Your team is implementing **AWS `AppConfig`** for feature flag management and dynamic configuration for a production microservice running on ECS.
>
> Walk me through:
> - Setting up AppConfig with a deployment strategy
> - How the application **polls for config changes** without restarting
> - Implementing a **rollout strategy** (linear/exponential) for config changes
> - Setting up **CloudWatch alarms** as rollback triggers if config change causes errors
> - How AppConfig differs from **SSM Parameter Store** for this use case

📁 **Reference:** `nawab312/AWS` — AppConfig, feature flags, dynamic configuration, deployment strategies sections

---

### Q346 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `MutatingAdmissionWebhook`** for **sidecar injection**. How does Istio's sidecar injector work?
>
> Write a MutatingWebhookConfiguration that automatically injects a **logging sidecar container** into every Pod in namespaces labeled `inject-logging=true`. The webhook should add the sidecar only if it is not already present. What are the failure modes if the webhook is unavailable?

📁 **Reference:** `nawab312/Kubernetes` → `10_CRDs_EXTENSIBILITY.md` and `11_ISTIO_SERVICE_MESH.md` — MutatingWebhook, sidecar injection, Istio sections

---

### Q347 — GitHub Actions | Scenario-Based | Advanced

> You need to implement **GitHub Actions for GitOps** — when infrastructure code changes are merged to `main`, automatically update the ArgoCD application manifests in a **separate GitOps repository**.
>
> The workflow should:
> - Build and push a new Docker image with git SHA tag
> - **Clone the GitOps repo**, update the image tag in the Helm values file
> - **Commit and push** the change to the GitOps repo
> - Create a **pull request** in the GitOps repo for the production update (requiring manual merge)
> - Comment on the original PR with the GitOps PR link

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` and `ArgoCD` — GitOps workflow, image promotion, cross-repo automation sections

---

### Q348 — Terraform | Scenario-Based | Advanced

> You need to implement **Terraform `postcondition`** checks for complex validation after resource creation.
>
> Write postconditions that validate:
> - After creating an RDS instance, verify it is in a **Multi-AZ configuration**
> - After creating an ALB, verify it has **at least 2 subnets in different AZs**
> - After creating an ECS service, verify the **desired count matches** the running count within 5 minutes
> - After creating an S3 bucket, verify **versioning and encryption** are enabled
>
> Explain what happens when a postcondition fails and how it differs from a `check` block.

📁 **Reference:** `nawab312/Terraform` — postcondition, precondition, check blocks, validation sections

---

### Q349 — Prometheus | Troubleshooting | Advanced

> Your Prometheus metrics show a **sudden gap** — all metrics have a gap of 15 minutes between 14:00 and 14:15. After 14:15 everything looks normal.
>
> Walk me through diagnosing what caused the metrics gap — covering: Prometheus WAL recovery, storage issues, network partition from targets, Prometheus restart, and how to determine which cause is responsible for this specific gap.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — WAL, TSDB recovery, metrics gap diagnosis sections

---

### Q350 — AWS | Conceptual | Advanced

> Explain **AWS `CloudFront`** in depth — what is an **origin**, **distribution**, **behavior**, and **cache policy**?
>
> What is the difference between **`TTL`**, **`Cache-Control`** headers, and **`invalidation`**? How do you implement a secure **signed URLs** pattern for private S3 content distribution? When would you use CloudFront vs API Gateway as the entry point for an API?

📁 **Reference:** `nawab312/AWS` — CloudFront, CDN, signed URLs, cache policies sections

---

### Q351 — Kubernetes | Scenario-Based | Advanced

> You are implementing a **zero-trust networking model** inside your Kubernetes cluster using **Cilium** (eBPF-based CNI) instead of standard NetworkPolicy.
>
> Explain what **eBPF** is and why Cilium uses it instead of iptables. What capabilities does Cilium provide beyond standard NetworkPolicy (L7 policies, identity-based policies, observability with Hubble)? Write a Cilium `NetworkPolicy` that enforces L7 HTTP method restrictions.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` and `07_SECURITY.md` — Cilium, eBPF, L7 policies, zero-trust networking sections

---

### Q352 — ArgoCD | Conceptual | Advanced

> Explain **ArgoCD `Diffing Customization`**. What is the `ignoreDifferences` field at the Application level vs the global `resource.customizations` in ArgoCD ConfigMap?
>
> When would you use global customizations vs per-application `ignoreDifferences`? Write configurations that globally ignore:
> - `caBundle` field in all `MutatingWebhookConfiguration` resources
> - `status` field in all custom resources
> - `rollingUpdate` strategy changes in all Deployments

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — diffing customization, ignoreDifferences, global resource customizations sections

---

### Q353 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that implements a **rolling restart** for a fleet of application servers:
> - Reads server list from inventory file (grouped by `primary` and `secondary` roles)
> - Restarts `secondary` servers first, then `primary` servers
> - For each server: drain connections (wait for active connections < 5), restart service, wait for health check to pass before proceeding to next server
> - If a server fails health check after restart, **pause and alert** (do not continue)
> - Maintains **rolling log** showing progress and each server's status
> - Has `--dry-run` mode that shows what would happen without executing

📁 **Reference:** `nawab312/DSA` → `Linux` — rolling restart, bash scripting, health checks, drain sections

---

### Q354 — AWS | Scenario-Based | Advanced

> Your **AWS EKS cluster** needs to integrate with **AWS Service Catalog** so that developers can self-service provision approved infrastructure (RDS, ElastiCache, SQS) for their applications without direct AWS console access.
>
> Design the workflow where:
> - Developer requests a new RDS instance via a Kubernetes **ServiceInstance** object (AWS Controllers for Kubernetes / ACK)
> - The request goes through an **approval workflow**
> - Infrastructure is provisioned and **connection details injected** as a Kubernetes Secret
> - Deprovisioning happens when the ServiceInstance is deleted

📁 **Reference:** `nawab312/AWS` and `nawab312/Kubernetes` — ACK (AWS Controllers for Kubernetes), Service Catalog, self-service provisioning sections

---

### Q355 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `Service` type `ExternalName`**. How does it work and what are its use cases?
>
> Also explain **`headless services`** (ClusterIP: None) — when would you use them and how do they work differently from regular ClusterIP services? How does a headless service enable direct pod-to-pod communication for StatefulSets?

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — ExternalName, headless services, DNS for StatefulSets sections

---

### Q356 — Grafana | Scenario-Based | Advanced

> You need to implement **Grafana `alerting` with multiple data sources** — some alerts are based on Prometheus metrics, some on Loki log queries, and some on CloudWatch metrics.
>
> Walk me through setting up:
> - A **Loki-based alert** that fires when error log rate exceeds 10/minute
> - A **multi-datasource alert** that uses both Prometheus (CPU) and Loki (error logs) with `AND` logic
> - **Alert grouping** so related alerts are sent together
> - **Mute timings** to suppress alerts during scheduled maintenance windows

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — multi-datasource alerting, Loki alerts, mute timings sections

---

### Q357 — Terraform | Conceptual | Advanced

> Explain **Terraform `graph`** command and the **dependency graph**. How does Terraform determine the order of resource creation and deletion?
>
> What is the difference between **implicit dependencies** (via references) and **explicit dependencies** (via `depends_on`)? When does `depends_on` cause unnecessary rebuilds and how do you avoid it? Give an example where `depends_on` is absolutely necessary vs where it causes performance problems.

📁 **Reference:** `nawab312/Terraform` — dependency graph, depends_on, implicit vs explicit dependencies sections

---

### Q358 — Git | Scenario-Based | Advanced

> Your team has **diverged branches** — `main` and `feature/payment-v2` have been diverging for 3 months. The feature branch has 150 commits and `main` has 200 new commits. You need to merge them.
>
> Walk me through your strategy — when would you use **merge**, **rebase**, or **squash merge**? How do you handle **merge conflicts at scale** when there are conflicts in 30 different files? What tools help visualize and resolve complex merges?

📁 **Reference:** `nawab312/CI_CD` → `Git` — large-scale merge, conflict resolution, merge strategies sections

---

### Q359 — AWS + Kubernetes | Scenario-Based | Advanced

> Your EKS application needs to process **video files uploaded to S3** — each video needs transcoding (takes 10-30 minutes). The workload is bursty — 0 to 1000 videos/hour.
>
> Design the complete architecture using:
> - **KEDA** to scale workers based on SQS queue depth
> - **Spot instances** for cost optimization with graceful handling of interruptions
> - **Checkpoint/resume** capability so interrupted jobs can restart from where they left off
> - Worker pods that handle **SIGTERM** gracefully to save progress before termination

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` and `nawab312/AWS` — KEDA, Spot instances, S3, SQS, job checkpointing sections

---

### Q360 — All Topics | System Design | Advanced

> You are designing a **developer platform (Internal Developer Platform - IDP)** using **Backstage** or equivalent for a company with 500 engineers across 20 teams. The platform must provide:
> - **Service catalog** — discoverability of all 200 microservices with owners, runbooks, SLOs
> - **Self-service infrastructure** — developers can provision approved infra via templates
> - **Golden paths** — opinionated CI/CD templates that encode best practices
> - **Developer portal** — unified view of deployments, alerts, on-call status, cost
> - **DORA metrics** — deployment frequency, lead time, MTTR, change failure rate
>
> Design the complete platform architecture and explain how each tool in your stack (ArgoCD, GitHub Actions, Terraform, Prometheus, Grafana) integrates into it.

📁 **Reference:** All repositories — Internal Developer Platform, platform engineering, DORA metrics design

---

## Key Topics Coverage (Q316–Q360)

| Topic | Questions |
|---|---|
| Kubernetes | Q316, Q321, Q326, Q332, Q336, Q341, Q346, Q351, Q355 |
| AWS | Q317, Q323, Q330, Q335, Q340, Q345, Q350, Q354, Q359 |
| Terraform | Q319, Q328, Q338, Q348, Q357 |
| Prometheus | Q320, Q331, Q349 |
| Grafana | Q337, Q356 |
| ELK Stack | Q325, Q342 |
| Linux / Bash | Q318, Q327, Q334, Q344, Q353 |
| ArgoCD | Q333, Q352 |
| Jenkins | Q324, Q343 |
| GitHub Actions | Q329, Q347 |
| Git | Q322, Q358 |
| Python | Q339 |
| System Design | Q360 |

---

## New Concepts Introduced in This Set

### Kubernetes
- Gateway API — GatewayClass, HTTPRoute, role separation
- PID pressure — node condition, PID limits at pod level
- Windows containers in EKS — mixed OS clusters
- `Lease` objects — node heartbeats, failure detection timing
- Cilium + eBPF — L7 NetworkPolicy, Hubble observability
- MutatingWebhook — sidecar injection implementation
- ExternalName services — use cases
- Headless services — StatefulSet DNS
- KEDA + SQS on EKS — complete ScaledObject setup
- CoreDNS performance — NodeLocal DNSCache, ndots tuning

### AWS
- ECS Fargate inter-service latency diagnosis
- DynamoDB hot partitions — adaptive capacity, burst capacity
- Network Firewall — hub-and-spoke, egress inspection
- AWS Cognito — User Pool vs Identity Pool, JWT token flow
- SQS DLQ — maxReceiveCount, replay mechanism, poison pills
- AppConfig — dynamic config, rollback triggers
- CloudFront — signed URLs, cache policies, CDN vs API Gateway
- ACK (AWS Controllers for Kubernetes) — self-service K8s provisioning
- S3 video processing — KEDA + Spot + checkpoint/resume

### Terraform
- `null_resource` / `terraform_data` — 3 use cases with triggers
- `sensitive` values — limitations, state file exposure
- count-to-for_each migration — moved blocks
- `postcondition` vs `check` block — failure behavior difference
- `graph` command — implicit vs explicit depends_on performance impact
- Policy as code with OPA/Conftest in CI

### Prometheus
- Alertmanager routing tree — group_by, group_wait, group_interval
- Metrics gap diagnosis — WAL recovery vs network partition vs restart
- Nginx Ingress monitoring — per-ingress-rule metrics
- Redis monitoring (new angle — hit rate formula, eviction alerts)

### Linux
- CFS scheduler — nice values, SCHED_FIFO vs SCHED_RR
- TCP RST packets — tcpdump diagnosis, keepalive settings
- Shared libraries — .so files, ldd, ldconfig, LD_LIBRARY_PATH
- `git bundle` — air-gapped environments

### Git
- `git bundle` — offline CI/CD
- `git rerere` (new angle — when most useful)
- Conventional commits — full automation pipeline (new angle)
- Large diverged branch merge — conflict resolution at scale

### ArgoCD
- Notifications — triggers, templates, GitHub commit status
- Diffing customization — global vs per-application ignoreDifferences
- Webhook sync failures (new angle — admission controller timeouts)

### Grafana
- Variables — chained queries, all 7 variable types
- Multi-datasource alerting — Prometheus + Loki AND logic
- Mute timings — maintenance window suppression

---

*Part 8 of DevOps/SRE Interview Questions — nawab312 GitHub repositories*
*Total question bank: Q1–Q360*
