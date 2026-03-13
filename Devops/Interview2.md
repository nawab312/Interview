## Q101 | Kubernetes → API Server | Conceptual

Explain how the Kubernetes API server handles watch requests from controllers. Describe the internal `watchCache` and how it differs from a direct etcd watch, the `ResourceVersion` mechanism that allows a controller to resume a broken watch without replaying all history, how the API server implements pagination for `LIST` requests using `continue` tokens, and what happens to a watch connection when the API server's `watchCache` is evicted due to memory pressure. Why do controllers that use `list-watch` never call `LIST` from etcd directly in production?

---

## Q102 | AWS → ALB & API Gateway | Scenario

Your ALB shows `HTTPCode_Target_5XX_Count` spiking to 500/minute but your application pods' own metrics show no errors. The ALB access logs show `502` responses with `target_status_code: -` (no response from target). Your backend is a Go HTTP service running in EKS. Walk through the exact diagnosis: the difference between ALB 502 (target returned invalid HTTP) vs 503 (no healthy targets) vs 504 (target timeout), how Go's default `http.Server` closes keep-alive connections and why this causes ALB 502 intermittently on deployment, the specific Go server configuration change (`ReadTimeout`, `WriteTimeout`, `IdleTimeout`, `shutdown graceful period`) that resolves rolling-deployment 502s permanently.

---

## Q103 | Terraform → Providers | Design Tradeoff

Your team uses a single Terraform root module that manages resources across 4 AWS accounts and 3 regions using multiple provider aliases. As the config grows, `terraform plan` takes 22 minutes because every provider refreshes all resources. Compare three strategies to reduce plan time: (1) `-target` scoped applies (risks and why you should never do this routinely), (2) splitting into separate state files with `terraform_remote_state` data sources including the circular dependency problem, (3) using `moved` blocks and `import` blocks (Terraform 1.5+) to refactor without destroying resources. Show the HCL syntax for declaring multiple aliased AWS providers across accounts using `assume_role` and how `provider` meta-argument is assigned on each resource.

---

## Q104 | Linux & Bash → Text Processing | Coding

Write a Bash script `parse_k8s_events.sh` that: (1) accepts a namespace as an argument (default: all namespaces), (2) runs `kubectl get events --sort-by='.lastTimestamp'` and parses the output using `awk` to extract: event type (`Warning` or `Normal`), reason, object kind/name, message, and count, (3) filters to show only `Warning` events from the last 30 minutes by parsing the `lastTimestamp` field and comparing to `date --date="-30 minutes"`, (4) groups by reason and shows frequency, (5) outputs a color-coded table using ANSI escape codes (red for `BackOff`, yellow for `FailedMount`, etc.), and (6) accepts `--watch` flag to continuously refresh every 10 seconds. Handle `kubectl` returning no events gracefully.

---

## Q105 | Prometheus & Grafana → Recording Rules | Conceptual

Explain why recording rules are essential for a Prometheus deployment with 3 million active series. Describe: the performance difference between evaluating a raw PromQL query at dashboard render time vs a precomputed recording rule, how recording rules are stored (as new time series in TSDB), the naming convention (`level:metric:operations`), how they interact with Prometheus federation when you want to expose aggregated metrics to a parent Prometheus, and the antipattern of creating recording rules that reference other recording rules more than 2 levels deep. Write a complete recording rule group for `http_request_duration_seconds` histogram that precomputes p50, p95, p99 per service with a 5m evaluation interval.

---

## Q106 | ArgoCD → SSO & RBAC | Config

Write the complete ArgoCD RBAC configuration using `argocd-rbac-cm` ConfigMap to implement: (1) a `dev-team` group (from GitHub OAuth/OIDC) that can sync and view Applications in projects `dev-*` but cannot delete Applications or modify project settings, (2) a `platform-team` group that has full access to all projects except cannot override sync policies on production projects, (3) a `readonly-auditor` group that can view all Applications, Repositories, and Clusters but cannot trigger any action, (4) a `ci-bot` local user (not SSO) with a minimal permission set to only trigger sync on specific Applications by name. Show both the `argocd-rbac-cm` policy and the `argocd-cm` OIDC connector config.

---

## Q107 | AWS → CloudWatch & Metrics | Coding

Write a Python script using Boto3 that: (1) accepts a list of EC2 instance IDs as CLI arguments, (2) queries CloudWatch for the following metrics over the last 24 hours with 5-minute resolution: `CPUUtilization`, `NetworkIn`, `NetworkOut`, `EBSReadOps`, `EBSWriteOps`, (3) uses `get_metric_data` with batching (not `get_metric_statistics`) to minimize API calls, (4) identifies instances where p95 CPU > 80% OR p95 NetworkIn > 80% of the theoretical maximum for the instance type (fetched from EC2 describe API), and (5) outputs a Markdown-formatted table sorted by p95 CPU descending. Handle the 500-metric-per-request limit of `get_metric_data` with chunking.

---

## Q108 | Kubernetes → Secrets Management | Design Tradeoff

Kubernetes Secrets are base64-encoded, not encrypted, and are stored in etcd in plaintext unless etcd encryption at rest is configured. Compare four approaches to secret delivery in Kubernetes: (1) native Secrets with etcd encryption using a KMS provider plugin (AWS KMS envelope encryption), (2) External Secrets Operator pulling from AWS Secrets Manager into native Secrets, (3) Vault Agent Injector mounting secrets as in-memory tmpfs files into Pods, and (4) CSI Secrets Store driver. For each, evaluate: what access is required to read the secret, whether the secret value appears in etcd, how rotation is handled, and the blast radius if the Kubernetes API server is compromised.

---

## Q109 | ELK Stack → Elasticsearch Performance | 2AM Fire

At 2 AM Kibana dashboards stop loading and Elasticsearch returns `503 Service Unavailable`. `curl -X GET 'http://elasticsearch:9200/_cluster/health'` returns `{"status":"red"}`. Walk through the exact diagnosis and recovery: interpreting `_cluster/health` output (unassigned shards, active primary shards), using `_cat/shards?v&h=index,shard,prirep,state,unassigned.reason` to find which shards are unassigned and why, using `_cluster/allocation/explain` to get the detailed reason, distinguishing between `NODE_LEFT` (node crashed) and `ALLOCATION_FAILED` (disk full), the immediate remediation for each case, and how you avoid split-brain if you need to force-allocate a primary shard using `reroute` with `allow_primary: true`.

---

## Q110 | GitHub Actions → Security Hardening | Design Tradeoff

A security review of your GitHub Actions workflows finds: workflows use `pull_request_target` trigger with `checkout` of the PR branch, several steps use `actions/cache@v3` which can be poisoned, third-party actions are pinned to tag names (mutable) rather than commit SHAs, and `GITHUB_TOKEN` has `permissions: write-all` by default. Redesign the complete security posture: explain the `pull_request_target` attack vector and correct trigger design for contributor PRs, how to pin all actions to immutable SHAs using `renovatebot`, how to scope `GITHUB_TOKEN` permissions using the `permissions` key at job level, and how to implement a required action allowlist using GitHub's organization-level policy.

---

## Q111 | AWS → EBS & Storage | Failure Mode

An EBS-backed EC2 instance starts showing intermittent `Input/output error` on filesystem operations. The CloudWatch `VolumeQueueLength` metric is consistently above 1. The volume is `gp2` type at 500GB. Describe the complete diagnosis: how `gp2` burst credits work and how to check if credits are exhausted via `BurstBalance` metric, the specific `VolumeQueueLength`, `VolumeReadOps`, `VolumeWriteOps` thresholds that indicate saturation, using `iostat -x 1` on the instance to correlate with CloudWatch, the migration path from `gp2` to `gp3` without downtime (using `ModifyVolume` API and the optimization state), and how `io1`/`io2` with provisioned IOPS would handle this differently. What is the live migration risk?

---

## Q112 | Python Scripting → Automation | Coding

Write a Python script `drift_detector.py` that detects infrastructure drift between Terraform state and live AWS resources. For every `aws_security_group` resource in the Terraform state file (read from `terraform show -json` output), the script must: (1) describe the actual security group from AWS using Boto3, (2) compare ingress/egress rules, tags, and description, (3) identify rules present in AWS but not in Terraform (manual additions), (4) identify rules in Terraform state but missing from AWS (external deletions), (5) output a colored diff per security group using `difflib.unified_diff`, and (6) exit with code `1` if any drift is found (for use in CI). Must handle pagination and use `assume_role` if `AWS_ASSUME_ROLE_ARN` env var is set.

---

## Q113 | Kubernetes → DaemonSets & Node Management | Scenario

You need to roll out a kernel parameter change (`net.core.somaxconn=65535`) to all 200 nodes in your EKS cluster without rebooting nodes during business hours. Design the complete approach using a privileged DaemonSet with a `hostPID: true` init container, the exact `sysctl` command, how to verify the change took effect on all nodes using a Job, how to handle nodes that join after the DaemonSet is deployed, and the security implications of running privileged DaemonSets (what can a compromised privileged DaemonSet Pod do to the host). Compare this approach to using EC2 launch template user data for a permanent change.

---

## Q114 | CI/CD → Pipeline Patterns | Config

Write a complete GitLab CI pipeline (`.gitlab-ci.yml`) for a Python microservice that implements: (1) a `test` stage running `pytest` with coverage report, failing if coverage < 80%, (2) a `security` stage running `bandit` (SAST) and `safety check` (dependency CVEs) in parallel, (3) a `build` stage that builds and pushes a Docker image to GitLab Container Registry tagged with `$CI_COMMIT_SHORT_SHA`, skipping push on MRs from forks, (4) a `deploy-staging` stage using `helm upgrade --install` triggered automatically on `main`, (5) a `deploy-prod` stage with a `when: manual` gate and `allow_failure: false`, only available on `main`. Use `needs:` for DAG-style dependencies and `rules:` not the deprecated `only/except`.

---

## Q115 | Terraform → Testing | Design Tradeoff

Your Terraform modules have no automated tests. A wrong variable value recently destroyed a production RDS instance. Design a complete testing pyramid for Terraform: (1) `tflint` and `checkov` for static analysis in CI (show the `.tflint.hcl` config), (2) `terraform validate` and `terraform plan` on every PR with plan output stored as an artifact, (3) unit tests with `terratest` in Go that deploy to a sandbox account and verify resource attributes (show a Go test function for the EKS module), (4) contract tests that verify outputs of one module match expected inputs of dependent modules, and (5) policy-as-code with OPA/Conftest (show a `.rego` policy that prevents `0.0.0.0/0` ingress on security groups). How do you run tests without incurring full AWS costs on every PR?

---

## Q116 | Prometheus & Grafana → Grafana Dashboards | Coding

Write a Grafana dashboard JSON (or Grafonnet Jsonnet) for a Kubernetes cluster overview panel that includes: (1) a stat panel showing cluster-wide CPU utilization as a percentage of allocatable CPU using the query `sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) / sum(kube_node_status_allocatable{resource="cpu"}) * 100`, (2) a time-series panel showing Pod restart rate by namespace over 24 hours, (3) a table panel listing the top 10 Pods by memory usage with namespace, pod name, and memory in Mi, (4) a heatmap panel of HTTP request latency distribution using `http_request_duration_seconds_bucket`. All panels must use template variables for `cluster` and `namespace` filtering. Output the complete JSON.

---

## Q117 | AWS → STS & Cross-Account | Conceptual

Explain the complete mechanics of AWS cross-account role assumption: what `sts:AssumeRole` exchanges (the trust policy validation, MFA conditions, `ExternalId` usage), how session credentials are scoped (the intersection of the role's policies and any passed session policy), why the maximum session duration for a role assumed by another role is capped at 1 hour regardless of the role's `MaxSessionDuration`, how `aws:PrincipalOrgID` conditions work in resource policies without listing every account, and the security implication of having `sts:AssumeRole` with `"Principal": "*"` without an `ExternalId` condition on roles that manage infrastructure (confused deputy attack). Show the trust policy fix.

---

## Q118 | ELK Stack → Mapping & Schema | Debug

The following Elasticsearch index mapping is causing: (1) numeric values being stored as strings (breaking range queries), (2) a `mapping explosion` from a dynamic JSON field with hundreds of unique keys, and (3) full-text search returning no results on the `message` field despite the data being there. Identify every problem, explain the root cause, and provide the corrected mapping:

```json
{
  "mappings": {
    "dynamic": true,
    "properties": {
      "timestamp": { "type": "text" },
      "level": { "type": "text" },
      "response_time_ms": { "type": "text" },
      "status_code": { "type": "text" },
      "message": { "type": "keyword" },
      "user_id": { "type": "long" },
      "metadata": { "type": "object", "dynamic": true },
      "tags": { "type": "text" },
      "host": {
        "properties": {
          "ip": { "type": "text" },
          "name": { "type": "text" }
        }
      }
    }
  }
}
```

---

## Q119 | Kubernetes → StatefulSets | Failure Mode

A StatefulSet `kafka` with `replicas: 3` is being upgraded via a RollingUpdate. The update stalls after updating Pod `kafka-2` and `kafka-1`: Pod `kafka-0` is stuck in `Pending` due to the new image requiring 4Gi memory but the cluster has only 3.5Gi free. Describe: why StatefulSet rolling updates go in reverse order (2, 1, 0) unlike Deployments, how `updateStrategy.rollingUpdate.partition` can be used as a canary mechanism, what `kubectl rollout undo statefulset kafka` does and whether it immediately recovers kafka-0, the `OnDelete` update strategy as an alternative and when it is appropriate, and how `PodManagementPolicy: Parallel` changes the failure mode in this scenario.

---

## Q120 | GitHub Actions → Artifact Management | Scenario

Your GitHub Actions builds produce Docker images, test coverage reports, compiled binaries, and Terraform plan files as artifacts. After 3 months, GitHub artifact storage costs are $800/month and artifact downloads are slow because artifacts are 2GB+ per build. Design a complete artifact management strategy: using `actions/upload-artifact` compression settings, moving large artifacts to S3 with pre-signed URLs using a custom action, implementing a retention policy per artifact type (plans: 7 days, coverage: 30 days, release binaries: 1 year), and creating a cleanup workflow that runs nightly using the GitHub API to delete expired artifacts. Include the GitHub Actions API call to list and delete artifacts programmatically.

---

## Q121 | Linux & Bash → Kernel & System | Conceptual

Explain the Linux kernel's OOM killer in detail: how it scores processes using `oom_score_adj` and memory consumption, why it sometimes kills the wrong process, the `vm.overcommit_memory` settings (0, 1, 2) and their effect on OOM behavior, how `cgroups v2` memory limits interact with the OOM killer differently from `cgroups v1`, and the difference between a process being OOM-killed by the cgroup OOM killer (container) vs the global OOM killer. A JVM process with `-Xmx4g` is being killed by the OOM killer even though it seems to be within its heap limit. Explain the two most common reasons (native memory + JVM overhead, container limit vs JVM perception of total RAM) and the exact JVM flags to fix it.

---

## Q122 | ArgoCD → Git Repositories & Helm | Scenario

You are adopting ArgoCD to manage 80 Helm chart deployments across dev, staging, and prod. Each environment has different `values.yaml` overrides. You need to design the Git repository structure for values and evaluate: (1) one monorepo with `environments/<env>/<app>/values.yaml`, (2) one repo per environment, (3) using Helm's `--set` overrides in the Application spec (not Git-stored). For each, describe how a developer promotes a values change from dev to prod, how ArgoCD diff shows what will change, how you prevent prod values from being overridden by a dev push, and how Helm chart version upgrades are managed across all environments simultaneously. Which structure works best with ApplicationSets?

---

## Q123 | AWS → Networking → PrivateLink | Conceptual

Explain AWS PrivateLink: how an interface VPC endpoint uses ENIs in your subnets to route traffic to the service provider's NLB without traversing the internet or VPC peering, why traffic does NOT appear in VPC Flow Logs at the provider side, the DNS resolution chain (how `vpce-*.amazonaws.com` resolves to private IPs in your VPC), the difference between a gateway endpoint (S3, DynamoDB) and interface endpoint (all other services), how endpoint policies work as a resource-based policy, and the scenario where `aws s3 cp` from an EC2 instance fails despite a gateway endpoint existing because the bucket policy requires `aws:SourceVpc` and the request comes from a VPC-peered account.

---

## Q124 | Python Scripting → CLI Tools | Coding

Write a Python CLI tool `kubecost` using `argparse` that provides cost visibility for Kubernetes workloads. It must: (1) connect to the Kubernetes API to list all Deployments and their CPU/memory requests across all namespaces, (2) fetch EC2 on-demand pricing from AWS Pricing API for the node instance types in the cluster (use the `boto3` `pricing` client), (3) calculate the approximate monthly cost per Deployment based on its resource requests as a fraction of node cost, (4) support `--namespace`, `--sort-by cost|cpu|memory`, and `--top N` flags, (5) output a rich terminal table with columns: `namespace`, `deployment`, `replicas`, `cpu_request`, `memory_request`, `estimated_monthly_usd`. Cache pricing data for 24 hours in `~/.kubecost_cache.json`. No external cost-tooling libraries.

---

## Q125 | Terraform → Import & Refactoring | Scenario

Your team manages an S3 bucket and its policy that were created manually 2 years ago and have never been in Terraform state. The bucket has 500TB of data and cannot be recreated. Additionally, a large refactor is needed: an `aws_instance` resource must be moved from `module.legacy` to `module.compute` without destroying and recreating it. Walk through: (1) using `terraform import` for the S3 bucket and policy (show exact commands and what you must write in HCL first), (2) using the `moved` block (Terraform 1.1+) for the instance refactor (show exact HCL), (3) verifying with `terraform plan` that both operations show no changes after import/move, and (4) the risk of running `terraform apply` immediately after import if your HCL does not perfectly match the existing resource.

---

## Q126 | Kubernetes → Init Containers & Sidecars | Design Tradeoff

Compare init containers, sidecar containers (long-running), and ephemeral containers for three use cases: (1) waiting for a database to be ready before starting the main application, (2) shipping application logs to a central aggregator, and (3) live debugging a running production Pod without restarting it. For each use case, explain the lifecycle differences (init vs sidecar termination behavior, resource accounting, restart policy), the Kubernetes 1.29+ native sidecar container feature (`restartPolicy: Always` in `initContainers`) and how it differs from classic sidecars, and why an ephemeral container cannot be used for (1) or (2). Show the Pod spec snippet for the Kubernetes 1.29+ native sidecar log shipper.

---

## Q127 | CI/CD → GitHub Actions | Coding

Write a composite GitHub Action (stored in `.github/actions/notify-deploy/action.yml`) that posts a deployment status notification to both Slack and a GitHub Deployment environment. The action must accept inputs: `environment` (string), `status` (`success`/`failure`/`in_progress`), `service_name` (string), `image_tag` (string), `run_url` (string). It must: (1) create or update a GitHub Deployment using the GitHub REST API via `actions/github-script`, (2) set the Deployment Status to the appropriate GitHub state (`success`/`failure`/`in_progress`), (3) post a Slack message with Block Kit formatting showing service, environment, status (with emoji), image tag, and a link to the workflow run. Secrets `SLACK_WEBHOOK_URL` must be passed as an input (not accessed as env directly). Show the complete `action.yml`.

---

## Q128 | AWS → MSK Kafka | Config

Write the complete Terraform configuration to provision an Amazon MSK cluster with: (1) 3 broker nodes using `kafka.m5.large` across 3 AZs, (2) EBS storage of 1000GB per broker with auto-scaling enabled up to 2000GB, (3) Kafka version 3.5.1, (4) encryption in transit (`TLS_PLAINTEXT` for in-cluster, `TLS` for client-to-broker), (5) encryption at rest using a CMK (include the KMS key resource), (6) enhanced monitoring at `PER_TOPIC_PER_BROKER` level with CloudWatch, (7) a security group allowing port 9094 (TLS) only from a given CIDR variable, and (8) a configuration for `auto.create.topics.enable=false` and `default.replication.factor=3`. Include outputs for bootstrap brokers TLS endpoint.

---

## Q129 | Prometheus & Grafana → Alertmanager | Config

Write a complete Alertmanager configuration (`alertmanager.yml`) that implements: (1) a route tree where `severity: critical` alerts page PagerDuty immediately with 0s group_wait, (2) `severity: warning` alerts go to Slack with 5m group_wait and 1h repeat_interval, (3) alerts labeled `team: platform` override the above and always route to a dedicated `#platform-alerts` Slack channel regardless of severity, (4) a global inhibition rule that suppresses all warnings for a given `cluster` label when a `KubeClusterDown` critical alert is firing for the same cluster, (5) a time-based muting route that silences all non-critical alerts on weekends using `time_intervals`. Include the PagerDuty and Slack receiver configs with proper templating.

---

## Q130 | Linux & Bash → Networking | Coding

Write a Bash script `net_debug.sh` that performs a comprehensive network diagnostic on a Linux host and outputs a structured report. It must check and report: (1) all non-loopback interfaces with IP, MTU, RX/TX errors from `ip -s link`, (2) routing table and default gateway reachability via ICMP and TCP (port 80) using `ping` and `nc`, (3) DNS resolution time for 5 domains using `dig +stats` parsing the query time, (4) open listening ports and the process bound to each using `ss -tlnp`, (5) established connection count per destination IP (top 10), (6) packet loss percentage to the default gateway over 20 pings, and (7) iptables rule count per chain. Output everything as JSON to stdout and a human-readable summary to stderr.

---

## Q131 | Kubernetes → Cluster Upgrades | Scenario

You must upgrade an EKS cluster from Kubernetes 1.27 to 1.29 (requires two minor version upgrades) with zero downtime for production workloads. The cluster has 150 worker nodes, StatefulSets running Kafka and PostgreSQL, and PodDisruptionBudgets on all critical services. Describe the complete procedure: upgrading the control plane first (EKS managed) and what breaks during control plane upgrade, the node group upgrade strategy (managed node groups rolling update vs blue/green node group), how to verify API deprecations before upgrading using `pluto` or `kubectl deprecations`, the specific APIs removed in 1.28 and 1.29 that require application manifest updates, and how you validate the cluster is healthy at each step before proceeding to the next minor version.

---

## Q132 | ELK Stack → Logstash Pipelines | Design Tradeoff

Your current architecture sends all logs through a single Logstash pipeline processing 500,000 events/second. The pipeline has become a single point of failure and bottleneck. Compare three alternative architectures: (1) multiple independent Logstash pipelines using the `pipeline-to-pipeline` feature with a shared input queue, (2) replacing Logstash with Kafka Streams for filtering/transformation and using Elasticsearch Ingest Pipelines for enrichment, (3) replacing the entire stack with Fluent Bit (lightweight) + Fluentd (aggregation) + direct Elasticsearch bulk API. For each, evaluate: backpressure handling, per-pipeline failure isolation, throughput ceiling per node, and the Logstash-specific `Dead Letter Queue` behavior when Elasticsearch is unavailable.

---

## Q133 | AWS → CloudTrail & Audit | Coding

Write a Python script `cloudtrail_audit.py` that: (1) uses Boto3 CloudTrail `lookup_events` to find all IAM privilege escalation events in the last 7 days — specifically `AttachUserPolicy`, `AttachRolePolicy`, `PutUserPolicy`, `CreateAccessKey`, `AddUserToGroup`, and `AssumeRole` where the assumed role has `AdministratorAccess`, (2) deduplicates events by `userIdentity.arn` + `eventName` per hour, (3) for each suspicious event, fetches the full event detail including source IP, user agent, and request parameters, (4) generates a structured JSON report AND a Markdown summary grouped by IAM principal, and (5) posts a Slack alert if any events are found. Handle CloudTrail's 1-event-per-second lookup rate limit with proper throttling.

---

## Q134 | GitHub Actions → Reusable Workflows | Design Tradeoff

Your platform team maintains 8 reusable workflows called by 150 application repositories. A breaking change to the `deploy.yml` reusable workflow (changing an input name from `image_tag` to `container_tag`) would require updating all 150 caller repositories simultaneously. Design a versioning and backwards compatibility strategy: using Git tags on the platform repo and `@v2` references in callers vs `@main` references (tradeoffs of each), how to implement an input alias shim that accepts both `image_tag` and `container_tag` during a transition period using conditional expressions, how to detect which repositories still use the old input using the GitHub API to search workflow files, and the deprecation communication workflow using GitHub's `workflow_dispatch` to auto-create PRs in all consumer repositories.

---

## Q135 | Kubernetes → Resource Management | Debug

A namespace `analytics` has a ResourceQuota set but Pod scheduling is failing with `exceeded quota: requests.cpu` despite `kubectl describe resourcequota` showing available capacity. The Pods are not starting even though the math clearly shows enough CPU. Diagnose all possible explanations: the difference between ResourceQuota counting `requests` vs `limits`, what happens when a Pod spec has no `requests` set and a LimitRange with a `default` is present (LimitRange applies defaults AFTER quota check), how `status.used` in ResourceQuota can be stale, the `kubectl describe limitrange` output that would explain why the effective request is higher than specified, and the specific `kubectl` command that shows the exact reason a Pod failed admission.

---

## Q136 | Terraform → Workspaces & Environments | Config

Write a complete Terraform module structure and root configuration for managing 3 environments (dev, staging, prod) where: (1) each environment uses the same module but with different variable values stored in `terraform.tfvars` files per environment, (2) the prod environment has additional resources (WAF, enhanced logging) conditionally created using `var.environment == "prod"`, (3) the module outputs are used by a secondary Terraform configuration for the application layer via `terraform_remote_state`, (4) each environment's state is stored in a separate S3 key (`env://dev/terraform.tfstate`), and (5) a `locals` block derives environment-specific tags including `cost_center` from a map variable. Show the complete directory structure, backend configuration, and the `count`/`for_each` patterns for conditional prod resources.

---

## Q137 | Prometheus & Grafana → Tempo & Tracing | Conceptual

Explain how Grafana Tempo differs architecturally from Jaeger or Zipkin for distributed tracing at scale. Describe: Tempo's object-storage-first design (why it stores traces directly to S3/GCS with no search index by default), how `TraceQL` works and what the `metrics-generator` component adds, the difference between Tempo's `search` (tag-based, requires Tempo search enabled) vs trace lookup by ID, how exemplars in Prometheus link a high-latency metric spike to a specific Tempo trace ID, and how you would instrument a Python Flask service to emit traces to Tempo using OpenTelemetry SDK with automatic instrumentation. What is the cost model difference between Tempo (object storage) and Jaeger with Cassandra at 1TB/day of trace data?

---

## Q138 | AWS → ECR & Container Registry | Failure Mode

Your EKS cluster nodes start failing to pull images from ECR with `ImagePullBackOff`. The error in `kubectl describe pod` is `failed to pull image: 401 Unauthorized`. This worked fine yesterday. Describe all possible root causes and their diagnosis: the IAM role attached to the node group missing `ecr:GetAuthorizationToken` permission, the ECR authorization token expiry (12 hours) and how the kubelet caches the token vs requests a new one per pull, a lifecycle policy on the ECR repository that deleted the image version, a recently applied SCPs at the AWS Organization level blocking ECR access, and the cross-account ECR scenario where the image is in a different account and the ECR repository policy must explicitly allow the pulling account. Give the exact JSON for the ECR repository policy fix.

---

## Q139 | Linux & Bash → Performance Tuning | Scenario

A high-throughput network service on Linux is dropping packets at 100k connections/second despite the server having capacity. `ss -s` shows `TCP: ... timewait 80000`. `netstat -s` shows `SYNs to LISTEN sockets dropped`. Identify and fix: `net.ipv4.tcp_tw_reuse` and `net.ipv4.tcp_fin_timeout` for TIME_WAIT accumulation, `net.core.somaxconn` and `net.ipv4.tcp_max_syn_backlog` for the listen backlog, `net.ipv4.ip_local_port_range` for ephemeral port exhaustion, `ulimit -n` for file descriptor limits per process vs system-wide `fs.file-max`, and `net.core.netdev_max_backlog` for NIC receive queue. Provide the exact `sysctl` commands and the `/etc/sysctl.d/99-network-tuning.conf` file with correct values for a 100k-connection server.

---

## Q140 | ArgoCD → Helm & Kustomize | Design Tradeoff

Compare using Helm charts vs Kustomize overlays as the application delivery mechanism in ArgoCD for a platform serving 50 teams. Address: how ArgoCD renders Helm charts (does it run `helm template` and commit the output, or render at sync time), the `ignoreDifferences` field needed for Helm-generated resources that change on every sync (e.g., `helm.sh/chart` annotation), whether Kustomize `secretGenerator` works safely with ArgoCD (it generates new Secret names with hash suffixes — implications for Deployments referencing them), how `argocd app diff` works for each approach, and the scenario where a Helm chart upgrade changes a CRD — how ArgoCD handles CRD upgrades and why this requires special handling in both approaches.

---

## Q141 | AWS → Fargate & Serverless Containers | Conceptual

Explain the Fargate data plane architecture: how Fargate tasks run in isolated microVMs (using Firecracker), why you cannot exec into a Fargate container using `kubectl exec` on older EKS Fargate setups (and how EKS Fargate now supports exec via SSM Session Manager), how Fargate networking works (each task gets an ENI in your VPC — the VPC ENI trunking mechanism), the implication of no node-level DaemonSets on Fargate (how you do log collection and monitoring without DaemonSets), and why Fargate tasks have a minimum 20-second startup latency vs EC2 worker nodes (and what you do for latency-sensitive scaling).

---

## Q142 | ELK Stack → Scaling | Design Tradeoff

Your Elasticsearch cluster handles 1TB/day of ingest. The cluster is running 5 data nodes with 64GB RAM and 4TB SSD each. Ingest latency has grown to 8 seconds. Diagnose the bottleneck and design the scaling strategy: how to determine if the bottleneck is ingest (bulk indexing), query (search load), or merge (segment merge pressure) using `_nodes/stats` and `_cat/thread_pool`, the role of dedicated master nodes vs data nodes vs coordinating-only nodes, using `_cat/indices?v` to identify hot indices by `indexing.index_total`, the write performance impact of `refresh_interval` (default 1s) and why setting it to `30s` during bulk ingest improves throughput by 3×, and the ILM hot-warm-cold tier design that reduces SSD cost by 60%.

---

## Q143 | Kubernetes → Custom Metrics & KEDA | Scenario

Your message processing service needs to scale based on the depth of an SQS queue — scaling from 0 to 50 replicas as queue depth grows. Native HPA only supports CPU/memory. Design the complete solution using KEDA: the `ScaledObject` CRD manifest targeting your `Deployment`, the SQS `TriggerAuthentication` using IRSA (IAM Role for Service Accounts) so KEDA's operator can read SQS queue attributes without static credentials, the `minReplicaCount: 0` configuration and the implication of scale-to-zero (cold start latency for your consumer), the `pollingInterval` and `cooldownPeriod` values to avoid thrashing, and how to observe the KEDA scaling decisions via its metrics endpoint and Grafana dashboard.

---

## Q144 | Python Scripting → Subprocess | Coding

Write a Python module `kubectl_runner.py` with a class `KubectlRunner` that wraps `kubectl` for safe programmatic use. It must implement: (1) `run(args: list[str], timeout: int = 30, namespace: str | None = None) -> KubectlResult` that executes kubectl, captures stdout/stderr, and returns a typed dataclass, (2) `apply(manifest: dict, dry_run: bool = False) -> KubectlResult` that serializes the manifest to YAML (using `pyyaml`) and pipes it to `kubectl apply -f -`, (3) `wait(resource_type: str, name: str, condition: str, timeout: int = 120) -> bool`, (4) context manager support that automatically sets and restores `KUBECONFIG` env var, and (5) structured logging of every command with duration and exit code. Never use `shell=True`. All subprocess calls must use `subprocess.run` with explicit argument lists. Include type hints throughout.

---

## Q145 | AWS → WAF & Shield | Design Tradeoff

You need to protect a public-facing API (API Gateway + Lambda) from DDoS attacks, SQL injection, and credential stuffing (high-volume login attempts from rotating IPs). Compare AWS WAF v2 managed rules vs custom rules vs third-party WAF (Cloudflare, Imperva) at the CloudFront layer. Specifically: how AWS WAF rate-based rules work (token bucket per IP vs IP + URI), how to use AWS WAF labels to chain rules (flag a request with a label in one rule, block in another), the limitation of WAF for volumetric DDoS vs AWS Shield Advanced, and how AWS Bot Control's `TargetedBotControl` differs from the basic bot control managed rule group. For credential stuffing, what WAF rule logic (rate-limit on `/login` endpoint by IP + fingerprint) would you implement?

---

## Q146 | CI/CD → Jenkins | Coding

Write a Jenkins Shared Library class `src/com/company/AwsEcrPublisher.groovy` that provides a method `publish(Map config)` accepting: `imageName`, `dockerfilePath`, `buildArgs` (map), `tags` (list), `ecrRegion`, and `ecrAccountId`. The method must: (1) log in to ECR using `aws ecr get-login-password` piped to `docker login`, (2) build the image with BuildKit enabled (set `DOCKER_BUILDKIT=1`) and all provided build args, (3) tag and push all specified tags, (4) run `trivy image --exit-code 1 --severity CRITICAL` and fail the build if critical CVEs are found, (5) output the image digest after push by parsing `docker inspect --format='{{index .RepoDigests 0}}'`, and (6) handle ECR repository creation if it doesn't exist. Use `withCredentials` for no hardcoded credentials.

---

## Q147 | Kubernetes → Pod Lifecycle | Failure Mode

A Kubernetes Deployment rolling update causes a 30-second window of 503 errors from the load balancer even though `readinessProbe` is configured. The Pods show `Running` and `Ready` in `kubectl get pods` before traffic hits them. Explain the race condition: the gap between a Pod becoming `Ready` in Kubernetes (readinessProbe passes) and the endpoint being propagated through the chain (Endpoints object → kube-proxy → iptables/IPVS rules → ALB target group registration via the AWS Load Balancer Controller), the typical propagation latency for each hop, how `preStop` hook with a `sleep` resolves the termination side of this (old Pod receiving traffic after shutdown), and the complete Pod lifecycle configuration (`minReadySeconds`, `preStop`, `terminationGracePeriodSeconds`, `readinessProbe initialDelaySeconds`) that eliminates 503s.

---

## Q148 | Linux & Bash → System Security | Scenario

Your security team reports that a Linux server may have been compromised. You have read-only access and cannot take the server offline yet. Describe the live forensics procedure using only standard Linux tools: (1) checking for persistence mechanisms (`crontab -l` for all users, `/etc/cron.*`, systemd units in unusual paths, `~/.bashrc`/`~/.profile` modifications), (2) finding recently modified files in `/etc`, `/usr/bin`, `/usr/local/bin` using `find -newer /tmp/reference_time`, (3) checking for hidden processes by comparing `ps aux` PID list with `/proc` directory listing, (4) identifying unauthorized listening ports with `ss -tlnp` and `lsof -i`, (5) checking for unusual SUID/SGID binaries with `find / -perm /6000 -type f`, and (6) capturing network connections to C2 servers via `/proc/net/tcp` parsing. What should you NOT do during live forensics?

---

## Q149 | ArgoCD → Health Checks & Hooks | Config

Write ArgoCD Application manifest YAML that implements a blue/green deployment pattern using ArgoCD resource hooks. The Application must: (1) use a PreSync hook that runs a Job to take a database snapshot before any manifests are applied, (2) use a Sync hook with `Sync` wave `-1` to deploy the new version alongside the old (blue/green), (3) use a PostSync hook that runs an integration test Job and reports pass/fail, (4) use a SyncFail hook that runs a rollback Job if the PostSync test fails. Show the hook Job manifests with the correct `argocd.argoproj.io/hook` and `argocd.argoproj.io/hook-delete-policy` annotations. Explain the difference between `HookSucceeded` and `BeforeHookCreation` delete policies.

---

## Q150 | AWS → Lambda → Performance | Design Tradeoff

A Lambda function processing DynamoDB Streams has a p99 latency of 8 seconds on a 256MB configuration. The function deserializes DynamoDB events, enriches records by calling an internal HTTP API, and writes results to S3. Profile the function and design the optimizations: Lambda cold start reduction strategies (provisioned concurrency vs `INIT_TYPE=snap-start` for Java, Lambda SnapStart for Lambda Web Adapter), the impact of memory size on CPU allocation (Lambda's linear CPU scaling model), connection reuse by initializing `boto3` clients and `requests.Session` outside the handler, batching S3 writes using `io.BytesIO` and multipart upload, and parallelizing the HTTP enrichment calls using `asyncio` with `aiohttp` for the batch of DynamoDB records. Quantify the expected improvement for each optimization.

---

## Q151 | Prometheus & Grafana → Mimir | Conceptual

Explain how Grafana Mimir achieves horizontally scalable Prometheus-compatible metrics. Describe the architecture components: Distributor (consistent hashing ring, replication factor), Ingester (in-memory series with WAL, block upload to object storage), Querier (query federation from ingesters + store-gateway), Store-Gateway (indexes and chunks from object storage with LRU cache), Compactor (background compaction and retention enforcement). How does Mimir handle a 3-replica ingester setup where one ingester crashes mid-write? Explain the `quorum` write path and how a querier achieves consistent reads by querying multiple ingesters and deduplicating using the `replication factor / 2 + 1` quorum read.

---

## Q152 | ELK Stack → Kibana | Scenario

Your Kibana deployment serves 200 analysts who run large date-range aggregation queries. Kibana is running out of memory (heap exhaustion), and slow queries are cascading to Elasticsearch circuit breaker trips (`Data too large`). Design the complete remediation: Kibana query timeout settings (`xpack.reporting.queue.timeout`, `elasticsearch.requestTimeout`), Elasticsearch query circuit breaker configuration (`indices.breaker.total.limit`, `indices.breaker.request.limit`), how to use Kibana's Query Profiler to identify expensive aggregations, implementing user-level query limits via Kibana Spaces + Elasticsearch DLS to restrict date range to 30 days maximum (show the DLS query), and the Elasticsearch `search.max_buckets` setting that prevents aggregation explosions.

---

## Q153 | Kubernetes → Networking CNI | Conceptual

Explain the differences between three Kubernetes CNI plugins at the packet level: (1) AWS VPC CNI (each Pod gets a real VPC IP via ENI secondary IPs — how `aws-k8s-agent` pre-warms IPs, the `WARM_IP_TARGET` setting, and why node capacity is limited by ENI × secondary IPs per ENI), (2) Cilium with eBPF (how it replaces kube-proxy entirely, how eBPF programs attached to TC hooks replace iptables, and the performance difference at 100k connections/second), and (3) Flannel with VXLAN (overlay encapsulation overhead, MTU implications with AWS's 1500 MTU and the required 50-byte VXLAN overhead). For a cluster with 1000 Pods on 50 nodes, which CNI would you choose and why?

---

## Q154 | GitHub Actions → Large Scale CI | 2AM Fire

At 2 AM GitHub Actions reports that all workflows in your organization are queued but not running. Your self-hosted runner fleet (100 runners on EC2) shows all runners as `Idle` in the GitHub UI but jobs are not dispatching. Walk through the diagnosis: checking GitHub's status page first, then GitHub Actions runner application logs (`/var/log/actions-runner/*.log`), verifying runner registration tokens haven't expired (runners registered with expired tokens appear `Idle` but fail job pickup), checking if a recent GitHub App permission change revoked the `actions: write` scope, the runner group assignment (jobs querying for `runs-on: self-hosted` but runners are in a non-default group), and the exact GitHub API calls to list runner status and force re-register a runner.

---

## Q155 | Terraform → Sentinel & Policy | Config

Write an OPA/Conftest policy (`terraform_policies/security.rego`) that enforces the following rules on Terraform plan JSON output: (1) all S3 buckets must have `server_side_encryption_configuration` defined, (2) all security groups must not have ingress rules with `cidr_blocks` containing `0.0.0.0/0` on any port other than 80 and 443, (3) all EC2 instances must have a `Name` tag, (4) RDS instances must have `deletion_protection = true` when the Terraform workspace is `prod`, and (5) IAM roles must not have inline policies with `"Effect": "Allow"` and `"Action": "*"`. Show the complete `.rego` file and the `conftest test` invocation command. Include at least one `deny` rule and one `warn` rule.

---

## Q156 | AWS → ECS → Task Definitions | Config

Write a complete ECS task definition JSON for a production web service with: (1) two containers — `app` (your service) and `datadog-agent` (sidecar), (2) the `app` container with CPU 512, memory 1024, port mapping 8080, environment variables from SSM Parameter Store using `secrets` field, `awslogs` log driver to CloudWatch, health check using `CMD-SHELL curl -f http://localhost:8080/health`, and `dependsOn` the `datadog-agent` being `HEALTHY`, (3) the `datadog-agent` container with `DD_API_KEY` from Secrets Manager, UDP ports 8125 and 8126, (4) task-level CPU 1024 and memory 2048, `awsvpc` network mode, an execution role ARN and task role ARN as variables, and (5) `ephemeral_storage` of 40GB. Output valid JSON.

---

## Q157 | Python Scripting → Scheduling & Monitoring | Coding

Write a Python service `health_monitor.py` that runs as a long-lived daemon and: (1) reads a YAML config file (`services.yml`) listing service name, URL, expected HTTP status, and check interval in seconds, (2) schedules health checks using `asyncio` with each service checked on its own interval concurrently, (3) maintains a `ServiceState` dataclass tracking: current status, consecutive failures, last success time, total checks, and uptime percentage, (4) exposes the health state of all services via a simple HTTP server on port 9090 at `/health` (JSON) and `/metrics` (Prometheus text format), (5) sends a PagerDuty Events API v2 alert when a service has 3 consecutive failures, and (6) sends a recovery notification when it recovers. Reload config without restart on `SIGHUP`. Use only stdlib + `aiohttp` + `pyyaml`.

---

## Q158 | Kubernetes → Multi-tenancy | Design Tradeoff

Your platform must support 50 product teams sharing a single EKS cluster. Each team needs isolation of compute, network, and blast radius. Compare three multi-tenancy models: (1) namespace-per-team with ResourceQuotas, LimitRanges, NetworkPolicies, and RBAC — describe the blast radius if a team creates a ClusterRole by exploiting a misconfigured RoleBinding, (2) vCluster (virtual clusters) — how they work (a full Kubernetes API server running as a workload in a host namespace), the resource overhead per vCluster, and how networking between vClusters is handled, (3) separate EKS clusters per team — the operational cost at 50 clusters (addons, upgrades, monitoring), the API server cost per cluster. Which model handles a team needing to install a cluster-scoped CRD?

---

## Q159 | ELK Stack → Ingest Pipelines | Config

Write an Elasticsearch Ingest Pipeline JSON that processes incoming application log documents with the following transformations: (1) parse the `message` field using a Grok processor to extract `timestamp`, `level`, `service`, `trace_id`, and `msg` from a log line format like `2024-01-15T10:23:45Z INFO payment-service [abc123] Payment processed`, (2) convert `@timestamp` from the extracted `timestamp` using a Date processor with the pattern `ISO8601`, (3) use a GeoIP processor on `client_ip` field (if present), (4) add a `environment` field from a Pipeline parameter using a Set processor, (5) use a Script processor to compute `is_error` as a boolean based on `level` being `ERROR` or `FATAL`, (6) remove the original `message` field, and (7) fail the pipeline with a specific error tag if Grok doesn't match. Show the complete pipeline PUT API call.

---

## Q160 | AWS → Cost Anomaly Detection | Scenario

Your AWS bill spikes $50,000 in a single day. CloudWatch billing alarms didn't fire because the threshold was set too high. Using AWS Cost Anomaly Detection, Cost Explorer, and CloudTrail, walk through the investigation: how to use `GetCostAndUsage` API to break down the spike by service and linked account, the specific Cost Explorer filter and group-by dimensions to find the culprit service within 5 minutes, how to correlate the cost spike with CloudTrail API calls using `LookupEvents` filtered by the high-cost service and time window, the three most common causes of sudden $50k spikes (NAT Gateway data transfer from a misconfigured route, S3 Glacier restore fees, EC2 Spot Instance interruption causing ASG to launch On-Demand), and how to set up anomaly detection subscriptions with SNS to catch this in future.

---

## Q161 | Kubernetes → Jobs & CronJobs | Failure Mode

A CronJob runs a batch data processing job every hour. After a Kubernetes upgrade, you notice that failed Jobs are accumulating — `kubectl get jobs` shows 200 completed and 80 failed jobs from the past week. The cluster's etcd is approaching its object count limit. Describe: the `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` defaults (3 and 1) and why they were reset during upgrade, how to clean up existing jobs without a script using `kubectl delete jobs --field-selector status.successful=1`, the `activeDeadlineSeconds` and `backoffLimit` interaction that determines when a Job moves to `Failed` state, and how a Job with `restartPolicy: OnFailure` vs `restartPolicy: Never` accumulates failed Pods differently. Write the correct CronJob spec with history limits and proper failure policy.

---

## Q162 | GitHub Actions → OIDC Deep Dive | Conceptual

Explain the complete cryptographic and API flow of GitHub Actions OIDC federation with AWS. Describe: how GitHub generates a signed JWT for each workflow run (what claims are in the token — `sub`, `repository`, `ref`, `job_workflow_ref`, `environment`), where the JWT signing key material lives and how AWS validates the signature via the JWKS endpoint at `https://token.actions.githubusercontent.com/.well-known/jwks`, the IAM trust policy `Condition` that uses `StringLike` vs `StringEquals` on the `sub` claim (and the security implication of using `StringLike` with wildcards), why the OIDC token is short-lived (5 minutes) and how the `assume-role-with-web-identity` call exchanges it for temporary credentials, and the attack surface if an attacker forks your repo and triggers a workflow.

---

## Q163 | Linux & Bash → Advanced Scripting | Coding

Write a Bash script `service_failover.sh` that monitors a primary service and performs automatic failover to a standby. The script must: (1) accept `--primary-ip`, `--standby-ip`, `--port`, `--check-interval`, and `--failure-threshold` as named arguments, (2) use `nc -zw2` to check TCP connectivity with a 2-second timeout, (3) implement a state machine with states `PRIMARY_UP`, `PRIMARY_DEGRADED` (threshold not yet reached), and `FAILED_OVER`, (4) only failover after `--failure-threshold` consecutive failures (not a single blip), (5) on failover, update an AWS Route 53 record using `aws route53 change-resource-record-sets` with the standby IP (record ID and hosted zone ID from environment variables), (6) send a notification via SNS topic ARN, (7) prevent re-failover for 10 minutes after a successful failover, and (8) log all state transitions with timestamps to syslog via `logger`. Handle all `aws` CLI failures gracefully.

---

## Q164 | ArgoCD → Disaster Recovery Architecture | Design Tradeoff

Design a geo-redundant ArgoCD architecture that survives an entire AWS region failure. Your setup manages 20 EKS clusters across `us-east-1` and `eu-west-1`. Compare: (1) active-passive ArgoCD pair (primary in `us-east-1`, standby in `eu-west-1` with shared Git repos) — how failover is triggered, the RTO/RPO implications of the ArgoCD Redis cache (stores application state) not being replicated, and whether ApplicationSets auto-reconcile after failover, versus (2) active-active ArgoCD where each region manages only its local clusters — how you prevent split-brain when the same ApplicationSet exists in both ArgoCD instances pointing to the same clusters, and the shared secrets problem (cluster credentials stored in ArgoCD secrets must exist in both instances). Which approach do you implement and what is your RTO?

---

## Q165 | Python Scripting → Testing & Mocking | Coding

Write a comprehensive test suite `test_aws_operations.py` using `pytest` and `moto` (AWS mock library) that tests a module `aws_operations.py` containing three functions: (1) `create_tagged_bucket(bucket_name: str, tags: dict, region: str) -> str` — test successful creation, test name collision behavior, test invalid region, (2) `rotate_access_keys(username: str) -> tuple[str, str]` — test rotation when user has 0, 1, and 2 existing keys (AWS limits to 2), test that the old key is deleted, (3) `get_running_instance_count(tag_key: str, tag_value: str) -> int` — test with 0, 5, and 1000 instances (pagination test). Each test must use `moto` decorators, not live AWS. Include a `conftest.py` with shared fixtures. Test parametrize where appropriate. Assert both return values and side effects (boto3 calls that should or should not have been made).

---

## Q166 | Kubernetes → Admission & Policy | Design Tradeoff

Compare three approaches to enforcing policies on Kubernetes resources at admission time: (1) Open Policy Agent (OPA) + Gatekeeper — `ConstraintTemplate` CRDs with Rego policies, how violations are reported vs enforced, the `dryrun` enforcement action, and how you test Rego policies before deploying, (2) Kyverno — how it uses a CEL-based or JMESPath policy syntax instead of Rego, the `generate` and `mutate` policies (not just validation), and how Kyverno's policy reports work, (3) Pod Security Admission (PSA) built into Kubernetes 1.25+ — the three profiles (`privileged`, `baseline`, `restricted`) and their limitations (no custom rules, all-or-nothing per namespace). For a platform team needing to enforce 50 custom policies including auto-injection of sidecars, which approach is most maintainable?

---

## Q167 | AWS → DynamoDB Streams | Coding

Write a Python Lambda function `dynamodb_stream_processor.py` that processes DynamoDB Streams events. The function must: (1) handle all three event types (`INSERT`, `MODIFY`, `REMOVE`) with separate logic, (2) for `INSERT` and `MODIFY`, deserialize DynamoDB's typed JSON (e.g., `{"S": "value"}`, `{"N": "123"}`) into plain Python dicts using `boto3.dynamodb.types.TypeDeserializer`, (3) for `REMOVE` events, archive the deleted item to S3 at `s3://archive-bucket/deletes/<table_name>/<partition_key>/<sort_key>/<timestamp>.json`, (4) for `MODIFY` events, compute a diff between `OldImage` and `NewImage` and publish only the changed fields to an SNS topic, (5) handle batch processing failures by returning `batchItemFailures` (partial batch response) so only failed records are retried. Include the IAM policy JSON needed for the Lambda execution role.

---

## Q168 | CI/CD → Feature Flags | Design Tradeoff

Your team wants to decouple deployments from releases using feature flags. Compare three implementation approaches for a microservices platform: (1) LaunchDarkly (SaaS) — SDK integration, the risk of a flag evaluation outage (LaunchDarkly goes down), relay proxy for local evaluation, and the SDK polling vs streaming mode, (2) Unleash (self-hosted) — deployment complexity, the `unleash-proxy` for edge evaluation, and how Unleash handles gradual rollouts with user bucketing, (3) build-your-own using DynamoDB + Lambda@Edge for flag evaluation with CloudFront. For each, describe: flag change propagation latency, how you audit who changed a flag and when, how you A/B test using the flag system, and the kill switch pattern for an incident response.

---

## Q169 | Prometheus & Grafana → SLOs | Scenario

Design a complete SLO framework for a payment API with SLO targets: 99.9% availability (max 43.8 min/month downtime) and 95% of requests under 200ms. Using the Prometheus/Grafana stack: write the PromQL expressions for error rate SLI and latency SLI, implement multi-window multi-burn-rate alerting (the Google SRE Workbook approach: 1h/5m window at 14× burn rate, 6h/30m at 6×, 3d/6h at 1×), show the complete Prometheus alerting rules YAML, create a Grafana panel showing error budget remaining (as a percentage and in minutes) with a 30-day rolling window, and explain what `1 - (1 - SLO_target)` means in the context of error budget consumption rate alerting.

---

## Q170 | Linux & Bash → Containers | 2AM Fire

At 2 AM a container running in Docker on an EC2 host consumes 100% of available disk space. The host has a 100GB root volume. `df -h` shows `/var/lib/docker` consuming 98GB. `docker system df` shows 60GB in images, 30GB in containers (volumes show 2GB). Walk through the diagnosis: using `docker ps -a --size` to find the container with a huge writable layer (logs written to container filesystem instead of a volume), using `docker logs --tail 100 <container>` to identify the log-spewing container, identifying containers with `Exited` status accumulating writable layers, the `docker system prune` commands in order of safety (stopped containers, dangling images, build cache), reconfiguring Docker's log driver to `json-file` with `max-size` and `max-file` limits in `/etc/docker/daemon.json`, and how to do this without restarting the Docker daemon.

---

## Q171 | AWS → Secrets Manager & Parameter Store | Design Tradeoff

Compare AWS Secrets Manager vs SSM Parameter Store (Standard and Advanced tiers) for managing 2,000 secrets across 50 microservices with these requirements: automatic rotation for RDS passwords, cross-account secret sharing, per-secret access control, and secrets injected into ECS task definitions and Lambda environment variables. Address: the cost difference ($0.40/secret/month for Secrets Manager vs $0.05/10k API calls for Parameter Store Standard), the rotation Lambda function architecture in Secrets Manager (single-user vs multi-user rotation strategy for zero-downtime RDS rotation), why Parameter Store `SecureString` cannot do rotation natively, cross-account access via resource policies (Secrets Manager supports this natively; Parameter Store does not), and the API call latency implications of fetching secrets at container startup time.

---

## Q172 | Kubernetes → Service Discovery | Conceptual

Explain how CoreDNS provides service discovery in Kubernetes. Describe: the DNS search domain chain (`<service>.<namespace>.svc.cluster.local` and the 5-record search path that `ndots:5` causes for every DNS lookup), how this `ndots` setting causes every non-FQDN lookup to make 5 DNS queries before succeeding (and the latency implication for outbound calls to external services), the fix using FQDN (trailing dot) for external DNS calls, how `NodeLocal DNSCache` as a DaemonSet reduces DNS latency and CoreDNS load, and what the `coredns` ConfigMap's `forward` plugin does for DNS queries that don't match `cluster.local`. Show the exact `resolv.conf` from a Pod and trace the DNS queries for both `postgres` (internal) and `api.stripe.com` (external).

---

## Q173 | ELK Stack → Cross-Cluster Search | Scenario

Your organization runs separate Elasticsearch clusters per environment (dev, staging, prod) and a central analytics cluster. The analytics team needs to run federated queries across all four clusters simultaneously. Configure Cross-Cluster Search (CCS): the `_cluster/settings` API call to add remote clusters using the `sniff` vs `proxy` mode (explain when each is appropriate), the RBAC permissions required on both the local and remote clusters for the user running the cross-cluster query, the syntax for querying `prod-cluster:logs-*,staging-cluster:logs-*` in a single request, the performance implications (CCS query fan-out to remote shards, network RTT per reduce phase), and the behavior when one remote cluster is unreachable (`skip_unavailable` setting). Show the complete cross-cluster configuration via the API.

---

## Q174 | GitHub Actions → Matrix Strategy | Config

Write a GitHub Actions workflow that runs a comprehensive test matrix for a Python library that must support: Python versions 3.9, 3.10, 3.11, 3.12; OS platforms ubuntu-latest, windows-latest, macos-latest; and dependency variants `minimal` (lowest supported versions) and `latest` (newest versions). The workflow must: (1) use `matrix` strategy with `exclude` to skip `windows + Python 3.9 + minimal` (known incompatibility), (2) use `include` to add an extra combination `ubuntu + Python 3.12 + latest + experimental: true`, (3) set `fail-fast: false`, (4) use `continue-on-error: ${{ matrix.experimental }}` for experimental jobs, (5) generate a test report artifact per matrix combination named `test-results-<os>-<python>-<deps>.xml`, and (6) have a final `report` job that downloads all artifacts and posts a consolidated summary using `actions/github-script`.

---

## Q175 | Terraform → Dynamic Blocks | Coding

Write a Terraform module that creates an AWS ALB with a variable number of listeners, target groups, and listener rules — all defined via input variables using `dynamic` blocks. The module must accept: `var.listeners` as a list of objects (each with `port`, `protocol`, `ssl_policy`, `certificate_arn`), `var.target_groups` as a map of objects (with `port`, `protocol`, `health_check` nested object), and `var.listener_rules` as a list of objects (with `listener_port`, `priority`, `conditions` as a list of `{field, values}`, and `target_group_key`). Use `dynamic` blocks for all variable-length configurations, `for_each` on maps for target groups, and local values to build the listener-to-target-group reference map. Output the ALB DNS name, zone ID, and a map of target group ARNs keyed by name.

---

## Q176 | AWS → Aurora Serverless | Design Tradeoff

Compare Aurora Serverless v1 vs v2 for a SaaS application with highly variable workloads: unpredictable traffic from 0 to 10,000 concurrent connections, a development environment that should scale to zero, and a production environment requiring < 1 second scaling response. Address: the fundamental architecture difference (v1 pauses/resumes entire database vs v2 scales continuously without pausing), the v1 limitation with VPC endpoints and connection pooling (connections drop during scaling), v2's minimum ACU of 0.5 (cannot truly scale to zero), the RDS Proxy integration that solves connection draining during scaling for both versions, and the cost model comparison: v2 ACU-hours vs v1 pause/resume billing for a dev environment that is used 2 hours/day.

---

## Q177 | Kubernetes → Helm | Debug

The following Helm chart `values.yaml` and template produce a Deployment where the `resources` block is always rendered with default values ignoring whatever the user passes in, replica count ignores the value file, and the liveness probe renders with an incorrect port. Identify all bugs, explain each root cause, and show the corrected template:

```yaml
# values.yaml
replicaCount: 3
image:
  repository: myapp
  tag: "1.0.0"
resources:
  limits:
    cpu: 500m
    memory: 512Mi
livenessProbe:
  port: 8080
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: {{ .Values.replicas }}
  template:
    spec:
      containers:
      - name: app
        image: {{ .Values.image.repository }}:{{ .Values.image.tag | default "latest" }}
        resources:
          {{- if .resources }}
          limits:
            cpu: {{ .Values.resources.limits.cpu }}
            memory: {{ .Values.resources.limits.memory }}
          {{- else }}
          limits:
            cpu: 100m
            memory: 128Mi
          {{- end }}
        livenessProbe:
          httpGet:
            path: /health
            port: {{ .Values.livenessProbe.Port }}
```

---

## Q178 | AWS → CloudFormation StackSets | Scenario

You manage AWS infrastructure for 30 accounts using CloudFormation StackSets deployed from a management account. A StackSet update to add a new security group rule is failing for 8 of 30 accounts with `OUTDATED` status, while 22 completed successfully. Describe the investigation: checking StackSet instance status and the specific failure reason per account using `describe-stack-instance`, the common causes of OUTDATED status (accounts that have manually modified the stack, accounts with IAM permission issues for the `AWSCloudFormationStackSetExecutionRole`, accounts where the stack was deleted outside StackSets), how to identify and fix drift between the StackSet template and the account-level stack using `detect-stack-drift`, and whether you should use `OVERRIDE` to force-deploy to failed accounts and the risk of doing so.

---

## Q179 | Python Scripting → Async Patterns | Coding

Write an async Python service `event_pipeline.py` that implements a multi-stage event processing pipeline using `asyncio.Queue`. The pipeline must have three stages: (1) `producer` — reads JSON events from a Kafka topic using `aiokafka`, validates against a Pydantic schema, and puts valid events into `queue_raw` (drop invalid events and increment a counter), (2) `enricher` — reads from `queue_raw`, enriches each event by calling an async HTTP endpoint with a 2-second timeout, and puts enriched events into `queue_enriched`, running 10 concurrent enrichment workers, (3) `writer` — reads from `queue_enriched`, batches events into groups of 100 or flushes every 5 seconds (whichever comes first), and bulk-writes to Elasticsearch using `elasticsearch-py` async client. The service must expose Prometheus metrics for events per stage and queue depths. Implement graceful shutdown on `SIGTERM`.

---

## Q180 | Kubernetes → Cluster Networking | 2AM Fire

At 2 AM inter-Pod communication in your EKS cluster degrades: Pod-to-Pod latency jumps from 0.5ms to 200ms and 5% packet loss appears. Pod-to-Service traffic is unaffected. The issue affects only cross-node traffic, not same-node traffic. Walk through the diagnosis: checking if the CNI plugin (AWS VPC CNI) is healthy on all nodes (`kubectl get pods -n kube-system -l k8s-app=aws-node`), inspecting `ipamd` logs for IP address exhaustion (ENI cannot be attached because the EC2 instance hit the ENI limit), using `ping` with `traceroute` to identify which hop is introducing latency, checking if a node's network interface is in a degraded state using `ethtool -S eth0 | grep -i error`, and whether a recent AWS VPC CNI version upgrade changed MTU settings causing fragmentation. What is the one-command health check that confirms IP exhaustion on a node?

---

## Q181 | CI/CD → Deployment Strategies | Design Tradeoff

Compare blue/green, canary, and rolling deployments for a stateful payment service that uses a PostgreSQL database with schema migrations. Address: how each strategy handles the database migration problem (the old version must read a schema that the new version migrated), the expand/contract migration pattern and which deployment strategy it is most compatible with, how blue/green works with Kubernetes (two Deployments + Service selector swap or Argo Rollouts), how canary traffic splitting at 1% requires a Service mesh or Ingress controller with weighted routing, and the specific failure mode for each strategy when the new version has a memory leak that only appears after 30 minutes under production load (blue/green fully exposes it; canary limits blast radius). Which strategy requires the least database migration risk?

---

## Q182 | ELK Stack → Elasticsearch Security | Config

Write the Elasticsearch API calls (via `curl` or Kibana DevTools) to set up the following security configuration for a multi-tenant log platform: (1) enable TLS between all nodes using `xpack.security.transport.ssl` (list the required settings in `elasticsearch.yml`), (2) create an index template that assigns index ownership via a custom metadata field, (3) create a role `app_team_alpha` with: read access to `logs-alpha-*` indices, document-level security to filter `{ "term": { "team": "alpha" } }`, field-level security excluding `ssn` and `credit_card` fields, and Kibana access to a specific Space, (4) create an API key for the team scoped to only that role, and (5) create an ILM policy that transitions to frozen tier after 30 days. Show all API request bodies.

---

## Q183 | AWS → Step Functions | Scenario

You have a complex data processing workflow: validate input → parallel enrichment from 3 external APIs → merge results → conditionally trigger either fast or batch processing → notify via SNS. Currently implemented as a Lambda function with `asyncio` that regularly times out at 15 minutes. Redesign using AWS Step Functions: model the workflow as an ASL (Amazon States Language) JSON definition with `Parallel` state for the 3 enrichment APIs, `Choice` state for the conditional routing, error handling with `Catch` and `Retry` on each enrichment step, and `Map` state for batch processing. Explain when you would use Standard vs Express Workflows (exactly-once vs at-least-once), how Step Functions handles the Lambda timeout issue (Step Functions max 1 year; each Lambda max 15 minutes), and how to inspect a failed execution.

---

## Q184 | Kubernetes → GitOps Patterns | Design Tradeoff

You are designing the GitOps workflow for a platform where developers own their service's Kubernetes manifests but platform engineers own cluster infrastructure (namespaces, RBAC, resource quotas). Compare two Git repository structures: (1) one monorepo where all service manifests and platform config live together — how you prevent a developer from modifying another team's namespace definition using CODEOWNERS, and how ArgoCD ApplicationSets generate apps from directory structure, versus (2) separate repos per team with a platform repo for infrastructure — how ArgoCD gets access to 50 separate repos, the credential management challenge, and how a platform change (adding a new LimitRange to all namespaces) requires 50 PRs vs 1 PR. Design the `CODEOWNERS` file structure, the Git branching model, and the PR review gates for production changes.

---

## Q185 | Python Scripting → Data Parsing | Coding

Write a Python script `log_anomaly_detector.py` that implements a simple statistical anomaly detection system for application metrics in log files. It must: (1) accept a log file path and metric name (e.g., `response_time_ms`) as CLI arguments, (2) parse log lines matching the pattern `<ISO8601_TIMESTAMP> <KEY>=<VALUE>` using regex, (3) compute a rolling 5-minute window mean and standard deviation for the metric, (4) flag data points more than 3 standard deviations from the rolling mean as anomalies, (5) for detected anomalies, look back 30 seconds and forward 30 seconds to provide context lines, (6) output anomalies as JSON to stdout and a human-readable summary to stderr, (7) support `--from` and `--to` timestamp filters, and (8) handle log files up to 50GB using line-by-line streaming with `mmap` for seek performance. No ML libraries — only statistics from `statistics` stdlib module.

---

## Q186 | Kubernetes → Vertical Pod Autoscaler | Scenario

You deploy VPA in `Auto` mode on a production `api-gateway` Deployment with 10 replicas. The next morning you find 3 Pods were evicted and restarted with new resource limits, causing brief unavailability. Explain the complete VPA eviction mechanism: how VPA's `Updater` component decides to evict Pods (comparing current requests to recommended requests, the `updatePolicy.updateMode`), why VPA and HPA must not both target CPU on the same Deployment, how `minReplicas: 1` in VPA's `updatePolicy` limits evictions, how to use VPA in `Off` mode (recommendation only) to gather data without causing evictions, and the specific VPA annotation that exempts individual Pods from being evicted. What is the safe production pattern for using VPA recommendations without automatic evictions?

---

## Q187 | AWS → Transit Gateway | Config

Write Terraform configuration for a hub-and-spoke network topology using AWS Transit Gateway. The configuration must: (1) create a TGW with `amazon_side_asn = 64512`, DNS support enabled, and default route table association disabled, (2) create 3 VPC attachments (hub VPC, spoke-1 VPC, spoke-2 VPC) using variables for VPC IDs and subnet IDs, (3) create separate TGW route tables for `hub-rt` and `spokes-rt`, (4) associate spokes with `spokes-rt` and the hub with `hub-rt`, (5) propagate all VPC attachment routes to `hub-rt`, (6) add a static route in `spokes-rt` pointing `0.0.0.0/0` to the hub VPC attachment (for centralized egress), and (7) add routes in each spoke VPC's route table pointing to the other spoke's CIDR via TGW. Use `for_each` for the spoke route tables.

---

## Q188 | Prometheus & Grafana → Cardinality | 2AM Fire

At 2 AM Prometheus memory usage spikes from 8GB to 32GB and the process OOM-kills. After restart it OOM-kills again within 5 minutes. The underlying cause is cardinality explosion. Walk through the forensics using Prometheus's TSDB status API (`/api/v1/status/tsdb`): interpreting `seriesCountByMetricName`, `seriesCountByLabelValuePair`, and `headSeriesCount`, identifying the metric and label combination causing explosion (e.g., a `request_id` label added to an HTTP metric), using `topk(10, count({__name__=~".+"}) by (__name__))` to find the top series by count, the immediate mitigation (dropping the high-cardinality label via `metric_relabel_configs` in the scrape config), and how to prevent this in future using Prometheus's `--storage.tsdb.max-block-duration` and cardinality limits in the remote write config.

---

## Q189 | ELK Stack → Beats | Coding

Write a complete Metricbeat configuration and a custom Python Metricbeat module (using the HTTP JSON input approach or a standalone Python script using the Beats `beat-python-framework`) that collects the following custom metrics from an internal API endpoint `http://localhost:9999/api/v1/stats` (which returns JSON): `active_sessions` (integer), `queue_depth` (integer), `processing_rate` (float, events per second), `error_rate` (float, errors per second). The metrics must be collected every 30 seconds, tagged with `service: myapp` and `environment: $ENV`, sent to Elasticsearch with the index pattern `metricbeat-myapp-%{+YYYY.MM.dd}`, and include an ingest pipeline that adds a `@timestamp` from the collected time, not the document time. Show the complete Metricbeat YAML and the ingest pipeline definition.

---

## Q190 | Kubernetes → Service Accounts & IRSA | Config

Write the complete manifests and AWS CLI/Terraform commands to implement IRSA (IAM Roles for Service Accounts) for a `data-processor` application that needs to read from an S3 bucket `data-lake-prod` and write to DynamoDB table `processing-results`. The setup must include: (1) an EKS OIDC provider association (Terraform `aws_iam_openid_connect_provider` resource), (2) an IAM role with a trust policy using `StringEquals` to bind it to the specific ServiceAccount `data-processor` in namespace `data-platform`, (3) the IAM policy with minimal permissions (specific S3 bucket ARN, specific DynamoDB table ARN, only `s3:GetObject`/`s3:ListBucket` and `dynamodb:PutItem`/`dynamodb:GetItem`), (4) the Kubernetes ServiceAccount with the `eks.amazonaws.com/role-arn` annotation, and (5) the Deployment spec showing the ServiceAccount reference and the AWS_SDK environment check. Explain what happens if the Pod is restarted and the IAM role is deleted.

---

## Q191 | AWS → Lambda Layers & Packaging | Design Tradeoff

Your Lambda functions use 15 shared Python libraries totaling 180MB, causing 3-second cold starts and making the deployment package approach unmanageable. Compare three packaging strategies: (1) Lambda Layers — how they are mounted at `/opt/python`, the 5-layer limit per function, the 250MB unzipped limit per function (layers + function code), and the cross-account layer sharing model, (2) Lambda container images (ECR) — how they eliminate the 250MB limit (10GB image support), how the Lambda service caches image layers via a read-only overlay filesystem, the cold start implication vs zip-based Lambda, and how `docker pull` from ECR works inside the Lambda execution environment, (3) a shared EFS mount for libraries — latency of EFS access at function start vs the Lambda ephemeral `/tmp`. For a 5-second cold start requirement, which approach do you choose?

---

## Q192 | CI/CD → Pipeline Security | 2AM Fire

At 2 AM your security monitoring detects that AWS credentials from your CI/CD pipeline were used to create an IAM user from an unusual IP address. The credentials belong to a GitHub Actions workflow that deploys to EKS. Walk through the incident response: immediate containment (revoke the specific OIDC-generated session token vs rotate the underlying IAM role trust policy — tradeoffs), using CloudTrail to trace all API calls made with those credentials in the last 24 hours, checking for persistence mechanisms (new IAM users, new access keys, new Lambda functions, new EC2 instances), auditing the GitHub Actions workflow file history for injected steps using `git log -p .github/workflows/deploy.yml`, and the architectural changes (pin all action versions to SHA, add OIDC audience restrictions, enable CloudTrail alerting for IAM creates) that prevent recurrence.

---

## Q193 | Linux & Bash → Advanced | Coding

Write a Bash script `k8s_node_drain_safe.sh` that safely drains a Kubernetes node with the following safeguards: (1) accepts `--node-name` and `--max-wait-minutes` (default 10) as arguments, (2) before draining, checks that the node exists and is not already cordoned (exits with a warning if already draining), (3) checks that draining the node would not violate any PodDisruptionBudget by running `kubectl get pdb --all-namespaces -o json` and computing disruption budget availability, (4) cordons the node, (5) lists all non-DaemonSet Pods on the node and estimates drain time based on count, (6) runs `kubectl drain` with `--ignore-daemonsets --delete-emptydir-data --timeout=${max_wait}m`, (7) monitors drain progress and displays a live countdown, (8) on completion, verifies zero non-DaemonSet Pods remain, and (9) on failure, automatically uncordons the node and sends an SNS alert. All `kubectl` calls must have `--kubeconfig` set from environment variable.

---

## Q194 | ArgoCD → Progressive Delivery | Scenario

You are implementing progressive delivery using Argo Rollouts (ArgoCD's sibling project) for a high-traffic service. Design the complete Rollout resource for a canary deployment: (1) 5% canary for 5 minutes → automated analysis using Prometheus metric `sum(rate(http_errors_total{app="myservice",version="{{args.canary-hash}}"}[1m])) / sum(rate(http_requests_total{app="myservice",version="{{args.canary-hash}}"}[1m]))` (abort if > 1% error rate), (2) 25% for 10 minutes with the same analysis, (3) 100% promotion. Show the complete Rollout YAML, the `AnalysisTemplate` CRD with the PromQL query, and the `AnalysisRun` lifecycle (how it is created, how failure triggers rollback). Explain the difference between Argo Rollouts analysis and ArgoCD sync health checks.

---

## Q195 | Prometheus & Grafana → Remote Write | Design Tradeoff

Your organization has 15 Prometheus instances (one per Kubernetes cluster) and wants to centralize metrics in a single queryable store using Remote Write. Compare the remote write target options: Thanos Receive (how it handles the remote write receiver, replication to multiple ingesters, the `--receive.hashrings-file` for sharding), Grafana Mimir (the remote write path through the Distributor, WAL in the Mimir ruler), and Victoria Metrics Cluster (the `vminsert` component, consistent hashing, no WAL in the receive path). For each, describe: what happens to a Prometheus instance if the remote write target is unavailable for 2 hours (WAL replay capacity, the `--storage.tsdb.retention.time` interaction), the compression and authentication options for the remote write connection, and the cardinality limit enforcement at the remote write receiver layer.

---

## Q196 | AWS → Networking | Debug

The following Terraform-generated VPC configuration creates a private subnet where EC2 instances cannot reach the internet despite having a NAT Gateway configured. Find every mistake and provide the corrected configuration:

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  map_public_ip_on_launch = true
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_eip" "nat" {}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.private.id
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}
```

---

## Q197 | Kubernetes → Observability | 2AM Fire

At 2 AM a 10-node Kubernetes cluster's control plane becomes unreachable. You have out-of-band access to the nodes. `kubectl` times out. Describe the complete recovery procedure: (1) on the control plane node, checking `systemctl status kube-apiserver` and its logs, (2) checking etcd health independently with `etcdctl --endpoints=https://127.0.0.1:2379 endpoint health` using the certs from `/etc/kubernetes/pki/etcd/`, (3) if etcd has lost quorum (3-node etcd, 2 nodes down), the procedure to force-create a new single-member cluster from the surviving member using `etcdctl snapshot restore` or the `--force-new-cluster` flag and the data loss risk, (4) if apiserver is up but kubelet on all worker nodes shows `connection refused` — checking the apiserver's certificate SANs against the IP being used, and (5) recovering from an expired Kubernetes API server certificate using `kubeadm certs renew`.

---

## Q198 | Python Scripting → Infrastructure | Coding

Write a Python script `vpc_flow_log_analyzer.py` that downloads and analyzes VPC Flow Logs from S3 to answer: (1) which source IPs have the most REJECTED traffic (potential port scanners or intrusion attempts), (2) which internal IP pairs have the most traffic by bytes (for network topology understanding), (3) which destination ports receive the most REJECTED traffic from outside the VPC CIDR range, and (4) traffic trends by hour for the last 7 days. The script must: use `boto3` to list and download flow log files from `s3://vpc-flow-logs-bucket/AWSLogs/<account_id>/vpcflowlogs/<region>/<year>/<month>/<day>/`, process gzipped files without fully extracting them using `gzip.open`, handle the VPC Flow Logs v2 format correctly (14-column header), process 100GB+ of data efficiently using chunked reads and `collections.Counter`, and output results as a Markdown report. No pandas — use only stdlib + boto3.

---

## Q199 | CI/CD → Artifact Promotion | Design Tradeoff

Design a complete artifact promotion pipeline for a microservices platform where the same Docker image must be promoted from dev → staging → prod without rebuilding (immutable artifacts). The pipeline must: (1) build and push to ECR with the Git SHA tag in the CI step, (2) run integration tests against the SHA-tagged image in staging, (3) only promote (re-tag as `v1.2.3` and copy to a prod ECR repo in a different AWS account) after explicit approval, (4) maintain a promotion audit trail showing who approved, when, and from which Git SHA, (5) support rollback to any previous production tag. Address: the ECR cross-account copy mechanism (`ecr:BatchGetImage` + `ecr:PutImage` vs ECR replication rules), how Cosign image signatures travel with the image during promotion, and the GitOps model where the image tag in the Helm values file is updated by the pipeline after promotion (not by developers).

---

## Q200 | Platform Engineering → SRE Practices | Design Tradeoff

You are establishing SRE practices from scratch at a company with 50 engineers, 20 microservices, and no on-call rotation. The services have no SLOs, no runbooks, and incidents are handled ad hoc. Design the complete SRE bootstrapping plan for the first 90 days: (1) how to measure current reliability without SLOs (USE method for infrastructure, RED method for services, deriving implicit SLOs from current error rates), (2) the process for writing the first 5 runbooks (what triggers a runbook, the structure: alert → symptoms → diagnosis steps → remediation → escalation), (3) designing the on-call rotation (follow-the-sun vs follow-the-service, alert volume targets — no more than 5 actionable pages per on-call shift, using Alertmanager inhibition to achieve this), (4) the blameless post-mortem process and the 5 questions every post-mortem must answer, and (5) how you prioritize reliability work against feature development using error budget policy. What does failure look like at each stage?

---
