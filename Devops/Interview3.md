# DevOps / SRE / Cloud Interview Questions (91–135)

> Part 3 of interview preparation questions based on your GitHub repositories.
> All questions are unique and not repeated from previous sets (Q1–Q90).

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

## Questions

---

### Q91 — Kubernetes | Conceptual | Advanced

> Explain the **Kubernetes Admission Control** mechanism. What is the difference between a **Validating Admission Webhook** and a **Mutating Admission Webhook**?
>
> Give a real-world example of each and explain the **order in which they are called** during a resource creation request.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md` and `10_CRDs_EXTENSIBILITY.md` — Admission Controllers, Webhooks sections

---

### Q92 — AWS | Scenario-Based | Advanced

> Your company runs a **data pipeline** that processes files uploaded to S3. The pipeline uses:
> - S3 → triggers Lambda → processes file → writes to RDS → sends notification via SNS
>
> The pipeline is **failing silently** — files are uploaded but sometimes no notification is sent and no record appears in RDS. There are no errors in CloudWatch Logs.
>
> **How would you debug this end-to-end?**

📁 **Reference:** `nawab312/AWS` — Lambda, S3 events, SNS, RDS, CloudWatch, X-Ray sections

---

### Q93 — Linux / Bash | Conceptual | Advanced

> Explain the Linux **`iptables`** and **`nftables`** frameworks. How does packet filtering work in Linux?
>
> Walk me through the **5 built-in chains** (PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING) and explain what happens to a packet as it flows through them for an **incoming HTTP request** to a web server.

📁 **Reference:** `nawab312/DSA` → `Linux` — networking, iptables, packet filtering sections

---

### Q94 — Terraform | Scenario-Based | Advanced

> You need to manage **Terraform at scale** across **50 microservices**, each with their own infrastructure. Teams are spending too much time managing Terraform boilerplate and inconsistent module versions.
>
> How would you design a **Terraform platform** to standardize infrastructure across all teams while still giving them flexibility? What tools would you use?

📁 **Reference:** `nawab312/Terraform` — modules, Terraform Cloud/Enterprise, module registry sections

---

### Q95 — Prometheus | Conceptual | Advanced

> Explain **Prometheus federation** and **Thanos**. What problems do they solve?
>
> When would you use federation vs Thanos? Explain **Thanos components** — Sidecar, Store Gateway, Querier, Compactor — and what each one does.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — federation, long-term storage, Thanos sections

---

### Q96 — Kubernetes | Troubleshooting | Advanced

> Your **Kubernetes API server** is responding very slowly — `kubectl` commands take 30+ seconds to complete. The cluster has 500 nodes and 10,000 pods.
>
> What are the possible causes of API server slowness at scale, and how would you investigate and fix it?

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md` and `09_CLUSTER_OPERATIONS.md` — API server, etcd performance sections

---

### Q97 — Git | Conceptual | Advanced

> Explain **`git cherry-pick`**, **`git bisect`**, and **`git stash`**.
>
> Give a real-world DevOps scenario where you would use each one. When would `git bisect` be particularly useful in a CI/CD context?

📁 **Reference:** `nawab312/CI_CD` → `Git` — advanced Git commands sections

---

### Q98 — AWS | Conceptual | Advanced

> Explain the difference between **AWS EKS** (Elastic Kubernetes Service) and **AWS ECS** (Elastic Container Service) with **Fargate**.
>
> What are the **networking differences** between EKS with VPC CNI and ECS with awsvpc network mode? When would you choose EKS over ECS Fargate for a production workload?

📁 **Reference:** `nawab312/AWS` — EKS, ECS, Fargate, VPC CNI sections

---

### Q99 — Jenkins | Scenario-Based | Advanced

> You need to implement a **Jenkins pipeline** that:
> - Runs on **10 different microservices** in the same monorepo
> - Only triggers the pipeline for the **service that changed** (not all 10)
> - Uses **parallel execution** to run tests for changed services simultaneously
> - Has a **single deployment gate** — all services must pass before any deploys
>
> How would you design and implement this?

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — monorepo pipelines, parallel stages, conditional execution sections

---

### Q100 — ELK Stack | Conceptual | Advanced

> Explain **Elasticsearch Index Lifecycle Management (ILM)**. What are the **4 phases** of ILM and what happens in each?
>
> Give a real-world example of an ILM policy for application logs that:
> - Keeps hot data for 7 days
> - Moves to warm tier after 7 days
> - Deletes after 30 days

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — ILM, index management sections

---

### Q101 — Kubernetes | Scenario-Based | Advanced

> Your microservice is deployed on Kubernetes and needs to **gracefully handle shutdown**. Currently when a Pod is terminated during a rolling update, in-flight requests are being dropped causing **502 errors** for users.
>
> Explain the **Pod termination lifecycle** and walk me through the complete configuration needed to achieve **zero-downtime Pod shutdown**.

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md` — Pod lifecycle, terminationGracePeriodSeconds, preStop hooks sections

---

### Q102 — Linux / Bash | Troubleshooting | Advanced

> A production server suddenly has **extremely high load average** (load: 45.0 on an 8-core machine) but **CPU usage is only 15%**. The server is barely processing requests.
>
> What does this tell you? How would you identify the root cause, and what are the most likely culprits?

📁 **Reference:** `nawab312/DSA` → `Linux` — load average, I/O wait, process states sections

---

### Q103 — Terraform | Conceptual | Advanced

> Explain **Terraform `for_each`** vs **`count`** for creating multiple resources.
>
> What are the limitations of `count`? When does using `count` cause **resource replacement** that `for_each` avoids? Give a concrete example showing why `for_each` is preferred for most use cases.

📁 **Reference:** `nawab312/Terraform` — meta-arguments, for_each, count sections

---

### Q104 — GitHub Actions | Scenario-Based | Advanced

> You need to build a **matrix build** in GitHub Actions that:
> - Tests your application against **3 Node.js versions** (18, 20, 22) and **3 OS types** (ubuntu, windows, macos)
> - **Excludes** the combination of Node 18 + macos
> - **Fails fast** if any combination fails
> - Uploads **test artifacts** for failed combinations only
>
> Write the complete workflow.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — matrix builds, artifacts sections

---

### Q105 — AWS | Troubleshooting | Advanced

> Your **AWS RDS PostgreSQL** instance is experiencing **connection exhaustion** — the error `FATAL: remaining connection slots are reserved for non-replication superuser connections` is appearing in application logs.
>
> What causes this, how do you immediately fix it, and what is the long-term architectural solution?

📁 **Reference:** `nawab312/AWS` — RDS, connection pooling, RDS Proxy sections

---

### Q106 — Prometheus | Scenario-Based | Advanced

> You need to set up **Prometheus monitoring for a Kafka cluster** running on Kubernetes. The monitoring needs to cover:
> - Broker metrics (under-replicated partitions, ISR count)
> - Consumer group lag
> - Producer/consumer throughput
>
> Walk me through the complete setup including exporters, ServiceMonitors, and alerting rules.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — exporters, ServiceMonitors, Kafka monitoring sections

---

### Q107 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `affinity`** rules in detail. What is the difference between:
> - `nodeAffinity` vs `podAffinity` vs `podAntiAffinity`
> - `requiredDuringSchedulingIgnoredDuringExecution` vs `preferredDuringSchedulingIgnoredDuringExecution`
>
> Give a real-world example where you would use `podAntiAffinity` and explain why.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — affinity, anti-affinity sections

---

### Q108 — ArgoCD | Troubleshooting | Advanced

> Your ArgoCD is showing all applications as **`Unknown`** health status and the ArgoCD UI is very slow. The ArgoCD application controller Pod is consuming **very high CPU and memory**.
>
> What are the possible causes and how would you investigate and fix the ArgoCD performance issues?

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — performance tuning, application controller sections

---

### Q109 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that:
> - Parses an **Nginx access log** file
> - Calculates the **top 10 IPs** by request count
> - Calculates the **top 10 endpoints** by request count
> - Shows the **HTTP status code distribution** (200, 404, 500 etc.)
> - Shows the **average response time** per endpoint
> - Outputs everything in a clean, readable format

📁 **Reference:** `nawab312/DSA` → `Linux` — Bash scripting, text processing, awk, sed sections

---

### Q110 — AWS | Scenario-Based | Advanced

> Your company is implementing a **Zero Trust security model** on AWS. All internal services should:
> - **Never trust** any request by default, even from within the VPC
> - Authenticate every service-to-service call
> - Have **fine-grained authorization**
> - **Audit log** every API call
>
> What AWS services and patterns would you use to implement this?

📁 **Reference:** `nawab312/AWS` — IAM, mTLS, AWS App Mesh, CloudTrail, VPC sections

---

### Q111 — Kubernetes | Troubleshooting | Advanced

> You run `kubectl apply -f deployment.yaml` and the command hangs indefinitely — it never returns. No error is shown.
>
> What are the possible reasons `kubectl apply` would hang, and how would you diagnose and fix it?

📁 **Reference:** `nawab312/Kubernetes` → `14_TROUBLESHOOTING.md` — API server, webhooks, kubectl troubleshooting sections

---

### Q112 — Grafana | Conceptual | Advanced

> Explain **Grafana Loki** and how it differs from **Elasticsearch** for log aggregation.
>
> What is **LogQL** and how does it compare to Elasticsearch's KQL? Give examples of LogQL queries for:
> - Finding all ERROR logs from a specific service
> - Counting error rate over time
> - Extracting a field from a log line using a pattern

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — Loki, LogQL sections

---

### Q113 — Terraform | Troubleshooting | Advanced

> Your Terraform `apply` is **extremely slow** — it takes 45 minutes for a `plan` on a large infrastructure with 500+ resources. Teams are frustrated with the slow feedback loop.
>
> What are the causes of slow Terraform plans and how would you **optimize performance**?

📁 **Reference:** `nawab312/Terraform` — performance optimization, `-target`, parallelism sections

---

### Q114 — Python | Scenario-Based | Advanced

> Write a **Python script** that acts as a simple **Kubernetes health checker**:
> - Uses the **Kubernetes Python client** to connect to a cluster
> - Lists all Deployments across all namespaces
> - For each Deployment, checks if `ready replicas == desired replicas`
> - Outputs a **health report** showing healthy and unhealthy Deployments
> - Sends an alert to a **Slack webhook** for any unhealthy Deployments

📁 **Reference:** `nawab312/DSA` → `Python` — Kubernetes client, HTTP requests, JSON sections

---

### Q115 — AWS | Conceptual | Advanced

> Explain **AWS VPC Peering** vs **AWS Transit Gateway** vs **AWS PrivateLink**.
>
> When would you use each one? What are the **limitations of VPC Peering** that Transit Gateway solves? Give a real-world architecture example using Transit Gateway for a hub-and-spoke network model.

📁 **Reference:** `nawab312/AWS` — VPC Peering, Transit Gateway, PrivateLink sections

---

### Q116 — Kubernetes | Scenario-Based | Advanced

> You are implementing **canary deployments** on Kubernetes without a service mesh. You want to route **10% of traffic** to a new version (v2) and **90% to the current version** (v1).
>
> How would you implement this using only native Kubernetes resources? What are the limitations, and how would **Istio or Argo Rollouts** improve this?

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` and `11_ISTIO_SERVICE_MESH.md` — traffic management, canary deployments sections

---

### Q117 — ELK Stack | Troubleshooting | Advanced

> Your **Logstash pipeline** is falling behind — the pipeline input queue is growing and logs are arriving in Kibana with a **30-minute delay**. Logstash CPU is at 100%.
>
> What are the causes of Logstash performance bottlenecks and how would you tune it?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Logstash performance, pipeline tuning, persistent queues sections

---

### Q118 — Jenkins | Conceptual | Advanced

> Explain **Jenkins Blue Ocean** vs **Jenkins Classic UI**. What is a **Multibranch Pipeline** and how does it differ from a regular Pipeline job?
>
> Also explain **Jenkins Folder** and **Jenkins View** — how would you organize 100+ jobs in Jenkins for a large engineering organization?

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — Multibranch Pipeline, Jenkins organization sections

---

### Q119 — Linux / Bash | Conceptual | Advanced

> Explain **Linux namespaces** and **cgroups**. How do they form the foundation of **container technology**?
>
> List the **7 Linux namespaces** and explain what each one isolates. How does Docker use these internally when you run `docker run`?

📁 **Reference:** `nawab312/DSA` → `Linux` and `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md` — namespaces, cgroups sections

---

### Q120 — AWS | Scenario-Based | Advanced

> Your team is building a **multi-region active-active application** on AWS. The application serves users globally and must:
> - Route users to the **nearest region** automatically
> - Handle **region failover** within 60 seconds
> - Keep data **synchronized** across regions
> - Handle **split-brain** scenarios
>
> What AWS services and architecture patterns would you use?

📁 **Reference:** `nawab312/AWS` — Route53, Global Accelerator, DynamoDB Global Tables, RDS Global Database sections

---

### Q121 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `ConfigMap`** and **`Secret`** — what are the differences, limitations, and security considerations?
>
> What are the problems with storing sensitive data in Kubernetes Secrets? What are **better alternatives** for secrets management in production Kubernetes clusters?

📁 **Reference:** `nawab312/Kubernetes` → `05_CONFIGURATION_SECRETS.md` — ConfigMaps, Secrets, external secrets management sections

---

### Q122 — GitHub Actions | Conceptual | Advanced

> Explain the difference between **GitHub Actions `secrets`**, **`vars`** (variables), and **`environments`**.
>
> How do **environment protection rules** work? What is the difference between **repository secrets** and **environment secrets**? Write a workflow that uses different secrets for staging vs production deployments.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — secrets, variables, environments sections

---

### Q123 — Terraform | Scenario-Based | Advanced

> You need to provision **ephemeral environments** for every Pull Request — each PR gets its own isolated AWS environment (VPC, ECS service, RDS) that is automatically destroyed when the PR is closed.
>
> How would you implement this using Terraform and GitHub Actions? What challenges would you face and how would you solve them?

📁 **Reference:** `nawab312/Terraform` and `nawab312/CI_CD` → `GithubActions` — ephemeral environments, workspace management sections

---

### Q124 — Prometheus | Troubleshooting | Advanced

> Your Prometheus alerts are **firing and resolving repeatedly** every few minutes — this is called **alert flapping**. The on-call team is getting spammed with notifications.
>
> What causes alert flapping? How would you fix it in Prometheus alerting rules and in Alertmanager?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — alerting rules, Alertmanager, inhibition, silences sections

---

### Q125 — AWS | Conceptual | Advanced

> Explain **AWS Auto Scaling Groups (ASG)** in detail. What are the different **scaling policies** (Target Tracking, Step Scaling, Simple Scaling, Scheduled)?
>
> What is the difference between **scale-out** and **scale-in** cooldown periods? What is **instance warm-up** and why is it important?

📁 **Reference:** `nawab312/AWS` — Auto Scaling Groups, scaling policies sections

---

### Q126 — Kubernetes | Scenario-Based | Advanced

> Your company wants to implement **Pod security** hardening on Kubernetes. You need to ensure:
> - No containers run as **root**
> - No containers have **privilege escalation**
> - All containers have **read-only root filesystem**
> - No containers use **host networking or host PID**
>
> How would you enforce this across the entire cluster? Write the relevant Kubernetes resources.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md` — Pod Security Standards, Security Contexts, OPA/Gatekeeper sections

---

### Q127 — ArgoCD | Scenario-Based | Advanced

> You are using ArgoCD and need to implement **secret management** for your applications. Kubernetes Secrets are base64 encoded (not encrypted) and you cannot store them in Git.
>
> Explain at least **3 different approaches** to manage secrets with ArgoCD/GitOps and the **trade-offs** of each approach.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — secrets management, Sealed Secrets, External Secrets Operator sections

---

### Q128 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** for a production **deployment rollback system** that:
> - Maintains the **last 5 deployment versions** in a directory
> - Has a `deploy <version>` command to deploy a specific version
> - Has a `rollback` command to roll back to the **previous version**
> - Has a `list` command to show all available versions with timestamps
> - **Logs all deployment actions** with timestamps to a log file
> - Sends a **notification** (curl to Slack webhook) on every deploy/rollback

📁 **Reference:** `nawab312/DSA` → `Linux` — Bash scripting, functions, logging sections

---

### Q129 — AWS | Scenario-Based | Advanced

> Your company stores **PII (Personally Identifiable Information)** in DynamoDB and S3. A security audit requires you to implement:
> - **Encryption at rest** for all data
> - **Encryption in transit** for all API calls
> - **Data masking** for logs
> - **Access audit trail** for all reads/writes to PII data
> - **Data retention policy** — auto-delete after 2 years
>
> How would you implement each requirement on AWS?

📁 **Reference:** `nawab312/AWS` — KMS, DynamoDB encryption, S3 encryption, CloudTrail, data lifecycle sections

---

### Q130 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `ServiceAccount` tokens** — how have they changed from **static long-lived tokens** to **bound service account tokens** in newer Kubernetes versions?
>
> What is **IRSA (IAM Roles for Service Accounts)** in EKS and how does it work? Why is it better than using EC2 instance profiles for pod-level AWS access?

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md` — ServiceAccounts, IRSA, token projection sections

---

### Q131 — Grafana | Troubleshooting | Advanced

> Your Grafana instance is **very slow to load dashboards**. A dashboard that used to load in 2 seconds now takes 45 seconds. The Grafana server CPU and memory look normal.
>
> Walk me through your investigation — what are the possible causes and how would you fix them?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — dashboard performance, query optimization, caching sections

---

### Q132 — Terraform | Conceptual | Advanced

> Explain **Terraform `moved` block**, **`removed` block**, and **`import` block** (Terraform 1.5+).
>
> How do these new features improve on the old `terraform state mv` and `terraform import` CLI commands? Give a real-world example of using the `moved` block during a module refactor.

📁 **Reference:** `nawab312/Terraform` — Terraform 1.x features, state refactoring sections

---

### Q133 — Git | Scenario-Based | Advanced

> Your team accidentally committed and pushed **secrets (AWS credentials)** to a public GitHub repository 3 days ago. The commit is deep in the history.
>
> Walk me through your **complete incident response**:
> 1. Immediately secure the exposed credentials
> 2. Remove the secrets from Git history permanently
> 3. Prevent this from happening again

📁 **Reference:** `nawab312/CI_CD` → `Git` — git history rewriting, BFG Repo Cleaner, pre-commit hooks sections

---

### Q134 — AWS + Kubernetes | Scenario-Based | Advanced

> You are migrating a **monolithic application** from a single EC2 instance to **microservices on EKS**. The monolith has:
> - A PostgreSQL database with 1TB of data
> - File storage using local disk (images, uploads)
> - Session state stored in memory
> - Background jobs using cron
>
> What is your **migration strategy** for each component and what AWS/Kubernetes services would you use for each?

📁 **Reference:** `nawab312/AWS` and `nawab312/Kubernetes` — EKS, RDS, EFS, ElastiCache, Jobs/CronJobs sections

---

### Q135 — All Topics | System Design | Advanced

> You are designing the **observability and reliability platform** for a fintech company processing **1 million transactions per day**. Requirements:
> - **Regulatory compliance** — all logs retained for 7 years
> - **Real-time fraud detection** alerts — under 1 second latency
> - **99.999% uptime** for the payment processing service
> - **Full distributed tracing** across 30 microservices
> - **Capacity planning** — predict infrastructure needs 6 months ahead
>
> Design the complete platform covering: logging, metrics, tracing, alerting, SLOs, and capacity planning.

📁 **Reference:** All repositories — comprehensive observability and reliability platform design

---

## Key Topics Coverage (Q91–Q135)

| Topic | Questions |
|---|---|
| Kubernetes | Q91, Q96, Q101, Q107, Q111, Q116, Q121, Q126, Q130 |
| AWS | Q92, Q98, Q105, Q110, Q115, Q120, Q125, Q129, Q134 |
| Terraform | Q94, Q103, Q113, Q123, Q132 |
| Prometheus | Q95, Q106, Q124 |
| Grafana | Q112, Q131 |
| ELK Stack | Q100, Q117 |
| Linux / Bash | Q93, Q102, Q109, Q119, Q128 |
| ArgoCD | Q108, Q127 |
| Jenkins | Q99, Q118 |
| GitHub Actions | Q104, Q122 |
| Git | Q97, Q133 |
| Python | Q114 |
| System Design | Q135 |

---

## Important Concepts Not to Miss

### Kubernetes
- Admission Webhooks (Validating + Mutating)
- Pod termination lifecycle + graceful shutdown
- QoS classes + eviction order
- ServiceAccount tokens + IRSA
- Pod Security Standards
- Canary deployments without service mesh
- Init containers vs sidecars
- API server performance at scale

### AWS
- Zero Trust on AWS
- Multi-region active-active
- RDS connection exhaustion + RDS Proxy
- S3 PII data security
- Transit Gateway vs VPC Peering vs PrivateLink
- ASG scaling policies in detail

### Terraform
- `for_each` vs `count` — resource replacement issue
- Terraform at scale (50+ services)
- Ephemeral environments per PR
- Terraform 1.5+ `moved` and `import` blocks
- Slow plan optimization

### Prometheus
- Thanos for long-term storage
- Federation
- Alert flapping + Alertmanager remedies
- Kafka monitoring setup

### Linux
- iptables chains
- Linux namespaces + cgroups (container foundation)
- High load average with low CPU (I/O wait)
- OOM killer tuning

---

*Part 3 of DevOps/SRE Interview Questions — nawab312 GitHub repositories*
