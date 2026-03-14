# DevOps / SRE / Cloud Interview Questions (181–225)

> Part 5 of interview preparation questions based on your GitHub repositories.
> All questions are unique — not repeated from Q1–Q180.

---

## Table of Contents

| # | Topic | Type | Difficulty |
|---|---|---|---|
| 181 | Kubernetes | Conceptual | Advanced |
| 182 | AWS | Scenario-Based | Advanced |
| 183 | Linux / Bash | Scenario-Based | Advanced |
| 184 | Terraform | Conceptual | Advanced |
| 185 | Prometheus | Conceptual | Advanced |
| 186 | Kubernetes | Troubleshooting | Advanced |
| 187 | Git | Conceptual | Advanced |
| 188 | AWS | Conceptual | Advanced |
| 189 | Jenkins | Conceptual | Advanced |
| 190 | ELK Stack | Troubleshooting | Advanced |
| 191 | Kubernetes | Scenario-Based | Advanced |
| 192 | Linux / Bash | Conceptual | Advanced |
| 193 | Terraform | Troubleshooting | Advanced |
| 194 | GitHub Actions | Troubleshooting | Advanced |
| 195 | AWS | Troubleshooting | Advanced |
| 196 | Prometheus | Scenario-Based | Advanced |
| 197 | Kubernetes | Conceptual | Advanced |
| 198 | ArgoCD | Conceptual | Advanced |
| 199 | Linux / Bash | Troubleshooting | Advanced |
| 200 | AWS | Scenario-Based | Advanced |
| 201 | Kubernetes | Troubleshooting | Advanced |
| 202 | Grafana | Conceptual | Advanced |
| 203 | Terraform | Scenario-Based | Advanced |
| 204 | Python | Scenario-Based | Advanced |
| 205 | AWS | Conceptual | Advanced |
| 206 | Kubernetes | Scenario-Based | Advanced |
| 207 | ELK Stack | Scenario-Based | Advanced |
| 208 | Jenkins | Scenario-Based | Advanced |
| 209 | Linux / Bash | Scenario-Based | Advanced |
| 210 | AWS | Scenario-Based | Advanced |
| 211 | Kubernetes | Conceptual | Advanced |
| 212 | GitHub Actions | Scenario-Based | Advanced |
| 213 | Terraform | Conceptual | Advanced |
| 214 | Prometheus | Troubleshooting | Advanced |
| 215 | AWS | Conceptual | Advanced |
| 216 | Kubernetes | Scenario-Based | Advanced |
| 217 | ArgoCD | Troubleshooting | Advanced |
| 218 | Linux / Bash | Conceptual | Advanced |
| 219 | AWS | Scenario-Based | Advanced |
| 220 | Kubernetes | Conceptual | Advanced |
| 221 | Grafana | Scenario-Based | Advanced |
| 222 | Terraform | Scenario-Based | Advanced |
| 223 | Git | Scenario-Based | Advanced |
| 224 | AWS + Kubernetes | Scenario-Based | Advanced |
| 225 | All Topics | System Design | Advanced |

---

## Questions

---

### Q181 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `leaderElection`** mechanism. Why is leader election needed in Kubernetes controllers?
>
> Which Kubernetes control plane components use leader election? What happens if leader election fails? How would you debug a situation where multiple controller-manager Pods think they are the leader?

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md` — control plane HA, leader election sections

---

### Q182 — AWS | Scenario-Based | Advanced

> Your team has just enabled **AWS GuardDuty** and within the first hour you receive a critical finding:
>
> `UnauthorizedAccess:IAMUser/MaliciousIPCaller — API call from known malicious IP`
>
> Walk me through your **complete incident response process** — from the GuardDuty finding to containment, investigation, and remediation.

📁 **Reference:** `nawab312/AWS` — GuardDuty, IAM, CloudTrail, incident response sections

---

### Q183 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that implements a **TCP port scanner**:
> - Takes a hostname and port range as arguments (e.g., `./scan.sh 192.168.1.1 1-1024`)
> - Scans ports **in parallel** (max 50 concurrent)
> - Shows **open ports** with service name if identifiable
> - Shows **scan progress** as a percentage
> - Outputs results to both **stdout** and a timestamped log file
> - Has a **configurable timeout** per port (default 1 second)

📁 **Reference:** `nawab312/DSA` → `Linux` — networking, bash scripting, parallel execution, /dev/tcp sections

---

### Q184 — Terraform | Conceptual | Advanced

> Explain **Terraform `provider` configuration** in depth. What is the difference between **provider aliasing** and **multiple provider configurations**?
>
> Write a Terraform configuration that deploys resources in **3 different AWS regions simultaneously** using provider aliases, and explain how modules consume aliased providers.

📁 **Reference:** `nawab312/Terraform` — provider configuration, aliasing, multi-region sections

---

### Q185 — Prometheus | Conceptual | Advanced

> Explain **PromQL functions** in depth. What is the difference between:
> - `rate()` vs `irate()`
> - `increase()` vs `delta()`
> - `avg_over_time()` vs `avg()`
>
> When would `irate()` give misleading results? Give a concrete example showing when each function is the correct choice.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — PromQL functions sections

---

### Q186 — Kubernetes | Troubleshooting | Advanced

> Your Kubernetes cluster shows **`ImagePullBackOff`** on ALL pods across ALL nodes suddenly — not just one service. This was working fine 10 minutes ago.
>
> What are the cluster-wide causes of ImagePullBackOff (as opposed to per-pod causes) and how would you diagnose and fix this quickly?

📁 **Reference:** `nawab312/Kubernetes` → `14_TROUBLESHOOTING.md` — ImagePullBackOff, registry connectivity, credentials sections

---

### Q187 — Git | Conceptual | Advanced

> Explain **Git `reflog`** — what it tracks, how long it keeps data, and how it saves you from disaster.
>
> Give **3 real-world scenarios** where `git reflog` would be the only way to recover lost work. Write the exact commands you would use in each scenario to recover.

📁 **Reference:** `nawab312/CI_CD` → `Git` — reflog, recovery, advanced Git sections

---

### Q188 — AWS | Conceptual | Advanced

> Explain **AWS Step Functions**. What problem does it solve that Lambda alone cannot?
>
> What is the difference between **Standard** and **Express** workflows? Give a real-world DevOps use case — design a Step Functions workflow for a **multi-stage deployment pipeline** with approval gates, rollback on failure, and notification steps.

📁 **Reference:** `nawab312/AWS` — Step Functions, workflow orchestration sections

---

### Q189 — Jenkins | Conceptual | Advanced

> Explain **Jenkins Pipeline `when` directive** and **`input` step** in detail.
>
> Write a Jenkinsfile that:
> - Runs unit tests on every branch
> - Runs integration tests **only on `main` and `release/*` branches**
> - Requires **manual approval with a timeout of 30 minutes** before deploying to production
> - If approval times out, automatically **aborts the deployment** and sends a Slack notification

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — when directive, input step, pipeline conditions sections

---

### Q190 — ELK Stack | Troubleshooting | Advanced

> Your **Kibana** dashboard is showing `No results found` for a time range where you know logs exist in Elasticsearch. Direct Elasticsearch queries confirm the data is there.
>
> What are all the possible reasons Kibana would show no results despite data existing, and how do you fix each one?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Kibana index patterns, time fields, mapping sections

---

### Q191 — Kubernetes | Scenario-Based | Advanced

> You are implementing **Kubernetes cluster upgrades** in production. The cluster has:
> - 50 worker nodes
> - 500 running pods
> - Zero-downtime requirement
> - Mixed workloads (stateless and stateful)
>
> Walk me through your **complete upgrade strategy** from Kubernetes 1.28 to 1.30 — including control plane, worker nodes, add-ons, and validation at each step.

📁 **Reference:** `nawab312/Kubernetes` → `09_CLUSTER_OPERATIONS.md` — cluster upgrade, rolling upgrade sections

---

### Q192 — Linux / Bash | Conceptual | Advanced

> Explain **Linux memory management** concepts:
> - What is the difference between **virtual memory** and **physical memory**?
> - What are **huge pages** and when would you use them?
> - What is **swap** and what does high swap usage indicate?
> - Explain **memory overcommit** — what are the 3 overcommit modes in Linux?
> - What is **page cache** and how does it affect `free -h` output?

📁 **Reference:** `nawab312/DSA` → `Linux` — memory management, `/proc/meminfo`, vm settings sections

---

### Q193 — Terraform | Troubleshooting | Advanced

> A colleague ran `terraform apply` and it succeeded, but now `terraform plan` shows the same resources as **changed again** even though nothing in the `.tf` files changed.
>
> What causes **perpetual drift** in Terraform where plan always shows changes even after apply? List at least 5 root causes and the fix for each.

📁 **Reference:** `nawab312/Terraform` — state drift, `ignore_changes`, perpetual diff sections

---

### Q194 — GitHub Actions | Troubleshooting | Advanced

> Your GitHub Actions workflow is **consuming GitHub Actions minutes extremely fast**. A workflow that should take 5 minutes is taking 45 minutes and running on every commit.
>
> Walk me through how you would **audit and optimize** the workflow — covering: unnecessary triggers, missing caching, redundant steps, parallelization opportunities, and job concurrency settings.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — workflow optimization, caching, concurrency sections

---

### Q195 — AWS | Troubleshooting | Advanced

> Your **AWS ALB** is returning `504 Gateway Timeout` errors for about 5% of requests. The backend EC2 instances appear healthy in the target group.
>
> Walk me through the **complete troubleshooting process** for ALB 504 errors — what metrics to check, what logs to look at, and what are all the possible root causes?

📁 **Reference:** `nawab312/AWS` — ALB, access logs, target group health, idle timeout sections

---

### Q196 — Prometheus | Scenario-Based | Advanced

> You need to monitor **PostgreSQL** running on Kubernetes using Prometheus. Set up monitoring that covers:
> - Active connections vs max connections
> - Query execution time (slow queries)
> - Replication lag (for replicas)
> - Transaction rate and deadlocks
> - Database size growth
>
> Write the complete setup including exporter deployment, ServiceMonitor, recording rules, and alerting rules.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — postgres_exporter, database monitoring sections

---

### Q197 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `topology spread constraints`**. What problem do they solve that `podAntiAffinity` cannot solve efficiently at scale?
>
> Write a topology spread constraint that ensures pods are spread evenly across:
> - Availability zones (max skew of 1)
> - Nodes within each AZ (max skew of 2)
>
> What happens when the constraint cannot be satisfied?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — topology spread constraints sections

---

### Q198 — ArgoCD | Conceptual | Advanced

> Explain **ArgoCD `syncWaves`** and **`syncPhases`**. What problem do they solve?
>
> Give a real-world example where you have these resources that must be applied in a specific order:
> - Namespace and RBAC (must be first)
> - ConfigMaps and Secrets (must be second)
> - Database migration Job (must complete before app starts)
> - Application Deployment (must be last)
>
> Write the ArgoCD annotations to enforce this order.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — sync waves, sync phases, resource ordering sections

---

### Q199 — Linux / Bash | Troubleshooting | Advanced

> A production server is experiencing **network packet loss**. Users are reporting intermittent connection drops. `ping` to the server shows 15% packet loss.
>
> Walk me through the **complete network troubleshooting process** — which commands you would use, what you are looking for at each layer (physical, network, transport), and what the most common causes of packet loss are on Linux.

📁 **Reference:** `nawab312/DSA` → `Linux` — network troubleshooting, tc, ethtool, netstat sections

---

### Q200 — AWS | Scenario-Based | Advanced

> Your company is implementing **AWS Cost Anomaly Detection**. You receive an alert:
>
> `Cost anomaly detected: $15,000 unexpected spend in the last 24 hours on EC2`
>
> Walk me through how you would **investigate the cost spike**, identify the root cause, and implement controls to **prevent runaway costs** in future.

📁 **Reference:** `nawab312/AWS` — Cost Explorer, Cost Anomaly Detection, billing alerts, SCPs sections

---

### Q201 — Kubernetes | Troubleshooting | Advanced

> Your **Kubernetes DNS resolution is broken** — pods cannot resolve service names. `nslookup kubernetes.default` from a pod fails, but pods can communicate via IP addresses.
>
> Walk me through diagnosing and fixing Kubernetes DNS issues — covering CoreDNS, ConfigMap, network connectivity, and common failure modes.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — CoreDNS, DNS troubleshooting sections

---

### Q202 — Grafana | Conceptual | Advanced

> Explain **Grafana alerting architecture** in Grafana 9+ (unified alerting) vs the old **legacy alerting**.
>
> What is a **contact point**, **notification policy**, and **silence** in unified alerting? How does the **alert routing tree** work? Write a notification policy that routes:
> - P1 alerts → PagerDuty + Slack #incidents
> - P2 alerts → Slack #alerts only
> - All resolved alerts → Slack #resolved

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — unified alerting, notification policies sections

---

### Q203 — Terraform | Scenario-Based | Advanced

> Your Terraform configuration manages **AWS IAM policies** but the policy documents are becoming very complex — 500+ line JSON documents that are hard to read and maintain.
>
> How would you refactor the IAM policy management to be more maintainable? Show how to use **`aws_iam_policy_document` data source**, **heredoc**, and **templatefile()** function to manage complex IAM policies cleanly.

📁 **Reference:** `nawab312/Terraform` — IAM management, templatefile, policy documents sections

---

### Q204 — Python | Scenario-Based | Advanced

> Write a **Python script** that implements a **Terraform drift detector**:
> - Runs `terraform plan -json` and parses the JSON output
> - Identifies resources that have **drifted** (changed outside Terraform)
> - Groups drift by **resource type** and **change type** (update/delete/create)
> - Outputs a **summary report** with counts and details
> - Sends the report to a **Slack webhook** if drift is detected
> - Exits with code `1` if drift found (useful in CI/CD)

📁 **Reference:** `nawab312/DSA` → `Python` — JSON parsing, subprocess, Slack webhooks sections

---

### Q205 — AWS | Conceptual | Advanced

> Explain **AWS Secrets Manager** vs **AWS Parameter Store (SSM)**.
>
> What are the key differences in terms of **cost**, **rotation support**, **size limits**, **cross-account access**, and **integration with services**? When would you choose one over the other? How do you **rotate database credentials** automatically using Secrets Manager?

📁 **Reference:** `nawab312/AWS` — Secrets Manager, SSM Parameter Store, credential rotation sections

---

### Q206 — Kubernetes | Scenario-Based | Advanced

> You need to implement **Kubernetes node auto-provisioning** (Karpenter) to replace the Cluster Autoscaler. Your cluster has:
> - Workloads with varying CPU/memory requirements (1 CPU to 96 CPU jobs)
> - Need to use **Spot instances** for cost savings with fallback to On-Demand
> - Different instance families preferred for different workloads (compute-optimized, memory-optimized)
>
> Walk me through setting up Karpenter with these requirements.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — Cluster Autoscaler, node provisioning sections

---

### Q207 — ELK Stack | Scenario-Based | Advanced

> You need to implement **log-based alerting** using the ELK stack. When more than **100 ERROR logs appear within 5 minutes** from the payment service, an alert should:
> - Send a **Slack notification** with a sample of the error messages
> - Create a **PagerDuty incident** if it persists for more than 10 minutes
> - **Auto-resolve** when error rate drops below 10/minute
>
> Walk me through implementing this using **Elasticsearch Watcher** or **Kibana Alerting**.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Kibana alerting, Watcher, connectors sections

---

### Q208 — Jenkins | Scenario-Based | Advanced

> You need to implement a **Jenkins pipeline** that performs **database schema migrations** safely:
> - Runs **Flyway** migrations before deploying the new application version
> - If migration **fails** — abort deployment, do NOT deploy new code
> - If migration **succeeds** — deploy new application version
> - If application deployment **fails** after migration — the migration is NOT rolled back (forward-only)
> - Sends detailed **notification** of migration results (which scripts ran, duration)
>
> Write the complete Jenkinsfile.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — database migration pipeline, sequential stages, notifications sections

---

### Q209 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that implements a **simple service discovery** mechanism:
> - Reads a `services.conf` file with format: `service_name:host:port:health_check_path`
> - Checks health of each service by hitting the health check endpoint
> - Maintains a **`healthy_services.txt`** and **`unhealthy_services.txt`** file
> - Runs every 30 seconds in a loop
> - When a service transitions from healthy→unhealthy or vice versa, **logs the transition** with timestamp
> - Exposes a simple **HTTP response** via netcat showing current service status

📁 **Reference:** `nawab312/DSA` → `Linux` — bash scripting, curl, netcat, service health sections

---

### Q210 — AWS | Scenario-Based | Advanced

> Your company's **AWS bill has a $30,000 charge for data transfer** this month — 5x higher than usual. You need to identify the source.
>
> Walk me through how you would use **AWS Cost Explorer**, **VPC Flow Logs**, **CloudWatch**, and other tools to identify exactly which resources are generating the unexpected data transfer costs and how to reduce them.

📁 **Reference:** `nawab312/AWS` — Cost Explorer, VPC Flow Logs, data transfer costs, VPC endpoints sections

---

### Q211 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `projected volumes`**. What types of data can be combined into a projected volume?
>
> Give a real-world example of a Pod that needs:
> - A **ServiceAccount token** with a specific audience and expiry
> - A **ConfigMap** value
> - A **Secret** value
>
> All mounted at different paths within the same volume mount. Write the Pod spec.

📁 **Reference:** `nawab312/Kubernetes` → `04_STORAGE.md` and `05_CONFIGURATION_SECRETS.md` — projected volumes, token projection sections

---

### Q212 — GitHub Actions | Scenario-Based | Advanced

> Your team wants to implement **dependency review** and **security scanning** in GitHub Actions:
> - Block PRs that introduce **critical CVE vulnerabilities** in dependencies
> - Run **SAST (Static Application Security Testing)** on every PR
> - Scan Docker images for vulnerabilities before pushing to ECR
> - Generate a **SBOM (Software Bill of Materials)** on every release
> - Store scan results in **GitHub Security tab**
>
> Write the complete workflow implementing all security checks.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — security scanning, dependency review, SARIF sections

---

### Q213 — Terraform | Conceptual | Advanced

> Explain **Terraform `locals`** vs **`variables`** vs **`outputs`**. When would you use each?
>
> What is the difference between **input variables** with `default` values and **local values**? Write a real-world example where using `locals` to compute derived values makes the configuration significantly cleaner than repeating expressions.

📁 **Reference:** `nawab312/Terraform` — locals, variables, outputs, expressions sections

---

### Q214 — Prometheus | Troubleshooting | Advanced

> Your Prometheus is showing `context deadline exceeded` errors when scraping some targets, and those targets show as `DOWN` in the targets page. The services are actually running and healthy.
>
> What causes scrape timeouts in Prometheus? Walk me through diagnosing whether the issue is in Prometheus configuration, network, or the target itself, and how to fix it.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — scrape configuration, timeouts, target health sections

---

### Q215 — AWS | Conceptual | Advanced

> Explain **AWS Spot Instances** in depth. What is the **Spot interruption** process — how much notice do you get and what happens to the instance?
>
> How would you design an application on Kubernetes (EKS) to **handle Spot interruptions gracefully**? What is the role of **interruption handlers** like `aws-node-termination-handler`?

📁 **Reference:** `nawab312/AWS` and `nawab312/Kubernetes` — Spot Instances, interruption handling, cost optimization sections

---

### Q216 — Kubernetes | Scenario-Based | Advanced

> You need to implement **Kubernetes `NetworkPolicy`** for a 3-tier application:
> - **Frontend** (namespace: `web`) — only accessible from internet via Ingress
> - **Backend API** (namespace: `api`) — only accessible from frontend
> - **Database** (namespace: `db`) — only accessible from backend API
> - **Monitoring** (namespace: `monitoring`) — needs to scrape metrics from all tiers
>
> Write all required NetworkPolicies for complete isolation with the minimum required access.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` and `07_SECURITY.md` — NetworkPolicy, multi-tier isolation sections

---

### Q217 — ArgoCD | Troubleshooting | Advanced

> ArgoCD is syncing an application but the sync keeps **failing with `ComparisonError`**. The error message says:
>
> `rpc error: code = Unknown desc = Manifest generation error (cached): failed to generate manifest`
>
> What are all the possible causes of manifest generation errors in ArgoCD, and how would you diagnose and fix each one?

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — manifest generation, Helm, Kustomize errors sections

---

### Q218 — Linux / Bash | Conceptual | Advanced

> Explain **Linux `cgroups v1` vs `cgroups v2`**. What changed in v2 and why does it matter for Kubernetes?
>
> What **subsystems** does cgroups control? How does Kubernetes use cgroups to enforce **CPU and memory limits** on containers? What is the difference between CPU **`limits`** using CFS (Completely Fair Scheduler) throttling vs CPU **`requests`** using shares?

📁 **Reference:** `nawab312/DSA` → `Linux` and `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md` — cgroups, CPU throttling sections

---

### Q219 — AWS | Scenario-Based | Advanced

> You need to implement **blue/green deployment** on AWS ECS using **CodeDeploy**. The deployment must:
> - Shift traffic **gradually**: 10% → 30% → 100% over 30 minutes
> - Automatically **rollback** if CloudWatch alarms trigger
> - Have a **bake time** of 10 minutes at each traffic shift
> - Send **SNS notifications** at each deployment lifecycle event
>
> Walk me through the complete setup.

📁 **Reference:** `nawab312/AWS` — ECS, CodeDeploy, blue/green, traffic shifting sections

---

### Q220 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `CRD` (Custom Resource Definition)** versioning. What is **conversion webhooks** and why are they needed?
>
> If you have a CRD with `v1alpha1` in production and want to introduce `v1beta1` with schema changes, how do you handle the migration without breaking existing resources? What is the **`served`** and **`storage`** version concept?

📁 **Reference:** `nawab312/Kubernetes` → `10_CRDs_EXTENSIBILITY.md` — CRD versioning, conversion webhooks sections

---

### Q221 — Grafana | Scenario-Based | Advanced

> You are asked to implement **Grafana as Code** — all dashboards and datasources should be version controlled in Git and automatically provisioned.
>
> Walk me through:
> 1. Setting up **Grafana provisioning** via config files
> 2. Managing dashboards as **JSON in Git**
> 3. Using **Grafonnet** (Jsonnet library) to generate dashboards programmatically
> 4. Automating dashboard deployment via **CI/CD pipeline**

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — provisioning, Grafana as code sections

---

### Q222 — Terraform | Scenario-Based | Advanced

> You are building a **Terraform module** for deploying a containerized microservice on ECS Fargate that needs to be **production-ready**. The module must support:
> - Auto Scaling based on CPU and custom metrics
> - Service discovery via **AWS Cloud Map**
> - **Secret injection** from Secrets Manager at runtime
> - **Structured logging** to CloudWatch with log retention
> - **ALB integration** with health checks
>
> Design the module interface (variables and outputs) and key resource blocks.

📁 **Reference:** `nawab312/Terraform` and `nawab312/AWS` — ECS module design, Cloud Map, Secrets Manager sections

---

### Q223 — Git | Scenario-Based | Advanced

> Your team is adopting **trunk-based development** moving away from GitFlow. You have 20 developers who are used to long-lived feature branches.
>
> What changes to **Git workflow**, **CI/CD pipeline**, and **deployment strategy** are needed to make trunk-based development work safely? What is **feature flags** role in this transition and how do you implement them?

📁 **Reference:** `nawab312/CI_CD` → `Git` — trunk-based development, feature flags, CI/CD integration sections

---

### Q224 — AWS + Kubernetes | Scenario-Based | Advanced

> Your EKS cluster is running in a **private VPC** with no public internet access. You need to:
> - Pull container images from **public ECR/Docker Hub**
> - Allow pods to call **external APIs** (payment gateway, email service)
> - Access **AWS services** (S3, Secrets Manager) without going through internet
> - Developers need to run **`kubectl` commands** from their laptops
>
> Design the complete **network architecture** to satisfy all requirements.

📁 **Reference:** `nawab312/AWS` and `nawab312/Kubernetes` — private EKS, VPC endpoints, NAT Gateway, bastion sections

---

### Q225 — All Topics | System Design | Advanced

> You are designing a **GitOps-based platform** for a large enterprise with:
> - **200 microservices** across 5 teams
> - **4 environments**: dev, staging, UAT, production
> - **Compliance requirement**: every production change must have an audit trail and require 2 approvals
> - **Security requirement**: no human has direct `kubectl` access to production
> - **Reliability requirement**: platform itself must be self-healing
>
> Design the complete GitOps platform covering: Git repository structure, ArgoCD setup, CI pipeline, secret management, access control, audit logging, and disaster recovery of the platform itself.

📁 **Reference:** All repositories — comprehensive enterprise GitOps platform design

---

## Key Topics Coverage (Q181–Q225)

| Topic | Questions |
|---|---|
| Kubernetes | Q181, Q186, Q191, Q197, Q201, Q206, Q211, Q216, Q220 |
| AWS | Q182, Q188, Q195, Q200, Q205, Q210, Q215, Q219, Q224 |
| Terraform | Q184, Q193, Q203, Q213, Q222 |
| Prometheus | Q185, Q196, Q214 |
| Grafana | Q202, Q221 |
| ELK Stack | Q190, Q207 |
| Linux / Bash | Q183, Q192, Q199, Q209, Q218 |
| ArgoCD | Q198, Q217 |
| Jenkins | Q189, Q208 |
| GitHub Actions | Q194, Q212 |
| Git | Q187, Q223 |
| Python | Q204 |
| System Design | Q225 |

---

## New Concepts Introduced in This Set

### Kubernetes
- Leader election in control plane components
- Topology spread constraints vs podAntiAffinity
- Projected volumes (ServiceAccount token + ConfigMap + Secret)
- CRD versioning + conversion webhooks
- Karpenter vs Cluster Autoscaler
- Cluster upgrade strategy (50 nodes, zero downtime)
- CoreDNS troubleshooting

### AWS
- GuardDuty incident response
- AWS Step Functions workflow design
- Cost anomaly detection + investigation
- ALB 504 troubleshooting
- Blue/green on ECS with CodeDeploy + traffic shifting
- Spot instance interruption handling with aws-node-termination-handler
- Secrets Manager vs SSM Parameter Store — detailed comparison
- Data transfer cost investigation
- AWS Organizations SCPs

### Terraform
- Provider aliasing for multi-region
- Perpetual drift — 5 root causes
- `locals` vs `variables` vs `outputs` — detailed comparison
- IAM policy management with `aws_iam_policy_document`
- Production-ready ECS Fargate module design

### Prometheus
- `rate()` vs `irate()` vs `increase()` vs `delta()` — when to use each
- Scrape timeout diagnosis
- PostgreSQL monitoring setup

### Linux
- systemd unit files with resource limits
- Memory management — huge pages, overcommit modes, page cache
- cgroups v1 vs v2 — Kubernetes CPU throttling
- Network packet loss troubleshooting
- TCP port scanning with /dev/tcp

### Git
- `git reflog` — 3 recovery scenarios
- Trunk-based development transition
- Feature flags role in CI/CD

### ArgoCD
- `syncWaves` and `syncPhases` — resource ordering
- Manifest generation error diagnosis

### Grafana
- Unified alerting — contact points, notification policies, routing tree
- Grafana as Code — provisioning + Grafonnet

---

*Part 5 of DevOps/SRE Interview Questions — nawab312 GitHub repositories*
*Total question bank: Q1–Q225*
