# DevOps / SRE / Cloud Interview Questions (136–180)

> Part 4 of interview preparation questions based on your GitHub repositories.
> All questions are unique — not repeated from Q1–Q135.

---

## Table of Contents

| # | Topic | Type | Difficulty |
|---|---|---|---|
| 136 | Kubernetes | Conceptual | Advanced |
| 137 | AWS | Scenario-Based | Advanced |
| 138 | Linux / Bash | Conceptual | Advanced |
| 139 | Terraform | Conceptual | Advanced |
| 140 | Prometheus | Scenario-Based | Advanced |
| 141 | Kubernetes | Troubleshooting | Advanced |
| 142 | Git | Conceptual | Advanced |
| 143 | AWS | Conceptual | Advanced |
| 144 | Jenkins | Troubleshooting | Advanced |
| 145 | ELK Stack | Scenario-Based | Advanced |
| 146 | Kubernetes | Scenario-Based | Advanced |
| 147 | Linux / Bash | Troubleshooting | Advanced |
| 148 | Terraform | Scenario-Based | Advanced |
| 149 | GitHub Actions | Conceptual | Advanced |
| 150 | AWS | Troubleshooting | Advanced |
| 151 | Prometheus | Conceptual | Advanced |
| 152 | Kubernetes | Conceptual | Advanced |
| 153 | ArgoCD | Conceptual | Advanced |
| 154 | Linux / Bash | Scenario-Based | Advanced |
| 155 | AWS | Scenario-Based | Advanced |
| 156 | Kubernetes | Troubleshooting | Advanced |
| 157 | Grafana | Scenario-Based | Advanced |
| 158 | Terraform | Troubleshooting | Advanced |
| 159 | Python | Scenario-Based | Advanced |
| 160 | AWS | Conceptual | Advanced |
| 161 | Kubernetes | Scenario-Based | Advanced |
| 162 | ELK Stack | Conceptual | Advanced |
| 163 | Jenkins | Scenario-Based | Advanced |
| 164 | Linux / Bash | Conceptual | Advanced |
| 165 | AWS | Scenario-Based | Advanced |
| 166 | Kubernetes | Conceptual | Advanced |
| 167 | GitHub Actions | Scenario-Based | Advanced |
| 168 | Terraform | Conceptual | Advanced |
| 169 | Prometheus | Troubleshooting | Advanced |
| 170 | AWS | Conceptual | Advanced |
| 171 | Kubernetes | Scenario-Based | Advanced |
| 172 | ArgoCD | Scenario-Based | Advanced |
| 173 | Linux / Bash | Scenario-Based | Advanced |
| 174 | AWS | Scenario-Based | Advanced |
| 175 | Kubernetes | Conceptual | Advanced |
| 176 | Grafana | Conceptual | Advanced |
| 177 | Terraform | Scenario-Based | Advanced |
| 178 | Git | Scenario-Based | Advanced |
| 179 | AWS + Kubernetes | Scenario-Based | Advanced |
| 180 | All Topics | System Design | Advanced |

---

## Questions

---

### Q136 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `NetworkPolicy`** in detail. What is the difference between **ingress** and **egress** policies?
>
> If no NetworkPolicy exists in a namespace, what is the default behavior? Write a NetworkPolicy that:
> - Denies ALL ingress and egress by default
> - Allows ingress only from pods with label `role=frontend`
> - Allows egress only to port 5432 (PostgreSQL) and DNS (port 53)

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` and `07_SECURITY.md` — NetworkPolicy sections

---

### Q137 — AWS | Scenario-Based | Advanced

> Your company has a **Kinesis Data Stream** that processes real-time events from 10 million IoT devices. The stream has **10 shards** but you are seeing:
> - `ProvisionedThroughputExceededException` errors
> - Consumer lag growing
> - Some events being dropped
>
> How would you diagnose and fix the throughput issues, and how would you decide when to **reshard** the stream?

📁 **Reference:** `nawab312/AWS` — Kinesis Data Streams, sharding, enhanced fan-out sections

---

### Q138 — Linux / Bash | Conceptual | Advanced

> Explain the Linux **`systemd`** init system. How does it differ from the older **SysV init**?
>
> Explain **systemd units** — what are the different unit types (service, timer, socket, target)? Write a **systemd service file** for a Node.js application that:
> - Starts automatically on boot
> - Restarts on failure with a 5-second delay
> - Has resource limits (CPU and memory)
> - Logs to journald

📁 **Reference:** `nawab312/DSA` → `Linux` — systemd, service management sections

---

### Q139 — Terraform | Conceptual | Advanced

> Explain **Terraform `data sources`**. How are they different from resources?
>
> Give **5 real-world examples** of data sources you would commonly use in AWS Terraform configurations, and explain what problem each one solves. When would using a data source be preferable to hardcoding a value?

📁 **Reference:** `nawab312/Terraform` — data sources, AWS provider sections

---

### Q140 — Prometheus | Scenario-Based | Advanced

> You need to implement **Prometheus alerting** for a Kubernetes cluster with these requirements:
> - Alert when any **node disk usage** exceeds 80%
> - Alert when **pod restarts** more than 5 times in 1 hour
> - Alert when **cluster CPU** is above 85% for 10 minutes
> - Alert when a **Deployment has 0 available replicas**
> - All alerts must route to **different Slack channels** based on severity
>
> Write the complete **PrometheusRules** and **AlertmanagerConfig**.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — alerting rules, Alertmanager routing sections

---

### Q141 — Kubernetes | Troubleshooting | Advanced

> Your application Pods are running but the **Horizontal Pod Autoscaler shows `<unknown>/50%`** for CPU utilization instead of a real value.
>
> What does this mean, why does it happen, and walk me through the complete fix?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — HPA, Metrics Server sections

---

### Q142 — Git | Conceptual | Advanced

> Explain **Git hooks** — what are they, where do they live, and how do they work?
>
> Give examples of **5 useful Git hooks** for a DevOps team. Write a **`pre-commit` hook** that:
> - Prevents committing files larger than 1MB
> - Scans for hardcoded secrets (AWS keys, passwords)
> - Runs linting before allowing the commit

📁 **Reference:** `nawab312/CI_CD` → `Git` — Git hooks, pre-commit framework sections

---

### Q143 — AWS | Conceptual | Advanced

> Explain **AWS ElastiCache** — what is the difference between **Redis** and **Memcached** modes?
>
> When would you choose Redis over Memcached? Explain **Redis Cluster mode** vs **Redis Replication Group**. Give a real-world use case where ElastiCache Redis would dramatically improve application performance.

📁 **Reference:** `nawab312/AWS` — ElastiCache, Redis, Memcached sections

---

### Q144 — Jenkins | Troubleshooting | Advanced

> Your Jenkins build is **consistently failing** with `java.lang.OutOfMemoryError: Java heap space` after running for about 20 minutes.
>
> What causes Jenkins OOM errors, how do you diagnose which part of the pipeline is causing it, and what are the short-term and long-term fixes?

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — JVM tuning, memory management, build optimization sections

---

### Q145 — ELK Stack | Scenario-Based | Advanced

> Your team wants to implement **centralized logging** for a Kubernetes cluster with 50 microservices. Logs need to be:
> - Collected from all Pod stdout/stderr
> - Enriched with **Kubernetes metadata** (namespace, pod name, labels)
> - Parsed and structured (JSON vs plaintext logs)
> - Stored for **30 days** in Elasticsearch
> - Searchable in Kibana within **5 seconds** of generation
>
> Design the complete logging architecture.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Filebeat, Logstash, Kubernetes logging sections

---

### Q146 — Kubernetes | Scenario-Based | Advanced

> You need to implement **SSL/TLS certificate management** at scale in Kubernetes. You have 20 services each needing their own TLS certificate from **Let's Encrypt**.
>
> Walk me through setting up **cert-manager** end-to-end — from installation to automatic certificate issuance and renewal for all 20 services.

📁 **Reference:** `nawab312/Kubernetes` → `SSLCertificate_Kubernetes.md` — cert-manager, Let's Encrypt, Ingress TLS sections

---

### Q147 — Linux / Bash | Troubleshooting | Advanced

> A production application is throwing `Too many open files` errors. `ulimit -n` shows 1024 on the server. You increase it but the errors keep coming back after a few hours.
>
> Walk me through the **complete investigation** — how do you find which process is leaking file descriptors, how do you set limits permanently, and how do you prevent this in future?

📁 **Reference:** `nawab312/DSA` → `Linux` — file descriptors, ulimit, `/proc/sys/fs` sections

---

### Q148 — Terraform | Scenario-Based | Advanced

> You are asked to build a **Terraform module** for an ECS service that will be reused by 15 teams. The module needs to be:
> - **Versioned** and published to a private Terraform registry
> - **Backward compatible** — teams on older versions should not be forced to upgrade
> - **Well documented** with input/output descriptions
> - **Tested** before publishing
>
> Walk me through the complete module design, versioning strategy, and testing approach.

📁 **Reference:** `nawab312/Terraform` — module design, versioning, Terraform Registry sections

---

### Q149 — GitHub Actions | Conceptual | Advanced

> Explain **GitHub Actions self-hosted runners** vs **GitHub-hosted runners**.
>
> When would you use self-hosted runners? What are the **security risks** of self-hosted runners and how do you mitigate them? How would you set up **auto-scaling self-hosted runners** on AWS using EC2?

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — self-hosted runners, security sections

---

### Q150 — AWS | Troubleshooting | Advanced

> Your **AWS ECS service** is showing tasks repeatedly cycling between `RUNNING` and `STOPPED` states. The service never reaches its desired count of 3 running tasks.
>
> Walk me through the complete troubleshooting process to identify why ECS tasks keep stopping.

📁 **Reference:** `nawab312/AWS` — ECS, task definitions, CloudWatch Logs, health checks sections

---

### Q151 — Prometheus | Conceptual | Advanced

> Explain **Prometheus `remote_write`** and **`remote_read`**. What are they used for?
>
> What is **Prometheus TSDB (Time Series Database)** — how does it store data on disk (chunks, blocks, WAL)? What happens when Prometheus runs out of disk space?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — TSDB internals, remote storage sections

---

### Q152 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `Taints` and `Tolerations`** in detail.
>
> What are the **3 taint effects** (`NoSchedule`, `PreferNoSchedule`, `NoExecute`) and how do they differ? Give a real-world scenario where you would use `NoExecute` and explain what happens to existing Pods when you add a `NoExecute` taint to a node.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — taints, tolerations sections

---

### Q153 — ArgoCD | Conceptual | Advanced

> Explain **ArgoCD RBAC** — how do you control who can do what in ArgoCD?
>
> What is the difference between ArgoCD **Projects** and ArgoCD **Applications**? How do Projects provide **multi-tenancy** in ArgoCD — limiting which Git repos, clusters, and namespaces a team can deploy to?

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — RBAC, Projects, multi-tenancy sections

---

### Q154 — Linux / Bash | Scenario-Based | Advanced

> Write a **production-grade Bash script** for rotating application logs that:
> - Compresses logs older than **7 days** using gzip
> - Deletes compressed logs older than **30 days**
> - Handles log rotation **without interrupting** the running application (uses `copytruncate`)
> - Sends a **weekly summary report** via email showing disk space saved
> - Has proper **error handling** — exits with non-zero code on failure
> - Is safe to run as a **cron job**

📁 **Reference:** `nawab312/DSA` → `Linux` — log rotation, cron, bash scripting sections

---

### Q155 — AWS | Scenario-Based | Advanced

> Your company wants to implement **cost optimization** on AWS. The monthly bill is $50,000 and you have been asked to reduce it by 30%.
>
> What would be your **systematic approach** to identify waste and reduce costs? List at least **10 concrete actions** across EC2, RDS, S3, and data transfer costs.

📁 **Reference:** `nawab312/AWS` — Cost Explorer, Reserved Instances, Savings Plans, Right-sizing sections

---

### Q156 — Kubernetes | Troubleshooting | Advanced

> You notice that your Kubernetes cluster's **etcd is showing high latency** — `etcd_disk_wal_fsync_duration_seconds` p99 is above 10ms. This is causing API server slowness.
>
> What causes etcd disk latency, how do you diagnose it, and what are the fixes?

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md` and `09_CLUSTER_OPERATIONS.md` — etcd performance sections

---

### Q157 — Grafana | Scenario-Based | Advanced

> You need to build a **Grafana OnCall** (or equivalent) alerting system for your team. Design an **alerting strategy** that:
> - Has **3 severity levels**: P1 (wake up now), P2 (fix within 4 hours), P3 (fix next business day)
> - Routes P1 alerts to **PagerDuty** with phone call escalation
> - Routes P2 alerts to **Slack** with 30-minute follow-up
> - Routes P3 to a **Jira ticket** automatically
> - Has **alert deduplication** to avoid spam
>
> What tools and configurations would you use?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` and `Prometheus` — alerting, Alertmanager routing sections

---

### Q158 — Terraform | Troubleshooting | Advanced

> After a team member refactored Terraform modules, `terraform plan` now shows **dozens of resource replacements** that should NOT be happening — the actual infrastructure should remain unchanged.
>
> What are the most common causes of **unintended resource replacements during refactoring** and how do you fix each one without destroying infrastructure?

📁 **Reference:** `nawab312/Terraform` — `moved` block, state manipulation, lifecycle `ignore_changes` sections

---

### Q159 — Python | Scenario-Based | Advanced

> Write a **Python script** that implements a simple **CI/CD pipeline status dashboard**:
> - Calls the **GitHub Actions API** to get the last 10 workflow runs for a repository
> - For each run shows: workflow name, status, duration, triggered by, branch
> - Color-codes the output: green for success, red for failure, yellow for in-progress
> - Calculates and shows the **success rate** over the last 10 runs
> - Accepts the repository name as a command-line argument

📁 **Reference:** `nawab312/DSA` → `Python` — REST API calls, argparse, JSON sections

---

### Q160 — AWS | Conceptual | Advanced

> Explain **AWS CloudFormation** vs **Terraform** for infrastructure as code.
>
> What are the **key differences** in state management, provider support, and language? In what scenarios would you choose CloudFormation over Terraform? How does **AWS CDK** (Cloud Development Kit) fit into this comparison?

📁 **Reference:** `nawab312/AWS` and `nawab312/Terraform` — IaC comparison sections

---

### Q161 — Kubernetes | Scenario-Based | Advanced

> You are running a **financial application** on Kubernetes that processes transactions. The application has strict requirements:
> - Pods must **never be evicted** during processing
> - Must run on **dedicated hardware** (no shared nodes)
> - Must have **guaranteed CPU and memory** at all times
> - Pod startup must complete within **30 seconds** or be killed
>
> Write the complete Kubernetes configuration to meet all these requirements.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` and `07_SECURITY.md` — QoS Guaranteed, taints, resource management sections

---

### Q162 — ELK Stack | Conceptual | Advanced

> Explain **Elasticsearch query DSL** vs **KQL (Kibana Query Language)**.
>
> Write Elasticsearch queries to:
> - Find all logs with `level: ERROR` from the last 24 hours
> - Find the **top 10 most common error messages** using aggregations
> - Find logs where response time is **greater than 2 seconds**
> - Search across **multiple indices** matching a pattern

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Elasticsearch query DSL, aggregations sections

---

### Q163 — Jenkins | Scenario-Based | Advanced

> You need to implement **dynamic Jenkins agents** that:
> - Spin up on **AWS EC2** only when a build is triggered
> - Use different **instance types** for different job types (t3.medium for unit tests, c5.2xlarge for integration tests)
> - **Terminate automatically** after the build completes
> - Use **spot instances** to reduce cost
>
> How would you configure this using the **Jenkins EC2 plugin** or **Kubernetes plugin**?

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — dynamic agents, EC2 plugin, cloud configuration sections

---

### Q164 — Linux / Bash | Conceptual | Advanced

> Explain **Linux signals** in detail. What are the most important signals a DevOps engineer should know?
>
> Explain the difference between **`SIGTERM`**, **`SIGKILL`**, **`SIGHUP`**, **`SIGINT`**, and **`SIGCHLD`**. Why can't `SIGKILL` be caught or ignored? How do containers handle signals differently from regular processes?

📁 **Reference:** `nawab312/DSA` → `Linux` — signals, kill, trap sections

---

### Q165 — AWS | Scenario-Based | Advanced

> You are building a **serverless data processing pipeline** on AWS that must handle **100,000 events per second** from IoT devices:
> - Events arrive via **API Gateway**
> - Need **real-time processing** and **batch processing**
> - Results stored in **S3** and **DynamoDB**
> - Must be **cost-effective** at scale
>
> Design the complete architecture with AWS services.

📁 **Reference:** `nawab312/AWS` — Kinesis, Lambda, API Gateway, DynamoDB, S3 sections

---

### Q166 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `PersistentVolume` reclaim policies** — `Retain`, `Recycle`, and `Delete`.
>
> What happens to the data when a `PersistentVolumeClaim` is deleted under each policy? What is **`StorageClass` volume binding mode** (`Immediate` vs `WaitForFirstConsumer`) and why does it matter for multi-AZ clusters?

📁 **Reference:** `nawab312/Kubernetes` → `04_STORAGE.md` — PV, PVC, StorageClass sections

---

### Q167 — GitHub Actions | Scenario-Based | Advanced

> You need to implement a **GitHub Actions workflow** for a Python library that:
> - Runs tests on **Python 3.10, 3.11, 3.12** across **Linux, Windows, macOS**
> - Automatically **publishes to PyPI** when a tag is pushed (e.g., `v1.2.3`)
> - Uses **OIDC trusted publishing** to PyPI (no API keys stored)
> - Generates a **changelog** from commit messages
> - Creates a **GitHub Release** with the changelog attached
>
> Write the complete workflow.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — PyPI publishing, OIDC, release automation sections

---

### Q168 — Terraform | Conceptual | Advanced

> Explain **Terraform `output` values** and their use cases. How do you share outputs between Terraform configurations using **`terraform_remote_state`**?
>
> What are the **security risks** of storing sensitive values in Terraform outputs? How should you handle sensitive outputs like database passwords or private keys?

📁 **Reference:** `nawab312/Terraform` — outputs, remote state, sensitive values sections

---

### Q169 — Prometheus | Troubleshooting | Advanced

> Your Prometheus is showing **`TSDB compaction failed`** errors in its logs and the `prometheus_tsdb_compactions_failed_total` metric is increasing.
>
> What is TSDB compaction, why does it fail, and what are the consequences if compaction cannot run? Walk me through the investigation and fix.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — TSDB, compaction, disk management sections

---

### Q170 — AWS | Conceptual | Advanced

> Explain **AWS WAF (Web Application Firewall)**. What types of attacks does it protect against?
>
> How do you integrate AWS WAF with **ALB** and **CloudFront**? Write an example WAF rule that:
> - Rate limits requests to more than 1000/5 minutes per IP
> - Blocks requests from specific countries
> - Blocks SQL injection attempts

📁 **Reference:** `nawab312/AWS` — WAF, ALB, CloudFront, security sections

---

### Q171 — Kubernetes | Scenario-Based | Advanced

> Your team is onboarding a **new microservice** to an existing Kubernetes cluster. The service needs to:
> - Connect to an **external Redis** (ElastiCache) using TLS
> - Pull images from a **private ECR** repository
> - Access **AWS Secrets Manager** for database credentials
> - Be accessible only from within the cluster (no public exposure)
>
> Write the complete Kubernetes manifests and explain each configuration decision.

📁 **Reference:** `nawab312/Kubernetes` → `05_CONFIGURATION_SECRETS.md` and `07_SECURITY.md` — Secrets, IRSA, imagePullSecrets sections

---

### Q172 — ArgoCD | Scenario-Based | Advanced

> You want to implement **progressive delivery** using ArgoCD and **Argo Rollouts**. Specifically:
> - Deploy new versions using **canary strategy** — 10% → 30% → 100% traffic
> - Automatically **promote** if error rate stays below 1%
> - Automatically **rollback** if error rate exceeds 5%
> - Use **Prometheus metrics** to drive the promotion decision
>
> Walk me through the complete setup.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — Argo Rollouts, canary analysis, Prometheus integration sections

---

### Q173 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that performs a **health check** across a fleet of servers:
> - Reads server list from a file
> - Checks: SSH reachability, disk usage < 80%, CPU load < 70%, memory usage < 85%, specific service running
> - Runs all checks **in parallel** (max 10 concurrent)
> - Generates a **color-coded HTML report** showing pass/fail for each check per server
> - Emails the report if any server fails any check

📁 **Reference:** `nawab312/DSA` → `Linux` — parallel execution, SSH, HTML generation sections

---

### Q174 — AWS | Scenario-Based | Advanced

> Your company is moving to **AWS Organizations** with multiple AWS accounts. You need to implement:
> - **Centralized logging** — all CloudTrail logs from all accounts to one S3 bucket
> - **Security guardrails** — SCPs preventing any account from disabling CloudTrail
> - **Centralized billing** — consolidated billing with cost allocation tags
> - **Cross-account access** — developers can assume roles in dev accounts but not prod
>
> Walk me through the complete AWS Organizations setup.

📁 **Reference:** `nawab312/AWS` — AWS Organizations, SCPs, CloudTrail, cross-account sections

---

### Q175 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `EndpointSlices`** and how they improve upon the original **`Endpoints`** resource.
>
> What scaling problems did the original Endpoints object cause in large clusters? How does `kube-proxy` use EndpointSlices to implement Service load balancing? What is the role of **`kube-proxy` modes** (iptables vs ipvs) in Service traffic routing?

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — Endpoints, EndpointSlices, kube-proxy sections

---

### Q176 — Grafana | Conceptual | Advanced

> Explain **Grafana Tempo** for distributed tracing. How does it integrate with **Prometheus** and **Loki** to provide the **3 pillars of observability** (metrics, logs, traces) in a unified Grafana stack?
>
> What is **exemplar** in Prometheus and how does it create a link between a metric spike and a specific trace in Tempo?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — Tempo, distributed tracing, exemplars sections

---

### Q177 — Terraform | Scenario-Based | Advanced

> You need to implement **Terraform CI/CD** using **Atlantis** (or an equivalent GitOps-for-Terraform tool).
>
> Explain how Atlantis works. What is the workflow from **PR creation to terraform apply**? How does it handle **locking**, **plan approval**, and **multiple workspaces**? What are the security considerations for running Atlantis?

📁 **Reference:** `nawab312/Terraform` and `nawab312/CI_CD` — Atlantis, Terraform GitOps sections

---

### Q178 — Git | Scenario-Based | Advanced

> Your team has a **monorepo** with 20 services. The Git repository has grown to **15GB** because developers have been committing large binary files and build artifacts for 2 years.
>
> How would you:
> 1. Identify what is making the repo large
> 2. Remove large files from Git history permanently
> 3. Prevent this from happening again
> 4. Handle the impact on all developers who have cloned the repo

📁 **Reference:** `nawab312/CI_CD` → `Git` — BFG Repo Cleaner, git-filter-repo, `.gitattributes`, LFS sections

---

### Q179 — AWS + Kubernetes | Scenario-Based | Advanced

> You are running **EKS** and your cluster nodes are **running out of IP addresses**. The VPC was created with a `/24` CIDR block (256 IPs) and you now have 200 nodes each consuming multiple IPs for pods.
>
> Explain why this happens with the **AWS VPC CNI plugin**, what the options are to solve it, and walk me through implementing **prefix delegation** to dramatically increase pod density per node.

📁 **Reference:** `nawab312/AWS` and `nawab312/Kubernetes` — VPC CNI, EKS networking, prefix delegation sections

---

### Q180 — All Topics | System Design | Advanced

> You are the **Principal SRE** at a company that runs a **global e-commerce platform** handling Black Friday traffic — normally 10,000 RPS but spikes to **500,000 RPS** for 6 hours.
>
> Design the complete **capacity planning and traffic management strategy**:
> - How do you **predict** the required capacity?
> - How do you **pre-scale** before the event?
> - How do you handle **auto-scaling** during the event?
> - What **circuit breakers and load shedding** strategies do you implement?
> - How do you **monitor and respond** in real-time during the event?
> - What is your **rollback plan** if something goes wrong?

📁 **Reference:** All repositories — comprehensive capacity planning and traffic management design

---

## Key Topics Coverage (Q136–Q180)

| Topic | Questions |
|---|---|
| Kubernetes | Q136, Q141, Q146, Q152, Q156, Q161, Q166, Q171, Q175, Q179 |
| AWS | Q137, Q143, Q150, Q155, Q160, Q165, Q170, Q174, Q179 |
| Terraform | Q139, Q148, Q158, Q168, Q177 |
| Prometheus | Q140, Q151, Q169 |
| Grafana | Q157, Q176 |
| ELK Stack | Q145, Q162 |
| Linux / Bash | Q138, Q147, Q154, Q164, Q173 |
| ArgoCD | Q153, Q172 |
| Jenkins | Q144, Q163 |
| GitHub Actions | Q149, Q167 |
| Git | Q142, Q178 |
| Python | Q159 |
| System Design | Q180 |

---

## Important Concepts Covered in This Set

### Kubernetes
- NetworkPolicy ingress/egress default behavior
- Pod termination lifecycle + graceful shutdown hooks
- Taints/Tolerations — NoExecute effect
- Admission Webhooks (Validating + Mutating)
- EndpointSlices vs Endpoints — scaling fix
- PV reclaim policies (Retain/Recycle/Delete)
- StorageClass WaitForFirstConsumer
- Financial app — Guaranteed QoS + dedicated nodes
- EKS VPC CNI IP exhaustion + prefix delegation

### AWS
- Kinesis shard throughput + resharding
- ElastiCache Redis vs Memcached
- ECS task cycling troubleshooting
- AWS Organizations + SCPs
- WAF rules — rate limiting, geo-blocking, SQLi
- Serverless 100K events/sec architecture
- CloudFormation vs Terraform vs CDK
- AWS cost optimization — 10+ concrete actions

### Terraform
- Data sources — 5 real-world examples
- Module versioning + private registry
- `for_each` vs `count` — replacement issue
- Outputs — sensitive value handling
- Atlantis GitOps workflow

### Prometheus
- `remote_write` / `remote_read`
- TSDB internals (WAL, chunks, blocks)
- TSDB compaction failures
- Alert flapping investigation
- Kafka monitoring with Prometheus

### Linux
- systemd unit files + resource limits
- Signals — SIGTERM vs SIGKILL vs SIGHUP
- iptables 5 chains packet flow
- File descriptor leak investigation

### Git
- Git hooks — pre-commit secret scanning
- Monorepo large file cleanup (BFG / git-filter-repo)

---

*Part 4 of DevOps/SRE Interview Questions — nawab312 GitHub repositories*
*Total question bank: Q1–Q180*
