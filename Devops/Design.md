# 50 System Design Interview Questions — DevOps / SRE / Platform Engineering
> Senior / Staff level · Top-tier product company

---

## Q1 | CI/CD Pipelines → GitHub Actions + ArgoCD | CI/CD Design

Your organization has 200 engineers committing to a monorepo containing 40 microservices. The current Jenkins-based pipeline takes 45 minutes end-to-end, causes frequent merge queue collisions, and has no meaningful separation between staging and production promotion gates. Design a modern CI/CD system using GitHub Actions for CI and ArgoCD for GitOps-based CD that reduces pipeline time to under 12 minutes, enforces environment promotion gates, and supports per-service independent deployments at a rate of 80+ deploys per day.

**Design constraints to address:**
- Affected-service detection to avoid rebuilding all 40 services on every commit
- Promotion gates: automated test coverage thresholds, security scan (Trivy/Snyk) pass, and mandatory peer approval for prod
- ArgoCD ApplicationSet pattern to manage 40 services across 3 environments (dev, staging, prod) without config duplication
- Rollback strategy: how does an engineer revert a single service in prod within 90 seconds without touching other services

---

## Q2 | AWS Infrastructure → Multi-AZ RDS + Aurora | HA Architecture

You run a financial SaaS product on AWS with a PostgreSQL RDS instance (db.r6g.4xlarge, Multi-AZ) serving 5,000 writes/second and 50,000 reads/second at peak. The current architecture has no read replica routing, point-in-time recovery is tested once a year, and the RTO is informally "a few hours." The business now mandates a 99.99% availability SLA with RTO < 5 minutes and RPO < 30 seconds. Design a new database architecture to meet these SLAs — you may migrate to Aurora PostgreSQL if it helps.

**Design constraints to address:**
- Read/write split routing strategy at the application layer (PgBouncer, RDS Proxy, or application-level)
- Aurora Global Database vs. Multi-AZ failover — justify the tradeoff against cost
- Automated failover testing strategy that runs in production without causing downtime
- Connection pool exhaustion: what happens when a sudden 10x spike in connections hits during a failover event

---

## Q3 | Terraform & IaC → Module Design + State Management | Platform Design

Your platform team owns 150 Terraform modules used by 12 product teams. State is stored in a single S3 bucket with one state file per environment, causing frequent state lock contention and a single blast radius for all infra. Three teams have begun making manual AWS console changes that drift from state, and there is no automated remediation. Design a scalable IaC architecture for 12 teams with state isolation, a module registry, and automated drift detection.

**Design constraints to address:**
- State file partitioning strategy: per-team, per-service, per-component — define the boundaries and justify them
- Module versioning and breaking change management: how do teams consume v2.0 of a module without being force-upgraded
- Drift detection pipeline: design a scheduled Terraform plan workflow that alerts on drift but does not auto-apply
- Sentinel or OPA policy enforcement: enforce tagging standards, instance type allowlists, and region restrictions at plan time

---

## Q4 | Kubernetes & EKS → Multi-Cluster Strategy | Scaling Design

Your company runs a single EKS cluster (500 nodes, mixed instance types) serving all workloads: internal tools, customer-facing APIs, batch jobs, and ML training pipelines. A noisy ML job regularly causes CPU throttling on customer-facing pods despite resource limits. An AWS AZ outage 6 months ago took down the entire cluster for 22 minutes. Design a multi-cluster EKS architecture for 500 engineers that provides workload isolation, AZ fault tolerance, and independent scaling for different workload classes.

**Design constraints to address:**
- Cluster topology: how many clusters, what workload affinity rules, and how do you manage cross-cluster service discovery
- Control plane HA: EKS managed control plane behavior during AZ loss and how to mitigate the 22-minute outage scenario
- Cluster Autoscaler vs. Karpenter: evaluate both for mixed instance type provisioning and justify your choice
- Multi-cluster GitOps with ArgoCD: how do you deploy the same application to 3 clusters with environment-specific overrides without config explosion

---

## Q5 | Observability → Full Stack Design | Observability Design

You are building the observability stack for a platform serving 50M daily active users across 80 microservices. Currently there is no distributed tracing, logs are written to CloudWatch Logs with no structured schema, and on-call engineers spend 40 minutes average on MTTD during incidents. Design a production-grade observability platform using open-source tooling (Prometheus, Grafana, Loki, Tempo, OpenTelemetry) that reduces MTTD to under 5 minutes and supports SLO-based alerting.

**Design constraints to address:**
- OpenTelemetry collector topology: where do collectors sit, how do you handle backpressure and data loss during a spike
- SLO alert design: error budget burn rate alerts (fast burn + slow burn) with specific Prometheus recording rules — explain the math
- Log cardinality explosion: how do you prevent teams from introducing high-cardinality labels that break Prometheus/Loki at scale
- Trace sampling strategy: what sampling approach do you use when full tracing would generate 500GB/day of trace data

---

## Q6 | Security Architecture → Zero Trust + mTLS | Security Architecture

Your microservices platform currently uses network-level segmentation (security groups) as the primary trust boundary — services inside the VPC trust each other implicitly. You need to move to a zero-trust model for 80 services across 3 EKS clusters. A recent pen test found 4 cases of lateral movement through compromised internal services. Design a zero-trust network architecture using service mesh (Istio or Cilium) that implements mTLS, RBAC-based service-to-service authorization, and observable network policy.

**Design constraints to address:**
- Certificate rotation: how do you rotate mTLS certs for 80 services with zero downtime, and what is the cert TTL strategy
- AuthorizationPolicy granularity: design policies at the namespace, service, and method level — give a concrete example for a payments service
- Cilium vs. Istio tradeoffs: eBPF-based network policy vs. sidecar mesh — evaluate for 500-node clusters with 10Gbps inter-service traffic
- Brownfield migration: how do you incrementally roll out mTLS to 80 existing services without breaking any of them on day one

---

## Q7 | Cost Optimization → EKS Spot + Reserved | Cost Optimization

Your AWS bill is $2.4M/month. EKS compute accounts for $1.1M of that, with 80% of workloads running on On-Demand instances. A cost audit reveals: 35% of pods are over-provisioned by 3x, Spot instances are used for less than 5% of workloads, and no Reserved Instances or Savings Plans are in place. Design a FinOps strategy to reduce the compute bill by 50% within 6 months without degrading p99 latency SLOs or increasing incident rate.

**Design constraints to address:**
- Spot instance interruption handling: design a graceful shutdown architecture for stateless services running on Spot, including PodDisruptionBudgets and node drain timing
- VPA vs. HPA vs. KEDA: which autoscaler for which workload type, with specific examples — don't say "use all three" without justifying when
- Savings Plans vs. Reserved Instances vs. Compute Savings Plans: how do you commit to the right level without over-committing as workloads shift
- Rightsizing pipeline: design an automated monthly rightsizing report that identifies over-provisioned namespaces and generates actionable Helm value PRs

---

## Q8 | CI/CD Pipelines → Supply Chain Security | Security Architecture

You ship 150 container images per day across 40 microservices. A security audit reveals: base images are pulled from Docker Hub without pinning, no SBOM is generated, no image signing is in place, and a compromised third-party npm package in your build could reach production in under 2 hours. Design a software supply chain security architecture that implements image signing, SBOM generation, vulnerability gating, and provenance verification from commit to production.

**Design constraints to address:**
- Sigstore/Cosign integration: how do you sign images in CI, store signatures in OCI registries, and verify at admission control in Kubernetes
- SBOM generation and storage: which format (SPDX vs. CycloneDX), where it lives, and how it is queried during an incident response scenario
- Vulnerability gate thresholds: how do you distinguish between "block this deploy" (CVSS 9.0+) and "create a ticket" (CVSS 4.0–8.9) without alert fatigue
- Dependency confusion and typosquatting: design a private npm/PyPI mirror architecture that prevents teams from accidentally pulling malicious packages

---

## Q9 | Disaster Recovery → Multi-Region RTO/RPO | Disaster Recovery

Your e-commerce platform processes $8M/day in GMV. It runs in a single AWS region (us-east-1). You have no tested DR plan. The business requires RTO < 15 minutes and RPO < 1 minute for a full region failure. Design a multi-region DR architecture for this platform including compute, data, DNS failover, and a runbook that a single on-call engineer can execute at 3am without scripted automation failing.

**Design constraints to address:**
- Data replication: Aurora Global Database vs. DMS continuous replication vs. application-level dual-write — evaluate each against the 1-minute RPO constraint
- DNS failover: Route 53 health checks + latency routing vs. Route 53 ARC (Application Recovery Controller) — when does each fail and what is the gap
- Stateful workload recovery: design the recovery sequence for Redis (ElastiCache), S3, and RDS in the correct dependency order with estimated time per step
- DR test cadence: design a quarterly game day that validates RTO without taking production offline, including how you handle the data divergence from the test

---

## Q10 | Kubernetes & EKS → Cluster Upgrade Strategy | Migration Design

You are running EKS 1.24 across 3 clusters (600 nodes total). EKS 1.24 reaches end-of-life in 60 days. You have 80 microservices, 15 CRDs from various operators, and 6 deprecated API versions (extensions/v1beta1, policy/v1beta1) that teams are still using. An upgrade attempt in staging last quarter caused 3 operator failures and a 40-minute outage. Design a zero-downtime upgrade path from EKS 1.24 to 1.29 across 3 clusters serving production traffic.

**Design constraints to address:**
- Deprecated API migration: design a tooling pipeline (pluto, kubent) to detect and remediate all deprecated API usage before the upgrade window
- Node group rolling upgrade: Blue/Green node groups vs. in-place managed node group updates — pick one and explain the rollback path if a node fails to join
- Operator and CRD compatibility matrix: how do you validate that all 15 operators are compatible with target version before starting the cluster upgrade
- Multi-version upgrade path (1.24→1.25→1.26...→1.29): design the sequencing, testing gates between each hop, and total calendar timeline given your constraints

---

## Q11 | Observability → Distributed Tracing at Scale | Observability Design

Your platform generates 2 billion spans per day across 80 services. You currently store 100% of traces in Jaeger backed by Elasticsearch, costing $180K/month in storage alone. P99 trace query latency is 45 seconds. Engineers use traces primarily for post-incident root cause analysis and performance regression detection during deploys. Design a cost-optimized tracing architecture that retains full fidelity for high-value traces, reduces storage cost by 70%, and keeps query latency under 3 seconds.

**Design constraints to address:**
- Tail-based sampling with OpenTelemetry Collector: design a sampling strategy that captures 100% of error traces, 100% of traces over 2s, and 1% of healthy traces
- Storage tiering: hot storage (Tempo/ClickHouse) for last 7 days, cold storage (S3 Parquet via Jaeger remote storage) for 90 days — design the query federation layer
- Exemplars: how do you link Prometheus metrics to trace IDs so engineers can jump from a Grafana dashboard spike to the exact trace, at 2B spans/day
- Trace-based testing in CI: design a pipeline stage that compares p99 latency of critical trace paths against a baseline and fails the deploy if regression exceeds 15%

---

## Q12 | AWS Infrastructure → VPC + Network Design | Security Architecture

You are designing the AWS network architecture for a regulated healthcare SaaS company that must achieve HIPAA compliance. The architecture serves 3 product teams, needs to support dev/staging/prod environments, requires private connectivity to 3 on-premises data centers, and must ensure that a misconfigured security group in one team's environment cannot expose another team's PHI workloads. Design the VPC architecture from scratch.

**Design constraints to address:**
- VPC topology: shared VPC vs. per-team VPCs with VPC peering vs. AWS Transit Gateway — evaluate each against blast radius isolation and operational overhead
- PrivateLink vs. VPC peering vs. Transit Gateway for cross-team service communication — give specific decision criteria
- Egress control: design internet egress architecture using NAT Gateway vs. AWS Network Firewall with domain-based blocking for PHI workloads
- Flow logs + GuardDuty integration: design the detection pipeline for lateral movement, data exfiltration, and port scanning alerts with < 2-minute detection latency

---

## Q13 | Cost Optimization → S3 Storage Tiering | Cost Optimization

Your data platform stores 4PB of data in S3 Standard across 12 buckets. Monthly S3 cost is $92K. A storage audit reveals: 60% of objects have not been accessed in over 180 days, 20% are duplicated across buckets due to legacy ETL pipelines, and 15% are temporary files never deleted. Design an S3 lifecycle and governance architecture that reduces monthly cost by 60% without breaking any downstream data consumers.

**Design constraints to address:**
- Lifecycle policy design: define the transition rules (Standard → Standard-IA → Glacier Instant Retrieval → Glacier Deep Archive) with specific day thresholds per object class
- Access pattern analysis: how do you determine which objects are safe to tier without querying 4PB of access logs — design the S3 Storage Lens + Athena query pipeline
- Duplicate detection and deduplication: design a pipeline to identify cross-bucket duplicates at 4PB scale without downloading objects
- Restore latency SLAs: how do you communicate to data consumers that a Glacier Deep Archive restore takes 12–48 hours, and design a pre-warm strategy for known batch jobs

---

## Q14 | Developer Platform → Internal Developer Platform | Platform Design

Your organization has grown from 50 to 300 engineers in 18 months. Engineers spend an average of 3 days provisioning a new microservice (creating repos, CI pipelines, AWS infra, DNS, monitoring dashboards, PagerDuty integrations). There are 7 different IaC patterns in use and no golden paths. Design an Internal Developer Platform (IDP) that reduces service bootstrapping to under 2 hours via self-service, enforces golden paths, and doesn't require engineers to learn Terraform.

**Design constraints to address:**
- Backstage scaffolder templates: design the template flow for a new microservice — what it provisions, what it doesn't, and how it handles async infra provisioning (Terraform Apply can take 8 minutes)
- Abstraction boundary: what is the right level of abstraction for compute (ECS task? EKS Deployment? Lambda?) so platform teams can change the underlying infra without breaking the developer contract
- Day-2 operations in the IDP: how does an engineer scale their service, change environment variables, view logs, and trigger a rollback — all without leaving the IDP UI
- Compliance guardrails: enforce that every service has a cost center tag, an oncall rotation, a runbook URL, and a data classification label at provisioning time — not as a suggestion

---

## Q15 | CI/CD Pipelines → GitOps Workflow Design | CI/CD Design

Your team is adopting GitOps with ArgoCD for 60 services across dev, staging, and prod environments. The current workflow has engineers directly editing Kubernetes manifests in a shared repo, causing frequent merge conflicts. There is no automated image tag promotion between environments, and prod deployments require a manual JIRA ticket approval that takes 48 hours on average. Design a complete GitOps workflow that automates promotion, enforces approval gates programmatically, and reduces prod deploy cycle time from 48 hours to under 2 hours.

**Design constraints to address:**
- Repository structure: mono-repo vs. split config/app repos — pick one, justify it, and define the directory structure for 60 services × 3 environments
- Automated image promotion: how does a new image tag in dev automatically propagate to staging after tests pass, and to prod after staging soak time completes — without any human editing YAML
- ArgoCD ApplicationSet + generators: design the generator strategy (git directory, matrix, or cluster generators) that avoids copy-paste and supports per-environment overrides
- Drift detection and self-heal policy: how do you configure ArgoCD to alert on drift in prod but auto-sync in dev, and what triggers a Slack alert vs. a PagerDuty page

---

## Q16 | Multi-Tenancy Design → SaaS Platform Isolation | Multi-Tenancy Design

You are building a multi-tenant SaaS analytics platform that will serve 500 enterprise customers. Tenants range from 10-user startups to 50,000-user enterprises. The platform runs on Kubernetes and processes up to 100TB of customer data daily. A noisy tenant should never degrade another tenant's query performance, and tenant data must be cryptographically isolated. Design the multi-tenancy architecture for compute, storage, and networking.

**Design constraints to address:**
- Tenant isolation models: shared namespace + NetworkPolicy vs. dedicated namespace per tenant vs. dedicated node pool per tier — define when you use each based on tenant size/tier
- ResourceQuota and LimitRange design: design a three-tier quota system (small/medium/enterprise) with specific CPU, memory, and PVC limits that prevent noisy-neighbor behavior
- Data isolation: design the encryption key hierarchy (AWS KMS, per-tenant CMK) and how query engines (Presto/Trino) enforce tenant data boundaries at the storage layer
- Tenant onboarding automation: design the Kubernetes resource creation pipeline that spins up all namespaces, RBAC, quotas, and network policies for a new tenant in under 60 seconds

---

## Q17 | Security Architecture → Secrets Management | Security Architecture

Your organization has 300 engineers deploying 80 microservices. A security audit found secrets in 47 Git repositories, hardcoded database credentials in 12 Dockerfiles, and no rotation policy for any secrets. Three services share the same database credentials. Design a centralized secrets management architecture using HashiCorp Vault (or AWS Secrets Manager) that eliminates hardcoded secrets, enforces least-privilege access, and automates rotation for all secret types.

**Design constraints to address:**
- Vault vs. AWS Secrets Manager: evaluate for a multi-cloud future, developer ergonomics, dynamic secret generation, and operational overhead — make a recommendation
- Dynamic secrets for RDS: design the Vault database secrets engine flow for PostgreSQL — how credentials are issued per-service, TTL strategy, and what happens when a service crashes mid-TTL
- Kubernetes integration: ESO (External Secrets Operator) vs. Vault Agent Sidecar vs. CSI driver — pick one, explain the secret sync behavior during pod restart and pod scheduling on a new node
- Rotation runbook: for a rotated RDS master password, design the zero-downtime rotation sequence including Vault, RDS, and all consuming application pods

---

## Q18 | Scaling Design → Global API Gateway | Scaling Design

Your API platform currently serves 10M requests/day from a single AWS region (us-east-1) with p99 latency of 180ms for users in Asia and Europe. Traffic is expected to grow 10x (100M requests/day) over 18 months as the product launches in APAC and EMEA. Design a globally distributed API gateway architecture that serves sub-50ms p99 latency for users globally and scales to 100M requests/day with no manual capacity planning.

**Design constraints to address:**
- CloudFront + API Gateway vs. CloudFront + ALB vs. CloudFront + Lambda@Edge vs. Fastly/Cloudflare Workers — evaluate each for latency, cost at 100M req/day, and operational complexity
- Cache hit ratio design: for an API where 40% of responses are cacheable (user-agnostic catalog data), design the cache key strategy, TTL hierarchy, and cache invalidation pipeline
- Rate limiting at the edge: design a distributed rate limiting system that enforces per-customer quotas globally without a single centralized Redis becoming a bottleneck
- Regional failover: what happens when us-east-1 is degraded — how does the gateway route traffic to us-west-2, and how do you prevent cache stampede during failover

---

## Q19 | Terraform & IaC → Drift Detection + Remediation | Migration Design

Your organization has 400 AWS resources that were created manually before Terraform adoption. 60% of your production infrastructure has drifted from Terraform state. Attempts to run `terraform apply` on drifted resources have caused 3 production outages in the past year. Design a systematic process to bring all 400 resources under Terraform management, detect and remediate ongoing drift, and prevent future drift without blocking engineers from using the AWS console for emergency changes.

**Design constraints to address:**
- Import strategy: `terraform import` vs. Terraformer vs. former2 for mass import — evaluate accuracy, time cost, and human review requirements for 400 resources
- Drift detection pipeline: scheduled `terraform plan` in CI with no write permissions, alerting on drift with Slack severity levels (informational vs. critical), and owner attribution per resource
- Emergency console access: design the process where an engineer can make a console change during an incident and have it automatically reconciled into Terraform within 24 hours
- State blast radius: how do you partition state so that a corrupted state file for team A's infra doesn't prevent team B from deploying

---

## Q20 | Observability → Cost-Aware Metrics Architecture | Cost Optimization

Your Prometheus + Thanos stack ingests 15 million active time series, growing 30% month-over-month. Monthly observability infrastructure cost is $85K (Thanos object storage, query frontend, compactor). Query latency for 30-day range queries is 90 seconds. Engineers are creating dashboards with hundreds of high-cardinality label combinations. Design a metrics architecture that caps cost growth, maintains query performance at 3x current cardinality, and enforces cardinality governance without alienating engineering teams.

**Design constraints to address:**
- Cardinality governance: design a pre-merge CI check that estimates cardinality contribution of new metrics and blocks PRs that would add >10K new series without review
- Thanos vs. Grafana Mimir vs. VictoriaMetrics: evaluate for write throughput (15M series), query performance, and total cost of ownership at 3x scale
- Recording rules strategy: design a tiered recording rule architecture that pre-aggregates high-cardinality series into low-cardinality summaries for dashboards, reducing query time from 90s to under 5s
- Metric retention tiers: design a two-tier retention policy (full resolution for 15 days, 5-minute aggregates for 1 year) with the Thanos compactor configuration that implements it

---

## Q21 | Disaster Recovery → Database Backup Architecture | Disaster Recovery

Your platform has 18 PostgreSQL RDS databases, 6 Redis clusters, and 3 DynamoDB tables. Current backup strategy: automated RDS snapshots (daily), no Redis persistence, and DynamoDB point-in-time recovery enabled but never tested. A recent data corruption incident took 6 hours to recover because no one knew the restore procedure. RPO requirement: 5 minutes for transactional databases. RTO requirement: 30 minutes for full service restoration. Design a backup and recovery architecture that meets these SLAs across all three database types.

**Design constraints to address:**
- Continuous WAL archiving for RDS PostgreSQL using pgBackRest or AWS native: design the archiving pipeline to S3, including cross-region replication for DR, and validate the 5-minute RPO claim mathematically
- Redis persistence: RDB vs. AOF for each cluster based on workload type (session cache vs. rate limiter vs. leaderboard) — and why you might use neither for some use cases
- Recovery runbook automation: design a Lambda-based automated restore orchestrator that can recover a corrupted RDS instance to a specific point-in-time and update Route 53 DNS, triggered by a single Slack command
- Annual DR test: design a chaos engineering exercise that simulates a full primary region database failure and validates RTO/RPO claims without exposing customer data to the DR environment

---

## Q22 | Kubernetes & EKS → RBAC + Multi-Team Access | Security Architecture

Your EKS cluster is shared by 15 product teams and a platform team. Currently all engineers have cluster-admin access "for productivity." A misconfigured `kubectl delete` command by a junior engineer last month deleted a production namespace. Sensitive services (payments, auth) share the same cluster as internal tools with much lower security requirements. Design an EKS RBAC architecture that enforces least privilege, isolates sensitive workloads, and doesn't require the platform team to process every `kubectl` request manually.

**Design constraints to address:**
- Namespace-to-team mapping: design the RBAC role hierarchy (ClusterRole, Role, RoleBinding) with specific permission sets for developer, lead, and platform engineer personas
- AWS IAM to Kubernetes RBAC mapping: design the `aws-auth` ConfigMap (or EKS Access Entries in newer versions) structure that maps 15 team IAM roles to appropriate Kubernetes groups without a platform engineer touching it for every new hire
- Privileged workload isolation: design the admission control policy (Kyverno or OPA Gatekeeper) that prevents non-platform teams from deploying privileged containers or hostPath mounts
- Audit logging: design the EKS audit log pipeline to CloudWatch + Athena that can answer "who deleted namespace X and when" within 60 seconds of an incident

---

## Q23 | CI/CD Pipelines → Database Migration Safety | CI/CD Design

Your application deploys 40 times per week, and each deploy may include a database schema migration (Flyway/Liquibase). Two migrations in the last quarter caused production outages: one added a NOT NULL column without a default, another renamed a column while old pods still referenced the old name. You need zero-downtime migrations for a PostgreSQL database with 800GB of data and 5,000 writes/second. Design a migration safety framework built into the CI/CD pipeline.

**Design constraints to address:**
- Expand/Contract pattern enforcement: design a CI linting step that detects "unsafe" migrations (column renames, NOT NULL additions, index creation without CONCURRENTLY) and blocks the deploy
- Blue/Green deploy + migration compatibility: define the rule that migrations must be backward-compatible for N-1 deploy versions, and how you enforce this in code review
- Large table migration strategy: design the online schema change approach (pt-online-schema-change or pg_rewrite) for adding an index to a 500M-row table with <1% replication lag impact
- Rollback: since database migrations are often irreversible, design the "migration undo" strategy — and specifically when you should not roll back application code to avoid data loss

---

## Q24 | Multi-Tenancy Design → Kubernetes Namespace Quotas | Multi-Tenancy Design

You operate a Kubernetes-based platform with 80 tenant namespaces. Three incidents in the last month involved a single tenant consuming all available cluster memory, starving other tenants. There is no cost attribution per tenant, making it impossible to bill accurately. Tenants range from 5 to 500 pods. Design a quota and cost attribution architecture for 80 tenants that prevents resource starvation, enables per-tenant cost showback/chargeback, and scales to 200 tenants without manual intervention.

**Design constraints to address:**
- LimitRange + ResourceQuota design: define default container resource limits, namespace-level quotas, and priority classes — show the specific YAML structure for a medium-tier tenant
- Hierarchical namespaces (HNC) vs. flat namespaces for tenants with multiple sub-teams — evaluate the operational complexity vs. isolation tradeoff
- Cost attribution pipeline: design the Kubecost or OpenCost integration that maps per-namespace CPU/memory/storage consumption to a per-tenant monthly cost, including spot instance discount allocation
- Burst capacity: design a system where a tenant can temporarily exceed their quota (up to 2x for 15 minutes) without requiring a platform team ticket, using PriorityClass and preemption

---

## Q25 | AWS Infrastructure → Compute Modernization | Migration Design

Your organization runs 200 EC2 instances (mixed m5, c5, r5) managed by Ansible, paying $180K/month for compute. 60% of instances have average CPU utilization below 8%. Workloads include: a Rails monolith (always-on), a Python batch job (runs 4 hours nightly), a Node.js API (bursty, 0→2000 RPS in 30 seconds), and 40 internal tooling services used only during business hours. Design a compute modernization strategy that migrates these workloads to the right compute primitive (ECS, EKS, Lambda, Fargate) and reduces compute cost by 55%.

**Design constraints to address:**
- Workload-to-compute mapping: justify the migration target for each of the four workload types above based on scaling pattern, cost model, and operational overhead
- Monolith to containers: design the Dockerization + ECS migration path for the Rails monolith that maintains zero downtime during cutover, including session state and file upload handling
- Lambda cold start mitigation for the bursty Node.js API: evaluate Lambda provisioned concurrency vs. ECS with KEDA scale-to-zero vs. Fargate Spot — compare cost at 0 RPS and at 2000 RPS
- Ansible decommissioning: design the parallel-run period where Ansible and the new compute platform coexist, including configuration parity validation and rollback trigger criteria

---

## Q26 | Observability → Incident Response Tooling | Observability Design

Your on-call rotation handles 120 alerts per week across 80 services, but 70% are either false positives or require no action. Mean time to acknowledge is 8 minutes; mean time to resolve is 95 minutes. Engineers report "alert fatigue" as the top frustration in quarterly surveys. Design an incident response system with intelligent alert routing, automated triage, and structured incident coordination that reduces actionable alert volume by 60% and MTTR by 40%.

**Design constraints to address:**
- Alert quality pipeline: design the feedback loop where engineers can mark alerts as "noise" and that signal is used to automatically adjust thresholds using Prometheus recording rules
- Multi-window burn rate alert design: replace all threshold-based alerts for 5 critical services with SLO burn rate alerts — show the specific alert expressions and the math behind the windows
- Automated triage runbooks: design the PagerDuty + AWS Lambda integration that auto-executes diagnostic runbooks (memory dump, log tail, dependency health check) and attaches results to the incident before the on-call engineer is even paged
- Incident coordination: design the Slack-based war room workflow — channel creation, stakeholder notification, timeline capture, and automated postmortem draft generation using incident data

---

## Q27 | Cost Optimization → FinOps Program Design | Cost Optimization

Your AWS bill has grown from $800K to $2.4M/month over 18 months. There is no cost ownership at the team level — all costs go to a single AWS account. Engineers don't know the cost of their services, and there is no process to review or challenge cost anomalies. Design a FinOps program architecture including tooling, processes, tagging taxonomy, and chargeback mechanisms that creates cost accountability without slowing down engineering velocity.

**Design constraints to address:**
- AWS account structure: evaluate single account + cost allocation tags vs. multi-account with AWS Organizations for cost isolation at the team level — include the migration path from your current single-account setup
- Tagging enforcement: design the AWS Config + SCP policy that prevents untagged resource creation, retroactively tags 10,000 existing untagged resources, and reports compliance weekly
- Anomaly detection: design the AWS Cost Anomaly Detection + SNS pipeline that alerts the right team (not just cloud ops) within 4 hours of a cost spike exceeding 20% week-over-week
- Engineering incentives: design a monthly cost review process where teams see their spend, compare to budget, and have a lightweight process to challenge or optimize — without it becoming a bureaucratic burden

---

## Q28 | Kubernetes & EKS → Stateful Workloads Design | HA Architecture

You need to run a 3-node Kafka cluster, a 6-node Elasticsearch cluster, and a 3-replica PostgreSQL (Patroni) cluster on EKS. Your team has previously run all stateful workloads on dedicated EC2 instances. Leadership wants to consolidate onto EKS to reduce operational overhead. Design the stateful workload architecture on EKS that achieves the same reliability (99.95%) as the EC2 setup, handles node failures gracefully, and doesn't compromise on data durability.

**Design constraints to address:**
- Storage class design: gp3 vs. io2 for each workload, IOPS provisioning, and encryption — justify the per-workload storage choice based on access patterns
- StatefulSet pod identity and PVC binding: design the failure recovery sequence when a Kafka broker pod is evicted from a node — how the pod reschedules, reattaches its EBS volume, and rejoins the cluster
- Node affinity + pod anti-affinity: design the scheduling rules that ensure Kafka brokers, Elasticsearch data nodes, and Patroni replicas never share the same AZ or underlying host
- Operator selection: Strimzi (Kafka) + ECK (Elasticsearch) + CloudNativePG (PostgreSQL) — how do you manage 3 different operators, their CRDs, and upgrade lifecycles simultaneously

---

## Q29 | Security Architecture → IAM + Least Privilege at Scale | Security Architecture

Your AWS environment has grown to 15 accounts, 300 IAM roles, and 80 Lambda functions. A recent security review found 40 roles with `*:*` admin policies, 15 Lambda functions with full S3 access when they only write to one bucket, and no automated mechanism to detect privilege escalation paths. Design an IAM governance architecture that enforces least privilege for all human and machine identities, detects policy drift, and scales to 500 engineers without a security team bottleneck.

**Design constraints to address:**
- IAM Access Analyzer + permission boundaries: design the automated workflow that generates least-privilege policies from CloudTrail data using Access Analyzer and creates a PR for human review before applying
- Permission boundaries for developer-created roles: design the SCP + permission boundary strategy that allows developers to create IAM roles for their Lambda functions but prevents them from creating roles with more permissions than they themselves have
- Cross-account role chaining: design the hub-and-spoke IAM role architecture for 15 accounts using AWS Organizations that allows the platform team to audit all accounts from a central account
- Privilege escalation detection: design the AWS Config rule + Lambda function that detects roles with privilege escalation paths (e.g., `iam:CreatePolicyVersion` + `iam:AttachRolePolicy`) and alerts within 5 minutes

---

## Q30 | Migration Design → Monolith to Microservices | Migration Design

Your 7-year-old Rails monolith handles 25M requests/day, manages user auth, payments, inventory, notifications, and reporting in a single 600K-line codebase with a shared PostgreSQL database. The team has decided to extract microservices incrementally. Three previous extraction attempts failed because the shared database created circular dependencies. Design a 24-month migration strategy that extracts 5 domains into microservices without a big-bang rewrite and without a single hour of planned downtime.

**Design constraints to address:**
- Strangler Fig pattern implementation: design the routing layer (NGINX, API Gateway, or feature flag service) that progressively routes traffic from the monolith to new services, including how you handle partial rollouts
- Shared database decomposition: design the database-per-service migration sequence — including the dual-write period, data sync validation, and the specific order of extraction that avoids circular foreign key dependencies
- Event-driven integration: design the event bus (Kafka or EventBridge) integration for the notifications domain extraction, including how you ensure no notifications are lost during the cutover week
- Monolith performance during migration: how do you prevent the remaining monolith from degrading as you extract services that currently benefit from in-process calls to shared business logic

---

## Q31 | Developer Platform → Self-Service Kubernetes Access | Platform Design

Your 250-engineer organization has a single platform team of 6 managing all Kubernetes access. Engineers open tickets for: namespace creation, RBAC changes, resource quota increases, ingress configurations, and secret provisioning. The ticket backlog is 80 items with a 5-day SLA. Engineers are bypassing the process and making direct changes. Design a self-service Kubernetes governance platform that eliminates the ticket queue for 90% of requests while maintaining security and compliance guardrails.

**Design constraints to address:**
- Crossplane or ACK for self-service infra: design the Composite Resource (XR) abstraction that allows an engineer to request a "database with high availability" without knowing Aurora vs. RDS Multi-AZ
- Approval workflow integration: design the GitOps PR-based approval flow for namespace quota increases — engineer opens a PR, automated policy check runs, team lead approves, ArgoCD applies
- Kyverno policy as self-service guardrails: design policies that auto-generate NetworkPolicies, ResourceQuotas, and LimitRanges when a new namespace is created, removing the need for platform team manual config
- Audit trail: how do you maintain a complete audit log of all self-service actions (who requested, who approved, what was applied) in a queryable format for compliance reviews

---

## Q32 | Cost Optimization → Lambda + Serverless Architecture | Cost Optimization

Your team runs 120 Lambda functions processing a total of 2 billion invocations per month, at a current cost of $45K/month. A cost analysis reveals: 30 functions are over-provisioned (1GB RAM for functions using 128MB), 20 functions have timeouts set to 15 minutes but complete in 30 seconds, and 15 functions are triggered synchronously via API Gateway at 50ms intervals creating a polling pattern. Design a Lambda cost optimization strategy that reduces monthly cost by 50% without degrading function performance or reliability.

**Design constraints to address:**
- AWS Lambda Power Tuning automation: design the CI/CD pipeline step that runs power tuning on every Lambda function change and recommends the optimal memory configuration before deploy
- Polling pattern elimination: redesign the 15 polling Lambda functions using EventBridge Pipes, SQS, or Kinesis Data Streams — show the architecture for the highest-volume use case and the cost comparison
- Graviton2 migration: evaluate ARM64 (Graviton2) vs. x86 for Lambda across the 120 functions — which workload types benefit most, what is the migration risk, and how do you batch the migration
- Lambda SnapStart vs. provisioned concurrency for Java functions with 3-second cold starts: evaluate cost and latency tradeoffs at 10 RPS, 100 RPS, and 1000 RPS

---

## Q33 | HA Architecture → Multi-Region Active-Active | HA Architecture

Your customer-facing authentication service handles 50M login requests/day with a 99.99% SLA (52 minutes downtime/year). It currently runs active-passive in us-east-1/us-west-2. The passive region has never served live traffic, and the last failover test took 18 minutes. Design an active-active multi-region architecture for the auth service that eliminates the cold failover problem, handles session state globally, and achieves the 99.99% SLA.

**Design constraints to address:**
- Global session store: DynamoDB Global Tables vs. Redis Global Datastore vs. JWT-based stateless tokens — evaluate each for 50M logins/day, cross-region consistency, and failover behavior
- Traffic routing: Route 53 latency routing vs. AWS Global Accelerator vs. Cloudflare Load Balancing — evaluate for sub-100ms global login latency and automatic failover within 30 seconds
- Split-brain prevention: in active-active with DynamoDB Global Tables, design the conflict resolution strategy for concurrent writes to the same session record from two regions
- Chaos engineering validation: design the monthly test that sends 100% of traffic to one region for 10 minutes, validates the other region handles full load, and automatically rolls back — without the on-call team being paged

---

## Q34 | Observability → Log Architecture at Scale | Observability Design

Your platform generates 500GB of application logs per day across 80 microservices. Current stack: unstructured logs shipped to CloudWatch Logs, no log correlation across services, and a 90-second delay from log emission to query availability. Engineers use grep-based log searching that times out on anything more than a 1-hour window. A compliance requirement mandates 7-year log retention with the ability to query any log within 4 hours of a legal hold request. Design a log architecture that meets both the operational and compliance requirements.

**Design constraints to address:**
- Structured logging standard: design the JSON log schema with mandatory fields (traceId, spanId, tenantId, service, severity, requestId) and the OpenTelemetry-based enforcement mechanism in CI
- Hot/warm/cold log tiering: design the pipeline from application to Loki (7 days hot) → S3 Parquet via Athena (90 days warm) → S3 Glacier (7 years cold), including the query federation that makes all tiers queryable through one interface
- Log sampling vs. full retention: design a smart sampling strategy that keeps 100% of ERROR/WARN logs and 5% of INFO/DEBUG, reducing volume by 70% while maintaining full incident reconstruction capability
- Legal hold workflow: design the S3 Object Lock + compliance mode workflow that prevents deletion of specific tenant's logs upon receiving a legal hold request, with a 4-hour SLA from request to confirmed hold

---

## Q35 | Migration Design → On-Premises to AWS | Migration Design

Your company runs a 200-server on-premises data center (VMware vSphere) with a $3.2M/year run cost (hardware refresh, co-lo, power). Workloads include: a 50-node Hadoop cluster, 30 LAMP stack applications, an Oracle Database 19c (12TB, 10K TPS), and 40 Windows Server workloads. The data center lease expires in 18 months. Design a cloud migration strategy that migrates all workloads to AWS within 18 months, reduces total cost by 35%, and retires the data center on schedule.

**Design constraints to address:**
- Migration wave planning: use the 7R framework (Rehost, Replatform, Refactor, Retire, Retain, Relocate, Repurchase) and assign each of the four workload categories to the appropriate R — justify the Hadoop cluster decision specifically (S3 + EMR vs. lift-and-shift)
- Oracle Database migration: AWS DMS + Schema Conversion Tool for Oracle → Aurora PostgreSQL — design the parallel-run validation strategy that confirms zero data loss before cutover for a 12TB, 10K TPS database
- MGN (Application Migration Service) for LAMP apps: design the replication, testing, and cutover sequence for 30 servers with sub-1-hour cutover windows per server
- Cost model validation: design the monthly cost comparison report during migration that tracks on-premises run rate vs. AWS spend and validates the 35% reduction target before decommissioning on-premises hardware

---

## Q36 | Kubernetes & EKS → Network Policy Design | Security Architecture

Your EKS cluster runs 80 microservices with no NetworkPolicies — every pod can communicate with every other pod. A lateral movement incident last quarter showed that a compromised frontend pod could reach the payments database directly. The platform team wants to enforce a default-deny posture with explicit allow rules, but there is no inventory of which services actually communicate with each other. Design a NetworkPolicy architecture for 80 services that achieves default-deny without breaking any legitimate traffic.

**Design constraints to address:**
- Traffic discovery: design the Cilium Hubble or Calico flow log analysis pipeline that maps all actual pod-to-pod communication patterns over 30 days and generates a proposed NetworkPolicy set
- Policy rollout strategy: design the shadow mode → audit mode → enforce mode progression for NetworkPolicies, including how you catch false positives before enforcement breaks production
- DNS egress policy: design the FQDN-based egress policy that allows services to call `api.stripe.com` and `smtp.sendgrid.com` by domain name (not IP) using Cilium or Calico DNS policies
- Policy lifecycle management: design the GitOps workflow for NetworkPolicy changes — who can approve, what automated tests validate connectivity after policy changes, and how you handle emergency exceptions

---

## Q37 | Scaling Design → Database Read Scaling | Scaling Design

Your PostgreSQL database (Aurora, db.r6g.8xlarge) serves a social platform with 20M DAU. Reads account for 95% of queries — user profiles, feed generation, content lookups. Current read replica count: 2. During peak hours (7-10pm), p99 query latency spikes to 800ms, and you're seeing read replica CPU at 90%. Feed generation queries are the primary culprit (complex JOINs across 5 tables). Design a read scaling architecture that handles 5x peak traffic growth and brings p99 latency below 100ms.

**Design constraints to address:**
- Read replica scaling: Aurora Auto Scaling for read replicas vs. pre-provisioned additional replicas — evaluate the cold start time of a new Aurora replica (typically 10-15 minutes) against your traffic spike pattern
- Caching layer design: Redis (ElastiCache) cache-aside vs. write-through for user profiles and feed data — design the cache warming strategy for a new user following 500 accounts simultaneously
- Feed generation redesign: evaluate pre-computed feeds (fan-out on write) vs. real-time aggregation (fan-out on read) for a social feed — make the recommendation for your 20M DAU scale and justify the storage cost tradeoff
- Connection management: Aurora Proxy for connection pooling — design the pool sizing per service and the failover behavior when the primary fails and Aurora Proxy needs to reconnect all 2,000 pooled connections

---

## Q38 | CI/CD Pipelines → Feature Flag Architecture | Platform Design

Your organization deploys 80 times per day but still does "big bang" feature releases that require coordinating 5 teams simultaneously. Three product incidents in the past year were caused by features released to 100% of users before adequate soak time. Design a feature flag system that decouples code deployment from feature release, enables per-segment rollouts, and integrates with your CI/CD pipeline and observability stack.

**Design constraints to address:**
- Feature flag system design: LaunchDarkly vs. Unleash vs. custom implementation — evaluate for 200 engineers, 50M users, multi-language SDK support, and cost at scale
- Flag evaluation latency: design the SDK + local cache architecture that ensures flag evaluation adds <1ms to request latency, and define the stale flag behavior when the flag service is unreachable
- Canary release automation: design the automated rollout pipeline that progresses a flag from 1% → 10% → 50% → 100% based on error rate and p99 latency staying within SLO thresholds
- Flag debt management: design the process that automatically creates Jira tickets for feature flags older than 30 days, identifies flags permanently at 100%, and blocks new flag creation if the total flag count exceeds 500

---

## Q39 | Disaster Recovery → Kubernetes Control Plane Recovery | Disaster Recovery

Your EKS cluster control plane is managed by AWS, but etcd corruption or a mass node failure could leave your 500-node cluster in an unrecoverable state. Your critical application manifests live in Git (ArgoCD), but custom operator CRDs, cluster-level RBAC, and admission webhook configurations are not backed up. Design a Kubernetes cluster disaster recovery strategy with RTO < 30 minutes for a full cluster rebuild and RPO < 1 hour for all configuration.

**Design constraints to address:**
- Velero backup strategy: design the backup schedule and scope (namespaced resources, cluster-scoped resources, PV snapshots) with retention policies, and validate that a restore from backup actually works by running monthly restore drills
- Configuration as Code completeness: design the audit that identifies all cluster-level resources (ClusterRoles, ValidatingWebhookConfigurations, PodSecurityPolicies, StorageClasses) not in Git and creates the GitOps migration plan
- Cluster rebuild runbook: design the step-by-step automated runbook (Terraform + ArgoCD bootstrap) that provisions a new EKS cluster, applies all base configuration, and syncs all ArgoCD applications within 30 minutes
- Cross-region cluster pre-warm: design the passive standby cluster in a secondary region that runs at 20% capacity, receives ArgoCD syncs, and can scale to full capacity within 10 minutes of a primary region failure

---

## Q40 | Multi-Tenancy Design → SaaS Cost Attribution | Cost Optimization

Your B2B SaaS platform serves 200 enterprise customers on shared infrastructure. You cannot tell which customer consumes what compute because all workloads run in a shared EKS cluster with no cost tagging at the tenant level. A 3x pricing renegotiation with a large customer is blocked because you cannot show them their actual infrastructure cost. Design a tenant-level cost attribution system for a shared Kubernetes platform that produces per-tenant cost reports accurate within 5%.

**Design constraints to address:**
- OpenCost/Kubecost allocation model: design the cost allocation methodology that splits shared infrastructure costs (ingress controller, monitoring, service mesh) across tenants by fair-share vs. usage-weighted distribution
- Workload tagging strategy: design the namespace label + pod annotation taxonomy that maps 100% of pods to a tenant, team, and environment — and the Kyverno policy that makes untagged pod creation impossible
- Network egress attribution: design the approach to attributing AWS data transfer costs per tenant when multiple tenants share the same NAT Gateway and CloudFront distribution
- Cost showback report: design the monthly automated report that each customer's account manager receives, showing compute, storage, network, and support costs vs. their contracted tier limits

---

## Q41 | Terraform & IaC → Multi-Account Module Strategy | Platform Design

Your organization has grown to 20 AWS accounts (dev, staging, prod per team plus shared services accounts). Every team has their own Terraform modules with no sharing, leading to 15 different implementations of "an S3 bucket with standard security settings." Security patches (like S3 block public access) must be applied 20 times manually. Design a Terraform module ecosystem for 20 accounts and 15 teams that enforces security baselines, enables module reuse, and doesn't create a platform team bottleneck.

**Design constraints to address:**
- Module registry: private Terraform Registry (Terraform Cloud) vs. Git-based module sourcing with semantic versioning — design the module release pipeline including automated testing, version bumping, and changelog generation
- Module abstraction levels: design a three-layer module hierarchy (resource modules → service modules → product modules) with concrete examples of what belongs at each layer for EKS, RDS, and VPC
- Security baseline enforcement: design the module wrapper pattern that adds mandatory tagging, encryption, and logging to every S3 bucket, RDS instance, and EKS cluster without allowing teams to override security-critical defaults
- Breaking change management: when a module's v2.0 changes a required variable, design the migration path that notifies all 15 consuming teams, provides a migration guide, and gives a 30-day deprecation window before v1.x EOL

---

## Q42 | Scaling Design → Event-Driven Architecture | Scaling Design

Your order processing system handles 5K orders/minute at peak. The current architecture is synchronous: API → Rails monolith → PostgreSQL, with each order taking 800ms to process (inventory check, payment, fulfillment, email). The system falls over at 8K orders/minute during flash sales. Design an event-driven order processing architecture that handles 50K orders/minute (10x peak), maintains exactly-once processing semantics, and ensures no order is lost even if any single component fails.

**Design constraints to address:**
- Message broker selection: Kafka vs. SQS + SNS vs. EventBridge for order events — evaluate for exactly-once delivery, ordering guarantees, consumer group semantics, and cost at 50K/minute
- Saga pattern for distributed transactions: design the choreography-based saga for the order processing flow (inventory reserve → payment charge → fulfillment trigger → email send) with explicit compensation actions for each step failure
- Idempotency: design the idempotency key strategy that prevents duplicate order processing when a consumer restarts mid-batch or when SQS delivers a message twice
- Backpressure handling: design the consumer scaling strategy (KEDA + SQS queue depth) that automatically scales processing workers during flash sales while preventing downstream services (payment API) from being overwhelmed

---

## Q43 | Security Architecture → Container Security Posture | Security Architecture

A security scan of your production Kubernetes cluster reveals: 60% of containers run as root, 30% have read-write root filesystems, 15 containers mount the Docker socket, and no pod is subject to a PodSecurityPolicy (deprecated) or PodSecurityAdmission standard. A container escape vulnerability (like CVE-2024-0132) could compromise your entire cluster. Design a container security posture management (CSPM) architecture that enforces container security standards across 80 services without breaking existing deployments.

**Design constraints to address:**
- Pod Security Standards migration: design the namespace-by-namespace migration from no policy to `restricted` PSA standard — define the assessment, remediation, and enforcement sequence over a 90-day timeline
- Kyverno policy set: design the 5 most critical policies for your environment (no root containers, no privileged containers, required securityContext fields, disallowed host namespaces, required image digest pinning) with enforce vs. audit mode rollout plan
- Runtime security: Falco rule design for the top 5 container escape and cryptomining detection patterns — define the alert → quarantine → incident response automation flow
- Image provenance: design the admission webhook (Kyverno + Cosign) that rejects any image not signed by your internal CI pipeline, with an emergency override procedure for incident response

---

## Q44 | Migration Design → EKS Version Upgrade Automation | Migration Design

You manage 8 EKS clusters across 4 AWS accounts. Historically, each cluster upgrade takes 3 days of manual work per cluster (24 days total), and the process is inconsistently documented — each engineer follows their own checklist. AWS releases a new EKS version every 3 months and deprecates old ones in 14 months, meaning you're perpetually behind. Design an automated EKS upgrade pipeline that reduces per-cluster upgrade time to 4 hours and can run across all 8 clusters with a single pipeline trigger.

**Design constraints to address:**
- Pre-upgrade validation automation: design the CI job that runs pluto (deprecated APIs), kube-no-trouble (add-on compatibility), and node readiness checks — and defines a pass/fail gate before the upgrade begins
- Add-on upgrade sequencing: design the dependency-ordered upgrade sequence for: VPC CNI → CoreDNS → kube-proxy → Cluster Autoscaler → EBS CSI → ALB Ingress Controller — and how you handle an add-on that doesn't support the target version yet
- Node group rolling update automation: design the Terraform automation that creates new node groups with the target AMI, cordons and drains old nodes in batches of 10%, and validates pod rescheduling health before proceeding
- Upgrade pipeline state machine: design the Step Functions or GitHub Actions workflow that orchestrates the full upgrade with automatic rollback if node readiness drops below 95% at any step, and sends a Slack progress update every 15 minutes

---

## Q45 | HA Architecture → Redis High Availability | HA Architecture

Your platform uses Redis for 5 use cases: user sessions (must not lose data), rate limiting (can tolerate 30-second loss), distributed locks (must be strongly consistent), leaderboards (can rebuild from database), and pub/sub for real-time notifications (at-most-once delivery acceptable). All 5 use cases share a single ElastiCache Redis cluster (3-node, Multi-AZ). Design separate Redis architectures for each use case that matches the durability and consistency requirements without over-engineering.

**Design constraints to address:**
- Cluster topology per use case: evaluate ElastiCache Cluster Mode Enabled vs. Replication Group vs. single-node for each of the 5 use cases — justify with the specific failure mode each topology protects against
- RedLock vs. ElastiCache for distributed locks: design the distributed lock architecture that handles the split-brain scenario where the Redis primary fails after acquiring a lock but before replicating to the replica
- Session data durability: design the AOF persistence + Redis sentinel setup (or ElastiCache failover) that ensures no sessions are lost during a primary node failure, including the 30-second failover window behavior
- Cost optimization: redesign the architecture to use the smallest appropriate instance type per use case — specifically, which use cases can use a cache.t3.micro vs. requiring cache.r6g.large, and why

---

## Q46 | Observability → SLO Design + Error Budget | Observability Design

Your platform has no formal SLOs. On-call engineers have no objective way to decide whether to escalate an incident or whether a deploy should be rolled back. The business wants 99.9% availability SLAs with customers but the engineering team doesn't know what 99.9% means operationally. Design a complete SLO framework for a platform with 5 critical user journeys (login, search, checkout, upload, notification delivery), including SLI definitions, alert design, and error budget policies.

**Design constraints to address:**
- SLI instrumentation: for each of the 5 user journeys, define the specific SLI metric (availability, latency, correctness), the data source (Prometheus metric, synthetic probe, or log-derived), and the measurement window
- Error budget policy: design the policy that defines what happens when 50% of monthly error budget is consumed (slow down risky deploys), 75% consumed (freeze non-critical deploys), and 100% consumed (all-hands reliability sprint)
- Multi-window multi-burn rate alerts: design the alert matrix (6x burn rate/1h window + 1x burn rate/6h window) for the checkout SLO — show the exact Prometheus expressions and justify the alert thresholds
- SLO review cadence: design the monthly SLO review process — who attends, what data is presented, how you distinguish "SLO breach due to infrastructure" vs. "SLO breach due to bad deployment," and how you feed results back into the next sprint's priorities

---

## Q47 | Platform Design → Developer Productivity Metrics | Platform Design

Your platform team needs to demonstrate ROI for a $2M annual investment in internal tooling. Leadership wants evidence that the IDP, CI/CD improvements, and golden paths are actually improving engineering productivity. Currently there are no DORA metrics tracked, no deployment frequency data, and no way to attribute a business outcome to a platform investment. Design a developer productivity measurement system that produces credible DORA metrics, identifies bottlenecks, and guides platform team investment priorities.

**Design constraints to address:**
- DORA metrics instrumentation: design the data collection pipeline for all four metrics (deployment frequency, lead time for changes, change failure rate, mean time to restore) — specify the exact data sources (GitHub API, PagerDuty API, ArgoCD events, CI webhooks) and the aggregation pipeline
- Metric gaming prevention: design the definitions carefully to prevent gaming — for example, how do you prevent teams from inflating "deployment frequency" by splitting every commit into multiple tiny deploys
- Space framework integration: design the survey + automated metric combination that captures the developer experience dimension (satisfaction, perceived productivity) alongside the objective DORA metrics
- Platform team ROI report: design the quarterly report structure that shows: baseline metrics, current metrics, which platform investments drove which improvements, and the dollar value of productivity gains (using fully-loaded engineer cost as the unit)

---

## Q48 | AWS Infrastructure → Data Lake Architecture | Scaling Design

Your company needs a data lake to ingest data from 50 microservices (Kafka events), 3 operational databases (PostgreSQL via CDC), and 5 SaaS tools (Salesforce, Zendesk, Stripe via API). The data lake must support: SQL analytics by 50 data analysts, ML feature engineering by 10 data scientists, and real-time dashboards with < 5-minute data freshness. Expected data volume: 10TB/day ingestion, 500TB total. Monthly cost budget: $80K. Design the AWS data lake architecture.

**Design constraints to address:**
- Storage layer: S3 with Delta Lake vs. Apache Iceberg vs. Hudi — evaluate for time-travel queries, CDC merge performance, and compatibility with Athena, Spark, and Presto
- Ingestion pipeline: Kafka → Kinesis Data Firehose → S3 for streaming; DMS → S3 for CDC; AWS Glue for SaaS API sources — design the fan-in architecture and the schema registry that prevents schema drift from breaking downstream queries
- Compute layer: Athena (serverless) vs. EMR Serverless vs. Redshift Serverless — design the tiered query compute strategy that uses Athena for ad-hoc SQL, EMR for ML batch jobs, and a materialized view layer for sub-5-minute dashboard freshness
- Cost governance: design the S3 Intelligent-Tiering + Iceberg table maintenance (compaction, snapshot expiry) pipeline that keeps the monthly storage + compute cost within $80K as volume doubles

---

## Q49 | CI/CD Pipelines → Canary Deployment Architecture | CI/CD Design

Your team deploys a customer-facing API that serves 30M requests/day. Historically, 20% of production incidents were caused by deployments. The current deployment process is blue/green with instantaneous traffic cutover, giving no time to detect regressions before 100% of users are affected. Design a canary deployment architecture using Flagger + Istio (or ArgoCD Rollouts) that progressively shifts traffic with automatic analysis and rollback for a Kubernetes-based API service.

**Design constraints to address:**
- Canary analysis metrics: define the exact metrics (error rate, p99 latency, custom business metric like checkout success rate) and thresholds that Flagger uses to decide promote vs. rollback — and how you set baseline values without historical data
- Traffic shifting strategy: design the weight progression (1% → 5% → 25% → 50% → 100%) with analysis intervals, minimum observation periods, and the specific conditions that trigger an immediate emergency rollback vs. a gradual rollback
- Header-based canary for internal testing: design the mechanism that allows QA engineers and internal users to opt into the canary via a request header before any percentage-based traffic shift begins
- Canary with stateful services: how do you handle a canary deployment of a service that also requires a database schema migration — specifically, how do you ensure both canary and stable pods can read/write the same database during the canary window

---

## Q50 | Multi-Tenancy Design → Compliance + Audit Architecture | Security Architecture

Your SaaS platform must achieve SOC 2 Type II, ISO 27001, and GDPR compliance across 200 enterprise tenants. A compliance audit in 3 months requires: complete audit logs for all data access, evidence of encryption at rest and in transit, proof of access review completion, and data residency controls for EU tenants. Your current environment has no centralized audit trail, no data residency enforcement, and access reviews are done manually in spreadsheets. Design the compliance architecture to pass the audit and maintain continuous compliance posture.

**Design constraints to address:**
- Audit log architecture: design the immutable audit trail using AWS CloudTrail + CloudWatch + S3 Object Lock that captures all API calls, data access events, and human actions — with a query SLA of < 4 hours for "show all accesses to tenant X's data in the last 90 days"
- Data residency for EU tenants: design the AWS multi-region architecture with explicit data residency enforcement (S3 bucket policies, RDS region locks, DynamoDB regional tables) that prevents EU tenant data from leaving eu-west-1, including the CI gate that catches residency violations before deploy
- Continuous compliance monitoring: design the AWS Security Hub + Config + Macie pipeline that monitors 25 compliance controls continuously and generates the quarterly SOC 2 evidence package automatically — reducing the manual audit prep from 3 weeks to 2 days
- Access review automation: design the quarterly access review workflow that pulls all IAM, Kubernetes RBAC, and database user privileges, sends structured review requests to team leads, tracks approvals/revocations, and produces the auditor-ready evidence PDF automatically
