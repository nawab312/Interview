# DevOps / SRE / Cloud Interview Questions (Q136–Q180)

> Each question includes: Key Points to Cover + Interview Tip
> Format mirrors Q64 reference style.

---

## Table of Contents

| # | Topic | Type | Difficulty |
|---|---|---|---|
| 136 | Kubernetes | Conceptual | Advanced |
| 137 | AWS | Scenario-Based | Advanced |
| 138 | Linux / Bash | Conceptual | Advanced |
| 139 | Terraform | Conceptual | Advanced |
| 140 | Prometheus | Scenario-Based | Advanced |
| 141 | Kubernetes | Troubleshooting | Advanced |
| 142 | Git | Conceptual | Advanced |
| 143 | AWS | Conceptual | Advanced |
| 144 | Jenkins | Troubleshooting | Advanced |
| 145 | ELK Stack | Scenario-Based | Advanced |
| 146 | Kubernetes | Scenario-Based | Advanced |
| 147 | Linux / Bash | Troubleshooting | Advanced |
| 148 | Terraform | Scenario-Based | Advanced |
| 149 | GitHub Actions | Conceptual | Advanced |
| 150 | AWS | Troubleshooting | Advanced |
| 151 | Prometheus | Conceptual | Advanced |
| 152 | Kubernetes | Conceptual | Advanced |
| 153 | ArgoCD | Conceptual | Advanced |
| 154 | Linux / Bash | Scenario-Based | Advanced |
| 155 | AWS | Scenario-Based | Advanced |
| 156 | Kubernetes | Troubleshooting | Advanced |
| 157 | Grafana | Scenario-Based | Advanced |
| 158 | Terraform | Troubleshooting | Advanced |
| 159 | Python | Scenario-Based | Advanced |
| 160 | AWS | Conceptual | Advanced |
| 161 | Kubernetes | Scenario-Based | Advanced |
| 162 | ELK Stack | Conceptual | Advanced |
| 163 | Jenkins | Scenario-Based | Advanced |
| 164 | Linux / Bash | Conceptual | Advanced |
| 165 | AWS | Scenario-Based | Advanced |
| 166 | Kubernetes | Conceptual | Advanced |
| 167 | GitHub Actions | Scenario-Based | Advanced |
| 168 | Terraform | Conceptual | Advanced |
| 169 | Prometheus | Troubleshooting | Advanced |
| 170 | AWS | Conceptual | Advanced |
| 171 | Kubernetes | Scenario-Based | Advanced |
| 172 | ArgoCD | Scenario-Based | Advanced |
| 173 | Linux / Bash | Scenario-Based | Advanced |
| 174 | AWS | Scenario-Based | Advanced |
| 175 | Kubernetes | Conceptual | Advanced |
| 176 | Grafana | Conceptual | Advanced |
| 177 | Terraform | Scenario-Based | Advanced |
| 178 | Git | Scenario-Based | Advanced |
| 179 | AWS + Kubernetes | Scenario-Based | Advanced |
| 180 | All Topics | System Design | Advanced |

---

### Q136 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `NetworkPolicy`** in detail. What is the difference between ingress and egress policies? Write a NetworkPolicy that denies all by default, allows ingress only from `role=frontend`, and allows egress only to port 5432 and DNS.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md`, `07_SECURITY.md`

```
Default behavior (no NetworkPolicy):
  → All pods can talk to all other pods — flat unrestricted network
  → No NetworkPolicy = fully open (permissive default)
  → Once ANY NetworkPolicy selects a pod, only explicitly allowed
    traffic is permitted for that pod

NetworkPolicy applies to selected pods only:
  podSelector: {}      → selects ALL pods in namespace
  podSelector: {matchLabels: {app: db}} → selects only db pods

Ingress policy:
  → Controls incoming traffic TO the selected pods
  → from: defines allowed sources (pods, namespaces, IP blocks)

Egress policy:
  → Controls outgoing traffic FROM the selected pods
  → to: defines allowed destinations

Step 1 — Default deny all (must apply first):
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: default-deny-all
    namespace: production
  spec:
    podSelector: {}       # select ALL pods
    policyTypes:
    - Ingress
    - Egress
    # no rules = deny everything

Step 2 — Allow ingress from frontend pods only:
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-frontend-ingress
    namespace: production
  spec:
    podSelector:
      matchLabels:
        app: backend
    policyTypes:
    - Ingress
    ingress:
    - from:
      - podSelector:
          matchLabels:
            role: frontend     # AND condition with namespaceSelector
      ports:
      - port: 8080
        protocol: TCP

Step 3 — Allow egress to PostgreSQL + DNS:
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-db-dns-egress
    namespace: production
  spec:
    podSelector:
      matchLabels:
        app: backend
    policyTypes:
    - Egress
    egress:
    - ports:
      - port: 5432
        protocol: TCP        # PostgreSQL
    - ports:
      - port: 53
        protocol: UDP        # DNS (MUST allow or all DNS breaks)
      - port: 53
        protocol: TCP        # DNS over TCP (large responses)

AND vs OR — most common mistake:
  from:
  - namespaceSelector:     # SAME item = AND
      matchLabels:
        env: prod
    podSelector:           # no dash = same item
      matchLabels:
        role: frontend
  → Only frontend pods IN prod namespace

  from:
  - namespaceSelector:     # SEPARATE items = OR
      matchLabels:
        env: prod
  - podSelector:           # separate dash = different item
      matchLabels:
        role: frontend
  → ALL pods in prod namespace OR frontend pods in ANY namespace

CNI requirement:
  NetworkPolicy only works if CNI supports it:
  ✓ Calico, Cilium, Weave Net
  ✗ Flannel (objects exist but do nothing)
  EKS: VPC CNI alone does NOT enforce NetworkPolicy
       Need: Network Policy Controller add-on or Calico
```

💡 **Interview tip:** The AND vs OR semantics of `from` selectors is the most frequently asked NetworkPolicy gotcha — namespaceSelector and podSelector in the same list item are AND; in separate list items they are OR. The second most important point: always allow UDP/TCP port 53 for DNS before applying a default-deny egress policy — forgetting this breaks all DNS resolution immediately and is a very common production mistake. Also mention that NetworkPolicy requires a CNI that supports it — on EKS with default VPC CNI, applying NetworkPolicy objects does nothing without the Network Policy Controller add-on.

---

### Q137 — AWS | Scenario-Based | Advanced

> Your **Kinesis Data Stream** has 10 shards, showing `ProvisionedThroughputExceededException`, growing consumer lag, and dropped events. Diagnose and fix.

📁 **Reference:** `nawab312/AWS` — Kinesis Data Streams, sharding, enhanced fan-out

```
Kinesis shard limits (per shard):
  Write: 1 MB/s OR 1,000 records/second
  Read:  2 MB/s (shared across ALL consumers on that shard)

  10 shards total:
  Write capacity: 10 MB/s, 10,000 records/sec
  Read capacity:  20 MB/s total (2 MB/s × 10 shards)

Step 1 — Identify the bottleneck:
  CloudWatch metrics to check:
    GetRecords.IteratorAgeMilliseconds → consumer lag (age of oldest unread record)
    WriteProvisionedThroughputExceeded → producers hitting write limit
    ReadProvisionedThroughputExceeded  → consumers hitting read limit

  aws cloudwatch get-metric-statistics \
    --namespace AWS/Kinesis \
    --metric-name WriteProvisionedThroughputExceeded \
    --dimensions Name=StreamName,Value=my-stream

Step 2 — Identify hot shards:
  # Hot shard = one shard receiving disproportionate traffic
  # Caused by: bad partition key (all records use same key)
  aws kinesis describe-stream-summary --stream-name my-stream
  # Check: are all shards receiving equal traffic?

  Fix bad partition key:
  # BAD: same partition key for all records
  partition_key = "device-type"       # all IoT devices same type

  # GOOD: high-cardinality partition key
  partition_key = device_id           # unique per device
  partition_key = str(uuid.uuid4())   # random (even spread)

Step 3 — Fix write throttling:
  Option A: Reshard (increase shard count):
    aws kinesis update-shard-count \
      --stream-name my-stream \
      --target-shard-count 20 \    # double from 10
      --scaling-type UNIFORM_SCALING
    # Takes ~10-15 minutes, stream stays available during resharding
    # Cost: $0.015 per shard-hour → $0.30/hour for 20 shards

  Option B: Producer-side retry with exponential backoff:
    # SDK handles retries automatically with backoff
    # Ensure: KinesisRetryPolicy configured in producer

Step 4 — Fix read throttling (consumer lag):
  Problem: multiple consumers share 2 MB/s per shard
  3 consumers × same shard = each gets only 0.67 MB/s

  Option A: Enhanced Fan-Out (each consumer gets dedicated 2 MB/s):
    aws kinesis register-stream-consumer \
      --stream-arn arn:aws:kinesis:...:stream/my-stream \
      --consumer-name my-consumer-app
    # Each registered consumer: 2 MB/s dedicated read throughput
    # Cost: $0.015 per consumer-shard-hour additional

  Option B: Fan-out via separate Kinesis streams:
    Stream A → Lambda (fan-out) → Stream B (consumer 1)
                               → Stream C (consumer 2)

Reshard decision logic:
  Reshard when:
  → Write: consistently > 80% of shard write capacity
  → Read:  IteratorAgeMilliseconds growing over time (lag increasing)
  → Single producer throughput nearing 1 MB/s limit

  Calculate required shards:
    Peak write rate: 8 MB/s
    Buffer:          +25% = 10 MB/s required
    Shards needed:   10 MB/s ÷ 1 MB/s per shard = 10 shards minimum
    Recommendation:  15 shards (50% headroom)
```

💡 **Interview tip:** Hot shards from a bad partition key are the most common Kinesis throughput problem — always check partition key cardinality first. Using device ID or a UUID guarantees even distribution across shards. Enhanced Fan-Out is the answer when multiple consumers are competing for the same shard read throughput — each consumer gets a dedicated 2 MB/s pipe. Mention that resharding is not instant — it takes 10-15 minutes and temporarily doubles the number of shards (old + new), so plan resharding during low-traffic periods.

---

### Q138 — Linux / Bash | Conceptual | Advanced

> Explain **systemd** vs **SysV init**. Explain unit types. Write a systemd service file for a Node.js app with auto-restart, resource limits, and journald logging.

📁 **Reference:** `nawab312/DSA` → `Linux` — systemd, service management

```
SysV init (legacy):
  → Shell scripts in /etc/init.d/ for each service
  → Sequential startup (service A must finish before B starts)
  → Manual dependency management (no built-in dependency resolution)
  → No socket activation, no process supervision
  → Hard to restart on failure (manual watchdog scripts needed)

systemd (modern — default on Ubuntu 16+, RHEL 7+):
  → Parallel service startup (faster boot)
  → Declarative unit files (not shell scripts)
  → Automatic dependency resolution (After=, Requires=, Wants=)
  → Built-in process supervision and auto-restart
  → Socket activation (start service only when connection arrives)
  → Resource limits via cgroups integration
  → Centralized logging via journald

Unit types:
  .service  → manages a daemon/process (most common)
  .timer    → cron replacement — runs at scheduled times
  .socket   → socket activation — starts service on first connection
  .target   → group of units (like runlevels): multi-user.target
  .mount    → manages filesystem mount points
  .path     → watches filesystem paths, triggers service on change
  .slice    → organizes units into cgroup hierarchy

Node.js app service file:
  # /etc/systemd/system/myapp.service

  [Unit]
  Description=My Node.js Application
  After=network.target          # start after networking is up
  Requires=network.target       # hard dependency on network

  [Service]
  Type=simple
  User=nodeuser                 # run as non-root user
  Group=nodeuser
  WorkingDirectory=/opt/myapp
  ExecStart=/usr/bin/node /opt/myapp/server.js
  ExecReload=/bin/kill -HUP $MAINPID   # graceful reload on SIGHUP

  # Restart policy
  Restart=on-failure            # restart if exits non-zero
  RestartSec=5s                 # wait 5 seconds before restart
  StartLimitIntervalSec=60s     # within 60 seconds
  StartLimitBurst=3             # allow max 3 restarts, then give up

  # Environment
  EnvironmentFile=/opt/myapp/.env   # load env vars from file
  Environment=NODE_ENV=production

  # Resource limits (cgroup-based)
  CPUQuota=50%                  # max 50% of one CPU core
  MemoryMax=512M                # hard memory limit (OOM kill at 512MB)
  MemoryHigh=400M               # soft limit (throttle at 400MB)
  TasksMax=100                  # max 100 threads/processes

  # Security hardening
  NoNewPrivileges=true
  PrivateTmp=true               # isolated /tmp
  ReadOnlyPaths=/              # read-only filesystem
  ReadWritePaths=/opt/myapp/logs /tmp

  # Logging — goes to journald automatically
  StandardOutput=journal
  StandardError=journal
  SyslogIdentifier=myapp       # tag in journald

  [Install]
  WantedBy=multi-user.target   # start in normal boot

Manage the service:
  systemctl daemon-reload           # reload after editing unit file
  systemctl enable myapp            # start on boot
  systemctl start myapp             # start now
  systemctl status myapp            # check status
  journalctl -u myapp -f            # follow logs
  journalctl -u myapp --since "1h ago"  # last hour of logs

Timer unit (cron replacement):
  # /etc/systemd/system/cleanup.timer
  [Unit]
  Description=Daily cleanup

  [Timer]
  OnCalendar=daily              # run at midnight
  Persistent=true               # run if missed (machine was off)

  [Install]
  WantedBy=timers.target
```

💡 **Interview tip:** The key systemd advantage over SysV is parallel startup and built-in supervision — systemd watches the process and restarts it automatically on failure, eliminating the need for tools like Supervisor or PM2 for Node.js. `StartLimitBurst=3` combined with `StartLimitIntervalSec=60s` prevents a crash-looping service from consuming all resources — after 3 crashes in 60 seconds, systemd gives up and marks the service as failed. Mention that `MemoryMax` vs `MemoryHigh`: `MemoryHigh` is a soft limit that triggers throttling while `MemoryMax` is a hard limit that triggers OOM kill.

---

### Q139 — Terraform | Conceptual | Advanced

> Explain **Terraform `data sources`** vs resources. Give 5 real-world AWS data source examples and explain what problem each solves.

📁 **Reference:** `nawab312/Terraform` — data sources, AWS provider

```
Resource vs Data Source:
  Resource:    Terraform CREATES and MANAGES this (owns lifecycle)
  Data Source: Terraform READS existing infrastructure (read-only)

  resource "aws_vpc" "new" { cidr_block = "10.0.0.0/16" }
  → Terraform creates this VPC, tracks it in state

  data "aws_vpc" "existing" { tags = { Name = "prod-vpc" } }
  → Terraform reads the VPC that already exists (created by another team)
  → No state management, no destroy risk

When to use data source vs hardcode:
  Hardcode: "vpc-0abc123"        → breaks if VPC is recreated (ID changes)
  Data source: lookup by tag     → always finds the right VPC dynamically
  Data source also enables: cross-team references without coupling state files

5 Real-world data source examples:

1. aws_vpc — look up existing VPC by tag:
   data "aws_vpc" "main" {
     tags = { Name = "production-vpc", Environment = "prod" }
   }
   resource "aws_subnet" "app" {
     vpc_id     = data.aws_vpc.main.id   # dynamic, not hardcoded
     cidr_block = "10.0.10.0/24"
   }
   Problem solved: networking team owns VPC, app team reads it
   without needing to know the VPC ID

2. aws_ami — find latest approved AMI:
   data "aws_ami" "ubuntu" {
     most_recent = true
     owners      = ["099720109477"]    # Canonical (Ubuntu)
     filter {
       name   = "name"
       values = ["ubuntu/images/hvm-ssd/ubuntu-22.04-amd64-server-*"]
     }
   }
   resource "aws_instance" "web" {
     ami = data.aws_ami.ubuntu.id      # always gets latest AMI
   }
   Problem solved: no need to update AMI ID manually every month

3. aws_caller_identity — get current account ID:
   data "aws_caller_identity" "current" {}
   resource "aws_s3_bucket_policy" "this" {
     policy = jsonencode({
       Statement = [{
         Principal = { AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root" }
       }]
     })
   }
   Problem solved: policy works in any account without hardcoding account IDs

4. aws_route53_zone — look up hosted zone:
   data "aws_route53_zone" "main" {
     name = "company.com."
   }
   resource "aws_route53_record" "api" {
     zone_id = data.aws_route53_zone.main.zone_id
     name    = "api.company.com"
     type    = "A"
   }
   Problem solved: zone ID never hardcoded — works across environments

5. terraform_remote_state — read outputs from another state:
   data "terraform_remote_state" "networking" {
     backend = "s3"
     config = {
       bucket = "company-terraform-state"
       key    = "networking/terraform.tfstate"
       region = "us-east-1"
     }
   }
   resource "aws_db_subnet_group" "main" {
     subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
   }
   Problem solved: app team reads networking team's outputs
   without duplicating resources or tightly coupling state files

Data sources execute at plan time:
  → If the looked-up resource does not exist → plan fails
  → Use depends_on if data source requires a resource to exist first
  → Data sources are refreshed on every plan (always current)
```

💡 **Interview tip:** `terraform_remote_state` is the most architecturally important data source — it's how you share outputs between independent Terraform configurations without coupling them into one monolithic state file. The `aws_ami` data source with `most_recent = true` is a best practice for keeping EC2 launch templates current without manual AMI updates. A common gotcha: data sources run at plan time, so if you're creating a VPC and trying to look it up with a data source in the same apply, the lookup will fail — you must reference the resource directly in that case.

---

### Q140 — Prometheus | Scenario-Based | Advanced

> Write complete **PrometheusRules** and **AlertmanagerConfig** for: node disk > 80%, pod restarts > 5 in 1hr, cluster CPU > 85% for 10min, Deployment with 0 replicas. Route to different Slack channels by severity.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`

```yaml
# PrometheusRule — alerting rules
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cluster-alerts
  labels:
    release: prometheus
spec:
  groups:
  - name: node_alerts
    rules:
    - alert: NodeDiskUsageHigh
      expr: |
        (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}
           / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}) * 100 > 80
      for: 5m
      labels:
        severity: warning
        channel: infra-alerts
      annotations:
        summary: "Node {{ $labels.instance }} disk usage {{ $value | printf \"%.0f\" }}%"
        description: "Disk {{ $labels.mountpoint }} on {{ $labels.instance }} is above 80%"

    - alert: NodeDiskCritical
      expr: |
        (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}
           / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}) * 100 > 95
      for: 2m
      labels:
        severity: critical
        channel: oncall-critical
      annotations:
        summary: "CRITICAL: Node {{ $labels.instance }} disk at {{ $value | printf \"%.0f\" }}%"

  - name: pod_alerts
    rules:
    - alert: PodRestartingFrequently
      expr: |
        increase(kube_pod_container_status_restarts_total[1h]) > 5
      for: 0m
      labels:
        severity: warning
        channel: app-alerts
      annotations:
        summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} restarted {{ $value }} times in 1hr"

  - name: cluster_alerts
    rules:
    - alert: ClusterCPUHigh
      expr: |
        avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) * 100 > 85
      for: 10m
      labels:
        severity: warning
        channel: infra-alerts
      annotations:
        summary: "Cluster CPU usage at {{ $value | printf \"%.0f\" }}%"

    - alert: DeploymentUnavailable
      expr: |
        kube_deployment_status_replicas_available == 0
          and kube_deployment_spec_replicas > 0
      for: 1m
      labels:
        severity: critical
        channel: oncall-critical
      annotations:
        summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} has 0 available replicas"

---
# AlertmanagerConfig — routing by severity/channel label
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: cluster-alert-routing
  namespace: monitoring
spec:
  route:
    receiver: default-slack
    groupBy: ['alertname', 'namespace']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 4h
    routes:
    - matchers:
      - name: channel
        value: oncall-critical
      receiver: oncall-critical-slack
      repeatInterval: 1h

    - matchers:
      - name: channel
        value: infra-alerts
      receiver: infra-slack
      repeatInterval: 6h

    - matchers:
      - name: channel
        value: app-alerts
      receiver: app-slack
      repeatInterval: 4h

  receivers:
  - name: default-slack
    slackConfigs:
    - apiURL:
        name: slack-webhook-secret
        key: url
      channel: '#alerts-general'
      title: '[{{ .Status | toUpper }}] {{ .CommonAnnotations.summary }}'
      text: '{{ .CommonAnnotations.description }}'

  - name: oncall-critical-slack
    slackConfigs:
    - apiURL:
        name: slack-webhook-secret
        key: url
      channel: '#oncall-critical'
      title: ':rotating_light: CRITICAL: {{ .CommonAnnotations.summary }}'

  - name: infra-slack
    slackConfigs:
    - apiURL:
        name: slack-webhook-secret
        key: url
      channel: '#infra-alerts'

  - name: app-slack
    slackConfigs:
    - apiURL:
        name: slack-webhook-secret
        key: url
      channel: '#app-alerts'
```

💡 **Interview tip:** The disk usage query using `node_filesystem_avail_bytes / node_filesystem_size_bytes` with `fstype!~"tmpfs|overlay"` exclusion is important — without filtering, you get noise from container overlay filesystems. The `DeploymentUnavailable` rule uses a compound condition: `replicas_available == 0 AND spec_replicas > 0` — the second condition prevents false alerts for intentionally scaled-down deployments. Routing by a custom label (`channel`) on the alert is cleaner than routing by severity alone — it lets you send different types of alerts to different teams without complex routing trees.

---

### Q141 — Kubernetes | Troubleshooting | Advanced

> HPA shows `<unknown>/50%` for CPU. What does this mean, why does it happen, complete fix?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

```
What <unknown>/50% means:
  Target: 50% CPU utilisation
  Current: <unknown> = HPA cannot read current CPU metrics
  HPA is blind — it cannot make scaling decisions
  Result: HPA does nothing, no scale-out even under heavy load

Step 1 — Check HPA status:
  kubectl describe hpa my-hpa -n production

  Events section shows:
  "unable to get metrics for resource cpu: unable to fetch metrics
   from resource metrics API: the server is currently unable to handle
   the request (get pods.metrics.k8s.io)"

  This confirms: Metrics Server is not reachable

Step 2 — Check if Metrics Server is installed:
  kubectl get deployment metrics-server -n kube-system
  → Error: not found → Metrics Server not installed

  Install Metrics Server:
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

  For EKS:
  helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
  helm install metrics-server metrics-server/metrics-server \
    -n kube-system

Step 3 — If Metrics Server installed but HPA still shows unknown:
  kubectl get pods -n kube-system | grep metrics-server
  kubectl logs -n kube-system deployment/metrics-server

  Common error:
  "x509: certificate signed by unknown authority"
  → Metrics Server cannot verify kubelet TLS cert

  Fix — add --kubelet-insecure-tls flag:
  kubectl edit deployment metrics-server -n kube-system
  # Add to args:
  - --kubelet-insecure-tls
  - --kubelet-preferred-address-types=InternalIP

Step 4 — Check pod has CPU resource REQUEST defined:
  # HPA reads CPU utilisation as: actual CPU / requested CPU
  # Without a request, the denominator is 0 → undefined → <unknown>

  # BAD — no resource request:
  containers:
  - name: app
    image: myapp:v1
    # no resources block

  # GOOD — resource request defined:
  containers:
  - name: app
    image: myapp:v1
    resources:
      requests:
        cpu: "100m"      # HPA uses this as the denominator
        memory: "128Mi"

Step 5 — Verify metrics are flowing:
  kubectl top nodes             # should show CPU/memory per node
  kubectl top pods -n production # should show CPU/memory per pod
  # If these work, HPA should work too

Step 6 — Check HPA configuration:
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: my-app
    minReplicas: 2
    maxReplicas: 20
    metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50   # 50% of requested CPU
```

💡 **Interview tip:** The root cause is almost always one of two things: Metrics Server not installed (or crashed), or the target pods missing CPU resource requests. Both result in `<unknown>`. The resource request is the denominator in the HPA calculation — `actual_cpu / requested_cpu` — so without a request, the percentage is undefined. In production, always enforce resource requests via LimitRange or OPA/Gatekeeper policies to prevent this class of HPA failure. On EKS, the `--kubelet-insecure-tls` flag is commonly needed because EKS kubelets use certificates the Metrics Server can't verify by default.

---

### Q142 — Git | Conceptual | Advanced

> Explain **Git hooks**. Give 5 useful hooks for a DevOps team. Write a `pre-commit` hook that blocks files > 1MB, scans for secrets, and runs linting.

📁 **Reference:** `nawab312/CI_CD` → `Git`

```
Git hooks:
  → Shell scripts that run automatically at specific Git lifecycle events
  → Live in: .git/hooks/ (local, not committed to repo by default)
  → Or: managed by pre-commit framework (.pre-commit-config.yaml in repo)
  → Client-side hooks: run on developer's machine
  → Server-side hooks: run on the Git server (GitLab, GitHub Enterprise)

Hook types and when they run:
  pre-commit      → before commit is created (can prevent commit)
  commit-msg      → validates commit message format
  pre-push        → before pushing to remote (can prevent push)
  post-commit     → after commit is created (notifications, etc.)
  post-merge      → after git merge completes
  post-checkout   → after git checkout
  pre-receive     → server-side: before push is accepted
  post-receive    → server-side: after push is accepted (trigger CI)

5 useful Git hooks for a DevOps team:

1. pre-commit — block secrets and large files (see below)
2. commit-msg — enforce conventional commit format:
   #!/bin/bash
   MSG=$(cat "$1")
   PATTERN="^(feat|fix|chore|docs|ci|refactor|test|perf)(\(.+\))?: .{10,}"
   if ! echo "$MSG" | grep -qE "$PATTERN"; then
     echo "ERROR: Commit message must follow: type(scope): description"
     echo "Example: feat(auth): add OAuth2 login support"
     exit 1
   fi

3. pre-push — run tests before pushing:
   #!/bin/bash
   npm test || { echo "Tests failed — push blocked"; exit 1; }

4. post-commit — update Jira ticket status:
   #!/bin/bash
   BRANCH=$(git rev-parse --abbrev-ref HEAD)
   TICKET=$(echo "$BRANCH" | grep -oE 'PROJ-[0-9]+')
   [ -n "$TICKET" ] && curl -X POST "$JIRA_URL/issue/$TICKET/transitions" ...

5. post-merge — install dependencies after pulling:
   #!/bin/bash
   if git diff-tree -r --name-only --no-commit-id ORIG_HEAD HEAD | grep -q "package.json"; then
     npm install
   fi

pre-commit hook — blocks large files + secrets + linting:
  #!/bin/bash
  set -e

  echo "Running pre-commit checks..."

  # Check 1: Block files larger than 1MB
  MAX_SIZE=1048576  # 1MB in bytes
  for file in $(git diff --cached --name-only); do
    if [ -f "$file" ]; then
      SIZE=$(wc -c < "$file")
      if [ "$SIZE" -gt "$MAX_SIZE" ]; then
        echo "ERROR: $file is $(($SIZE / 1024))KB — exceeds 1MB limit"
        echo "Use Git LFS for large files: git lfs track '$file'"
        exit 1
      fi
    fi
  done

  # Check 2: Scan for hardcoded secrets
  PATTERNS=(
    'AKIA[0-9A-Z]{16}'           # AWS Access Key
    'aws_secret_access_key\s*=\s*[A-Za-z0-9/+]{40}'
    'password\s*=\s*["\x27][^"]{8,}'
    'private_key.*BEGIN'
    'Authorization:\s*Bearer\s+[A-Za-z0-9._-]{20,}'
  )

  for file in $(git diff --cached --name-only --diff-filter=ACM); do
    for pattern in "${PATTERNS[@]}"; do
      if git show ":$file" 2>/dev/null | grep -qiE "$pattern"; then
        echo "ERROR: Potential secret found in $file"
        echo "Pattern matched: $pattern"
        echo "Use git-secrets or detect-secrets to manage this"
        exit 1
      fi
    done
  done

  # Check 3: Run ESLint on staged JS/TS files
  JS_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(js|ts|jsx|tsx)$')
  if [ -n "$JS_FILES" ]; then
    npx eslint $JS_FILES || { echo "ESLint failed"; exit 1; }
  fi

  echo "All pre-commit checks passed"

Install via pre-commit framework (recommended over manual hooks):
  pip install pre-commit
  pre-commit install   # installs .git/hooks/pre-commit
  # .pre-commit-config.yaml committed to repo — shared by all developers
```

💡 **Interview tip:** The pre-commit framework (pre-commit.com) is the production answer for managing Git hooks — hooks are defined in `.pre-commit-config.yaml` committed to the repo, so every developer gets the same hooks automatically after `pre-commit install`. Manual hook scripts in `.git/hooks/` are not committed and must be set up per developer manually. For secrets scanning, `detect-secrets` and `gitleaks` are more comprehensive than manual regex patterns. The `commit-msg` hook enforcing Conventional Commits format is especially valuable — it enables automatic changelog generation and semantic versioning from commit history.

---

### Q143 — AWS | Conceptual | Advanced

> Explain **ElastiCache Redis vs Memcached**. When to choose Redis? Explain Redis Cluster mode vs Replication Group. Real-world performance use case.

📁 **Reference:** `nawab312/AWS` — ElastiCache, Redis, Memcached

```
Memcached:
  → Simple distributed cache — key/value string storage only
  → Multi-threaded (uses multiple CPU cores efficiently)
  → No persistence — data lost on restart
  → No replication — node failure = data loss
  → No data structures — strings only
  → Horizontal scaling by adding nodes (consistent hashing)
  → Use when: pure caching, no persistence, maximum simplicity

Redis:
  → Rich data structures: strings, lists, sets, sorted sets, hashes,
    streams, bitmaps, HyperLogLog, geospatial
  → Optional persistence: RDB snapshots + AOF (append-only file)
  → Replication: master → replicas (read scaling + failover)
  → Pub/Sub messaging
  → Lua scripting (atomic complex operations)
  → TTL per key (automatic expiry)
  → Single-threaded (one CPU core for commands — but very fast)

Choose Redis over Memcached when:
  ✓ Need persistence (sessions that survive restart)
  ✓ Need pub/sub messaging between services
  ✓ Need sorted sets (leaderboards, priority queues)
  ✓ Need atomic operations on complex data types
  ✓ Need replication for read scaling or HA
  ✓ TTL per individual key (Memcached supports TTL too but no per-type)

Redis Replication Group (standard mode):
  ┌─────────────┐
  │   Primary   │  ← ALL writes go here
  └──────┬──────┘
         │ async replication
  ┌──────┴──────┐   ┌─────────┐
  │  Replica 1  │   │Replica 2│   ← reads distributed here
  └─────────────┘   └─────────┘

  One primary, 1-5 read replicas
  Automatic failover: replica promoted on primary failure (Multi-AZ)
  Max data size: limited to single node memory
  Use for: read-heavy workloads, HA, single dataset < node RAM

Redis Cluster mode:
  Data sharded across multiple primary nodes (up to 500 shards)
  Each primary has replicas for HA
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │Shard 1   │  │Shard 2   │  │Shard 3   │
  │Primary+R │  │Primary+R │  │Primary+R │
  │slots 0-5k│  │slots 5k-10k│ │slots 10k+│
  └──────────┘  └──────────┘  └──────────┘
  Total capacity: 3 × node_memory
  Use for: datasets larger than single node, millions of keys

Real-world performance use case — session store:

  Without ElastiCache:
    User login → session stored in app server memory
    10 app servers × each has some sessions
    Load balancer routes user to different server → session gone → logout
    Solution: sticky sessions → defeats load balancing purpose

  With ElastiCache Redis:
    User login → session stored in Redis (key: session-id, value: user data)
    Any app server handles any request → all read from shared Redis
    Session TTL: 30 minutes (auto-expire idle sessions)
    Latency: < 1ms (in-memory, same VPC)
    Before: DB query per request for session validation (~10ms)
    After: Redis GET per request (~0.5ms) → 20× faster

  Other use cases:
    Rate limiting: INCR + EXPIRE per IP (atomic counter)
    Leaderboard: sorted sets (ZADD, ZRANK, ZRANGE)
    Pub/Sub: microservices event notification
    Distributed lock: SET key value NX EX 30 (atomic lock with TTL)
```

💡 **Interview tip:** The most important distinction is: Memcached is pure cache (no persistence, no replication), Redis is a data structure server that can also be used as a cache. Choose Redis for almost everything in production — the persistence and replication features make it much more operationally safe. Cluster mode vs Replication Group: if your dataset fits in one node's memory (say, < 50GB), use Replication Group (simpler); if you need more memory or write throughput, use Cluster mode. The session store use case is the most relatable example for interviewers — it directly solves the sticky session problem while improving latency by 10-20×.

---

### Q144 — Jenkins | Troubleshooting | Advanced

> Jenkins build fails with `java.lang.OutOfMemoryError: Java heap space` after ~20 minutes. Diagnose and fix.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — JVM tuning, memory management

```
What causes Jenkins OOM:
  Jenkins master JVM runs: job scheduling, plugin execution, build coordination
  Build agents JVM runs: actual compilation, test execution
  OOM can occur on either

Common causes:
  1. Large test reports or build artifacts held in memory
  2. Plugins leaking memory (storing build history in memory)
  3. Too many concurrent builds exceeding heap
  4. Groovy scripts in pipelines creating large objects
  5. XML parsing of large pom.xml or configuration files

Step 1 — Identify where OOM occurs:
  # Check the full stack trace:
  grep -A 20 "OutOfMemoryError" ~/.jenkins/logs/jenkins.log

  java.lang.OutOfMemoryError: Java heap space
    at java.util.Arrays.copyOf(Arrays.java:3210)
    at org.jenkinsci.plugins.pipeline.maven.MavenSpyLogProcessor...
  → Maven plugin holding all test results in memory

Step 2 — Check current JVM settings:
  # Jenkins startup script (varies by install method):
  cat /etc/default/jenkins | grep JAVA_ARGS
  # or systemd unit:
  systemctl cat jenkins | grep JAVA_OPTS

  Typical default: -Xmx256m (only 256MB!) → clearly too low

Step 3 — Increase Jenkins heap:
  # /etc/default/jenkins or systemd override:
  JAVA_ARGS="-Xmx4g -Xms1g -XX:+UseG1GC -XX:MaxGCPauseMillis=200"

  # For Jenkins in Docker/Kubernetes:
  env:
  - name: JAVA_OPTS
    value: "-Xmx4g -Xms2g -XX:+UseG1GC"

  # For build agents running Groovy/Maven:
  MAVEN_OPTS="-Xmx2g -Xms512m"

Step 4 — Enable GC logging to find leak:
  JAVA_ARGS="... -Xlog:gc*:file=/var/log/jenkins/gc.log:time,uptime:filecount=5,filesize=20m"
  # Shows: when GC runs, how much freed, how often

Step 5 — Fix in pipeline code:
  # Groovy in pipelines runs in Jenkins master JVM
  # Large string operations in pipelines → memory pressure

  # BAD — loading entire file into string:
  def content = readFile('huge-test-report.xml')  // 500MB XML in memory

  # GOOD — stream processing or use stash/unstash for large files:
  archiveArtifacts artifacts: 'test-reports/**'  // store, don't load into memory

Step 6 — Build history and artifact cleanup:
  # Jenkins stores build history in memory for quick access
  # 1000 builds × artifacts = GB of memory

  # Discard old builds in pipeline:
  options {
    buildDiscarder(logRotator(
      numToKeepStr: '20',         # keep last 20 builds
      artifactNumToKeepStr: '5'   # keep artifacts for last 5 only
    ))
  }

Step 7 — Offload builds to agents (not master):
  # Jenkins master should ONLY coordinate, not run builds
  # Set master executors = 0:
  Manage Jenkins → Nodes → Built-In Node → # of executors = 0
  # All builds run on agents with their own JVM = separate heap
```

💡 **Interview tip:** Setting Jenkins master executors to 0 is the most important operational best practice — the master should only orchestrate, never run build code. When builds run on the master, any memory leak in a build plugin directly impacts Jenkins availability. For the OOM itself, increasing `-Xmx` is the immediate fix, but the root cause is almost always either insufficient build discard policy (thousands of builds accumulating in memory) or a plugin reading large files entirely into memory. Always check the stack trace — the class name in the trace points directly to which plugin is the culprit.

---

### Q145 — ELK Stack | Scenario-Based | Advanced

> Design **centralized Kubernetes logging** for 50 microservices — collect, enrich with K8s metadata, parse JSON/plaintext, store 30 days, searchable in 5 seconds.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack`

```
Architecture:

  Pod stdout/stderr
       │
       ▼
  Fluent Bit (DaemonSet)     ← runs on every node, reads container logs
       │
       ▼
  Kafka (buffer)             ← decouples ingestion from processing
       │
       ▼
  Logstash (processors)      ← parse, enrich, transform
       │
       ▼
  Elasticsearch (storage)    ← hot tier: 30 days
       │
       ▼
  Kibana (UI)                ← search, dashboards

Step 1 — Fluent Bit DaemonSet (collection + basic enrichment):

  apiVersion: apps/v1
  kind: DaemonSet
  metadata:
    name: fluent-bit
    namespace: logging
  spec:
    template:
      spec:
        serviceAccountName: fluent-bit   # needs read access to pod metadata
        containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.1
          volumeMounts:
          - name: varlog
            mountPath: /var/log
          - name: config
            mountPath: /fluent-bit/etc/
        volumes:
        - name: varlog
          hostPath:
            path: /var/log

  Fluent Bit config (fluent-bit.conf):
  [INPUT]
      Name              tail
      Path              /var/log/containers/*.log
      Parser            docker           # parse Docker JSON log format
      Tag               kube.*
      Refresh_Interval  5
      Mem_Buf_Limit     50MB

  [FILTER]
      Name              kubernetes
      Match             kube.*
      Kube_URL          https://kubernetes.default.svc:443
      Kube_CA_File      /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      Kube_Token_File   /var/run/secrets/kubernetes.io/serviceaccount/token
      Labels            On              # adds pod labels to log
      Annotations       Off             # skip annotations (reduce noise)
      # Adds: kubernetes.namespace_name, kubernetes.pod_name,
      #       kubernetes.container_name, kubernetes.labels.*

  [OUTPUT]
      Name  kafka
      Match *
      Brokers kafka-broker:9092
      Topics k8s-logs

Step 2 — Logstash (parsing + transformation):

  input {
    kafka {
      bootstrap_servers => "kafka-broker:9092"
      topics => ["k8s-logs"]
      codec => json
    }
  }

  filter {
    # Parse JSON app logs
    if [log] =~ /^\{/ {
      json { source => "log" }    # app logs JSON directly into fields
    }
    # Parse plaintext logs (nginx, etc.)
    else if [kubernetes][container_name] == "nginx" {
      grok {
        match => { "log" => "%{COMBINEDAPACHELOG}" }
      }
    }

    # Add @timestamp from log or use ingestion time
    date {
      match => ["timestamp", "ISO8601"]
      target => "@timestamp"
      remove_field => ["timestamp"]
    }

    # Route to correct index per namespace
    mutate {
      add_field => { "[@metadata][index]" =>
        "k8s-logs-%{[kubernetes][namespace_name]}-%{+YYYY.MM}" }
    }
  }

  output {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "%{[@metadata][index]}"
      template_name => "k8s-logs"
    }
  }

Step 3 — Elasticsearch index template (30-day ILM):

  PUT _index_template/k8s-logs
  {
    "index_patterns": ["k8s-logs-*"],
    "template": {
      "settings": {
        "number_of_shards": 2,
        "number_of_replicas": 1,
        "index.lifecycle.name": "k8s-logs-30d",
        "index.lifecycle.rollover_alias": "k8s-logs"
      }
    }
  }

  ILM policy (30-day retention):
  PUT _ilm/policy/k8s-logs-30d
  {
    "policy": {
      "phases": {
        "hot":    { "min_age": "0ms",  "actions": { "rollover": { "max_age": "1d", "max_size": "20gb" }}},
        "delete": { "min_age": "30d",  "actions": { "delete": {}}}
      }
    }
  }

5-second searchability SLA:
  → Elasticsearch default refresh interval: 1 second
  → Index refresh: every 1s makes new docs searchable
  → Fluent Bit → Kafka → Logstash → ES: total latency 3-5 seconds typical
  → For near-real-time: Fluent Bit direct to ES (skip Kafka/Logstash)
    but lose the buffer and transformation capability
```

💡 **Interview tip:** The Kafka buffer between Fluent Bit and Logstash is the key architectural decision — without it, if Elasticsearch is slow or restarting, Fluent Bit's memory buffer fills up and logs are lost. Kafka provides durability and allows Logstash to process at its own pace. Fluent Bit is preferred over Filebeat for Kubernetes because its Kubernetes filter natively enriches logs with pod metadata via the API server — namespace, pod name, labels — without any configuration. The 5-second searchability requirement is met by Elasticsearch's 1-second default refresh interval.

---

### Q146 — Kubernetes | Scenario-Based | Advanced

> Set up **cert-manager** end-to-end for 20 services needing Let's Encrypt TLS certificates — from installation to automatic renewal.

📁 **Reference:** `nawab312/Kubernetes` → `SSLCertificate_Kubernetes.md`

```
Step 1 — Install cert-manager:
  helm repo add jetstack https://charts.jetstack.io
  helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --set installCRDs=true

  # Verify:
  kubectl get pods -n cert-manager
  # cert-manager-xxx, cert-manager-cainjector-xxx, cert-manager-webhook-xxx

Step 2 — Create ClusterIssuer (Let's Encrypt):

  # Staging first (rate limit friendly):
  apiVersion: cert-manager.io/v1
  kind: ClusterIssuer
  metadata:
    name: letsencrypt-staging
  spec:
    acme:
      server: https://acme-staging-v02.api.letsencrypt.org/directory
      email: admin@company.com
      privateKeySecretRef:
        name: letsencrypt-staging-key
      solvers:
      - http01:
          ingress:
            class: nginx    # must match your Ingress class

  # Production (after testing staging works):
  apiVersion: cert-manager.io/v1
  kind: ClusterIssuer
  metadata:
    name: letsencrypt-prod
  spec:
    acme:
      server: https://acme-v02.api.letsencrypt.org/directory
      email: admin@company.com
      privateKeySecretRef:
        name: letsencrypt-prod-key
      solvers:
      - http01:
          ingress:
            class: nginx

Step 3 — Ingress with TLS annotation (automatic certificate):

  # For each of your 20 services:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: payment-ingress
    annotations:
      cert-manager.io/cluster-issuer: "letsencrypt-prod"  # triggers cert issuance
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
  spec:
    tls:
    - hosts:
      - payment.company.com
      secretName: payment-tls   # cert-manager creates this Secret
    rules:
    - host: payment.company.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: payment-service
              port:
                number: 8080

  # cert-manager sees this Ingress → creates Certificate object automatically
  # Certificate object → cert-manager requests from Let's Encrypt via HTTP-01 challenge

Step 4 — What cert-manager does automatically:
  1. Sees Ingress with cert-manager annotation
  2. Creates Certificate CR in same namespace
  3. Creates CertificateRequest + Order + Challenge
  4. Exposes /.well-known/acme-challenge/<token> via Ingress
  5. Let's Encrypt validates → issues certificate
  6. cert-manager stores cert in the named Secret (payment-tls)
  7. Nginx Ingress uses the Secret for TLS termination

Step 5 — Monitor certificate status:
  kubectl get certificate -A
  # NAME          READY  SECRET        AGE
  # payment-tls   True   payment-tls   5d

  kubectl describe certificate payment-tls -n production
  # Events: Successfully issued certificate
  # Not After: 2024-04-15 (Let's Encrypt certs valid 90 days)

  # cert-manager auto-renews at 2/3 of lifetime (60 days = renew at day 60)
  # Renewal is fully automatic — no manual intervention needed

Step 6 — Wildcard certificate (for all 20 services at once):
  # One cert for *.company.com → covers all subdomains
  # Requires DNS-01 challenge (not HTTP-01):

  apiVersion: cert-manager.io/v1
  kind: Certificate
  metadata:
    name: wildcard-company
  spec:
    secretName: wildcard-company-tls
    issuerRef:
      name: letsencrypt-prod
      kind: ClusterIssuer
    commonName: "*.company.com"
    dnsNames:
    - "*.company.com"
    - "company.com"

  # DNS-01 requires: cert-manager → Route53 (create TXT record for validation)
  # Needs: IAM role with route53:ChangeResourceRecordSets permission
```

💡 **Interview tip:** Always test with the Let's Encrypt staging environment first — the production environment has strict rate limits (5 certificates per domain per week) and a bad configuration will exhaust your quota quickly. The annotation `cert-manager.io/cluster-issuer` on the Ingress is the trigger that tells cert-manager to automatically create and manage the certificate — without this annotation, cert-manager does nothing. For 20 services, consider using a wildcard certificate with DNS-01 challenge instead of 20 individual certificates — this uses one Let's Encrypt request and one Secret, shared by all Ingresses.

---

### Q147 — Linux / Bash | Troubleshooting | Advanced

> Application throwing `Too many open files`. `ulimit -n` shows 1024. Increasing it doesn't stick after a few hours. Complete investigation and permanent fix.

📁 **Reference:** `nawab312/DSA` → `Linux` — file descriptors, ulimit

```
What "too many open files" means:
  Every open file, socket, pipe = one file descriptor (FD)
  Process has a limit on how many FDs it can open simultaneously
  ulimit -n = soft limit for current shell session
  1024 = default soft limit (very low for production apps)

Step 1 — Find which process is hitting the limit:
  # Find processes with most open FDs:
  lsof 2>/dev/null | awk '{print $2}' | sort | uniq -c | sort -rn | head -20
  # Output: count PID — highest count is your culprit

  # Get FD count for a specific PID:
  ls -1 /proc/<PID>/fd | wc -l

  # See what those FDs are:
  ls -la /proc/<PID>/fd | head -30
  # Mostly sockets → connection leak
  # Mostly regular files → file handle leak

Step 2 — Check current limits for the process:
  cat /proc/<PID>/limits
  # Max open files   1024   4096   files
  # soft=1024  hard=4096

Step 3 — Identify the leak type:
  # Watch FD count grow in real time:
  watch -n 1 "ls -1 /proc/<PID>/fd | wc -l"
  # If count grows steadily → leak (FDs opened but never closed)
  # If count stabilises → just need higher limit

  # For network connections specifically:
  ss -s | grep -i estab         # established socket count
  lsof -p <PID> | grep -c ESTABLISHED

Step 4 — Increase limits permanently (the right way):

  Method 1 — /etc/security/limits.conf:
    # Add for specific user or app user:
    myappuser  soft  nofile  65536
    myappuser  hard  nofile  65536
    # Or for all users:
    *          soft  nofile  65536
    *          hard  nofile  65536
    # Applies after NEXT LOGIN — doesn't affect running processes

  Method 2 — systemd service (recommended):
    [Service]
    LimitNOFILE=65536
    # Takes effect on next service restart
    # Overrides limits.conf for this service

  Method 3 — /etc/sysctl.conf (system-wide kernel limit):
    fs.file-max = 2097152        # total FDs for entire system
    sysctl -p                    # apply without reboot

  Why increasing ulimit doesn't stick:
    ulimit -n 65536              # only applies to CURRENT shell session
    After shell exits → gone
    After service restarts → reverts to default
    Fix: set in limits.conf AND systemd unit, not just ulimit command

Step 5 — Fix the application leak (root cause):
  # DB connection leak (Java/Python):
    try:
      conn = get_db_connection()
      result = conn.execute(query)
      return result
      # BUG: conn.close() never called if exception occurs

    # FIX: use context manager / try-finally:
    with get_db_connection() as conn:
      result = conn.execute(query)
      return result    # auto-closes on exit

  # File handle leak:
    f = open('/tmp/report.csv')
    data = f.read()
    # BUG: f.close() missing

    with open('/tmp/report.csv') as f:  # FIX
      data = f.read()

Step 6 — Monitor proactively:
  # Alert when FD usage > 80% of limit:
  alert: HighFileDescriptorUsage
  expr: process_open_fds / process_max_fds > 0.8
  for: 5m
  # process_open_fds exposed by node_exporter and most app exporters
```

💡 **Interview tip:** The "doesn't stick" detail in the question is the key — `ulimit -n` only sets the limit for the current shell session. The permanent fix has two parts: `limits.conf` for login sessions AND `LimitNOFILE` in the systemd unit for services. Both are needed because systemd ignores `limits.conf` by default. The root cause is almost always a resource leak in application code — increasing the limit buys time but doesn't fix the underlying problem. Always check if FD count is growing over time (leak) or just high but stable (needs higher limit).

---

### Q148 — Terraform | Scenario-Based | Advanced

> Build a **Terraform module for ECS service** used by 15 teams — versioned, published to private registry, backward compatible, documented, tested.

📁 **Reference:** `nawab312/Terraform` — module design, versioning, registry

```
Module structure (well-organized):
  modules/ecs-service/
  ├── main.tf           # ECS task definition, service, IAM
  ├── variables.tf      # all inputs with descriptions and types
  ├── outputs.tf        # service ARN, task definition ARN, etc.
  ├── versions.tf       # required_providers with version constraints
  ├── README.md         # generated by terraform-docs
  └── examples/
      ├── basic/        # minimal working example
      └── with-autoscaling/  # advanced example

variables.tf — typed, described, defaulted:
  variable "service_name" {
    type        = string
    description = "Name of the ECS service. Used as prefix for all resources."
    validation {
      condition     = can(regex("^[a-z][a-z0-9-]{2,30}$", var.service_name))
      error_message = "Service name must be lowercase alphanumeric with hyphens, 3-31 chars."
    }
  }

  variable "desired_count" {
    type        = number
    description = "Desired number of running tasks."
    default     = 2
    validation {
      condition     = var.desired_count >= 1 && var.desired_count <= 100
      error_message = "desired_count must be between 1 and 100."
    }
  }

  # New optional feature — backward compatible (has default):
  variable "enable_execute_command" {
    type        = bool
    description = "Enable ECS Exec for interactive debugging."
    default     = false   # existing callers unaffected (default = off)
  }

Versioning strategy (semantic versioning):
  v1.0.0  → initial release
  v1.1.0  → new OPTIONAL variable (backward compatible)
  v1.2.0  → new output added (backward compatible)
  v2.0.0  → BREAKING: renamed variable or removed feature

  Teams pin to major version:
  module "api" {
    source  = "registry.company.com/platform/ecs-service/aws"
    version = "~> 1.0"   # any 1.x version (auto-gets latest minor/patch)
  }

  Breaking changes process:
  1. Release v2.0.0 with migration guide in CHANGELOG
  2. Keep v1.x branch maintained for 6 months (security fixes only)
  3. Notify all teams via email/Slack with migration steps
  4. After 6 months: deprecate v1.x

Private registry options:
  Option A: Terraform Cloud Private Registry (simplest)
    terraform login registry.terraform.io
    # Upload module via VCS connection or directly

  Option B: GitLab Terraform Registry:
    # Push tag to GitLab → auto-published
    source = "gitlab.company.com/platform/terraform/ecs-service"

  Option C: S3 + Git tag (simplest self-hosted):
    source = "git::https://github.com/company/tf-modules.git//ecs-service?ref=v1.2.0"

Testing strategy:

  Level 1 — Validation (fast, no AWS calls):
    terraform init
    terraform validate
    terraform fmt -check
    tflint --module   # lint for best practices and deprecated syntax

  Level 2 — Plan testing with Terratest (Go):
    func TestECSServiceModule(t *testing.T) {
      opts := &terraform.Options{
        TerraformDir: "../examples/basic",
        Vars: map[string]interface{}{
          "service_name": "test-svc",
          "cluster_arn":  "arn:aws:ecs:...",
        },
      }
      defer terraform.Destroy(t, opts)
      terraform.InitAndApply(t, opts)

      serviceARN := terraform.Output(t, opts, "service_arn")
      assert.Contains(t, serviceARN, "arn:aws:ecs")
    }
    # Creates real AWS resources → runs tests → destroys
    # Run in isolated test AWS account

  Level 3 — Contract testing:
    # Validate outputs match expected types and format
    # Check resource naming conventions are followed

Documentation (auto-generated):
  terraform-docs markdown . > README.md
  # Generates: inputs table, outputs table, resource list
  # Run in CI on every PR to keep docs current
```

💡 **Interview tip:** Backward compatibility is the most important design constraint for shared modules — a breaking change in a shared module can break 15 teams simultaneously. The pattern of having `default` values for all new optional variables ensures existing callers are not affected. Terratest is the standard answer for module testing — it creates real AWS resources, validates outputs, then destroys everything. Run Terratest in a dedicated AWS sandbox account with strict IAM limits to prevent test resources from accumulating costs. The `~> 1.0` version constraint in the module source is the recommended pattern — it takes any 1.x version automatically for non-breaking improvements.

---

### Q149 — GitHub Actions | Conceptual | Advanced

> Explain **self-hosted vs GitHub-hosted runners**. Security risks of self-hosted. Auto-scaling on AWS EC2.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions`

```
GitHub-hosted runners:
  → AWS/Azure/Google VMs managed by GitHub
  → Clean environment per job (fresh VM every time)
  → Available: ubuntu-latest, windows-latest, macos-latest
  → Pre-installed: common tools (node, python, docker, kubectl, etc.)
  → Limitations:
    - 2 vCPU, 7GB RAM, 14GB SSD (standard)
    - 20,000 minutes/month free (then $0.008/min for Linux)
    - No access to private network (VPC, RDS, internal services)
    - Cannot cache Docker layers between jobs (fresh disk each time)

Self-hosted runners:
  → Your own machines (EC2, on-prem, VMs)
  → Persistent environment (reuse between jobs)
  → Access to private network (RDS, internal services, private ECR)
  → Custom tools and configurations pre-installed
  → No minute limits (you pay for EC2 only)
  → Required when: builds need VPC access, special hardware (GPU), large caches

Security risks of self-hosted runners:

  Risk 1: Code execution on your infrastructure
    PR from external contributor → workflow runs → malicious code
    executed ON your runner → has access to secrets, VPC, everything
    Fix: NEVER use self-hosted runners for public repositories
    Fix: Require approval for PRs from first-time contributors

  Risk 2: Runner token exposure
    Runner registration token can register other runners
    If token leaked → attacker registers their runner
    Fix: short-lived registration tokens, rotate regularly

  Risk 3: Persistent state between jobs
    Malicious job leaves backdoor files on runner
    Next job (different repo/team) picks up the poisoned state
    Fix: use ephemeral runners (new instance per job, terminated after)

  Risk 4: Runner has broad IAM permissions
    Runner EC2 instance has AWS IAM role
    Any job that runs can call AWS APIs with that role
    Fix: minimal IAM role on runner (only permissions needed for CI)
         use OIDC to give per-workflow IAM permissions instead

Auto-scaling self-hosted runners on AWS:

  Option A: Actions Runner Controller (ARC) on Kubernetes:
    Kubernetes operator that manages runner pods
    Scale: 0 pods when idle, N pods when jobs queued
    kubectl apply -f https://github.com/actions/actions-runner-controller/...

    # Runner scale set (auto-scales based on job queue):
    apiVersion: actions.github.com/v1alpha1
    kind: AutoscalingRunnerSet
    spec:
      minRunners: 0
      maxRunners: 20
      runnerGroup: "k8s-runners"

  Option B: EC2 Auto Scaling Group + Lambda trigger:
    GitHub webhook → Lambda → check queue depth → scale ASG
    ASG launches EC2 → runner registers → jobs run → EC2 terminates

    # User data script on EC2:
    #!/bin/bash
    cd /home/ubuntu
    ./config.sh --url https://github.com/company --token $RUNNER_TOKEN \
      --labels ec2,linux --ephemeral    # --ephemeral: deregister after one job
    ./run.sh
    # After job completes, instance terminates via lifecycle hook

  Cost comparison:
    GitHub-hosted (ubuntu): $0.008/min = $0.48/hr
    EC2 c5.2xlarge (spot):  $0.068/hr
    For builds > 8 minutes: self-hosted EC2 spot = cheaper
    For sporadic builds: GitHub-hosted simpler (no EC2 management)
```

💡 **Interview tip:** The security risk of self-hosted runners on public repositories is the most important point — a fork PR can trigger workflows that execute arbitrary code on your runner with access to your VPC and secrets. The ephemeral runner pattern (new instance per job, terminated after) mitigates the persistent-state risk and is the recommended production pattern. Actions Runner Controller (ARC) on Kubernetes is the cleanest modern solution — it manages runner pods automatically and scales to zero when idle, eliminating idle costs.

---

### Q150 — AWS | Troubleshooting | Advanced

> **ECS service tasks repeatedly cycling between RUNNING and STOPPED**. Never reaches desired count of 3. Complete troubleshooting.

📁 **Reference:** `nawab312/AWS` — ECS, task definitions, CloudWatch Logs, health checks

```
Step 1 — Check stopped task reason:
  aws ecs describe-tasks \
    --cluster my-cluster \
    --tasks $(aws ecs list-tasks --cluster my-cluster \
              --desired-status STOPPED --query 'taskArns[0]' --output text)

  stoppedReason field shows:
  "Essential container in task exited"
  → Your container is crashing (exit code non-zero)

  "Task failed ELB health checks in (elb my-alb)"
  → Container is running but health check failing

  "Cannotpullcontainer: ... No basic auth credentials"
  → Cannot pull image from ECR — permissions issue

  "ResourceInitializationError: unable to pull secrets"
  → Secrets Manager/Parameter Store access issue

Step 2 — Check container logs:
  # CloudWatch Logs → /ecs/<cluster>/<service>
  aws logs get-log-events \
    --log-group-name /ecs/my-service \
    --log-stream-name ecs/app/<task-id>

  Common log errors:
  → "Connection refused" to RDS → security group or wrong endpoint
  → "OOMKilled" → container exceeding memory limit
  → "Error: Cannot find module '/app/server.js'" → wrong image/path

Step 3 — Check exit code:
  aws ecs describe-tasks ... | jq '.tasks[].containers[].exitCode'
  → exit code 1  → application error (check logs)
  → exit code 137 → OOM killed (increase memory in task definition)
  → exit code 139 → segfault
  → exit code 255 → application crashed at startup

Step 4 — Check task definition resource limits:
  # Container might be getting OOM killed:
  aws ecs describe-task-definition --task-definition my-service
  # Check: memory: 512 → might be too low for JVM apps
  # JVM default heap + overhead often > 512MB
  # Fix: increase to 1024 or 2048MB

Step 5 — Check health check configuration:
  # ECS health check (separate from ALB health check):
  healthCheck:
    command: ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
    interval: 30
    timeout: 5
    retries: 3
    startPeriod: 60   # IMPORTANT: give app time to start before health checks

  # Without startPeriod: health check fires immediately
  # App needs 45 seconds to start → health check fails 3 times → task killed

Step 6 — Check ALB target group health check:
  aws elbv2 describe-target-health \
    --target-group-arn arn:aws:elasticloadbalancing:...

  # If all targets show unhealthy:
  # Check: is the health check path correct? /health vs /healthz vs /
  # Check: is the port correct? (8080 vs 443 vs 80)
  # Check: does the SG allow ALB → container port?

Step 7 — Check VPC/Security group connectivity:
  # Task cannot reach RDS → crashes on startup:
  # ECS task SG → allows outbound → RDS SG port 5432?
  aws ec2 describe-security-groups --group-ids <ecs-task-sg>
  aws ec2 describe-security-groups --group-ids <rds-sg>
  # RDS SG must have inbound rule: from ECS task SG on port 5432

Step 8 — Check IAM task role:
  # Task might need AWS API access (S3, Secrets Manager, etc.):
  aws ecs describe-task-definition --task-definition my-service \
    | jq '.taskDefinition.taskRoleArn'
  # If null → task has no IAM permissions
  # If set → check the role has required permissions
```

💡 **Interview tip:** The `stoppedReason` field is the first thing to check — it categorically tells you why the task stopped without any guessing. The most common cause is a missing `startPeriod` in the health check — without it, ECS starts checking health immediately, and an app that takes 30-60 seconds to initialize fails all health checks and gets terminated before it can start serving traffic. The second most common is OOM kill (exit code 137) — JVM applications especially need much more memory than the default task definition settings.

---

### Q151 — Prometheus | Conceptual | Advanced

> Explain **`remote_write`** and **`remote_read`**. Explain **Prometheus TSDB** internals — WAL, chunks, blocks. What happens when disk fills up?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`

```
remote_write — send metrics to external storage:
  Prometheus → (HTTP POST batches) → remote storage backend
  Backends: Thanos Receiver, Cortex, Mimir, InfluxDB, VictoriaMetrics

  prometheus.yml:
  remote_write:
  - url: "http://thanos-receive:19291/api/v1/receive"
    queue_config:
      max_samples_per_send: 10000
      batch_send_deadline: 5s
      max_shards: 200       # parallel write workers
    write_relabel_configs:  # filter which metrics to send
    - source_labels: [__name__]
      regex: "critical_.*"
      action: keep          # only send metrics starting with critical_

  Use case: long-term retention in Thanos while keeping local TSDB for fast recent queries

remote_read — query external storage transparently:
  PromQL query → Prometheus checks local TSDB + queries remote_read backend
  User gets unified result (no distinction between local and remote data)

  remote_read:
  - url: "http://thanos-query:10902/api/v1/read"
    read_recent: false   # only use remote for data older than local retention

TSDB internals — how Prometheus stores data on disk:

  data/
  ├── chunks_head/           ← current active data (in memory + mapped to disk)
  ├── 01ABC123.../           ← immutable block (2 hours of data)
  │   ├── chunks/            ← compressed time series data
  │   ├── index              ← label index for fast lookup
  │   ├── meta.json          ← block metadata (min/max time, stats)
  │   └── tombstones         ← marks deleted series
  └── wal/                   ← Write-Ahead Log
      ├── 00000001           ← WAL segment files
      └── 00000002

  Write-Ahead Log (WAL):
    Every new sample written to WAL FIRST (disk durability)
    WAL is replayed on crash recovery → no data loss
    WAL is flushed to chunks every 2 hours (configurable)
    WAL segment size: 128MB each, keeps last 3 segments

  Head block (in-memory + mmap):
    Most recent 2-4 hours in memory (fast reads/writes)
    Backed by memory-mapped files in chunks_head/
    Series indexed in memory for instant label lookup

  Immutable blocks (on disk):
    Every 2 hours: head block sealed → written as immutable block
    Block contains: all series data + full index
    Compaction: blocks merged over time (2h → 6h → 24h → 48h...)
    Benefits: compression improves, fewer files to scan

  What happens when disk fills up:
    Phase 1: Compaction fails (cannot write new blocks)
             prometheus_tsdb_compactions_failed_total increases
    Phase 2: WAL cannot be written → Prometheus logs errors
    Phase 3: Prometheus cannot accept new samples
             Returns: "out of space" → scrapers start getting errors
    Phase 4: Prometheus may crash (depending on version)

    Fix when disk is full:
    # Emergency: delete old blocks (data loss):
    ls -la /prometheus/data/
    rm -rf /prometheus/data/01OLDEST_BLOCK_ID

    # Proper fix: add storage, reduce retention:
    --storage.tsdb.retention.time=15d   # reduce from 30d to 15d
    --storage.tsdb.retention.size=10GB  # size-based retention limit
```

💡 **Interview tip:** The WAL (Write-Ahead Log) is the key to understanding Prometheus crash recovery — since every sample goes to WAL before being processed, a crashed Prometheus replays the WAL on restart and recovers all recent data. The two-hour block cycle is important: data stays in the "head" block in memory for the current chunk period, then gets sealed into an immutable on-disk block — this is why Prometheus memory usage is relatively predictable (proportional to active series, not total data). For disk full scenarios, `--storage.tsdb.retention.size` is the safer limit — it prevents disk exhaustion automatically.

---

### Q152 — Kubernetes | Conceptual | Advanced

> Explain **Taints and Tolerations** — 3 taint effects. Real scenario for `NoExecute` and what happens to existing pods.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

```
Taints — applied to NODES (repel pods):
  kubectl taint nodes node1 key=value:Effect
  Purpose: prevent pods from being scheduled on a node UNLESS they tolerate the taint

Tolerations — applied to PODS (allow scheduling on tainted nodes):
  spec:
    tolerations:
    - key: "key"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"

3 Taint Effects:

  NoSchedule:
    → New pods without matching toleration are NOT scheduled on this node
    → Existing pods on this node are NOT affected (keep running)
    → Use for: dedicated nodes, node maintenance preparation

  PreferNoSchedule:
    → Scheduler TRIES to avoid placing pods without toleration on this node
    → But WILL place them if no other nodes are available
    → Soft version of NoSchedule
    → Use for: soft preference (prefer other nodes but not strict)

  NoExecute:
    → New pods without toleration are NOT scheduled (same as NoSchedule)
    → EXISTING pods without matching toleration ARE EVICTED
    → This is the ONLY effect that impacts running pods
    → With tolerationSeconds: pod can stay for N seconds before eviction

Real scenario — node hardware failure:
  Node controller detects node unreachable (NotReady for >5 minutes)
  → Automatically adds NoExecute taint: node.kubernetes.io/unreachable
  → All pods WITHOUT explicit toleration are evicted after tolerationSeconds=300 (5 minutes)
  → ReplicaSet controller sees pods missing → creates replacements on healthy nodes
  → This is the automatic self-healing mechanism!

  Pods with toleration (DaemonSets):
    tolerations:
    - key: "node.kubernetes.io/unreachable"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 0    # DaemonSet pods evicted immediately (then recreated)
    # OR:
    - operator: "Exists"      # tolerate ALL taints (DaemonSet default)

Manual NoExecute for controlled maintenance:
  # Drain a node for maintenance:
  kubectl taint nodes node1 maintenance=true:NoExecute
  # All pods evicted → rescheduled on other nodes
  # Remove when maintenance done:
  kubectl taint nodes node1 maintenance=true:NoExecute-

  kubectl cordon vs kubectl drain vs taint:
    kubectl cordon node1:   adds NoSchedule taint (no new pods, existing stay)
    kubectl drain node1:    evicts all pods + cordons (NoExecute effect + delete)
    kubectl taint + NoExecute: same as drain but more granular control

Dedicated nodes pattern (GPU, high-memory):
  # Taint GPU nodes:
  kubectl taint nodes gpu-node gpu=true:NoSchedule

  # Only ML jobs can run here:
  spec:
    tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
    nodeSelector:
      gpu: "true"

  # Regular pods don't tolerate → can't schedule on GPU nodes
  # GPU nodes reserved exclusively for ML workloads
```

💡 **Interview tip:** `NoExecute` is the only taint effect that impacts running pods — this is the most important distinction to get right. The node controller automatically applies `node.kubernetes.io/unreachable:NoExecute` when a node goes NotReady, which is the mechanism behind Kubernetes self-healing. The `tolerationSeconds` on that automatic taint is 300 seconds (5 minutes) by default — this is why pods take about 5 minutes to be evicted after a node failure, which directly matches the timeline in the Node NotReady troubleshooting scenario.

---

### Q153 — ArgoCD | Conceptual | Advanced

> Explain **ArgoCD RBAC**. Difference between **Projects** and **Applications**. How Projects provide multi-tenancy.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD`

```
ArgoCD RBAC:
  Based on Casbin policy engine (not Kubernetes RBAC)
  Two built-in roles:
    role:admin    → full access to everything
    role:readonly → read-only access (view all apps, no changes)

  Custom roles in argocd-rbac-cm ConfigMap:
  p, role:dev-team, applications, sync, production/*, allow
  p, role:dev-team, applications, get, production/*, allow
  p, role:dev-team, applications, create, staging/*, allow
  p, role:dev-team, applications, delete, staging/*, allow

  Format: p, subject, resource, action, object, effect
    subject: role or user
    resource: applications, clusters, repositories, projects
    action:   get, create, update, delete, sync, override, action
    object:   <project>/<app-name> or * for all

  Assign role to SSO group:
  g, "github-org:dev-team", role:dev-team
  g, "github-org:platform", role:admin

ArgoCD Application:
  → Represents ONE deployment (one Git repo path → one K8s cluster/namespace)
  → Defines: source (Git repo + path + revision) + destination (cluster + namespace)
  → Owns the sync state for that deployment

ArgoCD Project:
  → Groups multiple Applications
  → Enforces boundaries: which repos, clusters, and namespaces are allowed
  → Multi-tenancy: each team has their own Project with restricted scope

Project provides multi-tenancy:

  Platform team creates project per team:
  apiVersion: argoproj.io/v1alpha1
  kind: AppProject
  metadata:
    name: team-payments
    namespace: argocd
  spec:
    description: "Payments team — production deployments"

    # Which Git repos this team can deploy FROM:
    sourceRepos:
    - "https://github.com/company/payments-service"
    - "https://github.com/company/payments-config"
    # NOT: "https://github.com/company/infra" (platform team only)

    # Which clusters and namespaces this team can deploy TO:
    destinations:
    - server: "https://kubernetes.default.svc"
      namespace: "payments-prod"
    - server: "https://kubernetes.default.svc"
      namespace: "payments-staging"
    # NOT: namespace "kube-system" or "monitoring" (locked to platform)

    # Which K8s resources this team can manage:
    namespaceResourceWhitelist:
    - group: "apps"
      kind: Deployment
    - group: ""
      kind: Service
    # NOT: ClusterRole, ClusterRoleBinding (admin-only)

    # Deny elevated permissions in deployed manifests:
    namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota   # only platform can set quotas

    # Which roles can manage applications in this project:
    roles:
    - name: deploy
      policies:
      - p, proj:team-payments:deploy, applications, sync, team-payments/*, allow
      - p, proj:team-payments:deploy, applications, get, team-payments/*, allow

  Result: payments team can only deploy their own repos to their own namespaces
          cannot touch monitoring, infra, or other teams' namespaces
          cannot deploy ClusterRoles or escalate privileges
```

💡 **Interview tip:** AppProject is the ArgoCD equivalent of Kubernetes RBAC + NetworkPolicy for deployments — it restricts what teams can deploy where. Without Projects, any ArgoCD user with application-create permission could deploy anything to any namespace. The three key restrictions in a Project are: `sourceRepos` (which Git repos can be used), `destinations` (which clusters and namespaces are allowed), and `namespaceResourceWhitelist/Blacklist` (which Kubernetes resource types can be deployed). This is how a platform team safely gives 10 product teams self-service deployment access without risking cross-team interference.

---

### Q154 — Linux / Bash | Scenario-Based | Advanced

> Write a **production-grade log rotation script** — compress logs > 7 days, delete > 30 days, use copytruncate, weekly email summary, proper error handling.

📁 **Reference:** `nawab312/DSA` → `Linux`

```bash
#!/bin/bash
set -euo pipefail    # exit on error, undefined vars, pipe failures
IFS=$'\n\t'

# Configuration
LOG_DIR="${LOG_DIR:-/var/log/myapp}"
COMPRESS_AGE_DAYS=7
DELETE_AGE_DAYS=30
REPORT_EMAIL="${REPORT_EMAIL:-ops@company.com}"
SCRIPT_NAME=$(basename "$0")
LOCK_FILE="/var/run/${SCRIPT_NAME}.lock"
SUMMARY_FILE=$(mktemp)

# Logging
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$SCRIPT_NAME] $*" >&2; }
die() { log "ERROR: $*"; exit 1; }

# Cleanup on exit
cleanup() {
  rm -f "$LOCK_FILE" "$SUMMARY_FILE"
}
trap cleanup EXIT
trap 'die "Interrupted"' INT TERM

# Prevent concurrent runs
if [ -e "$LOCK_FILE" ]; then
  die "Already running (lock: $LOCK_FILE). PID: $(cat $LOCK_FILE)"
fi
echo $$ > "$LOCK_FILE"

# Validate
[ -d "$LOG_DIR" ] || die "Log directory not found: $LOG_DIR"
command -v gzip   >/dev/null || die "gzip not installed"
command -v mail   >/dev/null || log "WARNING: mail not installed — email disabled"

log "Starting log rotation for: $LOG_DIR"

COMPRESSED_COUNT=0
COMPRESSED_BYTES_SAVED=0
DELETED_COUNT=0
DELETED_BYTES=0
ERRORS=0

# Step 1 — Compress logs older than 7 days (copytruncate: safe for running apps)
log "Compressing logs older than ${COMPRESS_AGE_DAYS} days..."
while IFS= read -r -d '' file; do
  [ -f "$file" ] || continue
  [ "${file%.gz}" = "$file" ] || continue  # skip already compressed

  original_size=$(stat -c %s "$file" 2>/dev/null || echo 0)

  # copytruncate: copy file, then truncate original (app keeps writing to same inode)
  if cp "$file" "${file}.tmp" && gzip -f "${file}.tmp" && mv "${file}.tmp.gz" "${file}.gz"; then
    # Truncate original (app continues writing without reopening)
    : > "$file"
    saved=$((original_size - $(stat -c %s "${file}.gz" 2>/dev/null || echo 0)))
    COMPRESSED_BYTES_SAVED=$((COMPRESSED_BYTES_SAVED + saved))
    COMPRESSED_COUNT=$((COMPRESSED_COUNT + 1))
    log "Compressed: $file (saved $(numfmt --to=iec $saved 2>/dev/null || echo "${saved}B"))"
  else
    log "ERROR: Failed to compress $file"
    ERRORS=$((ERRORS + 1))
    rm -f "${file}.tmp" "${file}.tmp.gz"
  fi
done < <(find "$LOG_DIR" -name "*.log" -mtime +"$COMPRESS_AGE_DAYS" -print0)

# Step 2 — Delete compressed logs older than 30 days
log "Deleting logs older than ${DELETE_AGE_DAYS} days..."
while IFS= read -r -d '' file; do
  file_size=$(stat -c %s "$file" 2>/dev/null || echo 0)
  if rm -f "$file"; then
    DELETED_BYTES=$((DELETED_BYTES + file_size))
    DELETED_COUNT=$((DELETED_COUNT + 1))
    log "Deleted: $file"
  else
    log "ERROR: Failed to delete $file"
    ERRORS=$((ERRORS + 1))
  fi
done < <(find "$LOG_DIR" -name "*.gz" -mtime +"$DELETE_AGE_DAYS" -print0)

# Step 3 — Build summary report
cat > "$SUMMARY_FILE" << EOF
Weekly Log Rotation Summary
============================
Date:        $(date '+%Y-%m-%d %H:%M:%S')
Host:        $(hostname)
Log dir:     $LOG_DIR

Compressed:  $COMPRESSED_COUNT files
Space saved: $(numfmt --to=iec $COMPRESSED_BYTES_SAVED 2>/dev/null || echo "${COMPRESSED_BYTES_SAVED}B")

Deleted:     $DELETED_COUNT files
Space freed: $(numfmt --to=iec $DELETED_BYTES 2>/dev/null || echo "${DELETED_BYTES}B")

Errors:      $ERRORS
Current usage: $(du -sh "$LOG_DIR" | cut -f1)
EOF

cat "$SUMMARY_FILE"

# Step 4 — Send email if mail is available
if command -v mail >/dev/null && [ -n "$REPORT_EMAIL" ]; then
  SUBJECT="Log Rotation Summary — $(hostname) — $(date '+%Y-%m-%d')"
  if [ $ERRORS -gt 0 ]; then
    SUBJECT="[ERRORS] $SUBJECT"
  fi
  mail -s "$SUBJECT" "$REPORT_EMAIL" < "$SUMMARY_FILE" \
    && log "Summary emailed to: $REPORT_EMAIL" \
    || log "WARNING: Failed to send email"
fi

# Exit with error if any operations failed
[ $ERRORS -eq 0 ] || { log "Completed with $ERRORS errors"; exit 1; }
log "Log rotation completed successfully"
```

```
# Crontab entry (runs every Sunday at 2 AM):
0 2 * * 0 REPORT_EMAIL=ops@company.com /opt/scripts/rotate-logs.sh >> /var/log/rotate-logs.log 2>&1

# Or weekly email only (suppress daily output):
0 2 * * 0 /opt/scripts/rotate-logs.sh 2>&1 | grep -E "(ERROR|Summary|Compressed|Deleted)"
```

💡 **Interview tip:** The `copytruncate` approach (copy → gzip → truncate original) is critical for rotating logs of running processes — if you move or delete the original file, the running application still has an open file descriptor to the old inode and keeps writing there (the "disappeared" file). By truncating the original in-place, the application continues writing to the same inode without any interruption. The lock file pattern prevents two cron jobs from running simultaneously if the previous one is still running. `set -euo pipefail` is the production bash safety setting — exit on any error, treat undefined variables as errors, and propagate pipe failures.

---

### Q155 — AWS | Scenario-Based | Advanced

> Systematic approach to **reduce AWS bill by 30%** from $50,000/month. 10+ concrete actions across EC2, RDS, S3, and data transfer.

📁 **Reference:** `nawab312/AWS` — Cost Explorer, Reserved Instances, Savings Plans

```
Step 1 — Audit with Cost Explorer:
  Enable: Cost Explorer → monthly view by service + linked account
  Filter: last 3 months → find top 5 spending services
  Typical: EC2 + RDS + Data Transfer = 70-80% of total bill

Step 2 — EC2 cost actions:

  Action 1: Right-size over-provisioned instances (often 15-20% savings)
    aws compute-optimizer get-ec2-instance-recommendations
    → Shows: current type, recommended type, estimated savings
    → m5.2xlarge at 10% avg CPU → downsize to m5.large
    → Run for 2 weeks to gather data before resizing production

  Action 2: Purchase Savings Plans (30-40% discount vs on-demand)
    Cost Explorer → Savings Plans → "Purchase Recommendations"
    → Compute Savings Plan: flexible (any EC2/Fargate/Lambda)
    → EC2 Instance Savings Plan: locked to family/region (deeper discount)
    → Commit: 1-year (up to 40% off) or 3-year (up to 66% off)
    → Buy for BASELINE usage only — not peak

  Action 3: Spot instances for fault-tolerant workloads (70-90% savings)
    → CI/CD build agents: perfect for spot (job reruns on interruption)
    → Batch jobs, ML training: can checkpoint and resume
    → Dev/staging environments: acceptable to lose occasionally
    → Use Spot Fleet or ASG with mixed instances policy

  Action 4: Terminate idle instances
    CloudWatch: CPUUtilization < 5% for 14 days → candidate for termination
    → Create "idle EC2" alarm on all instances
    → Review monthly: terminate or downsize

  Action 5: Scheduled scaling (scale down outside business hours)
    Dev/staging: scale ASG to 0 instances at 8 PM, scale up at 8 AM
    aws autoscaling put-scheduled-update-group-action \
      --auto-scaling-group-name dev-asg \
      --scheduled-action-name scale-down-night \
      --recurrence "0 20 * * *" --desired-capacity 0 --min-size 0

Step 3 — RDS cost actions:

  Action 6: Reserved Instances for RDS (30-40% savings)
    Same principle as EC2 — commit 1 or 3 years for stable databases

  Action 7: Resize or use Aurora Serverless v2 for variable workloads
    Aurora Serverless v2: 0.5 ACU minimum (scales to 0.5 when idle)
    vs db.r5.large always running at $0.25/hour = $180/month
    Dev RDS: $30/month with Aurora Serverless v2

  Action 8: Stop dev/staging RDS instances on nights/weekends
    aws rds stop-db-instance --db-instance-identifier dev-db
    # Stops billing for compute (still billed for storage)
    # Max stop period: 7 days (auto-starts after 7 days)
    # Use Lambda + EventBridge to automate stop/start

Step 4 — S3 cost actions:

  Action 9: S3 Intelligent-Tiering for unpredictable access patterns
    Moves objects between tiers automatically based on access:
    Frequent (>30 days) → Infrequent → Archive Instant
    Zero retrieval fees, small monitoring charge ($0.0025/1000 objects)

  Action 10: S3 Lifecycle policies for old data
    Objects > 90 days → S3 Standard-IA (40% cheaper)
    Objects > 180 days → S3 Glacier Instant (68% cheaper)
    Objects > 1 year → S3 Glacier Deep Archive (95% cheaper)

Step 5 — Data transfer cost actions:

  Action 11: Use S3 Transfer Acceleration only when needed
    Default S3 uploads don't use Transfer Acceleration
    But if enabled globally → extra cost even for same-region uploads

  Action 12: VPC endpoints for S3 and DynamoDB
    EC2 → S3 via internet gateway → $0.09/GB data transfer cost
    EC2 → S3 via VPC Gateway Endpoint → $0.00 (free)
    aws ec2 create-vpc-endpoint --vpc-id vpc-xxx \
      --service-name com.amazonaws.us-east-1.s3

  Action 13: Same-region vs cross-region transfers
    Audit: are services in same region talking to each other?
    Cross-region data transfer: $0.02/GB
    Same-region: free (within same AZ to AZ: $0.01/GB)
    Consolidate services to same region/AZ where possible

Estimated savings from all actions:
  Right-sizing:        $3,000/month (6%)
  Savings Plans:       $8,000/month (16%)
  Spot instances:      $3,000/month (6%)
  RDS optimization:    $2,000/month (4%)
  S3 tiering:         $1,500/month (3%)
  VPC endpoints:       $500/month (1%)
  Total:              $18,000/month (36%) → exceeds 30% target
```

💡 **Interview tip:** Savings Plans are almost always the highest ROI action for stable workloads — committing to 1 year of compute at your baseline usage level saves 30-40% with very low risk. The key is buying at baseline, not at peak — you'll use on-demand for spikes and still save money overall. Right-sizing EC2 instances is high-visibility and safe — AWS Compute Optimizer does the analysis automatically. Stopping dev/staging RDS instances at night is quick wins with zero architectural changes. For the interview, show systematic thinking: audit first (Cost Explorer), then tackle biggest spenders, then automate prevention.

---

### Q156 — Kubernetes | Troubleshooting | Advanced

> **etcd WAL fsync latency p99 > 10ms** causing API server slowness. Causes, diagnosis, fixes.

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md`, `09_CLUSTER_OPERATIONS.md`

```
Why etcd fsync latency matters:
  Every write to etcd goes through WAL (Write-Ahead Log)
  WAL requires fsync to disk before acknowledging write
  High fsync latency = slow writes = slow API server responses
  p99 > 10ms: etcd best practice violation
  p99 > 25ms: etcd will start dropping leadership (very bad)

Step 1 — Confirm the metric:
  curl http://localhost:2381/metrics | grep wal_fsync
  etcd_disk_wal_fsync_duration_seconds_bucket → check p99 value

  # From Prometheus:
  histogram_quantile(0.99,
    rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))
  # Should be < 0.010 (10ms)

Step 2 — Check disk type and performance:
  # Check what disk etcd is on:
  df -h /var/lib/etcd
  lsblk -d -o NAME,ROTA    # ROTA=1 = spinning disk, ROTA=0 = SSD

  # Test raw disk performance:
  fio --name=test --filename=/var/lib/etcd/fio-test \
    --rw=write --bs=4k --size=100M --fsync=1 --fdatasync=1 \
    --ioengine=sync --iodepth=1 --numjobs=1
  # Expected on SSD: < 500 microseconds per fsync
  # Expected on HDD: 5-15 milliseconds per fsync (too slow!)

Step 3 — Check disk I/O contention:
  iostat -x 1 5
  # Check: %util for the etcd disk
  # If %util > 50%: disk is contended
  # Is another process sharing the etcd disk?

  # CRITICAL: etcd must NEVER share disk with:
  # - application data
  # - container image layers (containerd data)
  # - Kubernetes logs
  # On cloud: etcd should have a dedicated EBS volume (gp3 or io1)

Step 4 — Check for noisy neighbour (cloud):
  # AWS: check EBS CloudWatch metrics:
  aws cloudwatch get-metric-statistics \
    --namespace AWS/EBS \
    --metric-name VolumeWriteOps
  # High VolumeQueueLength → disk is saturated

  # Fix: increase EBS IOPS:
  aws ec2 modify-volume \
    --volume-id vol-etcd \
    --volume-type gp3 \
    --iops 3000 \        # gp3 default is 3000 IOPS (free)
    --throughput 125     # or increase to 250 MB/s

  # For very high throughput: io1/io2:
  # io2: up to 64,000 IOPS, 1,000 MB/s throughput

Step 5 — Check for etcd DB size:
  etcdctl endpoint status --write-out=table
  # DB SIZE column > 5GB → large DB causes slower compaction = more I/O

  # Compact and defrag:
  REVISION=$(etcdctl endpoint status --write-out=json | jq '.[0].Status.header.revision')
  etcdctl compact $REVISION
  etcdctl defrag --endpoints=https://etcd-0:2379  # defrag followers first

Step 6 — Check Linux I/O scheduler:
  cat /sys/block/<disk>/queue/scheduler
  # For SSDs: should be "none" or "mq-deadline" (not "cfq")
  echo "none" > /sys/block/nvme0n1/queue/scheduler

Step 7 — etcd tuning for slower disks (short-term workaround):
  # Increase heartbeat and election timeouts:
  --heartbeat-interval=200      # default: 100ms (increase for slow disk)
  --election-timeout=2000       # default: 1000ms (increase for slow disk)
  # This prevents false leader elections due to missed heartbeats
  # But does NOT fix the underlying disk problem
```

💡 **Interview tip:** The most important production rule about etcd: it must run on dedicated SSDs — never on the same disk as container images, logs, or application data. On AWS, a dedicated gp3 EBS volume with 3000 IOPS (the default, free) is usually sufficient for etcd. For very large clusters, io2 with 10,000+ IOPS may be needed. The `fio` benchmark with `--fsync=1` is the definitive test — it measures exactly what etcd measures (sequential writes with fsync). If fio shows > 5ms average fsync, the disk is too slow for etcd regardless of what type it claims to be.

---

### Q157 — Grafana | Scenario-Based | Advanced

> Design an **alerting strategy** with P1/P2/P3 severity levels — PagerDuty for P1, Slack for P2, Jira for P3, with deduplication.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana`, `Prometheus`

```
Severity definitions:
  P1 (Critical): service down, data loss, security breach
     Response: wake someone up NOW, fix within 30 minutes
     Examples: payment service 0 replicas, production database down,
               SSL certificate expired, 100% error rate

  P2 (High): degraded performance, partial outage, SLO at risk
     Response: fix within 4 hours (business hours or on-call)
     Examples: high error rate (>5%), p99 latency > SLO,
               disk > 85%, node NotReady

  P3 (Low): warning, action needed but not urgent
     Response: fix next business day, create ticket
     Examples: disk > 70%, certificate expiring in 30 days,
               deprecated API usage, cost anomaly

Architecture:

  Prometheus → AlertManager → routing → PagerDuty / Slack / Jira

AlertManager configuration:

  global:
    resolve_timeout: 5m

  route:
    receiver: default-slack
    group_by: ['alertname', 'cluster', 'service']
    group_wait: 30s          # collect alerts for 30s before first notification
    group_interval: 5m       # wait before re-notifying about same group
    repeat_interval: 4h      # re-alert if still firing after 4 hours

    routes:
    # P1: immediate PagerDuty call
    - matchers:
      - name: severity
        value: p1
      receiver: pagerduty-critical
      group_wait: 10s          # faster for P1
      repeat_interval: 30m     # re-page every 30 minutes

    # P2: Slack with follow-up
    - matchers:
      - name: severity
        value: p2
      receiver: slack-high
      repeat_interval: 2h

    # P3: Jira ticket
    - matchers:
      - name: severity
        value: p3
      receiver: jira-low
      repeat_interval: 24h     # remind once per day

  receivers:
  - name: pagerduty-critical
    pagerduty_configs:
    - routing_key: $PAGERDUTY_KEY
      severity: critical
      description: "{{ .CommonAnnotations.summary }}"
      details:
        service: "{{ .CommonLabels.service }}"
        runbook: "{{ .CommonAnnotations.runbook_url }}"

  - name: slack-high
    slack_configs:
    - api_url: $SLACK_WEBHOOK_P2
      channel: '#oncall-p2'
      title: ':warning: P2 Alert: {{ .CommonAnnotations.summary }}'
      text: |
        *Service:* {{ .CommonLabels.service }}
        *Namespace:* {{ .CommonLabels.namespace }}
        *Description:* {{ .CommonAnnotations.description }}
        *Runbook:* {{ .CommonAnnotations.runbook_url }}
      send_resolved: true      # notify when alert resolves

  - name: jira-low
    webhook_configs:
    - url: "https://jira.company.com/api/v2/issue"
      http_config:
        bearer_token: $JIRA_TOKEN
      send_resolved: false     # don't close ticket automatically

Deduplication:
  AlertManager deduplication happens via group_by:
    group_by: ['alertname', 'cluster', 'service']
    → Multiple alerts for same alertname+cluster+service = ONE notification
    → Prevents 50 "NodeDiskHigh" alerts sending 50 pages

  Inhibition rules — silence lower severity when higher fires:
  inhibit_rules:
  - source_matchers:
    - name: severity
      value: p1
    target_matchers:
    - name: severity
      value: p2
    equal: ['service']
    # If P1 fires for payment-service, silence P2 for payment-service
    # Avoids: service is down (P1) PLUS high latency (P2) = double page

  Silence for maintenance windows:
    amtool silence add alertname=".*" --duration=2h \
      --comment="Planned maintenance 2024-01-15"
```

💡 **Interview tip:** The inhibition rule is the key deduplication mechanism that most people miss — when a service is completely down (P1), you don't want to also get paged for every P2 symptom of that same service (high error rate, slow response, etc.). The inhibition silences child symptoms when the parent cause is already firing. For Jira integration, use a webhook receiver pointing to the Jira API — AlertManager can create tickets directly without any middleware. Set `send_resolved: false` for P3 Jira tickets so the ticket stays open until a human closes it.

---

### Q158 — Terraform | Troubleshooting | Advanced

> After module refactoring, `terraform plan` shows **dozens of unexpected resource replacements**. Fix without destroying infrastructure.

📁 **Reference:** `nawab312/Terraform` — `moved` block, lifecycle, state

```
Common causes of unintended replacements:

Cause 1 — Resource renamed or moved to module:
  # Before:
  resource "aws_s3_bucket" "logs" { bucket = "company-logs" }

  # After refactor (moved to module):
  module "logging" {
    source = "./modules/s3"
  }

  Terraform sees: aws_s3_bucket.logs DESTROY + module.logging.aws_s3_bucket.this CREATE
  Fix — moved block tells Terraform about the rename:
  moved {
    from = aws_s3_bucket.logs
    to   = module.logging.aws_s3_bucket.this
  }
  → Plan now shows: resource moved (no destroy/create)

Cause 2 — for_each key changed:
  # Before:
  for_each = toset(["us-east-1", "us-west-2"])
  resource: aws_s3_bucket.this["us-east-1"]

  # After: key format changed
  for_each = { east = "us-east-1", west = "us-west-2" }
  resource: aws_s3_bucket.this["east"]  ← different key → replace!

  Fix — moved block for each renamed key:
  moved {
    from = aws_s3_bucket.this["us-east-1"]
    to   = aws_s3_bucket.this["east"]
  }

Cause 3 — Computed field causing replacement:
  Plan shows: # aws_db_instance.main must be replaced
  ~ availability_zone = "us-east-1a" → (known after apply)
  Forces replacement: [availability_zone]

  Fix — lifecycle ignore_changes:
  resource "aws_db_instance" "main" {
    lifecycle {
      ignore_changes = [availability_zone]  # don't replace if AZ changes
    }
  }

Cause 4 — Provider version upgrade changed defaults:
  # AWS provider 4.x vs 5.x: default values changed for some resources
  # Plan shows fields "changing" that were not in your config

  Fix — explicitly set the value in config to match current state:
  aws ec2 describe-instances ... | jq '.Reservations[].Instances[].EbsOptimized'
  → Add to resource: ebs_optimized = false  # make explicit, stop drifting

Cause 5 — Module version upgrade with breaking changes:
  # Module v2.0.0 renames a variable or changes resource naming convention
  # All resources using that module show replacement

  Fix — use moved blocks in the module itself, or pin to old version:
  module "api" {
    source  = "..."
    version = "~> 1.0"  # stay on 1.x until you're ready to migrate
  }

Cause 6 — terraform state mv was run inconsistently:
  # State file has old resource address, code has new address
  # Mismatch = Terraform wants to destroy old + create new

  Fix — align state with code using moved block OR terraform state mv:
  terraform state mv \
    'aws_security_group.old_name' \
    'aws_security_group.new_name'

General process for safe refactoring:
  1. terraform plan -out=plan.out       → see all changes
  2. terraform show plan.out | grep "must be replaced"  → find replacements
  3. For each unexpected replacement:
     a. Add moved block (if rename)
     b. Add lifecycle ignore_changes (if computed field)
     c. terraform state mv (if manual state fix needed)
  4. terraform plan -out=plan.out       → verify 0 replacements
  5. terraform apply plan.out
```

💡 **Interview tip:** The `moved` block is the safest tool for refactoring — it's tracked in Git, PR-reviewable, and idempotent (can be applied multiple times safely). After the apply succeeds, you can delete the moved blocks. `lifecycle { ignore_changes }` is the right fix for fields that are set by AWS at creation time and shouldn't trigger updates (availability zone, some ARNs). The key debugging command is `terraform plan | grep "must be replaced"` — this shows exactly which resources will be destroyed and recreated, which is your hit list to fix before applying.

---

### Q159 — Python | Scenario-Based | Advanced

> Write a **Python CI/CD pipeline status dashboard** — GitHub Actions API, last 10 runs, color-coded, success rate, CLI argument for repo.

📁 **Reference:** `nawab312/DSA` → `Python`

```python
#!/usr/bin/env python3

import argparse
import json
import os
import sys
import urllib.request
import urllib.error
from datetime import datetime, timezone


# ANSI color codes
GREEN  = "\033[92m"
RED    = "\033[91m"
YELLOW = "\033[93m"
BLUE   = "\033[94m"
RESET  = "\033[0m"
BOLD   = "\033[1m"


def colorize(text, color):
    if not sys.stdout.isatty():  # no colors when piped
        return text
    return f"{color}{text}{RESET}"


def github_api_request(endpoint, token=None):
    url = f"https://api.github.com{endpoint}"
    headers = {
        "Accept": "application/vnd.github.v3+json",
        "X-GitHub-Api-Version": "2022-11-28",
    }
    if token:
        headers["Authorization"] = f"Bearer {token}"

    req = urllib.request.Request(url, headers=headers)
    try:
        with urllib.request.urlopen(req, timeout=10) as resp:
            return json.loads(resp.read())
    except urllib.error.HTTPError as e:
        if e.code == 404:
            print(colorize(f"ERROR: Repository not found: {endpoint}", RED))
        elif e.code == 401:
            print(colorize("ERROR: Invalid GitHub token (set GITHUB_TOKEN env var)", RED))
        else:
            print(colorize(f"ERROR: GitHub API returned {e.code}: {e.reason}", RED))
        sys.exit(1)
    except urllib.error.URLError as e:
        print(colorize(f"ERROR: Network error: {e.reason}", RED))
        sys.exit(1)


def format_duration(seconds):
    if seconds is None:
        return "N/A"
    mins, secs = divmod(int(seconds), 60)
    return f"{mins}m{secs:02d}s" if mins else f"{secs}s"


def format_status(status, conclusion):
    if status == "in_progress":
        return colorize("● RUNNING", YELLOW)
    elif status == "queued":
        return colorize("○ QUEUED", BLUE)
    elif conclusion == "success":
        return colorize("✓ SUCCESS", GREEN)
    elif conclusion == "failure":
        return colorize("✗ FAILURE", RED)
    elif conclusion == "cancelled":
        return colorize("⊘ CANCELLED", YELLOW)
    elif conclusion == "skipped":
        return colorize("⊘ SKIPPED", BLUE)
    else:
        return colorize(f"? {(conclusion or status or 'unknown').upper()}", YELLOW)


def parse_duration(run):
    if not run.get("created_at") or not run.get("updated_at"):
        return None
    created = datetime.fromisoformat(run["created_at"].replace("Z", "+00:00"))
    updated = datetime.fromisoformat(run["updated_at"].replace("Z", "+00:00"))
    return (updated - created).total_seconds()


def main():
    parser = argparse.ArgumentParser(
        description="GitHub Actions pipeline status dashboard"
    )
    parser.add_argument(
        "repository",
        help="Repository in format: owner/repo (e.g. nawab312/myapp)"
    )
    parser.add_argument(
        "--runs", "-n",
        type=int,
        default=10,
        help="Number of recent runs to show (default: 10)"
    )
    parser.add_argument(
        "--workflow", "-w",
        help="Filter by workflow name (optional)"
    )
    args = parser.parse_args()

    token = os.environ.get("GITHUB_TOKEN")
    if not token:
        print(colorize("WARNING: GITHUB_TOKEN not set — API rate limit is 60/hr", YELLOW))

    repo = args.repository
    endpoint = f"/repos/{repo}/actions/runs?per_page={args.runs}"
    if args.workflow:
        endpoint += f"&workflow_id={args.workflow}"

    print(f"\n{BOLD}GitHub Actions Dashboard — {repo}{RESET}")
    print("─" * 80)

    data = github_api_request(endpoint, token)
    runs = data.get("workflow_runs", [])

    if not runs:
        print(colorize("No workflow runs found.", YELLOW))
        sys.exit(0)

    # Print header
    print(f"{'STATUS':<22} {'WORKFLOW':<28} {'BRANCH':<18} {'TRIGGERED BY':<16} {'DURATION':<10}")
    print("─" * 80)

    success_count = 0
    completed_count = 0

    for run in runs:
        status     = run.get("status", "unknown")
        conclusion = run.get("conclusion")
        workflow   = run.get("name", "unknown")[:26]
        branch     = run.get("head_branch", "unknown")[:16]
        actor      = run.get("triggering_actor", {}).get("login", "unknown")[:14]
        duration   = format_duration(parse_duration(run))
        status_str = format_status(status, conclusion)

        if status == "completed":
            completed_count += 1
            if conclusion == "success":
                success_count += 1

        print(f"{status_str:<31} {workflow:<28} {branch:<18} {actor:<16} {duration}")

    # Summary
    print("─" * 80)
    if completed_count > 0:
        success_rate = (success_count / completed_count) * 100
        rate_color = GREEN if success_rate >= 90 else YELLOW if success_rate >= 70 else RED
        rate_str = colorize(f"{success_rate:.0f}%", rate_color)
        print(f"Success rate: {rate_str} ({success_count}/{completed_count} completed runs)")
    else:
        print("No completed runs in this set.")
    print()


if __name__ == "__main__":
    main()
```

```
Usage:
  export GITHUB_TOKEN=ghp_xxxxx
  python dashboard.py nawab312/myapp
  python dashboard.py nawab312/myapp --runs 20
  python dashboard.py nawab312/myapp --workflow "CI"

Output:
  GitHub Actions Dashboard — nawab312/myapp
  ────────────────────────────────────────────────────────────────────────────────
  STATUS                 WORKFLOW                     BRANCH             TRIGGERED BY     DURATION
  ────────────────────────────────────────────────────────────────────────────────
  ✓ SUCCESS              CI Build and Test            main               nawab312         3m42s
  ✗ FAILURE              CI Build and Test            feature/auth       nawab312         1m15s
  ● RUNNING              Deploy Production            main               nawab312         0m45s
  ────────────────────────────────────────────────────────────────────────────────
  Success rate: 67% (2/3 completed runs)
```

💡 **Interview tip:** `sys.stdout.isatty()` is the professional touch — it disables ANSI color codes when output is piped to a file or another command, preventing escape codes from appearing as garbage in logs. The `urllib.request` usage without third-party libraries makes the script dependency-free and deployable anywhere. In production, you'd replace this with `requests` for better error handling and timeout control. The GitHub token check with a helpful warning (not an error) is good UX — the script works without a token but with reduced rate limits.

---

### Q160 — AWS | Conceptual | Advanced

> Explain **CloudFormation vs Terraform vs AWS CDK**. Key differences in state, provider support, language. When to choose CloudFormation?

📁 **Reference:** `nawab312/AWS`, `nawab312/Terraform`

```
CloudFormation:
  Language:      YAML or JSON templates (declarative)
  State:         AWS manages state internally (no state file to manage)
  Provider:      AWS only (no Azure, GCP, or third-party)
  Drift:         CloudFormation drift detection (compares template vs reality)
  Cost:          Free (pay only for resources created)
  Change sets:   Preview changes before applying (like terraform plan)
  Rollback:      Automatic rollback on stack failure (built-in)

  Strengths:
  → Tight AWS integration (some resources only in CloudFormation, not Terraform)
  → No state file to lose or corrupt (AWS manages it)
  → Automatic rollback: if resource fails, entire stack rolls back
  → AWS Service Catalog: share approved stacks across org
  → StackSets: deploy same stack to multiple accounts/regions

  Weaknesses:
  → AWS only — cannot manage non-AWS resources
  → Verbose YAML: 500-line templates for simple stacks
  → Slow: CloudFormation API calls are serialized (slower than Terraform parallel)
  → Limited logic: no loops or dynamic blocks without macros

Terraform:
  Language:      HCL (HashiCorp Configuration Language) — readable, concise
  State:         You manage state file (S3 + DynamoDB locking for teams)
  Provider:      1000+ providers — AWS, Azure, GCP, Kubernetes, GitHub, Datadog
  Cost:          Open source (free); Terraform Cloud ($20/user/month)

  Strengths:
  → Multi-cloud: manage AWS + Azure + GCP in same codebase
  → Rich language: loops, conditionals, dynamic blocks, functions
  → Large ecosystem: community modules for everything
  → Plan is separate from apply (safe preview)
  → State manipulation: terraform state mv, import, taint

  Weaknesses:
  → State file management: loses state = disaster
  → No automatic rollback: partial applies leave partial infrastructure
  → Provider version mismatches can break configurations

AWS CDK (Cloud Development Kit):
  Language:      TypeScript, Python, Java, Go, .NET (real programming languages)
  Underneath:    Generates CloudFormation templates (synthesizes to CFN)
  State:         Same as CloudFormation (AWS manages)
  Cost:          Free (open source)

  Strengths:
  → Full programming language: loops, classes, abstractions, unit testing
  → L3 Constructs: high-level abstractions (one line creates ECS service + ALB + IAM)
  → Type safety: IDE autocomplete, compile-time errors
  → Constructs Library: reusable patterns (like Terraform modules but typed)

  Weaknesses:
  → Compiles to CloudFormation: limited to CloudFormation capabilities
  → AWS only (no multi-cloud)
  → Larger learning curve (requires programming language knowledge)
  → Generated CFN templates are huge and hard to read

When to choose CloudFormation:
  ✓ AWS-only organization (no multi-cloud requirement)
  ✓ Need automatic rollback on deployment failure (critical for production)
  ✓ Compliance: audit trail in CloudFormation events (who deployed what)
  ✓ AWS Service Catalog: governance team publishes approved stacks
  ✓ StackSets: deploy same stack to 50 accounts with one command
  ✓ No state file risk (regulated industry — state file loss is unacceptable)

When to choose Terraform:
  ✓ Multi-cloud or non-AWS resources to manage (GitHub, Datadog, Kubernetes)
  ✓ Team already knows HCL
  ✓ Need Terraform module ecosystem
  ✓ Large team with module reuse patterns

When to choose CDK:
  ✓ Development team comfortable with TypeScript/Python
  ✓ Want high-level abstractions with type safety
  ✓ AWS-only but want programmatic logic (CDK > CloudFormation YAML)
  ✓ Want unit tests for infrastructure code

In practice: many teams use Terraform for infrastructure + CDK for application stacks
```

💡 **Interview tip:** The automatic rollback on failure is CloudFormation's unique advantage that Terraform cannot match — if deploying a 20-resource stack and resource 15 fails, CloudFormation rolls back all 19 previous resources automatically. In Terraform, you're left with a partially applied state. For regulated industries where audit trails are critical, CloudFormation's native AWS event history (stored in AWS, not in your state file) is more defensible than a Terraform state file. CDK is the answer for teams that want type safety and IDE support — it generates CloudFormation, combining the best of both worlds for AWS-only shops.

---

### Q161 — Kubernetes | Scenario-Based | Advanced

> Financial app — **pods must never be evicted**, run on **dedicated hardware**, **guaranteed CPU/memory**, startup must complete in **30 seconds or be killed**.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`, `07_SECURITY.md`

```yaml
# Step 1 — Taint dedicated nodes (hardware reserved for financial app)
# kubectl taint nodes financial-node-1 dedicated=financial:NoSchedule
# kubectl taint nodes financial-node-2 dedicated=financial:NoSchedule
# kubectl label nodes financial-node-1 node-type=financial

# Step 2 — PriorityClass (prevents eviction under node pressure)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: financial-critical
value: 2000000000        # max value — higher than system-node-critical (2000001000)
                         # Use 1000000 in practice to stay below system components
preemptionPolicy: PreemptLowerPriority
globalDefault: false
description: "Financial transaction processing — must not be evicted"

---
# Step 3 — Complete Deployment configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: financial-processor
  namespace: finance
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # never have fewer than desired replicas
      maxSurge: 1
  template:
    spec:
      # Dedicated hardware via toleration + nodeSelector
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "financial"
        effect: "NoSchedule"
      nodeSelector:
        node-type: financial   # must match node label

      # Guaranteed QoS — prevents eviction under memory pressure
      # Guaranteed QoS requires: requests == limits for ALL containers
      containers:
      - name: processor
        image: financial-processor:v2.1.0
        resources:
          requests:
            cpu: "2"             # exactly 2 CPUs
            memory: "4Gi"        # exactly 4GB
          limits:
            cpu: "2"             # MUST equal requests for Guaranteed QoS
            memory: "4Gi"        # MUST equal requests for Guaranteed QoS

        # Startup probe — kill if not ready in 30 seconds
        startupProbe:
          httpGet:
            path: /ready
            port: 8080
          failureThreshold: 6    # 6 × 5s = 30 seconds max startup time
          periodSeconds: 5
          initialDelaySeconds: 0

        # Liveness probe — only starts after startupProbe succeeds
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          periodSeconds: 10
          failureThreshold: 3

      # Guaranteed QoS requires: ALL containers (including init) have requests==limits
      initContainers:
      - name: db-migration
        image: financial-processor:v2.1.0
        command: ["./migrate"]
        resources:
          requests:
            cpu: "500m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"

      # Priority class — highest priority, evicted last
      priorityClassName: financial-critical

      # Grace period for in-flight transactions
      terminationGracePeriodSeconds: 120

      # Spread across dedicated nodes (anti-affinity)
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: financial-processor
            topologyKey: kubernetes.io/hostname

---
# Step 4 — PodDisruptionBudget (prevent voluntary disruptions)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: financial-pdb
  namespace: finance
spec:
  minAvailable: 2          # at least 2 pods running at all times
  selector:
    matchLabels:
      app: financial-processor
```

```
Why each configuration achieves the requirements:

  Never evicted:
    requests == limits → Guaranteed QoS class → evicted LAST (after BestEffort + Burstable)
    PriorityClass value: 1000000 → preempts lower priority pods, not preempted by them
    PodDisruptionBudget minAvailable: 2 → blocks node drains that would violate this

  Dedicated hardware:
    Taint on nodes: NoSchedule → regular pods cannot schedule here
    Toleration on pods → only financial pods tolerate the taint
    nodeSelector → additionally pins to labeled nodes (belt-and-suspenders)

  Guaranteed CPU/memory:
    requests == limits for ALL containers → Kubernetes sets Guaranteed QoS
    CPU: 2 cores always reserved, never throttled to less
    Memory: 4GB always reserved, OOM only kills if exceeds 4GB limit

  30-second startup kill:
    startupProbe failureThreshold: 6 × periodSeconds: 5 = 30 seconds
    If app not ready in 30 seconds → killed and restarted
    liveness/readiness disabled until startupProbe succeeds (no premature kill)
```

💡 **Interview tip:** Guaranteed QoS is achieved only when ALL containers (including init containers and sidecars) have `requests == limits` — one container without this setting drops the entire pod to Burstable QoS and makes it eligible for eviction. The `PodDisruptionBudget` is the often-forgotten piece that prevents voluntary disruptions (node drain, rolling node upgrades) from violating availability. The startup probe with `failureThreshold × periodSeconds = 30` is more flexible than a hard timeout — it gives the exact 30-second budget while allowing early success if the app starts faster.

---

### Q162 — ELK Stack | Conceptual | Advanced

> Explain **Elasticsearch query DSL vs KQL**. Write queries: ERROR logs last 24h, top 10 error messages aggregation, response time > 2s, multi-index search.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack`

```
KQL (Kibana Query Language):
  → Simple, human-readable syntax for Kibana search bar
  → No aggregations — filtering only
  → Examples:
    level: ERROR                           # field match
    level: ERROR AND service: payment      # boolean AND
    response_time > 2000                   # range
    message: "connection refused"          # phrase match
    NOT status: 200                        # negation

Query DSL (Elasticsearch Domain-Specific Language):
  → Full JSON API — filtering, aggregations, sorting, highlighting
  → More powerful but more verbose
  → Used for: dashboards, complex analytics, API-level queries

Query 1 — ERROR logs from last 24 hours:
  GET /logs-*/_search
  {
    "query": {
      "bool": {
        "must": [
          { "match": { "level": "ERROR" }},
          { "range": {
              "@timestamp": {
                "gte": "now-24h",
                "lte": "now"
              }
          }}
        ]
      }
    },
    "sort": [{ "@timestamp": { "order": "desc" }}],
    "size": 100
  }

Query 2 — Top 10 most common error messages (aggregation):
  GET /logs-*/_search
  {
    "size": 0,              # don't return raw hits, only aggregation
    "query": {
      "match": { "level": "ERROR" }
    },
    "aggs": {
      "top_errors": {
        "terms": {
          "field": "message.keyword",   # .keyword = exact match (not analyzed)
          "size": 10,
          "order": { "_count": "desc" }
        }
      }
    }
  }
  # Response shows: top 10 error messages with count per message

Query 3 — Response time greater than 2 seconds:
  GET /logs-*/_search
  {
    "query": {
      "range": {
        "response_time_ms": {
          "gt": 2000         # greater than 2000 milliseconds
        }
      }
    },
    "sort": [{ "response_time_ms": { "order": "desc" }}],
    "_source": ["@timestamp", "service", "endpoint", "response_time_ms", "user_id"],
    "size": 50
  }

Query 4 — Search across multiple indices with pattern:
  GET /logs-app-*,logs-nginx-*,logs-db-*/_search
  {
    "query": {
      "multi_match": {
        "query": "payment timeout",
        "fields": ["message", "error.message", "log.original"]
      }
    }
  }

  # OR using index alias (better practice):
  GET /all-logs/_search    # alias pointing to multiple indices
  {
    "query": { "match_all": {} },
    "aggs": {
      "by_index": {
        "terms": { "field": "_index" }   # see which index each result came from
      }
    }
  }

Performance tips:
  Use filter context instead of query context for non-scoring filters:
  {
    "query": {
      "bool": {
        "filter": [              # filter: no relevance scoring = faster + cacheable
          { "term": { "level": "ERROR" }},
          { "range": { "@timestamp": { "gte": "now-1h" }}}
        ]
      }
    }
  }

  vs must context:
  "must": [{ "match": { "level": "ERROR" }}]  # calculates relevance score (slower)
  # Use "must" when you need relevance ranking (search engines)
  # Use "filter" for exact matching (log queries) — faster
```

💡 **Interview tip:** The `filter` vs `must` distinction in bool queries is a performance detail that shows depth — `filter` context skips relevance score calculation and is cacheable by Elasticsearch, making it significantly faster for exact match queries like log filtering. The `.keyword` suffix on text fields for aggregations is frequently misunderstood — Elasticsearch analyzes text fields (splits into tokens) which makes terms aggregation impossible; `.keyword` is the unanalyzed version that stores exact values. Always use `.keyword` for aggregations on string fields.

---

### Q163 — Jenkins | Scenario-Based | Advanced

> Implement **dynamic Jenkins agents on AWS EC2** — spot instances, different types per job, auto-terminate after build.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — dynamic agents, EC2 plugin

```
Option A — Jenkins EC2 Plugin (classic approach):

  Install: Manage Jenkins → Plugins → EC2 Plugin

  Configure in Manage Jenkins → Clouds → Add new cloud → Amazon EC2:

  Cloud configuration:
    Region: us-east-1
    Access Key: (use IAM role on Jenkins master — not credentials)
    Key Pair: jenkins-key

  AMI configuration (per agent type):
  AMI 1 — General (unit tests):
    AMI ID: ami-0abcdef123 (Ubuntu with Jenkins agent + basic tools)
    Instance type: t3.medium
    Number of executors: 1
    Label: general
    Security Groups: jenkins-agents-sg
    Spot instance: ✓ Enabled
    Spot max bid: $0.05/hr
    Idle termination time: 5 minutes  # terminate 5 min after job ends

  AMI 2 — Heavy (integration/build):
    AMI ID: ami-0abcdef456 (Ubuntu with Docker, JDK, Maven)
    Instance type: c5.2xlarge
    Label: heavy
    Spot instance: ✓ Enabled
    Spot max bid: $0.20/hr
    Init script: |
      #!/bin/bash
      docker pull company/build-tools:latest

  In Jenkinsfile — request specific agent type:
  pipeline {
    agent { label 'general' }    // unit tests → t3.medium spot
    stages {
      stage('Unit Test') { steps { sh 'npm test' }}

      stage('Integration Test') {
        agent { label 'heavy' }  // heavy → c5.2xlarge spot
        steps { sh 'mvn verify -Pintegration' }
      }
    }
  }

Option B — Kubernetes Plugin (recommended for EKS):

  Install: Manage Jenkins → Plugins → Kubernetes Plugin

  Configure cloud → Kubernetes:
    Kubernetes URL: https://kubernetes.default.svc
    Namespace: jenkins-agents

  Pod templates (per job type):
  Pod template 1 — lightweight:
    Label: k8s-small
    Container template:
      Image: jenkins/inbound-agent:latest
      CPU request: 500m, limit: 1
      Memory request: 512Mi, limit: 1Gi

  Pod template 2 — Docker builds:
    Label: k8s-docker
    Container template 1: jenkins/inbound-agent
    Container template 2: docker:dind (Docker-in-Docker)
      privileged: true
      env: DOCKER_TLS_CERTDIR=""

  In Jenkinsfile:
  pipeline {
    agent {
      kubernetes {
        label 'k8s-docker'
        yaml '''
          apiVersion: v1
          kind: Pod
          spec:
            containers:
            - name: jnlp
              image: jenkins/inbound-agent:latest
            - name: docker
              image: docker:dind
              securityContext:
                privileged: true
        '''
      }
    }
    stages {
      stage('Build') {
        steps {
          container('docker') {
            sh 'docker build -t myapp:${GIT_COMMIT} .'
          }
        }
      }
    }
  }

Spot instance interruption handling:
  # EC2 Spot interruption = 2-minute warning before termination
  # Jenkins EC2 plugin does NOT handle interruption gracefully by default

  Solution: use On-Demand instances for critical jobs:
    agent { label 'heavy-ondemand' }  // separate AMI config, not spot

  Or: handle in pipeline:
    options {
      retry(2)   // retry on spot interruption
    }

Cost comparison:
  c5.2xlarge On-Demand: $0.34/hr
  c5.2xlarge Spot:      ~$0.10/hr (70% savings)
  For 100 build-hours/month: $34 vs $10
```

💡 **Interview tip:** The Kubernetes plugin is the modern preferred approach over the EC2 plugin — pod-based agents start in seconds (vs minutes for EC2 instances) and clean up perfectly since Kubernetes handles pod lifecycle. However, for GPU jobs, very large memory requirements, or Docker builds, EC2 instances may still be needed. For spot instances, always have a fallback strategy — either retry logic in the pipeline or a secondary on-demand instance pool in Jenkins cloud configuration that activates when spot capacity is unavailable.

---

### Q164 — Linux / Bash | Conceptual | Advanced

> Explain **Linux signals** — SIGTERM, SIGKILL, SIGHUP, SIGINT, SIGCHLD. Why can't SIGKILL be caught? How do containers handle signals differently?

📁 **Reference:** `nawab312/DSA` → `Linux` — signals, kill, trap

```
Linux signals:
  Asynchronous notifications sent to a process
  Source: kernel, another process (with permissions), or the process itself

Key signals:

  SIGTERM (15) — graceful termination request:
    Default action: terminate the process
    Can be caught: YES (process can handle it)
    Can be ignored: YES (but bad practice for daemons)
    Use: kubectl delete pod sends SIGTERM first
         Well-behaved apps: close connections, flush buffers, exit cleanly
         "Please stop when you're ready"

  SIGKILL (9) — immediate forceful termination:
    Default action: kernel kills the process immediately
    Can be caught: NO — kernel enforces this, bypasses process handler
    Can be ignored: NO
    Use: last resort when SIGTERM doesn't work
         kubectl delete pod --grace-period=0 --force
         After terminationGracePeriodSeconds expires
         "You are dead NOW"
    Warning: leaves no chance for cleanup — open files not flushed,
             connections not closed, temp files not deleted

  SIGHUP (1) — hang up / reload:
    Original use: terminal disconnected (before daemons existed)
    Modern use: signal daemon to reload config WITHOUT restarting
    Example: nginx -s reload sends SIGHUP → nginx reloads config files
             prometheus sends SIGHUP → reloads alerting rules
    Can be caught: YES

  SIGINT (2) — interrupt (Ctrl+C):
    Sent by terminal when user presses Ctrl+C
    Default: terminate the process
    Can be caught: YES
    Common pattern: first Ctrl+C → graceful stop, second → SIGKILL
    Well-behaved CLI tools: catch SIGINT, clean up temp files, exit

  SIGCHLD — child process state change:
    Sent to parent when child process: exits, stops, or resumes
    Used by: init systems (systemd), shell, process supervisors
    Shell use: when background job exits → shell gets SIGCHLD
    init (PID 1) use: must handle SIGCHLD to reap zombie processes

Why SIGKILL cannot be caught or ignored:
  Security and reliability guarantee:
  If any process could catch SIGKILL, a malicious or buggy process
  could become unkillable — even root couldn't stop it.
  Kernel bypasses all signal handlers for SIGKILL:
  → Process receives no notification
  → No cleanup code runs
  → Process is removed from process table immediately

Containers and signals:
  Problem — PID 1 in containers:
    Normal Linux: PID 1 = systemd (handles SIGTERM, reaps zombies)
    Container: PID 1 = your app (often doesn't handle signals properly)

    docker stop mycontainer:
    → sends SIGTERM to PID 1 (your app)
    → waits 10 seconds (default grace period)
    → sends SIGKILL if still running

  Problem — shell script as PID 1:
    Entrypoint: CMD ["./start.sh"]
    start.sh is PID 1
    SIGTERM goes to start.sh (not your Java/Node app)
    start.sh may ignore SIGTERM → no graceful shutdown

  Fix 1 — exec form (app becomes PID 1 directly):
    CMD ["node", "server.js"]      # exec form: node is PID 1
    not: CMD "node server.js"      # shell form: sh is PID 1, node is PID 2

  Fix 2 — tini as init:
    docker run --init myimage      # adds tini as PID 1
    tini: proper signal forwarding, zombie reaping
    Kubernetes: add to Dockerfile:
    RUN apt-get install -y tini
    ENTRYPOINT ["/tini", "--", "node", "server.js"]

  Kubernetes SIGTERM flow:
    kubectl delete pod:
    → preStop hook executes (if configured)
    → SIGTERM sent to container PID 1
    → Wait terminationGracePeriodSeconds (default 30s)
    → SIGKILL if still running

  Trap signals in bash scripts:
    #!/bin/bash
    cleanup() {
      echo "Received SIGTERM, cleaning up..."
      kill $CHILD_PID      # forward to child process
      wait $CHILD_PID      # wait for child to finish
      exit 0
    }
    trap cleanup SIGTERM SIGINT

    node server.js &       # start app in background
    CHILD_PID=$!
    wait $CHILD_PID        # wait for app to exit
```

💡 **Interview tip:** The PID 1 container signal problem is the most practically relevant point — many Dockerfiles use shell form `CMD "node server.js"` which means `sh -c "node server.js"` is PID 1 and SIGTERM goes to the shell, not to Node. The shell ignores SIGTERM by default, so the container gets SIGKILL after the grace period with no graceful shutdown. The fix is `CMD ["node", "server.js"]` (exec form) or using `tini`. In Kubernetes, this directly connects to the zero-downtime shutdown question — without proper SIGTERM handling, pods can't drain in-flight requests before being killed.

---

### Q165 — AWS | Scenario-Based | Advanced

> Design a **serverless pipeline for 100,000 events/second** from IoT devices — API Gateway, real-time + batch processing, S3 + DynamoDB storage, cost-effective.

📁 **Reference:** `nawab312/AWS` — Kinesis, Lambda, API Gateway, DynamoDB, S3

```
100,000 events/second challenge:
  API Gateway + Lambda: max ~10,000 concurrent executions by default
  DynamoDB: can handle 100K writes/sec (on-demand mode)
  But: 100K Lambda invocations/sec × 100ms = 10,000 concurrent → throttling

Architecture (event-driven, tiered):

  Ingestion tier:
    IoT Device → API Gateway (REST) → Kinesis Data Streams
    OR: IoT Device → IoT Core → Kinesis Data Streams (better for IoT protocol)

  Kinesis Data Streams setup:
    100K events/sec at 1KB average = 100 MB/s
    Shard capacity: 1 MB/s write per shard
    Required shards: 100 MB/s ÷ 1 MB/s = 100 shards
    Cost: 100 shards × $0.015/shard-hr = $1.50/hr = $1,080/month

  Real-time processing tier:
    Kinesis → Lambda (Event Source Mapping)
      batchSize: 500             # process 500 events per Lambda invocation
      bisectBatchOnError: true   # on error, split batch and retry
      parallelizationFactor: 10  # 10 Lambda invocations per shard simultaneously

    Lambda function:
      Parse event, validate, transform
      Write hot data → DynamoDB (real-time lookups, device state)
      Forward anomalies → SNS (fraud/alert triggering)
      Write to Kinesis Firehose → S3 (batch storage)

  DynamoDB (real-time storage):
    Table: device-events
    Partition key: device_id
    Sort key: timestamp
    On-demand mode: auto-scales for 100K writes/sec
    TTL: 24 hours (keep only recent data in DynamoDB)
    Cost: ~$1.25 per million writes = $10,800/day at 100K/sec (very expensive!)

    Optimization: write to DynamoDB only for high-priority devices/anomalies
    → Reduces DynamoDB cost 90%

  Batch processing tier (S3 + Athena):
    Kinesis Firehose → S3 (batched every 5 minutes or 128MB)
    S3 path: s3://iot-data/year=2024/month=01/day=15/hour=14/
    Format: Parquet (columnar, compressed — 10× cheaper queries)

    Athena: SQL queries on historical S3 data
    Cost: $5/TB scanned → Parquet reduces scan cost 10×

    Glue Crawler: auto-discover new S3 partitions daily
    Glue ETL: nightly batch aggregations (device health scores, daily summaries)

  Alerting tier:
    Lambda → SNS if anomaly detected (temperature > threshold, etc.)
    SNS → Lambda → PagerDuty/Slack
    SNS → SQS → Lambda for fan-out processing

Complete flow:
  IoT Device
    │ MQTT/HTTP
    ▼
  API Gateway / IoT Core
    │
    ▼
  Kinesis Data Streams (100 shards)
    │
    ├──► Lambda (real-time) ──► DynamoDB (hot: last 24hr)
    │                        └► SNS (anomalies)
    │
    └──► Kinesis Firehose ──► S3 (cold: all data, Parquet)
                                │
                                └──► Athena (batch queries)
                                └──► Glue ETL (nightly aggregations)

Cost estimate:
  Kinesis: $1,080/month (100 shards)
  Lambda: ~$500/month (100K events/sec × 24hr × 100ms × $0.0000166)
  DynamoDB: $500/month (anomalies only)
  S3: $50/month (compressed Parquet)
  Kinesis Firehose: $300/month ($0.029/GB)
  Total: ~$2,430/month for 100K events/sec
```

💡 **Interview tip:** The key architectural insight is separating real-time from batch — DynamoDB for sub-millisecond lookups on recent data, S3 + Athena for historical analytics. Writing 100K events/sec to DynamoDB would cost over $10,000/day — the optimization is to only write high-priority events to DynamoDB and batch everything else to S3 via Kinesis Firehose. Kinesis Firehose is the critical middle tier — it automatically buffers, compresses, and delivers data to S3 without any custom code, and can convert JSON to Parquet on the fly using a Lambda transformation.

---

### Q166 — Kubernetes | Conceptual | Advanced

> Explain **PersistentVolume reclaim policies** — Retain, Recycle, Delete. `StorageClass` volume binding `Immediate` vs `WaitForFirstConsumer` for multi-AZ clusters.

📁 **Reference:** `nawab312/Kubernetes` → `04_STORAGE.md`

```
PersistentVolume reclaim policies:
  What happens to the underlying storage when a PVC is deleted?

  Retain:
    → PV status becomes "Released" (not available for new claims)
    → Underlying storage (EBS, NFS) is NOT deleted
    → Data is preserved on the volume
    → Admin must manually reclaim: delete PV object + decide what to do with data
    → Use for: production databases, data that must survive PVC deletion
    → Risk: orphaned PVs accumulate costs if not cleaned up

  Delete:
    → PV and underlying storage (EBS volume) deleted automatically
    → Data is permanently destroyed
    → Default for dynamically provisioned PVs (StorageClass default)
    → Use for: ephemeral data, cache volumes, dev/test environments
    → Risk: accidental PVC deletion = permanent data loss

  Recycle (deprecated, removed in newer K8s):
    → Performs rm -rf on the volume (basic scrub)
    → PV returned to Available for new claims
    → Replaced by: dynamic provisioning + Delete policy

Reclaim policy in practice:

  Production database PVC deletion scenario:
    kubectl delete pvc postgres-data

    Retain policy:
    1. PVC deleted
    2. PV status: Released (bound PVC reference cleared)
    3. EBS volume: still exists, still has all data
    4. Admin sees: kubectl get pv → Released status
    5. Admin decides: archive data or reassign to new PVC
    6. Manual cleanup: kubectl delete pv + manage EBS volume

    Delete policy:
    1. PVC deleted
    2. PV deleted automatically
    3. EBS volume deleted immediately
    4. Data is gone forever

  Change reclaim policy on existing PV:
    kubectl patch pv pv-name -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

StorageClass volume binding modes:

  Immediate (default):
    → PV provisioned as soon as PVC is created
    → EBS volume created in any available AZ (usually AZ where scheduler picks)
    → Problem in multi-AZ: EBS created in us-east-1a, pod scheduled to us-east-1b
      → Pod cannot start (EBS volumes are AZ-specific)
      → Must be in same AZ as the node

  WaitForFirstConsumer:
    → PV provisioning DELAYED until a pod is scheduled that uses this PVC
    → Scheduler picks the node first (considering all constraints)
    → THEN provisioner creates EBS volume in the same AZ as the chosen node
    → Guarantees: pod and EBS volume always in same AZ

  StorageClass with WaitForFirstConsumer:
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: gp3-wait
  provisioner: ebs.csi.aws.com
  volumeBindingMode: WaitForFirstConsumer   # ← critical for multi-AZ
  reclaimPolicy: Retain                     # keep data on PVC delete
  parameters:
    type: gp3
    iops: "3000"
    throughput: "125"

  Why it matters for StatefulSets:
    StatefulSet with 3 replicas across 3 AZs:
    Immediate: all 3 EBS volumes might land in same AZ → replicas can't spread
    WaitForFirstConsumer: each pod scheduled to its AZ → EBS created there
    Result: true multi-AZ distribution for stateful workloads
```

💡 **Interview tip:** `WaitForFirstConsumer` is essential for any multi-AZ StatefulSet deployment — without it, you can end up with an EBS volume in us-east-1a and a pod that gets scheduled to us-east-1b, causing the pod to be permanently stuck in Pending. This is a very common production issue when teams upgrade from single-AZ to multi-AZ clusters without updating their StorageClass. The Retain reclaim policy should be default for any production database — the Delete policy is convenient for development but causes permanent data loss if someone accidentally deletes a PVC, which has happened in production more times than anyone admits.

---

### Q167 — GitHub Actions | Scenario-Based | Advanced

> Python library workflow — matrix (3.10/3.11/3.12 × 3 OS), **PyPI OIDC publishing on tag**, changelog from commits, GitHub Release.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions`

```yaml
name: CI and Release

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        python: ["3.10", "3.11", "3.12"]
        os: [ubuntu-latest, windows-latest, macos-latest]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python }}
          cache: pip

      - name: Install dependencies
        run: |
          pip install -e ".[dev]"

      - name: Run tests
        run: pytest tests/ --cov=mylib --cov-report=xml -v

      - name: Upload coverage
        if: matrix.os == 'ubuntu-latest' && matrix.python == '3.12'
        uses: codecov/codecov-action@v4

  build:
    needs: test
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0    # full history needed for changelog

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Extract version from tag
        id: version
        run: echo "version=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

      - name: Build package
        run: |
          pip install build
          python -m build    # creates dist/*.whl and dist/*.tar.gz

      - name: Upload dist artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  changelog:
    needs: test
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    outputs:
      changelog: ${{ steps.changelog.outputs.changelog }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate changelog from commits
        id: changelog
        run: |
          PREV_TAG=$(git describe --tags --abbrev=0 HEAD^ 2>/dev/null || echo "")
          if [ -n "$PREV_TAG" ]; then
            COMMITS=$(git log ${PREV_TAG}..HEAD --pretty=format:"- %s (%an)" \
              --no-merges)
          else
            COMMITS=$(git log --pretty=format:"- %s (%an)" --no-merges)
          fi

          # Categorise by conventional commit type
          FEATURES=$(echo "$COMMITS" | grep "^- feat" || echo "")
          FIXES=$(echo "$COMMITS" | grep "^- fix" || echo "")
          OTHER=$(echo "$COMMITS" | grep -v "^- feat\|^- fix" || echo "")

          CHANGELOG=""
          [ -n "$FEATURES" ] && CHANGELOG+="### Features\n${FEATURES}\n\n"
          [ -n "$FIXES" ]    && CHANGELOG+="### Bug Fixes\n${FIXES}\n\n"
          [ -n "$OTHER" ]    && CHANGELOG+="### Other\n${OTHER}\n"

          # Store as multiline output
          {
            echo 'changelog<<EOF'
            echo -e "$CHANGELOG"
            echo 'EOF'
          } >> $GITHUB_OUTPUT

  publish-pypi:
    needs: [build, changelog]
    runs-on: ubuntu-latest
    environment:
      name: pypi
      url: https://pypi.org/project/mylib/${{ needs.build.outputs.version }}/
    permissions:
      id-token: write   # REQUIRED for OIDC trusted publishing

    steps:
      - name: Download dist
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/

      - name: Publish to PyPI (OIDC — no API key needed)
        uses: pypa/gh-action-pypi-publish@release/v1
        # No username/password — uses OIDC token exchange
        # PyPI configured to trust this repo + environment

  github-release:
    needs: [build, changelog, publish-pypi]
    runs-on: ubuntu-latest
    permissions:
      contents: write   # required to create releases

    steps:
      - name: Download dist
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ github.ref_name }}
          name: "Release ${{ github.ref_name }}"
          body: ${{ needs.changelog.outputs.changelog }}
          files: dist/*
          draft: false
          prerelease: ${{ contains(github.ref_name, 'alpha') || contains(github.ref_name, 'beta') }}
```

```
# PyPI OIDC setup (one-time configuration on PyPI):
# pypi.org → Your Account → Publishing → Add a new publisher
#   Owner:       your-github-org
#   Repository:  mylib
#   Workflow:    ci.yml
#   Environment: pypi
# No API tokens stored anywhere — GitHub proves identity via OIDC
```

💡 **Interview tip:** OIDC trusted publishing to PyPI is the modern best practice — it eliminates the need to store a PyPI API token in GitHub Secrets entirely. GitHub proves to PyPI that the workflow is running from the correct repository and environment using a short-lived OIDC token, and PyPI grants upload permission. The `environment: pypi` in the publish job is required — PyPI's trusted publisher configuration matches against the environment name. The `fetch-depth: 0` in checkout is critical for changelog generation — without it, GitHub Actions does a shallow clone with only the latest commit, and `git log` for previous tags returns nothing.

---

### Q168 — Terraform | Conceptual | Advanced

> Explain **Terraform `output` values** and `terraform_remote_state`. Security risks of sensitive outputs. How to handle secrets in outputs.

📁 **Reference:** `nawab312/Terraform` — outputs, remote state, sensitive values

```
Terraform outputs:
  → Export values from a module or configuration for external use
  → Displayed at end of terraform apply
  → Accessible via: terraform output <name>
  → Used for: cross-configuration references, human visibility

Basic output:
  output "vpc_id" {
    value       = aws_vpc.main.id
    description = "The ID of the main VPC"
  }

  output "alb_dns_name" {
    value       = aws_lb.main.dns_name
    description = "DNS name to point Route53 records to"
  }

Sensitive outputs:
  output "db_password" {
    value     = random_password.db.result
    sensitive = true    # marks as sensitive
  }

  Behavior of sensitive = true:
  → terraform apply: shows "[sensitive]" instead of value
  → terraform output db_password: requires explicit request
  → terraform output -raw db_password: shows raw value (for scripts)
  → STILL stored in state file as PLAINTEXT

Security risks of sensitive outputs:

  Risk 1: Stored in state file as plaintext:
    terraform.tfstate contains: "db_password": "SuperSecret123"
    State file in S3: anyone with S3 GetObject = can read all passwords
    Fix: enable S3 server-side encryption + bucket policy (least privilege)

  Risk 2: Logged in CI/CD pipelines:
    terraform output db_password > /tmp/secrets  # logged in CI
    Fix: never output secrets to stdout in CI/CD
    Fix: use terraform_remote_state data source to read value directly

  Risk 3: Referenced in other configs (propagation):
    data "terraform_remote_state" "app" {
      ...
    }
    locals {
      db_pass = data.terraform_remote_state.app.outputs.db_password
    }
    → db_pass now in local state file too

Better alternatives for secrets:

  Option A: Don't output secrets — use Secrets Manager:
    resource "aws_secretsmanager_secret_version" "db" {
      secret_id     = aws_secretsmanager_secret.db.id
      secret_string = jsonencode({ password = random_password.db.result })
    }
    # Output only the SECRET ARN (not the value):
    output "db_secret_arn" {
      value = aws_secretsmanager_secret.db.arn
    }
    # Other configs use: aws_secretsmanager_secret_version data source

  Option B: Mark sensitive and use encrypted state:
    output "db_password" {
      value     = random_password.db.result
      sensitive = true
    }
    # Encrypt state with KMS:
    terraform {
      backend "s3" {
        encrypt        = true
        kms_key_id     = "arn:aws:kms:..."
        bucket         = "terraform-state"
        key            = "prod/terraform.tfstate"
      }
    }

terraform_remote_state (cross-config reference):
  # Config A exports VPC ID:
  output "private_subnet_ids" {
    value = aws_subnet.private[*].id
  }

  # Config B reads it:
  data "terraform_remote_state" "networking" {
    backend = "s3"
    config = {
      bucket = "terraform-state"
      key    = "networking/terraform.tfstate"
      region = "us-east-1"
    }
  }
  resource "aws_instance" "web" {
    subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
  }

  When to use terraform_remote_state vs data sources:
    remote_state: read ANY output from another Terraform config
    data sources: query AWS API directly (more up-to-date, no state coupling)
    Prefer data sources when possible — remote_state creates tight coupling
    between configurations (both must use same backend, same state format)
```

💡 **Interview tip:** The most important point: `sensitive = true` only hides values in terminal output — the value is still stored as plaintext in the state file. True secret security requires either not outputting secrets at all (use Secrets Manager ARN instead of the actual secret) or encrypting the state file with a KMS key. The `terraform_remote_state` tight coupling is a real operational problem at scale — if the networking config changes its output names, all dependent configs break. Data sources (like `aws_vpc` by tag) are more resilient because they query AWS directly rather than depending on another Terraform state file's structure.

---

### Q169 — Prometheus | Troubleshooting | Advanced

> Prometheus showing **`TSDB compaction failed`** errors. What is compaction, why does it fail, what are consequences, investigation and fix?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`

```
What is TSDB compaction:
  Prometheus stores data in blocks (2-hour segments of immutable data)
  Compaction = merging small blocks into larger ones + applying downsampling

  data/ directory:
  01ABC (2 hours) ─┐
  01DEF (2 hours) ─┼──► compact ──► 01GHI (6 hours)
  01JKL (2 hours) ─┘

  Benefits of compaction:
  → Fewer files to scan for queries (faster query performance)
  → Better compression (larger blocks compress better)
  → Apply tombstones (delete marked series permanently)
  → Apply chunk downsampling

  Schedule:
    Every 2 hours: head block sealed → new 2h block
    Compaction runs in background: 2h → 6h → 24h → 48h → 7d blocks

Why compaction fails:

  Cause 1: Disk full (most common):
    prometheus_tsdb_compactions_failed_total increases
    Logs: "compaction failed: open temp dir: no space left on device"
    Cannot write compacted block → failure

  Cause 2: Corruption in source block:
    Logs: "compaction failed: overlapping blocks"
    Block metadata reports overlapping time ranges
    Cause: Prometheus crashed during compaction → partial blocks

  Cause 3: Too many open files:
    Compaction opens all chunk files in source blocks simultaneously
    For large TSDB: thousands of files → hits ulimit
    Logs: "too many open files"

  Cause 4: OOM during compaction:
    Large TSDB + heavy compaction → high memory usage
    Prometheus OOM killed by kernel mid-compaction → corrupted blocks

Consequences of failed compaction:
  Short term: query performance degrades (many small blocks to scan)
  Medium term: disk space grows unbounded (no cleanup of old data)
  Long term: Prometheus may fail to start (too many block files to load)
  Worst case: block corruption → data loss on restart

Step 1 — Identify the error:
  kubectl logs prometheus-0 -n monitoring | grep -i "compaction\|tsdb"

  prometheus_tsdb_compactions_failed_total > 0   → compaction failing
  prometheus_tsdb_head_chunks                    → check memory usage
  process_open_fds / process_max_fds > 0.8       → FD limit approaching

Step 2 — Check disk usage:
  kubectl exec prometheus-0 -- df -h /prometheus
  kubectl exec prometheus-0 -- du -sh /prometheus/data/*
  # If disk > 90%: emergency action needed

  Emergency: reduce retention immediately:
  kubectl edit statefulset prometheus
  # Add/change arg: --storage.tsdb.retention.time=7d
  # After restart: prometheus will delete blocks older than 7 days

Step 3 — Fix corrupted blocks:
  # Check for block issues:
  kubectl exec prometheus-0 -- promtool tsdb analyze /prometheus/data

  # If overlapping blocks: manually delete problematic block:
  kubectl exec prometheus-0 -- ls /prometheus/data
  kubectl exec prometheus-0 -- rm -rf /prometheus/data/01CORRUPTED_BLOCK
  kubectl delete pod prometheus-0  # restart to reload blocks

Step 4 — Fix FD limit:
  # In StatefulSet / Deployment:
  securityContext:
    {}
  # In container spec:
  resources:
    limits:
      memory: 4Gi
  # Add to pod spec:
  # (use LimitRange or SecurityContext at cluster level)

  # Or set in Prometheus container:
  # Add ulimit via initContainer or security context

Step 5 — Prevent future failures:
  # Add retention size limit (auto-cleanup):
  --storage.tsdb.retention.size=10GB  # never exceed 10GB

  # Monitor and alert:
  alert: PrometheusStorageCritical
  expr: (prometheus_tsdb_storage_blocks_bytes /
         node_filesystem_size_bytes{mountpoint="/prometheus"}) > 0.8
  for: 10m
  annotations:
    summary: "Prometheus disk at {{ $value | humanizePercentage }}"
```

💡 **Interview tip:** `--storage.tsdb.retention.size` is the production safety net — without it, Prometheus will fill the disk completely and crash. Setting a size limit causes automatic deletion of the oldest blocks when approaching the limit. The most actionable immediate fix for a full disk is reducing `--storage.tsdb.retention.time` to a shorter period (7d instead of 30d) — this triggers deletion of old blocks on next compaction cycle. Corrupted blocks (overlapping time ranges) are usually caused by a crash during compaction — the fix is to identify and delete the corrupted block, as Prometheus cannot repair it.

---

### Q170 — AWS | Conceptual | Advanced

> Explain **AWS WAF**. What attacks does it protect against? Write rules: rate limit 1000/5min per IP, block specific countries, block SQL injection.

📁 **Reference:** `nawab312/AWS` — WAF, ALB, CloudFront

```
AWS WAF — Web Application Firewall:
  Operates at Layer 7 (HTTP/HTTPS) — inspects request content
  Protects: ALB, API Gateway, CloudFront, AppSync, Cognito

  Attacks WAF protects against:
  → SQL Injection (SQLi): malicious SQL in query params/body
  → Cross-Site Scripting (XSS): malicious scripts in inputs
  → Request flooding / DDoS: rate-based rules
  → Bad bots: known crawler/scanner signatures
  → Geo-based threats: block entire countries
  → Known malicious IPs: AWS threat intelligence lists
  → OWASP Top 10: via AWS Managed Rule Groups

Web ACL structure:
  Web ACL → Rules (evaluated in priority order) → Action (Allow/Block/Count)
  First matching rule wins

Rule 1 — Rate limiting (1000 requests per 5 minutes per IP):
  {
    "Name": "RateLimitPerIP",
    "Priority": 1,
    "Statement": {
      "RateBasedStatement": {
        "Limit": 1000,
        "AggregateKeyType": "IP",
        "EvaluationWindowSec": 300
      }
    },
    "Action": { "Block": {} },
    "VisibilityConfig": {
      "SampledRequestsEnabled": true,
      "CloudWatchMetricsEnabled": true,
      "MetricName": "RateLimitPerIP"
    }
  }

  aws wafv2 create-web-acl \
    --name "production-waf" \
    --scope REGIONAL \          # or CLOUDFRONT (must be us-east-1)
    --default-action Allow={} \
    --rules file://rules.json \
    --visibility-config SampledRequestsEnabled=true,...

Rule 2 — Block specific countries (geo-blocking):
  {
    "Name": "BlockCountries",
    "Priority": 2,
    "Statement": {
      "GeoMatchStatement": {
        "CountryCodes": ["CN", "RU", "KP", "IR"]
      }
    },
    "Action": { "Block": {} },
    "VisibilityConfig": { "MetricName": "BlockCountries", ... }
  }

Rule 3 — SQL Injection protection:
  {
    "Name": "BlockSQLInjection",
    "Priority": 3,
    "Statement": {
      "SqliMatchStatement": {
        "FieldToMatch": {
          "AllQueryArguments": {}    # check all query params
        },
        "TextTransformations": [
          { "Priority": 1, "Type": "URL_DECODE" },
          { "Priority": 2, "Type": "HTML_ENTITY_DECODE" },
          { "Priority": 3, "Type": "LOWERCASE" }
        ],
        "SensitivityLevel": "HIGH"
      }
    },
    "Action": { "Block": {} },
    ...
  }

Rule 4 — AWS Managed Rules (recommended baseline):
  {
    "Name": "AWSManagedRulesCommonRuleSet",
    "Priority": 10,
    "OverrideAction": { "None": {} },
    "Statement": {
      "ManagedRuleGroupStatement": {
        "VendorName": "AWS",
        "Name": "AWSManagedRulesCommonRuleSet"
      }
    },
    ...
  }

  AWS Managed Rule Groups:
    AWSManagedRulesCommonRuleSet     → OWASP Top 10 (SQLi, XSS, etc.)
    AWSManagedRulesKnownBadInputsRuleSet → known malicious patterns
    AWSManagedRulesSQLiRuleSet       → comprehensive SQL injection
    AWSManagedRulesAmazonIpReputationList → known malicious IPs
    AWSManagedRulesBotControlRuleSet → bot detection + CAPTCHA

Attach Web ACL to ALB:
  aws wafv2 associate-web-acl \
    --web-acl-arn arn:aws:wafv2:us-east-1:123:regional/webacl/production-waf/xxx \
    --resource-arn arn:aws:elasticloadbalancing:us-east-1:123:loadbalancer/app/my-alb/xxx

For CloudFront: Web ACL must be created in us-east-1 (global):
  aws wafv2 create-web-acl --scope CLOUDFRONT --region us-east-1

WAF pricing:
  $5/Web ACL/month
  $1/Rule/month
  $0.60/million requests inspected
  AWS Managed Rules: additional per-rule charge
```

💡 **Interview tip:** The `TextTransformations` in the SQLi rule is important — attackers encode SQL injection attempts to evade detection (URL encoding: `%27` for quote, HTML encoding: `&#39;`). Multiple transformations ensure the rule catches encoded variants. AWS Managed Rules (especially `AWSManagedRulesCommonRuleSet`) is the answer for most production deployments — they're continuously updated by AWS security researchers and cover the OWASP Top 10 with zero configuration. Always start WAF rules in `Count` mode instead of `Block` to verify they don't cause false positives before enabling blocking in production.

---

### Q171 — Kubernetes | Scenario-Based | Advanced

> Onboard new microservice — connect to **external Redis TLS**, pull from **private ECR**, access **AWS Secrets Manager via IRSA**, cluster-internal only.

📁 **Reference:** `nawab312/Kubernetes` → `05_CONFIGURATION_SECRETS.md`, `07_SECURITY.md`

```yaml
# Step 1 — ServiceAccount with IRSA annotation (AWS Secrets Manager access)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-service-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/payment-service-role
    # IAM role must have: secretsmanager:GetSecretValue on specific secrets

---
# Step 2 — Secret for ECR image pull (or use IRSA on node role)
# For private ECR: node IAM role typically has ECR pull permissions
# For explicit imagePullSecret:
apiVersion: v1
kind: Secret
metadata:
  name: ecr-pull-secret
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
# Generate: kubectl create secret docker-registry ecr-pull-secret \
#   --docker-server=123456789.dkr.ecr.us-east-1.amazonaws.com \
#   --docker-username=AWS \
#   --docker-password=$(aws ecr get-login-password --region us-east-1)

---
# Step 3 — ConfigMap for Redis TLS CA certificate
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-tls-config
  namespace: production
data:
  redis-ca.crt: |
    -----BEGIN CERTIFICATE-----
    MIIBxxx...  # ElastiCache TLS CA certificate
    -----END CERTIFICATE-----

---
# Step 4 — Complete Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      serviceAccountName: payment-service-sa    # IRSA for Secrets Manager

      # Pull from private ECR
      imagePullSecrets:
      - name: ecr-pull-secret
      # OR: rely on node IAM role with AmazonEC2ContainerRegistryReadOnly

      containers:
      - name: payment
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/payment-service:v1.2.3

        env:
        # Redis connection (TLS endpoint)
        - name: REDIS_URL
          value: "rediss://my-cluster.abc.cache.amazonaws.com:6380"  # rediss:// = TLS
        - name: REDIS_TLS_CA_FILE
          value: "/etc/redis-tls/redis-ca.crt"

        # Secrets Manager — app reads at runtime using IRSA credentials
        - name: AWS_SECRETS_ARN
          value: "arn:aws:secretsmanager:us-east-1:123:secret:prod/payment/db"
        # App code: boto3.client('secretsmanager').get_secret_value(SecretId=os.environ['AWS_SECRETS_ARN'])

        # AWS region for SDK (required for IRSA token exchange)
        - name: AWS_REGION
          value: "us-east-1"

        volumeMounts:
        - name: redis-tls
          mountPath: /etc/redis-tls
          readOnly: true

        ports:
        - containerPort: 8080
          name: http

        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "1"
            memory: "512Mi"

      volumes:
      - name: redis-tls
        configMap:
          name: redis-tls-config    # mount CA cert for Redis TLS verification

---
# Step 5 — ClusterIP Service (internal only — no LoadBalancer, no NodePort)
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: production
spec:
  type: ClusterIP              # only reachable from within cluster
  selector:
    app: payment-service
  ports:
  - name: http
    port: 80
    targetPort: 8080

---
# Step 6 — NetworkPolicy (restrict to cluster-internal only)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: production
    ports:
    - port: 8080
  egress:
  - ports:               # Redis TLS
    - port: 6380
      protocol: TCP
  - ports:               # AWS APIs (Secrets Manager, STS for IRSA)
    - port: 443
      protocol: TCP
  - ports:               # DNS
    - port: 53
      protocol: UDP
```

💡 **Interview tip:** The `rediss://` (double 's') URL scheme is how most Redis clients indicate TLS — a common mistake is using `redis://` which silently connects without TLS. Mounting the CA certificate via ConfigMap is cleaner than setting `REDIS_TLS_INSECURE=true` which disables certificate verification entirely. For Secrets Manager, the best practice is having the application call the Secrets Manager API at runtime rather than injecting the secret value as an environment variable — this way, secrets are refreshed automatically on rotation without pod restarts.

---

### Q172 — ArgoCD | Scenario-Based | Advanced

> Implement **progressive delivery** with Argo Rollouts — canary 10%→30%→100%, auto-promote on error rate < 1%, auto-rollback on error rate > 5%.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — Argo Rollouts, canary analysis

```
Step 1 — Install Argo Rollouts:
  kubectl create namespace argo-rollouts
  kubectl apply -n argo-rollouts \
    -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

Step 2 — Convert Deployment to Rollout:
  apiVersion: argoproj.io/v1alpha1
  kind: Rollout
  metadata:
    name: payment-api
    namespace: production
  spec:
    replicas: 10
    selector:
      matchLabels:
        app: payment-api
    template:
      metadata:
        labels:
          app: payment-api
      spec:
        containers:
        - name: payment
          image: payment-api:v1.0.0
          resources:
            requests: { cpu: "250m", memory: "256Mi" }

    strategy:
      canary:
        # Traffic routing via Ingress (or Istio VirtualService)
        canaryService: payment-api-canary    # separate Service for canary pods
        stableService: payment-api-stable    # separate Service for stable pods

        # Canary analysis — runs between steps
        analysis:
          startingStep: 1     # start analysis after step 1 (10% canary)
          templates:
          - templateName: error-rate-check

        steps:
        # Step 1: route 10% of traffic to canary
        - setWeight: 10
        # Step 2: pause 5 minutes and run analysis
        - pause: { duration: 5m }
        # Step 3: increase to 30% if analysis passed
        - setWeight: 30
        # Step 4: pause 10 minutes and run analysis
        - pause: { duration: 10m }
        # Step 5: full rollout
        - setWeight: 100

Step 3 — AnalysisTemplate (Prometheus-driven):
  apiVersion: argoproj.io/v1alpha1
  kind: AnalysisTemplate
  metadata:
    name: error-rate-check
    namespace: production
  spec:
    metrics:
    - name: error-rate
      interval: 1m           # evaluate every minute
      successCondition: result[0] < 0.01    # pass if error rate < 1%
      failureCondition: result[0] > 0.05    # fail if error rate > 5%
      failureLimit: 2         # allow 2 failures before aborting rollout
      provider:
        prometheus:
          address: http://prometheus-operated:9090
          query: |
            sum(rate(http_requests_total{
              service="payment-api",
              status=~"5..",
              version="{{args.canary-version}}"
            }[5m]))
            /
            sum(rate(http_requests_total{
              service="payment-api",
              version="{{args.canary-version}}"
            }[5m]))

  # Pass canary version as argument in Rollout spec:
  analysis:
    args:
    - name: canary-version
      valueFrom:
        podTemplateHashValue: Latest

Step 4 — Services for traffic splitting:
  # Stable service (unchanged pods)
  apiVersion: v1
  kind: Service
  metadata:
    name: payment-api-stable
    namespace: production
  spec:
    selector:
      app: payment-api
      # Argo Rollouts adds: rollouts-pod-template-hash=<stable-hash>
    ports:
    - port: 80

  # Canary service (new version pods)
  apiVersion: v1
  kind: Service
  metadata:
    name: payment-api-canary
    namespace: production
  spec:
    selector:
      app: payment-api
      # Argo Rollouts adds: rollouts-pod-template-hash=<canary-hash>
    ports:
    - port: 80

  # Ingress weights (nginx ingress):
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: payment-api
    annotations:
      nginx.ingress.kubernetes.io/canary: "true"
      nginx.ingress.kubernetes.io/canary-weight: "10"  # managed by Argo Rollouts
  spec:
    rules:
    - host: payment.company.com
      http:
        paths:
        - path: /
          backend:
            service: { name: payment-api-canary, port: { number: 80 }}

Step 5 — Monitoring rollout progress:
  kubectl argo rollouts get rollout payment-api --watch -n production
  kubectl argo rollouts dashboard              # web UI on port 3100

  # Manual promotion (skip pause):
  kubectl argo rollouts promote payment-api -n production

  # Manual rollback (abort and revert):
  kubectl argo rollouts abort payment-api -n production
```

💡 **Interview tip:** The `successCondition` and `failureCondition` being separate thresholds is the key to avoiding flapping — success requires error rate < 1%, but failure only triggers at > 5%. The 1-5% range is "inconclusive" which continues analysis without automatic action. The `failureLimit: 2` means two consecutive failures (2 minutes of > 5% error rate) before aborting — one bad minute doesn't trigger a rollback. For the Prometheus query, using `version` label on metrics to separate canary from stable traffic is essential — without it, the analysis queries the aggregate error rate which includes stable traffic and masks canary problems.

---

### Q173 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash fleet health check script** — reads server list, parallel checks (SSH, disk, CPU, memory, service), max 10 concurrent, color-coded HTML report, email on failure.

📁 **Reference:** `nawab312/DSA` → `Linux`

```bash
#!/bin/bash
set -euo pipefail

SERVERS_FILE="${1:-servers.txt}"
MAX_PARALLEL=10
REPORT_FILE="/tmp/health-report-$(date +%Y%m%d-%H%M%S).html"
EMAIL_TO="${EMAIL_TO:-ops@company.com}"
CHECK_SERVICE="${CHECK_SERVICE:-nginx}"
DISK_THRESHOLD=80
CPU_THRESHOLD=70
MEM_THRESHOLD=85
RESULTS_DIR=$(mktemp -d)

# Colors for HTML
RED="#FF4444"; GREEN="#44BB44"; YELLOW="#FFAA00"; GRAY="#CCCCCC"

# Check one server — runs in subshell for parallel execution
check_server() {
  local server="$1"
  local result_file="${RESULTS_DIR}/${server//\//_}.json"
  local ssh_ok=false disk_ok=false cpu_ok=false mem_ok=false svc_ok=false
  local disk_pct=0 cpu_pct=0 mem_pct=0 error=""

  # SSH reachability
  if ssh -o ConnectTimeout=5 -o StrictHostKeyChecking=no \
         -o BatchMode=yes "$server" "echo ok" &>/dev/null 2>&1; then
    ssh_ok=true

    # Run all checks in one SSH session
    remote_data=$(ssh -o ConnectTimeout=10 -o StrictHostKeyChecking=no \
                      -o BatchMode=yes "$server" \
                      "df -h / | awk 'NR==2{print \$5}' | tr -d '%'; \
                       top -b -n1 | grep 'Cpu' | awk '{print \$2}' | tr -d 'us,'; \
                       free | awk '/Mem/{printf \"%.0f\", \$3/\$2*100}'; \
                       systemctl is-active ${CHECK_SERVICE} 2>/dev/null || echo inactive" \
                  2>/dev/null || echo "0 0 0 inactive")

    disk_pct=$(echo "$remote_data" | sed -n '1p')
    cpu_pct=$(echo "$remote_data"  | sed -n '2p')
    mem_pct=$(echo "$remote_data"  | sed -n '3p')
    svc_status=$(echo "$remote_data" | sed -n '4p')

    [[ "${disk_pct:-99}" -lt $DISK_THRESHOLD ]] && disk_ok=true
    [[ $(echo "${cpu_pct:-99} < $CPU_THRESHOLD" | bc -l 2>/dev/null) -eq 1 ]] && cpu_ok=true
    [[ "${mem_pct:-99}" -lt $MEM_THRESHOLD ]]  && mem_ok=true
    [[ "$svc_status" == "active" ]]            && svc_ok=true
  else
    error="SSH unreachable"
  fi

  # Write JSON result
  cat > "$result_file" <<EOF
{
  "server": "$server",
  "ssh": $ssh_ok,
  "disk": $disk_ok,
  "disk_pct": ${disk_pct:-0},
  "cpu": $cpu_ok,
  "cpu_pct": ${cpu_pct:-0},
  "mem": $mem_ok,
  "mem_pct": ${mem_pct:-0},
  "service": $svc_ok,
  "error": "$error"
}
EOF
}

# Run checks with max parallelism (semaphore via background jobs)
echo "Checking servers from: $SERVERS_FILE"
active=0
while IFS= read -r server || [ -n "$server" ]; do
  [ -z "$server" ] || [[ "$server" == "#"* ]] && continue
  check_server "$server" &
  active=$((active + 1))
  [ $active -ge $MAX_PARALLEL ] && { wait -n 2>/dev/null || wait; active=$((active - 1)); }
done < "$SERVERS_FILE"
wait  # wait for remaining

# Generate HTML report
TOTAL=$(ls "$RESULTS_DIR"/*.json 2>/dev/null | wc -l)
FAILED=$(grep -l '"ssh": false\|"disk": false\|"cpu": false\|"mem": false\|"service": false' \
         "$RESULTS_DIR"/*.json 2>/dev/null | wc -l)

cat > "$REPORT_FILE" <<HTML
<!DOCTYPE html><html><head><title>Fleet Health Report</title>
<style>
  body{font-family:monospace;margin:20px;background:#1a1a1a;color:#eee}
  h1{color:#fff}
  table{width:100%;border-collapse:collapse}
  th{background:#333;padding:8px;text-align:left}
  td{padding:6px 8px;border-bottom:1px solid #333}
  .ok{color:${GREEN}} .fail{color:${RED}} .warn{color:${YELLOW}}
  .summary{margin-bottom:20px;font-size:1.2em}
</style></head><body>
<h1>Fleet Health Report — $(date '+%Y-%m-%d %H:%M:%S')</h1>
<div class="summary">
  Total: <b>$TOTAL</b> |
  <span class="ok">Healthy: $((TOTAL - FAILED))</span> |
  <span class="fail">Unhealthy: $FAILED</span>
</div>
<table>
<tr><th>Server</th><th>SSH</th><th>Disk</th><th>CPU</th><th>Memory</th><th>Service (${CHECK_SERVICE})</th><th>Notes</th></tr>
HTML

for f in "${RESULTS_DIR}"/*.json; do
  server=$(jq -r .server "$f")
  ssh=$(jq -r .ssh "$f")
  disk=$(jq -r .disk "$f"); disk_pct=$(jq -r .disk_pct "$f")
  cpu=$(jq -r .cpu "$f");   cpu_pct=$(jq -r .cpu_pct "$f")
  mem=$(jq -r .mem "$f");   mem_pct=$(jq -r .mem_pct "$f")
  svc=$(jq -r .service "$f")
  err=$(jq -r .error "$f")

  cell() { [ "$1" = "true" ] && echo "<span class='ok'>✓</span>" || echo "<span class='fail'>✗</span>"; }

  echo "<tr>
    <td>$server</td>
    <td>$(cell $ssh)</td>
    <td>$(cell $disk) ${disk_pct}%</td>
    <td>$(cell $cpu) ${cpu_pct}%</td>
    <td>$(cell $mem) ${mem_pct}%</td>
    <td>$(cell $svc)</td>
    <td>${err}</td>
  </tr>" >> "$REPORT_FILE"
done

echo "</table></body></html>" >> "$REPORT_FILE"

# Email if failures found
if [ "$FAILED" -gt 0 ] && command -v mail >/dev/null; then
  echo "Sending alert: $FAILED of $TOTAL servers have issues"
  SUBJECT="[ALERT] Fleet Health: $FAILED/$TOTAL servers unhealthy — $(date '+%Y-%m-%d %H:%M')"
  mail -s "$SUBJECT" -a "Content-Type: text/html" \
    "$EMAIL_TO" < "$REPORT_FILE" && echo "Alert emailed to $EMAIL_TO"
fi

echo "Report: $REPORT_FILE"
rm -rf "$RESULTS_DIR"
```

💡 **Interview tip:** The semaphore pattern using `wait -n` (wait for any one background job to finish) is key for bounded parallelism — without it you'd either run all checks sequentially (slow) or all at once (hundreds of SSH connections simultaneously). `wait -n` requires bash 4.3+; the fallback `wait` without `-n` waits for ALL jobs which is less efficient but safe. Running all remote checks in a single SSH session (one connection per server with multiple commands) is significantly faster than opening a new SSH connection for each check.

---

### Q174 — AWS | Scenario-Based | Advanced

> Set up **AWS Organizations** — centralized CloudTrail logging, SCP preventing CloudTrail disable, consolidated billing, cross-account developer access (dev but not prod).

📁 **Reference:** `nawab312/AWS` — AWS Organizations, SCPs, CloudTrail

```
AWS Organizations structure:
  Management Account (billing, governance)
  └── Root
      ├── OU: Security (audit, log-archive accounts)
      ├── OU: Production
      │   ├── Account: prod-us-east-1
      │   └── Account: prod-eu-west-1
      └── OU: Development
          ├── Account: dev-team-a
          └── Account: dev-team-b

Step 1 — Create Organizations:
  aws organizations create-organization --feature-set ALL
  aws organizations create-organizational-unit \
    --parent-id r-xxxx --name Security
  aws organizations create-organizational-unit \
    --parent-id r-xxxx --name Production

Step 2 — Centralized CloudTrail (all accounts → one S3 bucket):

  In log-archive account (Security OU): create S3 bucket:
  aws s3api create-bucket --bucket company-cloudtrail-logs-123456789

  S3 bucket policy (allows all org accounts to write):
  {
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": { "Service": "cloudtrail.amazonaws.com" },
        "Action": "s3:PutObject",
        "Resource": "arn:aws:s3:::company-cloudtrail-logs-123456789/AWSLogs/*",
        "Condition": {
          "StringEquals": {
            "aws:SourceOrgID": "o-xxxxxxxxxxxx"   # allow all org accounts
          }
        }
      }
    ]
  }

  Create Organizations trail (from management account):
  aws cloudtrail create-trail \
    --name org-trail \
    --s3-bucket-name company-cloudtrail-logs-123456789 \
    --is-organization-trail \               # covers ALL member accounts
    --is-multi-region-trail \               # all regions
    --enable-log-file-validation
  aws cloudtrail start-logging --name org-trail

Step 3 — SCP: prevent disabling CloudTrail:
  aws organizations create-policy \
    --name "PreventCloudTrailDisable" \
    --type SERVICE_CONTROL_POLICY \
    --content '{
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Deny",
        "Action": [
          "cloudtrail:DeleteTrail",
          "cloudtrail:StopLogging",
          "cloudtrail:UpdateTrail",
          "cloudtrail:PutEventSelectors"
        ],
        "Resource": "*"
      }]
    }'

  # Apply SCP to root (all accounts including production):
  aws organizations attach-policy \
    --policy-id p-xxxxxxxxxx \
    --target-id r-xxxx   # root

  Additional security SCPs for production OU:
  {
    "Effect": "Deny",
    "Action": [
      "guardduty:DeleteDetector",
      "config:DeleteConfigurationRecorder",
      "securityhub:DisableSecurityHub"
    ],
    "Resource": "*"
  }

Step 4 — Consolidated billing:
  → Automatic when using AWS Organizations
  → Cost Explorer: shows total + per-account breakdown
  → Cost allocation tags: enforce via SCP or tag policies

  Tag policy (require cost-center tag on all resources):
  {
    "tags": {
      "cost-center": {
        "tag_key": { "@@assign": "cost-center" },
        "enforced_for": { "@@assign": ["ec2:instance", "rds:db"] }
      }
    }
  }

Step 5 — Cross-account access (dev but NOT prod):

  In each DEV account: create IAM role allowing developers:
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::MGMT-ACCOUNT:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": { "sts:ExternalId": "dev-access-2024" }
      }
    }]
  }

  In MANAGEMENT account: IAM group for developers:
  {
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": [
      "arn:aws:iam::DEV-ACCOUNT-A:role/DeveloperRole",
      "arn:aws:iam::DEV-ACCOUNT-B:role/DeveloperRole"
    ]
    # NOT: prod accounts
  }

  SCP on Production OU — block developer role assumption:
  {
    "Effect": "Deny",
    "Action": "sts:AssumeRole",
    "Resource": "*",
    "Condition": {
      "ArnLike": {
        "aws:PrincipalArn": "arn:aws:iam::*:user/developer-*"
      }
    }
  }
```

💡 **Interview tip:** The Organizations CloudTrail with `--is-organization-trail` creates a single trail that automatically captures events from all current and future member accounts — no per-account configuration needed. The SCP on the root that denies `cloudtrail:DeleteTrail` is the security guardrail — even if someone in a member account has full admin permissions, the SCP overrides them. For cross-account access, the pattern of granting AssumeRole from the management account (not directly between accounts) provides a single choke point where access can be audited and revoked.

---

### Q175 — Kubernetes | Conceptual | Advanced

> Explain **EndpointSlices vs Endpoints**. Scaling problems with Endpoints. How kube-proxy uses EndpointSlices. kube-proxy iptables vs IPVS modes.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md`

```
Original Endpoints resource:
  Created per Service → contains ALL pod IPs for that Service
  apiVersion: v1
  kind: Endpoints
  metadata:
    name: my-service    # same name as Service
  subsets:
  - addresses:
    - ip: 10.244.1.5   # pod 1
    - ip: 10.244.2.3   # pod 2
    - ip: 10.244.3.8   # pod 3
    ... (all 500 pods for a large service)

Scaling problem with original Endpoints:
  1. Single object: entire pod list in ONE Kubernetes object
     500 pods = one 50KB Endpoints object

  2. Any pod change = full object update:
     Pod 1 restarts → its IP changes → entire Endpoints object rewritten
     500 pods × 500 nodes × every pod update = massive etcd churn

  3. kube-proxy watches Endpoints:
     Every change → kube-proxy on ALL nodes receives entire new Endpoints object
     500 nodes × 50KB object × frequent pod changes = huge bandwidth to etcd/API server

  At 1000+ pods per service: Endpoints becomes a bottleneck

EndpointSlices (GA in K8s 1.21):
  → Multiple smaller slice objects replace one large Endpoints object
  → Max 100 endpoints per EndpointSlice (configurable)
  → 500 pods = 5 EndpointSlices

  apiVersion: discovery.k8s.io/v1
  kind: EndpointSlice
  metadata:
    name: my-service-abc12     # auto-generated suffix
    labels:
      kubernetes.io/service-name: my-service
  addressType: IPv4
  endpoints:
  - addresses: ["10.244.1.5"]
    conditions: { ready: true, serving: true, terminating: false }
    nodeName: node-1
    zone: us-east-1a
  ports:
  - port: 8080
    protocol: TCP

  Benefits:
  → Incremental updates: pod change → only update 1 slice (not all 500 pods)
  → Topology hints: zone label enables topology-aware routing
  → Better conditions: ready, serving, terminating (for graceful shutdown)
  → 100× less bandwidth for large services on pod updates

How kube-proxy uses EndpointSlices:
  kube-proxy watches EndpointSlices (not Endpoints)
  For each Service: collects all slices → builds forwarding rules

  iptables mode (default, O(n) lookup):
    kube-proxy writes iptables rules:
    KUBE-SERVICES → KUBE-SVC-xxx (per service)
    KUBE-SVC-xxx → 33% KUBE-SEP-1, 33% KUBE-SEP-2, 33% KUBE-SEP-3
    KUBE-SEP-xxx → DNAT to pod IP

    For 500 pods: 500+ iptables rules to traverse per packet
    O(n) — linear slowdown as services and pods grow
    Suitable for: <1000 services

  IPVS mode (recommended for large clusters, O(1) lookup):
    Uses Linux kernel IPVS (IP Virtual Server) hash tables
    Each Service = IPVS virtual server
    Each pod = IPVS real server under that virtual server
    Hash lookup → O(1) regardless of pod count

    Additional load balancing algorithms:
      rr  (round-robin)
      lc  (least connections)
      sh  (source hash — session affinity)
      wrr (weighted round-robin)

    Enable IPVS:
    kubectl edit configmap kube-proxy -n kube-system
    # Set: mode: "ipvs"
    # Set: ipvs.scheduler: "rr"

  Cilium/eBPF mode (replaces kube-proxy entirely):
    eBPF programs in kernel — even faster than IPVS
    No iptables, no IPVS — direct hash map lookup
    Most performant, requires Cilium CNI
```

💡 **Interview tip:** The bandwidth efficiency of EndpointSlices is the key scalability improvement — with 500 pods and one pod updating frequently, the original Endpoints approach sends the entire 50KB object to every node on every update. EndpointSlices send only the affected 100-pod slice, reducing update bandwidth by 5×. The iptables vs IPVS distinction matters for large clusters — iptables is O(n) per packet, meaning a cluster with 10,000 services and millions of packets per second spends significant CPU just traversing iptables rules. IPVS hash tables are O(1) per lookup regardless of scale.

---

### Q176 — Grafana | Conceptual | Advanced

> Explain **Grafana Tempo** for distributed tracing. Integration with Prometheus and Loki for the 3 pillars. What are **exemplars** and how they link metrics to traces?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana`

```
3 Pillars of observability:
  Metrics:  "What is happening?" — numbers over time (Prometheus)
  Logs:     "What happened exactly?" — events with context (Loki)
  Traces:   "Where did time go?" — distributed request flow (Tempo)

  The power: correlate all three for the same incident

Grafana Tempo:
  → Distributed tracing backend (receives and stores traces)
  → Compatible: OpenTelemetry, Jaeger, Zipkin (drop-in replacement)
  → Stores traces in S3/GCS/Azure Blob (cheap object storage)
  → TraceQL: query language for searching traces
  → Integrates natively with Grafana (Explore → Traces view)

  Key difference from Jaeger:
    Jaeger: requires search backend (Elasticsearch) — expensive
    Tempo: traces stored in object storage (S3) — very cheap
           but: no full-text search (only lookup by trace ID or labels)
    Use Tempo when: you have trace IDs from logs/metrics, don't need full-text search
    Use Jaeger when: need to search traces by arbitrary fields

How the 3 pillars integrate in Grafana:

  Scenario: p99 latency spike at 14:32

  1. METRICS (Prometheus):
     Dashboard shows: p99 latency spikes at 14:32
     "Something is slow — but what?"

  2. LOGS (Loki):
     Click "View logs" for the same time range → Loki query:
     {service="payment"} |= "timeout"
     → See ERROR: "database connection timeout after 5s"
     → Log line contains trace_id="abc123def456"

  3. TRACES (Tempo):
     Click trace_id link → Tempo shows full request trace:
     ├── API Gateway (2ms)
     ├── payment-service (4800ms total)
     │   ├── auth-check (10ms)
     │   ├── RDS query (4780ms) ← HERE is the slow span
     │   └── response (5ms)
     → RDS was slow — not the application code

  All 3 views in Grafana without leaving the UI

Exemplars — linking metrics to traces:

  Without exemplars:
    You see: p99 latency = 4.8 seconds
    You think: which specific requests were slow?
    You search: manually query Tempo hoping to find slow traces
    Time wasted: 15-30 minutes

  With exemplars:
    Prometheus histogram metric stores exemplars on high-value observations
    Exemplar = {trace_id: "abc123", timestamp: 14:32:05, value: 4.8s}
    → The 4.8-second observation is tagged with its trace ID

    In Grafana chart: hover over the latency spike point
    → See small dots on the chart (exemplar markers)
    → Click dot → link to exact trace in Tempo
    → Time to root cause: 30 seconds

  Prometheus configuration for exemplars:
    storage.exemplars-max: 100000    # store up to 100K exemplars
    # (enabled by default in Prometheus 2.26+)

  Application code (emit exemplars):
    from prometheus_client import Histogram
    REQUEST_LATENCY = Histogram('http_request_duration_seconds', ...)

    # In request handler:
    with REQUEST_LATENCY.time():
        result = process_request()
    # Prometheus client auto-attaches current trace_id as exemplar
    # if OpenTelemetry context is present in the thread

  Grafana Explore → Metrics → toggle "Exemplars" button:
    → Charts show colored dots where exemplars exist
    → Each dot = link to specific trace in Tempo
    → Complete: metric → exemplar → trace → spans → slow database call

Tempo setup:
  deployment_type: all
  tempo:
    storage:
      trace:
        backend: s3
        s3:
          bucket: company-tempo-traces
          region: us-east-1

  # Grafana datasource:
  datasources:
  - name: Tempo
    type: tempo
    url: http://tempo:3200
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki      # link traces → Loki logs
        filterByTraceID: true
      tracesToMetrics:
        datasourceUid: prometheus # link traces → Prometheus metrics
```

💡 **Interview tip:** Exemplars are the "magic link" between metrics and traces that most candidates don't know about — they allow you to jump from a latency spike on a Grafana chart directly to the specific slow trace in Tempo with one click. This reduces mean time to root cause from 30 minutes to 30 seconds. Tempo's S3-based storage is the cost advantage over Jaeger + Elasticsearch — storing terabytes of traces in S3 costs cents per GB vs dollars per GB for Elasticsearch. The three-pillar correlation (metric spike → log with trace ID → trace with slow span) is the complete observability story that interviewers want to hear.

---

### Q177 — Terraform | Scenario-Based | Advanced

> Explain **Atlantis** for Terraform GitOps — workflow from PR to apply, locking, plan approval, multiple workspaces, security considerations.

📁 **Reference:** `nawab312/Terraform`, `nawab312/CI_CD`

```
What Atlantis is:
  Open-source, self-hosted Terraform automation tool
  Runs on Kubernetes (or EC2) — receives GitHub/GitLab webhooks
  Automates: plan on PR open, apply on PR merge/comment
  State: remains in your S3 backend (Atlantis doesn't own state)
  Cost: free (vs Terraform Cloud: $20/user/month)

Complete workflow (PR to apply):

  1. Developer creates PR with Terraform changes:
     git checkout -b feature/add-rds
     # modify main.tf
     git push origin feature/add-rds
     # Open Pull Request on GitHub

  2. GitHub webhook → Atlantis:
     Atlantis receives: "PR opened for files matching terraform/**"

  3. Atlantis runs terraform plan automatically:
     # atlantis.yaml controls which projects to plan:
     version: 3
     projects:
     - name: production-rds
       dir: terraform/production/rds
       workspace: default
       autoplan:
         enabled: true
         when_modified: ["*.tf", "../modules/**/*.tf"]

     Plan output posted as GitHub PR comment:
     "atlantis plan for production-rds"
     ──────────────────────────────────
     Plan: 1 to add, 0 to change, 0 to destroy
     + aws_db_instance.analytics
     ──────────────────────────────────

  4. Code review and plan review:
     Team reviews both code AND plan output in the PR
     Plan must be reviewed — accidental destroys caught before apply

  5. Approval and apply:
     Reviewer comments: atlantis apply -p production-rds
     # OR: merge PR (if apply_requirements: merged is configured)

  6. Atlantis runs terraform apply:
     Executes the plan → resources created
     Comments: "Apply complete! Resources: 1 added"
     PR can now be merged

Locking:
  Atlantis locks the project during plan+apply:
  If another PR tries to plan the same project while locked:
  → Atlantis: "Locked by PR #45 until apply completes"
  Prevents concurrent applies (race conditions on state)
  Lock released: after apply or plan discard

  # Discard a plan (release lock without applying):
  atlantis plan discard -p production-rds

Apply requirements (access control):
  # Only allow apply if:
  apply_requirements:
  - approved                  # PR has required approvals
  - mergeable                 # PR has no conflicts
  - undiverged                # branch is up to date with base

Multiple workspaces:
  atlantis.yaml with multiple workspaces:
  projects:
  - name: production
    dir: terraform/
    workspace: production
  - name: staging
    dir: terraform/
    workspace: staging

  Comment: atlantis plan -p production  → runs plan for production workspace
           atlantis plan -p staging     → runs plan for staging workspace
  Both can run in same PR independently

Security considerations:

  Risk 1: Atlantis has broad AWS permissions (it applies infrastructure)
    Mitigation: use IAM roles with least privilege per project
    Mitigation: separate IAM roles per environment (prod vs staging)

  Risk 2: PR from external contributor triggers plan → code execution on Atlantis server
    Mitigation: allowlist: only allow plans from org members
    atlantis.yaml:
    allowed_repos: ["company/*"]
    require_approval_for_plan: true  # external PRs need approval first

  Risk 3: Plan output in PR comment may reveal infrastructure details
    Mitigation: private repositories only
    Mitigation: restrict Atlantis comment to team members

  Atlantis deployment (Kubernetes):
  helm repo add runatlantis https://runatlantis.github.io/atlantis
  helm install atlantis runatlantis/atlantis \
    --set orgAllowlist="github.com/mycompany/*" \
    --set github.user=atlantis-bot \
    --set github.token="$GITHUB_TOKEN" \
    --set github.secret="$WEBHOOK_SECRET" \
    --set serviceAccount.annotations."eks.amazonaws.com/role-arn"="arn:aws:iam::123:role/atlantis"
```

💡 **Interview tip:** The key advantage of Atlantis over manual Terraform runs is that the plan output is visible in the PR review — your entire team sees exactly what infrastructure will change before approving the PR. This turns infrastructure changes into a code review process with full auditability. The locking mechanism is critical for safety — without it, two people could run `terraform apply` on the same project simultaneously and corrupt state. For security, the `allowed_repos` configuration is essential for any open-source or semi-public repository to prevent external PRs from running code on your Atlantis server.

---

### Q178 — Git | Scenario-Based | Advanced

> **15GB monorepo** from large files committed over 2 years. Identify bloat, clean history permanently, prevent recurrence, handle impact on developers.

📁 **Reference:** `nawab312/CI_CD` → `Git` — BFG, git-filter-repo, LFS

```
Step 1 — Identify what is making the repo large:

  # Check total repo size:
  git count-objects -vH
  # size-pack: actual compressed size

  # Find the top 20 largest objects in entire history:
  git rev-list --objects --all \
    | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
    | awk '/^blob/ {print $3, $4}' \
    | sort -rn \
    | head -20
  # Shows: size filename — reveals which files are the problem

  # Or use git-sizer (easier):
  git sizer --verbose
  # Shows: blob count, tree count, largest blobs, deepest trees

  Common culprits:
    *.jar, *.war — build artifacts committed accidentally
    *.zip, *.tar.gz — release archives
    node_modules/ — dependencies committed
    *.pdf, *.psd — large binary assets
    *.mp4, *.mov — videos

Step 2 — Remove large files from history:

  Method A — BFG Repo Cleaner (faster, simpler):
    # Download: https://rtyley.github.io/bfg-repo-cleaner/
    git clone --mirror https://github.com/company/monorepo.git
    cd monorepo.git

    # Remove all files > 1MB ever committed:
    java -jar bfg.jar --strip-blobs-bigger-than 1M

    # Remove specific files by name:
    java -jar bfg.jar --delete-files "*.jar"
    java -jar bfg.jar --delete-folders "node_modules"

    # Clean up and push:
    git reflog expire --expire=now --all
    git gc --prune=now --aggressive
    git push --force

  Method B — git-filter-repo (modern, recommended):
    pip install git-filter-repo
    git clone https://github.com/company/monorepo.git
    cd monorepo

    # Remove path from entire history:
    git filter-repo --path node_modules/ --invert-paths
    git filter-repo --path dist/ --invert-paths
    git filter-repo --path "*.jar" --invert-paths

    # Remove files larger than 1MB:
    git filter-repo --strip-blobs-bigger-than 1M

    # Force push all branches and tags:
    git push origin --force --all
    git push origin --force --tags

Step 3 — Prevent recurrence:

  .gitignore (add immediately):
    node_modules/
    dist/
    build/
    *.jar
    *.war
    *.zip
    *.tar.gz

  Git LFS (for legitimate large files — design assets, datasets):
    git lfs install
    git lfs track "*.psd"
    git lfs track "*.pdf"
    git add .gitattributes    # commit LFS tracking rules
    # Now: large files stored in LFS (S3/external), Git has pointers

  pre-commit hook (block large files at commit time):
    #!/bin/bash
    MAX=1048576   # 1MB
    for f in $(git diff --cached --name-only); do
      size=$(git cat-file -s ":$f" 2>/dev/null || echo 0)
      [ "$size" -gt "$MAX" ] && echo "ERROR: $f too large ($size bytes)" && exit 1
    done

  GitHub Actions check:
    - name: Check for large files
      run: |
        git diff --name-only HEAD~1 HEAD | while read f; do
          [ -f "$f" ] && size=$(wc -c < "$f") || size=0
          [ "$size" -gt 1048576 ] && echo "FAIL: $f is ${size} bytes" && exit 1
        done

Step 4 — Impact on existing developer clones:

  Force push rewrites all commit SHAs → all local clones diverged
  Notify team IMMEDIATELY:

  # Email/Slack message to all developers:
  "IMPORTANT: repository history has been rewritten to remove large files.
   Your local clone is now diverged and cannot be simply pulled.

   Action required (do this TODAY):
   1. Stash or commit any local uncommitted work
   2. Delete your local clone: rm -rf monorepo/
   3. Re-clone fresh: git clone https://github.com/company/monorepo.git

   Any local branches with work in progress:
   git format-patch origin/main..your-branch  # export your commits
   # After re-cloning, apply the patches"
```

💡 **Interview tip:** BFG Repo Cleaner is faster than git-filter-repo for the common use case (remove files by name/size) because it only rewrites commits that touched those files. git-filter-repo is more flexible for complex rewrites. The most important operational step is notifying all developers immediately after the force push — their local clones have diverged and a simple `git pull` will fail or create a mess. The Git LFS migration is the proper long-term solution for legitimate large files (design assets, ML models) — they belong in object storage, not in Git history.

---

### Q179 — AWS + Kubernetes | Scenario-Based | Advanced

> **EKS running out of IP addresses** — `/24` VPC (256 IPs), 200 nodes consuming IPs for pods. Explain why, options to fix, implement prefix delegation.

📁 **Reference:** `nawab312/AWS`, `nawab312/Kubernetes` — VPC CNI, EKS networking

```
Why VPC CNI consumes so many IPs:
  AWS VPC CNI gives each pod a REAL VPC IP address
  Pod IPs are routable within the VPC — no overlay network (NAT-free)

  IP consumption per node (default mode):
    Each node pre-allocates: N + (N × maxPodsPerENI) IP addresses
    Where N = number of ENIs the instance supports

  Example: m5.large
    Max ENIs: 3
    IPs per ENI: 10
    Total IPs: 3 × 10 = 30 IPs pre-allocated per node
    (even if no pods are running on that node!)

  200 nodes × 30 IPs = 6,000 IPs needed
  /24 has 256 IPs → exhausted at ~8 nodes!

  Where IPs go:
    1 IP per node (primary ENI private IP)
    + N IPs pre-warmed per node (WARM_IP_TARGET, default keeps some ready)
    + 1 IP per running pod

Option 1 — Add secondary CIDR blocks to VPC (quick fix):
  aws ec2 associate-vpc-cidr-block \
    --vpc-id vpc-xxx \
    --cidr-block 100.64.0.0/16    # RFC 6598 (shared address space)
  # Add subnets from new CIDR per AZ
  # VPC CNI automatically uses new subnets
  # 100.64.0.0/16 provides 65,536 IPs
  # Trade-off: nodes have IPs from /24, pods from 100.64.x.x (routable internally)

Option 2 — Custom networking (pods use different subnet than nodes):
  aws eks create-addon --cluster-name my-cluster --addon-name vpc-cni
  kubectl set env daemonset aws-node -n kube-system \
    AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true

  Create ENIConfig per AZ:
  apiVersion: crd.k8s.amazonaws.com/v1alpha1
  kind: ENIConfig
  metadata:
    name: us-east-1a
  spec:
    subnet: subnet-pods-us-east-1a    # dedicated pod subnet, /19 = 8190 IPs
    securityGroups: [sg-pods-sg]

  Nodes use /24 subnet for node IPs
  Pods use /19 dedicated pod subnets → separate IP space

Option 3 — IPv4 prefix delegation (most impactful — recommended):

  What prefix delegation does:
    Instead of allocating individual IPs per pod, assigns /28 prefix blocks
    Each /28 = 16 IPs
    m5.large: 3 ENIs × 1 prefix per ENI = 3 × 16 = 48 pods per node
    vs default: 3 × 10 = 30 pods per node (60% more pods per node)
    But: fewer actual VPC IPs consumed per node (1 assignment per prefix)

  Enable prefix delegation:
    kubectl set env daemonset aws-node -n kube-system \
      ENABLE_PREFIX_DELEGATION=true \
      WARM_PREFIX_TARGET=1      # keep 1 warm prefix ready

  Update max pods on new nodes:
    # Instance-specific max pods with prefix delegation:
    m5.large: (3 ENIs - 1) × 14 prefixes × 16 IPs = 448 pods max
    vs default: (3 ENIs - 1) × 10 IPs = 20 pods max

    # Update launch template user data:
    /etc/eks/bootstrap.sh my-cluster \
      --use-max-pods false \
      --kubelet-extra-args '--max-pods=110'

  Result:
    Before: 200 nodes × 30 IPs = 6,000 IPs consumed from VPC
    After:  200 nodes × (1 IP for node + 1 prefix/28 pre-warmed) = ~200 IPs for nodes
            Pods use addresses within /28 blocks (still VPC IPs but aggregated)
    IP efficiency: dramatically improved

  Prefix delegation node groups:
    Must use new nodes (existing nodes not updated automatically)
    Managed node groups: drain old nodes, add new with prefix delegation enabled
    Check: kubectl get node -o json | jq '.items[].metadata.annotations'
    # Look for: k8s.amazonaws.com/eniConfig or VPC CNI annotations

Complete fix recommendation:
  Step 1: add secondary CIDR 100.64.0.0/16 immediately (unblocks new nodes)
  Step 2: create dedicated pod subnets from the new CIDR
  Step 3: enable custom networking + prefix delegation for new node groups
  Step 4: drain old nodes and replace with new prefix-delegation-enabled nodes
  Step 5: monitor with: kubectl get node -o custom-columns="NAME:.metadata.name,CAPACITY:.status.capacity.pods"
```

💡 **Interview tip:** The VPC CNI IP exhaustion problem is very common for teams that start with a small VPC CIDR without planning for Kubernetes pod IP needs. The immediate fix is adding a secondary CIDR (100.64.0.0/16 is the standard choice — it's a large block from RFC 6598 shared address space that doesn't conflict with typical private RFC 1918 ranges). Prefix delegation is the long-term architectural fix that significantly reduces IP consumption — instead of reserving 30 individual IPs per node, you reserve one /28 prefix. The key operational note: existing nodes don't automatically benefit from prefix delegation — you need to drain and replace them.

---

### Q180 — All Topics | System Design | Advanced

> Design **Black Friday capacity management** for 10K→500K RPS spike — predict capacity, pre-scale, auto-scale during event, circuit breakers, real-time monitoring, rollback plan.

📁 **Reference:** All repositories

```
Scale challenge:
  Normal: 10,000 RPS (baseline)
  Black Friday peak: 500,000 RPS (50× multiplier)
  Duration: 6 hours
  Requirement: zero downtime, graceful degradation if needed

Step 1 — Capacity prediction (6 weeks before):

  Historical analysis:
    Last year's Black Friday: 350,000 RPS peak
    Growth rate: 40% YoY increase
    Prediction: 350K × 1.4 = 490K RPS → plan for 500K

  Service-level breakdown:
    API servers:         500K RPS ÷ 2K RPS/pod = 250 pods needed
    Database:            500K RPS × 5% DB-bound = 25K DB queries/sec
    Cache hit rate:      95% target → only 5% DB queries
    DB required:         25K × 0.05 = 1,250 queries/sec → 5× current capacity

  Infrastructure math:
    Current ASG: 20 × c5.xlarge (handles 10K RPS)
    Black Friday: 20 × 50 = 1,000 instances (naively)
    After optimization: 200 instances (with 5× tuning)

Step 2 — Pre-scaling (1 week before):

  Database:
    RDS: upgrade instance class (db.r5.2xlarge → db.r5.16xlarge)
    Read replicas: scale from 2 → 6
    ElastiCache: increase cluster size, pre-warm cache
    Connection pooling: RDS Proxy maxConnections review

  Application tier:
    ASG: set min capacity to 80% of predicted need (400 instances)
    → Don't wait for auto-scaling to ramp up during the event
    aws autoscaling update-auto-scaling-group \
      --auto-scaling-group-name prod-api-asg \
      --min-size 150 \
      --desired-capacity 200

  CDN:
    CloudFront: pre-warm CDN cache with product images
    S3 Transfer Acceleration: enable for static asset delivery
    Estimated: 70% of requests served from CDN (no origin hit)

  Load testing (1 week before):
    Gradually ramp: 10K → 50K → 200K → 500K RPS in staging
    Identify bottlenecks before the real event
    Tools: k6, Locust, AWS Load Testing Solution

Step 3 — Auto-scaling during event:

  Aggressive scale-out settings (day before):
    Target Tracking policy:
      Target: 40% CPU (vs normal 60%) → scale out earlier
    Cooldown: scale-out 30s (vs normal 300s) → react faster
    Instance warm-up: reduce to 60s (pre-baked AMI with app preloaded)

  Predictive scaling:
    aws autoscaling put-scaling-policy \
      --policy-name predictive-black-friday \
      --policy-type PredictiveScaling \
      --predictive-scaling-configuration file://predictive-config.json
    # AWS ML model predicts load and pre-scales before it hits

  Kubernetes HPA:
    scaleTargetRef: deployment
    metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 40    # scale out aggressively at 40%
    behavior:
      scaleUp:
        stabilizationWindowSeconds: 0    # no delay on scale-out
        policies:
        - type: Percent
          value: 100               # can double pods every 30 seconds
          periodSeconds: 30

Step 4 — Circuit breakers and load shedding:

  Circuit breaker (per downstream service):
    Library: resilience4j (Java), hystrix, or Envoy
    Config per service:
      failure_threshold: 50%     # open circuit at 50% failure rate
      wait_duration: 30s         # wait 30s before half-open state
      ring_buffer_size: 10       # evaluate last 10 requests

  Load shedding strategies:
    Priority queuing: checkout > browse > recommendations
      → Drop recommendation API first if overloaded
      → Never drop payment or checkout requests

    Rate limiting at API Gateway:
      Normal users:    1000 req/min
      Anonymous:       100 req/min
      Bot signatures:  10 req/min → likely scrapers

    Feature degradation:
      If cache hit rate < 80%: disable personalization (expensive DB queries)
      If DB latency > 500ms: show cached product prices (accept stale data)
      If inventory service down: show "add to cart" but skip real-time stock check

Step 5 — Real-time monitoring during event:

  War room dashboard (Grafana):
    Panel 1: RPS (current vs capacity)
    Panel 2: p99 latency (SLO: < 200ms)
    Panel 3: Error rate (SLO: < 0.1%)
    Panel 4: EC2 instance count (current scaling state)
    Panel 5: Database connection pool utilisation
    Panel 6: Cache hit rate

  Alerting thresholds (relaxed for Black Friday):
    Normal: alert at p99 > 100ms
    Black Friday: alert at p99 > 500ms (expect higher due to load)
    Circuit breaker open: PAGE IMMEDIATELY
    Error rate > 1%: PAGE IMMEDIATELY

  On-call team:
    War room: all-hands (SRE, backend, DBA, DevOps)
    Rotation: 2-hour shifts (not 6 continuous hours)
    Escalation: VP Engineering paged if error rate > 2% for > 5 minutes

Step 6 — Rollback plan:

  L1: Feature flag rollback (seconds):
    Disable problematic feature → instant (no deployment needed)
    LaunchDarkly or AWS AppConfig: flip flag for 500K users instantly

  L2: Service version rollback (2 minutes):
    kubectl rollout undo deployment/payment-api
    # Or: ECS force new deployment with previous task definition

  L3: Traffic shifting (5 minutes):
    Route53: switch weight from new deployment → previous stable version
    ALB weighted target groups: 100% → old version

  L4: Full traffic drop to maintenance mode (last resort):
    CloudFront: serve static "maintenance" page
    ALB: redirect all traffic to maintenance endpoint
    Message: "We're experiencing high demand — please try in 10 minutes"

  Post-event:
    Debrief within 24 hours: what worked, what didn't
    Scale back down (save costs): reduce ASG to normal size
    Write blameless post-mortem for any incidents
    Update runbook with lessons learned
```

💡 **Interview tip:** The most impressive part of this answer is the pre-scaling strategy — don't rely on auto-scaling to react to Black Friday traffic; pre-scale to 80% of predicted need before the event starts. Auto-scaling has latency (instance launch + warm-up takes 2-5 minutes) and you can't afford to be scaling reactively when RPS is spiking 50×. The load shedding priority hierarchy (checkout > browse > recommendations) shows mature thinking — you accept degraded features to protect the core revenue path. Mention the load test 1 week before the event as the single most important preparation step — it reveals surprises before they affect real revenue.

---

*DevOps/SRE Interview Questions Q136–Q180 — nawab312 GitHub repositories*
*Total question bank: Q1–Q180*
