# DevOps / SRE / Cloud Interview Questions (46–90)

> Continuation of interview preparation — questions only for self-practice.
> Based on your GitHub repositories. References provided for each question.

---

## Table of Contents

| # | Topic | Type | Difficulty |
|---|---|---|---|
| 46 | Kubernetes | Conceptual | Advanced |
| 47 | AWS | Scenario-Based | Advanced |
| 48 | Terraform | Conceptual | Advanced |
| 49 | Kubernetes | Scenario-Based | Advanced |
| 50 | Linux / Bash | Conceptual | Advanced |
| 51 | Prometheus | Scenario-Based | Advanced |
| 52 | Kubernetes | Troubleshooting | Advanced |
| 53 | AWS | Conceptual | Advanced |
| 54 | Git | Scenario-Based | Advanced |
| 55 | Jenkins | Conceptual | Advanced |
| 56 | Kubernetes | Conceptual | Advanced |
| 57 | Linux / Bash | Scenario-Based | Advanced |
| 58 | AWS | Troubleshooting | Advanced |
| 59 | ArgoCD | Scenario-Based | Advanced |
| 60 | Terraform | Troubleshooting | Advanced |
| 61 | Kubernetes | Scenario-Based | Advanced |
| 62 | ELK Stack | Scenario-Based | Advanced |
| 63 | GitHub Actions | Conceptual | Advanced |
| 64 | AWS | Scenario-Based | Advanced |
| 65 | Prometheus | Troubleshooting | Advanced |
| 66 | Kubernetes | Conceptual | Advanced |
| 67 | Linux / Bash | Scenario-Based | Advanced |
| 68 | Terraform | Scenario-Based | Advanced |
| 69 | AWS | Conceptual | Advanced |
| 70 | Kubernetes | Troubleshooting | Advanced |
| 71 | Grafana | Scenario-Based | Advanced |
| 72 | Python | Scenario-Based | Advanced |
| 73 | Jenkins | Troubleshooting | Advanced |
| 74 | Kubernetes | Conceptual | Advanced |
| 75 | AWS | Scenario-Based | Advanced |
| 76 | Linux / Bash | Conceptual | Advanced |
| 77 | ArgoCD | Conceptual | Advanced |
| 78 | Terraform | Conceptual | Advanced |
| 79 | Kubernetes | Scenario-Based | Advanced |
| 80 | AWS | Troubleshooting | Advanced |
| 81 | ELK Stack | Conceptual | Advanced |
| 82 | GitHub Actions | Troubleshooting | Advanced |
| 83 | Kubernetes | Conceptual | Advanced |
| 84 | Prometheus + Grafana | Scenario-Based | Advanced |
| 85 | AWS | Conceptual | Advanced |
| 86 | Linux / Bash | Troubleshooting | Advanced |
| 87 | Terraform | Scenario-Based | Advanced |
| 88 | Kubernetes | Conceptual | Advanced |
| 89 | AWS + Kubernetes | System Design | Advanced |
| 90 | All Topics | System Design | Advanced |

---

## Questions

---

### Q46 — Kubernetes | Conceptual | Advanced

> Explain what a **Kubernetes Operator** is. How is it different from a regular **Controller**?
>
> Give a real-world example of when you would build or use an Operator instead of a standard Kubernetes resource like a Deployment or StatefulSet.

📁 **Reference:** `nawab312/Kubernetes` → `10_CRDs_EXTENSIBILITY.md` — Operators, CRDs, and Controllers sections

---

### Q47 — AWS | Scenario-Based | Advanced

> Your company has a **microservices architecture** on AWS where services communicate with each other. You are seeing **intermittent timeout errors** between Service A and Service B.
>
> Service A calls Service B via HTTP. Sometimes the call succeeds, sometimes it times out with no clear pattern.
>
> **How would you systematically investigate this, and what are the possible root causes?**

📁 **Reference:** `nawab312/AWS` — VPC, Security Groups, Load Balancers, and CloudWatch sections

---

### Q48 — Terraform | Conceptual | Advanced

> What is the difference between **`terraform workspace`** and having **separate directories per environment** (like `environments/dev`, `environments/prod`)?
>
> What are the **pros and cons** of each approach, and which one would you recommend for a production setup and why?

📁 **Reference:** `nawab312/Terraform` — Workspaces and environment management sections

---

### Q49 — Kubernetes | Scenario-Based | Advanced

> You have a **multi-tenant Kubernetes cluster** where different teams share the same cluster. Team A's application is consuming **excessive CPU and memory**, causing other teams' applications to experience **resource starvation and evictions**.
>
> How would you **isolate resources between teams** and **prevent one team from affecting others**?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — Resource Quotas, LimitRanges, Namespaces, and Priority Classes sections

---

### Q50 — Linux / Bash | Conceptual | Advanced

> Explain the difference between a **process** and a **thread** in Linux.
>
> Also explain what **zombie processes** and **orphan processes** are — how they are created, what problems they cause, and how you would identify and clean them up.

📁 **Reference:** `nawab312/DSA` → `Linux` — process management, signals, and process states sections

---

### Q51 — Prometheus | Scenario-Based | Advanced

> You are asked to monitor a **Node.js microservice** running on Kubernetes. The service currently exposes **no metrics**.
>
> Walk me through how you would instrument the application, expose metrics, and write **PromQL queries** to answer these questions:
> - What is the **request rate** over the last 5 minutes?
> - What is the **p99 latency** of the `/api/checkout` endpoint?
> - What percentage of requests are **returning 5xx errors**?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — instrumentation, PromQL sections

---

### Q52 — Kubernetes | Troubleshooting | Advanced

> A Node in your Kubernetes cluster shows status **`NotReady`**. Pods on that node are being evicted and rescheduled on other nodes.
>
> **Walk me through your complete troubleshooting process** to identify why the node is NotReady and how you would recover it.

📁 **Reference:** `nawab312/Kubernetes` → `09_CLUSTER_OPERATIONS.md` and `14_TROUBLESHOOTING.md`

---

### Q53 — AWS | Conceptual | Advanced

> Explain the difference between **AWS CloudWatch**, **AWS CloudTrail**, and **AWS Config**.
>
> Give a real-world example of when you would use each one, and explain how they complement each other in a **security and compliance** context.

📁 **Reference:** `nawab312/AWS` — CloudWatch, CloudTrail, and AWS Config sections

---

### Q54 — Git | Scenario-Based | Advanced

> Your team uses **GitFlow** branching strategy. A critical bug is found in production. You need to:
> - Fix it **immediately** without waiting for the current feature branch cycle
> - Deploy the fix to **production**
> - Make sure the fix is also included in the **current development branch**
>
> Walk me through the **exact Git commands** you would use from start to finish.

📁 **Reference:** `nawab312/CI_CD` → `Git` — branching strategies and hotfix workflow sections

---

### Q55 — Jenkins | Conceptual | Advanced

> Explain the difference between **Declarative** and **Scripted** Jenkins pipelines.
>
> Also explain what **Shared Libraries** are in Jenkins and give a real-world example of when and how you would use them.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — Declarative vs Scripted pipeline and Shared Libraries sections

---

### Q56 — Kubernetes | Conceptual | Advanced

> Explain how **Kubernetes DNS** works. When a Pod makes a request to `backend.production.svc.cluster.local`, what happens step by step?
>
> Also explain what **ndots** is and how it affects DNS resolution performance.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — DNS in Kubernetes sections

---

### Q57 — Linux / Bash | Scenario-Based | Advanced

> You need to write a **Bash script** that:
> - Takes a list of servers from a file (`servers.txt`)
> - SSH into each server **in parallel**
> - Runs `df -h` and collects disk usage
> - Outputs a **consolidated report** showing servers where disk usage is above **80%**
> - Handles **SSH failures** gracefully (timeout, unreachable)

📁 **Reference:** `nawab312/DSA` → `Linux` — Bash scripting, SSH, parallel execution sections

---

### Q58 — AWS | Troubleshooting | Advanced

> Your application on AWS is experiencing **high latency** and **increased error rates** during business hours. The infrastructure looks healthy — EC2 instances are running, RDS is available, no alarms firing.
>
> But users are complaining. Where would you look and how would you investigate?

📁 **Reference:** `nawab312/AWS` — CloudWatch, ALB, RDS Performance Insights, X-Ray sections

---

### Q59 — ArgoCD | Scenario-Based | Advanced

> You need to set up a **GitOps workflow** using ArgoCD for a microservice that:
> - Has **3 environments**: dev, staging, prod
> - Uses **Helm charts** for deployment
> - Prod deployments require **manual approval**
> - Each environment has **different values** (replicas, resource limits, image tags)
>
> How would you structure the Git repository and configure ArgoCD Applications?

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — Helm integration, multi-environment, sync policies sections

---

### Q60 — Terraform | Troubleshooting | Advanced

> You run `terraform plan` and see **unexpected resource replacements** — Terraform wants to destroy and recreate resources that you did not change.
>
> What are the possible reasons this happens, and how would you investigate and prevent unwanted replacements?

📁 **Reference:** `nawab312/Terraform` — state management, resource lifecycle, and `ignore_changes` sections

---

### Q61 — Kubernetes | Scenario-Based | Advanced

> Your team is migrating a **stateful application** (PostgreSQL) from a VM to Kubernetes. The database has **500GB of data** and must have **minimal downtime** during migration.
>
> Walk me through your **migration strategy** — what Kubernetes resources you would use, how you would handle the data migration, and how you would minimize downtime.

📁 **Reference:** `nawab312/Kubernetes` → `04_STORAGE.md` and `02_WORKLOADS.md` — StatefulSets, PVCs, and Storage Classes sections

---

### Q62 — ELK Stack | Scenario-Based | Advanced

> Your **Elasticsearch cluster** is showing **RED health status**. Kibana shows no data and Logstash is reporting indexing failures.
>
> Walk me through your **complete investigation and recovery process**.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Elasticsearch cluster health, shards, and recovery sections

---

### Q63 — GitHub Actions | Conceptual | Advanced

> Explain the difference between **GitHub Actions `jobs`** and **`steps`**. When do jobs run in **parallel** vs **sequentially**?
>
> Also explain **reusable workflows** and **composite actions** — what they are and when you would use each.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — workflow structure, reusable workflows sections

---

### Q64 — AWS | Scenario-Based | Advanced

> Your company wants to implement a **disaster recovery (DR) strategy** for a critical application running on AWS in `us-east-1`. The **RTO is 1 hour** and **RPO is 15 minutes**.
>
> What DR strategy would you implement, what AWS services would you use, and how would you test it?

📁 **Reference:** `nawab312/AWS` — Disaster Recovery, RDS, Route53, S3 Cross-Region Replication sections

---

### Q65 — Prometheus | Troubleshooting | Advanced

> Your Prometheus instance is **consuming excessive memory** and eventually **crashing with OOM**. The `/metrics` endpoint of Prometheus itself shows extremely high `prometheus_tsdb_head_series` count.
>
> What is causing this, how would you investigate, and how would you fix it?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — TSDB, cardinality, and retention sections

---

### Q66 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes Service Mesh** and specifically **Istio**. What problems does it solve that standard Kubernetes networking cannot?
>
> Explain the difference between **mTLS**, **traffic shifting**, and **circuit breaking** in the context of Istio.

📁 **Reference:** `nawab312/Kubernetes` → `11_ISTIO_SERVICE_MESH.md`

---

### Q67 — Linux / Bash | Scenario-Based | Advanced

> You are investigating a production server where the application is **intermittently slow**. You suspect there is a **I/O bottleneck** — disk reads/writes are causing latency spikes.
>
> Walk me through the **Linux commands and tools** you would use to confirm and diagnose the I/O bottleneck.

📁 **Reference:** `nawab312/DSA` → `Linux` — I/O monitoring, iostat, iotop sections

---

### Q68 — Terraform | Scenario-Based | Advanced

> Your company is moving from **manually managed AWS infrastructure** to **Terraform**. There are hundreds of existing resources in AWS that were created manually via the console.
>
> How would you approach **importing all existing infrastructure** into Terraform without causing any downtime or disruptions?

📁 **Reference:** `nawab312/Terraform` — `terraform import`, state management sections

---

### Q69 — AWS | Conceptual | Advanced

> Explain the difference between **AWS ALB (Application Load Balancer)** and **AWS NLB (Network Load Balancer)**.
>
> When would you choose one over the other? Give real-world examples for each. Also explain what **target groups** are and how **health checks** work differently between ALB and NLB.

📁 **Reference:** `nawab312/AWS` — ALB, NLB, Load Balancing sections

---

### Q70 — Kubernetes | Troubleshooting | Advanced

> Your Kubernetes cluster is running fine but you notice **Pods are being evicted** frequently even though the nodes have available resources according to `kubectl top nodes`.
>
> What are the possible reasons for unexpected evictions and how would you investigate and stop them?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — eviction, QoS classes, and resource management sections

---

### Q71 — Grafana | Scenario-Based | Advanced

> You want to build a **Grafana dashboard** that shows the **SLO (Service Level Objective)** for your API:
> - **Availability SLO**: 99.9% uptime
> - **Latency SLO**: 95% of requests under 500ms
>
> Write the **PromQL queries** for each SLO and explain how you would set up an **error budget** panel.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` and `Prometheus` — SLO, SLA, error budget sections

---

### Q72 — Python | Scenario-Based | Advanced

> Write a **Python script** that:
> - Reads a list of EC2 instance IDs from a CSV file
> - Uses **boto3** to get the current state, instance type, and tags for each instance
> - Outputs a **formatted table** showing: Instance ID, State, Type, Name tag, and Environment tag
> - Handles **API errors and rate limiting** gracefully

📁 **Reference:** `nawab312/DSA` → `Python` — boto3, CSV handling, error handling sections

---

### Q73 — Jenkins | Troubleshooting | Advanced

> Your Jenkins master is **running out of disk space** on `/var/lib/jenkins`. The CI/CD pipelines are starting to fail with disk-related errors.
>
> What is consuming the disk space in Jenkins, and how would you **clean it up and prevent it from happening again**?

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — Jenkins maintenance, build artifacts, workspace cleanup sections

---

### Q74 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes Pod Quality of Service (QoS) classes** — **Guaranteed**, **Burstable**, and **BestEffort**.
>
> How does Kubernetes decide which class a Pod belongs to? What is the **eviction order** when a node runs out of memory, and how do QoS classes affect it?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — QoS classes and eviction sections

---

### Q75 — AWS | Scenario-Based | Advanced

> Your company stores sensitive customer data in **S3 buckets**. A security audit finds that some buckets are **publicly accessible** and some have **no encryption**.
>
> How would you **audit all S3 buckets** across your AWS account for security misconfigurations, fix them, and **prevent this from happening again** using automation?

📁 **Reference:** `nawab312/AWS` — S3 security, AWS Config, IAM policies sections

---

### Q76 — Linux / Bash | Conceptual | Advanced

> Explain the Linux **`/proc` filesystem**. What kind of information can you get from it?
>
> Give **5 practical examples** of how a DevOps/SRE engineer would use `/proc` to diagnose production issues, with the exact file paths and what they reveal.

📁 **Reference:** `nawab312/DSA` → `Linux` — `/proc` filesystem sections

---

### Q77 — ArgoCD | Conceptual | Advanced

> Explain the difference between **ArgoCD** and **Flux** as GitOps tools.
>
> Also explain what **ApplicationSets** are in ArgoCD and give a real-world example of when you would use them over individual Applications.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — ApplicationSets and GitOps comparison sections

---

### Q78 — Terraform | Conceptual | Advanced

> What are **Terraform `depends_on`**, **`lifecycle`**, and **`provisioner`** blocks?
>
> When would you use each one? Are there cases where you should **avoid** using them? Give real-world examples.

📁 **Reference:** `nawab312/Terraform` — resource lifecycle, dependencies, and provisioners sections

---

### Q79 — Kubernetes | Scenario-Based | Advanced

> Your company is running a **batch processing system** on Kubernetes. The system processes millions of records every night. Currently it uses a **Deployment** but you are experiencing:
> - Failed jobs leaving no trace
> - No retry on failure
> - Cannot track completion status
>
> How would you redesign this using the **right Kubernetes workload type**, and what configuration would you use to handle retries, parallelism, and cleanup?

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md` — Jobs and CronJobs sections

---

### Q80 — AWS | Troubleshooting | Advanced

> Your **AWS Lambda function** is working fine in dev but **timing out in production**. The function calls an RDS database and an external API.
>
> The timeout is set to 30 seconds but the function sometimes runs for the full 30 seconds and times out. Walk me through your investigation.

📁 **Reference:** `nawab312/AWS` — Lambda, RDS, VPC configuration, CloudWatch Logs Insights sections

---

### Q81 — ELK Stack | Conceptual | Advanced

> Explain how **Elasticsearch indexing** works internally. What are **shards** and **replicas**, and how do they affect **performance** and **availability**?
>
> If you have an index with 5 primary shards and 1 replica on a 3-node cluster, what happens to the cluster when **one node goes down**?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Elasticsearch architecture, shards, replicas sections

---

### Q82 — GitHub Actions | Troubleshooting | Advanced

> Your GitHub Actions workflow is **passing locally** (act) but **failing in the GitHub Actions runner** with:
>
> `Error: EACCES: permission denied, open '/github/workspace/output.json'`
>
> And separately, another workflow is **passing but taking 45 minutes** when it should take 5 minutes.
>
> Diagnose and fix **both issues**.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — runner permissions, caching, and workflow optimization sections

---

### Q83 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `PodDisruptionBudget` (PDB)**. What problem does it solve?
>
> Write a PDB for a deployment that has 5 replicas and must always have at least 3 running. What happens during a **node drain** if the PDB cannot be satisfied?

📁 **Reference:** `nawab312/Kubernetes` → `09_CLUSTER_OPERATIONS.md` — PodDisruptionBudget and cluster maintenance sections

---

### Q84 — Prometheus + Grafana | Scenario-Based | Advanced

> You are asked to implement **SLO monitoring** for a payment service with these requirements:
> - **Availability**: 99.95% over 30 days
> - **Latency**: 99% of requests under 200ms
> - Alert when **error budget is 50% consumed**
> - Alert when **error budget burn rate is too fast**
>
> Write the complete **Prometheus alerting rules** and explain the **multi-window, multi-burn-rate** alerting approach.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` and `Grafana` — SLO, error budget, burn rate alerting sections

---

### Q85 — AWS | Conceptual | Advanced

> Explain **AWS IAM roles, policies, and permission boundaries**.
>
> What is the difference between **identity-based policies** and **resource-based policies**? Give an example of each.
>
> Also explain **AWS STS (Security Token Service)** and how **cross-account access** works using IAM roles.

📁 **Reference:** `nawab312/AWS` — IAM, STS, cross-account access sections

---

### Q86 — Linux / Bash | Troubleshooting | Advanced

> A production server is experiencing **random kernel OOM (Out of Memory) kills**. The application is being killed randomly even though `free -h` shows available memory.
>
> Explain what the **OOM killer** is, how it decides which process to kill, and walk me through how you would **tune it** and **prevent critical processes from being killed**.

📁 **Reference:** `nawab312/DSA` → `Linux` — memory management, OOM killer, `/proc/sys/vm` sections

---

### Q87 — Terraform | Scenario-Based | Advanced

> You need to deploy infrastructure across **5 AWS regions** using Terraform. The same set of resources (VPC, ECS, RDS) needs to be created in each region with region-specific configurations.
>
> How would you structure your Terraform code to handle **multi-region deployments** efficiently without massive code duplication?

📁 **Reference:** `nawab312/Terraform` — modules, providers, multi-region deployment sections

---

### Q88 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `init containers`**. How are they different from regular containers and **sidecar containers**?
>
> Give **3 real-world use cases** for init containers and write a Pod spec that uses an init container to wait for a database to be ready before starting the main application.

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md` — init containers and sidecar patterns sections

---

### Q89 — AWS + Kubernetes | System Design | Advanced

> Design a **complete CI/CD and infrastructure architecture** for a company that:
> - Has **20 microservices** all deployed on **EKS**
> - Needs **blue/green deployments** for zero downtime
> - Uses **multiple AWS accounts** (dev, staging, prod)
> - Needs **secrets management** for database passwords and API keys
> - Requires **audit logging** of all deployments
>
> What tools, services, and patterns would you use end-to-end?

📁 **Reference:** All repositories — this is a system design question covering all topics

---

### Q90 — All Topics | System Design | Advanced

> You are the lead SRE at a company with a **microservices application** running on Kubernetes. The CTO wants to achieve:
> - **99.99% uptime SLO**
> - **Mean Time to Recovery (MTTR) under 5 minutes**
> - **Zero-downtime deployments**
> - **Full observability** (metrics, logs, traces)
> - **Automated incident response**
>
> Design the **complete SRE platform** to achieve these goals. Cover: deployment strategy, monitoring, alerting, on-call, chaos engineering, and runbooks.

📁 **Reference:** All repositories — comprehensive SRE platform design

---

## Key Topics Coverage (Q46–Q90)

| Topic | Questions |
|---|---|
| Kubernetes | Q46, Q49, Q52, Q56, Q61, Q66, Q70, Q74, Q79, Q83, Q88 |
| AWS | Q47, Q53, Q58, Q64, Q69, Q75, Q80, Q85, Q89 |
| Terraform | Q48, Q60, Q68, Q78, Q87 |
| Prometheus | Q51, Q65, Q84 |
| Grafana | Q71, Q84 |
| ELK Stack | Q62, Q81 |
| Linux / Bash | Q50, Q57, Q67, Q76, Q86 |
| ArgoCD | Q59, Q77 |
| Jenkins | Q55, Q73 |
| GitHub Actions | Q63, Q82 |
| Git | Q54 |
| Python | Q72 |
| System Design | Q89, Q90 |

---

*Part 2 of DevOps/SRE Interview Questions — nawab312 GitHub repositories*
