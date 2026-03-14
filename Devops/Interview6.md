# DevOps / SRE / Cloud Interview Questions (226–270)

> Part 6 of interview preparation questions based on your GitHub repositories.
> All questions are unique — not repeated from Q1–Q225.

---

## Table of Contents

| # | Topic | Type | Difficulty |
|---|---|---|---|
| 226 | Kubernetes | Conceptual | Advanced |
| 227 | AWS | Scenario-Based | Advanced |
| 228 | Linux / Bash | Conceptual | Advanced |
| 229 | Terraform | Scenario-Based | Advanced |
| 230 | Prometheus | Conceptual | Advanced |
| 231 | Kubernetes | Troubleshooting | Advanced |
| 232 | Git | Scenario-Based | Advanced |
| 233 | AWS | Conceptual | Advanced |
| 234 | Jenkins | Troubleshooting | Advanced |
| 235 | ELK Stack | Conceptual | Advanced |
| 236 | Kubernetes | Scenario-Based | Advanced |
| 237 | Linux / Bash | Troubleshooting | Advanced |
| 238 | Terraform | Conceptual | Advanced |
| 239 | GitHub Actions | Conceptual | Advanced |
| 240 | AWS | Troubleshooting | Advanced |
| 241 | Prometheus | Troubleshooting | Advanced |
| 242 | Kubernetes | Conceptual | Advanced |
| 243 | ArgoCD | Scenario-Based | Advanced |
| 244 | Linux / Bash | Scenario-Based | Advanced |
| 245 | AWS | Scenario-Based | Advanced |
| 246 | Kubernetes | Troubleshooting | Advanced |
| 247 | Grafana | Troubleshooting | Advanced |
| 248 | Terraform | Troubleshooting | Advanced |
| 249 | Python | Scenario-Based | Advanced |
| 250 | AWS | Conceptual | Advanced |
| 251 | Kubernetes | Scenario-Based | Advanced |
| 252 | ELK Stack | Scenario-Based | Advanced |
| 253 | Jenkins | Conceptual | Advanced |
| 254 | Linux / Bash | Conceptual | Advanced |
| 255 | AWS | Scenario-Based | Advanced |
| 256 | Kubernetes | Conceptual | Advanced |
| 257 | GitHub Actions | Scenario-Based | Advanced |
| 258 | Terraform | Scenario-Based | Advanced |
| 259 | Prometheus | Scenario-Based | Advanced |
| 260 | AWS | Conceptual | Advanced |
| 261 | Kubernetes | Scenario-Based | Advanced |
| 262 | ArgoCD | Conceptual | Advanced |
| 263 | Linux / Bash | Scenario-Based | Advanced |
| 264 | AWS | Scenario-Based | Advanced |
| 265 | Kubernetes | Conceptual | Advanced |
| 266 | Grafana | Scenario-Based | Advanced |
| 267 | Terraform | Conceptual | Advanced |
| 268 | Git | Conceptual | Advanced |
| 269 | AWS + Kubernetes | Troubleshooting | Advanced |
| 270 | All Topics | System Design | Advanced |

---

## Questions

---

### Q226 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `ownerReferences`** and **garbage collection**. How does Kubernetes know to delete child resources when a parent is deleted?
>
> What is the difference between **foreground** and **background** cascading deletion? Give a real-world scenario where you would use `--cascade=orphan` to delete a parent without deleting its children.

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md` — ownerReferences, garbage collection, deletion cascade sections

---

### Q227 — AWS | Scenario-Based | Advanced

> Your company uses **AWS Config** and receives a compliance violation:
>
> `NON_COMPLIANT: S3 bucket 'prod-data-bucket' does not have server-side encryption enabled`
>
> There are 200 S3 buckets in the account. How would you:
> 1. Fix the immediate violation
> 2. Audit ALL buckets for encryption compliance
> 3. Auto-remediate using AWS Config rules + SSM Automation
> 4. Prevent new non-compliant buckets using SCPs

📁 **Reference:** `nawab312/AWS` — AWS Config, S3 encryption, auto-remediation, SCPs sections

---

### Q228 — Linux / Bash | Conceptual | Advanced

> Explain **Linux `SELinux`** and **`AppArmor`** — what are they, what problem do they solve, and how do they differ?
>
> What are SELinux **contexts**, **types**, and **policies**? What does `enforcing`, `permissive`, and `disabled` mode mean? How does Kubernetes interact with SELinux/AppArmor for container security?

📁 **Reference:** `nawab312/DSA` → `Linux` and `nawab312/Kubernetes` → `07_SECURITY.md` — SELinux, AppArmor, MAC sections

---

### Q229 — Terraform | Scenario-Based | Advanced

> You need to implement **Terraform testing** for your infrastructure modules. Walk me through:
> 1. Using **`terraform validate`** and **`terraform fmt`** in CI
> 2. Writing unit tests with **Terratest** (Go)
> 3. Using **`terraform plan`** output for contract testing
> 4. Integration testing against a real AWS environment
> 5. Cost estimation using **Infracost** in CI pipelines

📁 **Reference:** `nawab312/Terraform` and `nawab312/CI_CD` — Terraform testing, validation, Terratest sections

---

### Q230 — Prometheus | Conceptual | Advanced

> Explain **Prometheus `scrape_config` relabeling**. What is `relabel_configs` vs `metric_relabel_configs` and when is each applied?
>
> Write relabeling rules that:
> - Drop all metrics where the `env` label is `dev`
> - Rename the label `pod_name` to `pod`
> - Add a static label `cluster="prod-us-east"` to all metrics
> - Drop any metric where the value is `0`

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — relabeling, scrape configuration sections

---

### Q231 — Kubernetes | Troubleshooting | Advanced

> Your Kubernetes cluster has nodes in `MemoryPressure` condition. Pods are being evicted but new pods are immediately evicted again as soon as they are scheduled.
>
> Walk me through diagnosing the memory pressure — what metrics and conditions to check, how eviction thresholds work (`evictionHard` vs `evictionSoft`), and how to resolve the situation without adding new nodes.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — eviction, memory pressure, kubelet configuration sections

---

### Q232 — Git | Scenario-Based | Advanced

> Your team maintains a **library** used by 10 other repositories as a Git submodule. A breaking change needs to be deployed to all 10 repositories simultaneously.
>
> Walk me through the **complete workflow** for managing Git submodules — how to update a submodule, pin specific versions, update all dependent repositories, and the pitfalls of submodules that make teams prefer alternatives like **package managers or monorepos**.

📁 **Reference:** `nawab312/CI_CD` → `Git` — submodules, dependency management sections

---

### Q233 — AWS | Conceptual | Advanced

> Explain **AWS EventBridge** (formerly CloudWatch Events). What types of event sources does it support?
>
> Design an event-driven architecture using EventBridge that:
> - Triggers a Lambda when an EC2 instance is **terminated unexpectedly**
> - Triggers a Step Function when an RDS backup **fails**
> - Routes **GuardDuty findings** to a security Slack channel
> - Schedules a **weekly cost report** generation

📁 **Reference:** `nawab312/AWS` — EventBridge, event-driven architecture, rules sections

---

### Q234 — Jenkins | Troubleshooting | Advanced

> Your Jenkins pipeline is **randomly failing** with `hudson.remoting.ChannelClosedException: channel is already closed` errors. The failure happens at different stages each time and cannot be reproduced consistently.
>
> What causes Jenkins channel closed exceptions, how do you diagnose them, and what are the fixes?

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — agent connectivity, channel errors, network stability sections

---

### Q235 — ELK Stack | Conceptual | Advanced

> Explain **Elasticsearch `mapping`** — what is dynamic mapping vs explicit mapping? What are **`text`** vs **`keyword`** field types and when would you use each?
>
> What is **`mapping explosion`** and how do you prevent it? Write an explicit mapping for an application log document that has: timestamp, log level, service name, message, and a dynamic `metadata` object.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Elasticsearch mapping, field types sections

---

### Q236 — Kubernetes | Scenario-Based | Advanced

> You are running a **machine learning training job** on Kubernetes that requires:
> - Access to **GPU resources** (NVIDIA A100)
> - Very large datasets mounted from **NFS/EFS**
> - Job should run to completion even if it takes **48 hours**
> - If the node fails mid-job, the job should **resume from checkpoint** not restart from scratch
> - Only 2 GPU nodes exist — job must not block other workloads from CPU nodes
>
> Design the complete Kubernetes configuration.

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md` and `04_STORAGE.md` — Jobs, GPU resources, PVC sections

---

### Q237 — Linux / Bash | Troubleshooting | Advanced

> A critical production process is hanging — it is stuck in **`D` state** (uninterruptible sleep) and cannot be killed even with `kill -9`.
>
> What does the `D` state mean? What is causing it? Why can `SIGKILL` not kill a process in `D` state? What are your options to recover the system without a reboot?

📁 **Reference:** `nawab312/DSA` → `Linux` — process states, D state, I/O wait, kernel hang sections

---

### Q238 — Terraform | Conceptual | Advanced

> Explain **Terraform `check` blocks** and **`precondition`/`postcondition`** assertions introduced in Terraform 1.2+.
>
> What is the difference between a `precondition` (on a resource/data source) and a `check` block? Write examples that:
> - Validate that an AMI ID is not older than 90 days before creating an EC2 instance
> - Assert that a created ALB has at least 2 subnets in different AZs
> - Check that an external API is reachable before applying infrastructure

📁 **Reference:** `nawab312/Terraform` — validation, check blocks, preconditions sections

---

### Q239 — GitHub Actions | Conceptual | Advanced

> Explain **GitHub Actions `concurrency`** groups. What problem do they solve?
>
> Write workflow configurations for these scenarios:
> - Cancel in-progress deployments when a new commit is pushed to the same branch
> - Allow only one deployment to production at a time (queue others, don't cancel)
> - Allow parallel CI runs per branch but only one deployment per environment

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — concurrency, deployment queuing sections

---

### Q240 — AWS | Troubleshooting | Advanced

> Your **AWS Lambda function** is failing with:
>
> `Task timed out after 15.00 seconds`
>
> But when you test the same function locally it completes in 2 seconds. The function queries RDS and calls an external API.
>
> What causes Lambda functions to timeout specifically in AWS but not locally, and walk me through diagnosing each possible cause?

📁 **Reference:** `nawab312/AWS` — Lambda, VPC configuration, cold starts, RDS connectivity sections

---

### Q241 — Prometheus | Troubleshooting | Advanced

> Prometheus is showing `ALERTS` metric in firing state but **Alertmanager is NOT sending notifications**. The alert shows as `firing` in Prometheus UI but nothing appears in Alertmanager UI either.
>
> Walk me through diagnosing the break in the Prometheus → Alertmanager → notification pipeline.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — Alertmanager integration, alert routing, inhibition sections

---

### Q242 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `ResourceQuota` for non-compute resources**. What objects beyond CPU and memory can you quota?
>
> Also explain **`PriorityLevelConfiguration`** and **`FlowSchema`** — how does Kubernetes API Priority and Fairness (APF) prevent one misbehaving client from starving others of API server capacity?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` and `01_CORE_ARCHITECTURE_FUNDAMENTALS.md` — ResourceQuota, API Priority sections

---

### Q243 — ArgoCD | Scenario-Based | Advanced

> You need to implement **ArgoCD Image Updater** to automatically update container image tags in Git when a new image is pushed to ECR.
>
> Walk me through:
> - How Image Updater works
> - Configuration for tracking `semver` tags (only update on new minor versions)
> - How it commits back to Git (write-back method)
> - How to prevent it from updating production automatically but allow staging

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — Image Updater, automated image tracking sections

---

### Q244 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that implements a **database backup validator**:
> - Takes a PostgreSQL backup file (`.sql.gz`) as input
> - Restores it to a **temporary Docker container**
> - Runs **validation queries** (row counts, key table existence, referential integrity checks)
> - Reports pass/fail for each validation
> - Cleans up the Docker container regardless of success/failure
> - Exits with code `0` if all validations pass, `1` if any fail

📁 **Reference:** `nawab312/DSA` → `Linux` — bash scripting, Docker, PostgreSQL, trap cleanup sections

---

### Q245 — AWS | Scenario-Based | Advanced

> Your team is implementing **AWS Service Control Policies (SCPs)** for a multi-account setup. Write SCPs that:
> - Prevent anyone (including root) from **disabling CloudTrail**
> - Prevent creation of resources **outside approved regions** (us-east-1, eu-west-1 only)
> - Prevent **leaving AWS Organizations**
> - Require all EC2 instances to have a **`CostCenter` tag**
> - Allow **only specific IAM roles** to create/delete production RDS instances

📁 **Reference:** `nawab312/AWS` — AWS Organizations, SCPs, guardrails sections

---

### Q246 — Kubernetes | Troubleshooting | Advanced

> A **StatefulSet** is stuck — it has been in the middle of a rolling update for 2 hours. Pod `my-app-1` is `Terminating` but never finishes terminating. The StatefulSet controller is waiting for it.
>
> What causes StatefulSet pods to get stuck in `Terminating`? Walk me through diagnosing and safely resolving the stuck termination without data loss.

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md` and `14_TROUBLESHOOTING.md` — StatefulSet rolling update, stuck termination, finalizers sections

---

### Q247 — Grafana | Troubleshooting | Advanced

> Your Grafana dashboards are showing **incorrect data** — the metrics appear to show values from the wrong time zone. Some panels show data 5.5 hours behind what Prometheus shows directly.
>
> What causes time zone issues in Grafana and how do you fix them? Also explain what **`$__timeFilter`**, **`$__interval`**, and **`$__range`** template variables are in Grafana and when each should be used.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — time zones, template variables, dashboard variables sections

---

### Q248 — Terraform | Troubleshooting | Advanced

> You are trying to destroy a Terraform-managed resource but it fails with:
>
> `Error: Instance cannot be destroyed. A resource is still referencing this resource.`
>
> And separately, another destroy fails with:
>
> `Error: deleting S3 Bucket: BucketNotEmpty`
>
> How do you handle each error and safely destroy the resources?

📁 **Reference:** `nawab312/Terraform` — destroy dependencies, S3 force destroy, dependency graph sections

---

### Q249 — Python | Scenario-Based | Advanced

> Write a **Python script** that implements an **AWS resource tagger**:
> - Scans all EC2 instances, RDS instances, and S3 buckets across **multiple AWS regions**
> - Identifies resources **missing required tags** (`Environment`, `Team`, `CostCenter`)
> - Generates a **CSV report** of non-compliant resources with account ID, region, resource ID, resource type, and missing tags
> - Optionally applies **default tags** to non-compliant resources (with `--fix` flag)
> - Uses **concurrent.futures** for parallel region scanning

📁 **Reference:** `nawab312/DSA` → `Python` — boto3, concurrent.futures, CSV, multi-region sections

---

### Q250 — AWS | Conceptual | Advanced

> Explain **AWS Direct Connect** vs **AWS VPN** for connecting on-premises networks to AWS.
>
> What are the differences in **bandwidth**, **latency**, **reliability**, and **cost**? What is **Direct Connect Gateway** and how does it enable connectivity to multiple VPCs and regions from a single Direct Connect connection?

📁 **Reference:** `nawab312/AWS` — Direct Connect, VPN, hybrid connectivity sections

---

### Q251 — Kubernetes | Scenario-Based | Advanced

> You need to implement **Kubernetes workload identity** for pods running in EKS to access AWS services **without storing any credentials**. Specifically:
> - Pod A needs `s3:GetObject` on a specific bucket
> - Pod B needs `dynamodb:PutItem` on a specific table
> - Pod C needs `secretsmanager:GetSecretValue` for specific secrets
> - Each pod should have **minimum required permissions** (least privilege)
>
> Walk me through the complete IRSA setup for all three pods.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md` and `nawab312/AWS` — IRSA, OIDC, ServiceAccount IAM sections

---

### Q252 — ELK Stack | Scenario-Based | Advanced

> Your Elasticsearch cluster has **unassigned shards** — the cluster health is `RED`. Some indices are showing `UNASSIGNED` primary shards.
>
> Walk me through the complete process of diagnosing WHY shards are unassigned and the different remediation steps depending on the root cause (disk watermark, node failure, replica configuration, etc).

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — shard allocation, cluster health, disk watermark sections

---

### Q253 — Jenkins | Conceptual | Advanced

> Explain **Jenkins `stash`/`unstash`** vs **Jenkins `archiveArtifacts`**. When would you use each?
>
> Also explain **Jenkins `fingerprinting`** — what it is, how it works, and how it helps track which build produced which artifact across multiple pipelines. Write a Jenkinsfile demonstrating all three mechanisms.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — artifact management, stash, fingerprinting sections

---

### Q254 — Linux / Bash | Conceptual | Advanced

> Explain **Linux `epoll`** vs **`select`** vs **`poll`** for I/O multiplexing. Why does `epoll` scale better for high-concurrency servers?
>
> How does this relate to the **C10K problem**? How do modern web servers like Nginx and tools like Node.js use event loops and `epoll` to handle thousands of concurrent connections with minimal threads?

📁 **Reference:** `nawab312/DSA` → `Linux` — I/O multiplexing, epoll, event-driven architecture sections

---

### Q255 — AWS | Scenario-Based | Advanced

> Your **API Gateway** is returning `429 Too Many Requests` errors to legitimate users even though the backend Lambda function is not being throttled.
>
> Explain all the **different throttling limits** that exist in API Gateway (account-level, stage-level, method-level, usage plans) and how to diagnose which limit is being hit and how to fix it.

📁 **Reference:** `nawab312/AWS` — API Gateway, throttling, usage plans, quotas sections

---

### Q256 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `CSI (Container Storage Interface)`** drivers. What problem did CSI solve that the in-tree volume plugins had?
>
> What are **`CSIDriver`**, **`CSINode`**, and **`VolumeAttachment`** objects? Walk through what happens internally when a Pod mounts an EBS volume — from the PVC being bound to the volume being available inside the container.

📁 **Reference:** `nawab312/Kubernetes` → `04_STORAGE.md` — CSI, volume lifecycle, EBS provisioner sections

---

### Q257 — GitHub Actions | Scenario-Based | Advanced

> You need to implement a **GitHub Actions workflow** for **infrastructure drift detection**:
> - Runs `terraform plan` on a **schedule** (every 6 hours)
> - If drift is detected, **creates a GitHub Issue** with the plan output
> - **Labels the issue** by severity (destructive changes = P1, modifications = P2, additions = P3)
> - If the same drift issue already exists (open), **comments on it** instead of creating a new one
> - Sends a **Slack notification** only for P1 (destructive) drift
>
> Write the complete workflow.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` and `nawab312/Terraform` — scheduled workflows, GitHub API, drift detection sections

---

### Q258 — Terraform | Scenario-Based | Advanced

> You need to manage **Kubernetes resources** (Deployments, Services, ConfigMaps) using Terraform alongside your AWS infrastructure. Your team is debating between using the **Terraform Kubernetes provider** vs managing K8s resources with **Helm provider** vs keeping K8s manifests in a separate **ArgoCD-managed Git repo**.
>
> Explain the trade-offs of each approach and give your recommendation with justification.

📁 **Reference:** `nawab312/Terraform` and `nawab312/CI_CD` → `ArgoCD` — Terraform K8s provider, Helm provider, GitOps separation sections

---

### Q259 — Prometheus | Scenario-Based | Advanced

> You need to implement **multi-cluster monitoring** — you have 5 Kubernetes clusters (dev, staging, prod-us, prod-eu, prod-ap) and need a **single Grafana instance** to query metrics from all clusters.
>
> Design the monitoring architecture using either:
> - Prometheus federation
> - Thanos
> - Grafana Agent with remote_write
>
> Explain which you would choose and why, with the complete architecture.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` and `Grafana` — multi-cluster monitoring, Thanos, federation sections

---

### Q260 — AWS | Conceptual | Advanced

> Explain **AWS Fargate** networking in depth. What is the **`awsvpc` network mode** and how does it differ from `bridge` and `host` modes in ECS?
>
> When a Fargate task starts, how does it get its IP address? How do Fargate tasks communicate with each other and with the internet? What are the networking limitations of Fargate compared to EC2-based ECS tasks?

📁 **Reference:** `nawab312/AWS` — Fargate, awsvpc network mode, task networking sections

---

### Q261 — Kubernetes | Scenario-Based | Advanced

> You are implementing **Kubernetes `PodSecurityAdmission`** (the replacement for deprecated PodSecurityPolicy). Your cluster needs:
> - `kube-system` namespace → **privileged** (system components need root)
> - `monitoring` namespace → **baseline** (node-exporter needs hostPath)
> - `production` namespace → **restricted** (strict security, no root)
> - Any violation in `production` → **deny** the pod, log the violation
>
> Write the complete configuration using PodSecurity admission labels and explain each security level.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md` — PodSecurityAdmission, security standards sections

---

### Q262 — ArgoCD | Conceptual | Advanced

> Explain how **ArgoCD handles Helm chart deployments** vs raw Kubernetes manifests vs Kustomize.
>
> What is the difference between ArgoCD deploying a Helm chart vs running `helm install` directly? What Helm features does ArgoCD **NOT support** and why? How do you pass **Helm values** differently between environments in ArgoCD?

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — Helm integration, Kustomize, tool detection sections

---

### Q263 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that implements a **GitOps-style config drift detector** for server configurations:
> - Reads expected configuration from a **Git repository** (pulled fresh each run)
> - Checks actual system state: installed packages, running services, file contents, sysctl values
> - Reports **drift** between expected and actual state
> - Can run in `--check` mode (report only) or `--fix` mode (apply changes)
> - Generates a **JSON report** of all drift found
> - Sends alert to Slack if drift is detected in `--check` mode

📁 **Reference:** `nawab312/DSA` → `Linux` — configuration management, bash scripting, git, sysctl sections

---

### Q264 — AWS | Scenario-Based | Advanced

> Your company runs a **SaaS application** where each customer gets their own isolated environment. You have 500 customers and need to provision new customer infrastructure automatically when they sign up.
>
> Design an **automated tenant provisioning system** on AWS that:
> - Creates isolated VPC, RDS, S3 bucket per customer
> - Takes under **5 minutes** from signup to ready
> - Uses infrastructure as code
> - Handles **deprovisioning** when customer churns
> - Tracks all tenant resources for billing

📁 **Reference:** `nawab312/AWS` and `nawab312/Terraform` — multi-tenant, automated provisioning, Lambda, Step Functions sections

---

### Q265 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `Finalizers`**. What are they, how do they work, and what problems do they solve?
>
> What happens when you `kubectl delete` a resource that has a finalizer? Give a real-world example where finalizers prevent data loss. What happens if a finalizer controller crashes — how do you safely remove a stuck finalizer?

📁 **Reference:** `nawab312/Kubernetes` → `10_CRDs_EXTENSIBILITY.md` — finalizers, deletion lifecycle sections

---

### Q266 — Grafana | Scenario-Based | Advanced

> Your team wants to implement **Grafana annotations** to correlate deployments with metric changes on dashboards.
>
> Walk me through:
> - Adding deployment annotations automatically from your CI/CD pipeline via Grafana API
> - Showing annotations on all dashboards that show metrics for the deployed service
> - Using annotation queries from a datasource (e.g., querying deployment events from Elasticsearch)
> - Setting up **annotation alerts** — marking periods when an alert was firing

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — annotations, API, dashboard correlation sections

---

### Q267 — Terraform | Conceptual | Advanced

> Explain **Terraform `dynamic` blocks**. What problem do they solve and when should you use them vs not use them?
>
> Write a Terraform resource using dynamic blocks that creates an **AWS Security Group** where:
> - Ingress rules come from a variable list of port/cidr combinations
> - Egress rules are conditionally included based on a boolean variable
> - Tags are dynamically generated from a map variable

📁 **Reference:** `nawab312/Terraform` — dynamic blocks, meta-arguments, expressions sections

---

### Q268 — Git | Conceptual | Advanced

> Explain **`git worktree`**. What problem does it solve and when would a DevOps engineer use it?
>
> Also explain **`git sparse-checkout`** — how does it help with large monorepos where you only need a subset of files? How would you use sparse-checkout in a CI/CD pipeline to speed up checkout time for a 15GB monorepo?

📁 **Reference:** `nawab312/CI_CD` → `Git` — worktree, sparse-checkout, monorepo optimization sections

---

### Q269 — AWS + Kubernetes | Troubleshooting | Advanced

> Your EKS pods are getting `OOMKilled` but the memory usage shown in `kubectl top pods` is much **lower than the configured limit**. The OOMKill is happening even though the pod appears to be well within limits.
>
> Explain why this happens — the difference between **container memory limits** (cgroup), **JVM heap**, **off-heap memory**, and **kernel memory**. How do you properly size memory limits for JVM-based applications on Kubernetes?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` and `nawab312/AWS` — JVM memory, cgroup limits, container memory sections

---

### Q270 — All Topics | System Design | Advanced

> You are the **SRE lead** responsible for a **payments processing system** that must achieve **PCI DSS compliance** while running on Kubernetes on AWS.
>
> Design the complete **compliant infrastructure**:
> - Network segmentation for cardholder data environment (CDE)
> - Encryption in transit and at rest for all payment data
> - Access control — no shared credentials, full audit trail
> - Vulnerability management — automated scanning in CI/CD
> - Incident response — detection, containment, notification within 72 hours
> - Key management using AWS KMS
> - Log integrity — tamper-proof audit logs
>
> Cover both AWS architecture and Kubernetes security configuration.

📁 **Reference:** All repositories — PCI DSS compliance, security architecture design

---

## Key Topics Coverage (Q226–Q270)

| Topic | Questions |
|---|---|
| Kubernetes | Q226, Q231, Q236, Q242, Q246, Q251, Q256, Q261, Q265, Q269 |
| AWS | Q227, Q233, Q240, Q245, Q250, Q255, Q260, Q264, Q269 |
| Terraform | Q229, Q238, Q248, Q258, Q267 |
| Prometheus | Q230, Q241, Q259 |
| Grafana | Q247, Q266 |
| ELK Stack | Q235, Q252 |
| Linux / Bash | Q228, Q237, Q244, Q254, Q263 |
| ArgoCD | Q243, Q262 |
| Jenkins | Q234, Q253 |
| GitHub Actions | Q239, Q257 |
| Git | Q232, Q268 |
| Python | Q249 |
| System Design | Q270 |

---

## New Concepts Introduced in This Set

### Kubernetes
- `ownerReferences` and garbage collection — foreground vs background deletion
- `MemoryPressure` eviction — `evictionHard` vs `evictionSoft`
- StatefulSet stuck termination — finalizers
- GPU workloads on Kubernetes — Job configuration
- `D` state (uninterruptible sleep) — cannot be killed
- CSI drivers — volume lifecycle from PVC to container
- PodSecurityAdmission — replacing deprecated PSP
- Topology spread constraints (new angle — what happens when unsatisfiable)
- API Priority and Fairness (APF) — FlowSchema, PriorityLevelConfiguration
- `Finalizers` — what they are, stuck finalizer removal

### AWS
- AWS Config auto-remediation with SSM Automation
- EventBridge — event-driven architecture design
- API Gateway throttling levels — account, stage, method, usage plan
- Fargate awsvpc networking in depth
- Direct Connect vs VPN — bandwidth, latency, Direct Connect Gateway
- AWS Step Functions — Standard vs Express workflows
- SCP examples — region restriction, tag enforcement, CloudTrail protection
- Automated SaaS tenant provisioning
- Lambda timeout in VPC — cold start vs connectivity issue

### Terraform
- `check` blocks and `precondition`/`postcondition` (Terraform 1.2+)
- Terraform testing — Terratest, Infracost
- `dynamic` blocks — Security Group example
- Perpetual drift — root causes
- Destroy failures — BucketNotEmpty, dependency errors
- Terraform K8s provider vs Helm provider vs ArgoCD

### Prometheus
- `relabel_configs` vs `metric_relabel_configs` — execution timing
- Alertmanager not receiving alerts — pipeline diagnosis
- Multi-cluster monitoring — Thanos vs federation vs remote_write
- Scrape timeout diagnosis

### Linux
- SELinux vs AppArmor — MAC security
- `epoll` vs `select` vs `poll` — C10K problem
- cgroups v1 vs v2 — CPU throttling in Kubernetes (new angle)
- Process in D state — cannot kill, recovery options

### Git
- Git submodules — update workflow and pitfalls
- `git reflog` — 3 recovery scenarios
- `git worktree` — parallel branch work
- `git sparse-checkout` — CI/CD monorepo optimization

### ArgoCD
- Image Updater — semver tracking, write-back to Git
- `syncWaves` (new example — database migration ordering)
- Helm in ArgoCD vs `helm install` — what ArgoCD does NOT support

### Grafana
- Time zone issues and template variables (`$__timeFilter`, `$__interval`, `$__range`)
- Annotations — CI/CD integration via API
- Unified alerting notification policy routing tree

---

*Part 6 of DevOps/SRE Interview Questions — nawab312 GitHub repositories*
*Total question bank: Q1–Q270*
