# DevOps / SRE / Cloud Interview Questions (406–450)

> Part 10 — FINAL set of interview preparation questions based on your GitHub repositories.
> Focused on filling remaining gaps. All questions unique — not repeated from Q1–Q405.

---

## Table of Contents

| # | Topic | Type | Difficulty |
|---|---|---|---|
| 406 | Kubernetes | Conceptual | Advanced |
| 407 | AWS | Conceptual | Advanced |
| 408 | Linux / Bash | Conceptual | Advanced |
| 409 | Prometheus | Conceptual | Advanced |
| 410 | Kubernetes | Scenario-Based | Advanced |
| 411 | AWS | Scenario-Based | Advanced |
| 412 | ArgoCD | Scenario-Based | Advanced |
| 413 | Linux / Bash | Troubleshooting | Advanced |
| 414 | Kubernetes | Conceptual | Advanced |
| 415 | AWS | Conceptual | Advanced |
| 416 | Prometheus | Scenario-Based | Advanced |
| 417 | Kubernetes | Troubleshooting | Advanced |
| 418 | AWS | Scenario-Based | Advanced |
| 419 | Grafana | Conceptual | Advanced |
| 420 | Linux / Bash | Scenario-Based | Advanced |
| 421 | Kubernetes | Conceptual | Advanced |
| 422 | AWS | Conceptual | Advanced |
| 423 | ArgoCD | Conceptual | Advanced |
| 424 | Terraform | Conceptual | Advanced |
| 425 | Kubernetes | Scenario-Based | Advanced |
| 426 | AWS | Scenario-Based | Advanced |
| 427 | Linux / Bash | Conceptual | Advanced |
| 428 | Prometheus | Conceptual | Advanced |
| 429 | Kubernetes | Troubleshooting | Advanced |
| 430 | AWS | Troubleshooting | Advanced |
| 431 | Grafana | Scenario-Based | Advanced |
| 432 | Kubernetes | Conceptual | Advanced |
| 433 | AWS | Scenario-Based | Advanced |
| 434 | Linux / Bash | Scenario-Based | Advanced |
| 435 | Kubernetes | Scenario-Based | Advanced |
| 436 | AWS | Conceptual | Advanced |
| 437 | Prometheus | Troubleshooting | Advanced |
| 438 | Kubernetes | Conceptual | Advanced |
| 439 | AWS | Scenario-Based | Advanced |
| 440 | Linux / Bash | Conceptual | Advanced |
| 441 | Kubernetes | Scenario-Based | Advanced |
| 442 | AWS | Conceptual | Advanced |
| 443 | ArgoCD | Troubleshooting | Advanced |
| 444 | Grafana | Conceptual | Advanced |
| 445 | Kubernetes | Troubleshooting | Advanced |
| 446 | AWS | Scenario-Based | Advanced |
| 447 | Linux / Bash | Scenario-Based | Advanced |
| 448 | Kubernetes | Conceptual | Advanced |
| 449 | AWS + Kubernetes | Scenario-Based | Advanced |
| 450 | All Topics | System Design | Advanced |

---

## Questions

---

### Q406 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `Cluster API (CAPI)`**. What problem does it solve and how does it work?
>
> What is a **Management Cluster** vs a **Workload Cluster**? What are `Cluster`, `Machine`, `MachineDeployment`, and `MachineHealthCheck` CRDs? How does CAPI enable **declarative cluster lifecycle management** — creation, scaling, upgrades, and deletion of Kubernetes clusters using Kubernetes itself?

📁 **Reference:** `nawab312/Kubernetes` → `09_CLUSTER_OPERATIONS.md` — Cluster API, cluster lifecycle management sections

---

### Q407 — AWS | Conceptual | Advanced

> Explain **AWS `Gateway Load Balancer (GWLB)`**. How does it differ from ALB and NLB?
>
> What is a **GWLB Endpoint** and how does traffic flow through it? Design an architecture using GWLB to insert **third-party network appliances** (firewall, IDS/IPS) transparently into traffic between VPCs — without changing routing on the application side.

📁 **Reference:** `nawab312/AWS` — GWLB, network appliances, transparent traffic inspection sections

---

### Q408 — Linux / Bash | Conceptual | Advanced

> Explain **Linux `perf`** tool — what is it and what can it measure?
>
> Walk through how you would use `perf` to diagnose a production performance issue:
> - `perf top` for real-time CPU profiling
> - `perf record` + `perf report` for flame graph generation
> - `perf stat` for CPU counter analysis
> - `perf trace` for syscall tracing
>
> What is a **flame graph** and how do you interpret it? How does `perf` compare to `strace` for performance analysis?

📁 **Reference:** `nawab312/DSA` → `Linux` — perf tool, profiling, flame graphs, performance analysis sections

---

### Q409 — Prometheus | Conceptual | Advanced

> Explain **OpenTelemetry** and its relationship with Prometheus. What is the **OpenTelemetry Collector** and how does it fit into a modern observability stack?
>
> What is the difference between **OTLP (OpenTelemetry Protocol)** and **Prometheus exposition format**? How does the OpenTelemetry Collector act as a **universal observability pipeline** — receiving from multiple sources and exporting to multiple backends (Prometheus, Jaeger, Loki, Datadog)?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — OpenTelemetry, OTLP, observability pipeline sections

---

### Q410 — Kubernetes | Scenario-Based | Advanced

> You need to run **WebAssembly (WASM) workloads** on Kubernetes alongside container workloads. Explain what **WasmEdge** and **`containerd-wasm-shims`** provide.
>
> What is a **`RuntimeClass`** for WASM and how does it differ from the standard runc or gVisor runtime? What are the benefits of WASM for serverless-style workloads (startup time, size, security isolation)? Write the Kubernetes configuration to schedule a WASM workload.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md` and `13_DOCKER_CONTAINER_FUNDAMENTALS.md` — RuntimeClass, WASM, container runtimes sections

---

### Q411 — AWS | Scenario-Based | Advanced

> Your company is implementing **AWS `MSK (Managed Streaming for Apache Kafka)`**. Design a production MSK cluster that:
> - Has **3 brokers across 3 AZs** for HA
> - Uses **MSK Serverless** vs provisioned — when to choose each
> - Implements **mTLS authentication** for producers and consumers
> - Has **topic-level ACLs** restricting which clients can produce/consume
> - Monitors consumer group lag with CloudWatch
> - Has **auto-scaling** for broker storage
>
> Walk me through the complete setup.

📁 **Reference:** `nawab312/AWS` — MSK, Kafka, mTLS, consumer group lag, auto-scaling sections

---

### Q412 — ArgoCD | Scenario-Based | Advanced

> You need to integrate **ArgoCD with HashiCorp Vault** for secrets management using the **`argocd-vault-plugin`**. Walk me through:
> - How the plugin works (template substitution vs sidecar approach)
> - Configuring Vault AppRole authentication for ArgoCD
> - Writing Kubernetes manifests with `<path:secret/data/myapp#key>` placeholders
> - Handling **secret rotation** — how does ArgoCD re-render manifests when Vault secrets change?
> - Security considerations — what access does the plugin need?

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — Vault plugin, secret management, GitOps secrets sections

---

### Q413 — Linux / Bash | Troubleshooting | Advanced

> A production application is showing **random memory corruption** — data structures are getting corrupted in memory with no apparent pattern. The application uses shared memory (`POSIX shm_open`).
>
> Walk me through debugging memory corruption on Linux:
> - Using **Valgrind** (memcheck) to detect memory errors
> - Using **AddressSanitizer (ASAN)** for faster detection
> - Using **`/proc/PID/maps`** to inspect memory layout
> - What causes shared memory corruption specifically
> - How `mprotect()` can help isolate corruption

📁 **Reference:** `nawab312/DSA` → `Linux` — memory debugging, Valgrind, ASAN, shared memory sections

---

### Q414 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `VolumeSnapshot`** and **`VolumeSnapshotClass`**. How do they work with CSI drivers?
>
> Walk through the complete workflow:
> - Creating a VolumeSnapshot from a PVC
> - Restoring a PVC from a VolumeSnapshot
> - Scheduling **automated snapshots** using a CronJob
> - Cross-namespace snapshot restore
> - What happens to the snapshot if the original PVC is deleted?

📁 **Reference:** `nawab312/Kubernetes` → `04_STORAGE.md` — VolumeSnapshot, CSI snapshots, backup workflow sections

---

### Q415 — AWS | Conceptual | Advanced

> Explain **AWS `Batch`** — when would you use it over Lambda, ECS, or EKS for batch processing?
>
> What is a **Compute Environment** (managed vs unmanaged), **Job Queue**, and **Job Definition**? How does AWS Batch handle **Spot instance interruptions** for long-running batch jobs? Design a batch pipeline that processes 10,000 genomics files in parallel with dependency ordering between job steps.

📁 **Reference:** `nawab312/AWS` — AWS Batch, compute environments, job queues, Spot handling sections

---

### Q416 — Prometheus | Scenario-Based | Advanced

> You need to implement **Prometheus `Alertmanager` high availability** — running multiple Alertmanager instances so that if one fails, alerts are still delivered.
>
> Explain how Alertmanager **gossip protocol** works for HA clustering. What is the `--cluster.peer` flag? How does Alertmanager prevent **duplicate notifications** when running in HA mode? Write the Kubernetes StatefulSet configuration for a 3-replica Alertmanager HA cluster.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — Alertmanager HA, gossip, deduplication sections

---

### Q417 — Kubernetes | Troubleshooting | Advanced

> Your Kubernetes cluster is running **Istio service mesh** and you are seeing `503 Service Unavailable` errors between services that were working yesterday. No changes were deployed.
>
> Walk me through diagnosing Istio-specific 503 errors — covering: Envoy proxy logs, `istioctl proxy-status`, `istioctl analyze`, mTLS policy mismatches, sidecar injection issues, and DestinationRule/VirtualService misconfiguration.

📁 **Reference:** `nawab312/Kubernetes` → `11_ISTIO_SERVICE_MESH.md` — Istio troubleshooting, Envoy, mTLS sections

---

### Q418 — AWS | Scenario-Based | Advanced

> Your company needs to implement **AWS `IPv6`** for a new EKS cluster because the corporate IPv4 address space is exhausted. Walk me through:
> - Enabling **dual-stack** (IPv4 + IPv6) on the VPC and subnets
> - Configuring EKS for **IPv6-only pod networking**
> - How the **VPC CNI** assigns IPv6 addresses to pods
> - Security group rules for IPv6
> - Which AWS services **do not support IPv6** yet (common gotchas)
> - How **egress-only internet gateway** replaces NAT for IPv6

📁 **Reference:** `nawab312/AWS` and `nawab312/Kubernetes` — IPv6, dual-stack EKS, egress-only IGW sections

---

### Q419 — Grafana | Conceptual | Advanced

> Explain **Grafana `Plugins`** — what are the 3 types of plugins (panel plugins, data source plugins, app plugins)?
>
> How do you install and manage plugins in a Kubernetes-deployed Grafana? What is the **Grafana plugin development framework** and what APIs does it expose? Give examples of popular community plugins that extend Grafana significantly (e.g., `grafana-piechart-panel`, `grafana-worldmap-panel`, `grafana-clock-panel`).

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — plugins, plugin management, installation sections

---

### Q420 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that implements **automatic kernel parameter tuning** for a high-performance application server:
> - Detects server role (web server, database, cache, message queue) from a config file
> - Applies **role-specific sysctl settings** (network buffers, file descriptors, VM settings)
> - Verifies current values before changing and **rolls back** if application health check fails after tuning
> - Makes settings **persistent** across reboots via `/etc/sysctl.d/`
> - Generates a **before/after comparison report**
> - Validates that settings are within safe bounds before applying

📁 **Reference:** `nawab312/DSA` → `Linux` — sysctl, kernel tuning, network parameters, vm.swappiness sections

---

### Q421 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `ImagePolicyWebhook`** admission controller. How does it enforce image policies cluster-wide?
>
> What is the difference between `ImagePolicyWebhook` and **`OPA Gatekeeper`** for image policy enforcement? Write an OPA Gatekeeper `ConstraintTemplate` and `Constraint` that:
> - Blocks pods using `latest` tag
> - Only allows images from approved registries (`123456.dkr.ecr.us-east-1.amazonaws.com`)
> - Requires all images to have a digest (`@sha256:...`) instead of mutable tags

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md` — image policy, OPA Gatekeeper, admission control sections

---

### Q422 — AWS | Conceptual | Advanced

> Explain **AWS `EventBridge Pipes`** — how does it differ from standard EventBridge rules?
>
> What is the **enrichment** step in a Pipe and when would you use it? Design a Pipe that:
> - Sources from a **DynamoDB stream** (new order events)
> - Filters for only `INSERT` events
> - Enriches with additional customer data by calling a **Lambda**
> - Targets an **SQS queue** for order processing
>
> How does this compare to a Lambda-based fan-out approach?

📁 **Reference:** `nawab312/AWS` — EventBridge Pipes, DynamoDB streams, event enrichment sections

---

### Q423 — ArgoCD | Conceptual | Advanced

> Explain **ArgoCD `Generators`** in ApplicationSet — beyond the basics. Deep dive into:
> - **`SCM Provider` generator** — auto-create apps for every repo in a GitHub org
> - **`Pull Request` generator** — create preview environments for every open PR
> - **`Cluster Decision Resource` generator** — create apps based on a custom CR
> - **`Plugin` generator** — call an external API to generate app parameters
>
> Give a real-world example for the Pull Request generator — every PR gets a dedicated namespace with the application deployed for review.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — ApplicationSet generators, SCM provider, PR environments sections

---

### Q424 — Terraform | Conceptual | Advanced

> Explain **Terraform `import` blocks** (Terraform 1.5+) vs the older `terraform import` CLI command.
>
> What is **`terraform generate config`** (experimental in 1.5, stable in 1.6+) and how does it auto-generate `.tf` configuration for imported resources? Walk through importing an **existing complex AWS infrastructure** (VPC with subnets, route tables, NACLs) using import blocks and generated config.

📁 **Reference:** `nawab312/Terraform` — import blocks, config generation, Terraform 1.5+ features sections

---

### Q425 — Kubernetes | Scenario-Based | Advanced

> You need to implement **Kubernetes `NetworkPolicy` for a service mesh environment** where Istio is already providing mTLS. What is the relationship between NetworkPolicy and Istio `AuthorizationPolicy`?
>
> Write both a Kubernetes `NetworkPolicy` and an Istio `AuthorizationPolicy` that together enforce:
> - Only `frontend` service can call `backend` service on `/api/*` paths
> - Only HTTP GET and POST methods allowed (no DELETE)
> - Only authenticated service accounts can make requests
> - All other traffic denied at both the network AND application layer

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` and `11_ISTIO_SERVICE_MESH.md` — NetworkPolicy + Istio AuthorizationPolicy, defense in depth sections

---

### Q426 — AWS | Scenario-Based | Advanced

> You need to implement **AWS `Glue`** for an ETL pipeline that:
> - Crawls S3 data lake and updates **Glue Data Catalog** automatically
> - Transforms raw JSON logs into **Parquet format** with partitioning by date
> - Handles **schema evolution** (new fields added to source data)
> - Runs on a **schedule** triggered by new S3 objects
> - Failed jobs should **retry and alert** via SNS
>
> Walk me through the Glue job, crawler, and trigger setup.

📁 **Reference:** `nawab312/AWS` — AWS Glue, ETL, Data Catalog, S3 data lake sections

---

### Q427 — Linux / Bash | Conceptual | Advanced

> Explain **Linux `BPF (Berkeley Packet Filter)`** and **`eBPF`** — what is the difference and why is eBPF revolutionary?
>
> What are **eBPF programs** and where can they be attached (tracepoints, kprobes, uprobes, network hooks)? What are **eBPF maps** used for? Give 5 real-world DevOps/SRE use cases for eBPF — including how tools like **`bpftrace`**, **`bcc`**, and **Cilium** use it. Why is eBPF safer than kernel modules?

📁 **Reference:** `nawab312/DSA` → `Linux` — eBPF, BPF programs, kernel observability sections

---

### Q428 — Prometheus | Conceptual | Advanced

> Explain **Prometheus `agent mode`** (introduced in Prometheus 2.32). What problem does it solve?
>
> How does agent mode differ from a full Prometheus server — what features are disabled (no local query, no alerting, no recording rules)? When would you deploy Prometheus in agent mode vs full mode? How does it integrate with **`remote_write`** for centralized metrics collection in a multi-cluster environment?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — agent mode, remote_write, multi-cluster sections

---

### Q429 — Kubernetes | Troubleshooting | Advanced

> Your Kubernetes **`kubectl logs`** command is returning `Error from server: context deadline exceeded` for a specific Pod. The Pod is running and healthy. Other kubectl commands work fine.
>
> What causes log retrieval to fail independently of Pod health? Walk me through diagnosing — covering: kubelet log endpoint, node network connectivity to API server, log size limits, container runtime log rotation, and how the API server proxies log requests to the kubelet.

📁 **Reference:** `nawab312/Kubernetes` → `14_TROUBLESHOOTING.md` — kubectl logs, kubelet, log streaming sections

---

### Q430 — AWS | Troubleshooting | Advanced

> Your **AWS S3 bucket** is throwing `SlowDown: Please reduce your request rate` errors. The application is making 10,000 requests/second to the same bucket.
>
> Explain S3's **request rate limits** and the **prefix-based partitioning** model. How do you redesign the key naming strategy to distribute load across partitions? What S3 features help with high-throughput workloads (S3 Transfer Acceleration, multipart upload, byte-range fetches)?

📁 **Reference:** `nawab312/AWS` — S3 performance, request rate limits, prefix partitioning sections

---

### Q431 — Grafana | Scenario-Based | Advanced

> You need to implement **Grafana `SLO dashboards`** using the **Grafana SLO plugin** or custom PromQL. Build a complete SLO dashboard that shows:
> - **Rolling 30-day availability** with SLO target line (99.9%)
> - **Error budget remaining** (absolute minutes and percentage)
> - **Error budget burn rate** over 1h, 6h, 24h windows
> - **SLO compliance history** — was the SLO met each of the last 30 days?
> - **Toil indicator** — how much manual intervention happened this month
>
> Write the PromQL queries and explain the dashboard layout.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` and `Prometheus` — SLO dashboard, error budget visualization sections

---

### Q432 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `Multi-tenancy` patterns** — what are the 4 levels of isolation (namespace, cluster, node, VM)?
>
> Compare **soft multi-tenancy** (shared cluster with RBAC + NetworkPolicy + ResourceQuota) vs **hard multi-tenancy** (dedicated clusters per tenant). What tools implement hard multi-tenancy within a single cluster — **vcluster**, **Capsule**, **HNC (Hierarchical Namespace Controller)**? When is each approach appropriate?

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md` and `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — multi-tenancy, isolation levels sections

---

### Q433 — AWS | Scenario-Based | Advanced

> You are implementing **AWS `EMR (Elastic MapReduce)`** for big data processing. Your team needs to run **Spark jobs** that process 50TB of data daily from S3.
>
> Walk me through:
> - EMR on EC2 vs **EMR Serverless** vs **EMR on EKS** — when to use each
> - Cost optimization using **Spot instances for task nodes**
> - Auto-scaling for EMR clusters based on YARN metrics
> - Storing **Spark job results** back to S3 in optimal format (Parquet + Delta Lake)
> - Monitoring job performance with **EMR Studio**

📁 **Reference:** `nawab312/AWS` — EMR, Spark, Spot instances, EMR Serverless sections

---

### Q434 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that implements a **Linux system hardening checker** based on CIS benchmarks:
> - Checks: SSH configuration (PasswordAuth disabled, root login disabled, max auth tries)
> - Checks: Password policy (min length, complexity, expiry)
> - Checks: Filesystem mounts (noexec on /tmp, nosuid on removable media)
> - Checks: Network settings (IP forwarding disabled, ICMP redirects disabled)
> - Checks: Audit daemon running and logging to remote syslog
> - Outputs a **compliance score** (pass/fail per check with CIS benchmark reference)
> - Generates a **remediation script** for all failed checks

📁 **Reference:** `nawab312/DSA` → `Linux` — CIS benchmarks, system hardening, security configuration sections

---

### Q435 — Kubernetes | Scenario-Based | Advanced

> You are implementing **Kubernetes `cost optimization`** using **Goldilocks** and **VPA recommendations** to right-size all workloads in the cluster.
>
> Walk me through:
> - Installing Goldilocks and how it uses VPA in `Off` mode
> - Interpreting Goldilocks recommendations (Guaranteed vs Burstable QoS)
> - How to **automate right-sizing** based on recommendations
> - Implementing **namespace-level cost budgets** with alerts
> - Using **Kubecost** for per-team cost attribution and chargeback
> - What metrics to track for ongoing cost efficiency

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — Goldilocks, VPA, cost optimization, Kubecost sections

---

### Q436 — AWS | Conceptual | Advanced

> Explain **AWS `Graviton` processors** and their significance for DevOps/SRE.
>
> What are the **performance and cost benefits** of Graviton3 vs x86? What are the **compatibility considerations** when migrating containerized workloads to Graviton (architecture-specific binaries, multi-arch images)? How do you implement a **gradual migration** of EKS workloads from x86 to Graviton using node groups and pod affinity?

📁 **Reference:** `nawab312/AWS` and `nawab312/Kubernetes` — Graviton, ARM64, multi-arch, EKS node groups sections

---

### Q437 — Prometheus | Troubleshooting | Advanced

> Your Prometheus `histogram_quantile()` queries are returning **inaccurate percentiles** — the p99 latency shown in Grafana is significantly lower than what users are experiencing.
>
> What causes histogram quantile inaccuracy? Explain **bucket boundary effects**, **interpolation assumptions**, and why having buckets that are **too wide** causes large errors. How do you choose optimal histogram buckets? What is **`native histograms`** in Prometheus 2.40+ and how does it solve the bucket configuration problem?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — histogram accuracy, native histograms, bucket configuration sections

---

### Q438 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `OpenTelemetry Operator`**. What does it provide beyond just deploying the OpenTelemetry Collector?
>
> What is **`auto-instrumentation`** — how does the operator inject instrumentation into application pods **without code changes** using init containers and environment variables? Which languages are supported? Write the `Instrumentation` CR that auto-instruments a Java application to send traces to Jaeger and metrics to Prometheus.

📁 **Reference:** `nawab312/Kubernetes` → `08_OBSERVABILITY.md` — OpenTelemetry Operator, auto-instrumentation, tracing sections

---

### Q439 — AWS | Scenario-Based | Advanced

> Your company is implementing **AWS `FinOps`** practices — engineering teams should own their cloud costs and optimize them continuously.
>
> Design a complete FinOps implementation covering:
> - **Tagging strategy** enforced via SCPs and Config rules
> - **Cost allocation** to teams via Cost Explorer tag groups
> - **Budget alerts** per team with automated Slack notifications
> - **Rightsizing recommendations** automated via Compute Optimizer
> - **Waste detection** — unattached EBS volumes, unused Elastic IPs, idle RDS
> - **Chargeback/showback** reports sent weekly to team leads

📁 **Reference:** `nawab312/AWS` — FinOps, Cost Explorer, Compute Optimizer, tagging strategy sections

---

### Q440 — Linux / Bash | Conceptual | Advanced

> Explain **Linux `strace`** in depth — what does it trace and when would you use it vs `ltrace` vs `perf trace`?
>
> Give 5 real-world DevOps scenarios where `strace` reveals the root cause that no other tool shows:
> - Application failing silently with no logs
> - Permission denied errors with no obvious cause
> - Application hanging with no CPU usage
> - Unexpectedly slow file operations
> - Network connection failures
>
> Write the exact `strace` commands for each scenario.

📁 **Reference:** `nawab312/DSA` → `Linux` — strace, syscall tracing, debugging sections

---

### Q441 — Kubernetes | Scenario-Based | Advanced

> You need to implement **Kubernetes `NetworkPolicy` enforcement with audit logging** — you want to know which connections are being denied by NetworkPolicy without disrupting traffic.
>
> Walk me through implementing **Calico's `NetworkPolicy` audit mode** or **Cilium's policy audit mode**:
> - Running policies in audit mode (log denies, don't block)
> - Generating **traffic baseline** before enforcing
> - Gradually enabling enforcement per namespace
> - Correlating audit logs with specific workloads
> - Automating NetworkPolicy generation from observed traffic

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` and `07_SECURITY.md` — Calico, Cilium audit mode, policy enforcement sections

---

### Q442 — AWS | Conceptual | Advanced

> Explain **AWS `DataSync`** vs **AWS `Transfer Family`** vs **AWS `Storage Gateway`**.
>
> When would you use each for hybrid storage scenarios? Design a solution using the appropriate service for each of these use cases:
> - Daily sync of 10TB on-premises NFS share to S3
> - Providing SFTP access to external partners to upload files to S3
> - On-premises applications that need low-latency access to S3-backed storage via NFS mount

📁 **Reference:** `nawab312/AWS` — DataSync, Transfer Family, Storage Gateway, hybrid storage sections

---

### Q443 — ArgoCD | Troubleshooting | Advanced

> ArgoCD is showing a **`OutOfSync`** status but when you look at the diff, the only difference is **ordering of list items** in a Kubernetes resource (e.g., environment variables in a Deployment are in a different order than in Git).
>
> Why does ArgoCD consider ordering differences as drift? How do you configure ArgoCD to **ignore ordering differences** for specific fields? Write the `resource.customizations` configuration in the ArgoCD ConfigMap to ignore environment variable ordering and container ordering in Deployments.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — resource customizations, ordering diff, ignoreDifferences sections

---

### Q444 — Grafana | Conceptual | Advanced

> Explain **Grafana `Unified Alerting` evaluation engine** in depth. How does Grafana evaluate alert rules differently from Prometheus alerting rules?
>
> What is **`NoData`** and **`Error`** state handling in Grafana alerts? How do you configure an alert to **not fire when there is no data** (which is normal behavior during low-traffic periods)? What is the `Keep Last State` option and when is it appropriate vs dangerous?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — unified alerting, NoData handling, evaluation engine sections

---

### Q445 — Kubernetes | Troubleshooting | Advanced

> Your Kubernetes cluster's **`kubelet` is OOMKilled** repeatedly on several nodes. This causes all Pods on those nodes to be evicted and the node to become NotReady temporarily.
>
> What causes the kubelet process itself to run out of memory? What are **`kubeReserved`** and **`systemReserved`** settings in kubelet configuration and why are they critical? How do you properly size these reservations and prevent kubelet OOM?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` and `09_CLUSTER_OPERATIONS.md` — kubelet reserved resources, node allocatable sections

---

### Q446 — AWS | Scenario-Based | Advanced

> You are implementing a **security incident response automation** on AWS using **Lambda, EventBridge, and Systems Manager**:
> - GuardDuty finding: `UnauthorizedAccess:EC2/SSHBruteForce` → automatically add attacker IP to WAF block list
> - GuardDuty finding: `CryptoCurrency:EC2/BitcoinTool` → automatically isolate the EC2 instance (remove from security group, snapshot, quarantine)
> - IAM finding: `Policy:IAMUser/RootCredentialUsage` → immediately notify security team via PagerDuty and freeze root account
>
> Write the EventBridge rules and Lambda automation for each scenario.

📁 **Reference:** `nawab312/AWS` — GuardDuty, EventBridge, Lambda automation, incident response sections

---

### Q447 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** for **automated log analysis and pattern detection** across a distributed system:
> - Collects logs from multiple servers via SSH simultaneously (parallel with `xargs -P`)
> - Extracts **error patterns** using configurable regex patterns from a pattern file
> - Performs **correlation** — finds errors that appear across multiple servers within the same 30-second window
> - Identifies **cascading failures** — server A error followed by server B error within 60 seconds
> - Outputs a **timeline** of correlated events with server names and timestamps
> - Generates a **root cause hypothesis** based on which server had the first error

📁 **Reference:** `nawab312/DSA` → `Linux` — log analysis, parallel SSH, correlation, pattern detection sections

---

### Q448 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `sigstore` and `cosign`** for container image signing and verification.
>
> What problem does image signing solve that digest pinning alone does not? How does **Cosign** sign images using keyless signing with **Sigstore's Fulcio CA** and **Rekor transparency log**? How do you enforce signed images only in Kubernetes using **`policy-controller`** admission webhook? Write the policy that only allows images signed by your CI/CD pipeline.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md` and `13_DOCKER_CONTAINER_FUNDAMENTALS.md` — image signing, cosign, sigstore, supply chain security sections

---

### Q449 — AWS + Kubernetes | Scenario-Based | Advanced

> You are designing a **cost-optimized EKS architecture** for a startup with variable workloads. The requirements are:
> - Minimize costs during **off-hours** (scale to near zero at night)
> - Handle **morning traffic spikes** within 2 minutes
> - Use **Spot instances** for 80% of workload with On-Demand fallback
> - Implement **bin packing** to maximize node utilization
> - Automatically **right-size** pods based on actual usage
>
> Design the complete architecture using Karpenter, KEDA, VPA, cluster sleep schedules, and Spot interruption handling.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` and `nawab312/AWS` — EKS cost optimization, Karpenter, Spot, KEDA sections

---

### Q450 — All Topics | System Design | Advanced

> You are the **founding SRE** at a Series B startup (50 engineers, 20 microservices, $2M/month AWS bill growing 20% monthly). The CTO wants you to build a **world-class engineering platform** over the next 12 months that enables:
> - Engineers to **deploy 10+ times per day** with confidence
> - **Mean Time to Recovery (MTTR) under 10 minutes** for any incident
> - **Infrastructure costs grow at half the rate** of the business
> - **Zero security incidents** from infrastructure misconfigurations
> - New engineers **productive within 1 week** of joining
>
> Design your complete 12-month roadmap — what you build in months 1-3, 4-6, 7-9, and 10-12. For each phase, specify the tools, the outcomes achieved, and the metrics that prove success. Use everything you know from Kubernetes, AWS, Terraform, CI/CD, monitoring, and SRE practices.

📁 **Reference:** All repositories — complete SRE platform roadmap, engineering excellence design

---

## Key Topics Coverage (Q406–Q450)

| Topic | Questions |
|---|---|
| Kubernetes | Q406, Q410, Q414, Q417, Q421, Q425, Q429, Q432, Q435, Q438, Q441, Q445, Q448 |
| AWS | Q407, Q411, Q415, Q418, Q422, Q426, Q430, Q433, Q436, Q439, Q442, Q446, Q449 |
| Terraform | Q424 |
| Prometheus | Q409, Q416, Q428, Q437 |
| Grafana | Q419, Q431, Q444 |
| ELK Stack | — (fully covered in previous sets) |
| Linux / Bash | Q408, Q413, Q420, Q427, Q434, Q440, Q447 |
| ArgoCD | Q412, Q423, Q443 |
| Jenkins | — (fully covered in previous sets) |
| GitHub Actions | — (fully covered in previous sets) |
| Git | — (fully covered in previous sets) |
| Python | Q384 (covered previous set) |
| System Design | Q450 |

---

## Final Coverage Summary — All 450 Questions

### Kubernetes (~125 questions)
✅ Core Architecture (etcd, API server, scheduler, controller manager)
✅ Workloads (Pods, Deployments, StatefulSets, DaemonSets, Jobs, CronJobs)
✅ Networking (Services, Ingress, Gateway API, NetworkPolicy, DNS, CNI)
✅ Storage (PV, PVC, StorageClass, CSI, VolumeSnapshot)
✅ Scheduling (HPA, VPA, KEDA, ResourceQuota, LimitRange, Affinity, Taints)
✅ Security (RBAC, PSA, OPA, mTLS, image signing, cosign)
✅ Observability (Prometheus Operator, OpenTelemetry Operator)
✅ Operations (Upgrades, Cluster API, Backup/Restore, etcd)
✅ Extensibility (CRDs, Operators, Admission Webhooks)
✅ Service Mesh (Istio, Cilium, eBPF)
✅ Multi-tenancy (vcluster, Capsule, HNC)
✅ Runtimes (gVisor, WASM, containerd)
✅ Cost Optimization (Goldilocks, Kubecost, Karpenter)

### AWS (~90 questions)
✅ Compute (EC2, ECS, EKS, Lambda, Fargate, Batch, Graviton)
✅ Networking (VPC, ALB, NLB, GWLB, CloudFront, Route53, Direct Connect)
✅ Storage (S3, EBS, EFS, DataSync, Transfer Family, Storage Gateway)
✅ Database (RDS, DynamoDB, ElastiCache, Aurora, RDS Proxy)
✅ Security (IAM, GuardDuty, Macie, Inspector, Security Hub, WAF, Shield)
✅ CI/CD (CodePipeline, CodeBuild, CodeDeploy)
✅ Messaging (SQS, SNS, EventBridge, Kinesis, MSK)
✅ Data (Glue, Athena, EMR, Lake Formation)
✅ Governance (Organizations, Control Tower, Config, SCPs)
✅ Cost (Cost Explorer, Savings Plans, Compute Optimizer, FinOps)
✅ Hybrid (Direct Connect, VPN, Transit Gateway, Cloud WAN)

### All Other Topics
✅ Terraform — complete coverage (modules, state, backends, testing, policy)
✅ Prometheus — complete coverage (metrics, alerting, Thanos, OTel, agent mode)
✅ Grafana — complete coverage (dashboards, alerting, Loki, Tempo, plugins)
✅ ELK Stack — complete coverage (indexing, ILM, snapshots, security)
✅ Linux/Bash — complete coverage (networking, memory, storage, perf, eBPF)
✅ Git — complete coverage (workflows, history, hooks, advanced commands)
✅ Jenkins — complete coverage (pipelines, agents, shared libraries, JCasC)
✅ GitHub Actions — complete coverage (OIDC, matrix, security, optimization)
✅ ArgoCD — complete coverage (GitOps, generators, hooks, Vault, notifications)
✅ Python — complete coverage (boto3, K8s client, automation scripts)

---

## Estimated Interview Readiness

| Topic | Coverage | Readiness |
|---|---|---|
| Kubernetes | 95% | ⭐⭐⭐⭐⭐ |
| AWS | 92% | ⭐⭐⭐⭐⭐ |
| Terraform | 95% | ⭐⭐⭐⭐⭐ |
| Prometheus | 93% | ⭐⭐⭐⭐⭐ |
| Grafana | 90% | ⭐⭐⭐⭐⭐ |
| ELK Stack | 90% | ⭐⭐⭐⭐⭐ |
| Linux / Bash | 92% | ⭐⭐⭐⭐⭐ |
| Git | 95% | ⭐⭐⭐⭐⭐ |
| Jenkins | 92% | ⭐⭐⭐⭐⭐ |
| GitHub Actions | 93% | ⭐⭐⭐⭐⭐ |
| ArgoCD | 94% | ⭐⭐⭐⭐⭐ |
| Python | 88% | ⭐⭐⭐⭐ |

---

*Part 10 — FINAL set of DevOps/SRE Interview Questions — nawab312 GitHub repositories*
*Total question bank: Q1–Q450*
*You are now comprehensively prepared for Senior DevOps / SRE / Cloud Engineer interviews.*
