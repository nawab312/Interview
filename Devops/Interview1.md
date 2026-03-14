# DevOps / SRE / Cloud Interview Questions

> All questions are based on your GitHub repositories. References are provided for each question so you can revise the relevant section.

---

## Table of Contents

| # | Topic | Type | Difficulty |
|---|---|---|---|
| 1 | Kubernetes | Conceptual | Beginner |
| 2 | Kubernetes | Scenario-Based | Beginner-Intermediate |
| 3 | Kubernetes | Conceptual | Intermediate |
| 4 | Kubernetes | Troubleshooting | Intermediate |
| 5 | Terraform | Conceptual | Intermediate |
| 6 | Prometheus | Conceptual | Intermediate |
| 7 | AWS | Scenario-Based | Intermediate |
| 8 | Linux / Bash | Scenario-Based | Intermediate |
| 9 | ArgoCD | Conceptual | Intermediate |
| 10 | Jenkins / CI-CD | Troubleshooting | Intermediate |
| 11 | Git | Conceptual | Intermediate |
| 12 | ELK Stack | Conceptual | Intermediate |
| 13 | GitHub Actions | Scenario-Based | Intermediate |
| 14 | Grafana | Conceptual | Intermediate |
| 15 | Python | Scenario-Based | Intermediate |
| 16 | Kubernetes | Scenario-Based | Advanced |
| 17 | Terraform | Troubleshooting | Advanced |
| 18 | AWS | Conceptual | Advanced |
| 19 | Kubernetes | Scenario-Based | Advanced |
| 20 | Linux / Bash | Troubleshooting | Advanced |
| 21 | Kubernetes | Conceptual | Advanced |
| 22 | AWS | Scenario-Based | Advanced |
| 23 | Prometheus | Conceptual | Advanced |
| 24 | ArgoCD | Troubleshooting | Advanced |
| 25 | Terraform | Scenario-Based | Advanced |
| 26 | Kubernetes Networking | Scenario-Based | Advanced |
| 27 | Git | Conceptual | Advanced |
| 28 | ELK Stack | Troubleshooting | Advanced |
| 29 | AWS | Scenario-Based | Advanced |
| 30 | Kubernetes | Conceptual | Advanced |
| 31 | GitHub Actions | Scenario-Based | Advanced |
| 32 | Terraform | Conceptual | Advanced |
| 33 | Kubernetes | Troubleshooting | Advanced |
| 34 | Jenkins | Scenario-Based | Advanced |
| 35 | AWS | Conceptual | Advanced |
| 36 | Prometheus + Grafana | Scenario-Based | Advanced |
| 37 | Linux / Bash | Scenario-Based | Advanced |
| 38 | Kubernetes | Conceptual | Advanced |
| 39 | Terraform + AWS | Scenario-Based | Advanced |
| 40 | Kubernetes + Prometheus + Grafana | Scenario-Based | Advanced |
| 41 | AWS | Conceptual | Advanced |
| 42 | AWS + Kubernetes | Troubleshooting | Advanced |
| 43 | Kubernetes | Conceptual | Advanced |
| 44 | Linux + Bash | Scenario-Based | Advanced |
| 45 | CI-CD + Kubernetes + AWS | System Design | Advanced |

---

## Questions

---

### Q1 — Kubernetes | Conceptual | Beginner

> When a Pod is scheduled on a Node and the kubelet starts it, the Pod goes through several phases. Can you walk me through the **Pod lifecycle phases** — from `Pending` to `Running` to `Succeeded/Failed` — and explain what happens at each stage?

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`

---

### Q2 — Kubernetes | Scenario-Based | Beginner-Intermediate

> You deploy an application and the Pod shows status `Running`, but your users are still unable to reach the service. You check and the Pod has been Running for 2 minutes.
>
> **What are the possible reasons, and how would you diagnose this?**

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md` and `03_NETWORKING.md`

---

### Q3 — Kubernetes | Conceptual | Intermediate

> Can you explain the difference between a **Deployment** and a **StatefulSet**? When would you choose one over the other? Give a real-world example for each.

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`

---

### Q4 — Kubernetes | Troubleshooting | Intermediate

> You have a Pod stuck in **`CrashLoopBackOff`**. Walk me through your **step-by-step troubleshooting process** to identify and fix the root cause.

📁 **Reference:** `nawab312/Kubernetes` → `14_TROUBLESHOOTING.md`

---

### Q5 — Terraform | Conceptual | Intermediate

> What is the difference between **`terraform plan`** and **`terraform apply`**? Also, what is the purpose of the **Terraform state file** and what risks come with it?

📁 **Reference:** `nawab312/Terraform` — state management and core workflow sections

---

### Q6 — Prometheus | Conceptual | Intermediate

> What are the **4 core metric types** in Prometheus? Explain each with a real-world example of when you would use it.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`

---

### Q7 — AWS | Scenario-Based | Intermediate

> Your application is running on an **EC2 instance** and it needs to read files from an **S3 bucket**. A junior engineer suggests hardcoding AWS Access Keys inside the application code to authenticate.
>
> **Why is this a bad idea, and what is the correct AWS-native way to handle this?**

📁 **Reference:** `nawab312/AWS` — IAM, IAM Roles, EC2 Instance Profiles sections

---

### Q8 — Linux / Bash | Scenario-Based | Intermediate

> You are on-call and get an alert that a production server is consuming unexpectedly **high CPU**. Walk me through how you would **identify the process** and **investigate the root cause** using command line tools.

📁 **Reference:** `nawab312/DSA` → `Linux` — process management and system monitoring sections

---

### Q9 — ArgoCD | Conceptual | Intermediate

> What is the difference between ArgoCD's **Sync** and **Refresh** operations? Also explain what **App of Apps** pattern is and why it is used.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — Sync, Refresh, and App of Apps sections

---

### Q10 — Jenkins / CI-CD | Troubleshooting | Intermediate

> Your Jenkins pipeline was working fine yesterday, but today it is **failing at the build stage** with the error:
>
> `ERROR: Could not resolve dependencies for project: Connection timed out`
>
> **How would you troubleshoot this, and what are the possible root causes?**

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — pipeline troubleshooting and build failures sections

---

### Q11 — Git | Conceptual | Intermediate

> What is the difference between **`git merge`** and **`git rebase`**? When would you use one over the other in a team environment?

📁 **Reference:** `nawab312/CI_CD` → `Git` — merge vs rebase section

---

### Q12 — ELK Stack | Conceptual | Intermediate

> Can you explain the **role of each component** in the ELK Stack — **Elasticsearch, Logstash, and Kibana**? Also, what is **Beats** and where does it fit in the pipeline?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — ELK architecture and Beats sections

---

### Q13 — GitHub Actions | Scenario-Based | Intermediate

> You have a GitHub Actions workflow that **builds and deploys** your application to AWS. You want to make sure the deployment to **production** only happens when a PR is merged into `main`, and it should **never trigger on a push to a feature branch**.
>
> How would you structure your workflow file to achieve this?

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — workflow triggers and event filters sections

---

### Q14 — Grafana | Conceptual | Intermediate

> What is the difference between a **Grafana Dashboard** and a **Grafana Alert**? Also explain what a **Data Source** is in Grafana and name at least **4 data sources** Grafana supports.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — dashboards, alerts, and data sources sections

---

### Q15 — Python | Scenario-Based | Intermediate

> You are given a log file where each line looks like this:
>
> ```
> 2024-01-15 10:23:45 ERROR Service unavailable
> 2024-01-15 10:24:01 INFO Request received
> 2024-01-15 10:24:05 ERROR Timeout exceeded
> 2024-01-15 10:24:10 WARNING High memory usage
> ```
>
> Write a **Python script** that reads this log file, counts the **occurrences of each log level** (ERROR, INFO, WARNING etc.), and prints a summary.

📁 **Reference:** `nawab312/DSA` → `Python` — file handling, string manipulation, and dictionaries sections

---

### Q16 — Kubernetes | Scenario-Based | Advanced

> Your production cluster has a **memory leak** in one of your microservices. The Pod keeps consuming more and more memory until it gets **OOMKilled** and restarts, causing intermittent downtime.
>
> **How would you handle this situation both immediately and long-term?**

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — Resource Requests/Limits and OOMKilled sections

---

### Q17 — Terraform | Troubleshooting | Advanced

> You run `terraform apply` and it **fails halfway through** — some resources got created, some didn't. Now your **state file is out of sync** with the real infrastructure.
>
> How would you **recover from this situation** without destroying and recreating everything from scratch?

📁 **Reference:** `nawab312/Terraform` — state management, `terraform import`, and `terraform state` commands

---

### Q18 — AWS | Conceptual | Advanced

> Explain the difference between **Security Groups** and **Network ACLs (NACLs)** in AWS. If you have both configured, in what **order** are they evaluated, and what happens if they conflict?

📁 **Reference:** `nawab312/AWS` — VPC Security, Security Groups, and Network ACLs sections

---

### Q19 — Kubernetes | Scenario-Based | Advanced

> Your production cluster has a microservice experiencing **high latency** during peak traffic. The service has **HPA** configured, but Pods are **scaling up too late** — by the time new Pods are ready, the latency spike has already impacted users.
>
> **What are the possible reasons HPA is reacting slowly, and how would you fix it?**

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — HPA, Metrics Server, and scaling behavior sections

---

### Q20 — Linux / Bash | Troubleshooting | Advanced

> You are on-call and get an alert that a production server is **running out of disk space**. You SSH into the server and confirm with `df -h` that the root partition is at **95% usage**.
>
> Walk me through your **complete step-by-step process** to identify what is consuming the disk space and how you would free it up **without causing downtime**.

📁 **Reference:** `nawab312/DSA` → `Linux` — disk management, filesystem commands, and log management sections

---

### Q21 — Kubernetes | Conceptual | Advanced

> Explain how **RBAC (Role-Based Access Control)** works in Kubernetes. What is the difference between a **Role** and a **ClusterRole**? And what is the difference between a **RoleBinding** and a **ClusterRoleBinding**?
>
> Give a real-world example of when you would use each.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md` — RBAC, Roles, and ClusterRoles sections

---

### Q22 — AWS | Scenario-Based | Advanced

> Your company has a **multi-tier application** running on AWS:
> - Frontend (EC2) in a **Public Subnet**
> - Backend API (EC2) in a **Private Subnet**
> - Database (RDS) in a **Private Subnet**
>
> The Backend API needs to **download packages from the internet** during deployment, but it is in a private subnet with **no direct internet access**.
>
> **How would you architect this, and what AWS components would you use to solve this?**

📁 **Reference:** `nawab312/AWS` — VPC, NAT Gateway, Internet Gateway, and subnet architecture sections

---

### Q23 — Prometheus | Conceptual | Advanced

> What is the difference between **Recording Rules** and **Alerting Rules** in Prometheus? When would you use a Recording Rule, and can you write an example of both?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — Recording Rules and Alerting Rules sections

---

### Q24 — ArgoCD | Troubleshooting | Advanced

> You have an ArgoCD application that is showing status **`OutOfSync`** even after you click **Sync** and it completes successfully. The app keeps going back to `OutOfSync` within a few minutes.
>
> **What are the possible reasons for this, and how would you investigate and fix it?**

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — Sync Status, Resource Drift, and `ignoreDifferences` sections

---

### Q25 — Terraform | Scenario-Based | Advanced

> Your team has grown and **multiple engineers** are now working on the same Terraform codebase. You are noticing:
> - Engineers are duplicating code across environments (dev, staging, prod)
> - There is no consistency between environments
> - Sometimes two engineers run `terraform apply` simultaneously and corrupt the state
>
> **How would you restructure the Terraform codebase to solve all these problems?**

📁 **Reference:** `nawab312/Terraform` — Terraform Modules, Workspaces, Remote State, and State Locking sections

---

### Q26 — Kubernetes Networking | Scenario-Based | Advanced

> You have two microservices running in **different namespaces** in Kubernetes:
> - `frontend` in namespace `web`
> - `backend` in namespace `api`
>
> The frontend Pod **cannot reach** the backend Service. You have **NetworkPolicies** enabled in your cluster.
>
> **Walk me through how you would diagnose this, and what NetworkPolicy would you write to allow the traffic?**

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — Network Policies, cross-namespace communication, and DNS resolution sections

---

### Q27 — Git | Conceptual | Advanced

> Explain the difference between **`git reset`** and **`git revert`**. When would you use one over the other, especially in the context of a **shared branch in a team environment**?
>
> Also explain the three modes of `git reset` — **`--soft`**, **`--mixed`**, and **`--hard`**.

📁 **Reference:** `nawab312/CI_CD` → `Git` — undoing changes and reset vs revert sections

---

### Q28 — ELK Stack | Troubleshooting | Advanced

> Your application is sending logs to **Logstash**, but logs are **not appearing in Kibana**. The application is running fine and generating logs.
>
> **Walk me through your complete troubleshooting process** — from the application all the way to Kibana.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Logstash pipeline, Elasticsearch indexing, and Kibana index patterns sections

---

### Q29 — AWS | Scenario-Based | Advanced

> Your company is running a **highly available web application** on AWS. During a recent incident, your **entire application went down** because it was deployed in a **single Availability Zone** and that AZ had an outage.
>
> How would you **redesign the architecture** to be **highly available across multiple AZs**?

📁 **Reference:** `nawab312/AWS` — High Availability, Auto Scaling Groups, Load Balancers, and Multi-AZ deployments sections

---

### Q30 — Kubernetes | Conceptual | Advanced

> Explain how **Kubernetes Ingress** works end-to-end. What is the difference between an **Ingress Resource** and an **Ingress Controller**?
>
> Also explain how **TLS termination** works with Ingress and write an example Ingress manifest that:
> - Routes `/api` to a backend service
> - Routes `/` to a frontend service
> - Has TLS enabled

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — Ingress, Ingress Controller, and TLS termination sections

---

### Q31 — GitHub Actions | Scenario-Based | Advanced

> You are building a **CI/CD pipeline using GitHub Actions** for a microservices application. The pipeline needs to:
> - Run **tests** on every PR
> - Build and push a **Docker image** to ECR on merge to `main`
> - Deploy to **Kubernetes** using `kubectl` after image is pushed
> - Use **OIDC authentication** to AWS instead of storing long-lived credentials
>
> Write the **complete GitHub Actions workflow** for this.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — Docker builds, AWS ECR, kubectl deploy, and OIDC authentication sections

---

### Q32 — Terraform | Conceptual | Advanced

> What is the difference between **`terraform taint`** and **`terraform import`**?
>
> Also explain what **`terraform refresh`** does and when you would use it. Give a real-world scenario for each.

📁 **Reference:** `nawab312/Terraform` — state management, taint, import, and refresh commands

---

### Q33 — Kubernetes | Troubleshooting | Advanced

> You are running a **rolling update** on a Kubernetes Deployment and you notice the rollout is **stuck** — new Pods are in `Pending` state and old Pods are not being terminated.
>
> ```bash
> kubectl rollout status deployment/my-app
> Waiting for deployment "my-app" rollout to finish: 2 out of 5 new replicas have been updated...
> ```
>
> **What are the possible reasons and how would you diagnose and fix this?**

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md` and `14_TROUBLESHOOTING.md` — Rolling Updates, Deployment Strategies, and Pod Pending state sections

---

### Q34 — Jenkins | Scenario-Based | Advanced

> You need to design a **Jenkins pipeline** for a microservice that:
> - Runs on a **Kubernetes cluster** (Jenkins agents run as Pods)
> - Has **4 stages**: Test → Build Docker Image → Push to ECR → Deploy to K8s
> - Different stages run in **different containers**
> - Sensitive values come from **Jenkins Credentials Store**
> - If **deploy stage fails**, it should automatically **notify a Slack channel**
>
> Write the complete **Jenkinsfile** for this.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — Kubernetes agents, multi-container pods, credentials, and post actions sections

---

### Q35 — AWS | Conceptual | Advanced

> Explain the difference between **SQS** and **SNS** in AWS. When would you use one over the other?
>
> Also explain the **Fan-out pattern** — what it is and how you would implement it using SNS + SQS together. Give a real-world DevOps/SRE use case.

📁 **Reference:** `nawab312/AWS` — SQS, SNS, and messaging patterns sections

---

### Q36 — Prometheus + Grafana | Scenario-Based | Advanced

> Your team gets a **PagerDuty alert** at 3 AM:
>
> `FIRING: HighErrorRate — HTTP 5xx errors > 5% for service payment-service for 5 minutes`
>
> Walk me through your **complete incident investigation process** — what dashboards you check, what PromQL queries you run, how you correlate metrics with logs, and how you identify the root cause.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`, `Grafana`, and `ELK_Stack`

---

### Q37 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that:
> - Monitors a directory for any **new files** added
> - When a new file is detected, **logs the filename, size, and timestamp** to a log file
> - If the file size is **greater than 100MB**, sends an **alert** (print to stderr and log as ALERT)
> - Runs **continuously** and handles **SIGINT (Ctrl+C) gracefully** — cleaning up and printing a summary before exiting

📁 **Reference:** `nawab312/DSA` → `Linux` — Bash scripting, file monitoring, signals, and traps sections

---

### Q38 — Kubernetes | Conceptual | Advanced

> What is **etcd** in Kubernetes and what role does it play?
>
> If **etcd goes down** in a production cluster, what happens to:
> - **Running Pods**
> - **New deployments**
> - **Scaling operations**
> - **kubectl commands**
>
> How would you **backup and restore etcd**?

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md` and `09_CLUSTER_OPERATIONS.md`

---

### Q39 — Terraform + AWS | Scenario-Based | Advanced

> Your team manages **3 environments** (dev, staging, prod) on AWS using Terraform. A developer accidentally ran:
>
> ```bash
> terraform destroy
> ```
>
> in the **production environment** and it destroyed critical infrastructure including **RDS databases, EC2 instances, and S3 buckets**.
>
> **How would you:**
> 1. **Immediately respond** to minimize data loss
> 2. **Recover** the destroyed infrastructure and data
> 3. **Prevent this from ever happening again**

📁 **Reference:** `nawab312/Terraform` and `nawab312/AWS` — state management, RDS snapshots, S3 versioning, and Terraform safeguards sections

---

### Q40 — Kubernetes + Prometheus + Grafana | Scenario-Based | Advanced

> You are asked to set up **complete observability** for a new microservice being deployed to Kubernetes. The requirements are:
> - Expose **custom application metrics** from the microservice
> - Prometheus should **automatically discover and scrape** the metrics
> - Set up **alerting rules** for high error rate and high latency
> - Create a **Grafana dashboard** showing key RED metrics (Rate, Errors, Duration)
> - Alerts should notify a **Slack channel**
>
> Walk me through the **complete setup end-to-end**.

📁 **Reference:** `nawab312/Kubernetes` → `08_OBSERVABILITY.md`, `nawab312/Monitoring-and-Observability` → `Prometheus` and `Grafana`

---

### Q41 — AWS | Conceptual | Advanced

> Explain the difference between **AWS Lambda** and **AWS ECS (Elastic Container Service)**.
>
> Your team is debating whether to run a new microservice as a **Lambda function** or as a **container on ECS**.
>
> What questions would you ask to make this decision, and what are the **trade-offs** of each approach?

📁 **Reference:** `nawab312/AWS` — Lambda, ECS, serverless, and container orchestration sections

---

### Q42 — AWS + Kubernetes | Troubleshooting | Advanced

> Your application is running on **EKS**. Pods are running fine but they **cannot reach an RDS database** in a private subnet.
>
> A junior engineer says *"It was working yesterday, nothing changed"*.
>
> **Walk me through your complete troubleshooting process** — from the Pod all the way to RDS.

📁 **Reference:** `nawab312/AWS` — VPC, Security Groups, RDS sections and `nawab312/Kubernetes` → `03_NETWORKING.md`

---

### Q43 — Kubernetes | Conceptual | Advanced

> Explain the difference between **HPA (Horizontal Pod Autoscaler)**, **VPA (Vertical Pod Autoscaler)**, and **KEDA (Kubernetes Event Driven Autoscaler)**.
>
> When would you use each one? Can you use **HPA and VPA together**? What happens if you try?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — HPA, VPA, and autoscaling sections

---

### Q44 — Linux + Bash | Scenario-Based | Advanced

> You are an SRE and you receive an alert:
>
> `High number of CLOSE_WAIT connections on production server — current count: 850`
>
> The server is running a **Java web application**.
>
> **What is a `CLOSE_WAIT` connection, why does it happen, what is the risk, and how would you investigate and fix it?**

📁 **Reference:** `nawab312/DSA` → `Linux` — networking, TCP states, and socket management sections

---

### Q45 — CI-CD + Kubernetes + AWS | System Design | Advanced

> You are joining a startup as their first **DevOps/SRE engineer**. They have a monolithic application running on a single EC2 instance with no CI/CD, no monitoring, and manual deployments via SSH.
>
> The CTO asks you to **design and implement a complete DevOps platform** from scratch. The company has 10 developers, uses GitHub, and wants to deploy to AWS.
>
> **What would you build, in what order, and why?**

📁 **Reference:** All repositories — this is a capstone question covering all topics

---

## Key Topics Covered

| Topic | Questions |
|---|---|
| Kubernetes | Q1, Q2, Q3, Q4, Q16, Q19, Q21, Q26, Q30, Q33, Q38, Q40, Q43 |
| AWS | Q7, Q18, Q22, Q29, Q35, Q39, Q41, Q42 |
| Terraform | Q5, Q17, Q25, Q32, Q39 |
| Prometheus | Q6, Q23, Q36, Q40 |
| Grafana | Q14, Q36, Q40 |
| ELK Stack | Q12, Q28 |
| Linux / Bash | Q8, Q20, Q37, Q44 |
| ArgoCD | Q9, Q24 |
| Jenkins / CI-CD | Q10, Q34 |
| GitHub Actions | Q13, Q31 |
| Git | Q11, Q27 |
| Python | Q15 |
| System Design | Q45 |

---

*Generated from interview session — nawab312 GitHub repositories*
