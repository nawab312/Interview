## Q201 | Kubernetes → etcd | Conceptual

Explain etcd's Raft consensus algorithm as it applies to a 3-node Kubernetes control plane. Describe: how a leader is elected (randomized election timeouts, vote requests, quorum requirement), what happens to write requests during a leader election (they are rejected until a new leader is elected), the difference between a `follower`, `candidate`, and `leader` state, how etcd determines that a follower has fallen behind enough to require a snapshot transfer rather than log replay (`--snapshot-count` and the WAL compaction threshold), and what `etcdctl defrag` does and when it is dangerous to run in production. At what point does adding a 4th etcd member make the cluster less fault-tolerant than 3 members?

---

## Q202 | AWS → EKS → Pod Identity | Design Tradeoff

EKS Pod Identity (GA in late 2023) replaces IRSA for associating IAM roles with Kubernetes service accounts. Compare the two mechanisms: how IRSA uses OIDC federation with a JWT token mounted in the Pod vs how EKS Pod Identity uses the `eks-pod-identity-agent` DaemonSet to vend credentials via an in-cluster endpoint, the operational difference (IRSA requires an OIDC provider per cluster; Pod Identity uses a central EKS-managed endpoint), the trust policy syntax difference (IRSA uses `sts:AssumeRoleWithWebIdentity`; Pod Identity uses `eks.amazonaws.com` as a trusted service principal), cross-account role assumption support in each, and the specific scenario where IRSA is still required over Pod Identity (EKS Fargate, older Kubernetes versions, IAM Roles Anywhere).

---

## Q203 | Terraform → Backends | Failure Mode

A `terraform apply` is running in CI when the CI runner is forcibly terminated by an OOM kill. The S3 backend has a DynamoDB lock table. Describe the exact failure mode: the DynamoDB lock entry (`LockID`, `Info`, `Who`, `Created`) that remains after the runner dies, how subsequent `terraform plan` or `apply` runs fail with `Error acquiring the state lock`, the information in the lock entry that identifies the stale lock holder, the manual unlock command (`terraform force-unlock <LOCK_ID>`) and the risk of running it if a plan is still running elsewhere, how to set `lock_timeout` on the backend config to auto-retry, and the automation pattern that always calls `force-unlock` on CI job cancellation to prevent stale locks from blocking deployments.

---

## Q204 | Linux & Bash → eBPF & Observability | Conceptual

Explain how eBPF programs attach to kernel events to provide observability without modifying kernel source or inserting kernel modules. Describe: the eBPF verifier's role (why arbitrary code cannot be loaded — loop limits, stack size, pointer safety), how `kprobes` vs `tracepoints` vs `uprobes` differ as attachment points, how `bpf_map` types (`hash`, `array`, `perf_event_array`, `ringbuf`) pass data from kernel space to userspace, and how tools like `bpftrace`, `bcc`, and Cilium use eBPF differently. Write a `bpftrace` one-liner that traces all `execve` syscalls, prints the PID, UID, and full command line, and filters out calls from PID 1 to reduce init system noise.

---

## Q205 | GitHub Actions → Environments & Deployments | Config

Write a complete GitHub Actions workflow that implements environment-based deployment gates using GitHub Environments. The workflow must: (1) deploy to `staging` automatically on every push to `main`, (2) require manual approval from the `platform-team` GitHub team before deploying to `production`, (3) use environment-specific secrets (`DATABASE_URL`, `API_KEY`) via the `environment:` key, (4) post the deployment URL back to the PR that triggered the merge using the GitHub Deployments API via `actions/github-script`, (5) implement a 10-minute timeout that cancels the workflow if the approval job has not started, and (6) run a smoke test after each deployment that curls the deployed endpoint and fails the workflow on non-200. Include protection rule configuration as inline comments.

---

## Q206 | Prometheus & Grafana → Exemplars | Scenario

Your team has Prometheus metrics and Grafana Tempo traces deployed but they are not linked. Latency spikes appear in Grafana dashboards but finding the responsible trace requires manually cross-referencing timestamps. Implement exemplar-based trace linking: explain what a Prometheus exemplar is (a sample with extra labels including `traceID` attached to a histogram observation), how the OpenTelemetry SDK or `prometheus_client` Python library adds exemplars to histogram metrics, the Prometheus scrape config change needed for exemplar ingestion, how Grafana's Explore view uses exemplars to link a latency spike directly to the Tempo trace, and the storage implication (exemplars live in a fixed-size ring buffer, not in TSDB). Show the Python code to emit a histogram observation with a trace ID exemplar using `prometheus_client`.

---

## Q207 | AWS → Lambda → Cold Starts | 2AM Fire

At 2 AM your API Gateway + Lambda stack starts returning timeouts after a period of inactivity. CloudWatch shows `Init Duration` of 8–12 seconds on every invocation. The Lambda is Python 3.11 with a 512MB config, a 180MB deployment package, and imports `pandas`, `numpy`, and `boto3` at module level. Walk through the diagnosis and remediation: how cold starts are measured (`Init Duration` = class loading + runtime init + handler module import), specific Python import optimization techniques (lazy imports, deferred `importlib` loading), why 180MB packages cause slow cold starts (Lambda downloads the package before execution), Provisioned Concurrency cost model, and Lambda SnapStart (Java caveat). Quantify the expected improvement per optimization change.

---

## Q208 | Kubernetes → Scheduling → Priority | Scenario

Your cluster runs critical payment-processing Pods and batch analytics Pods. At peak load the scheduler cannot fit all Pods and batch jobs start failing. You want to ensure payment Pods always schedule even if it means evicting batch jobs. Design the complete priority-based scheduling setup: `PriorityClass` manifests for `critical-payment` (value: 1000000) and `batch-analytics` (value: 1000), how the scheduler uses priority for preemption (high-priority Pending Pod triggers eviction of lower-priority Pods), the `preemptionPolicy: Never` option and when to use it, how PodDisruptionBudgets interact with preemption (PDBs are respected), and what happens to a preempted batch Pod. Show both PriorityClass manifests and a Pod spec for each type.

---

## Q209 | ELK Stack → Hot-Warm-Cold Architecture | Config

Write the complete Elasticsearch configuration for a hot-warm-cold-frozen tiered architecture serving 90-day log retention at 1TB/day ingest. Include: (1) `elasticsearch.yml` node role configuration for hot (`data_hot`, `data_content`), warm (`data_warm`), cold (`data_cold`), and frozen (`data_frozen`) nodes, (2) the complete ILM policy JSON: rollover at 50GB or 1 day on hot, move to warm after 3 days with force-merge to 1 segment and shrink to 1 shard, move to cold after 14 days, move to frozen (searchable snapshot) after 30 days, and delete after 90 days, (3) the index template assigning the policy and `_tier_preference` routing, and (4) S3 snapshot repository config for the frozen tier. Explain the difference between `partially_mounted` and `fully_mounted` index states.

---

## Q210 | Python Scripting → AWS SDK | Coding

Write a Python module `aws_tag_enforcer.py` with a class `TagEnforcer` that: (1) scans all EC2 instances, RDS instances, S3 buckets, and Lambda functions across all enabled regions, (2) checks for required tags: `Environment`, `Owner`, `CostCenter`, and `Service`, (3) for non-compliant resources applies default tag values from a config dict (e.g., `{"Owner": "unknown", "CostCenter": "unallocated"}`), (4) never overwrites existing tag values, (5) generates a JSON compliance report: `{resource_type: {resource_id: {missing_tags: [], applied_tags: {}}}}`, (6) supports `--dry-run` mode, and (7) uses `concurrent.futures.ThreadPoolExecutor` with configurable concurrency to scan regions in parallel. Handle S3's global nature correctly. Include type hints throughout.

---

## Q211 | ArgoCD → Wave-Based Sync | Design Tradeoff

ArgoCD sync waves allow ordering resource creation within a single sync. Design the wave strategy for deploying a complete stack: `Namespace` and `ServiceAccount` (wave -3), CRDs (wave -2), ConfigMaps and Secrets (wave -1), Deployments and Services (wave 0), Ingress (wave 1), and a post-deployment smoke-test Job (wave 2). Explain: how ArgoCD determines wave completion (all resources in wave N are `Healthy` before wave N+1 starts), what happens when a wave-0 Deployment never becomes `Healthy` (sync blocks, wave 1 never starts), the difference between sync waves and sync hooks (hooks run outside the wave system), and why placing CRDs in wave -2 is safer than relying on ArgoCD's `--server-side-apply` CRD handling. Show the annotation syntax and a concrete example of wrong wave ordering causing a production outage.

---

## Q212 | AWS → Kinesis Data Firehose | Scenario

You have a Kinesis Data Firehose delivery stream sending JSON logs to S3 with a Lambda transformation function that enriches records with geo-location from IP. After a Lambda deployment, Firehose starts buffering records for 15 minutes then dropping 3% to the S3 error prefix. Diagnose: how Firehose's Lambda transformation works (Firehose sends batches, Lambda returns `Ok`, `Dropped`, or `ProcessingFailed` per record), what causes `ProcessingFailed` (Lambda timeout, response exceeding 6MB, malformed JSON), Firehose's retry behavior for Lambda failures (retries up to 24 hours then routes to error prefix), how the transformation buffer size and interval interact with Lambda timeout, and the exact Lambda response JSON structure Firehose expects.

---

## Q213 | Kubernetes → Custom Controllers | Coding

Write a simplified Kubernetes controller in Python using the `kubernetes` client library that watches `ConfigMap` objects with annotation `config-reloader.io/managed: "true"` and automatically restarts any `Deployment` in the same namespace annotated with `config-reloader.io/watch-configmap: "<configmap-name>"` by patching `spec.template.metadata.annotations` with a new timestamp. The controller must: use `watch.Watch()` for event streaming (not polling), handle `ADDED`, `MODIFIED`, and `DELETED` events, implement leader election using a Kubernetes `Lease` object for safe multi-replica operation, log all restart actions with the triggering ConfigMap name, and handle `ApiException` with retry logic. Include the RBAC manifests needed to run this controller.

---

## Q214 | CI/CD → Build Caching | Design Tradeoff

Your Docker builds in CI take 18 minutes due to `npm install` running on every build even when `package-lock.json` hasn't changed. Compare four caching strategies: (1) Docker layer caching with `--cache-from` using the previous image — explain why `COPY package-lock.json` must precede `RUN npm install`, (2) BuildKit inline cache (`--cache-to type=inline`) vs registry cache (`--cache-to type=registry`) — what metadata is stored and where, (3) GitHub Actions `actions/cache` mounting `node_modules` between runs with a `hashFiles('**/package-lock.json')` cache key, (4) a dedicated S3-backed remote cache for Nx or Turborepo monorepos. For each, state the cache invalidation trigger, maximum achievable cache hit rate, and behavior when the cache backend is unavailable.

---

## Q215 | Prometheus & Grafana → Loki Architecture | Conceptual

Explain Loki's distributed architecture: Distributor (consistent hashing of streams by label set), Ingester (in-memory chunks with WAL, stream ownership via hash ring), Querier (fan-out to ingesters and storage), Ruler (alerting on log data using LogQL), and Compactor (chunk compaction and retention). Describe: how Loki's label-based indexing differs from Elasticsearch's inverted index (Loki stores only labels in the index, full log content in compressed chunks in object storage), why Loki is cheap at scale but slow for full-text search, how `chunk_target_size` and `max_chunk_age` affect ingester memory pressure, and what happens when an ingester crashes before flushing (WAL replay mechanics). Write a LogQL query that counts error log rate by Kubernetes namespace over the last 5 minutes.

---

## Q216 | AWS → Organizations & SCPs | Config

Write AWS Organizations Service Control Policy JSON for four guardrails: (1) deny resource creation outside `us-east-1` and `eu-west-1` while allowing global services (IAM, CloudFront, Route53 — list exact service principals to exempt using `NotAction`), (2) deny any action that disables CloudTrail or modifies CloudTrail S3 bucket policies, (3) deny creation of IAM users with programmatic access keys to enforce SSO-only access, (4) require IMDSv2 by denying `RunInstances` when `MetadataOptions.HttpTokens` is not `required`. For each SCP, explain why `NotAction` with an explicit Deny is more maintainable than listing every allowed action, and show the exact condition key syntax.

---

## Q217 | Kubernetes → Ingress & Gateway API | Design Tradeoff

The Kubernetes Ingress API is being superseded by the Gateway API (stable in 1.28). Compare them for a platform managing 50 teams: (1) Ingress — the lack of role separation (one resource controls routing, TLS, and load balancing), the annotation proliferation problem, and the `IngressClass` mechanism added in 1.18, (2) Gateway API — the role separation model (`GatewayClass` owned by platform, `Gateway` owned by cluster ops, `HTTPRoute` owned by app teams), how `parentRefs` allow an app team's `HTTPRoute` to bind to a platform-managed `Gateway` without cluster-admin access, and the `ReferenceGrant` CRD for cross-namespace routing. For a platform migrating from nginx Ingress to AWS Load Balancer Controller with Gateway API, describe the migration path and any traffic continuity risks.

---

## Q218 | Python Scripting → Concurrency | Coding

Write a Python script `parallel_terraform.py` that orchestrates Terraform applies across multiple directories in dependency-aware order. It must: (1) read a YAML `dependencies.yml` mapping module paths to their dependencies (a DAG), (2) use `graphlib.TopologicalSorter` (Python 3.9+) to determine execution order, (3) execute independent modules in parallel using `asyncio.create_subprocess_exec` — max parallelism configurable via `--max-parallel`, (4) stream stdout/stderr from each apply in real time prefixed with the module name, (5) stop remaining applies if any apply fails but let in-progress applies finish, (6) output a final summary table with module name, status, duration, and exit code, and (7) generate a DOT format dependency graph file. Handle DAG cycles with `graphlib.CycleError` and exit cleanly.

---

## Q219 | ELK Stack → Elasticsearch Aggregations | Coding

Write an Elasticsearch Query DSL request body that: finds the top 10 services by total request count in the last 7 days, for each service shows hourly request rate as a time-series using `date_histogram` with `calendar_interval: 1h`, computes the percentage of requests with `status_code >= 500`, computes p95 and p99 of `response_time_ms` using `percentiles` aggregation, and uses a `bucket_selector` pipeline aggregation to filter out services with fewer than 1000 total requests. The response must use `size: 0` (no documents, aggregations only). Show the complete JSON request body and explain the nesting order of aggregations and why the `bucket_selector` must be a sibling of the sub-aggregations it filters.

---

## Q220 | AWS → CloudFront → Lambda@Edge | Scenario

Your CloudFront distribution serves a React SPA from S3. You need to: (1) redirect all requests to `index.html` for client-side routing, (2) add security headers (`Content-Security-Policy`, `X-Frame-Options`, `Strict-Transport-Security`) to all responses, (3) implement A/B testing by routing 10% of users to a `v2/` S3 prefix based on a cookie, and (4) block requests from 50 IP ranges. Evaluate each requirement against Lambda@Edge vs CloudFront Functions (viewer-only, limited JS runtime). Implement (1) and (2) as CloudFront Functions (show the complete JS code) and (3) as a Lambda@Edge origin request function (show the Node.js code). Explain why (4) should use WAF instead of either compute option and what the cost difference is.

---

## Q221 | GitHub Actions → Composite Actions | Coding

Write a composite GitHub Action at `.github/actions/setup-eks-tools/action.yml` that installs and configures Kubernetes tooling consistently. The action must: install `kubectl`, `helm`, `kubeconform`, and `kustomize` at specified versions (inputs with defaults), authenticate to EKS by calling `aws eks update-kubeconfig` (requires `cluster-name` and `aws-region` inputs), verify the connection with `kubectl cluster-info`, and cache all tool binaries between runs using `actions/cache` keyed on tool versions. The action must work on both `ubuntu-latest` and `self-hosted` Linux runners, detect runner architecture (amd64 vs arm64), and download the correct binaries. Include proper `inputs` and `outputs` definitions (output the `kubeconfig` path). Show the complete `action.yml`.

---

## Q222 | AWS → AppMesh vs Istio | Design Tradeoff

You are choosing a service mesh for a 200-service platform running entirely on EKS. Compare AWS App Mesh vs Istio: App Mesh's managed control plane (no control plane to operate) vs Istio's `istiod` which you must run and upgrade, App Mesh's native AWS integration (CloudMap, X-Ray, CloudWatch) vs Istio's ecosystem (Prometheus, Jaeger, Kiali), the mTLS model (App Mesh requires ACM Private CA vs Istio's built-in CA), App Mesh's limited traffic policy features vs Istio's `DestinationRule`/`VirtualService` expressiveness, and the migration path if you need to move from App Mesh to Istio later. For a team prioritizing minimal operational overhead with full AWS observability integration, which would you choose in 2024 and what is the long-term risk of that choice?

---

## Q223 | Kubernetes → Security → Pod Security | 2AM Fire

At 2 AM a security alert fires: a container in the `payments` namespace is reading files from `/proc/<host_pid>/` indicating a possible container escape or privileged container abuse. The container is `Running`. Describe immediate containment and forensics: the fastest way to isolate the Pod without deleting it (applying a NetworkPolicy that denies all ingress/egress), capturing process state (`kubectl exec` to run `ps aux`, checking `/proc/1/status`, inspecting `/proc/1/root` for host filesystem access), checking the Pod spec for `securityContext.privileged: true`, `hostPID: true`, `hostPath: /` volumes, and identifying which PodSecurityAdmission enforcement mode would have prevented this Pod from ever being scheduled. What `audit` mode PodSecurityAdmission data would have given you advance warning before it reached `enforce` mode?

---

## Q224 | Terraform → Sensitive Variables | Config

Write a Terraform configuration demonstrating best practices for sensitive values. The config must: (1) declare `variable "db_password"` with `sensitive = true` showing how it appears (redacted) in `terraform plan`, (2) use `data "aws_secretsmanager_secret_version"` as the preferred pattern over accepting passwords as variables, (3) pass the sensitive value to an `aws_db_instance` resource and mark the `output "db_endpoint"` as `sensitive` to prevent logging, (4) show why storing sensitive values in `terraform.tfvars` committed to Git is dangerous and the `.gitignore` pattern to prevent it, (5) demonstrate using `TF_VAR_db_password` environment variable as an alternative. Explain what `sensitive = true` does and does NOT protect against — it is output masking only, not encryption.

---

## Q225 | Prometheus & Grafana → Alerting Rules | Debug

The following Prometheus alerting rule and Alertmanager config cause alert storms: the alert fires 50+ times per minute for the same condition, recovers and re-fires within seconds, and Alertmanager deduplication is not working. Identify every bug, explain the root cause of each, and provide corrected versions:

```yaml
groups:
- name: api-alerts
  rules:
  - alert: HighErrorRate
    expr: |
      sum(rate(http_requests_total{status=~"5.."}[1m]))
      /
      sum(rate(http_requests_total[1m])) > 0.05
    for: 0m
    labels:
      severity: critical
      team: {{ $labels.team }}
    annotations:
      summary: "Error rate is {{ $value }}"
  - alert: HighErrorRate
    expr: http_error_rate > 0.05
    for: 30s
```

```yaml
route:
  group_by: ['alertname']
  group_wait: 0s
  group_interval: 10s
  repeat_interval: 1m
  receiver: slack
receivers:
- name: slack
  slack_configs:
  - api_url: "https://hooks.slack.com/..."
    channel: "#alerts"
    send_resolved: false
```

---

## Q226 | AWS → ElastiCache | Failure Mode

Your ElastiCache Redis cluster (cluster mode disabled, 1 primary + 2 read replicas) undergoes a primary node failure. Describe the automatic failover sequence: how ElastiCache detects the primary failure (heartbeat timeout via ElastiCache's quorum — not Redis Sentinel), typical failover duration (30–60 seconds), what happens to in-flight write operations during the window, how the DNS primary endpoint updates after failover, why client connection pools that cache the resolved IP (not the DNS name) continue to fail after failover until restarted, and the specific `redis-py` client configuration (`retry_on_timeout`, `socket_keepalive`, `health_check_interval`) that enables automatic reconnection. How does enabling cluster mode change the failover story?

---

## Q227 | Linux & Bash → Systemd | Scenario

A critical service `payment-processor` occasionally crashes silently (exits with code 0 but is broken) and other times hangs (process running but not responding). Design the complete systemd unit file that handles both cases: `Type=notify` with `sd_notify(READY=1)` for readiness signaling, a watchdog using `WatchdogSec=30` that the service must ping via `sd_notify(WATCHDOG=1)` to prove liveness, `Restart=always` with `RestartSec=5`, `StartLimitBurst=5` and `StartLimitIntervalSec=300` to prevent restart storms, `MemoryMax=2G` and `CPUQuota=200%` for resource control, and an `ExecStartPre` that verifies a dependent service is reachable before starting. Show the complete `.service` file and explain how the watchdog catches the silent hang that `Restart=on-failure` would miss.

---

## Q228 | Kubernetes → CSI Drivers | Conceptual

Explain the Container Storage Interface architecture in Kubernetes: the role of `external-provisioner` (watches PVCs and calls `CreateVolume` RPC), `external-attacher` (calls `ControllerPublishVolume`), `node-driver-registrar` (registers the CSI driver with kubelet), and the driver's own node plugin (implements `NodePublishVolume` to mount into the Pod). Describe what happens at each RPC boundary when a Pod with a PVC is scheduled to a node: the `VolumeAttachment` resource lifecycle, and the kubelet's CSI call to the node plugin. What is the difference between `ReadWriteOnce`, `ReadOnlyMany`, and `ReadWriteMany` at the CSI driver level, and why does the AWS EBS CSI driver not support `ReadWriteMany`? What CSI driver would you use for `ReadWriteMany` on EKS?

---

## Q229 | GitHub Actions → Secrets Rotation | Coding

Your GitHub organization has 300 repositories each storing a `DEPLOY_TOKEN` secret granting write access to your container registry. The token is shared across all repos and expires every 90 days, requiring manual rotation. Write a Python script `rotate_github_secrets.py` that automates rotation: (1) generates a new registry token, (2) fetches each repo's public key using `GET /repos/{owner}/{repo}/actions/public-key`, (3) encrypts the new secret using `PyNaCl` (`libnacl` sealed box with the repo's public key), (4) uploads the encrypted secret via `PUT /repos/{owner}/{repo}/actions/secrets/{secret_name}`, (5) iterates across all 300 repos with pagination, (6) handles rate limiting (GitHub allows 1000 requests/hour for secrets API). Show the complete script with encryption logic and error handling.

---

## Q230 | ELK Stack → Performance Optimization | Scenario

Your Elasticsearch cluster handles 50,000 writes/second but indexing latency spikes to 5 seconds during peak hours. `_nodes/stats/indices` shows `indexing.index_time_in_millis` growing linearly. Diagnose and fix: the role of the indexing buffer (`indices.memory.index_buffer_size`, default 10% of heap), how translog fsync (`index.translog.durability: request` vs `async`) affects write latency, why too many dynamic fields cause mapping updates that lock the index, the `refresh_interval` change from 1s to 30s during bulk ingest windows and how to automate it with ILM, the `thread_pool.write.size` setting (should not exceed CPU core count), and how to use the `_bulk` API correctly (optimal batch size: 5–15MB). Show the index settings API call that applies all these optimizations simultaneously.

---

## Q231 | AWS → IAM Identity Center | Design Tradeoff

Your company is migrating from IAM users with long-lived access keys to AWS IAM Identity Center (SSO) for 200 engineers across 30 AWS accounts. Design the complete permission set architecture: how permission sets map to IAM roles in each account (automatic provisioning), the difference between AWS managed policy permission sets and custom inline policy permission sets, implementing JIT privilege escalation (a `RequestElevation` permission set active for 4 hours requiring MFA re-auth), SCIM provisioning integration with Okta for automatic user/group sync, and the specific Terraform resource (`aws_ssoadmin_account_assignment`) that assigns permission sets to accounts. How do you implement break-glass access to production that is auditable, time-limited, and does not require a standing IAM role with admin access?

---

## Q232 | Kubernetes → Network Policy Advanced | Config

Write the complete NetworkPolicy manifests for a three-tier application in namespace `app` (`frontend`, `backend`, `database` Pods) plus a `monitoring` namespace running Prometheus: (1) `frontend` receives traffic only from the Ingress controller namespace on port 443, (2) `frontend` initiates connections only to `backend` on port 8080, (3) `backend` receives only from `frontend` on 8080 and Prometheus on 9090, (4) `backend` initiates connections only to `database` on 5432 and to DNS on UDP 53, (5) `database` receives only from `backend` on 5432 and Prometheus on 9187, (6) no Pod can initiate connections to the EC2 metadata endpoint `169.254.169.254`. Show all policies with comments explaining why each rule is necessary and what breaks if it is missing.

---

## Q233 | CI/CD → Release Management | Design Tradeoff

Design a semantic versioning and release management system for a monorepo with 20 microservices. The system must: auto-generate versions from Conventional Commits (`feat:` → minor, `fix:` → patch, `BREAKING CHANGE:` → major), generate per-service changelogs only for commits touching that service's directory, create GitHub Releases with artifact attachments (Docker image digests, Helm chart tarballs), enforce immutable version tags, and support hotfix workflows (patching a specific older version without picking up newer features). Compare `semantic-release` (Node.js, direct tag-on-merge) vs `release-please` (Google's Go tool, PR-based release model) vs Conventional Changelog for monorepos. What is the branching strategy implication of each tool's release model?

---

## Q234 | Linux & Bash → Disk & Filesystem | 2AM Fire

At 2 AM a production server reports `No space left on device` but `df -h` shows the root filesystem at only 60% full. Describe the full investigation: checking inode exhaustion with `df -i`, using `find / -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head` to find the directory consuming the most inodes, checking for deleted-but-open files holding disk space with `lsof | grep deleted`, how a process holding an open FD to a deleted 50GB log file keeps the space allocated, the `fuser` and `lsof -p <pid>` commands to identify the file handle without restarting the process, checking for filesystem errors with `dmesg | grep -i 'ext4\|xfs\|i/o error'`, and the `tune2fs -l /dev/xvda1 | grep 'Reserved block'` reserved block space that can be reclaimed immediately without downtime.

---

## Q235 | ArgoCD → Multi-Tenancy | Design Tradeoff

You manage an ArgoCD instance serving 30 teams each with 5–20 applications. Teams must not see or modify each other's applications. Design the complete multi-tenancy model: ArgoCD Projects as the isolation boundary (`spec.destinations` limits deployment targets, `spec.sourceRepos` limits Git repos), RBAC where team leads have `update` and `sync` but not `delete` in their project, how a platform admin creates Projects and grants access without giving teams cluster-admin, and the `AppProject` manifest for a sample team. Explain why ArgoCD multi-tenancy is not a full security boundary (a team with Git write access could deploy a privileged Pod if the destination namespace PSA policy allows it), and what Kubernetes-level controls must complement ArgoCD's Project isolation.

---

## Q236 | AWS → OpenSearch | Scenario

Your Amazon OpenSearch Service domain is at 85% disk utilization and write operations are being rejected with `ClusterBlockException [FORBIDDEN/8/index write]`. The domain has 3 data nodes with 1TB EBS volumes each. Describe the immediate remediation: how OpenSearch's disk-based watermarks work (`cluster.routing.allocation.disk.watermark.low/high/flood_stage`), the API call to temporarily disable the flood-stage block while you add capacity, how to increase EBS volume size using `UpdateDomainConfig` without downtime, the ISM policy for automatic index deletion equivalent to Elasticsearch's ILM, and how to use `_shrink` and `_clone` APIs on read-only indices to reduce shard count and reclaim space immediately. What is the minimum viable disk headroom you should maintain, and what CloudWatch alarm threshold enforces it?

---

## Q237 | Kubernetes → Debugging Tools | Coding

Write a Python script `k8s_top_resources.py` implementing a `kubectl top`-like tool using the Kubernetes Metrics API. The script must: (1) authenticate using in-cluster config or kubeconfig (auto-detect), (2) query `apis/metrics.k8s.io/v1beta1/pods` with optional namespace filtering, (3) display a live-updating table (refresh every 5 seconds using `rich.live`) showing Pod name, namespace, container name, CPU in millicores, memory in Mi, and percentage of requested CPU and memory (cross-referenced with Pod spec requests from the core API), (4) support `--sort-by cpu|memory`, `--namespace`, and `--top N` flags, (5) highlight Pods using more than 90% of their requested resources in red, and (6) handle Metrics Server not being installed with a graceful error message. Use the `kubernetes` Python client library.

---

## Q238 | AWS → Cognito | Conceptual

Explain AWS Cognito's two main components: User Pools (authentication — user storage, sign-in, MFA, password reset) vs Identity Pools (authorization — exchanges tokens for temporary AWS credentials via `AssumeRoleWithWebIdentity`). Describe: the JWT token flow (User Pool issues ID token, access token, and refresh token), how the hosted UI handles OAuth2 authorization code flow, the difference between an app client with and without a client secret for server-side vs SPA usage, how Identity Pools support unauthenticated guest access with a separate IAM role, and the adaptive authentication feature. Walk through the complete flow of a mobile app user logging in, getting an ID token, and using it to access an S3 bucket via Identity Pool-vended temporary credentials. What is the token expiration strategy and how do you handle silent refresh?

---

## Q239 | CI/CD → Testing Strategy | Design Tradeoff

Design a complete test strategy for a microservices platform where integration tests take 45 minutes and flaky tests cause 30% of CI failures. Address: (1) the test pyramid for microservices (unit → contract → integration → E2E) and recommended ratio, (2) consumer-driven contract testing with Pact — how the consumer writes a contract, the provider verifies it, and how this eliminates integration environments for API compatibility testing, (3) test environment strategy (shared staging vs ephemeral environments per PR using namespace isolation), (4) flaky test detection and quarantine using a `@flaky` marker and separate CI job that retries 3 times before failing, (5) local development using `docker-compose` with service stubs vs Telepresence for live cluster debugging. What is the fastest achievable integration test time with full parallelization, and what is the irreducible minimum?

---

## Q240 | Prometheus & Grafana → Grafana Alloy | Conceptual

Grafana Alloy (formerly Grafana Agent) replaces the Prometheus Agent for metrics, logs, and traces collection. Explain: how Alloy uses a `river`/`alloy` pipeline configuration language (source → processor → sink components) that differs fundamentally from Prometheus's scrape config, how the `prometheus.scrape` component replaces a standard Prometheus scrape config, how Alloy's clustering mode divides scrape targets among instances using a hash ring without duplication, the `otelcol.*` components that receive OpenTelemetry data and forward to Tempo/Mimir/Loki, and why Alloy as a DaemonSet with `loki.source.kubernetes` replaces Promtail. Show an Alloy config snippet that scrapes Kubernetes Pod annotations for service discovery and forwards metrics to a remote Prometheus-compatible endpoint with TLS.

---

## Q241 | AWS → CloudWatch → Logs Insights | Coding

Write AWS CloudWatch Logs Insights query strings for the following use cases against an ECS application log group: (1) find the top 20 ECS task ARNs by error count in the last 24 hours, filtering for log lines containing `ERROR` or `Exception`, (2) parse a structured JSON log `{"level":"error","duration_ms":423,"endpoint":"/api/pay"}` using the `parse` command and compute p95/p99 of `duration_ms` per `endpoint`, (3) detect log lines where `duration_ms > 1000` and find all log lines from the same `requestId` within 30 seconds using `filter` and the `diff` command, (4) count log events per 5-minute bucket over the last 6 hours as a time-series using `bin()`. Show the complete query string for each use case.

---

## Q242 | Kubernetes → Pod Networking | Debug

Two Pods in the same namespace can reach the Kubernetes API server and external internet but cannot reach each other via Pod IP even though NetworkPolicy permits it. Both Pods are on different nodes. The CNI is Calico. Describe the exact diagnostic sequence: using `kubectl exec` to run `ping` and capturing the failure mode (timeout vs ICMP unreachable), using `calicoctl node status` to check BGP peering between nodes, inspecting `calicoctl get workloadendpoints` for endpoint policy, verifying `ip_forward` is enabled on nodes (`cat /proc/sys/net/ipv4/ip_forward`), using `tcpdump -i any -n host <pod-ip>` on both source and destination nodes to trace where packets are dropped, and verifying the node routing table has a route for the remote Pod CIDR. What does a BGP peering failure look like in `calicoctl node status` and what is the fix?

---

## Q243 | AWS → SQS → FIFO Queues | Design Tradeoff

Compare SQS Standard vs FIFO queues for an order management system where each order must be processed exactly once and events for the same order must be in sequence (create → pay → fulfill → ship). Address: how FIFO `MessageGroupId` provides per-group ordering with a single consumer per group, the FIFO throughput ceiling (3,000 messages/second with batching vs Standard's near-unlimited throughput), the `MessageDeduplicationId` and its 5-minute deduplication window, why a FIFO DLQ must also be a FIFO queue, the `ReceiveRequestAttemptId` for deduplication of receive calls, and the architectural implication of `MessageGroupId` contention when all order events share the same group. At what order volume does FIFO's throughput limit force you to shard by `MessageGroupId` strategy?

---

## Q244 | Terraform → Count vs For_Each | Design Tradeoff

A team uses `count` to create multiple similar AWS resources but encounters ordering and deletion issues. Explain: why `count` is fragile (removing an element from the middle of the list causes Terraform to destroy and recreate all subsequent resources), how `for_each` with a map or set solves this by using stable keys as resource addresses, the specific error `Error: Invalid for_each argument` that occurs when the `for_each` value is unknown at plan time (e.g., depends on a resource attribute not yet created), and the `toset()` vs `tomap()` function difference for `for_each` inputs. Show a before/after migration from `count` to `for_each` for an `aws_iam_user` resource, including how the `terraform state mv` commands must be run to avoid destroying existing users.

---

## Q245 | Kubernetes → Horizontal Pod Autoscaler | Debug

An HPA targeting a Deployment shows `<unknown>/50%` for the CPU metric in `kubectl get hpa` and never scales. The Pods are running and consuming CPU. Walk through every possible cause: Metrics Server not installed or not returning metrics for these Pods, the HPA controller unable to reach the Metrics API (RBAC issue — `system:aggregated-metrics-reader` ClusterRoleBinding), the Deployment's Pod spec missing `resources.requests.cpu` (HPA computes utilization as current usage / request; without a request there is no denominator), the `--horizontal-pod-autoscaler-sync-period` on the controller-manager, and the `scaleTargetRef` pointing to the wrong Deployment name or apiVersion. Provide the exact `kubectl` commands that confirm or rule out each cause, and show the corrected HPA YAML with a working `metrics` block.

---

## Q246 | Python Scripting → Log Parsing | Coding

Write a Python script `nginx_log_analyzer.py` that parses nginx access logs in combined log format and generates a security-focused anomaly report. The script must: (1) accept a log file path or stdin, (2) parse each line using a compiled regex extracting: remote IP, timestamp, HTTP method, URI, status code, bytes sent, referrer, and user agent, (3) detect and report: IPs making more than 1000 requests in any 5-minute sliding window (potential DDoS), URIs matching common web attack patterns (`../`, `<script>`, `UNION SELECT`, `eval(`), user agents matching known scanner signatures (`nikto`, `sqlmap`, `nmap`), and unusually large response bodies (bytes > 10MB), (4) output a JSON report with severity (`critical`, `warning`, `info`) per finding, (5) process 10GB+ files using line-by-line streaming with no full-file loading. Use only stdlib — no third-party dependencies.

---

## Q247 | AWS → Route 53 Resolver | Scenario

Your EKS workloads need to resolve on-premises DNS hostnames (e.g., `db.internal.corp`) for database connections over Direct Connect. Currently all DNS from EKS Pods uses CoreDNS which can only resolve cluster-internal names and public DNS. Design the complete hybrid DNS architecture: Route 53 Resolver inbound endpoint (allows on-premises resolvers to forward queries to your VPC), Route 53 Resolver outbound endpoint (allows VPC resources to forward queries to on-premises resolvers), the Resolver rule that forwards `internal.corp` queries to your on-premises DNS server IP, how this Resolver rule is shared across 10 VPCs using AWS RAM (Resource Access Manager), and the CoreDNS ConfigMap change that forwards `internal.corp` queries to the Route 53 Resolver outbound endpoint IP rather than the upstream public resolver.

---

## Q248 | CI/CD → Jenkins → Declarative Pipelines | Debug

The following Jenkins declarative pipeline fails intermittently with `FlowInterruptedException` during the deploy stage, and the `post { failure { } }` block sometimes doesn't execute. Identify all bugs, explain each root cause, and provide the corrected pipeline:

```groovy
pipeline {
    agent any
    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        AWS_REGION = "us-east-1"
    }
    stages {
        stage('Build') {
            steps {
                sh 'docker build -t myapp:${IMAGE_TAG} .'
            }
        }
        stage('Test') {
            parallel {
                stage('Unit') { steps { sh 'make test-unit' } }
                stage('Lint') { steps { sh 'make lint' } }
            }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps {
                withCredentials([string(credentialsId: 'aws-token', variable: 'AWS_TOKEN')]) {
                    sh '''
                        aws ecs update-service --cluster prod --service myapp \
                        --force-new-deployment
                        sleep 120
                        aws ecs wait services-stable --cluster prod --service myapp
                    '''
                }
            }
        }
    }
    post {
        failure {
            slackSend(message: "Build ${BUILD_NUMBER} failed")
        }
        always {
            sh 'docker rmi myapp:${IMAGE_TAG} || true'
        }
    }
}
```

---

## Q249 | Prometheus & Grafana → Federation | Design Tradeoff

You have 8 Prometheus instances — one per region, each scraping 500k time series — and need a global view for cross-region dashboards and alerts. Compare three federation approaches: (1) Prometheus hierarchical federation (a global Prometheus scrapes `/federate` endpoint from each regional Prometheus with `match[]` selectors) — explain the performance implications and why federating raw series doesn't scale, (2) Thanos Query with Sidecar — how the global Thanos Query fans out to regional Thanos Sidecars, the gRPC StoreAPI protocol, and how deduplication works when the same metric exists in two regions, (3) Grafana Mimir with remote write from all 8 instances into a central Mimir cluster. For each approach, describe what happens when one regional Prometheus is down during a global dashboard query.

---

## Q250 | Kubernetes → Pod Disruption | 2AM Fire

At 2 AM your Kubernetes cluster undergoes a node drain as part of scheduled maintenance. A PodDisruptionBudget on your `checkout-api` Deployment (3 replicas, `maxUnavailable: 1`) should allow the drain to proceed while keeping 2 replicas running. Instead, `kubectl drain` hangs indefinitely. Walk through the diagnosis: checking `kubectl get pdb checkout-api -o yaml` to see `disruptionsAllowed` (if 0, why — readiness probe failures reducing `currentHealthy` below `minAvailable`), checking if an earlier PVC detach is keeping a Pod in `Terminating` state indefinitely (blocking drain progress), checking for a `preStop` hook that is sleeping longer than `terminationGracePeriodSeconds`, and the `kubectl drain --force --disable-eviction` escape hatch and its risk. What is the one PDB misconfiguration that permanently blocks all node drains?

---

## Q251 | ELK Stack → Alerting with Watcher | Config

Write an Elasticsearch Watcher definition (using the Watch API) that: (1) runs every 5 minutes, (2) queries the `logs-*` index for documents in the last 5 minutes where `level: ERROR` and `service: payment-api`, (3) triggers an alert if more than 50 error documents are found, (4) uses a `condition.script` to also check that the error count increased by more than 20% compared to the previous 5-minute window (using a second aggregation query), (5) sends a Slack webhook notification with the error count and top 3 error messages (using `transform.search` to get representative messages), and (6) implements throttle to suppress repeated alerts for 30 minutes. Show the complete Watch JSON including `trigger`, `input`, `condition`, `transform`, `actions`, and `throttle_period`.

---

## Q252 | AWS → S3 → Lifecycle & Intelligent Tiering | Design Tradeoff

You manage an S3 bucket storing 5PB of data: 10% accessed daily (hot), 30% accessed monthly (warm), and 60% never accessed after 90 days (cold/archive). Compare four cost optimization strategies: (1) manual lifecycle rules transitioning to S3-IA after 30 days, Glacier after 90 days, and Deep Archive after 365 days — show the lifecycle XML and the retrieval latency tradeoffs, (2) S3 Intelligent-Tiering — how the monitoring tier works, the minimum object size (128KB), the monthly monitoring fee per object ($0.0025/1000 objects), and break-even calculation vs manual lifecycle, (3) S3 Glacier Instant Retrieval vs Flexible Retrieval for the 60% archive tier, (4) using S3 Storage Lens to identify actual access patterns before committing to a storage class strategy. What is the one scenario where Intelligent-Tiering costs more than manual lifecycle rules?

---

## Q253 | Linux & Bash → AWK & Stream Processing | Coding

Write a Bash script `metrics_aggregator.sh` that reads a stream of metrics from stdin in the format `<timestamp_epoch> <metric_name> <value> <tag_key>=<tag_value>` (one per line, potentially millions of lines) and: (1) computes min, max, mean, and p95 per `metric_name` per `tag_value` using only `awk` (no Python, no sort+uniq tricks — use awk arrays), (2) outputs a TSV report sorted by metric name then tag value, (3) supports a `--window <seconds>` flag that only aggregates metrics within the last N seconds relative to the maximum timestamp seen in the stream, (4) handles missing or malformed lines gracefully by skipping them and counting them, and (5) outputs a footer line showing total lines processed, lines skipped, and wall-clock processing time. The script must process 1 million lines in under 10 seconds on a modern CPU. Use only awk and standard Unix tools.

---

## Q254 | Kubernetes → Service Topology | Conceptual

Explain Kubernetes topology-aware routing (formerly topology-aware hints, stable in 1.27). Describe: how the `EndpointSlice` controller adds `hints.forZones` to endpoints based on the zone of the backing Pod, how kube-proxy uses these hints to prefer routing traffic to endpoints in the same availability zone as the source Pod, the `service.kubernetes.io/topology-mode: auto` annotation that enables it, the fallback behavior when zone-local endpoints are unhealthy (falls back to cluster-wide routing), and the conditions under which topology hints are NOT populated (service has fewer than 3 endpoints per zone, endpoints are too unevenly distributed). For a 3-AZ EKS cluster with a 6-replica service, walk through exactly how traffic from a Pod in `us-east-1a` is routed with and without topology hints.

---

## Q255 | GitHub Actions → Dynamic Matrix | Coding

Write a GitHub Actions workflow that generates a dynamic test matrix at runtime. The workflow must: (1) have a first job `discover-services` that runs `find services/ -name 'Makefile' -maxdepth 2` to discover testable services and outputs the list as a JSON array using `core.setOutput`, (2) have a second job `test` that uses `fromJson(needs.discover-services.outputs.services)` as its matrix, running `make test` in each service directory, (3) have a third job `build` that only runs for services where the test passed AND the service directory contains changes (using `tj-actions/changed-files` output combined with the matrix), (4) limit total parallel jobs to 10 using `max-parallel`, and (5) have a final `summary` job that collects all test results using a matrix output and posts a consolidated report as a PR comment. Show the complete workflow YAML.

---

## Q256 | AWS → Bedrock & AI Infrastructure | Scenario

Your platform team must provision infrastructure for a team building an LLM application using Amazon Bedrock. The application needs: a RAG pipeline (documents stored in S3, embeddings in a vector database), Bedrock API access with usage quotas per team, and logging of all Bedrock API calls for compliance. Design the infrastructure: the IAM policy that allows `bedrock:InvokeModel` for specific foundation models only (show the exact resource ARN pattern for `anthropic.claude-3-sonnet`), the VPC endpoint for Bedrock to prevent traffic from leaving the VPC, how to use Bedrock Guardrails to prevent PII leakage in responses, the CloudWatch log group and resource policy for Bedrock model invocation logging, and the AWS OpenSearch Serverless collection (or Aurora pgvector) for embedding storage. What are the Bedrock service quota limits that a production workload will hit first?

---

## Q257 | Terraform → Provider Versioning | Failure Mode

After a routine `terraform init -upgrade`, the next `terraform plan` destroys and recreates 47 resources including production RDS instances despite no changes to `.tf` files. Describe: how provider version upgrades can change the Terraform resource schema causing perpetual diffs, the `.terraform.lock.hcl` file's role (pins exact provider versions and their checksums), why `terraform init -upgrade` ignores the lock file and updates to the latest provider within the version constraint, the specific scenario where the AWS provider changed the `aws_db_instance` schema (e.g., `parameter_group_name` becoming a list vs string), how to pin the provider version in the `required_providers` block to prevent this, and the procedure to safely upgrade providers one at a time in a test environment before applying to production.

---

## Q258 | Kubernetes → Cluster Federation | Design Tradeoff

Your organization needs to deploy the same application to 12 Kubernetes clusters across 4 cloud providers and 3 regions for multi-cloud redundancy. Compare three multi-cluster management approaches: (1) Cluster API (CAPI) for unified cluster lifecycle management — how it provisions clusters on AWS, GCP, and Azure using a declarative API, the `MachineDeployment` resource, and how you deploy workloads after provisioning, (2) Admiralty (formerly multicluster-scheduler) — how it delegates Pod scheduling across clusters using virtual nodes, the `MulticlusterService` for cross-cluster service discovery, (3) Liqo — how it extends the Kubernetes API across clusters by mapping remote namespaces. For day-2 operations (version upgrades, policy enforcement, observability), which approach has the least operational surface area?

---

## Q259 | ELK Stack → Machine Learning | Scenario

Your Elasticsearch cluster has the ML node role enabled. A security team asks you to use Elasticsearch ML to detect anomalous login patterns (spike in failed logins for a specific user, unusual login times, new geographic locations). Design the ML job: the `datafeed` configuration that queries `security-logs-*` for login events, the `single_metric` vs `multi_metric` vs `population` job type (population analysis is correct here — comparing each user's behavior to the population), the `over_field_name: username` and `by_field_name: source_ip` configuration, the `bucket_span` choice (15 minutes for near-real-time detection), how to configure a Watch to trigger when anomaly score exceeds 75, and the `forecast` API to predict future login volume. What is the minimum data history required before ML anomaly scores are reliable?

---

## Q260 | AWS → Direct Connect | Conceptual

Explain AWS Direct Connect architecture: the physical connection from your data center to an AWS Direct Connect location (colocation facility), the role of the Direct Connect Gateway for connecting to multiple VPCs across regions using a single connection, the difference between a Private VIF (Virtual Interface) for VPC access and a Public VIF for AWS public services (S3, DynamoDB) without traversing the internet, how Transit VIF with a Direct Connect Gateway + Transit Gateway enables full mesh connectivity across VPCs, the BGP session configuration (ASN, MD5 authentication, prefixes advertised), and the redundancy options (two connections to different DX locations, LAG aggregation). At what bandwidth threshold does Direct Connect become cheaper than NAT Gateway + internet data transfer for S3 access?

---

## Q261 | Kubernetes → Taints & Node Affinity Combined | Config

Write Kubernetes manifests for a complex scheduling scenario on a 100-node EKS cluster with three node groups: `general` (50 nodes, no taints), `gpu` (30 nodes, tainted `nvidia.com/gpu=true:NoSchedule`), and `spot` (20 nodes, tainted `node.kubernetes.io/spot=true:NoSchedule`). Requirements: (1) a `model-training` Job must run only on GPU nodes, tolerate the GPU taint, and use `requiredDuringSchedulingIgnoredDuringExecution` node affinity for the `gpu` node group label, (2) a `web-api` Deployment must avoid Spot nodes entirely using a node anti-affinity rule, (3) a `background-worker` Deployment should prefer Spot nodes but fall back to general nodes, (4) a `logging-agent` DaemonSet must run on ALL nodes including GPU and Spot. Show all four manifests with complete scheduling configuration.

---

## Q262 | Python Scripting → Configuration Management | Coding

Write a Python module `config_manager.py` implementing a configuration system with layered override support for a multi-environment platform. It must provide a `ConfigManager` class that: (1) loads configuration from multiple YAML files in priority order: `defaults.yaml` → `environment/<env>.yaml` → `service/<service>.yaml` → environment variables (highest priority), (2) implements deep merge (nested dicts are merged, not overwritten — show the recursive merge logic), (3) supports dot-notation access (`config.get("database.host")`) and raises a typed `ConfigKeyError` for missing required keys, (4) validates the merged config against a Pydantic schema (show an example schema with nested models), (5) supports hot-reload by watching config files with `watchdog` and re-merging on change, (6) provides a `config.diff(other_config)` method returning changed keys. Include unit tests for the merge logic edge cases.

---

## Q263 | AWS → ECS → Service Connect | Conceptual

Explain ECS Service Connect, introduced in 2022, as an alternative to AWS App Mesh for service-to-service communication within ECS clusters. Describe: how Service Connect uses the Envoy proxy as a sidecar (injected automatically, not manually configured), the `serviceConnectConfiguration` block in the ECS Service definition, how service discovery names work (a short DNS name like `payments` that resolves within the namespace), how Service Connect provides built-in retries, timeouts, and circuit breaking without writing application code, the difference between Service Connect and Service Discovery (Cloud Map), and the observability integration (CloudWatch Container Insights automatically shows service-to-service latency and error rates). For a team already using App Mesh, what is the migration path and what App Mesh features have no Service Connect equivalent?

---

## Q264 | CI/CD → Deployment Verification | Scenario

Your deployments complete successfully (Helm returns exit 0, all Pods are Running) but 15% of deployments silently introduce regressions that are only caught by users 30 minutes later. Design a deployment verification gate that runs automatically after every deployment and blocks promotion if it detects a regression: (1) smoke tests using `curl` for critical API endpoints with response schema validation (not just HTTP 200), (2) comparison of key Prometheus metrics between the old and new version using the `promtool` query API or custom script — specifically error rate, p99 latency, and throughput — with automatic rollback if any metric degrades by more than 10%, (3) synthetic transaction tests using Playwright that simulate a complete user journey (login → add to cart → checkout), and (4) log anomaly detection comparing error log rate before and after deployment using a 5-minute baseline window. Show the complete verification script structure.

---

## Q265 | Prometheus & Grafana → Dashboards as Code | Config

Write a Grafonnet (Jsonnet) or Terraform `grafana_dashboard` resource definition for a Kubernetes workload dashboard. The dashboard must have: (1) a template variable for `namespace` populated from `label_values(kube_pod_info, namespace)`, (2) a row for "Traffic" with a time-series panel for RPS using `sum(rate(http_requests_total{namespace="$namespace"}[5m])) by (pod)`, (3) a row for "Errors" with a stat panel showing current error rate percentage and a gauge panel showing error budget remaining (based on a 99.9% SLO target), (4) a row for "Saturation" with CPU and memory usage panels that use `requests` as the denominator for utilization percentage, (5) all panels using the `$__rate_interval` variable for rate windows, and (6) an annotation layer showing Kubernetes deployment events. Output the complete JSON or Jsonnet.

---

## Q266 | Linux & Bash → Security Hardening | Coding

Write a Bash script `audit_ssh_config.sh` that audits SSH server and client configurations across a fleet of hosts. The script must: (1) accept a file containing a list of hostnames as input, (2) SSH to each host and check `/etc/ssh/sshd_config` for: `PermitRootLogin no`, `PasswordAuthentication no`, `MaxAuthTries` ≤ 3, `Protocol 2`, `AllowTcpForwarding no` (unless explicitly required), `ClientAliveInterval` ≤ 300, (3) check for authorized_keys files in non-standard locations using `find / -name authorized_keys 2>/dev/null`, (4) verify running `sshd` process matches the installed binary (detects binary replacement), (5) output a per-host compliance report in CSV format: `hostname,check_name,status,current_value,expected_value`, (6) run all host checks in parallel with a max of 20 concurrent SSH connections, and (7) exit with code 1 if any critical check fails. Use SSH multiplexing (`ControlMaster`) to reduce connection overhead.

---

## Q267 | ArgoCD → Image Updater | Scenario

Your team wants to automate image tag updates in ArgoCD when a new Docker image is pushed to ECR — currently developers must manually update the image tag in Git after every CI build. Implement ArgoCD Image Updater: the `argocd-image-updater` tool's annotation-based configuration on the ArgoCD Application resource, the `argocd-image-updater.argoproj.io/image-list` annotation specifying the ECR image and update strategy (`semver`, `latest`, `digest`, or `name`), how Image Updater writes the new tag back to Git (the `git write-back` method vs the `argocd` method), the IRSA configuration needed for Image Updater to authenticate to ECR (the `ecr:DescribeImages` and `ecr:GetAuthorizationToken` permissions), and the race condition between Image Updater writing to Git and ArgoCD detecting the change and syncing. What is the recommended workflow for a production environment where arbitrary `latest` tag updates should not auto-deploy?

---

## Q268 | AWS → CodePipeline vs GitHub Actions | Design Tradeoff

Your organization's security team mandates that production deployment pipelines must: run entirely within the AWS account boundary (no SaaS CI/CD), use IAM roles (not stored credentials), have a complete audit trail in CloudTrail, and integrate natively with AWS CodeArtifact for artifact storage. Compare CodePipeline + CodeBuild vs self-hosted GitHub Actions runners on EC2: how CodePipeline's source stage polls CodeCommit or S3 (no webhook latency, but 1-minute polling delay), CodeBuild's IAM execution role (no stored credentials) vs GitHub Actions OIDC, CodePipeline's native integration with CodeDeploy for ECS blue/green deployments, and the specific CodePipeline limitation (no DAG-style parallel stages in V1 — V2 added parallel stages in 2023). For a team already on GitHub for source control but needing AWS-native deployments, what is the hybrid architecture?

---

## Q269 | Kubernetes → Resource Quotas | Scenario

A namespace `data-science` has a ResourceQuota limiting `requests.cpu: 100` cores and `requests.memory: 200Gi`. A data scientist reports they cannot launch a new Jupyter notebook Pod even though `kubectl describe resourcequota` shows 40 cores and 80Gi available. Walk through the complete diagnosis: checking if a LimitRange is applying default requests/limits that exceed what the Pod spec specifies, verifying the Pod's actual computed requests including init containers (init container requests count toward quota during their execution phase), checking if `requests.pods` or `count/pods` quota is exhausted, verifying the `PodSecurityAdmission` is not rejecting the Pod before the quota check even runs, and examining whether the namespace has a `requests.storage` quota blocking a PVC that the notebook needs. What `kubectl` command gives you the full admission rejection reason in one step?

---

## Q270 | Python Scripting → gRPC & Protobuf | Coding

Write a Python gRPC client `k8s_metrics_client.py` that connects to a custom metrics gRPC server (assume a proto definition is given) and: (1) establishes a gRPC channel with mTLS (client certificate, server CA cert — paths from environment variables), (2) implements a `stream_metrics(namespace: str, pod_selector: dict)` method that opens a server-side streaming RPC and yields `MetricEvent` objects as they arrive, (3) handles stream reconnection with exponential backoff (max 5 retries, starting at 1 second) when the server disconnects, (4) implements a `get_aggregated_metrics(namespace: str, window_seconds: int) -> AggregatedMetrics` unary RPC call with a 10-second deadline, (5) logs all RPC calls with method name, status code, and duration using a `grpc.UnaryClientInterceptor`, and (6) provides an async version using `grpc.aio`. Include the proto snippet showing the service definition this client implements.

---

## Q271 | AWS → Network Firewall | Design Tradeoff

Your security team requires deep packet inspection and domain-based URL filtering for all egress traffic from EKS workloads. Compare three approaches: (1) AWS Network Firewall — a stateful Suricata-based firewall deployed in a dedicated firewall subnet with a Gateway Load Balancer endpoint, how traffic is routed through it using VPC route tables (traffic goes: Pod → NAT GW subnet route → GWLB endpoint → Network Firewall → internet), the rule groups (stateless 5-tuple rules vs stateful domain-list rules), and the cost ($0.395/hour + $0.065/GB), (2) Squid proxy as a DaemonSet with `HTTPS_PROXY` environment variable injection via a mutating webhook, (3) Cilium's DNS-based network policies that use eBPF to filter by FQDN without a proxy. For a PCI-DSS compliant environment, which approach satisfies the requirement for logging all outbound connections with domain names?

---

## Q272 | Kubernetes → Backup & Restore | Scenario

Your production EKS cluster must be backed up for disaster recovery. Design a complete backup strategy using Velero: the Velero installation on EKS with IRSA for S3 access, the `Schedule` CRD for daily full cluster backups (all namespaces) and hourly backups of `payments` and `orders` namespaces, how Velero backs up PersistentVolumes (using CSI volume snapshots via the `velero.io/csi-volumesnapshot-class` annotation vs Velero's legacy Restic/Kopia file-level backup), the `--include-cluster-resources` flag for backing up CRDs and ClusterRoles, testing the backup with a restore to a separate namespace, and the specific Velero bug where restoring a namespace that already exists partially fails. What is your RTO/RPO for a complete cluster restore, and what cannot be backed up by Velero (etcd contents, Kubernetes secrets encrypted at rest keys)?

---

## Q273 | CI/CD → Monorepo CI Optimization | Design Tradeoff

Your monorepo containing 25 services runs full CI for all services on every commit, taking 90 minutes. Only 1–3 services change per commit on average. Design a complete affected-service detection and selective CI system: (1) using `git diff --name-only HEAD~1` combined with a service ownership map (`CODEOWNERS` equivalent) to determine which services are affected, (2) handling transitive dependencies (service B depends on shared library L — a change to L must trigger CI for B even though B's files didn't change), representing dependencies as a DAG in `deps.json`, (3) the GitHub Actions or Jenkins implementation that turns affected services into a dynamic matrix, (4) the cache warming strategy so that unaffected services' test dependencies are still cached for the next run, and (5) the escape hatch for platform-level changes (`changes to Dockerfile.base` → rebuild all services). Show the dependency resolution algorithm in pseudocode.

---

## Q274 | Prometheus & Grafana → Runbook Automation | Scenario

You want to reduce MTTR for common alerts by embedding runbook automation directly into the alerting workflow. Design a system where Prometheus alerts automatically trigger diagnostic scripts and attach the output to the Slack notification: (1) Alertmanager webhook receiver that POSTs alert payloads to a custom Python Flask service, (2) the Flask service maps `alertname` to a diagnostic script (e.g., `HighMemoryUsage` → run `kubectl top pods --namespace=$namespace` and `kubectl describe node $node`), (3) the script runs asynchronously (job queue using `celery` + Redis) and posts the output back to the Slack thread of the original alert notification, (4) a `max_runtime: 30s` timeout per diagnostic script with output truncation to 3000 characters for Slack's message limit, and (5) a feedback button in Slack (`Did this help? Yes/No`) that feeds back into a simple accuracy tracker. Show the Flask route and Celery task structure.

---

## Q275 | ELK Stack → Rollover & Aliases | Config

Write the complete Elasticsearch setup for an index rollover pattern serving a write-heavy logging use case. The setup must include: (1) an index template `logs-app` that applies to `logs-app-*` indices with mappings, settings (`number_of_shards: 3`, `codec: best_compression`), and an alias configuration, (2) the initial index `logs-app-000001` created with the write alias `logs-app-write` and read alias `logs-app-read`, (3) a rollover API call (`POST /logs-app-write/_rollover`) with conditions: max age 1 day, max docs 50 million, max primary shard size 50GB, (4) an ILM policy that automates the rollover without manual API calls, (5) a search request that uses the read alias to query across all rolled-over indices transparently, and (6) the re-index procedure if you need to change a mapping on existing data. Explain why you need both a write alias and a read alias.

---

## Q276 | AWS → GuardDuty Automation | Coding

Write a Python Lambda function `guardduty_auto_remediate.py` triggered by EventBridge when GuardDuty findings are published to an SNS topic. The function must: (1) parse the GuardDuty finding JSON from the SNS message, (2) for finding type `UnauthorizedAccess:EC2/SSHBruteForce` — automatically add the source IP to a WAF IP set (using `wafv2:UpdateIPSet`), (3) for `InstanceCredentialExfiltration:EC2/NoInstanceProfile` — attach an explicit Deny IAM policy to the role used by the instance, (4) for `Backdoor:EC2/C&CActivity.B` — create a VPC security group rule blocking all outbound traffic from the instance and send a PagerDuty critical alert, (5) for all findings — write a structured record to a DynamoDB table with finding ID, type, resource ARN, remediation action taken, and timestamp, and (6) never take automated action on `severity < 7` findings — log them only. Include IAM policy JSON for the Lambda execution role.

---

## Q277 | Kubernetes → Multi-Container Patterns | Design Tradeoff

Explain three Kubernetes multi-container Pod patterns and give a production use case for each: (1) Sidecar — a log shipping container that reads from a shared `emptyDir` volume written by the main app, the lifecycle coupling risk (Pod dies if sidecar crashes), and how Kubernetes 1.29 native sidecars with `restartPolicy: Always` in `initContainers` fix the coupling problem, (2) Ambassador — a proxy container that translates the application's simple outbound connections into authenticated, TLS-wrapped calls to external services (e.g., a Vault agent that intercepts database credential requests), (3) Adapter — a container that transforms the main app's non-standard metrics format into Prometheus format (e.g., a StatsD-to-Prometheus bridge). For each pattern, explain: when a sidecar becomes a liability (adds resource overhead to every Pod even when not needed), and when you would use a DaemonSet instead.

---

## Q278 | AWS → Athena & Data Lake | Scenario

Your team stores 10TB of compressed JSON CloudTrail logs in S3 partitioned by `year/month/day/hour`. Security auditors need to query 90 days of logs in under 60 seconds. Currently raw JSON queries take 25 minutes and cost $12 per query. Design the complete optimization: (1) using AWS Glue Crawler to create a Data Catalog table with partition projection for the date partitions (show the partition projection configuration), (2) converting JSON to Parquet using a Glue ETL job or Apache Spark on EMR — explain the column pruning and predicate pushdown benefits, (3) using Athena workgroup result caching (5-minute TTL) to avoid re-running identical queries, (4) Athena partition pruning by always including `year`, `month`, `day` in WHERE clauses, (5) the specific cost calculation: 10TB scan at $5/TB vs Parquet columnar with 10× compression (approximately $0.50/TB equivalent). What is the expected query time after optimization?

---

## Q279 | GitHub Actions → Workflow Triggers | Design Tradeoff

You have a CI workflow that must run in three different modes: (1) on every PR — run tests only, no deployment, (2) on merge to main — run tests and deploy to staging, (3) manually triggered with an `environment` input — deploy to any specified environment. Currently three separate workflow files duplicate 80% of the same content. Redesign using a single workflow file with `workflow_call` for reusability: the `on:` trigger block using `pull_request`, `push`, and `workflow_dispatch` with an `inputs:` block, `if:` conditionals on jobs to implement the three modes, how `github.event_name` and `github.ref` are used in conditions, the specific issue with `workflow_dispatch` not having `github.event.pull_request` context, and how you prevent the deploy job from running when a PR is opened from a fork (since forks don't have access to environment secrets). Show the complete workflow YAML.

---

## Q280 | Linux & Bash → Memory Management | Conceptual

Explain Linux memory management concepts relevant to running containerized workloads: the difference between `Resident Set Size (RSS)`, `Virtual Memory Size (VSZ)`, `Proportional Set Size (PSS)`, and `Unique Set Size (USS)` and which metric accurately reflects a container's true memory consumption for billing/quota purposes, how the kernel's page cache interacts with container memory limits (anonymous memory vs file-backed pages), why `cgroups v2` memory accounting is more accurate than v1, the `memory.memsw.limit_in_bytes` setting that limits swap usage per cgroup, and how a Java application with `-Xmx4g` can be OOM-killed by the kernel even with a container limit of 8g (native memory regions: metaspace, code cache, thread stacks, direct byte buffers). What is the rule of thumb for setting a Java container's memory limit relative to `-Xmx`?

---

## Q281 | ArgoCD → Rollout Strategies | Config

Write the complete Argo Rollouts `Rollout` resource (replacing a standard Deployment) for a `checkout-service` that implements a blue/green deployment with the following requirements: (1) use the `blueGreen` strategy with `autoPromotionEnabled: false` (manual promotion), (2) configure `prePromotionAnalysis` using an `AnalysisTemplate` that runs for 5 minutes checking the error rate via PromQL is below 1%, (3) configure `postPromotionAnalysis` that runs for 10 minutes after traffic switch, (4) set `scaleDownDelaySeconds: 300` to keep the old (blue) version running for 5 minutes after promotion for quick rollback, (5) integrate with the AWS Load Balancer Controller using `activeService` and `previewService` pointing to two separate Kubernetes Services. Show the complete Rollout YAML, the AnalysisTemplate YAML, and the two Service manifests.

---

## Q282 | AWS → Cost Allocation | Scenario

Your AWS bill shows $200k/month but the finance team cannot attribute costs to the 15 product teams sharing the account. Design a complete cost allocation system: (1) tagging strategy — mandatory tags (`team`, `environment`, `service`, `cost_center`) enforced via SCP, how to tag resources created by EKS (the `cluster-autoscaler/autodiscover` and AWS Load Balancer Controller automatically tag EC2 and ELB resources — show the IAM tag condition), (2) AWS Cost Categories — creating a rule-based cost category that groups untagged costs into a `Shared Infrastructure` bucket, (3) split-charge rules for shared services (NAT Gateway, EKS control plane cost — $0.10/hour — split proportionally by node count), (4) a monthly cost report using Cost Explorer API grouped by `team` tag, exported to S3 as CSV and auto-emailed via SES, (5) chargeback vs showback model tradeoffs for internal teams. Show the Cost Categories rule JSON.

---

## Q283 | Kubernetes → Webhook Admission | Coding

Write a Python Flask application that implements a Kubernetes Mutating Admission Webhook. The webhook must: (1) listen on HTTPS (TLS cert/key paths from environment variables), (2) receive `AdmissionReview` v1 JSON POST requests, (3) for every Pod admission request, inject the following if not already present: an environment variable `POD_NAMESPACE` from `fieldRef: metadata.namespace`, a `checksum/config` annotation with the SHA256 of the Pod's ConfigMap names (to trigger rolling restarts on ConfigMap changes), and a resource limit of `cpu: 100m` / `memory: 128Mi` if no limits are set, (4) return a valid `AdmissionReview` response with a base64-encoded JSON Patch, (5) return `allowed: true` even if the mutation fails (never block Pod creation), and (6) expose `/healthz` for liveness probes. Show the complete Flask application and the `MutatingWebhookConfiguration` manifest.

---

## Q284 | AWS → EventBridge Pipes | Conceptual

Explain AWS EventBridge Pipes, introduced in 2022, as a managed point-to-point integration service. Describe: how a Pipe connects a source (SQS, DynamoDB Streams, Kinesis, Kafka) to a target (Lambda, Step Functions, EventBridge bus, API destination) with optional filtering and enrichment in between, the filtering step (how you write filter patterns to drop events before they reach the enrichment Lambda, reducing cost), the enrichment step (a Lambda or API Gateway that transforms the event payload inline), how Pipes handles batching and partial failures from SQS sources (the `batchItemFailures` response), and how EventBridge Pipes compare to a hand-rolled Lambda that polls SQS and posts to another service (Pipes manages polling, batching, retry, and DLQ automatically). For a DynamoDB Streams → Elasticsearch sync pattern, show the Pipe configuration and the filter pattern that forwards only `INSERT` and `MODIFY` events.

---

## Q285 | Linux & Bash → Observability | Coding

Write a Bash script `system_snapshot.sh` that captures a comprehensive system state snapshot for post-incident analysis. When invoked with `--incident-id <id>`, it must capture and save to `/var/incident-snapshots/<id>/`: (1) `ps-full.txt` — full process list with CPU, memory, open files count per process, (2) `netstat.txt` — full socket table with state distribution summary, (3) `disk-io.txt` — 30-second `iostat -x 1 30` capture, (4) `memory.txt` — `/proc/meminfo`, `free -m`, and top 20 processes by RSS, (5) `kernel-log.txt` — last 500 lines of `dmesg` with timestamps, (6) `cgroup-usage.txt` — for every cgroup in `/sys/fs/cgroup/memory/`, capture `memory.usage_in_bytes` and `memory.limit_in_bytes`, (7) `iptables-rules.txt` — all iptables chains and rules, (8) a `summary.json` with key metrics (load average, memory pressure %, top CPU process). All captures must complete within 90 seconds total. Archive the directory as a `.tar.gz` at the end.

---

## Q286 | Kubernetes → Observability → OpenTelemetry | Design Tradeoff

Design a complete OpenTelemetry observability pipeline for a 50-service Kubernetes platform. Address: (1) the OpenTelemetry Operator deployment — how it injects the OTel Collector as a sidecar or DaemonSet using the `Instrumentation` CRD and auto-instrumentation for Python/Java/Node.js without code changes, (2) the Collector pipeline configuration: receivers (`otlp`, `k8sattributes` for enriching spans with Pod metadata), processors (`batch`, `memory_limiter`, `resourcedetection`), and exporters (Tempo for traces, Mimir/Prometheus for metrics, Loki for logs), (3) the `k8sattributes` processor's role in adding `k8s.namespace.name`, `k8s.pod.name`, and `k8s.deployment.name` to every span, (4) sampling strategy (tail-based vs head-based sampling — when to use each), and (5) the cost implication of 100% trace sampling vs 1% tail-based sampling at 10,000 RPS. Show the OTel Collector pipeline YAML.

---

## Q287 | AWS → Secrets Manager Rotation | Coding

Write the complete AWS Lambda function `rotate_rds_password.py` that implements the Secrets Manager rotation contract for an RDS PostgreSQL database using the multi-user rotation strategy (creates a new user, validates, then deletes the old user — zero downtime). The function must implement all four rotation steps that Secrets Manager calls: `createSecret` (generate a new password, store as `AWSPENDING` in the secret), `setSecret` (create a new PostgreSQL user with the pending credentials using `psycopg2`), `testSecret` (connect to RDS using `AWSPENDING` credentials and run a test query), `finishSecret` (move `AWSPENDING` to `AWSCURRENT`, delete the old PostgreSQL user). The function must read the RDS endpoint from the secret JSON, handle the VPC-internal network connection (no public endpoint), and implement proper rollback if `setSecret` fails. Include the Lambda execution role IAM policy and the rotation schedule configuration.

---

## Q288 | CI/CD → Security Scanning | Design Tradeoff

Design a complete supply chain security pipeline for container images and Terraform code. The pipeline must enforce: (1) SAST for Python application code using `bandit` — integrate into pre-commit hooks and CI, show the `.bandit` config that suppresses false positives for specific rules, (2) SCA (Software Composition Analysis) for Python dependencies using `pip-audit` checking against the OSV database, (3) container image scanning using `trivy` — the difference between `trivy image` (pulls from registry) vs `trivy fs` (scans local filesystem) vs `trivy sbom` (scans SBOM), configuring severity thresholds (`--severity CRITICAL,HIGH`) and exit codes, (4) SBOM generation using `syft` in CycloneDX format and attestation using `cosign attest`, (5) Terraform IaC scanning using `checkov` with a custom check for your organization's tagging policy. For each tool, show the exact CI step YAML.

---

## Q289 | Prometheus & Grafana → Capacity Planning | Scenario

You need to capacity plan for Prometheus storage and compute growth over the next 12 months. Your cluster currently has 1 million active time series, scrapes every 15 seconds, and retains data for 30 days. Calculate: (1) the storage requirement using the formula `active_series × bytes_per_sample × samples_per_second × retention_seconds` (assume 1.5 bytes/sample with compression), (2) the memory requirement (Prometheus needs approximately 3KB of RAM per active time series for the head block), (3) the CPU requirement (Prometheus rule evaluation and query processing), (4) the growth projection assuming 20% monthly series growth with the same retention, (5) at what series count you must migrate to Thanos or Mimir (typically 5–10 million series for a single Prometheus), and (6) the `--storage.tsdb.retention.size` flag as a safety backstop. Show the math and produce a 12-month forecast table.

---

## Q290 | AWS → Lambda → Extensions | Conceptual

Explain the AWS Lambda Extensions API and how it enables external monitoring, security, and telemetry tools to run alongside Lambda functions. Describe: how an extension registers with the Lambda service using the `/extension/register` API, the extension lifecycle phases (`Init`, `Invoke`, `Shutdown`) and how they map to the Lambda lifecycle, how an internal extension (runs in the same process as the function) differs from an external extension (runs as a separate process), how the Datadog Lambda extension uses this to send metrics without requiring a network call from the function code itself, and the performance implication of extensions (they extend the Lambda Init phase and can add to cold start time). Why does a Lambda extension with `INVOKE` subscription receive `Next` events even during warm invocations, and what is the correct pattern for flushing telemetry at shutdown vs after every invocation?

---

## Q291 | Kubernetes → Service Accounts Automation | Coding

Write a Kubernetes controller in Python (`sa_token_rotator.py`) that automatically rotates short-lived service account tokens for applications that cannot use projected service account tokens. The controller must: (1) watch for `ServiceAccount` objects with the annotation `token-rotator.io/enabled: "true"` and `token-rotator.io/rotation-interval-hours: "4"`, (2) create a `TokenRequest` (using `apis/authentication.k8s.io/v1`) with a 4-hour expiry for the ServiceAccount, (3) write the token to a `Secret` in the same namespace named `<sa-name>-rotated-token`, (4) schedule re-rotation 30 minutes before expiry using `asyncio` — not a CronJob, (5) handle the case where the Secret already exists (update the token data), and (6) expose a Prometheus metric `token_rotations_total{namespace, service_account, status}`. Include the complete RBAC manifests and the Deployment spec for the controller.

---

## Q292 | AWS → CodeArtifact | Scenario

Your organization wants to proxy and cache PyPI, npm, and Maven packages through AWS CodeArtifact to: prevent dependency confusion attacks, ensure packages are available even if upstream registries go down, and scan packages for vulnerabilities before developers can use them. Design the complete setup: the CodeArtifact domain and repository hierarchy (upstream connections to PyPI, npmjs, Maven Central), the `pip` and `npm` configuration to use CodeArtifact as the registry (using `aws codeartifact get-authorization-token` for short-lived tokens), how to configure the package origin control to block direct publishing of packages with the same name as public packages (preventing dependency confusion), integrating `inspector2` for package scanning before packages are cached, and the IAM policy for a developer role that allows read-only access to CodeArtifact but not publish. Show the `pip.conf` and `.npmrc` configuration.

---

## Q293 | Kubernetes → Debugging with Ephemeral Containers | Scenario

A production Pod running a distroless container image has no shell, no `ps`, no `curl`, and no debugging tools. The Pod is exhibiting a memory leak but you cannot reproduce it locally. Describe the complete live debugging procedure using ephemeral containers: the `kubectl debug` command to inject a debugging container sharing the target container's process namespace (`--target` flag), how `shareProcessNamespace: true` allows the debug container to see the target's processes with `ps aux`, using `nsenter -t <pid> -m -u -i -n -p` to enter the target container's namespaces from the debug container, running `gdb`, `strace`, or `py-spy` against the target process from the debug container, and the security implication of ephemeral containers (they bypass normal admission webhooks). What are the limitations of ephemeral containers compared to `kubectl exec` in a non-distroless container?

---

## Q294 | Linux & Bash → Cgroups v2 | Conceptual

Explain the key architectural differences between cgroups v1 and cgroups v2 and their implications for containerized workloads. Describe: how v1 uses separate hierarchies per subsystem (memory, cpu, blkio each have their own tree) vs v2's single unified hierarchy, why v2's `io` controller is more accurate than v1's `blkio` (v2 accounts for both buffered writes and direct I/O), the v2 `memory.oom.group` setting that kills the entire cgroup (Pod) when any process is OOM-killed rather than just the offending process, how `systemd` on modern Linux hosts uses v2 exclusively and what this means for Docker and Kubernetes (they must use the `cgroupfs` cgroup driver or `systemd` cgroup driver consistently), and how `containerd`'s `SystemdCgroup = true` in `/etc/containerd/config.toml` must match the kubelet's `--cgroup-driver` flag. What breaks when they are mismatched?

---

## Q295 | GitHub Actions → OpenID Connect Deep Dive | Coding

Write a Python script `validate_gha_oidc_token.py` that validates a GitHub Actions OIDC JWT token for use in a custom internal token exchange service (not AWS). The script must: (1) fetch the JWKS from `https://token.actions.githubusercontent.com/.well-known/jwks` with caching (re-fetch only if the cached version is older than 1 hour), (2) decode the JWT header to get the `kid` (key ID), (3) find the matching public key in the JWKS and construct an RSA public key object using `cryptography` library, (4) verify the JWT signature, expiry, and the `iss` claim equals `https://token.actions.githubusercontent.com`, (5) extract and validate the `sub` claim against an allowlist of `repo:org/repo:ref:refs/heads/main` patterns using regex, (6) return a typed `GHAIdentity` dataclass with `repository`, `workflow`, `ref`, `sha`, and `environment` fields. Include error handling for expired tokens, unknown `kid`, and invalid signatures.

---

## Q296 | AWS → CloudWatch → Contributor Insights | Scenario

Your API Gateway is experiencing intermittent throttling (HTTP 429) for specific API keys and IP addresses, but the default CloudWatch metrics only show aggregate throttle counts without attribution. Configure CloudWatch Contributor Insights to identify the top contributors: write the Contributor Insights rule JSON that analyzes API Gateway access logs to surface the top 10 API keys by request count and the top 10 source IPs by 429 response count, explain how Contributor Insights rules differ from Logs Insights queries (always-on rules vs on-demand queries), the `SPLIT` function for parsing log fields, the `FILTER` for isolating 429 responses, and how to create a CloudWatch Dashboard widget showing the top contributors as a leaderboard. Show the complete rule JSON and the approximate cost at 1 million log events per day.

---

## Q297 | Kubernetes → Finalizers | Failure Mode

A Namespace `old-project` is stuck in `Terminating` state. `kubectl delete namespace old-project` was run 2 hours ago. `kubectl get namespace old-project -o yaml` shows the namespace has a `DeletionTimestamp` but also has finalizers. Describe the complete diagnosis and recovery: how to identify which resources are blocking namespace deletion using `kubectl api-resources --verbs=list --namespaced -o name` combined with `kubectl get <resource> -n old-project` for each, why a custom CRD resource with a finalizer blocks deletion even after the Operator managing it is deleted (the controller that would process the finalizer is gone), how to manually remove a finalizer from a stuck resource using `kubectl patch` with a JSON merge patch removing the `finalizers` array entry, the risk of force-removing finalizers (the controller may never clean up the underlying infrastructure), and how to prevent this in your own Operators by implementing a reconcile loop that removes finalizers before they become blocking.

---

## Q298 | Python Scripting → Infrastructure Testing | Coding

Write a Python test suite `test_infrastructure.py` using `pytest` that validates live AWS infrastructure (not mocked) as part of a post-deployment verification step. The tests must: (1) verify an ALB returns HTTP 200 with a valid JSON response body from `/health` within 2 seconds, (2) verify an RDS cluster endpoint is reachable on port 5432 and accepts a connection with the correct SSL mode using `psycopg2`, (3) verify an EKS cluster has all nodes in `Ready` state using the Kubernetes Python client, (4) verify all Deployments in the `production` namespace have `availableReplicas == replicas`, (5) verify an SQS queue's `ApproximateNumberOfMessages` is below a threshold (no stuck messages from previous deployment), (6) verify Route53 DNS resolves the application domain to an IP within the expected CIDR range. All tests must be idempotent (read-only, no side effects), run in parallel using `pytest-xdist`, and complete in under 60 seconds.

---

## Q299 | CI/CD → Platform Engineering | Design Tradeoff

You are building a golden-path CI/CD template that 200 development teams will adopt. The template must be opinionated enough to provide value but flexible enough for teams to customize. Design the template architecture: (1) the layered customization model — platform-owned steps (security scanning, artifact signing, deployment verification) that cannot be overridden vs team-owned steps (build commands, test commands) that are fully configurable, (2) how you distribute the template (a GitHub template repository, a reusable workflow called with `uses:`, or a custom GitHub Action) and the versioning strategy, (3) the onboarding experience — a `platform init` CLI command that generates a minimal workflow file for a new service by asking 5 questions (language, deployment target, environment count), (4) the feedback loop — how you know when teams fork the template and diverge, using GitHub's dependency graph or a custom scanner, and (5) the migration strategy when you need to release a breaking change to the template. What is the one feature that drives template adoption faster than any other?

---

## Q300 | Platform Engineering → Toil Reduction | Scenario

Over the past quarter your platform team spent 60% of their time on repetitive operational tasks: manually creating namespaces and RBAC for new teams (3 hours each), rotating secrets on request (1 hour each), approving and manually executing Terraform changes for new S3 buckets and SQS queues (2 hours each), and responding to `how do I deploy X` questions in Slack. Your team receives 15 of these requests per week. Design the complete self-service automation system that reduces this toil to near zero: (1) a Backstage-based service catalog with a `Create New Team` template that triggers a GitHub Actions workflow to create the namespace, RBAC, ArgoCD Project, and Slack channel automatically, (2) an External Secrets Operator self-service model where teams create `ExternalSecret` resources in their namespace without platform involvement, (3) a Crossplane-based `Composition` that lets teams create an `AppInfrastructure` CRD and get an S3 bucket + SQS queue + IAM role provisioned automatically with guardrails, (4) a developer portal FAQ bot using Bedrock + RAG over your Confluence documentation, and (5) the metrics you use to prove toil reduction to leadership (time-to-first-deployment for new teams, tickets per engineer per week). What toil cannot be automated and how do you make peace with it?

---
