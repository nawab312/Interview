## Q1 | Kubernetes → Scheduler Internals | Conceptual

Explain the sequence of events that occurs from the moment a Pod spec is submitted via `kubectl apply` to the moment the container is running on a node. Include the roles of the API server, etcd, the scheduler's filtering and scoring phases, the kubelet, and the container runtime. What happens if the scheduler cannot find a suitable node?

---

## Q2 | AWS → VPC Networking | Scenario

Your application team reports intermittent timeouts connecting from an ECS Fargate service in a private subnet to an RDS PostgreSQL instance in a different private subnet within the same VPC. The connection works fine from a bastion host in a public subnet. Walk through every layer you would inspect — security groups, NACLs, route tables, DNS resolution, and RDS subnet groups — and describe the exact AWS CLI or console commands you would use to isolate the issue.

---

## Q3 | Terraform → State Management | Failure Mode

A colleague runs `terraform apply` on a pipeline while you are simultaneously running it locally against the same workspace. Both runs succeed independently but the state file now reflects only one set of changes. Describe the exact failure mode, what the state file looks like afterward, how Terraform's state locking should have prevented this, and the step-by-step recovery process including `terraform state` subcommands you would use to reconcile the drift.

---

## Q4 | CI/CD & Jenkins → Pipeline Security | Design Tradeoff

Your Jenkins instance stores AWS credentials as global credentials used across 40 pipelines. A security audit flags this as a blast-radius risk. Compare three alternatives — Jenkins credential scoping + folders, OIDC-based dynamic credentials via AWS IAM Roles Anywhere, and a HashiCorp Vault + Jenkins plugin integration — evaluating secret rotation, auditability, blast radius, and implementation complexity. Which would you recommend for a 200-engineer org and why?

---

## Q5 | Linux & Bash → Performance | 2AM Fire

At 2 AM an alert fires: a Linux host's load average has spiked to 80 on a 16-core machine. SSH is slow but reachable. Write the exact sequence of commands you run in the first 5 minutes — covering CPU, memory, I/O, process state (D-state processes), kernel and OOM log inspection — and explain what each output tells you. Include how you distinguish CPU-bound from I/O-bound saturation.

---

## Q6 | Prometheus & Grafana → PromQL | Coding

Write a PromQL query that calculates the 99th percentile request latency per service over the last 5 minutes, using a `http_request_duration_seconds` histogram metric that has labels `service`, `method`, and `status_code`. Then extend the query to alert when the p99 for any service exceeds 500ms for more than 3 consecutive minutes. Show both the recording rule and the alerting rule YAML.

---

## Q7 | ArgoCD → Sync Strategies | Conceptual

Explain the difference between ArgoCD's `Automated` sync with `selfHeal: true` vs `selfHeal: false`, and describe the exact sequence of events when a developer manually patches a Deployment directly in the cluster using `kubectl`. Under what production conditions would you intentionally disable self-healing, and what compensating controls would you put in place?

---

## Q8 | AWS → ECS & Fargate | 2AM Fire

At 2 AM your ECS Fargate service drops from 10 running tasks to 2. The service is behind an ALB. CloudWatch shows no recent deployment. Walk through the exact investigation: ECS service events, stopped task reasons, CloudWatch Container Insights, ALB target group health, and IAM task role permissions. What are the five most common root causes for silent task termination on Fargate and how do you distinguish between them?

---

## Q9 | Kubernetes → Networking | Failure Mode

A Pod in namespace `frontend` cannot reach a Pod in namespace `backend` via its ClusterIP Service. Both Pods are running and their readiness probes pass. Describe every component in the packet path — iptables/IPVS rules, kube-proxy, CoreDNS, and the CNI plugin — and list at least five distinct causes of this failure with the exact `kubectl` and Linux commands that would confirm or rule out each one.

---

## Q10 | GitHub Actions → Secrets & Security | Design Tradeoff

You need to allow GitHub Actions workflows in 15 repositories to push Docker images to ECR and deploy to EKS without storing long-lived AWS credentials in GitHub Secrets. Design the complete OIDC federation setup: the IAM OIDC provider configuration, the trust policy JSON, the permissions boundary, and the workflow YAML snippet that assumes the role. Explain what prevents a workflow in a forked repository from assuming the same role.

---

## Q11 | ELK Stack → Elasticsearch Architecture | Conceptual

Explain how Elasticsearch distributes and replicates data using primary and replica shards. If a 5-node cluster has an index with 3 primary shards and 1 replica each, and 2 nodes fail simultaneously, describe the exact cluster state transition: which shards become unassigned, how the cluster health changes, whether writes are still accepted, and the recovery sequence when nodes rejoin. What is the minimum number of nodes needed to avoid split-brain with the default quorum settings?

---

## Q12 | Python Scripting → Boto3 | Coding

Write a Python script using Boto3 that finds all EC2 instances across all AWS regions that have been stopped for more than 30 days (using the `StateTransitionReason` field or CloudTrail), tags them with `stale=true` and `stale_since=<date>`, and outputs a CSV report with columns: `region`, `instance_id`, `instance_type`, `name_tag`, `stopped_since`, `estimated_monthly_savings`. The script must handle pagination, rate limiting with exponential backoff, and credential errors gracefully.

---

## Q13 | Terraform → Modules | Design Tradeoff

Your team has a monolithic Terraform root module with 500 resources covering networking, compute, and databases for three environments. Describe the tradeoffs of three refactoring strategies: (1) environment-based workspaces, (2) separate state files per layer with remote state data sources, and (3) Terragrunt with DRY configurations. Address state blast radius, dependency management, plan performance, and team collaboration. Which approach scales best to 10 teams and 20 environments?

---

## Q14 | Kubernetes → Security | Config

Write a complete Kubernetes PodSecurityPolicy (or Pod Security Admission configuration for Kubernetes 1.25+) that enforces: non-root user execution, read-only root filesystem, dropped ALL capabilities with only `NET_BIND_SERVICE` added back, no privilege escalation, and seccomp profile `runtime/default`. Show both the policy/config YAML and a sample Pod spec that satisfies it, and one that would be rejected with the reason.

---

## Q15 | AWS → Observability | 2AM Fire

At 2 AM a CloudWatch alarm fires on `5XXErrorRate > 5%` for your API Gateway. Your downstream Lambda functions show no errors in their logs. Walk through the full investigation: API Gateway access logs vs execution logs, Lambda destination vs CloudWatch Logs Insights queries, X-Ray traces, and the difference between integration errors and gateway errors. Provide the exact CloudWatch Logs Insights query you'd run to identify which endpoints are failing and the HTTP status code distribution.

---

## Q16 | CI/CD & Jenkins → Architecture | Design Tradeoff

Compare Jenkins with ephemeral agent pods on Kubernetes (using the Kubernetes plugin) versus a self-hosted GitHub Actions runner fleet on EC2 Auto Scaling Groups for a build system that handles 500 concurrent builds with Docker-in-Docker requirements. Address agent startup latency, Docker socket security, resource bin-packing, secret management, and operational toil. Under what circumstances would you prefer the Jenkins approach in 2024?

---

## Q17 | Linux & Bash → Scripting | Coding

Write a Bash script that monitors a directory for files matching `*.log` older than 24 hours, compresses them with gzip preserving the original filename and timestamp, moves them to a dated archive subdirectory (`./archive/YYYY-MM-DD/`), and logs each operation with a timestamp to `/var/log/log_archiver.log`. The script must be idempotent, handle filenames with spaces, and send a summary email via `sendmail` if more than 100 files were archived in a single run. Include a `--dry-run` flag.

---

## Q18 | ArgoCD → ApplicationSets | Conceptual

Explain how ArgoCD ApplicationSets work and describe the difference between the `List`, `Cluster`, `Git`, and `Matrix` generators. Give a concrete example of using the `Matrix` generator to deploy a microservice to every combination of 5 clusters × 3 environments from a single ApplicationSet manifest. What happens to child Applications when the ApplicationSet is deleted, and how do you control this behavior?

---

## Q19 | AWS → RDS & Databases | Failure Mode

Your RDS Aurora PostgreSQL Multi-AZ cluster undergoes an unplanned failover. Describe the exact sequence: what triggers automatic failover, how long the DNS TTL affects application reconnection, how the writer endpoint changes, and what happens to in-flight transactions. Your application uses a connection pool (PgBouncer). What connection pool settings determine whether applications recover automatically vs hang indefinitely? What CloudWatch metrics would you alarm on to detect prolonged failover?

---

## Q20 | Kubernetes → Workloads | Debug

A Deployment has `replicas: 3` but only 1 Pod is consistently in `Running` state; the other 2 cycle between `Pending` and `ContainerCreating`. The container image exists in the registry. Provide the exact `kubectl` commands to diagnose this, what output from `kubectl describe pod`, `kubectl get events`, and node-level `crictl` commands would reveal, and walk through the five most likely causes: image pull secrets, resource quotas, node taints, PVC binding failures, and init container failures.

---

## Q21 | Prometheus & Grafana → High Availability | Design Tradeoff

You need to run Prometheus in a highly available configuration for a 2000-node Kubernetes cluster generating 5 million active time series. Compare three approaches: (1) Prometheus with Thanos sidecar + object storage, (2) Prometheus with Cortex/Mimir remote write, and (3) Victoria Metrics cluster. For each, address: deduplication of metrics from redundant Prometheus instances, long-term storage cost, query federation, and the operational complexity of the compaction layer. Which would you choose and why?

---

## Q22 | GitHub Actions → Monorepo Patterns | Config

Write a complete GitHub Actions workflow YAML for a monorepo containing services at `services/auth/`, `services/payments/`, and `services/notifications/`. The workflow must: (1) detect which service directories changed using `dorny/paths-filter`, (2) run service-specific tests only for changed services in parallel, (3) build and push Docker images with `sha` and `latest` tags only for changed services, and (4) trigger deployment only when running on `main`. Include proper job dependencies and matrix strategy.

---

## Q23 | ELK Stack → Index Lifecycle Management | Scenario

Your Elasticsearch cluster stores 90 days of application logs across 50 indices totaling 8TB. Hot nodes (NVMe SSD) are at 85% capacity. You need to implement ILM to: move indices to warm tier (HDDs) after 7 days, shrink shards from 5 to 1 after moving to warm, force-merge to 1 segment, and delete after 90 days. Write the complete ILM policy JSON and index template that applies it, and explain what happens to an index that is currently mid-rollover when you apply the new policy.

---

## Q24 | AWS → IAM & Security | Conceptual

Explain the evaluation logic AWS uses to determine whether an API call is allowed or denied, considering: SCPs at the organization level, permission boundaries on the IAM role, identity-based policies on the role, resource-based policies on the target resource, and VPC endpoint policies. Give an example where all five layers are relevant simultaneously (e.g., an EC2 instance with an instance profile trying to write to an S3 bucket in another account), and trace through exactly how AWS evaluates the allow/deny decision.

---

## Q25 | Terraform → CI/CD Integration | Config

Write a complete GitLab CI or GitHub Actions pipeline configuration for Terraform that implements: (1) `terraform validate` and `tflint` on every PR, (2) `terraform plan` with output stored as a PR comment using `tfcmt`, (3) manual approval gate before `terraform apply`, (4) automatic state unlock if a pipeline is cancelled mid-apply, and (5) Sentinel or OPA policy checks before apply. Show the complete YAML and explain how you handle the Terraform state lock during the approval wait period.

---

## Q26 | Kubernetes → Storage | Failure Mode

A StatefulSet with 3 replicas uses PersistentVolumeClaims backed by AWS EBS volumes (gp3). The node hosting Pod `stateful-app-1` is terminated by the ASG. Describe the exact sequence: how the PVC detaches from the old node, the multi-attach prevention issue unique to EBS, how long `volumeattachmenttimedout` can block the Pod from starting on a new node, and what Kubernetes node lifecycle controllers are involved. What specific EKS node termination handler configuration prevents a 10-minute recovery window?

---

## Q27 | Python Scripting → REST APIs | Coding

Write a Python class `K8sAuditAnalyzer` that authenticates to the Kubernetes API using a kubeconfig file or in-cluster service account (auto-detecting which), streams audit log events from the API server, filters for events where `verb` is `delete` and `resource` is `secrets`, and writes matching events to both a local JSON file and posts them to a Slack webhook. The class must handle connection drops with automatic reconnection, and the Slack posting must be rate-limited to max 1 message per 5 seconds. Include proper type hints and unit tests using `unittest.mock`.

---

## Q28 | ArgoCD → Multi-Cluster | Scenario

You manage 6 EKS clusters across 3 AWS accounts (dev, staging, prod) using a single ArgoCD instance running in the management account. Describe the complete architecture: how ArgoCD authenticates to remote clusters using IRSA or kubeconfig secrets, how you structure ApplicationSets to deploy to correct clusters based on environment labels, and what happens when ArgoCD's connection to a remote cluster is interrupted mid-sync. What RBAC model prevents a developer with access to dev ApplicationSets from accidentally deploying to prod?

---

## Q29 | AWS → Cost Optimization | Design Tradeoff

Your AWS bill for EC2 and EKS worker nodes is $80k/month. Identify and quantify the top 5 cost optimization levers: Savings Plans vs Reserved Instances vs Spot for EKS node groups, Karpenter vs Cluster Autoscaler bin-packing efficiency, right-sizing via Compute Optimizer, gp2 to gp3 volume migration, and NAT Gateway data transfer reduction via VPC endpoints. For each, describe the implementation risk, potential savings percentage, and the exact AWS CLI or API calls to gather the data needed to make the decision.

---

## Q30 | Linux & Bash → Text Processing | Coding

Write a Bash script that takes an nginx access log file as input and produces a report showing: (1) top 20 IPs by request count with their percentage of total traffic, (2) HTTP status code distribution as a bar chart using ASCII, (3) top 10 slowest endpoints by average response time (assuming log format includes `$request_time`), and (4) requests per minute as a time-series for the last hour. The script must work with log files larger than 10GB without loading the entire file into memory. Use only standard Unix tools (awk, sed, sort, uniq).

---

## Q31 | Kubernetes → Istio & Service Mesh | Conceptual

Explain how Istio's Envoy sidecar proxy intercepts network traffic without modifying application code. Describe the iptables rules injected by the init container, how mTLS is negotiated between two Pods both with sidecars, what happens to traffic from a Pod that does NOT have a sidecar injected, and how `PeerAuthentication` in `STRICT` mode affects this. If a service mesh operator disables sidecar injection for a namespace, what are the security implications for mTLS and traffic policy enforcement?

---

## Q32 | CI/CD & Jenkins → Plugins & Scaling | Failure Mode

Your Jenkins controller pod in Kubernetes runs out of heap memory and crashes during peak hours. Upon restart, several builds are stuck in `BUILDING` state with their agent pods still running. Describe: (1) how Jenkins recovers build state from its `JENKINS_HOME`, (2) what happens to the orphaned agent pods and how to clean them up, (3) how to tune JVM heap settings for a Kubernetes-deployed Jenkins, and (4) the architectural change (Jenkins Configuration as Code + ephemeral agents) that eliminates this class of failure permanently.

---

## Q33 | ELK Stack → Logstash | Debug

The following Logstash pipeline config is dropping ~15% of log events silently and causing `pipeline.workers` to spike to 100% CPU. Identify all bugs and performance issues, explain the root cause of each, and provide the corrected configuration:

```ruby
input {
  kafka {
    bootstrap_servers => "kafka:9092"
    topics => ["app-logs"]
    codec => "json"
    consumer_threads => 1
  }
}
filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:msg}" }
  }
  date {
    match => [ "timestamp", "ISO8601" ]
    timezone => "UTC"
  }
  if [level] == "DEBUG" {
    drop { }
  }
  mutate {
    remove_field => ["message", "timestamp"]
    gsub => ["msg", "\n", " "]
  }
  ruby {
    code => "event.set('hash', event.to_hash.to_s.hash)"
  }
}
output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
    template_overwrite => true
  }
}
```

---

## Q34 | Prometheus & Grafana → Alerting | Scenario

Your on-call rotation receives 200+ Prometheus alerts per week, 85% of which are either flapping (firing and resolving within minutes) or symptomatic (same root cause triggering 20 different alerts). Design a complete alert noise reduction strategy covering: `for` duration tuning, alert grouping in Alertmanager, inhibition rules to suppress symptoms when the root cause fires, routing based on severity and team ownership, and silence management automation. Provide the Alertmanager YAML for a concrete example involving a database being down triggering alerts in 5 upstream services.

---

## Q35 | AWS → Lambda & Serverless | Failure Mode

A Lambda function processing SQS messages starts seeing high `IteratorAge` on its DLQ. The function's error rate in CloudWatch is near zero, but messages are not being processed. Explain the possible causes: Lambda concurrency throttling and how reserved vs provisioned concurrency interact with SQS triggers, the SQS visibility timeout vs Lambda timeout interaction, Lambda VPC cold start delays causing SQS polling to back off, and the `maxReceiveCount` on the source queue. Provide the exact CloudWatch metrics and SQS queue attributes you'd inspect to isolate the cause.

---

## Q36 | Kubernetes → RBAC | Config

Write Kubernetes RBAC manifests (ServiceAccount, ClusterRole or Role, and binding) for three personas: (1) a CI/CD bot that can create/update/delete Deployments, Services, and ConfigMaps in the `production` namespace but cannot read Secrets, (2) a read-only auditor that can list all resources across all namespaces but cannot exec into Pods, and (3) a namespace admin for `staging` that has full control over that namespace only but cannot escalate privileges by creating new RoleBindings that grant more than they already have. Explain how the escalation prevention works.

---

## Q37 | Terraform → Advanced Patterns | Coding

Write a Terraform module in HCL that provisions an EKS cluster with: a managed node group that uses a launch template for custom AMI and user data, IRSA (IAM Roles for Service Accounts) for a given service account name and namespace, and an aws-auth ConfigMap entry for a given IAM role ARN. The module must accept a `var.environment` variable and use `lifecycle { prevent_destroy = true }` on the cluster resource only in production. Include `output` blocks for cluster endpoint, OIDC provider ARN, and node group role ARN. Use `for_each` for managing multiple node groups passed as a map variable.

---

## Q38 | GitHub Actions → Self-Hosted Runners | Design Tradeoff

You are migrating from GitHub-hosted runners to self-hosted runners on EC2 for compliance reasons (data must not leave your VPC). Compare two architectures: (1) persistent EC2 instances with a runner daemon using `--replace-existing-runner-with-same-name` for hot-standby, versus (2) ephemeral runners using the `actions/actions-runner-controller` (ARC) on EKS with spot instances. Address: runner registration token rotation, job isolation between tenants, autoscaling latency, cost model, and the security implications of `ACTIONS_RUNNER_HOOK_JOB_STARTED`.

---

## Q39 | Linux & Bash → Networking | 2AM Fire

At 2 AM you receive alerts that a Linux host cannot reach external endpoints, but internal cluster traffic is fine. Other hosts on the same subnet are unaffected. Run through your exact diagnostic sequence: checking routing table with `ip route`, testing ICMP vs TCP (`ping` vs `nc`/`curl`), inspecting iptables chains (`INPUT`, `OUTPUT`, `FORWARD`, `POSTROUTING`), checking for asymmetric routing via `ip rule`, DNS resolution with `dig` vs `/etc/resolv.conf`, and identifying if a container networking stack (Docker bridge, CNI) has corrupted the host network namespace.

---

## Q40 | ELK Stack → Query DSL | Coding

Write an Elasticsearch Query DSL JSON body that: (1) retrieves log documents from the past 6 hours where `level` is `ERROR` or `FATAL`, (2) excludes documents where `service` matches the pattern `health-check-*`, (3) must contain the phrase "connection refused" or "timeout" in the `message` field, (4) aggregates results by `service` with a sub-aggregation for `error_code`, (5) returns only the top 5 services by error count with each service's top 3 error codes. Include source filtering to return only `timestamp`, `service`, `level`, and `message` fields.

---

## Q41 | ArgoCD → Secrets Management | Design Tradeoff

ArgoCD Git repositories cannot store Kubernetes Secret manifests in plaintext. Compare four secrets management approaches for GitOps: (1) Sealed Secrets (Bitnami), (2) External Secrets Operator with AWS Secrets Manager, (3) Vault Agent Injector with AppRole authentication, and (4) SOPS with AWS KMS. For each, evaluate: the threat model (what happens if the Git repo is compromised), key rotation UX, operator complexity, and behavior when the secrets backend is temporarily unavailable. Which approach handles ephemeral environments (created/destroyed per PR) most gracefully?

---

## Q42 | AWS → SQS & SNS | Scenario

You have an SNS topic fanning out to 8 SQS queues consumed by different microservices. One consumer starts processing messages at 1/10th the normal rate due to a slow downstream database. The SQS queue for that consumer grows to 500,000 messages. Describe the cascading effects: SNS delivery retry behavior, the impact of message retention period, how SQS message visibility timeout interacts with slow consumers, whether backpressure propagates to the publisher, and the architectural changes (dead-letter queues, consumer scaling, queue-depth-based autoscaling using CloudWatch + KEDA) that prevent this scenario.

---

## Q43 | Kubernetes → Autoscaling | Conceptual

Explain the interaction between the Horizontal Pod Autoscaler (HPA), Vertical Pod Autoscaler (VPA), and Cluster Autoscaler (CA). Describe the race condition that occurs when HPA scales up Pods but CA hasn't yet provisioned nodes (pending pods), how VPA admission webhook conflicts with HPA when both target CPU, the `minReplicas` implication during a VPA in-place update, and how Karpenter's consolidation behavior can evict Pods that HPA just scaled up. Under what conditions do all three autoscalers working simultaneously cause oscillation?

---

## Q44 | Python Scripting → Error Handling & Testing | Coding

Write a Python module `ecs_deployer.py` with a function `deploy_service(cluster: str, service: str, image_uri: str, wait_timeout: int = 300) -> dict` that: (1) updates an ECS service's task definition with the new image URI, (2) triggers a deployment, (3) polls the service's deployment status every 10 seconds until the deployment completes or times out, (4) returns a dict with `status`, `duration_seconds`, `new_task_def_arn`, and `failed_task_arns`. Write corresponding pytest tests that mock Boto3 calls, test the timeout behavior, test partial failure (some tasks fail to start), and verify the returned dict schema. No external libraries beyond `boto3` and `pytest`.

---

## Q45 | CI/CD & Jenkins → Modern CI/CD Ecosystem | Design Tradeoff

Your organization is evaluating replacing Jenkins with one of: GitHub Actions, GitLab CI, Tekton, or Argo Workflows for a platform serving 500 developers building 200 microservices with complex DAG-style build pipelines (some builds fan out to 50 parallel jobs and fan back in). Evaluate each option on: pipeline-as-code expressiveness, DAG support, artifact caching, secrets management, self-hosted runner security model, and vendor lock-in. What are the hidden migration costs when moving 300 existing Jenkins pipelines?

---

## Q46 | Kubernetes → CRDs & Operators | Conceptual

Explain the Kubernetes Operator pattern: how a custom controller uses the informer-reflector-workqueue architecture to reconcile desired state in a CRD with actual cluster state. Describe what happens when the reconcile loop returns an error, how exponential backoff is implemented, what `Generation` vs `ResourceVersion` vs `ObservedGeneration` mean in a CRD status subresource, and why storing derived/computed state in the CRD spec (rather than status) is an antipattern. Give a concrete example of a reconcile loop that could cause infinite churn if not implemented carefully.

---

## Q47 | AWS → EKS | Config

Write the complete `eksctl` config file or Terraform + Helm combination to provision an EKS cluster with: (1) private API server endpoint only, (2) managed node groups using Bottlerocket AMI with IMDSv2 enforced and instance metadata hop limit set to 1, (3) IRSA enabled with the OIDC provider, (4) VPC CNI with custom networking to use secondary CIDR for Pod IPs, (5) aws-auth ConfigMap granting access to two IAM roles (read-only and admin), and (6) cluster logging enabled for `api`, `audit`, `authenticator`, and `controllerManager`. Show the complete config.

---

## Q48 | Prometheus & Grafana → Instrumentation | Coding

Write a Python Flask application that exposes Prometheus metrics including: (1) a `Counter` for HTTP requests labeled by `method`, `endpoint`, and `status_code`, (2) a `Histogram` for request duration with custom buckets `[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5]` labeled by `endpoint`, (3) a `Gauge` that reflects the current number of active database connections updated via a background thread every 30 seconds, and (4) an `Info` metric exposing app version and git commit SHA. The `/metrics` endpoint must be excluded from the request counter and duration histogram. Use the official `prometheus_client` library.

---

## Q49 | Linux & Bash → Storage & I/O | Scenario

A production database host shows periodic latency spikes every 5 minutes lasting 30 seconds each. The application team reports random slow queries correlating exactly with the spikes. Describe your investigation: using `iostat -x 1`, `iotop`, `blktrace`/`blkparse`, `dmesg` for I/O errors, checking for LVM snapshot activity, identifying write-back cache pressure with `/proc/sys/vm/dirty_ratio`, and distinguishing between filesystem-level issues (ext4 journal commits, XFS checkpointing) vs hardware-level issues (disk firmware, RAID rebuild). What specific output would confirm each hypothesis?

---

## Q50 | ArgoCD → Rollbacks | 2AM Fire

At 2 AM a bad deployment reaches production via ArgoCD. The application is throwing 500 errors. You need to rollback immediately. Walk through the exact steps: using `argocd app history`, `argocd app rollback`, understanding why ArgoCD rollback and GitOps philosophy conflict, what happens to the Git repo after a rollback (is it still ahead of cluster state?), how `selfHeal: true` will immediately re-apply the broken version if you don't also update Git, and the procedure for a safe rollback that doesn't fight the reconciler. Include the exact CLI commands.

---

## Q51 | AWS → CloudFront & CDN | Failure Mode

Your CloudFront distribution starts returning stale content 48 hours after you deployed new static assets to S3. The S3 bucket shows the new files. Describe every caching layer involved: CloudFront edge cache TTL vs origin cache-control headers vs S3 object metadata, how CloudFront's `Cache-Control: max-age` and `s-maxage` differ, what happens when you issue an invalidation vs just update the file, how versioned filenames eliminate this problem entirely, and why `/*` invalidations cost money and degrade CloudFront performance. Include the exact AWS CLI command to create a targeted invalidation.

---

## Q52 | GitHub Actions → Observability | Scenario

Your GitHub Actions workflows are becoming unreliable: jobs randomly fail with no clear error, workflow durations vary by 300% between identical runs, and you have no visibility into which step consumes most time. Design a complete observability solution: using `actions/github-script` to post job summaries, exporting step timings to a custom Prometheus pushgateway via `curl`, creating a Grafana dashboard tracking p95 workflow duration per repository and workflow name, and setting up alerting when any workflow's p95 exceeds 2× its historical baseline. Include the step-level YAML needed to instrument an existing workflow.

---

## Q53 | Kubernetes → Pod Disruption & Eviction | Design Tradeoff

Explain the difference between a PodDisruptionBudget (PDB), node eviction via the kubelet (eviction manager), preemption by the scheduler, and manual deletion. Your stateful application has a PDB with `minAvailable: 2` and 3 replicas. Describe what happens when: (1) you run `kubectl drain` on a node, (2) the node runs out of memory and the eviction manager kicks in, (3) a higher-priority Pod is scheduled and needs the node's resources. In which of these three scenarios does the PDB protect your application? Where are the gaps?

---

## Q54 | ELK Stack → Elasticsearch Sharding | Design Tradeoff

You are designing an Elasticsearch index strategy for a high-cardinality logging use case: 50 microservices each generating 100,000 log events/minute, events must be searchable for 30 days, and queries are always scoped to a single service and time window. Compare three indexing strategies: (1) one index per service per day (1,500 indices), (2) one index per day with routing by service, and (3) data streams with rollover based on size. Address: shard count explosion, cross-shard query overhead, index template management, ILM compatibility, and Elasticsearch's cluster state memory cost per shard (roughly 10KB/shard in master heap).

---

## Q55 | Terraform → Lifecycle Rules | Debug

The following Terraform configuration causes a perpetual diff on every `terraform plan` even though the infrastructure was not changed. Identify all issues, explain why each causes a perpetual diff, and provide the corrected code:

```hcl
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Web security group"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "web-sg"
    LastUpdated = timestamp()
    Environment = var.environment
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.latest.id
  instance_type = "t3.medium"
  user_data     = <<-EOF
    #!/bin/bash
    echo "Hello World" > /tmp/test.txt
  EOF

  root_block_device {
    volume_size = 20
  }

  lifecycle {
    ignore_changes = [ami]
  }
}

resource "aws_autoscaling_group" "web" {
  desired_capacity = 3
  max_size         = 10
  min_size         = 1

  tag {
    key                 = "Name"
    value               = "web-asg-instance"
    propagate_at_launch = true
  }
}
```

---

## Q56 | Python Scripting → Async Patterns | Coding

Write an async Python script using `asyncio` and `aiohttp` that checks the health of 200 internal service endpoints (provided as a list of URLs in a YAML config file) concurrently with a max concurrency of 20, times out individual requests after 5 seconds, retries failed requests up to 3 times with jitter-based backoff, and produces a structured JSON report with: `url`, `status` (`healthy`/`degraded`/`unreachable`), `response_time_ms`, `http_status_code`, `error_message`. The script must complete within 60 seconds regardless of how many services are degraded. Include a `--alert-webhook` flag that POSTs unhealthy services to a Slack webhook.

---

## Q57 | AWS → DynamoDB | Conceptual

Explain the internal architecture of DynamoDB: partition key hashing to partition routing, the role of DynamoDB's storage nodes and log-structured merge trees, how conditional writes achieve optimistic concurrency, and the difference between eventually consistent and strongly consistent reads at the storage layer. Describe a hot partition scenario: what causes it, how DynamoDB's adaptive capacity works, what the `ConsumedCapacityUnits` vs `ProvisionedThroughputExceededException` metrics reveal, and what partition key design changes permanently resolve it.

---

## Q58 | Kubernetes → Observability | Config

Write a complete Prometheus ServiceMonitor CRD manifest and corresponding Kubernetes Service manifest to scrape metrics from a Deployment `payment-service` in namespace `payments`. The application exposes metrics on port 8080 at path `/internal/metrics`, requires a bearer token (stored in a Secret `payment-service-metrics-token`) for authentication, and should be scraped every 15 seconds with a 10-second scrape timeout. Also write the PrometheusRule CRD manifest with an alerting rule that fires when `payment_processing_errors_total` rate over 5 minutes exceeds 0.1 per second.

---

## Q59 | CI/CD & Jenkins → Design Patterns | Coding

Write a Jenkins shared library (`vars/deployToEks.groovy`) that can be called from Jenkinsfiles as `deployToEks(cluster: 'prod-us-east-1', namespace: 'payments', imageTag: params.IMAGE_TAG)`. The library must: (1) authenticate to EKS using `aws eks update-kubeconfig`, (2) run a Helm upgrade with `--atomic` and `--timeout 5m`, (3) on failure, capture the last 50 log lines from the failed Pod and post them as a Jenkins build artifact and a Slack notification, (4) on success, post a deployment notification with the image tag and Git commit SHA to Slack. Handle missing kubeconfig and Helm binary gracefully.

---

## Q60 | Prometheus & Grafana → Loki & Logs | Scenario

You are replacing ELK with the LGTM stack (Loki, Grafana, Tempo, Mimir) for a 100-node Kubernetes cluster. Loki is configured with a filesystem backend. One week in, Loki query performance degrades: simple label queries take 30+ seconds. Walk through the diagnosis: checking Loki's query planner logs, understanding how Loki's label index (TSDB) differs from full-text inverted indices, why high cardinality labels (`pod_name`, `trace_id`) in log streams destroy performance, the chunk cache hit rate, and why switching to object storage (S3) with a compactor improves this. What label cardinality limits would you enforce?

---

## Q61 | AWS → VPC & Transit Gateway | Design Tradeoff

You are designing network connectivity for 20 AWS accounts across 4 regions, each account with 3 VPCs (dev/staging/prod). Compare: (1) VPC Peering mesh (requires N×(N-1)/2 peering connections), (2) Transit Gateway per region with peering between TGWs, and (3) AWS Cloud WAN with a global network policy. Address: route table management at scale, transitive routing limitations of VPC peering, bandwidth limits per attachment on TGW, the cost model (TGW attachment + data processing fees), and security segmentation via TGW route table domains. Which architecture supports adding a 21st account in under 30 minutes?

---

## Q62 | ArgoCD → Health Checks | Debug

An ArgoCD Application shows `Degraded` health status even though all Pods are Running and Ready. The sync status shows `Synced`. Identify the possible causes: custom resource health checks not defined in `resource.customizations`, a Deployment's `availableReplicas` temporarily less than `replicas` during a rollout, a Job that completed successfully but ArgoCD interprets as failed, and a Hook resource that failed cleanup. Provide the ArgoCD `configmap` YAML to add a custom health check for a CRD `type: MyCustomResource` that returns `Healthy` when `.status.phase == "Ready"`.

---

## Q63 | Linux & Bash → Process Management | 2AM Fire

At 2 AM a critical Java service on a Linux host enters a state where it consumes 100% CPU on all cores but processes no requests (all requests timeout). The JVM is running but unresponsive. Describe your exact response: getting a thread dump without killing the process (`kill -3` or `jstack`), using `jcmd` to diagnose GC pressure, using `top -H -p <pid>` to identify hot threads by TID, converting TID to hex for correlation with thread dump, identifying deadlocks vs GC storms vs infinite loops from the thread dump output, and making the kill/restart decision. What data do you preserve before restarting?

---

## Q64 | Kubernetes → Docker & Images | Design Tradeoff

You are auditing your organization's Docker image build practices. 40% of production images are over 2GB and built on `ubuntu:latest`. Propose a complete image optimization strategy: multi-stage builds with distroless or scratch base images, layer ordering for cache efficiency, `.dockerignore` patterns, BuildKit secret mounts vs ARG for credentials, SBOM generation with `syft`, vulnerability scanning with `trivy` in CI, and image signing with `cosign`. For a Node.js web application, demonstrate the before/after Dockerfile showing the size reduction and the specific optimizations applied.

---

## Q65 | ELK Stack → Security & RBAC | Scenario

Your Elasticsearch cluster is exposed to 50 application teams who should only see logs from their own services. Currently there is a single superuser used by all applications. Design the complete RBAC model using Elasticsearch native security: index-level privileges, document-level security (DLS) to filter by `service_name` field, field-level security (FLS) to hide `user.email` from non-admin roles, API key rotation strategy, and how Kibana Spaces map to Elasticsearch privileges. Show the Elasticsearch role JSON for an application team role that can read only their service's indices and cannot delete or modify mappings.

---

## Q66 | Python Scripting → File & OS Operations | Coding

Write a Python script `k8s_log_collector.py` that accepts a Kubernetes namespace as a CLI argument and: (1) uses the `kubernetes` Python client to list all Pods, (2) for each Pod, streams the last 5000 lines of logs from each container using the async log streaming API, (3) writes logs to `./logs/<namespace>/<pod-name>/<container-name>.log`, (4) for Pods with multiple containers, collects all containers in parallel, (5) creates a compressed tarball `<namespace>-logs-<timestamp>.tar.gz` when complete, and (6) prints a summary table showing pod name, container count, log lines collected, and any containers that failed. Use `argparse` for CLI, handle `ApiException` with meaningful error messages.

---

## Q67 | AWS → SNS & EventBridge | Design Tradeoff

Compare AWS SNS fan-out vs EventBridge event bus for routing events from a microservices platform where: 30 producers emit 15 different event types, 40 consumers need subset filtering (e.g., only `OrderCreated` events for orders over $1000 in the EU region), and event schema evolution must not break existing consumers. Address: SNS filter policy limits (5 filter conditions, limited operators) vs EventBridge rule pattern matching (content-based routing, `exists` checks, numeric ranges), schema registry and backward compatibility, dead-letter queue support, and the cost model at 100M events/day.

---

## Q68 | Kubernetes → Cluster Operations | 2AM Fire

At 2 AM you receive alerts that `kube-apiserver` is returning 503s intermittently. `kubectl` commands succeed roughly 1 in 3 attempts. You have out-of-band SSH access to the control plane nodes. Walk through the diagnosis: checking apiserver Pod logs for connection refused vs timeout vs OOMKill, inspecting etcd cluster health (`etcdctl endpoint health`, `etcdctl endpoint status`), checking if etcd is experiencing leader elections via `etcdctl` metrics, identifying if apiserver is overwhelmed by checking `--max-requests-inflight` and `--max-mutating-requests-inflight` in current load, and the immediate remediation steps that don't require cluster downtime.

---

## Q69 | Terraform → Security | Config

Write a Terraform configuration that provisions an S3 bucket for Terraform state storage with all security controls: versioning enabled, server-side encryption with a customer-managed KMS key (include the KMS key resource with automatic rotation), bucket policy that denies all access unless the request uses SSL and denies `s3:DeleteObject` except from a specific admin role ARN, S3 Block Public Access all four settings, access logging to a separate logging bucket, and a DynamoDB table for state locking. Parameterize the admin role ARN and the KMS key deletion window. Include all necessary IAM and KMS key policies.

---

## Q70 | Prometheus & Grafana → Exporters | Scenario

Your organization runs 200 MySQL instances across RDS and self-managed EC2. You need to deploy the `mysqld_exporter` at scale: describe the deployment architecture for RDS instances (cannot install sidecar, must use network reachability), how you manage exporter credentials without hardcoding passwords (using AWS Secrets Manager and a credentials renewal script), how to configure the exporter's `--collect.*` flags to minimize performance impact on production databases, how Prometheus service discovery finds all 200 exporters via EC2 SD or Consul, and how you alert on replica lag, connection pool exhaustion, and slow query count using PromQL.

---

## Q71 | GitHub Actions → Migration | Scenario

You are migrating 150 CircleCI pipelines to GitHub Actions. Each CircleCI config uses: orbs (reusable config packages), parallelism for test splitting, custom Docker executors, contexts for secrets, and approval jobs for production deployment gates. Map each CircleCI concept to its GitHub Actions equivalent, identify features that have no direct equivalent (specifically CircleCI's test splitting with `circleci tests split`), describe what you would build custom (a composite action or reusable workflow to replace a CircleCI orb), and create a migration risk matrix covering pipelines that use CircleCI's SSH debug feature and parallelism > 20.

---

## Q72 | Linux & Bash → Security & Permissions | Config

Write a Bash script that hardens a fresh Ubuntu 22.04 server following CIS Level 1 benchmarks for these specific items: (1) disables root SSH login and password authentication, (2) configures `auditd` rules to log all writes to `/etc/passwd`, `/etc/shadow`, and `/etc/sudoers`, (3) sets `umask 027` system-wide, (4) restricts `cron` to only the `root` user, (5) enables and configures `ufw` to allow only ports 22, 80, and 443, (6) disables unused kernel modules (`usb-storage`, `cramfs`, `freevxfs`) via `/etc/modprobe.d/`, and (7) configures `sysctl` parameters for TCP SYN flood protection and ICMP redirect rejection. The script must be idempotent.

---

## Q73 | AWS → Kinesis & Streaming | Failure Mode

A Kinesis Data Streams consumer (KCL-based) falls behind: `GetRecords.IteratorAgeMilliseconds` grows from 0 to 4 hours over 6 hours. The shard count is 20, and the consumer has 20 worker threads. Describe the failure analysis: how KCL distributes shards among workers, what causes one shard to "poison" a worker thread (slow deserialization, downstream write bottleneck), how KCL's lease stealing mechanism works and why it may not help if the problem is per-record processing time, the interaction between Kinesis's 7-day retention and iterator expiration, and the autoscaling remediation (resharding vs consumer parallelization via Enhanced Fan-Out).

---

## Q74 | Kubernetes → Troubleshooting | Debug

A CronJob runs every 5 minutes and creates Jobs, but the Jobs are not running their Pods. The Jobs show status `Complete: 0/1` immediately after creation without any Pod appearing in `kubectl get pods`. Identify all the possible causes: `startingDeadlineSeconds` expiration causing the Job to immediately fail, `activeDeadlineSeconds` set to 0, the Job's Pod template having a `RestartPolicy` of `Always` (invalid for Jobs), the namespace having a `LimitRange` that prevents Pod scheduling, and a misconfigured `imagePullPolicy: Never` with the image not present on any node. Provide the exact `kubectl` commands to diagnose each case.

---

## Q75 | ArgoCD → Notifications | Config

Write the complete ArgoCD notifications configuration (ConfigMap and Secret) to send Slack notifications for the following events: (1) application sync succeeded — include the app name, target revision (Git SHA), and author of the commit, (2) application health degraded — include the app name and a link to the ArgoCD UI, (3) sync failed — include the error message and the specific resources that failed. The Slack webhook URL must come from a Kubernetes Secret. Include the `subscriptions` annotation for an Application named `payment-service`. Show both the trigger definitions and the template YAML with Go templating for dynamic fields.

---

## Q76 | AWS → CloudFormation vs Terraform | Design Tradeoff

A regulated financial services company is debating CloudFormation vs Terraform for their entire AWS infrastructure spanning 15 accounts and 3 regions. They require: audit trails for all infrastructure changes, policy enforcement (no public S3 buckets, required tags), drift detection, and integration with their ServiceNow ITSM for change management. Evaluate both on: state management model differences (CFN stack vs TF state file), CloudFormation StackSets vs Terraform workspaces for multi-account, CFN Hooks vs Sentinel/OPA for policy, and the specific scenario of managing AWS resources not yet supported by the provider. What would change your recommendation?

---

## Q77 | Python Scripting → Data Parsing & CLI Tools | Coding

Write a Python CLI tool `awscost` using `click` that: (1) queries AWS Cost Explorer API for the last 30 days of costs grouped by `SERVICE` and `LINKED_ACCOUNT`, (2) supports `--output` flag with options `table` (rich terminal table), `csv`, and `json`, (3) supports `--threshold` flag to highlight services where spend increased more than X% versus the prior 30-day period, (4) supports `--forecast` flag to add a 30-day forecast using the Cost Explorer forecast API, and (5) caches results locally for 1 hour to avoid repeated API calls. The tool must work with AWS SSO profiles and handle the case where Cost Explorer data has up to 24-hour lag. Include proper `--help` text and error messages.

---

## Q78 | Kubernetes → EKS Specifics | 2AM Fire

At 2 AM EKS nodes start failing to join the cluster. `kubectl get nodes` shows them as `NotReady`. The nodes boot successfully (EC2 console shows running) and you can SSH in. On the node, `journalctl -u kubelet` shows `Failed to get node info: nodes "ip-10-0-1-45.ec2.internal" is forbidden`. Walk through the investigation: checking the `aws-auth` ConfigMap for the node IAM role mapping, verifying the node's IAM instance profile, checking if the node's bootstrap script correctly called `aws eks update-kubeconfig`, confirming the IAM role has the `AmazonEKSWorkerNodePolicy` and `AmazonEC2ContainerRegistryReadOnly` managed policies, and the exact fix for each root cause.

---

## Q79 | ELK Stack → Beats & Data Collection | Config

Write a complete Filebeat configuration to collect logs from: (1) Docker containers on the host using the `docker` input (auto-discover containers, add `container.name` and `container.image.name` metadata), (2) `/var/log/nginx/access.log` using the `nginx` module with custom log format parsing, (3) a multiline Java stack trace from `/var/log/myapp/app.log` that correctly reassembles lines starting with whitespace or `Caused by:` into single events. The output must go to Logstash on port 5044 with TLS mutual authentication (client certificate paths configurable via environment variables). Include the `setup.template` and `setup.ilm` configuration to auto-create the index template.

---

## Q80 | CI/CD & Jenkins → Security | Scenario

A security team audit reveals that your Jenkins master has 15 global credentials including AWS access keys, GitHub tokens, and SSH private keys — all used by pipelines from different teams. A developer reports that a malicious Jenkinsfile could potentially exfiltrate credentials via `withCredentials` block and then make arbitrary HTTP calls. Design the complete remediation: Jenkins credential scoping to folders, using credentials IDs that cannot be enumerated without explicit permission, restricting `withCredentials` usage via Script Security plugin approved signatures, adding a Network policy to block outbound connections from build agents to non-approved endpoints, and audit logging for credential access.

---

## Q81 | AWS → Route 53 & DNS | Failure Mode

Your application uses Route 53 health checks and DNS failover between a primary ALB in `us-east-1` and a secondary ALB in `us-west-2`. The primary region experiences a partial outage: the ALB is up but one of three AZs is degraded. Health checks show the primary as healthy (because 2/3 AZs respond). Users in the degraded AZ experience 30% of requests failing. Explain why Route 53 health checks failed to trigger failover, how you would configure more granular health checks (application-level checks using calculated health checks and individual endpoint health checks), the TTL implication during failover, and the Route 53 resolver query logging needed to verify failover is working.

---

## Q82 | Prometheus & Grafana → TSDB Internals | Conceptual

Explain how Prometheus's TSDB stores time series data: the WAL (Write-Ahead Log) and its role in crash recovery, how data is organized into 2-hour in-memory chunks before being compacted to disk blocks, the block structure (`chunks/`, `index`, `tombstones`, `meta.json`), how the compaction process merges blocks to improve query performance and reclaim space from deleted series, and what `--storage.tsdb.retention.size` vs `--storage.tsdb.retention.time` means operationally. At what point does adding more `label cardinality` cause Prometheus OOM, and how do recording rules mitigate this?

---

## Q83 | Linux & Bash → Scheduling & Automation | Coding

Write a Bash script `db_backup.sh` that: (1) connects to a PostgreSQL database using credentials from environment variables (`DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `PGPASSWORD`), (2) performs a `pg_dump` in custom format to a temp file, (3) encrypts the dump using `gpg --symmetric` with a passphrase from `GPG_PASSPHRASE` env var, (4) uploads the encrypted file to S3 at `s3://$BACKUP_BUCKET/postgres/$DB_NAME/$(date +%Y/%m/%d)/$DB_NAME-$(date +%H%M%S).dump.gpg` with server-side encryption, (5) verifies the upload by comparing `md5sum` of local file with S3 ETag, (6) deletes the temp file, and (7) exits with appropriate codes. Include error handling that cleans up temp files on failure.

---

## Q84 | Kubernetes → Network Policies | Config

Write Kubernetes NetworkPolicy manifests to implement this security model for namespace `payments`: (1) deny all ingress and egress by default, (2) allow ingress to Pods labeled `app: payment-api` on port 8080 only from Pods labeled `app: api-gateway` in namespace `ingress`, (3) allow egress from `app: payment-api` to Pods labeled `app: postgres` in namespace `databases` on port 5432 only, (4) allow egress from all Pods in `payments` namespace to `kube-dns` (CoreDNS) on UDP/TCP port 53, (5) allow egress from `app: payment-api` to any external IP on port 443. Explain why DNS egress must be explicitly allowed and what breaks if you forget it.

---

## Q85 | AWS → Security Hub & GuardDuty | Scenario

GuardDuty fires an alert: `UnauthorizedAccess:IAMUser/MaliciousIPCaller.Custom` for an IAM user making `AssumeRole` calls from a known malicious IP. You are the on-call security engineer. Describe the complete incident response: immediate containment (adding an explicit Deny policy vs deactivating access keys vs disabling the IAM user — tradeoffs of each), using CloudTrail to identify all API calls made in the 24 hours prior, checking for newly created IAM users, roles, or access keys that may indicate persistence, checking for S3 exfiltration via CloudTrail `GetObject` calls, and the post-incident steps to understand the initial access vector.

---

## Q86 | ArgoCD → App of Apps | Design Tradeoff

Compare two ArgoCD patterns for managing 80 applications across 4 clusters: (1) App of Apps pattern where a root Application manages child Application manifests stored in Git, versus (2) ApplicationSets with a Git directory generator. For each, describe: how adding a new application works operationally, what happens when the parent App or ApplicationSet is accidentally deleted, how RBAC limits which teams can modify which applications, and how you handle applications with cross-cluster dependencies (e.g., a cert-manager Application must deploy before the application that uses its CRDs). Which pattern better supports a golden path IDP (Internal Developer Platform)?

---

## Q87 | Python Scripting → Subprocess & OS | Coding

Write a Python module `helm_deployer.py` with a class `HelmDeployer` that wraps the `helm` CLI. It must implement: `upgrade_install(release, chart, namespace, values_files, set_values, dry_run=False)` that constructs and executes the `helm upgrade --install` command, captures both stdout and stderr separately, streams output to the logger in real-time (not buffered), returns a `DeployResult` dataclass with `returncode`, `stdout`, `stderr`, `duration_seconds`, and `release_info` (parsed from `helm get values --output json`). The class must validate that the `helm` binary exists at init time and raise a clear `HelmNotFoundError`. Include a context manager that automatically rolls back on exception. No shell=True.

---

## Q88 | AWS → MSK & Kafka | Failure Mode

Your Amazon MSK cluster with 3 brokers and replication factor 3 has one broker fail (broker-3 goes offline). Kafka's partition leadership election begins. Describe: which partitions lose their leader and how long the election takes (unclean vs clean leader election), how the producer's `acks=all` (or `acks=-1`) behaves during the election window, the role of `min.insync.replicas` in determining whether producers can continue writing, how consumer groups detect the leader change and trigger rebalance, and what the MSK console shows during this period. After broker-3 restarts, how does partition reassignment back to the preferred leader work, and do you need to trigger it manually?

---

## Q89 | Kubernetes → Admission Controllers | Conceptual

Explain the Kubernetes admission controller pipeline: the order in which mutating and validating admission webhooks fire relative to built-in admission plugins, how a `MutatingWebhookConfiguration` can inject a sidecar into every Pod, the `failurePolicy: Fail` vs `failurePolicy: Ignore` implication when the webhook server is down, the `namespaceSelector` and `objectSelector` fields and how they are used to exclude system namespaces, and how `reinvocationPolicy: IfNeeded` prevents infinite loops when one mutating webhook's output triggers another. What is the blast radius if an admission webhook with `failurePolicy: Fail` and `matchPolicy: Equivalent` is deployed cluster-wide with a bug?

---

## Q90 | CI/CD & Jenkins → Pipeline Optimization | Scenario

Your CI pipeline for a monorepo takes 45 minutes end-to-end: 5 min checkout, 10 min dependency install, 15 min compile, 8 min unit tests, 7 min integration tests. Your team runs 50 builds/hour. Identify and implement optimizations: Gradle/Maven build cache with a remote cache server, Docker layer caching via `--cache-from` and BuildKit, incremental compilation detection using affected module analysis, parallelizing the unit and integration test stages, and pre-building a "dependencies layer" Docker image that only rebuilds when the lockfile changes. Provide before/after pipeline YAML and estimate the time reduction per stage with realistic assumptions.

---

## Q91 | ELK Stack → OpenSearch Differences | Design Tradeoff

Your organization must decide whether to migrate from Elasticsearch 7.10 (OSS) to OpenSearch 2.x or upgrade to Elasticsearch 8.x. A key concern is that you use: cross-cluster replication, the anomaly detection plugin, `_security` API for RBAC, and Kibana dashboards built over 3 years. For each capability, state whether it is available in both, only in one, or incompatible at the API level. Describe the specific breaking changes in ES 8.x that require index migration (mappings, security model), and the specific OpenSearch divergences that prevent using official Elastic clients. What is the irreversible decision in this migration?

---

## Q92 | AWS → Auto Scaling & EC2 | 2AM Fire

At 2 AM an Auto Scaling Group stops launching new instances. `DesiredCapacity` is 20, `InService` is 8. CloudWatch shows `GroupInServiceInstances` dropping. The ASG's activity history shows `Failed to launch instance: Value (subnet-xxxxxx) for parameter subnetId is invalid`. Walk through the investigation: why an ASG might have a subnet ID that no longer exists, what happens to the ASG when its launch template references a deleted AMI vs deleted subnet, how to update the ASG to use a valid subnet without downtime, whether existing instances are affected, and how to prevent this with `aws ec2 describe-subnets` checks in your Terraform pipeline.

---

## Q93 | Kubernetes → Taints, Tolerations & Affinity | Scenario

You have a 50-node EKS cluster running mixed workloads. You need to: (1) dedicate 10 nodes (labeled `workload-type=gpu`) to GPU workloads only, (2) ensure a critical `payment-api` Deployment spreads across at least 3 AZs and never co-locates two replicas on the same node, (3) ensure a `log-collector` DaemonSet runs on ALL nodes including the GPU nodes, and (4) ensure a `batch-processor` Deployment prefers nodes in `us-east-1a` but can schedule elsewhere if unavailable. Write the complete YAML for the taint on GPU nodes, the tolerations, and the affinity/anti-affinity rules for each of the four workloads. Explain the difference between `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution`.

---

## Q94 | GitHub Actions → Reusable Workflows | Config

Write a reusable GitHub Actions workflow file `.github/workflows/docker-build-push.yml` that can be called from other workflows. It must accept inputs: `image_name` (string), `dockerfile_path` (string, default `./Dockerfile`), `context_path` (string, default `.`), and `push_to_ecr` (boolean). It must accept secrets: `aws_account_id` and `aws_region`. The workflow must: authenticate to ECR using OIDC (no stored credentials), build with BuildKit caching using GitHub Actions cache, tag images with the calling workflow's `github.sha` and `github.ref_name`, push only if `push_to_ecr` is true and the branch is `main` or matches `release/*`, and output the full image URI and image digest as workflow outputs.

---

## Q95 | Terraform → State Management | Coding

Write a Python script `terraform_state_audit.py` that uses `subprocess` to run `terraform state list` and `terraform state show` for each resource, parses the output into a structured dict mapping resource addresses to their attributes, identifies: (1) resources in state that no longer exist in any `.tf` file in the current directory (orphaned state), (2) resources that have drifted (compare state attributes to `terraform plan -json` output), and (3) resources with `lifecycle.prevent_destroy = true`. Output a color-coded terminal report using `rich` library and save a JSON report to `state_audit_<timestamp>.json`. Handle large state files (1000+ resources) with a progress bar. Requires only `subprocess`, `json`, `rich`, and standard library.

---

## Q96 | Linux & Bash → Containers & Cloud | 2AM Fire

At 2 AM, a Docker host running 30 containers starts exhibiting OOM kills. The kernel OOM killer log shows it is killing containers seemingly at random rather than the highest-memory container. Walk through the diagnosis: understanding how Docker translates `--memory` limits to cgroup `memory.limit_in_bytes`, why containers without `--memory` set have no cgroup limit (and become OOM kill candidates at the kernel level), how to identify which containers lack memory limits using `docker inspect` + `jq`, the interaction between host swap (`/proc/sys/vm/swappiness`) and container OOM behavior, and the immediate fix for containers without limits without downtime. Provide the one-liner to find all containers missing memory limits.

---

## Q97 | ArgoCD → Disaster Recovery | Scenario

Your ArgoCD instance is destroyed (the namespace is accidentally deleted along with all Application CRDs). You have Git as the source of truth and a recent Velero backup of the ArgoCD namespace. Describe two recovery paths: (1) full restore from Velero — the exact commands, how long it takes, and what state is lost between backup and deletion, and (2) fresh ArgoCD install + re-registration of all apps from Git — how you reconstruct Application CRDs if the Git repo contains `argocd/apps/` manifests, how you handle the fact that ArgoCD's repo credentials and cluster secrets are NOT in Git (they are in the ArgoCD namespace only), and how ApplicationSets vs App of Apps pattern affects the speed of each recovery path.

---

## Q98 | AWS → S3 & Storage | Failure Mode

A Lambda function that processes S3 event notifications stops receiving events. The S3 bucket has an event notification configured to trigger the Lambda on `s3:ObjectCreated:*`. New objects are being uploaded (verified via S3 inventory). Lambda invocation metrics show zero invocations. Describe the systematic diagnosis: verifying the Lambda resource-based policy allows `s3.amazonaws.com` to invoke the function, checking the S3 event notification configuration for correct prefix/suffix filters, verifying the Lambda function alias or version ARN in the notification matches the deployed function, checking for S3 Event Notification delivery failures in CloudWatch metrics (`NumberOfMessagesNotDelivered`), and how to test the end-to-end path with a manual `aws lambda invoke` + `aws s3 cp`.

---

## Q99 | Kubernetes → Service Mesh & Istio | 2AM Fire

At 2 AM you receive alerts that 100% of traffic to `checkout-service` is returning 503. Istio is deployed. Pod health checks pass. The Istio sidecar is injected. Walk through the Istio-specific investigation: using `istioctl proxy-status` to check xDS sync state, `istioctl proxy-config cluster` and `istioctl proxy-config route` to verify the Envoy config matches the intended VirtualService, checking `istioctl analyze` for misconfigurations, interpreting Envoy access logs (`%RESPONSE_FLAGS%` field — specifically `UF`, `UO`, `NR`), verifying the DestinationRule's `trafficPolicy.connectionPool` hasn't set `maxConnections: 0`, and checking if a recent `PeerAuthentication` change set `mtls: STRICT` before all sidecars were fully injected.

---

## Q100 | Platform Engineering → Architecture | Design Tradeoff

You are the founding Platform Engineer at a 300-engineer company that has outgrown its manual deployment process. Design the complete Internal Developer Platform (IDP) for the next 3 years: the golden path for a developer to go from `git push` to production in under 15 minutes with zero manual steps, the self-service infrastructure provisioning model (Backstage + Crossplane vs Backstage + Terraform + ServiceNow), the multi-tenancy model in Kubernetes (namespace-per-team vs vcluster vs separate clusters), the observability stack (what is standardized vs what teams choose), the secrets management strategy, and the on-call escalation path when a developer's self-service action breaks shared infrastructure. What are the two failure modes of platform teams, and how do you organizationally avoid them?

---
