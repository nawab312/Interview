# DevOps / SRE / Cloud Interview Questions (271–315)

> Part 7 of interview preparation questions based on your GitHub repositories.
> All questions are unique — not repeated from Q1–Q270.

---

## Table of Contents

| # | Topic | Type | Difficulty |
|---|---|---|---|
| 271 | Kubernetes | Conceptual | Advanced |
| 272 | AWS | Scenario-Based | Advanced |
| 273 | Linux / Bash | Conceptual | Advanced |
| 274 | Terraform | Conceptual | Advanced |
| 275 | Prometheus | Scenario-Based | Advanced |
| 276 | Kubernetes | Troubleshooting | Advanced |
| 277 | Git | Conceptual | Advanced |
| 278 | AWS | Conceptual | Advanced |
| 279 | Jenkins | Scenario-Based | Advanced |
| 280 | ELK Stack | Conceptual | Advanced |
| 281 | Kubernetes | Scenario-Based | Advanced |
| 282 | Linux / Bash | Troubleshooting | Advanced |
| 283 | Terraform | Scenario-Based | Advanced |
| 284 | GitHub Actions | Scenario-Based | Advanced |
| 285 | AWS | Troubleshooting | Advanced |
| 286 | Prometheus | Conceptual | Advanced |
| 287 | Kubernetes | Conceptual | Advanced |
| 288 | ArgoCD | Conceptual | Advanced |
| 289 | Linux / Bash | Scenario-Based | Advanced |
| 290 | AWS | Scenario-Based | Advanced |
| 291 | Kubernetes | Troubleshooting | Advanced |
| 292 | Grafana | Conceptual | Advanced |
| 293 | Terraform | Troubleshooting | Advanced |
| 294 | Python | Scenario-Based | Advanced |
| 295 | AWS | Conceptual | Advanced |
| 296 | Kubernetes | Scenario-Based | Advanced |
| 297 | ELK Stack | Troubleshooting | Advanced |
| 298 | Jenkins | Conceptual | Advanced |
| 299 | Linux / Bash | Conceptual | Advanced |
| 300 | AWS | Scenario-Based | Advanced |
| 301 | Kubernetes | Conceptual | Advanced |
| 302 | GitHub Actions | Troubleshooting | Advanced |
| 303 | Terraform | Conceptual | Advanced |
| 304 | Prometheus | Troubleshooting | Advanced |
| 305 | AWS | Conceptual | Advanced |
| 306 | Kubernetes | Scenario-Based | Advanced |
| 307 | ArgoCD | Troubleshooting | Advanced |
| 308 | Linux / Bash | Scenario-Based | Advanced |
| 309 | AWS | Scenario-Based | Advanced |
| 310 | Kubernetes | Conceptual | Advanced |
| 311 | Grafana | Scenario-Based | Advanced |
| 312 | Terraform | Scenario-Based | Advanced |
| 313 | Git | Scenario-Based | Advanced |
| 314 | AWS + Kubernetes | Scenario-Based | Advanced |
| 315 | All Topics | System Design | Advanced |

---

## Questions

---

### Q271 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `subresources`** — what are they and which built-in subresources exist (e.g., `/status`, `/scale`, `/exec`, `/log`)?
>
> Why is it important to separate the `/status` subresource from the main resource? How do you use RBAC to allow a user to **read pod logs** but not exec into pods? Write the exact Role manifest.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md` and `10_CRDs_EXTENSIBILITY.md` — subresources, RBAC fine-grained access sections

---

### Q272 — AWS | Scenario-Based | Advanced

> Your company wants to implement **immutable infrastructure** on AWS — every deployment creates new EC2 instances from a fresh AMI rather than updating existing ones.
>
> Design the complete workflow covering:
> - AMI baking with **Packer**
> - AMI testing before promotion
> - **Blue/green ASG swap** using Launch Templates
> - Draining old instances gracefully
> - Rollback strategy if new AMI is faulty

📁 **Reference:** `nawab312/AWS` — AMI, Launch Templates, ASG, blue/green sections

---

### Q273 — Linux / Bash | Conceptual | Advanced

> Explain **Linux `namespaces`** specifically from a **networking perspective**. What is a **network namespace** and how does it isolate network interfaces, routing tables, and iptables rules?
>
> How does `ip netns` work? Walk through how a **Docker container gets its own network namespace** and how the `veth pair` connects it to the host — with exact commands showing the plumbing.

📁 **Reference:** `nawab312/DSA` → `Linux` and `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md` — network namespaces, veth pairs, container networking sections

---

### Q274 — Terraform | Conceptual | Advanced

> Explain **Terraform `backends`** in depth. What is the difference between a **local backend** and a **remote backend**?
>
> List at least **5 different backend types** with their use cases. What is **backend partial configuration** and why is it useful for keeping sensitive backend config (like S3 bucket names) out of version control?

📁 **Reference:** `nawab312/Terraform` — backends, remote state, partial configuration sections

#### Key Points to Cover:
```
A backend in Terraform defines two things:
  Where state is stored — the `terraform.tfstate` file that tracks real-world resource mappings
  How operations are executed — locally (on your machine) or remotely (on a server)

Every Terraform configuration has a backend.
If you don't declare one, Terraform silently uses the `local` backend.

Local backend:
  → State stored at ./terraform.tfstate (or custom path)
  → No state locking — concurrent runs corrupt state
  → Operations run on the local machine
  → No collaboration — each developer has their own state
  → Acceptable for: learning, solo projects, CI with ephemeral runners
 
Remote backend (general):
  → State stored in shared external system
  → Locking prevents concurrent writes (DynamoDB, GCS native, etc.)
  → Some backends (Terraform Cloud) also run operations remotely
  → Enables team collaboration and auditability
  → Required for: any production or team workflow
 
State locking:
  → Prevents two engineers running apply simultaneously
  → DynamoDB used as lock table for S3 backend
  → Lock is acquired at plan, released after apply or on explicit unlock
  → terraform force-unlock <lock-id> used when lock is stuck
 
State file sensitivity:
  → Contains resource IDs, IPs, sometimes plaintext secrets
  → Must be encrypted at rest (S3 SSE, GCS CMEK)
  → Must have restricted IAM/ACL access
  → Never commit to version control
 
Backend operations (enhanced backends):
  → Terraform Cloud / Enterprise can execute plan+apply remotely
  → Standard backends (S3, GCS) only store state; operations remain local
  → Remote operations enable policy checks (Sentinel), notifications, audit logs

Backend Types and Their Use Cases
  local
     → Default if no backend block declared
     → State at ./terraform.tfstate or custom path
     → Use case: local dev, learning, throwaway environments
     → No locking, no encryption, no sharing
  s3 (AWS)
     → State in S3 bucket; locking via DynamoDB table
     → Supports SSE-S3, SSE-KMS encryption
     → Use case: AWS-native teams, most common production choice
     → Fine-grained IAM for access control per bucket/prefix
  terraform cloud / terraform enterprise
     → Enhanced backend: stores state AND runs operations remotely
     → Includes Sentinel policy engine, SSO, audit logging, team RBAC
     → Use case: enterprises needing governance, approval workflows, cost estimation
     → workspace-based isolation; supports VCS-driven runs

Honorable mentions:
   http    → any REST endpoint; useful for custom state servers
   consul  → HashiCorp Consul KV; locking built-in; used in Nomad-heavy shops
   pg      → PostgreSQL table; good for teams already running Postgres

Backend Partial Configuration
Terraform requires backend config at `init` time — before variables are resolved.
This means you cannot use `var.*` or `local.*` inside a `backend {}` block.
Partial configuration solves this by letting you omit
sensitive or environment-specific values from the checked-in code and supply them at init time instead.

How it works:
  → Commit a backend block with only non-sensitive structural config
  → Omit: bucket names, key paths, region, DynamoDB table, role ARNs
  → Supply the rest via one of three mechanisms at init time:
      -backend-config="bucket=my-secret-bucket"   (CLI flag)
      -backend-config=backend.hcl                 (separate HCL file, gitignored)
      Environment variables (backend-specific, e.g. AWS_DEFAULT_REGION)

Example — what lives in version control (main.tf):
  terraform {
    backend "s3" {
      # structural only — no bucket, no key, no region
      encrypt        = true
      dynamodb_table = ""   # supplied at init
    }
  }

Example — what lives outside version control (backend.hcl, in .gitignore):
  bucket         = "my-company-tfstate-prod"
  key            = "services/api/terraform.tfstate"
  region         = "us-east-1"
  dynamodb_table = "terraform-lock-table"
  role_arn       = "arn:aws:iam::123456789:role/TerraformStateRole"

Init command:
  terraform init -backend-config=backend.hcl

Why this matters:
  → Bucket names reveal infrastructure topology — treat as sensitive
  → Role ARNs and account IDs should not be public
  → Different environments (dev/staging/prod) use different buckets;
    partial config lets one codebase serve all environments
  → CI/CD pipelines inject backend config from secrets manager at runtime
  → Prevents accidental state cross-contamination between environments

> 💡 **Interview tip:** Most candidates describe backends as just "where state is stored." Stand out by explaining **state locking mechanics** (DynamoDB as a lock table, what happens on a stuck lock), the **distinction between standard and enhanced backends**, and why you **cannot use variables inside a backend block** — that last point is the root reason partial configuration exists and shows you've actually hit this constraint in practice.
 
> 💡 **Interview tip:** When asked about keeping sensitive config out of version control, don't just say "use environment variables." Walk through the **three partial configuration methods** (CLI flag, `-backend-config` HCL file, env vars), explain that the HCL file goes in `.gitignore`, and mention that CI/CD pipelines typically pull this file from a secrets manager (AWS Secrets Manager, Vault, GitHub Actions secrets) and write it to disk at init time. This shows production-grade thinking.
 
> 💡 **Interview tip:** Know the failure modes. What happens if two engineers run `apply` simultaneously against an S3 backend **without** a DynamoDB lock table? *(State corruption — last write wins, one apply's changes are silently overwritten.)* What does `terraform force-unlock` do and when is it dangerous? *(Releases a stuck lock — dangerous if another process is genuinely mid-apply, not just crashed.)* These scenarios separate candidates who've used Terraform in production from those who've only followed tutorials.

```

---

### Q275 — Prometheus | Scenario-Based | Advanced

> You need to implement **Prometheus monitoring for Redis** running as a Kubernetes StatefulSet. The monitoring should cover:
> - Memory usage vs `maxmemory` limit
> - Cache hit rate (`keyspace_hits` / total commands)
> - Connected clients vs `maxclients`
> - Replication lag for replicas
> - Evicted keys rate (indicates memory pressure)
>
> Write the complete setup — exporter, ServiceMonitor, recording rules, and 3 alerting rules.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — redis_exporter, cache monitoring sections

---

### Q276 — Kubernetes | Troubleshooting | Advanced

> Your Kubernetes cluster is experiencing **`Evicted` pods accumulating** — hundreds of evicted pods are showing up across all namespaces and cluttering `kubectl get pods` output. New evictions are happening every few minutes.
>
> What causes pod eviction accumulation? How do you identify the root cause of why pods are being evicted? How do you **bulk clean up** evicted pods? How do you prevent this from happening again?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` and `14_TROUBLESHOOTING.md` — eviction, pod cleanup sections

---

### Q277 — Git | Conceptual | Advanced

> Explain **`git rerere`** (Reuse Recorded Resolution). What problem does it solve and when is it most useful?
>
> Also explain **`git notes`** — what they are, how they differ from commit messages, and give a DevOps use case where git notes would be valuable (e.g., attaching CI build results to commits without modifying commit history).

📁 **Reference:** `nawab312/CI_CD` → `Git` — advanced Git features, rerere, notes sections

---

### Q278 — AWS | Conceptual | Advanced

> Explain **AWS `Savings Plans`** vs **`Reserved Instances`** vs **`Spot Instances`** vs **`On-Demand`**.
>
> What are **Compute Savings Plans** vs **EC2 Instance Savings Plans**? How do you decide the right mix of purchasing options for a production workload? What tools does AWS provide to recommend the optimal purchasing strategy?

📁 **Reference:** `nawab312/AWS` — cost optimization, Savings Plans, Reserved Instances sections

---

### Q279 — Jenkins | Scenario-Based | Advanced

> You need to implement a **Jenkins pipeline** that manages **feature flags** as part of the deployment process:
> - Before deploying, **enables the feature flag** for 1% of users (canary)
> - Monitors **error rate** for 10 minutes via Prometheus API
> - If error rate is acceptable → **gradually increases** flag to 10%, 50%, 100%
> - If error rate spikes → **disables the flag** and rolls back deployment
> - All flag changes are **logged with who triggered** the pipeline
>
> Write the Jenkinsfile structure for this progressive rollout.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — progressive delivery, feature flags, Prometheus integration sections

---

### Q280 — ELK Stack | Conceptual | Advanced

> Explain **Elasticsearch `aggregations`** in depth. What are the 4 main categories of aggregations?
>
> Write Elasticsearch aggregation queries that answer:
> - What are the **top 5 slowest API endpoints** by average response time in the last hour?
> - What is the **error rate per service** over the last 24 hours broken into 1-hour buckets?
> - What is the **geographic distribution** of users by country?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — aggregations, metric aggregations, bucket aggregations sections

---

### Q281 — Kubernetes | Scenario-Based | Advanced

> You are implementing **Kubernetes `NetworkPolicy`** for a microservice that uses **gRPC** for inter-service communication. The services communicate on port 50051.
>
> Write NetworkPolicies that:
> - Allow `service-a` to call `service-b` on port 50051
> - Allow `service-b` to call `service-c` on port 50051
> - Block all other inter-service traffic
> - Allow Prometheus to scrape metrics from all services on port 9090
> - Explain why gRPC over HTTP/2 needs special consideration vs HTTP/1.1

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — NetworkPolicy, gRPC, port-based policies sections

---

### Q282 — Linux / Bash | Troubleshooting | Advanced

> A production server's **NTP (Network Time Protocol)** is out of sync by 45 seconds. This is causing:
> - TLS certificate validation failures
> - JWT token validation errors
> - Distributed tracing timestamps being wrong
>
> Walk me through diagnosing the NTP issue, fixing it, and what monitoring you would set up to detect clock drift before it becomes a problem.

📁 **Reference:** `nawab312/DSA` → `Linux` — NTP, chrony, timedatectl, clock sync sections

---

### Q283 — Terraform | Scenario-Based | Advanced

> You need to implement **Terraform `policy as code`** using **OPA (Open Policy Agent)** or **Sentinel** to enforce infrastructure standards before `terraform apply`:
> - All S3 buckets must have encryption enabled
> - All EC2 instances must have the `CostCenter` tag
> - No security groups can have `0.0.0.0/0` on port 22
> - RDS instances must have `deletion_protection = true`
>
> Show how to implement this using `conftest` with OPA policies in a CI/CD pipeline.

📁 **Reference:** `nawab312/Terraform` and `nawab312/CI_CD` — policy as code, OPA, Conftest, Terraform compliance sections

---

### Q284 — GitHub Actions | Scenario-Based | Advanced

> You need to implement a **GitHub Actions workflow** for a **monorepo** with 5 services (frontend, backend, auth, payments, notifications). The workflow should:
> - Detect **which services changed** using path filters
> - Only run tests and build for **changed services**
> - Run changed service pipelines **in parallel**
> - Only deploy a service if its **tests pass AND it has changed**
> - Generate a **deployment summary** comment on the PR showing which services were tested/deployed
>
> Write the complete workflow using `paths-filter` and dynamic matrix.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — monorepo, path filters, dynamic matrix, PR comments sections

---

### Q285 — AWS | Troubleshooting | Advanced

> Your **AWS CloudFormation stack** is stuck in `UPDATE_ROLLBACK_FAILED` state. You cannot update it, delete it, or roll it back. The stack has critical production resources.
>
> What causes `UPDATE_ROLLBACK_FAILED` state? Walk me through the options to **safely recover** the stack without losing the production resources.

📁 **Reference:** `nawab312/AWS` — CloudFormation, stack recovery, continue rollback sections

---

### Q286 — Prometheus | Conceptual | Advanced

> Explain **Prometheus `exemplars`**. What are they, how do they work, and what OpenTelemetry standard do they follow?
>
> How do exemplars create a **link between a high-latency metric and a specific trace ID**? What changes are needed in application instrumentation, Prometheus configuration, and Grafana to use exemplars end-to-end?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — exemplars, OpenTelemetry, tracing integration sections

---

### Q287 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `LimitRange`** interaction with **`ResourceQuota`** in edge cases.
>
> What happens when:
> - A Pod requests more than the LimitRange `max` allows?
> - A namespace has a ResourceQuota but no LimitRange — can unlimited pods be created?
> - The LimitRange default is higher than the ResourceQuota hard limit?
>
> Also explain **`scopeSelector`** in ResourceQuota — how do you apply different quotas to `BestEffort` vs `NotBestEffort` pods?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — LimitRange, ResourceQuota edge cases sections

---

### Q288 — ArgoCD | Conceptual | Advanced

> Explain **ArgoCD `resource health`** assessment. How does ArgoCD determine if a resource is `Healthy`, `Progressing`, `Degraded`, or `Missing`?
>
> How do you write a **custom health check** in Lua for a CRD that ArgoCD does not natively understand? Give an example for a custom `DatabaseCluster` CRD that is healthy when `status.phase == "Running"` and `status.readyReplicas == spec.replicas`.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — resource health, custom health checks, Lua sections

---

### Q289 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that implements a **certificate expiry monitor**:
> - Reads a list of domains from a file (`domains.txt`)
> - Checks the SSL/TLS certificate expiry date for each domain
> - Checks both **HTTPS** (port 443) and optionally custom ports
> - Alerts (to Slack via webhook) if any certificate expires within **30 days**
> - Alerts **critically** if any expires within **7 days**
> - Generates a sorted report showing days until expiry for all domains
> - Runs via cron and skips domains that are unreachable

📁 **Reference:** `nawab312/DSA` → `Linux` — openssl, bash scripting, certificate management sections

---

### Q290 — AWS | Scenario-Based | Advanced

> You are implementing **AWS Lake Formation** for a data lake on S3. Your company has data from multiple sources that needs to be:
> - Catalogued in **AWS Glue Data Catalog**
> - Accessed by different teams with **column-level security** (some teams cannot see PII columns)
> - Queried efficiently using **Athena**
> - Partitioned by date for query performance
>
> Design the complete data lake architecture with access controls.

📁 **Reference:** `nawab312/AWS` — Lake Formation, Glue, Athena, S3 data lake sections

---

### Q291 — Kubernetes | Troubleshooting | Advanced

> Your Kubernetes `kube-scheduler` is logging `0/10 nodes are available: 10 node(s) had taint {node.kubernetes.io/not-ready: }, that the pod didn't tolerate` but ALL nodes show as `Ready` in `kubectl get nodes`.
>
> Why would the scheduler see nodes as tainted with `not-ready` when they appear Ready? What are the timing and race condition issues between node readiness and scheduler decisions?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` and `14_TROUBLESHOOTING.md` — scheduler, node taints, timing sections

---

### Q292 — Grafana | Conceptual | Advanced

> Explain **Grafana `Transformations`** — what are they and how do they work in the visualization pipeline?
>
> Give examples of transformations that solve these real-world problems:
> - Joining two queries (CPU from one source, Memory from another) into a single table
> - Converting a wide format time series to a long format
> - Filtering rows where value exceeds a threshold
> - Renaming fields returned from Prometheus to human-readable names
> - Calculating a new field (CPU usage %) from raw values

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — transformations, data manipulation sections

---

### Q293 — Terraform | Troubleshooting | Advanced

> Your `terraform init` is **failing** with:
>
> `Error: Failed to install provider: the local package for registry.terraform.io/hashicorp/aws doesn't match any of the checksums previously recorded in the dependency lock file`
>
> And separately another error:
>
> `Error: Could not retrieve the list of available versions for provider hashicorp/aws: could not connect to registry.terraform.io`
>
> Diagnose and fix both errors independently.

📁 **Reference:** `nawab312/Terraform` — provider installation, lock file, registry connectivity sections

---

### Q294 — Python | Scenario-Based | Advanced

> Write a **Python script** that implements a **Kubernetes cost analyzer**:
> - Connects to Kubernetes API and lists all Deployments across all namespaces
> - For each deployment calculates: requested CPU cores, requested memory GB, number of replicas
> - Estimates **monthly cost** based on configurable AWS instance pricing per CPU/GB
> - Identifies **over-provisioned** deployments (requested >> actual usage from Metrics API)
> - Generates a report sorted by estimated cost (highest first)
> - Highlights deployments with **>50% over-provisioning** as optimization candidates

📁 **Reference:** `nawab312/DSA` → `Python` — Kubernetes client, metrics API, cost calculation sections

---

### Q295 — AWS | Conceptual | Advanced

> Explain **AWS `Macie`** and **AWS `Inspector`** — what do they scan for and how do they differ?
>
> Also explain **AWS `Security Hub`** — how does it aggregate findings from GuardDuty, Macie, Inspector, and Config into a single view? What is the **AWS Security Finding Format (ASFF)** and how do you integrate custom security tools into Security Hub?

📁 **Reference:** `nawab312/AWS` — Macie, Inspector, Security Hub, security posture management sections

---

### Q296 — Kubernetes | Scenario-Based | Advanced

> You need to implement **Kubernetes `PodDisruptionBudget`** for a stateful application (Cassandra cluster) with specific requirements:
> - Minimum **2 out of 3 nodes** must always be available during voluntary disruptions
> - During a **cluster upgrade**, only 1 node should be drained at a time
> - If a Cassandra node is being drained and data replication is not complete, the drain should **block** until it is safe
>
> How do you configure PDB, and how do you implement the **application-aware drain blocking** using a custom PreStop hook?

📁 **Reference:** `nawab312/Kubernetes` → `09_CLUSTER_OPERATIONS.md` and `02_WORKLOADS.md` — PDB, drain, PreStop hook sections

---

### Q297 — ELK Stack | Troubleshooting | Advanced

> Your **Logstash** is showing `pipeline.workers` threads all stuck and the pipeline throughput has dropped to zero. CPU is 0% and the input queue is growing.
>
> What causes Logstash worker thread deadlock/stall? Walk me through diagnosing which stage is blocking (input/filter/output) and how to recover without losing queued events.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Logstash pipeline workers, persistent queues, deadlock sections

---

### Q298 — Jenkins | Conceptual | Advanced

> Explain **Jenkins `Groovy CPS (Continuation Passing Style)`** execution model. Why can't you use all Groovy features in a Jenkinsfile?
>
> What is the difference between **CPS-transformed** and **`@NonCPS`** methods? What types of code cause `NotSerializableException` in Jenkins pipelines and how do you fix them?

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — Groovy CPS, pipeline execution model, serialization sections

---

### Q299 — Linux / Bash | Conceptual | Advanced

> Explain **Linux `inode`** — what is it, what information does it store, and what does it NOT store?
>
> What is **inode exhaustion** — how can a filesystem run out of inodes while having free disk space? Give a real-world scenario where this happens in a DevOps context. How do you diagnose and fix inode exhaustion without unmounting the filesystem?

📁 **Reference:** `nawab312/DSA` → `Linux` — filesystem, inodes, df -i, inode exhaustion sections

---

### Q300 — AWS | Scenario-Based | Advanced

> You are implementing **AWS `CodePipeline`** for a complete CI/CD workflow that:
> - Sources from **GitHub** via CodeStar connection
> - Builds with **CodeBuild** (running unit tests + Docker build)
> - Scans image with **ECR image scanning**
> - Deploys to **ECS with CodeDeploy** (blue/green)
> - Has a **manual approval stage** before production
> - Sends **SNS notifications** on success and failure
>
> Walk me through the complete CodePipeline setup and compare it to your GitHub Actions equivalent.

📁 **Reference:** `nawab312/AWS` and `nawab312/CI_CD` — CodePipeline, CodeBuild, CodeDeploy, CI/CD comparison sections

---

### Q301 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `EndpointSlice` topology hints** and **topology-aware routing**. What problem does this solve for multi-AZ Kubernetes clusters?
>
> How does enabling topology hints reduce **cross-AZ traffic costs** and **latency**? What are the trade-offs and when would topology-aware routing cause problems?

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — EndpointSlices, topology hints, AZ-aware routing sections

---

### Q302 — GitHub Actions | Troubleshooting | Advanced

> Your GitHub Actions workflow is **failing intermittently** with rate limit errors when calling the GitHub API, and separately, a Docker build step is failing with:
>
> `toomanyrequests: You have reached your pull rate limit`
>
> How do you solve both rate limiting issues — GitHub API rate limits in workflows and Docker Hub pull rate limits in CI/CD?

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — rate limiting, Docker Hub, registry authentication sections

---

### Q303 — Terraform | Conceptual | Advanced

> Explain **Terraform `moved` block** use cases in depth. Show examples for:
> - Renaming a resource within the same module
> - Moving a resource into a module
> - Moving a resource out of a module
> - Renaming a module itself
>
> What is the difference between using `moved` block vs `terraform state mv` command, and why is the `moved` block preferred?

📁 **Reference:** `nawab312/Terraform` — moved block, state refactoring, module restructuring sections

---

### Q304 — Prometheus | Troubleshooting | Advanced

> Your **AlertManager** is showing alerts as `silenced` but you did not create any silences. Alerts are not being sent to Slack or PagerDuty. When you check AlertManager UI, the inhibition rules are matching and suppressing the alerts.
>
> Explain **AlertManager inhibition rules** — how they work, common misconfigurations, and how to debug why inhibition is suppressing alerts unexpectedly.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — Alertmanager, inhibition rules, debugging sections

---

### Q305 — AWS | Conceptual | Advanced

> Explain **AWS `PrivateLink`** in depth. How does it work technically (endpoint services, interface endpoints, gateway endpoints)?
>
> What is the difference between **interface endpoints** and **gateway endpoints**? Which AWS services support gateway endpoints (and why are they free)? Design a scenario where PrivateLink enables a SaaS provider to offer their service to customers without VPC peering.

📁 **Reference:** `nawab312/AWS` — PrivateLink, VPC endpoints, interface vs gateway endpoints sections

---

### Q306 — Kubernetes | Scenario-Based | Advanced

> You need to implement **Kubernetes `RuntimeClass`** to run certain sensitive workloads using **gVisor** (runsc) for enhanced isolation while running normal workloads with the standard runc runtime.
>
> Walk me through:
> - Installing and configuring gVisor on nodes
> - Creating RuntimeClass objects
> - Scheduling sensitive pods to use gVisor
> - What performance trade-offs exist with gVisor vs runc

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md` and `13_DOCKER_CONTAINER_FUNDAMENTALS.md` — RuntimeClass, container runtimes, sandboxing sections

---

### Q307 — ArgoCD | Troubleshooting | Advanced

> ArgoCD is showing a **`SyncFailed`** status with the error:
>
> `one or more objects failed to apply, reason: Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": the server is currently unable to handle the request`
>
> The Ingress controller webhook is timing out during ArgoCD sync. Walk me through diagnosing and fixing webhook-related sync failures in ArgoCD.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` and `nawab312/Kubernetes` → `14_TROUBLESHOOTING.md` — sync failures, webhooks, admission controllers sections

---

### Q308 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that implements **automated SSH key rotation** across a fleet of servers:
> - Generates a new **4096-bit RSA key pair**
> - Distributes the new public key to all servers in `inventory.txt`
> - Verifies the new key works **before removing the old key**
> - Removes the old public key from `authorized_keys` on all servers
> - Handles **failures gracefully** — if a server fails, keeps old key and logs the failure
> - Generates an **audit report** of successful and failed rotations
> - All actions are logged with timestamp and operator identity

📁 **Reference:** `nawab312/DSA` → `Linux` — SSH key management, bash scripting, security automation sections

---

### Q309 — AWS | Scenario-Based | Advanced

> Your company is being acquired and the new parent company requires all AWS resources to be **consolidated into their AWS Organization**. You need to migrate:
> - 3 AWS accounts with production workloads
> - All IAM roles and policies
> - VPC configurations
> - RDS databases with data
> - S3 buckets with hundreds of TB of data
>
> What is your **migration strategy and sequence**? What are the biggest risks and how do you mitigate them?

📁 **Reference:** `nawab312/AWS` — AWS Organizations, account migration, cross-account data transfer sections

---

### Q310 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `Service` session affinity** (`ClientIP` mode). How does it work and what are its limitations?
>
> Also explain **`topologyKeys`** in Services (now replaced by topology-aware hints). What was the problem with the original `topologyKeys` implementation?
>
> How does an **`ExternalTrafficPolicy: Local`** Service differ from `Cluster` policy, and what are the trade-offs?

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — Service affinity, ExternalTrafficPolicy, topology sections

---

### Q311 — Grafana | Scenario-Based | Advanced

> You need to build a **Grafana dashboard for Kubernetes cluster overview** that shows:
> - Cluster CPU and memory utilization with **capacity planning** projection (next 7 days)
> - Top 10 namespaces by resource consumption (CPU + memory)
> - Pod health summary (running/pending/failed counts) per namespace
> - Node pressure conditions (DiskPressure, MemoryPressure, PIDPressure)
> - Recent events (OOMKills, evictions) in the last hour
>
> Write the PromQL queries for each panel.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` and `Prometheus` — Kubernetes monitoring, cluster overview queries sections

---

### Q312 — Terraform | Scenario-Based | Advanced

> You need to implement **Terraform `for` expressions** and **`flatten`** function for complex infrastructure generation.
>
> Write Terraform code that:
> - Takes a map of `teams` with their `environments` and `services` as input
> - Creates an S3 bucket for each `team/environment/service` combination
> - Creates IAM roles with proper naming convention for each combination
> - Uses `for` expressions, `flatten`, and `setproduct` to generate all combinations without nested loops

📁 **Reference:** `nawab312/Terraform` — for expressions, flatten, setproduct, complex variable structures sections

---

### Q313 — Git | Scenario-Based | Advanced

> Your team is adopting **Conventional Commits** specification for commit messages. You need to enforce this in your CI/CD pipeline and automate:
> - **Commit message linting** on every PR (reject non-conforming commits)
> - **Automatic versioning** using semantic-release based on commit types
> - **CHANGELOG generation** from commit history
> - **Git tag creation** on release
>
> Walk me through implementing this end-to-end using GitHub Actions.

📁 **Reference:** `nawab312/CI_CD` → `Git` and `GithubActions` — conventional commits, semantic versioning, changelog automation sections

---

### Q314 — AWS + Kubernetes | Scenario-Based | Advanced

> You are running **EKS** and your team reports that pod-to-pod **network performance is degraded** — throughput between pods on different nodes is only 1 Gbps when it should be 10 Gbps. Pods on the same node communicate at full speed.
>
> Walk me through diagnosing inter-node pod network performance issues in EKS — covering VPC CNI configuration, EC2 instance networking limits, security group rules, and MTU settings.

📁 **Reference:** `nawab312/AWS` and `nawab312/Kubernetes` — EKS networking, VPC CNI, ENI, MTU, network performance sections

---

### Q315 — All Topics | System Design | Advanced

> You are designing a **self-healing infrastructure platform** for a cloud-native application. The platform must:
> - **Automatically detect** degraded components (pods, nodes, databases, queues)
> - **Automatically remediate** common failures without human intervention (restart pods, drain nodes, failover databases)
> - Have **circuit breakers** to stop auto-remediation if it is making things worse
> - **Escalate to on-call** when auto-remediation fails or when the situation is outside known patterns
> - Maintain a **remediation audit log** for compliance and post-incident review
> - Support **runbook automation** — structured remediation steps triggered by specific alerts
>
> Design the complete platform architecture covering: detection, decision engine, execution, circuit breaking, and escalation.

📁 **Reference:** All repositories — self-healing infrastructure, runbook automation, SRE platform design

---

## Key Topics Coverage (Q271–Q315)

| Topic | Questions |
|---|---|
| Kubernetes | Q271, Q276, Q281, Q287, Q291, Q296, Q301, Q306, Q310 |
| AWS | Q272, Q278, Q285, Q290, Q295, Q300, Q305, Q309, Q314 |
| Terraform | Q274, Q283, Q293, Q303, Q312 |
| Prometheus | Q275, Q286, Q304 |
| Grafana | Q292, Q311 |
| ELK Stack | Q280, Q297 |
| Linux / Bash | Q273, Q282, Q289, Q299, Q308 |
| ArgoCD | Q288, Q307 |
| Jenkins | Q279, Q298 |
| GitHub Actions | Q284, Q302 |
| Git | Q277, Q313 |
| Python | Q294 |
| System Design | Q315 |

---

## New Concepts Introduced in This Set

### Kubernetes
- Subresources — `/status`, `/scale`, `/exec`, fine-grained RBAC
- `ownerReferences` garbage collection — foreground vs background (new angle)
- Evicted pod accumulation — bulk cleanup
- Scheduler timing race condition with `not-ready` taint
- RuntimeClass — gVisor vs runc sandbox isolation
- `ExternalTrafficPolicy: Local` vs `Cluster`
- Service session affinity — limitations
- EndpointSlice topology hints — AZ-aware routing (cost/latency)
- LimitRange + ResourceQuota edge cases — `scopeSelector`
- PDB + PreStop hook — application-aware drain blocking

### AWS
- Immutable infrastructure — Packer + AMI baking + ASG blue/green swap
- EventBridge — event-driven architecture (Lambda, Step Functions triggers)
- CloudFormation `UPDATE_ROLLBACK_FAILED` — recovery options
- CodePipeline end-to-end vs GitHub Actions comparison
- Savings Plans vs Reserved Instances — detailed comparison + tools
- AWS Macie + Inspector + Security Hub + ASFF
- Lake Formation — column-level security, Glue Catalog, Athena
- AWS account migration during acquisition
- PrivateLink — interface vs gateway endpoints (new depth)

### Terraform
- Backend types — 5 backends, partial configuration
- Policy as code — OPA/Conftest in CI/CD
- `moved` block — 4 use cases (rename, into module, out of module, rename module)
- `terraform init` failures — checksum mismatch, registry connectivity
- `for` expressions + `flatten` + `setproduct` — complex resource generation
- `check` blocks + `precondition` — validation assertions (new examples)
- `dynamic` blocks (new angle — conditional egress rules)

### Prometheus
- `relabel_configs` vs `metric_relabel_configs` — execution order
- Exemplars — link metrics to traces
- Alertmanager inhibition — debugging unexpected suppression
- Redis monitoring — exporter, hit rate, eviction alerts

### Linux
- Network namespaces — `ip netns`, veth pairs, Docker plumbing
- `inode` exhaustion — diagnosis without unmounting
- NTP sync issues — chrony, timedatectl, clock drift monitoring
- `epoll` (new angle — C10K, Node.js event loop)
- `git rerere` and `git notes` — advanced recovery features

### Jenkins
- Groovy CPS — why not all Groovy works in Jenkinsfile
- `@NonCPS` — `NotSerializableException` fix
- Feature flag progressive rollout pipeline

### Git
- `git rerere` — reuse recorded resolution
- `git notes` — attach CI metadata to commits
- Conventional commits — full automation pipeline
- Git submodules — update workflow and pitfalls

### ArgoCD
- Resource health — custom Lua health checks for CRDs
- Sync failures due to webhook timeouts

### Grafana
- Transformations — join, filter, rename, calculate fields
- Dashboard annotations — CI/CD pipeline integration via API

---

*Part 7 of DevOps/SRE Interview Questions — nawab312 GitHub repositories*
*Total question bank: Q1–Q315*
