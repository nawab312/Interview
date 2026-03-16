# DevOps / SRE / Cloud Interview Questions (361–405)

> Part 9 of interview preparation questions based on your GitHub repositories.
> All questions are unique — not repeated from Q1–Q360.

---

## Table of Contents

| # | Topic | Type | Difficulty |
|---|---|---|---|
| 361 | Kubernetes | Conceptual | Advanced |
| 362 | AWS | Scenario-Based | Advanced |
| 363 | Linux / Bash | Conceptual | Advanced |
| 364 | Terraform | Conceptual | Advanced |
| 365 | Prometheus | Scenario-Based | Advanced |
| 366 | Kubernetes | Troubleshooting | Advanced |
| 367 | Git | Conceptual | Advanced |
| 368 | AWS | Conceptual | Advanced |
| 369 | Jenkins | Scenario-Based | Advanced |
| 370 | ELK Stack | Conceptual | Advanced |
| 371 | Kubernetes | Scenario-Based | Advanced |
| 372 | Linux / Bash | Troubleshooting | Advanced |
| 373 | Terraform | Scenario-Based | Advanced |
| 374 | GitHub Actions | Conceptual | Advanced |
| 375 | AWS | Troubleshooting | Advanced |
| 376 | Prometheus | Conceptual | Advanced |
| 377 | Kubernetes | Conceptual | Advanced |
| 378 | ArgoCD | Scenario-Based | Advanced |
| 379 | Linux / Bash | Scenario-Based | Advanced |
| 380 | AWS | Scenario-Based | Advanced |
| 381 | Kubernetes | Troubleshooting | Advanced |
| 382 | Grafana | Conceptual | Advanced |
| 383 | Terraform | Troubleshooting | Advanced |
| 384 | Python | Scenario-Based | Advanced |
| 385 | AWS | Conceptual | Advanced |
| 386 | Kubernetes | Scenario-Based | Advanced |
| 387 | ELK Stack | Scenario-Based | Advanced |
| 388 | Jenkins | Conceptual | Advanced |
| 389 | Linux / Bash | Conceptual | Advanced |
| 390 | AWS | Scenario-Based | Advanced |
| 391 | Kubernetes | Conceptual | Advanced |
| 392 | GitHub Actions | Scenario-Based | Advanced |
| 393 | Terraform | Conceptual | Advanced |
| 394 | Prometheus | Troubleshooting | Advanced |
| 395 | AWS | Conceptual | Advanced |
| 396 | Kubernetes | Scenario-Based | Advanced |
| 397 | ArgoCD | Conceptual | Advanced |
| 398 | Linux / Bash | Scenario-Based | Advanced |
| 399 | AWS | Scenario-Based | Advanced |
| 400 | Kubernetes | Conceptual | Advanced |
| 401 | Grafana | Scenario-Based | Advanced |
| 402 | Terraform | Scenario-Based | Advanced |
| 403 | Git | Scenario-Based | Advanced |
| 404 | AWS + Kubernetes | Scenario-Based | Advanced |
| 405 | All Topics | System Design | Advanced |

---

## Questions

---

### Q361 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `SidecarContainers`** (GA in Kubernetes 1.29) — the new native sidecar support. How is it different from init containers and regular containers?
>
> What problems did the old sidecar pattern (using init containers with `restartPolicy: Always`) solve? What lifecycle guarantees does the new native sidecar provide — specifically around startup ordering and graceful shutdown ordering?

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md` — sidecar containers, init containers, container lifecycle sections

---

### Q362 — AWS | Scenario-Based | Advanced

> Your company is implementing **AWS `Backup`** for centralized backup management across RDS, EBS, EFS, DynamoDB, and S3. The requirements are:
> - Daily backups with **30-day retention** for production
> - Weekly backups with **1-year retention** for compliance
> - Backups must be **copied to a different AWS region**
> - RDS backups must be **tested monthly** (restore and validate)
> - Alert when any backup **fails or is missing**
>
> Design the complete AWS Backup implementation.

📁 **Reference:** `nawab312/AWS` — AWS Backup, backup plans, cross-region copy, vault sections

---

### Q363 — Linux / Bash | Conceptual | Advanced

> Explain **Linux `control groups v2 (cgroups v2)` unified hierarchy** in detail. How does it differ from cgroups v1 in terms of the **hierarchy model**?
>
> What is **`memory.oom_group`** and how does it change OOM behavior? How do **`cpu.max`** and **`cpu.weight`** in cgroups v2 replace `cpu.cfs_quota_us` and `cpu.shares` from v1? What changes does this require for Kubernetes container runtimes?

📁 **Reference:** `nawab312/DSA` → `Linux` — cgroups v2, unified hierarchy, cpu.max, memory.oom_group sections

---

### Q364 — Terraform | Conceptual | Advanced

> Explain **Terraform `templatefile()` function** vs **`file()` function** vs **`templatestring()` function** (Terraform 1.6+).
>
> Write a real-world example that uses `templatefile()` to generate:
> - An **Nginx configuration** file with dynamic upstream servers from a list variable
> - A **user_data script** for EC2 that installs packages and configures the application based on environment variables
> - A **Kubernetes ConfigMap** manifest with values interpolated from Terraform variables

📁 **Reference:** `nawab312/Terraform` — templatefile, file, string functions, user_data sections

---

### Q365 — Prometheus | Scenario-Based | Advanced

> You need to implement **Prometheus `PushGateway`** for monitoring short-lived batch jobs that complete before Prometheus can scrape them.
>
> Walk me through:
> - When to use PushGateway vs not (anti-patterns to avoid)
> - Setting up PushGateway in Kubernetes
> - How a batch job pushes metrics with proper **job and instance labels**
> - **Metric staleness** — what happens to metrics after the job completes
> - How to clean up stale metrics automatically

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — PushGateway, batch monitoring, metric lifecycle sections

---

### Q366 — Kubernetes | Troubleshooting | Advanced

> Your Kubernetes cluster's **`kube-proxy`** is not correctly updating iptables rules when Services are updated. New pods are not receiving traffic even though they are healthy and the Service selector matches.
>
> Walk me through diagnosing kube-proxy issues — how to verify iptables rules are correct, how kube-proxy watches the API server, what causes kube-proxy to fall behind, and how to safely restart it without disrupting existing connections.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — kube-proxy, iptables, Service endpoints sections

---

### Q367 — Git | Conceptual | Advanced

> Explain **`git filter-repo`** (the modern replacement for `git filter-branch`). What are its advantages over `git filter-branch`?
>
> Write the exact commands to:
> - Remove a specific **directory** (`/secrets`) from all Git history
> - Remove all files with `.env` extension from history
> - **Extract a subdirectory** into its own repository while preserving history
> - Change the **email address** of all commits from an old email to a new one

📁 **Reference:** `nawab312/CI_CD` → `Git` — git filter-repo, history rewriting, repository surgery sections

---

### Q368 — AWS | Conceptual | Advanced

> Explain **AWS `RDS Proxy`** — what problem does it solve, how does it work, and what are its limitations?
>
> What is **connection multiplexing** and **connection pinning**? When does RDS Proxy pin a connection (preventing multiplexing)? How do you configure RDS Proxy for a Lambda function that uses PostgreSQL with IAM authentication?

📁 **Reference:** `nawab312/AWS` — RDS Proxy, connection pooling, Lambda integration, IAM auth sections

---

### Q369 — Jenkins | Scenario-Based | Advanced

> You need to implement a **Jenkins pipeline** for **multi-architecture Docker builds** (amd64 and arm64) that:
> - Builds Docker images for **both architectures in parallel**
> - Uses **`docker buildx`** with QEMU emulation or native arm64 agents
> - Creates a **multi-arch manifest** combining both images
> - Pushes to ECR with the same tag pointing to both architectures
> - Runs **architecture-specific tests** on each image before creating the manifest
>
> Write the complete Jenkinsfile.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` and `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md` — multi-arch builds, docker buildx, manifest sections

---

### Q370 — ELK Stack | Conceptual | Advanced

> Explain **Elasticsearch `percolator`** queries. What are they, how do they work, and what is the use case?
>
> Also explain **Elasticsearch `scroll API`** vs **`search_after`** vs **`pit` (point in time)**. When would you use each for paginating through large result sets? Which is preferred for modern Elasticsearch versions and why?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — percolator, scroll API, search_after, pagination sections

---

### Q371 — Kubernetes | Scenario-Based | Advanced

> You are implementing **Kubernetes cluster federation** — running workloads across **multiple Kubernetes clusters** (on-prem + AWS EKS + GKE) and need a unified control plane.
>
> Explain the approaches: **KubeFed v2**, **Admiralty**, **Liqo**, and **ArgoCD multi-cluster**. What problems does multi-cluster federation solve? Design a scenario where a stateless application automatically fails over from one cluster to another.

📁 **Reference:** `nawab312/Kubernetes` → `09_CLUSTER_OPERATIONS.md` — multi-cluster management sections

---

### Q372 — Linux / Bash | Troubleshooting | Advanced

> A production server is showing **kernel soft lockup** messages in `dmesg`:
>
> `BUG: soft lockup - CPU#3 stuck for 22s! [java:12345]`
>
> What is a soft lockup? What causes it? How is it different from a hard lockup? Walk me through investigating which process and kernel subsystem is responsible, and what actions you can take.

📁 **Reference:** `nawab312/DSA` → `Linux` — kernel lockups, watchdog, dmesg, CPU scheduling sections

---

### Q373 — Terraform | Scenario-Based | Advanced

> You need to implement **Terraform state locking** for a team where the DynamoDB lock table itself has become a bottleneck — multiple teams running `terraform plan` simultaneously are contending on the same lock.
>
> Explain how the locking mechanism works internally, why it can become a bottleneck at scale, and design a **sharded locking strategy** where different parts of infrastructure use different state files and lock tables.

📁 **Reference:** `nawab312/Terraform` — state locking, DynamoDB, state sharding, backend configuration sections

---

### Q374 — GitHub Actions | Conceptual | Advanced

> Explain **GitHub Actions `OpenID Connect (OIDC)`** in depth — how does the token exchange work between GitHub and AWS?
>
> What claims are in the GitHub OIDC token (`sub`, `aud`, `iss`, `repository`, `ref`, `environment`)? Write an AWS IAM trust policy that:
> - Only allows the `deploy` job from the `main` branch
> - Only allows deployments from the `production` environment
> - Restricts to a specific GitHub organization and repository

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — OIDC, JWT claims, IAM trust policy sections

---

### Q375 — AWS | Troubleshooting | Advanced

> Your **AWS ECS tasks** are failing to start with the error:
>
> `CannotPullContainerError: Error response from daemon: Head https://123456789.dkr.ecr.us-east-1.amazonaws.com/v2/my-app/manifests/latest: no basic auth credentials`
>
> Walk me through all the possible causes of ECR authentication failures for ECS tasks and how to fix each one.

📁 **Reference:** `nawab312/AWS` — ECS, ECR, IAM roles, task execution role, authentication sections

---

### Q376 — Prometheus | Conceptual | Advanced

> Explain **Prometheus `stale markers`** — what are they and why does Prometheus introduce them?
>
> What is the difference between a **stale metric** and a metric with **no data**? How does Prometheus handle the case where a scrape target disappears — what happens to the last known values? How does this affect **PromQL queries** that use `rate()` and how does the `staleness` window interact with `lookbackDelta`?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — staleness, stale markers, lookbackDelta sections

---

### Q377 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `Horizontal Pod Autoscaler v2` behavior configuration** in depth.
>
> What is the difference between `scaleUp.stabilizationWindowSeconds` and `scaleDown.stabilizationWindowSeconds`? How do **multiple scaling policies** interact (Max vs Min selectPolicy)? Write an HPA that:
> - Scales up aggressively (doubles pods every 30 seconds)
> - Scales down conservatively (removes at most 1 pod every 2 minutes)
> - Has a 5-minute stabilization window for scale-down

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — HPA v2, behavior, scaling policies sections

---

### Q378 — ArgoCD | Scenario-Based | Advanced

> You need to implement **ArgoCD `ApplicationSet`** with a **`Matrix generator`** to deploy a microservice across multiple clusters AND multiple environments.
>
> You have:
> - 3 clusters: `eks-us`, `eks-eu`, `eks-ap`
> - 3 environments per cluster: `dev`, `staging`, `prod`
> - Each combination needs different values (region-specific configs)
>
> Write the complete ApplicationSet manifest using Matrix generator combining a List generator and a Git generator. Explain how you would exclude the combination `eks-ap/dev`.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — ApplicationSet, Matrix generator, multi-cluster, exclusions sections

---

### Q379 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that implements a **real-time metrics collector** for a Linux server that:
> - Collects every 10 seconds: CPU per-core usage, memory breakdown (used/cached/available), disk read/write IOPS, network in/out bytes, TCP connection states count
> - Stores metrics in a **circular buffer** (keeps last 24 hours)
> - Exposes metrics in **Prometheus text format** via a simple HTTP server (using netcat or socat)
> - Detects and logs when any metric crosses a **configurable threshold**
> - Has a `--summary` flag that shows min/max/avg for each metric over the stored period

📁 **Reference:** `nawab312/DSA` → `Linux` — /proc/stat, /proc/net/dev, metrics collection, bash HTTP server sections

---

### Q380 — AWS | Scenario-Based | Advanced

> Your company is adopting **AWS `Control Tower`** to set up a new multi-account AWS environment with guardrails. Walk me through:
> - How Control Tower relates to AWS Organizations
> - What **landing zone** means and how it is structured
> - How **mandatory guardrails** and **strongly recommended guardrails** work
> - How to **enroll existing accounts** into Control Tower
> - What happens when a guardrail is violated — detection vs prevention

📁 **Reference:** `nawab312/AWS` — Control Tower, landing zone, guardrails, AWS Organizations sections

---

### Q381 — Kubernetes | Troubleshooting | Advanced

> Your **Kubernetes Ingress** is returning `502 Bad Gateway` for specific endpoints but not all. The backend Pods are healthy and responding correctly when called directly via `kubectl port-forward`.
>
> Walk me through diagnosing Ingress 502 errors — covering: Ingress controller logs, backend readiness probe timing, connection refused vs connection reset scenarios, upstream keepalive settings, and large request body handling.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` and `14_TROUBLESHOOTING.md` — Ingress 502, Nginx configuration, upstream sections

---

### Q382 — Grafana | Conceptual | Advanced

> Explain **Grafana `Explore`** mode — how does it differ from dashboards and when would you use it?
>
> What is **split view** in Explore and how does it help correlate metrics and logs? Explain **`derived fields`** in Grafana Loki — how do you configure a derived field that extracts a `traceId` from a log line and creates a clickable link to the corresponding Tempo trace?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — Explore, derived fields, Loki, Tempo correlation sections

---

### Q383 — Terraform | Troubleshooting | Advanced

> Your Terraform apply is failing with:
>
> `Error: Error modifying DB instance: InvalidParameterCombination: Cannot upgrade postgres from 14.7 to 14.9. A maintenance window must be specified`
>
> And separately:
>
> `Error: error updating Lambda Function: ResourceConflictException: The function is currently in the following state: Pending`
>
> Diagnose and resolve both errors — what are the root causes and what Terraform configurations prevent these in future?

📁 **Reference:** `nawab312/Terraform` — RDS maintenance window, Lambda state, apply error resolution sections

---

### Q384 — Python | Scenario-Based | Advanced

> Write a **Python script** that implements an **automated runbook executor**:
> - Reads a YAML runbook file defining steps (shell commands, HTTP calls, Kubernetes commands)
> - Executes steps **sequentially** with configurable timeout per step
> - On step failure: retries up to N times with exponential backoff
> - Supports **conditional steps** (only run if previous step output matches pattern)
> - Captures and stores **output of each step** in a structured execution log
> - Generates a **JSON execution report** with step results, durations, and overall status
> - Supports **dry-run mode** that shows what would be executed without running it

📁 **Reference:** `nawab312/DSA` → `Python` — subprocess, YAML, retry logic, runbook automation sections

---

### Q385 — AWS | Conceptual | Advanced

> Explain **AWS `Transit Gateway Connect`** and **GRE tunnel attachments** for connecting SD-WAN solutions.
>
> Also explain **AWS `Cloud WAN`** — how does it build on Transit Gateway to provide a managed global WAN? What is a **core network policy** and how does it define routing domains and segments? When would you use Cloud WAN vs a traditional Transit Gateway hub-and-spoke architecture?

📁 **Reference:** `nawab312/AWS` — Transit Gateway Connect, Cloud WAN, network architecture sections

---

### Q386 — Kubernetes | Scenario-Based | Advanced

> You need to implement **Kubernetes `ResourceClaim`** and **Dynamic Resource Allocation (DRA)** (Kubernetes 1.26+) for managing custom hardware resources (FPGAs, special network cards) that go beyond what standard GPU support offers.
>
> Explain what problem DRA solves that `extended resources` and `device plugins` cannot. What are `ResourceClass`, `ResourceClaim`, and `ResourceClaimTemplate` objects? How does DRA enable **structured parameters** for fine-grained resource allocation?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — Dynamic Resource Allocation, device plugins sections

---

### Q387 — ELK Stack | Scenario-Based | Advanced

> You need to implement **Elasticsearch `snapshot and restore`** for disaster recovery:
> - Automated daily snapshots to **S3**
> - Snapshots of specific indices only (not system indices)
> - **Incremental snapshots** to minimize storage cost
> - Restore procedure for partial index restore (specific date range only)
> - **Test restore** in a separate cluster monthly
>
> Write the complete Elasticsearch API calls for setup, scheduling, and restore.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — snapshot/restore, S3 repository, SLM (Snapshot Lifecycle Management) sections

---

### Q388 — Jenkins | Conceptual | Advanced

> Explain **Jenkins `Configuration as Code (JCasC)`** plugin. What problem does it solve?
>
> Write a JCasC YAML configuration that:
> - Configures Jenkins system settings (executors, admin email)
> - Sets up a **GitHub credentials** entry
> - Configures a **Kubernetes cloud** for dynamic agents
> - Sets up a **Multibranch Pipeline** job pointing to a GitHub repository
> - Enables specific **security settings** (disable signup, CSRF protection)

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — JCasC, configuration as code, declarative Jenkins setup sections

---

### Q389 — Linux / Bash | Conceptual | Advanced

> Explain **Linux `mmap`** system call. What is it, how does it work, and when would a process use it?
>
> What is the difference between **anonymous `mmap`** and **file-backed `mmap`**? How does `mmap` relate to **copy-on-write (CoW)** in container runtimes? How does Docker use CoW with overlay filesystems to share layers efficiently between containers?

📁 **Reference:** `nawab312/DSA` → `Linux` and `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md` — mmap, CoW, overlay filesystem sections

---

### Q390 — AWS | Scenario-Based | Advanced

> Your company needs to implement **AWS `Verified Access`** to provide zero-trust access to internal web applications without a VPN. Employees should be able to access internal apps from any device after **identity verification** through an IdP (Okta).
>
> Walk me through the architecture — how Verified Access integrates with **AWS IAM Identity Center**, trust providers, access policies, and how it differs from a traditional VPN + bastion approach.

📁 **Reference:** `nawab312/AWS` — Verified Access, zero-trust, IAM Identity Center, access policies sections

---

### Q391 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `server-side apply`** vs **client-side apply**. What problem does server-side apply solve?
>
> What is a **field manager** and how does it prevent conflicts when multiple controllers or users manage different fields of the same resource? What is **field ownership** and how does it handle merge conflicts differently from client-side apply? When would you use `--force-conflicts` flag?

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md` — server-side apply, field managers, ownership sections

---

### Q392 — GitHub Actions | Scenario-Based | Advanced

> You need to implement a **GitHub Actions workflow** that enforces **compliance checks** before merging:
> - Scans Terraform code with **`tfsec`** and blocks PRs with HIGH severity findings
> - Runs **`checkov`** on Kubernetes manifests and blocks if policy violations found
> - Checks that all Docker images use **specific approved base images** only
> - Verifies all new AWS resources have **required tags** (using `infracost` or `conftest`)
> - Posts a **detailed compliance report** as a PR comment
>
> Write the complete workflow with proper failure conditions and report formatting.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` and `nawab312/Terraform` — compliance scanning, tfsec, checkov, PR comments sections

---

### Q393 — Terraform | Conceptual | Advanced

> Explain **Terraform `lifecycle` meta-argument** in depth — all 5 options: `create_before_destroy`, `prevent_destroy`, `ignore_changes`, `replace_triggered_by`, and `postcondition`.
>
> Give a real-world example for EACH option where it is the correct solution. What happens when `create_before_destroy` conflicts with a unique constraint (e.g., DNS record or S3 bucket name)?

📁 **Reference:** `nawab312/Terraform` — lifecycle meta-argument, all options, real-world usage sections

---

### Q394 — Prometheus | Troubleshooting | Advanced

> Your Prometheus instance is showing high `prometheus_rule_evaluation_duration_seconds` — rule evaluations are taking 30+ seconds when they should take milliseconds. This is causing alerts to fire late or not at all.
>
> What causes slow rule evaluations? Walk me through diagnosing which specific rules are slow, why they are slow (high cardinality, expensive joins, missing recording rules), and how to optimize them.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — rule evaluation performance, recording rules optimization sections

---

### Q395 — AWS | Conceptual | Advanced

> Explain **AWS `Systems Manager (SSM)`** in depth — what are the key capabilities?
>
> Explain these specific features with real-world use cases:
> - **Session Manager** vs Bastion hosts
> - **Run Command** vs Ansible/Chef
> - **Patch Manager** — automated patching strategy
> - **Automation** — runbook automation
> - **Inventory** — asset management
>
> How would you use SSM to completely eliminate bastion hosts from your architecture?

📁 **Reference:** `nawab312/AWS` — SSM, Session Manager, Run Command, Patch Manager sections

---

### Q396 — Kubernetes | Scenario-Based | Advanced

> You are deploying a **multi-region Kubernetes application** where the same service runs in `us-east-1` and `eu-west-1`. You need **global load balancing** that:
> - Routes users to the **nearest region** based on latency
> - Automatically **fails over** to the other region if one is unhealthy
> - Keeps **session state** consistent across regions
> - Handles **database write** routing (writes always go to primary region)
>
> Design the complete architecture using AWS Global Accelerator, Route53, EKS, and DynamoDB Global Tables.

📁 **Reference:** `nawab312/AWS` and `nawab312/Kubernetes` — Global Accelerator, Route53 latency routing, multi-region EKS sections

---

### Q397 — ArgoCD | Conceptual | Advanced

> Explain **ArgoCD `Resource Hooks`** — what are the 5 hook types (`PreSync`, `Sync`, `PostSync`, `SyncFail`, `PostDelete`) and when does each run?
>
> Give a real-world example for each hook type. Write a `PreSync` hook that:
> - Checks if the database has pending migrations
> - Runs the migration Job
> - Fails the sync if migration fails (preventing partial deployment)
> - Cleans up the migration Job after completion

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — resource hooks, sync lifecycle, PreSync, PostSync sections

---

### Q398 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that implements **automated capacity planning** for a Kubernetes cluster:
> - Queries Prometheus API for CPU and memory usage trends over the last 30 days
> - Calculates **growth rate** per namespace (linear regression on usage data)
> - Projects when each namespace will **hit its ResourceQuota limit**
> - Identifies nodes that will run out of **allocatable resources** in the next 30 days
> - Generates a **capacity planning report** in markdown format
> - Sends weekly report via email and posts to Slack

📁 **Reference:** `nawab312/DSA` → `Linux` — Prometheus API, bash scripting, capacity planning, curl sections

---

### Q399 — AWS | Scenario-Based | Advanced

> Your company is implementing **AWS `Well-Architected Framework`** review for a production application. During the review, these issues are identified:
> - No **automated failover** for RDS
> - EC2 instances in single AZ
> - No **backup testing** — backups exist but are never verified
> - IAM users with **console access and no MFA**
> - S3 buckets with **no access logging**
>
> For each finding, provide the specific AWS service/configuration to remediate it and explain the Well-Architected pillar it addresses.

📁 **Reference:** `nawab312/AWS` — Well-Architected Framework, 6 pillars, remediation sections

---

### Q400 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `Network Plugin (CNI)` selection criteria**. Compare **Flannel**, **Calico**, **Weave**, **Cilium**, and **AWS VPC CNI** across these dimensions:
> - **Performance** (overlay vs underlay networking)
> - **NetworkPolicy support** (basic vs advanced L7)
> - **Scalability** (max nodes/pods)
> - **Observability** built-in
> - **When to choose each** in production

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — CNI plugins, network architecture, overlay vs underlay sections

---

### Q401 — Grafana | Scenario-Based | Advanced

> You need to implement a **Grafana reporting system** that:
> - Generates a **weekly PDF report** of key business and infrastructure metrics
> - Emails the report to **C-level executives** every Monday at 9 AM
> - The report includes: uptime SLO status, deployment frequency, MTTR, top 5 cost-consuming services
> - Report branding matches the company style (logo, colors)
>
> Walk me through using **Grafana Reporting** (Enterprise) or the **Grafana API + Playwright/puppeteer** approach for the open-source alternative.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — reporting, PDF export, Grafana API sections

---

### Q402 — Terraform | Scenario-Based | Advanced

> You need to implement **Terraform `drift detection` as a scheduled process** in your organization. Design a system that:
> - Runs `terraform plan` for ALL environments every 6 hours
> - Parses plan output to identify **which resources have drifted**
> - Categorizes drift by **severity** (destructive changes vs updates)
> - Creates **GitHub Issues** automatically for unresolved drift
> - **Auto-remediates** low-risk drift (tag changes) but escalates destructive drift
> - Tracks **drift trends** over time with metrics

📁 **Reference:** `nawab312/Terraform` and `nawab312/CI_CD` — drift detection, scheduled plans, automation sections

---

### Q403 — Git | Scenario-Based | Advanced

> Your team is migrating from **SVN (Subversion) to Git**. The SVN repository has 10 years of history, 5000 commits, multiple trunk/branches/tags, and SVN `externals` that reference other repositories.
>
> Walk me through the complete migration:
> - Using `git svn` or `svn2git` to migrate history
> - Handling SVN **branches and tags** → Git branches and tags
> - Converting **SVN externals** to Git submodules or subtrees
> - Validating the migration is complete and accurate
> - Cutting over the team with minimal disruption

📁 **Reference:** `nawab312/CI_CD` → `Git` — SVN to Git migration, git svn, history preservation sections

---

### Q404 — AWS + Kubernetes | Scenario-Based | Advanced

> You are implementing **GitOps for database schema migrations** in a Kubernetes environment with EKS and RDS PostgreSQL. The challenge is:
> - Schema migrations must run **before** the new application version starts
> - Migrations must be **idempotent** (safe to run multiple times)
> - Failed migrations must **block the deployment** (not allow broken app to start)
> - Migration history must be tracked and **auditable**
> - **Rollback** of a migration must be handled carefully (not all migrations are reversible)
>
> Design the complete implementation using ArgoCD hooks, Kubernetes Jobs, Flyway/Liquibase, and RDS.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD`, `nawab312/Kubernetes` → `02_WORKLOADS.md`, `nawab312/AWS` — database migrations, GitOps, PreSync hooks sections

---

### Q405 — All Topics | System Design | Advanced

> You are the **Principal Engineer** tasked with designing a **cloud-native platform migration** for a traditional enterprise:
> - Currently: 500 VMs on-premises, Oracle DB, WebLogic app servers, manual deployments
> - Target: Kubernetes on AWS, PostgreSQL on RDS, containerized microservices, fully automated CI/CD
> - Constraint: Cannot have more than **4 hours downtime** during migration
> - Timeline: 18 months
> - Team: 15 developers, 3 DevOps engineers
>
> Design the **phased migration roadmap** covering: assessment, containerization strategy, CI/CD pipeline building, data migration, traffic cutover, and post-migration optimization. Include which tools from your stack you would use at each phase.

📁 **Reference:** All repositories — enterprise migration, phased approach, cloud-native transformation

#### Key Points to Cover:
```
Phase 1 (Months 1–4): Foundation & Assessment
Start by auditing all 500 VMs — classify workloads into lift-and-shift, refactor, replace, or retire.
Build the AWS landing zone (VPCs, IAM, networking),
set up the CI/CD skeleton (GitLab/GitHub Actions + ArgoCD),
and begin containerizing stateless services first.
Oracle → PostgreSQL schema conversion analysis happens here using AWS Schema Conversion Tool.

Phase 2 (Months 5–10): Parallel Running
Run on-prem and AWS environments simultaneously.
Migrate microservices in waves using the strangler fig pattern — peel functionality off WebLogic monoliths incrementally.
Stand up RDS PostgreSQL, validate data fidelity using DMS, and
keep Oracle as the source of truth until cutover.
The 15 developers should be working in squads aligned to service boundaries.

Phase 3 (Months 11–16): Data Migration & Cutover
This is where the 4-hour downtime constraint becomes critical.
Use AWS DMS for continuous replication → stop writes → final sync → DNS flip → validate → announce done.
Blue/green deployment at the DNS layer is the key technique.
Any rollback must be executable in under 30 minutes.

Phase 4 (Months 17–18): Optimization & Decommission
Tune autoscaling, right-size RDS, implement observability (Prometheus + Grafana + OpenTelemetry), shut down on-prem VMs, and conduct a full retrospective.
```

> 💡 **Interview tip:** Lead with risk, not technology. Interviewers want to know you think about what can go wrong before you think about what's cool. Open every answer with the constraint that shapes the decision — in this case, the **4-hour downtime window**.

> 💡 **Interview tip:** Name the pattern before you explain it. Say **"strangler fig"** or **"blue/green"** before explaining how it works. It signals fluency; the explanation then becomes proof you actually understand it, not just memorized the term.

> 💡 **Interview tip:** Show you can communicate with both audiences. Engineers want to hear about **DMS CDC latency** and **Kubernetes readiness probes**. Executives want to hear about **rollback time** and **business continuity**. Practice switching registers in the same answer.

> 💡 **Interview tip:** The 4-hour window is the trap question. Interviewers will probe this hard. The right answer is: the window isn't about migration time, it's about **write quiescence**. You achieve this through continuous replication with DMS, so the actual downtime is only the final sync + DNS TTL expiry + smoke test. Have numbers ready: **DMS lag under 30 seconds**, **DNS TTL pre-lowered to 60 seconds** 48 hours before cutover.

> 💡 **Interview tip:** Anchor the team structure to the strategy. Don't just say "15 developers in squads." Say **"3 squads aligned to service domains — one per major bounded context — so Oracle schema ownership maps cleanly to a team. The 3 DevOps engineers own the platform layer and the DMS pipeline, not individual services."** This shows systems thinking.

> 💡 **Interview tip:** Know your failure modes. What happens if the final DMS sync takes longer than expected? *(Abort, roll back DNS, root cause in DMS metrics, reschedule.)* What if a PostgreSQL type conversion breaks in production? *(Schema validation suite run in staging with production data snapshots, `pg_dump` comparison.)* Having practiced failure paths is what separates **principal-level answers** from senior-level ones.

---

## Key Topics Coverage (Q361–Q405)

| Topic | Questions |
|---|---|
| Kubernetes | Q361, Q366, Q371, Q377, Q381, Q386, Q391, Q396, Q400 |
| AWS | Q362, Q368, Q375, Q380, Q385, Q390, Q395, Q399, Q404 |
| Terraform | Q364, Q373, Q383, Q393, Q402 |
| Prometheus | Q365, Q376, Q394 |
| Grafana | Q382, Q401 |
| ELK Stack | Q370, Q387 |
| Linux / Bash | Q363, Q372, Q379, Q389, Q398 |
| ArgoCD | Q378, Q397 |
| Jenkins | Q369, Q388 |
| GitHub Actions | Q374, Q392 |
| Git | Q367, Q403 |
| Python | Q384 |
| System Design | Q405 |

---

## New Concepts Introduced in This Set

### Kubernetes
- Native SidecarContainers (K8s 1.29 GA) — lifecycle guarantees
- kube-proxy iptables rule update diagnosis
- PIDPressure (new angle — per-pod PID limits)
- Dynamic Resource Allocation (DRA) — ResourceClaim, ResourceClaimTemplate
- Server-side apply — field managers, ownership, merge conflicts
- HPA v2 behavior — multiple policies, selectPolicy
- CNI plugin comparison — Flannel vs Calico vs Weave vs Cilium vs VPC CNI
- EndpointSlice topology hints (new angle — AZ routing trade-offs)
- Cluster federation — KubeFed, Admiralty, Liqo approaches

### AWS
- AWS Backup — centralized, cross-region, restore testing
- RDS Proxy — connection pinning, Lambda IAM auth
- Control Tower — landing zone, guardrail types, account enrollment
- AWS Verified Access — zero-trust without VPN
- Transit Gateway Connect + Cloud WAN
- AWS Well-Architected Framework — 6 pillars remediation
- DynamoDB hot partitions — adaptive capacity, burst capacity (new angle)
- Global Accelerator + Route53 multi-region failover
- AWS SSM — eliminating bastions completely
- ECR authentication failures — ECS task execution role

### Terraform
- `templatefile()` vs `file()` vs `templatestring()`
- State locking bottleneck — sharded strategy
- `null_resource` / `terraform_data` — triggers mechanism
- `lifecycle` all 5 options with real-world examples
- `sensitive` values — limitations, state file exposure (new angle)
- RDS maintenance window + Lambda state apply errors

### Prometheus
- PushGateway — when to use vs anti-patterns
- Stale markers — lookbackDelta, rate() interaction
- Rule evaluation performance — slow rules diagnosis
- Alertmanager routing tree (new angle — group settings detail)

### Linux
- cgroups v2 unified hierarchy — cpu.max, memory.oom_group
- Kernel soft lockup — watchdog, investigation
- `mmap` — anonymous vs file-backed, CoW in containers
- `git filter-repo` — 4 real use cases with exact commands

### Git
- `git bundle` — air-gapped (new angle — exact commands)
- `git filter-repo` vs `git filter-branch` — advantages
- SVN to Git migration — complete workflow
- Diverged branch merge — conflict resolution at scale (new angle)

### ArgoCD
- ApplicationSet Matrix generator — multi-cluster × multi-env
- Resource Hooks — all 5 types with examples
- Notifications — triggers, templates, GitHub commit status (new angle)

### ELK Stack
- Percolator queries
- scroll API vs search_after vs pit — when to use each
- Elasticsearch snapshot/restore — SLM, incremental, partial restore

### Grafana
- Explore mode — split view, derived fields, Tempo links
- Grafana reporting — PDF generation, scheduling
- Transformations (new angle — join, filter, calculate)

---

*Part 9 of DevOps/SRE Interview Questions — nawab312 GitHub repositories*
*Total question bank: Q1–Q405*
