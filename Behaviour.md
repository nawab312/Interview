# Interview Preparation — STAR Format Q&A
**Siddharth Singh | DevOps / SRE Engineer | 5 Years Experience**

---

> **STAR Format Key**
> - 🟢 **SITUATION** — Context and background
> - 🔵 **TASK** — Your specific responsibility or goal
> - 🟡 **ACTION** — Concrete steps you took
> - 🔴 **RESULT** — Measurable outcomes and impact

"I'm Siddharth, a DevOps and SRE engineer with around 5 years of experience, based out of Bengaluru.
I've spent most of my career working on AWS-based infrastructure — primarily around Kubernetes on EKS, Terraform for infrastructure as code, and CI/CD systems using Jenkins and ArgoCD.
My last two roles gave me very different but complementary experiences. At Infosys, I was working on core banking microservices — so the focus was heavily on stability, security, and PCI-DSS compliance. That's where I built my foundation in observability, incident management, and database reliability.
At Coffeebeans, I moved into e-commerce infrastructure — which meant dealing with flash sales, unpredictable traffic spikes, and performance at scale. That's where I got hands-on with things like event-driven autoscaling with KEDA, canary deployments, Redis caching, and cost optimization.
On the SRE side, I care a lot about the full loop — not just keeping things running, but reducing MTTR, improving alert quality, writing good RCAs, and making sure the same incident doesn't happen twice.
Outside of incident response, I spend time on proactive work — tuning infrastructure, improving runbooks, and automating away repetitive ops tasks."

---

## 🏗️ Infrastructure & Day-to-Day

### Q1. Walk me through what your day-to-day looks like as a DevOps/SRE engineer.

🟢 **SITUATION**
I was working across two organizations managing cloud-native infrastructure for production-grade workloads — core banking at Infosys and an e-commerce platform at Coffeebeans — both requiring high availability and fast incident response.

🔵 **TASK**
My responsibility spanned three areas: keeping infrastructure healthy and scalable, ensuring reliable deployments, and proactively improving system observability and reliability.

🟡 **ACTION**
On a typical day I covered three pillars:

- **Infrastructure** — Reviewed EKS node group health, managed scaling configs via Cluster Autoscaler and KEDA, and ensured all changes went through Terraform. Planned and rolled out capacity changes or new services when needed.
- **Deployments** — Monitored active ArgoCD rollouts, watched error rate dashboards in Grafana and CloudWatch during canary releases, investigated anomalies, and triggered rollbacks if needed. Also watched Jenkins pipelines for failures or flaky stages.
- **Reliability** — Watched Prometheus and Grafana dashboards for p99 latency, pod crash loops, ALB 5xx errors, and SQS queue lag. Was on PagerDuty on-call rotation, led incident investigations, wrote RCAs, and pushed through fixes — whether it was a DB connection issue, an autoscaling lag during traffic spikes, or a caching problem.

Beyond reactive work, I spent time each week on proactive improvements — tuning alert thresholds, writing automation scripts, reviewing cost reports for rightsizing opportunities, and improving runbooks.

🔴 **RESULT**
This structured approach helped reduce MTTD from 15 minutes to under 4 minutes at Coffeebeans and from 12 minutes to under 3 minutes at Infosys, while maintaining 99.95% uptime for critical payment workloads.

---

## 🗄️ RDS & Databases

### Q2. What all have you worked on in RDS, and can you walk me through a major incident?

🟢 **SITUATION**
At Infosys, I was responsible for the RDS instances backing core banking applications — payments were going through it, so any degradation directly impacted revenue. During peak hours one day, the database started choking — latency hit 4 seconds and payments began failing.

🔵 **TASK**
I needed to immediately stabilize the database and then implement a permanent fix to prevent the connection exhaustion from recurring.

🟡 **ACTION**
I diagnosed the issue — too many services were opening raw DB connections simultaneously without any pooling. I implemented pgBouncer as a connection pooling layer between the application and RDS. pgBouncer managed connection reuse efficiently, preventing the DB from being overwhelmed during traffic spikes. I also set up proper monitoring afterward — CloudWatch alarms for connection count, CPU, and storage, with Grafana dashboards pulling RDS metrics via the CloudWatch data source plugin.

On the security and compliance side, I ensured no application had hardcoded DB credentials — everything went through proper IAM roles. I also automated backup validation: a Lambda function triggered nightly by EventBridge restored the latest RDS snapshot to a temporary instance, ran basic SQL queries to confirm readability, then deleted the instance. Any failure sent an SNS alert to Slack.

🔴 **RESULT**
After pgBouncer was deployed, latency dropped from 4 seconds back to under 300ms and everything stabilized. The automated backup validation script gave confidence that backups were actually restorable — not just existing on paper.

---

### Q3. What is your day-to-day with RDS monitoring?

🟢 **SITUATION**
At both Infosys and Coffeebeans, RDS was a critical dependency for multiple services. Without proactive daily monitoring, issues like replication lag or slow queries would escalate into incidents before anyone noticed.

🔵 **TASK**
My responsibility was to ensure RDS health was continuously visible, anomalies were caught early, and any maintenance activities were handled safely through proper processes.

🟡 **ACTION**
Every morning I checked Grafana dashboards tracking: active connections (`DatabaseConnections`), query latency (`ReadLatency` / `WriteLatency`), replication lag on read replicas, and slow query count via Performance Insights. I had CloudWatch alarms set for high CPU, storage approaching limits, and connection count crossing threshold — these were independent of Grafana so they'd still fire even if Grafana went down.

For replication, I checked lag daily — increasing lag usually meant the primary was getting too much write traffic, signaling a need for query optimization or scaling. I also reviewed RDS Proxy connection pool metrics to confirm connections were being reused efficiently and the pool wasn't getting exhausted.

On the maintenance side, I handled parameter group changes, backup schedule validation, snapshot retention checks, and SSL certificate expiry monitoring via a Lambda+EventBridge script that called `describe-certificates` and alerted via SNS if a certificate was within 30 days of expiry. Any infra change — instance resize, adding a read replica, parameter group modification — went through Terraform with a PR review before applying.

🔴 **RESULT**
This daily monitoring routine meant issues were caught before they became incidents. On multiple occasions, spotting replication lag early allowed us to address query inefficiencies before they caused a production degradation.

---

### Q4. Why did you use CloudWatch alarms separately when Grafana already has alerting?

🟢 **SITUATION**
At Coffeebeans, we had Grafana dashboards pulling RDS metrics via the CloudWatch data source. A team member questioned why we needed separate CloudWatch alarms when Grafana could alert on the same metrics.

🔵 **TASK**
I needed to make a justified architectural decision about alerting redundancy for a production database backing an e-commerce platform.

🟡 **ACTION**
I explained the key distinction: Grafana alerts depend on Grafana being up. If the Grafana pod crashes or the data source connection fails, those alerts won't fire. CloudWatch alarms are native AWS — they operate completely independently of your monitoring stack. I set up CloudWatch alarms for the most critical RDS metrics (connection count, CPU, storage) with SNS notifications to PagerDuty, while Grafana provided the rich dashboards and visualization layer for investigation.

🔴 **RESULT**
This two-layer approach gave us alerting resilience. Grafana was the investigation and visualization tool; CloudWatch was the independent safety net that would still fire even if our entire observability stack had issues. For a production database handling payments, that independence was non-negotiable.

---

### Q5. How did you handle the Aurora connection saturation incident at Coffeebeans?

🟢 **SITUATION**
At Coffeebeans, we were running an e-commerce platform with unpredictable flash sale traffic. Despite having Multi-AZ Aurora PostgreSQL, read replicas, and RDS Proxy planned, during a peak traffic event Aurora connections saturated — DB response time hit 3.8 seconds, causing checkout failures.

🔵 **TASK**
As the on-call engineer, I needed to stabilize the database immediately, reduce response time to acceptable levels, and prevent recurrence through architectural improvements.

🟡 **ACTION**
Because RDS Proxy was already in our plan, I fast-tracked its deployment in front of Aurora to handle connection pooling. Simultaneously, I rerouted heavy read traffic — product catalog queries — to the read replica, reducing load on the primary. I coordinated the changes to happen without requiring a maintenance window. After stabilization, I built comprehensive Grafana dashboards tracking connection count, replication lag, and query latency to catch this pattern early in future.

🔴 **RESULT**
DB response time recovered from 3.8 seconds to 400ms within 35 minutes. The RDS Proxy and read replica routing became the permanent architecture. The Grafana dashboards gave the team early warning visibility, shifting us from reactive to proactive on database health.

---

### Q6. You mentioned validating SSL certificates for RDS — how exactly did you handle that?

🟢 **SITUATION**
RDS uses SSL certificates to encrypt connections between applications and the database. AWS manages these certificates, but it's the operator's responsibility to ensure applications are using the current certificate and that it hasn't expired — an expired certificate can break all DB connections across all services.

🔵 **TASK**
I needed to build a reliable, automated process to detect certificate expiry well in advance and rotate certificates with zero downtime.

🟡 **ACTION**
I implemented a two-layer approach. First, I configured AWS Health Dashboard alerts forwarded to our Slack channel via EventBridge + SNS so the team received AWS's own notifications automatically. Second, I wrote a Lambda function triggered by EventBridge on a schedule that called `aws rds describe-certificates`, checked the expiry date of the certificate in use, and sent an SNS alert if expiry was within 30 days.

When rotation was needed, the process was: download the new certificate bundle from AWS → update application connection strings to trust the new certificate → rotate the certificate on the RDS instance using `modify-db-instance` (zero downtime) → verify connections post-rotation.

🔴 **RESULT**
We never had a certificate expiry incident. The automated detection gave a 30-day advance warning window, which was more than enough time to plan and execute a zero-downtime rotation across all services.

---

## ☸️ EKS & Kubernetes

### Q7. How many EKS clusters were you managing and how were they structured?

🟢 **SITUATION**
At Coffeebeans, we needed separate environments for development, QA validation, and production, each with different reliability and access requirements — especially production, which served live customer traffic.

🔵 **TASK**
I was responsible for provisioning, managing, and evolving all three EKS clusters, with production being the highest-priority environment requiring multi-AZ resilience and strict IAM controls.

🟡 **ACTION**
I provisioned three clusters via a modularized Terraform codebase with separate variable files per environment:

- **Dev cluster** — For developer testing. Jenkins ran here for CI builds and dynamic agent pods.
- **Staging cluster** — For QA and pre-production validation. ArgoCD deployed here for staging sync.
- **Production cluster** — Multi-AZ across 3 availability zones, with separate On-Demand and Spot node groups, strict IAM policies, and no direct developer access.

All three shared the same Terraform module structure — only variable files differed — ensuring consistency and making environment parity easy to maintain.

🔴 **RESULT**
The modularized Terraform approach meant spinning up or modifying a cluster was predictable and auditable. Production's multi-AZ setup validated sub-5-minute pod rescheduling during resilience drills simulating AZ failures.

---

### Q8. Walk me through your EKS node group design and why you chose those instance types.

🟢 **SITUATION**
At Coffeebeans, the production EKS cluster needed to serve multiple workload types — critical customer-facing microservices, cluster system components, and async batch jobs — each with different resource profiles, reliability requirements, and cost tolerances.

🔵 **TASK**
I needed to design a node group architecture that isolated workloads appropriately, balanced cost and performance, and could scale safely under load.

🟡 **ACTION**
I designed three node groups with taints to enforce workload isolation:

**1. System Node Group** — Dedicated to cluster system components (CoreDNS, kube-proxy, Cluster Autoscaler, KEDA, AWS Load Balancer Controller).
- Instance: `t3.medium`, On-Demand
- Min: 2 / Max: 4
- Taint: `dedicated=system:NoSchedule`

**2. Application Node Group** — For all customer-facing microservices (order processing, checkout, product catalog).
- Instance: `m5.xlarge` (4 vCPU, 16GB), On-Demand
- Min: 3 (one per AZ) / Max: 12
- On-Demand because customer-facing workloads can't tolerate Spot interruptions

For `m5.xlarge` specifically: our Spring Boot pods had requests of ~500m CPU and ~1GB memory. On an `m5.xlarge`, CPU was the bottleneck at ~6–7 pods per node — a healthy density. We initially started on `m5.large`, monitored utilization in Grafana for 2 weeks, saw nodes consistently at 70–80% CPU, and rightsized up to `m5.xlarge`. Utilization then stabilized at 50–60%, which is a healthy range. `m5.2xlarge` would have been more than needed and increased blast radius.

**3. Batch Node Group** — For async workloads: SQS consumers, report generation, background jobs.
- Instance: `m5.large`, Spot
- Min: 0 / Max: 10
- Taint: `workload=batch:NoSchedule`
- Spot is safe here because if a node is interrupted, the SQS consumer simply picks up the message again — no customer impact.

🔴 **RESULT**
The three-group design eliminated resource contention between system components and application pods. Spot instances on the batch node group reduced EC2 costs meaningfully for non-critical workloads. Node density on `m5.xlarge` was validated — losing one node meant rescheduling 6–7 pods, not 20+, keeping blast radius manageable.

---

## 🔧 Jenkins & CI/CD

### Q9. How was Jenkins set up and why did you run it inside EKS instead of on a standalone EC2?

🟢 **SITUATION**
At Coffeebeans, we needed a CI/CD system that was reliable, scalable, and consistent with how we managed everything else — through Kubernetes and Terraform. Running Jenkins on a standalone EC2 would mean separate OS patching, manual upgrades, no auto-healing, and a single point of failure.

🔵 **TASK**
I needed to deploy and operate Jenkins in a way that was self-healing, scalable, and managed as code — not as a manually maintained server.

🟡 **ACTION**
I deployed Jenkins Master as a pod inside EKS using Helm in its own `cicd` namespace. This gave us: automatic pod restart on crash, resource limits enforced via Kubernetes, and the entire deployment managed via Helm values in Terraform.

For agents, I used the Jenkins Kubernetes Plugin — instead of static agent nodes sitting idle, Jenkins spun up a fresh pod for each build, ran the pipeline, and deleted it automatically. This meant agents scaled on demand and never wasted resources at idle.

For persistence, I configured an EBS-backed PVC (`gp3`, 50Gi) so Jenkins Master retained all job history, configs, and credentials across pod restarts.

Jenkins ran only on the Dev/Staging cluster — not production. There was no reason to consume production resources with a CI tool that serves no customer traffic. The deployment flow was:

```
Developer pushes code
        ↓
Jenkins on Dev cluster picks it up
        ↓
Builds image → runs tests → scans with Trivy → pushes to ECR
        ↓
Updates image tag in Git
        ↓
ArgoCD on Staging detects Git change → deploys to Staging
        ↓
After approval → ArgoCD on Production deploys to Production
```

🔴 **RESULT**
Jenkins pod restarts were handled automatically with zero data loss thanks to the EBS PVC. Dynamic agent pods eliminated idle resource waste. The GitOps handoff between Jenkins and ArgoCD gave a clear, auditable separation between CI (build, test, scan) and CD (deploy), with Git as the single source of truth.

---

## 🔐 IAM & Account Structure

### Q10. Walk me through your AWS account and IAM structure.

🟢 **SITUATION**
At Coffeebeans, a single AWS account for all environments created risks — a misconfigured IAM policy in development could potentially affect production resources, and there was no clear blast radius boundary between environments.

🔵 **TASK**
I needed to design an account and IAM structure that enforced environment isolation, applied least-privilege per environment, and centralized security and compliance tooling.

🟡 **ACTION**
I implemented a three-account structure:

| Account | Resources |
|---|---|
| **Dev/Staging** | Dev EKS cluster, Staging EKS cluster, Jenkins, ECR registry |
| **Production** | Production EKS cluster, RDS Aurora, Redis, SQS — strictest IAM policies, no direct developer access |
| **Security/Shared** | GuardDuty centrally enabled, CloudTrail logs, Terraform state S3 bucket |

In production, all EKS workloads used IRSA (IAM Roles for Service Accounts) — each pod's ServiceAccount was bound to a specific IAM role with only the permissions that service required. No static credentials existed anywhere in the cluster. Developers had no direct access to production resources — all changes went through ArgoCD syncing from Git.

🔴 **RESULT**
Strict account separation meant a compromised dev credential had zero impact on production. IRSA eliminated static credential exposure. Centralizing GuardDuty and CloudTrail in the shared account gave unified security visibility across all environments. The setup passed internal security audits with zero critical findings.

---

*Document generated from interview preparation notes | Siddharth Singh*
