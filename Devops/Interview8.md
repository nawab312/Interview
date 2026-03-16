# DevOps / SRE Interview Questions — Q316–Q360 Enhanced
## Key Points + Interview Tips Added to Every Question

---

### Q316 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `Gateway API`** — the next generation replacement for Ingress. What problems with the Ingress resource does Gateway API solve? What are the 4 core resources and how does role separation work?

#### Key Points to Cover:
```
Problems with Ingress that Gateway API solves:
  → Ingress is too simple: only supports HTTP/HTTPS (no TCP, UDP, gRPC natively)
  → Vendor annotations chaos: nginx.ingress.kubernetes.io/*, traefik.io/* etc.
    → Non-portable, vendor-locked configs
  → No role separation: one monolithic resource mixes infra + app concerns
  → Limited routing: only host/path matching, no header/weight-based routing
  → No multi-protocol support in a single resource

4 Core Gateway API Resources:

  GatewayClass:
    → Defines the controller implementation (like IngressClass)
    → Created by: infrastructure provider (cloud team, platform team)
    → Example: AWS Load Balancer Controller, nginx, Istio
    kind: GatewayClass
    metadata:
      name: aws-alb
    spec:
      controllerName: ingress.k8s.aws/alb

  Gateway:
    → Instance of a GatewayClass (the actual load balancer config)
    → Created by: cluster operator
    → Defines: listeners (ports, protocols, TLS)
    kind: Gateway
    metadata:
      name: prod-gateway
      namespace: infra
    spec:
      gatewayClassName: aws-alb
      listeners:
      - name: https
        port: 443
        protocol: HTTPS
        tls:
          certificateRefs:
          - name: prod-cert

  HTTPRoute:
    → Routing rules for HTTP traffic
    → Created by: application developer
    → Attaches to a Gateway, routes to Services
    kind: HTTPRoute
    metadata:
      name: payment-route
      namespace: payment-team
    spec:
      parentRefs:
      - name: prod-gateway
        namespace: infra
      rules:
      - matches:
        - path:
            type: PathPrefix
            value: /api/payments
        backendRefs:
        - name: payment-service
          port: 8080
          weight: 90
        - name: payment-service-canary
          port: 8080
          weight: 10    # traffic splitting — impossible with Ingress

  TCPRoute:
    → Routing for raw TCP traffic (databases, custom protocols)
    → Same pattern as HTTPRoute but for TCP

Role Separation (key differentiator):
  Infrastructure Provider → creates GatewayClass (once, global)
  Cluster Operator        → creates Gateway (per-cluster or per-env)
  Application Developer   → creates HTTPRoute/TCPRoute (per-app)

  → Each team only manages their layer
  → App developers don't need cluster-admin to configure routing
  → Platform team controls what infrastructure is available

Traffic splitting example (impossible with vanilla Ingress):
  backendRefs:
  - name: app-v1
    weight: 80
  - name: app-v2
    weight: 20
```

> 💡 **Interview tip:** The key insight interviewers look for: Gateway API solves the **role separation problem**. With Ingress, developers need to add vendor-specific annotations that touch infrastructure-level concerns. With Gateway API, the platform team owns Gateway, and developers own HTTPRoute — each team works in their own namespace with their own objects. Also mention that HTTPRoute supports **traffic splitting and header-based routing natively**, making canary deployments declarative without vendor plugins. If asked for a comparison: "Gateway API is to Ingress what Deployments were to ReplicationControllers — same goal, far more expressive."

---

### Q317 — AWS | Scenario-Based | Advanced

> ECS Fargate inter-service latency increased from 2ms to 200ms. No code changes. CPU and memory are normal. Walk through diagnosis.

#### Key Points to Cover:
```
Systematic diagnosis — start with symptoms, isolate the layer:

Step 1 — ALB Access Logs (is ALB adding latency?):
  → Enable ALB access logs to S3
  → Key fields: target_processing_time, response_processing_time, request_processing_time
  → If target_processing_time is high → latency is in the backend service
  → If request_processing_time is high → latency is at ALB/connection level
  aws logs filter-log-events --log-group-name /aws/alb/prod \
    --filter-pattern '{ $.target_processing_time > 0.1 }'

Step 2 — X-Ray Traces (which microservice call is slow?):
  → X-Ray shows distributed trace: Service A → Service B → RDS
  → Identify: which segment has high latency?
  → Check: is it the network hop or the processing time?
  → Service Map in console shows latency per service

Step 3 — Service Discovery / Cloud Map:
  → If services use Cloud Map for DNS: check DNS resolution time
  → DNS resolution slow → check Cloud Map health checks
  → Misconfigured TTL → stale IP → connection to wrong/dead task
  → Test: from inside task: time nslookup my-service.prod.svc

Step 4 — DNS Resolution inside Fargate:
  → Fargate tasks use VPC DNS (Route 53 Resolver)
  → High DNS query rate → throttling → latency spikes
  → Check: VPC DNS query metrics in CloudWatch
  → Fix: implement DNS caching inside containers (nscd)
  → ndots setting: if ndots:5, a query for "backend" tries 5 search domains first

Step 5 — VPC Flow Logs (are packets being dropped?):
  → Enable VPC Flow Logs on ENI level
  → Look for: REJECT records between service IPs
  → Security group change might block traffic → retry = latency
  → Check: flow log records with action=REJECT

Step 6 — Fargate Task Networking:
  → Each Fargate task gets its own ENI (awsvpc mode)
  → Check: ENI attachment delay (Fargate scale-out adds new ENI)
  → New tasks: cold start = slow first requests (image pull time)
  → Check: ECS service events for task replacement events
  aws ecs describe-services --cluster prod --services my-service \
    --query 'services[0].events[:10]'

Step 7 — Connection Pool exhaustion:
  → If latency is in connection waiting (X-Ray shows "pending" time)
  → Service B has connection pool full → requests queue
  → Check: active connections metric per task
  → Fix: increase connection pool size or scale out

Root cause checklist:
  High ALB latency        → SSL negotiation, connection draining
  High DNS latency        → Cloud Map TTL, VPC DNS throttling
  High service latency    → Connection pool, downstream DB, CPU throttle
  Packet drops/rejects    → Security group change, NACL change
  Intermittent spikes     → Fargate cold starts, task replacement
```

> 💡 **Interview tip:** Structure your answer as **layers**: client → ALB → service discovery → task → backend. At each layer there's a specific AWS tool to check. The X-Ray service map is the fastest way to pinpoint which hop is slow — if you have it instrumented. If not, mention that **this incident is the business case for enabling X-Ray**. A common gotcha: latency that correlates with deployments but isn't caused by the code change — it's caused by **connection draining on the old tasks** as ALB waits the deregistration delay (default 300 seconds) before fully routing to new tasks.

---

### Q318 — Linux / Bash | Conceptual | Advanced

> Explain Linux CFS Scheduler, nice values, ionice, real-time scheduling policies, and how Kubernetes CPU throttling interacts with CFS bandwidth control.

#### Key Points to Cover:
```
CFS — Completely Fair Scheduler:
  → Default Linux scheduler since kernel 2.6.23
  → Goal: give every process a "fair" share of CPU proportional to weight
  → Tracks vruntime (virtual runtime) per process — ns of CPU used,
    normalized by process weight
  → Always picks the process with the LOWEST vruntime to run next
  → Uses a red-black tree (self-balancing BST) as the run queue
    → leftmost node = lowest vruntime = next to run → O(log N) pick
  → Time slice is dynamic: proportional to weight, shrinks as more processes compete
  → No starvation: sleeping processes don't accumulate vruntime

nice values:
  → Range: -20 (highest priority) to +19 (lowest priority), default 0
  → Maps to weight: nice 0 = weight 1024, each step ~10% more/less CPU
  → nice -20 ≈ 3x more CPU than nice 0; nice +19 ≈ 5x less CPU
  → Set with: nice -n 10 command OR renice -n 10 -p <pid>
  → Only root can set negative nice values
  → nice controls CPU scheduling ONLY — not I/O

ionice:
  → Controls I/O scheduling priority (separate from CPU scheduling)
  → Three classes:
      Class 1 (Realtime):   gets I/O first, can starve others
      Class 2 (Best-effort): default, takes a priority 0–7
      Class 3 (Idle):       only gets I/O when nothing else needs disk
  → Set with: ionice -c 3 -p <pid>
  → Use case: backup jobs — nice +19 AND ionice -c 3 = zero impact

Scheduling Policies:
  SCHED_OTHER (normal):
    → CFS-based, used by all userspace processes
    → Weight determined by nice value
    → Not real-time — can be preempted by any RT task

  SCHED_FIFO (real-time):
    → First In First Out — runs until it voluntarily yields or blocks
    → NO time slice — can monopolize CPU forever
    → Priority 1–99 (higher = runs first)
    → Risk: a buggy SCHED_FIFO process at priority 99 can hang the system
    → Use: audio servers (PulseAudio), kernel threads
    → Set with: chrt -f 50 <command>

  SCHED_RR (real-time):
    → Round Robin with time slice
    → Same priority model (1–99) but processes at same priority share CPU
    → More forgiving than FIFO

  RT vs normal: ANY RT task preempts ALL SCHED_OTHER tasks
  RT priority 1 > nice -20

Kubernetes CPU throttling + CFS Bandwidth Control:
  → K8s CPU limits implemented via CFS bandwidth control (cgroups)
  → Two cgroup parameters:
      cpu.cfs_period_us:  the period window (default 100ms = 100,000 µs)
      cpu.cfs_quota_us:   how many µs of CPU per period
  → Example: limits.cpu = 500m (0.5 cores)
      quota = 0.5 × 100,000 = 50,000 µs
      → container gets 50ms of CPU per 100ms window
  → If container uses quota before period ends → THROTTLED
      → all processes in cgroup sleep until next period starts
      → shows up as: container.cpu.throttled_time in metrics
  → Throttling is NOT visible as high CPU — it looks like latency
  → Common mistake: tight CPU limit → bursty JVM GC throttled → tail latency
  → cpu.shares implements CPU requests (soft limit, weight-based)
  → Metric to watch: container_cpu_cfs_throttled_periods_total
```

> 💡 **Interview tip:** Most candidates describe CFS as "round robin with priorities" — that's wrong. The key insight is the **red-black tree + vruntime** model: always picks the leftmost node (lowest vruntime), making the algorithm O(log N) and mathematically fair. For Kubernetes, the killer answer: **CPU throttling looks like latency, not high CPU usage** — a container at 30% CPU can still be heavily throttled if it bursts above its limit within the 100ms window. Mention `container_cpu_cfs_throttled_periods_total` as the metric to watch. That separates SREs who've debugged this in production from those who only read about it.

---

### Q319 — Terraform | Scenario-Based | Advanced

> Implement `null_resource` and `terraform_data` for running local scripts as part of infrastructure provisioning. Give 3 real-world use cases and explain the `triggers` mechanism.

#### Key Points to Cover:
```
null_resource vs terraform_data:
  null_resource: older approach, requires "null" provider
  terraform_data: native TF 1.4+, no extra provider needed
  Both: run provisioners (local-exec, remote-exec) as part of apply

triggers mechanism:
  → Map of values; when any value changes → resource is re-created
  → Re-creation = provisioner runs again
  → Use: force re-run when dependent resource changes
  → triggers = { always_run = timestamp() } → run on EVERY apply

Use Case 1 — Database migration after RDS creation:
resource "terraform_data" "db_migration" {
  triggers_replace = {
    rds_endpoint = aws_db_instance.main.endpoint
    schema_hash  = filemd5("${path.module}/migrations/schema.sql")
  }

  provisioner "local-exec" {
    command = <<-EOT
      echo "Running migrations on ${aws_db_instance.main.endpoint}"
      psql -h ${aws_db_instance.main.endpoint} \
           -U ${var.db_user} \
           -d ${var.db_name} \
           -f ${path.module}/migrations/schema.sql
    EOT
    environment = {
      PGPASSWORD = var.db_password
    }
  }

  depends_on = [aws_db_instance.main]
}

Use Case 2 — Wait for application health after deployment:
resource "terraform_data" "wait_for_health" {
  triggers_replace = {
    service_arn = aws_ecs_service.app.id
  }

  provisioner "local-exec" {
    command = <<-EOT
      echo "Waiting for ECS service to stabilize..."
      aws ecs wait services-stable \
        --cluster ${var.cluster_name} \
        --services ${aws_ecs_service.app.name} \
        --region ${var.region}
      echo "Service is healthy"
    EOT
  }

  depends_on = [aws_ecs_service.app]
}

Use Case 3 — Generate config file and upload to S3:
resource "terraform_data" "upload_config" {
  triggers_replace = {
    config_hash = sha256(local.rendered_config)
    bucket_name = aws_s3_bucket.config.bucket
  }

  provisioner "local-exec" {
    command = <<-EOT
      cat > /tmp/app-config.json << 'EOF'
      ${local.rendered_config}
      EOF
      aws s3 cp /tmp/app-config.json \
        s3://${aws_s3_bucket.config.bucket}/config/app.json
      rm /tmp/app-config.json
    EOT
  }

  depends_on = [aws_s3_bucket.config]
}

When NOT to use null_resource/terraform_data:
  → If a Terraform resource/data source can do it natively — use that
  → For complex provisioning — use Ansible, Chef, or user_data scripts
  → null_resource doesn't appear in TF graph cleanly — use sparingly
```

> 💡 **Interview tip:** Terraform is fundamentally a **declarative** tool, but null_resource/terraform_data lets you inject **imperative** steps (scripts) into a declarative workflow. This is a pragmatic escape hatch — not the ideal solution. The interviewer is really asking: "Do you know when to break the declarative model?" The trigger mechanism is the critical concept: without triggers pointing to the dependency, Terraform doesn't know when to re-run the script. A common mistake: using `timestamp()` as a trigger — this runs the script on every `terraform apply` even when nothing changed. Use content hashes or specific resource attributes instead.

---

### Q320 — Prometheus | Conceptual | Advanced

> Explain Prometheus AlertManager routing in depth — routing tree, group_by, group_wait, group_interval, repeat_interval. Write a routing tree for critical/warning alerts.

#### Key Points to Cover:
```
AlertManager routing tree:
  → Single root route catches all alerts
  → Routes evaluated top-to-bottom, first match wins
  → Child routes inherit parent config unless overridden
  → continue: true → keeps evaluating after match (for multi-routing)

Key timing parameters:
  group_by: ["alertname", "cluster"]
    → Alerts sharing same label values are grouped together
    → One notification per group (not per alert)
    → Prevents: 100 separate Slack messages when 100 pods fail
    → Best practice: group by alertname + cluster + namespace

  group_wait: 30s
    → Wait this long before sending the FIRST notification for a new group
    → Allows more alerts to arrive and be bundled together
    → Critical alerts: short (30s), Warning: longer (5m)

  group_interval: 5m
    → After initial notification, wait this long before sending
      updates for the same group (if new alerts join or resolve)
    → Prevents: notification storm when alerts flap

  repeat_interval: 4h
    → If the group is still firing and unchanged, re-notify after this duration
    → Critical: shorter (1h), Warning: longer (24h)

Complete routing tree:
global:
  smtp_from: 'alertmanager@company.com'
  resolve_timeout: 5m

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'namespace']
    # When critical fires, suppress matching warning alerts

route:
  receiver: 'default-slack'
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  routes:
  # Critical → PagerDuty (fast)
  - match:
      severity: critical
    receiver: 'pagerduty-critical'
    group_wait: 30s
    group_interval: 1m
    repeat_interval: 1h
    continue: true   # also send to Slack

  # Critical AND production → wake someone up
  - match_re:
      severity: critical
      cluster: prod-.*
    receiver: 'pagerduty-prod'
    group_wait: 0s   # immediate

  # Warning → Slack only
  - match:
      severity: warning
    receiver: 'slack-warnings'
    group_wait: 5m
    group_interval: 10m
    repeat_interval: 24h

  # Maintenance window suppression (via time_intervals)
  - receiver: 'null'
    active_time_intervals: ['maintenance-window']
    match_re:
      severity: warning|info

receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
    - routing_key: 'YOUR_PD_KEY'
      description: '{{ .CommonAnnotations.summary }}'
      severity: '{{ .CommonLabels.severity }}'

  - name: 'slack-warnings'
    slack_configs:
    - channel: '#alerts-warning'
      api_url: 'https://hooks.slack.com/...'
      title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
      text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'null'  # silently discard

time_intervals:
  - name: maintenance-window
    time_intervals:
    - weekdays: ['saturday', 'sunday']
      times:
      - start_time: '02:00'
        end_time: '06:00'
```

> 💡 **Interview tip:** The most misunderstood concept: **inhibit_rules vs silences vs mute_time_intervals**. Inhibit rules suppress one alert when another fires (e.g., suppress all warnings when a critical fires for the same service — if the service is down, you don't need warning-level noise). Silences are one-time manual suppression with an expiry. Time intervals are recurring scheduled suppression. Also emphasize: **group_wait is not a delay — it's a batching window**. It waits for more alerts to arrive so they can be sent as a single notification rather than individually. This is critical for preventing alert storms.

---

### Q321 — Kubernetes | Troubleshooting | Advanced

> Kubernetes shows PIDPressure on nodes. Pods are evicted and new ones can't schedule. What causes it, and how do you investigate and fix?

#### Key Points to Cover:
```
What is PIDPressure:
  → Node condition: True when available PIDs on a node fall below threshold
  → PIDs = process IDs — Linux kernel limit, each thread consumes one PID
  → Default kernel limit: /proc/sys/kernel/pid_max (typically 32768 or 4194304)
  → kubelet eviction threshold: pid.available < 1000 (default)
    → Configurable in kubelet config: evictionHard.pid.available

Why it causes problems:
  → New processes cannot be created (fork() fails with EAGAIN)
  → Pod startup fails (container runtime can't fork)
  → SSH into node may fail (no PIDs to create shell)
  → systemd daemons can't restart

Node-level PID limits:
  kernel.pid_max: total PIDs for entire node
  kubelet eviction threshold: kubelet evicts pods when available PIDs < threshold
  kubelet PID reservations: --system-reserved=pid=500 --kube-reserved=pid=200

Pod/Container-level PID limits:
  spec:
    containers:
    - name: app
      resources:
        limits:
          # PID limit per container (Kubernetes 1.20+)
  
  OR via LimitRange:
  apiVersion: v1
  kind: LimitRange
  spec:
    limits:
    - type: Container
      max:
        cpu: "2"
      # Note: PID limits in LimitRange need feature gate

Investigation:
  # Check node condition:
  kubectl describe node <node-name> | grep -A5 "Conditions"
  # Look for: PIDPressure True

  # Check current PID usage on the node:
  cat /proc/sys/kernel/pid_max              # max PIDs
  cat /proc/loadavg                         # 5th field: total threads/processes
  ps aux | wc -l                            # running processes

  # Find PID-hungry pods:
  kubectl top pods -A --sort-by=cpu
  # Then SSH to node:
  ps --ppid <container-main-pid> | wc -l   # threads per container

  # Find process spawning abuse:
  ps aux | awk '{print $1}' | sort | uniq -c | sort -rn | head
  # Which user/app is spawning the most processes?

  # Check for fork bombs:
  watch -n1 "ps aux | wc -l"              # count growing rapidly?

Common culprits:
  → Shell scripts in containers: each shell command = new PID
  → Java apps with many threads
  → Log rotation scripts spawning many subprocesses
  → CI/CD runners (each build = many processes)
  → Bad scripts: while true; do sleep 1 & done   # fork bomb

Fix:
  Immediate: evict the offending pod
    kubectl delete pod <pid-heavy-pod> -n <ns>
  
  Medium-term: set PID limits per namespace via LimitRange
  Long-term: fix the application spawning too many threads/processes
  Node: increase kernel.pid_max if legitimate workload needs it
    sysctl -w kernel.pid_max=65536
```

> 💡 **Interview tip:** PIDPressure is rare compared to memory/CPU pressure, which is why it catches engineers off guard. The key insight: **PIDs are a resource like memory** — each thread, each process, each shell command consumes one. Java applications with many threads are a common culprit — a JVM with 200 threads consumes 200 PIDs. The investigation command most candidates miss: `ps aux | awk '{print $1}' | sort | uniq -c | sort -rn` — this groups processes by user, instantly showing which user (which container runtime user) is spawning the most. Also mention the practical impact: when PIDPressure hits, **SSH into the node may fail** because the shell can't fork — making diagnosis harder. Always have out-of-band access (AWS SSM Session Manager) for this reason.

---

### Q322 — Git | Conceptual | Advanced

> Explain `git bundle`. When would you use it? Give a real-world DevOps scenario for air-gapped environments. Write the exact commands.

#### Key Points to Cover:
```
What is git bundle:
  → A single file containing an entire git repository (or part of it)
  → Contains: commits, refs, pack objects — everything needed
  → Can be used as a remote (git fetch, git clone from bundle file)
  → Essentially: a portable git remote in a file

When regular git clone/push fails:
  → No network access between source and destination
  → Air-gapped networks (military, financial, secure environments)
  → CI/CD pipeline on offline build server
  → Transferring to a partner who can't access your git server
  → USB/physical media transfer

Commands — Creating and Using Bundles:

  # Create a full bundle (everything):
  git bundle create repo.bundle --all
  # Creates: repo.bundle with all branches and tags

  # Create a partial bundle (only main branch):
  git bundle create main.bundle main

  # Create incremental bundle (changes since last bundle):
  git bundle create updates.bundle ^v1.0 HEAD
  # ^ prefix = "exclude commits reachable from v1.0"
  # Contains: only commits AFTER v1.0

  # Verify a bundle is valid:
  git bundle verify repo.bundle

  # Clone from a bundle:
  git clone repo.bundle my-project
  cd my-project

  # Fetch updates from a bundle:
  git fetch /path/to/updates.bundle main:main

  # Use bundle as a remote:
  git remote add offline /path/to/repo.bundle
  git fetch offline

Real-world air-gapped CI/CD scenario:
  Secure environment setup:
  1. Developer workstation (internet access):
     git bundle create app.bundle --all
     # Transfer to USB/secure file transfer
     cp app.bundle /secure-transfer/

  2. Air-gapped CI/CD server (no internet):
     git clone /secure-transfer/app.bundle /opt/repos/app
     # Now build from local bundle-cloned repo

  Incremental updates (ongoing workflow):
  1. On workstation: git pull (get new commits)
  2. git bundle create delta.bundle ^$(cat last-bundle-tag) HEAD
  3. git tag last-bundle-tag HEAD
  4. Transfer delta.bundle to air-gapped server
  5. On air-gapped server: git fetch /transfer/delta.bundle main:main
  6. Run CI/CD build

  Comparison with alternatives:
  git archive → snapshot of files, no git history
  git bundle  → full git repository with history
  rsync       → files only, no git awareness
```

> 💡 **Interview tip:** Most engineers never use `git bundle` in normal environments — the question tests **breadth of git knowledge** and whether you've worked in regulated/secure environments. The key differentiator from `git archive`: bundle preserves the **entire git history and structure**, not just a file snapshot. You can `git clone` from a bundle and get a fully functional git repo with complete history. If you've worked in financial services, defense, or healthcare, mention that air-gapped deployments are a compliance requirement — code must never leave the secure perimeter via network, so physical media transfers with git bundles is the standard approach.

---

### Q323 — AWS | Conceptual | Advanced

> Explain AWS Cognito — User Pools vs Identity Pools. Walk through the complete token flow from user login to authenticated API call via API Gateway.

#### Key Points to Cover:
```
Cognito User Pool:
  → Authentication service (WHO are you?)
  → Manages: user directory, sign-up, sign-in, MFA, password reset
  → Issues: JWT tokens (id_token, access_token, refresh_token)
  → Integrates: social providers (Google, Facebook), SAML, OIDC
  → Like: managed Auth0 or Okta

Cognito Identity Pool:
  → Authorization service (WHAT can you access in AWS?)
  → Exchanges: any identity (User Pool token, Google, anonymous)
    → for temporary AWS credentials (STS AssumeRoleWithWebIdentity)
  → Issues: AWS access key + secret + session token
  → Allows: direct AWS SDK calls from browser/mobile
  → Like: STS + federated identity management

Token types from User Pool:
  id_token:
    → Contains: user attributes (email, name, custom attributes)
    → Used for: identifying the user in your application
    → Verified by: API Gateway as Cognito Authorizer
    → Audience (aud): your App Client ID

  access_token:
    → Contains: scopes, user's Cognito groups
    → Used for: calling Cognito APIs (get user info, etc.)
    → Audience: Cognito token endpoint (not your app client)

  refresh_token:
    → Long-lived (default: 30 days)
    → Used: to get new id_token/access_token without re-login
    → Never sent to API — only used with Cognito token endpoint

Complete flow — User Login to API Call:

  1. User submits credentials to Cognito User Pool:
     POST https://cognito-idp.us-east-1.amazonaws.com/
     Body: { AuthFlow: "USER_PASSWORD_AUTH",
             AuthParameters: { USERNAME: "user@example.com", PASSWORD: "..." } }

  2. Cognito returns tokens:
     { id_token: "eyJ...", access_token: "eyJ...", refresh_token: "eyJ..." }

  3. Frontend stores tokens (secure httpOnly cookie or memory)

  4. API call with id_token:
     GET https://api.example.com/orders
     Authorization: Bearer <id_token>

  5. API Gateway (Cognito Authorizer):
     → Validates id_token signature against User Pool public keys
     → Checks: token not expired, audience matches, issuer matches
     → Extracts: claims (user email, groups)
     → If valid: passes request to Lambda/backend

  6. Backend receives request:
     → API Gateway injects: $context.authorizer.claims.email
     → Backend knows: authenticated user identity

Token refresh flow:
  → Frontend detects id_token expiry (check exp claim)
  → Calls Cognito with refresh_token
  → Gets new id_token + access_token (refresh_token unchanged)
  → Continues API calls seamlessly

Identity Pool flow (for direct AWS access):
  id_token → Cognito Identity Pool → STS → temp AWS creds
  → Frontend uses AWS SDK with temp creds directly
  → Use case: S3 uploads from browser, DynamoDB direct access
```

> 💡 **Interview tip:** The most common confusion: **User Pool vs Identity Pool are not alternatives — they're complementary**. User Pool = authentication (JWT tokens). Identity Pool = AWS authorization (STS credentials). Most web apps only need User Pool + API Gateway. Identity Pool is for apps that need to call AWS services (S3, DynamoDB) directly from the client. Also clarify which token to use where: **id_token** is passed to your API Gateway (contains user info). **access_token** is for calling Cognito's own APIs. Many engineers mix these up and wonder why their API Gateway returns 401.

---

### Q324 — Jenkins | Conceptual | Advanced

> Explain Jenkins Pipeline Durability settings — PERFORMANCE_OPTIMIZED, SURVIVABLE_NONATOMIC, MAX_SURVIVABILITY. What are the trade-offs?

#### Key Points to Cover:
```
What Pipeline Durability Controls:
  → How often Jenkins persists pipeline state to disk
  → Determines: can a pipeline survive Jenkins master restart mid-build?
  → Trade-off: more durability = more disk I/O = slower pipelines

Three Durability Levels:

  PERFORMANCE_OPTIMIZED (fastest):
    → Minimal disk writes during pipeline execution
    → If Jenkins restarts mid-build: pipeline IS LOST
    → Best for: fast builds (<5 min), non-critical pipelines
    → Speed: up to 3x faster than MAX_SURVIVABILITY
    → Use when: build can be re-triggered easily, speed matters

  SURVIVABLE_NONATOMIC (middle ground):
    → Writes frequently but without atomic guarantees
    → If Jenkins restarts mid-build: pipeline USUALLY survives
    → Small risk of data corruption at exact crash moment
    → Best for: most production pipelines
    → Default recommendation for balanced workloads

  MAX_SURVIVABILITY (safest, slowest):
    → Writes every step atomically to disk
    → If Jenkins restarts mid-build: pipeline resumes from last step
    → Significant performance penalty (many small disk writes)
    → Best for: long-running pipelines (>30 min), deployments you can't re-run
    → Use when: pipeline has external side effects (API calls, DB changes)

Setting durability:
  // Pipeline level:
  properties([
    durabilityHint('PERFORMANCE_OPTIMIZED')
  ])

  // Global: Manage Jenkins → System Configuration → Pipeline Speed/Durability

  // Folder/multibranch level:
  // Set in Folder configuration

Mid-build restart behavior:
  → Jenkins master restarts (maintenance, crash, OOM kill)
  → PERFORMANCE_OPTIMIZED: builds gone, must re-trigger
  → MAX_SURVIVABILITY: builds resume from last checkpoint
  → Resume point: last completed step, not last completed stage

When to lower durability (performance-critical):
  → CI tests that run hundreds of times per day
  → Pipeline steps that are fast and idempotent
  → Kubernetes/cloud environments where master restarts are rare
  → Cost: wasted compute for re-runs vs cost of extra disk I/O

When to keep MAX_SURVIVABILITY:
  → Production deployments with manual approval gates
  → Pipelines that do database migrations
  → Any pipeline step that cannot be safely re-run (payment processing)
  → Pipelines that take hours (build, test, deploy cycle)
```

> 💡 **Interview tip:** This question tests whether you understand **Jenkins internals and operational trade-offs**. The key insight: durability is a **performance vs resilience knob**. Most teams don't touch this setting and leave it at default, which is why asking about it separates engineers who've tuned Jenkins at scale. For Kubernetes-hosted Jenkins (JenkinsX, Jenkins on EKS): pods restart more frequently than bare-metal servers, so durability settings matter more. If a Jenkins pod gets OOMKilled mid-deployment-pipeline, MAX_SURVIVABILITY means the deployment continues — PERFORMANCE_OPTIMIZED means it's lost and you potentially have partial deployment state.

---

### Q325 — ELK Stack | Scenario-Based | Advanced

> Implement complete ELK Stack security: TLS between components, RBAC, audit logging, and field-level security for PII masking.

#### Key Points to Cover:
```
Component security overview:
  Filebeat → Logstash: mTLS
  Logstash → Elasticsearch: TLS + Basic Auth
  Elasticsearch → Kibana: TLS + API Key or Basic Auth
  All secured via Elastic Stack Security (X-Pack, free since 7.x)

Step 1 — Generate certificates (using elasticsearch-certutil):
  # Generate CA:
  elasticsearch-certutil ca --out elastic-stack-ca.p12

  # Generate certs for each node:
  elasticsearch-certutil cert \
    --ca elastic-stack-ca.p12 \
    --name elasticsearch \
    --dns elasticsearch.company.internal \
    --out elasticsearch.p12

Step 2 — Enable TLS in elasticsearch.yml:
  xpack.security.enabled: true
  xpack.security.transport.ssl.enabled: true
  xpack.security.transport.ssl.keystore.path: elasticsearch.p12
  xpack.security.http.ssl.enabled: true
  xpack.security.http.ssl.keystore.path: elasticsearch.p12

Step 3 — RBAC — developer access to own service logs only:
  # Create index pattern per service:
  # payment-service-* → payment team only
  # auth-service-*    → auth team only

  # Create role via API:
  PUT /_security/role/payment-developer
  {
    "indices": [{
      "names": ["payment-service-*"],
      "privileges": ["read", "view_index_metadata"],
      "field_security": {
        "grant": ["@timestamp", "message", "level", "service"],
        "except": ["user.email", "payment.card_number"]  # mask PII
      }
    }]
  }

  # Create user:
  PUT /_security/user/alice
  {
    "password": "...",
    "roles": ["payment-developer"]
  }

Step 4 — Field-Level Security (mask PII):
  # Option 1: Exclude fields entirely (above — user can't see them)
  # Option 2: Anonymize via ingest pipeline before indexing:
  PUT /_ingest/pipeline/mask-pii
  {
    "processors": [
      {
        "script": {
          "source": """
            if (ctx.containsKey('email')) {
              ctx.email = ctx.email.replaceAll('(?<=.{3}).(?=.*@)', '*');
            }
            if (ctx.containsKey('card_number')) {
              ctx.card_number = 'XXXX-XXXX-XXXX-' + ctx.card_number.substring(15);
            }
          """
        }
      }
    ]
  }

Step 5 — Audit Logging:
  # elasticsearch.yml:
  xpack.security.audit.enabled: true
  xpack.security.audit.logfile.events.include:
    - authentication_success
    - authentication_failed
    - access_denied
    - document_access_granted  # who searched what

  # Audit logs written to: logs/elasticsearch_audit.json
  # Can be indexed back into ES for analysis (separate cluster recommended)

Step 6 — Logstash → Elasticsearch with credentials:
  output {
    elasticsearch {
      hosts => ["https://elasticsearch:9200"]
      ssl  => true
      cacert => "/etc/logstash/certs/ca.crt"
      user     => "logstash_writer"
      password => "${LOGSTASH_PASSWORD}"
    }
  }
```

> 💡 **Interview tip:** Field-level security is often confused with data masking at the application layer. The key distinction: **field-level security in Elasticsearch is enforced at the query engine level** — even if someone queries directly via curl with their credentials, they can't see masked fields. This is true row/column-level security. Also mention: when designing RBAC, the principle is **index-per-team** — each team's service logs go to their own index (payment-service-*, auth-service-*), and RBAC grants each team access only to their indices. This is simpler than complex field-level rules and is the production standard for multi-tenant ELK deployments.

---

### Q326 — Kubernetes | Scenario-Based | Advanced

> Set up Windows container workloads alongside Linux containers in EKS. What are the constraints, differences, and limitations?

#### Key Points to Cover:
```
Windows Nodes in EKS — Setup:

Step 1 — Enable Windows support in EKS:
  # EKS cluster needs Windows support enabled:
  eksctl utils install-vpc-controllers --cluster my-cluster --approve
  # Installs: vpc-admission-webhook, vpc-resource-controller

Step 2 — Create Windows node group:
  eksctl create nodegroup \
    --cluster my-cluster \
    --name windows-ng \
    --node-ami-family WindowsServer2022FullContainer \
    --node-type m5.xlarge \
    --nodes 2

Step 3 — Required node selector for Windows pods:
  spec:
    nodeSelector:
      kubernetes.io/os: windows
    # Without this: Windows pods might schedule on Linux nodes (will fail)

Step 4 — Required toleration (Windows nodes may have taints):
  tolerations:
  - key: "os"
    value: "windows"
    effect: NoSchedule

Scheduling constraints:
  → Windows pods MUST have nodeSelector: kubernetes.io/os: windows
  → Linux pods should have nodeSelector: kubernetes.io/os: linux
    (to prevent Linux pods landing on Windows nodes accidentally)
  → Cannot mix Linux and Windows containers in the same Pod
  → Windows nodes cannot run Linux system daemonsets (CoreDNS, etc.)

Windows vs Linux containers — key differences:
  Base images:
    Linux:   scratch, alpine, ubuntu, debian
    Windows: mcr.microsoft.com/windows/servercore:ltsc2022
             mcr.microsoft.com/windows/nanoserver:ltsc2022

  Supported features in Windows K8s:
    ✅ Deployments, StatefulSets, DaemonSets
    ✅ ConfigMaps, Secrets
    ✅ Services (ClusterIP, NodePort, LoadBalancer)
    ✅ Windows process isolation

    ❌ Host networking (hostNetwork: true) — NOT supported on Windows
    ❌ Privileged containers — NOT supported on Windows
    ❌ Linux-specific syscalls, /proc, /sys mounts
    ❌ Hyper-V isolation (deprecated in K8s)
    ❌ Some CSI drivers not Windows-compatible

  Resource limits (Windows):
    → CPU limits work but implementation differs from Linux (no cgroups)
    → Memory limits: OOMKill behavior differs
    → No huge pages support on Windows

  Networking (Windows):
    → Uses HNS (Host Network Service) instead of iptables
    → kube-proxy runs in userspace mode on Windows
    → VPC CNI for Windows works differently (IPv4 only)
    → DSR (Direct Server Return) not supported

Limitations in production:
  → Larger node cost (Windows Server licensing)
  → Larger image sizes (Windows base images are 3-6GB vs 50MB Alpine)
  → Slower pod startup (larger images, Windows startup time)
  → Fewer networking plugins support Windows
  → No GPU support for Windows nodes in EKS (as of 2024)
  → Windows nodes cannot run DaemonSets for Linux monitoring agents
    → Need separate Windows-compatible monitoring (Datadog Windows agent, etc.)

Tagging strategy for mixed clusters:
  Use node labels to enforce scheduling:
  linux pods:   nodeSelector: kubernetes.io/os: linux
  windows pods: nodeSelector: kubernetes.io/os: windows
```

> 💡 **Interview tip:** Windows containers in Kubernetes are a niche topic, but the question tests **practical knowledge of K8s limitations** — not just the happy path. The most important point: **you cannot mix Linux and Windows containers in the same pod**. A sidecar container must match the OS of the main container. This breaks many common patterns (Istio sidecar injection doesn't work on Windows pods by default). Also mention the **size problem**: Windows base images are 3-6GB, making pull times slow and node storage expensive. For the .NET Framework workload specifically, mention that .NET Core / .NET 5+ can run on Linux — the question is about .NET Framework specifically, which is Windows-only.

---

### Q327 — Linux / Bash | Troubleshooting | Advanced

> Production app has sporadic TCP RST packets. Walk through diagnosing TCP RST on Linux — tcpdump, ss, netstat, /proc/net/tcp, keepalive settings.

#### Key Points to Cover:
```
What causes RST packets:
  → Connection closed abruptly (as opposed to FIN graceful close)
  → Causes:
    a) Application called close() on socket while data still buffered
    b) Firewall/proxy dropped connection and sent RST on behalf
    c) Connection idle timeout (firewall, load balancer, NAT gateway)
    d) TCP keepalive not configured → idle connection dropped
    e) Server crashed and was restarted (new server sends RST to old conn)
    f) Sequence number mismatch (rare)
    g) Port not listening (connecting to closed port)

Step 1 — Capture RST packets with tcpdump:
  # Capture RSTs on any interface:
  tcpdump -i any 'tcp[tcpflags] & (tcp-rst) != 0' -w /tmp/rst-capture.pcap

  # More targeted (database port):
  tcpdump -i any 'tcp[tcpflags] & (tcp-rst) != 0 and port 5432' \
    -nn -v 2>&1 | head -100

  # Key fields in output:
  # Who sent RST? (src IP)
  # What port?
  # TCP sequence number at RST time

Step 2 — Check current connection states:
  # Modern tool (replaces netstat):
  ss -tan | grep -E 'ESTABLISHED|CLOSE_WAIT|TIME_WAIT|FIN_WAIT'
  ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn

  # Connections to database:
  ss -tan dst :5432

  # Check for high TIME_WAIT (connection churn):
  ss -tan state time-wait | wc -l

Step 3 — TCP keepalive settings:
  # Current values:
  sysctl net.ipv4.tcp_keepalive_time    # default: 7200s (2 hours!)
  sysctl net.ipv4.tcp_keepalive_intvl   # default: 75s
  sysctl net.ipv4.tcp_keepalive_probes  # default: 9

  # Problem: firewall/NAT drops idle connections after 5 min
  # But kernel keepalive doesn't probe for 2 hours → RST when app uses it

  # Fix: reduce keepalive interval:
  sysctl -w net.ipv4.tcp_keepalive_time=60     # probe after 60s idle
  sysctl -w net.ipv4.tcp_keepalive_intvl=10    # probe every 10s
  sysctl -w net.ipv4.tcp_keepalive_probes=3    # give up after 3 probes

Step 4 — Database connection pool idle timeout:
  # Java HikariCP example:
  # pool.idleTimeout = 600000 (10 min)
  # But firewall drops connections after 5 min
  # → App holds connection it thinks is alive → RST when used

  # Fix: set pool idleTimeout < firewall timeout
  # OR: enable TCP keepalive in JDBC:
  jdbc:postgresql://host/db?socketTimeout=30&tcpKeepAlive=true

Step 5 — Check /proc/net/tcp:
  # Raw socket table:
  cat /proc/net/tcp
  # Hex format: local_address rem_address state tx_queue rx_queue
  # State: 01=ESTABLISHED 06=TIME_WAIT 08=CLOSE_WAIT 0A=LISTEN

  # Convert hex IP to readable:
  python3 -c "import socket,struct; print(socket.inet_ntoa(struct.pack('<L', int('0F02000A', 16))))"

Step 6 — Application-level fix (Java):
  // Set SO_KEEPALIVE on socket:
  socket.setKeepAlive(true);

  // Or fix connection pool: validate connection before use
  // HikariCP: connectionTestQuery = "SELECT 1"
  // hikari.keepaliveTime = 30000  (30s keepalive)
```

> 💡 **Interview tip:** The **2-hour default keepalive** is the silent killer in cloud environments. AWS NAT Gateway drops idle connections after 350 seconds (~6 minutes). Azure Load Balancer: 4 minutes. But Linux's default TCP keepalive doesn't probe for 2 hours. So connections that sit idle for 7 minutes look alive to the application but are dead at the network layer. When the app finally uses it, it gets an RST. The fix is at both layers: reduce system keepalive to below the firewall timeout, AND configure the connection pool to validate connections before use. Either one alone doesn't fully solve it.

---

### Q328 — Terraform | Conceptual | Advanced

> Explain Terraform sensitive values. What does `sensitive = true` prevent? What are its limitations? How do you properly handle secrets without them leaking into state?

#### Key Points to Cover:
```
What sensitive = true does:
  → Marks variable/output as sensitive
  → Prevents value from appearing in:
    - terraform plan output (shows as (sensitive value))
    - terraform apply output
    - Console/terminal logs
  → Value IS still stored in state file in plaintext

Declaring sensitive values:
  # Variable:
  variable "db_password" {
    type      = string
    sensitive = true
  }

  # Output:
  output "connection_string" {
    value     = "postgres://user:${var.db_password}@${aws_db_instance.main.endpoint}/db"
    sensitive = true
  }

  # Automatically sensitive: outputs referencing sensitive inputs
  # (Terraform 0.15+ propagates sensitivity automatically)

What sensitive = true does NOT prevent:
  ❌ State file: terraform.tfstate contains value in PLAINTEXT
     → grep "db_password" terraform.tfstate → shows the actual value
  ❌ Provider logs: if provider logs are enabled, values may appear
  ❌ Crash dumps: Terraform panic output may include values
  ❌ Environment variable workarounds:
     TF_VAR_db_password=mypassword → appears in shell history
  ❌ Plan file: saved plan may contain values

Proper secret handling strategies:

Strategy 1 — AWS Secrets Manager + data source (recommended):
  data "aws_secretsmanager_secret_version" "db_pass" {
    secret_id = "prod/db/password"
  }
  # Secret never in .tf files or vars
  # Retrieved at apply time, stored in state

  resource "aws_db_instance" "main" {
    password = jsondecode(data.aws_secretsmanager_secret_version.db_pass.secret_string)["password"]
  }

Strategy 2 — Never store in state with random_password:
  resource "random_password" "db_pass" {
    length  = 32
    special = true
  }

  resource "aws_secretsmanager_secret_version" "db_pass" {
    secret_id     = aws_secretsmanager_secret.db.id
    secret_string = random_password.db_pass.result
    # Password generated by TF, stored in Secrets Manager
    # State contains it, but Secrets Manager is source of truth
  }

Strategy 3 — Vault dynamic secrets:
  provider "vault" {}
  data "vault_generic_secret" "db" {
    path = "secret/prod/database"
  }
  # Vault provides short-lived dynamic credentials
  # Credentials rotate, minimizing exposure window

State file protection (critical):
  # Remote backend with encryption:
  terraform {
    backend "s3" {
      bucket  = "terraform-state"
      key     = "prod/terraform.tfstate"
      encrypt = true         # Server-side encryption
      # State contains plaintext secrets but S3 encrypts at rest
    }
  }

  # State access controls:
  # Limit who can read the state file (IAM policy on S3 bucket)
  # Enable S3 versioning (audit trail)
  # Enable CloudTrail on S3 GetObject for state reads

  # For highest security: Terraform Cloud/Enterprise
  # → State encrypted with customer-managed key
  # → Variable sets with sensitive variables never in state
  # → Remote execution: credentials never touch developer machine
```

> 💡 **Interview tip:** The most important point that trips candidates: `sensitive = true` is a **display control, not a security control**. It prevents the value from being printed to your terminal, but the value is still in the state file in plaintext. The state file is the real security boundary. Production strategy: state file in S3 with SSE-KMS encryption + strict IAM limiting who can read the state + avoid putting secrets in Terraform at all (use data sources to fetch from Secrets Manager at apply time). The ideal: **secrets never appear in .tf files** — they live in Secrets Manager and Terraform just references them by ARN.

---

### Q329 — GitHub Actions | Conceptual | Advanced

> Explain GitHub Actions permissions at workflow/job/step level. What are risks of write-all? Write a workflow with minimum required permissions.

#### Key Points to Cover:
```
GITHUB_TOKEN:
  → Auto-generated token for each workflow run
  → Scoped to the repository
  → Permissions controlled at workflow/job level
  → Expires when workflow finishes
  → Default permissions: depends on repo settings
    (Organization can set default to: read-all or write-all)

Permissions scope:
  permissions:
    actions: read|write|none
    checks: read|write|none
    contents: read|write|none      # repo content, tags, releases
    deployments: read|write|none
    id-token: write                # OIDC — needed for AWS auth
    issues: read|write|none
    packages: read|write|none      # GitHub Packages/Container Registry
    pull-requests: read|write|none # comment on PRs, approve
    statuses: read|write|none      # commit status checks
    security-events: write         # upload SARIF security findings

Risks of permissions: write-all:
  → Compromised action in your workflow → full repo access
  → Can: push malicious code, delete releases, modify issues
  → Supply chain attack: action you use gets compromised → steals token
  → write-all means: any action in your workflow has full write access
  → Token can be exfiltrated via environment variable leak

Principle of least privilege examples:

  Create Release:     contents: write
  Push to branch:     contents: write
  Comment on PR:      pull-requests: write
  Read repo code:     contents: read
  Publish to GHCR:    packages: write, contents: read
  OIDC (AWS auth):    id-token: write, contents: read
  Upload SARIF:       security-events: write, contents: read
  Update PR status:   statuses: write
  Read issues:        issues: read

Minimum-permission workflow example:
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# Default: restrict everything
permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    # Inherits contents: read from top-level (enough for checkout)
    steps:
    - uses: actions/checkout@v4
    - run: npm ci && npm test

  release:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: write        # to create release and push tag
      packages: write        # to push to GitHub Container Registry
      id-token: write        # for OIDC AWS authentication
      pull-requests: write   # to comment release link on PR
    steps:
    - uses: actions/checkout@v4
    - name: Authenticate to AWS via OIDC
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-actions
        aws-region: us-east-1
    - name: Push to ECR and Deploy
      run: |
        aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URL
        docker build -t $IMAGE .
        docker push $IMAGE

  security-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write   # upload SARIF
    steps:
    - uses: actions/checkout@v4
    - uses: github/codeql-action/analyze@v3

  pr-comment:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    permissions:
      pull-requests: write    # comment only — nothing else
    steps:
    - uses: actions/github-script@v7
      with:
        script: |
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            body: 'Tests passed ✅'
          })
```

> 💡 **Interview tip:** GitHub Actions permissions have a **cascade**: top-level `permissions` sets the default for all jobs; job-level `permissions` overrides for that job only; you cannot grant MORE than what's at the top level. The key operational concern: **third-party actions run with the same permissions as your job**. If you use `uses: some-action/unknown@v1` with `write-all`, that action can read your secrets and push to your repo. This is the supply chain attack vector — Actions are code running in your CI/CD with your credentials. Least privilege limits the blast radius. Also mention pinning actions to **full commit SHA** (`uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29`) instead of tags — tags can be moved by the action author.

---

### Q330 — AWS | Troubleshooting | Advanced

> DynamoDB ProvisionedThroughputExceededException even with auto-scaling. Explain hot partitions, adaptive capacity, burst capacity, and how to diagnose/fix.

#### Key Points to Cover:
```
Why auto-scaling doesn't immediately help:
  → Auto-scaling reacts to CloudWatch metrics (ConsumedWriteCapacityUnits)
  → Scale-up takes: 1-3 minutes (metric window + scaling action)
  → During that window: throttling still occurs
  → Also: auto-scaling scales table capacity, not partition capacity

Hot partition problem:
  DynamoDB partitions data by partition key
  Each partition gets a share of total provisioned throughput
  Example:
    Table: 1000 WCU total, 10 partitions = 100 WCU per partition
    If ALL writes go to partition with key "user-001" → 100 WCU limit hit
    Even if other partitions are idle → cannot borrow their capacity
    Result: throttling on hot partition despite table having capacity

Causes of hot partitions:
  → Bad partition key design: using sequential IDs, dates, low-cardinality values
  → Access pattern mismatch: one product/user is viral (99% of traffic)
  → Time-series data: current hour's partition gets all writes
  → Status-based partition key: "PENDING" = all new orders

Adaptive Capacity (automatic, free):
  → DynamoDB automatically identifies and isolates hot partitions
  → Reallocates capacity from cold partitions to hot partitions
  → Takes effect: within seconds to minutes
  → Limitation: cannot exceed overall table provisioned capacity
  → Limitation: cannot exceed physical partition maximum (~3000 RCU / 1000 WCU)

Burst Capacity:
  → Each partition retains up to 5 minutes of unused capacity
  → Can absorb short spikes without throttling
  → Consumed automatically: no configuration needed
  → Limitation: not infinite — depletes quickly under sustained load
  → Cannot be predicted reliably

Diagnosis:
  # Check for hot partition symptoms:
  aws cloudwatch get-metric-statistics \
    --namespace AWS/DynamoDB \
    --metric-name ThrottledRequests \
    --dimensions Name=TableName,Value=my-table \
    --start-time 2024-01-01T10:00:00Z \
    --end-time 2024-01-01T11:00:00Z \
    --period 60 \
    --statistics Sum

  # Check consumed vs provisioned:
  # If consumed << provisioned but throttling occurs → hot partition
  # CloudWatch Contributor Insights (for partition key analysis):
  # Enable: DynamoDB → Table → Monitoring → CloudWatch Contributor Insights
  # Shows: top partition keys by request count and throttle count

Solutions:
  1. Better partition key design:
     → High cardinality: UUID, user_id (millions of values)
     → Avoid: date, status, boolean, sequential numbers

  2. Write sharding (for hot keys you can't avoid):
     → Append random suffix to partition key: "product_001_3" (suffix 1-10)
     → Spreads writes across 10 partitions
     → Read: must query all 10 shards and merge (Scatter-Gather)

  3. DAX (DynamoDB Accelerator):
     → In-memory cache in front of DynamoDB
     → Read-heavy hot keys: served from cache, not DynamoDB
     → Reduces read load on hot partitions dramatically

  4. Switch to On-Demand mode:
     → No capacity planning needed
     → DynamoDB scales instantly per-request
     → Cost: higher per-request vs provisioned for predictable workloads
     → For unpredictable/spiky traffic: on-demand eliminates throttling
```

> 💡 **Interview tip:** The hot partition problem reveals a fundamental DynamoDB architecture principle: **table-level capacity ≠ partition-level capacity**. You can provision 10,000 WCU at the table level, but if all writes hit the same partition key, that partition is limited to its allocated share (~1000 WCU per physical partition). The diagnostic smoking gun: **ThrottledRequests > 0 while ConsumedWriteCapacityUnits << ProvisionedWriteCapacityUnits**. That tells you it's a distribution problem (hot partition), not a capacity problem. The fix is always partition key design — reactive fixes (more capacity, auto-scaling) don't solve the root cause.

---

### Q331 — Prometheus | Scenario-Based | Advanced

> Set up complete Prometheus monitoring for Nginx Ingress Controller: per-ingress-rule request rate, upstream response time, error rates, SSL expiry, with ServiceMonitor and alerting rules.

#### Key Points to Cover:
```
Step 1 — Enable Nginx Ingress metrics:
  # values.yaml for Nginx Ingress Helm chart:
  controller:
    metrics:
      enabled: true
      service:
        annotations:
          prometheus.io/scrape: "true"
          prometheus.io/port: "10254"
    serviceMonitor:
      enabled: true
      additionalLabels:
        release: prometheus  # must match Prometheus selector

Step 2 — ServiceMonitor (if using Prometheus Operator):
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: nginx-ingress
    namespace: ingress-nginx
    labels:
      release: prometheus
  spec:
    selector:
      matchLabels:
        app.kubernetes.io/name: ingress-nginx
    endpoints:
    - port: metrics
      interval: 30s
      path: /metrics

Key Nginx Ingress Metrics:
  nginx_ingress_controller_requests           # Counter: total requests
  nginx_ingress_controller_request_duration_seconds  # Histogram: latency
  nginx_ingress_controller_response_size_bytes       # Histogram: response size
  nginx_ingress_controller_upstream_latency_seconds  # Histogram: backend latency
  nginx_ingress_controller_ssl_expire_time_seconds   # Gauge: cert expiry

Labels available:
  ingress, namespace, service, status, method, path, host

Step 3 — Recording Rules (per-ingress metrics):
  groups:
  - name: nginx_ingress_recording
    rules:
    # Request rate per ingress rule:
    - record: nginx:requests:rate5m
      expr: |
        sum by(ingress, namespace, service, status)(
          rate(nginx_ingress_controller_requests[5m])
        )

    # Error rate per ingress:
    - record: nginx:error_rate:rate5m
      expr: |
        sum by(ingress, namespace)(
          rate(nginx_ingress_controller_requests{status=~"[45].."}[5m])
        )
        /
        sum by(ingress, namespace)(
          rate(nginx_ingress_controller_requests[5m])
        )

    # P99 upstream latency per backend service:
    - record: nginx:upstream_latency_p99:rate5m
      expr: |
        histogram_quantile(0.99,
          sum by(ingress, service, le)(
            rate(nginx_ingress_controller_upstream_latency_seconds_bucket[5m])
          )
        )

Step 4 — Alerting Rules:
  groups:
  - name: nginx_ingress_alerts
    rules:
    # High 5xx error rate per ingress:
    - alert: NginxHighErrorRate
      expr: nginx:error_rate:rate5m > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate on ingress {{ $labels.ingress }}"
        description: "Error rate is {{ $value | humanizePercentage }} on {{ $labels.ingress }}"

    # High upstream latency:
    - alert: NginxHighUpstreamLatency
      expr: nginx:upstream_latency_p99:rate5m > 1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High upstream latency for {{ $labels.service }}"

    # SSL certificate expiring:
    - alert: NginxSSLCertExpiringSoon
      expr: |
        (nginx_ingress_controller_ssl_expire_time_seconds - time()) / 86400 < 30
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "SSL cert expiring in {{ $value | humanizeDuration }} for {{ $labels.host }}"

    # Zero traffic (possible ingress misconfiguration):
    - alert: NginxIngressNoTraffic
      expr: |
        sum by(ingress)(rate(nginx_ingress_controller_requests[15m])) == 0
        and
        sum by(ingress)(nginx_ingress_controller_requests offset 1h) > 100
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "Ingress {{ $labels.ingress }} has zero traffic (was active 1h ago)"
```

> 💡 **Interview tip:** The key insight for per-ingress-rule metrics: Nginx Ingress exposes the `ingress` label on its metrics, so `sum by(ingress, namespace)` gives you per-ingress breakdown automatically — no custom instrumentation needed. The **SSL certificate expiry alert** is something most teams forget until a cert expires in production. The formula `(expire_time - time()) / 86400 < 30` gives days remaining — alert at 30 days to give enough time to renew. For cert-manager users: also set up a separate cert-manager certificate expiry alert as a belt-and-suspenders approach.

---

### Q332 — Kubernetes | Conceptual | Advanced

> Explain Kubernetes Lease objects. How are they used for leader election, node heartbeats, and component health? How does node failure detection work?

#### Key Points to Cover:
```
What are Lease objects:
  → Lightweight Kubernetes objects in the coordination.k8s.io API group
  → Purpose: distributed coordination — who holds a "lease" (lock)
  → Stored in etcd like any other K8s object
  → Fields: holderIdentity, leaseDurationSeconds, acquireTime, renewTime

  apiVersion: coordination.k8s.io/v1
  kind: Lease
  metadata:
    name: kube-controller-manager
    namespace: kube-system
  spec:
    holderIdentity: "master-1"
    leaseDurationSeconds: 15
    acquireTime: "2024-01-15T10:00:00Z"
    renewTime: "2024-01-15T10:05:30Z"   ← updated every 2s

Use Case 1 — Leader Election (Control Plane HA):
  Problem: Running multiple kube-controller-manager instances (HA)
  → Only ONE should act as leader (prevent duplicate operations)
  Solution: Lease-based leader election
  → Each controller manager tries to update the Lease with its ID
  → Whoever successfully updates = leader
  → Leader renews Lease every 2 seconds
  → If leader doesn't renew within leaseDuration (15s):
    → Other instances detect stale Lease → new election
  → Same pattern: kube-scheduler, cloud controller manager

Use Case 2 — Node Heartbeats:
  → Each node gets a Lease object: kube-node-lease namespace, name = node name
  → kubelet updates its Lease every 10 seconds (leaseDurationSeconds: 40s)
  → Node lifecycle controller watches all node Leases
  → Much more efficient than updating the Node object itself
    → Node object: large (many fields) → expensive etcd write
    → Lease object: tiny → cheap etcd write

  Location: kubectl get lease -n kube-node-lease

Use Case 3 — Custom Controller Leader Election:
  → Your custom controller can use Lease for HA:
  import "k8s.io/client-go/tools/leaderelection"
  → Framework handles: acquire, renew, release, detect failure

Node failure detection timeline:
  T+0:    Node crashes / network partition
  T+10s:  Last kubelet Lease renewal (before crash)
  T+40s:  Lease expires (leaseDurationSeconds=40)
  T+40s:  Node lifecycle controller detects: Lease not renewed
  T+40s:  Node condition updated: Ready → Unknown
  T+~60s: node-monitor-grace-period (default 40s after Unknown)
  T+~300s: node-monitor-eviction-rate (default 5m after Unknown)
           → Pod eviction begins from node marked Unknown

  Key tunable:
    --node-monitor-grace-period (kubelet): 40s
    --node-monitor-period (controller): 5s (how often controller checks)

Total detection: ~40-60 seconds from crash to node marked Unknown
Total eviction: ~5 minutes from Unknown to pods evicted
```

> 💡 **Interview tip:** Before Lease objects (Kubernetes 1.13), node heartbeats were implemented by kubelet updating the **Node object** every 10 seconds. With thousands of nodes, this created massive etcd load — each heartbeat was a write to a large object containing all node status, conditions, allocatable resources etc. Lease objects are tiny (just a timestamp + holder identity), reducing etcd write load by ~10x. This is a great example of **Kubernetes internals evolving for scalability**. For the failure detection timeline: interviewers often ask "how long until a dead pod is rescheduled?" — the answer is 5-7 minutes (40s detection + 5min eviction timer + scheduling time), which surprises most people who expect faster failover.

---

### Q333 — ArgoCD | Scenario-Based | Advanced

> Configure ArgoCD notifications: Slack on Degraded, detailed Slack on sync fail, GitHub commit status, PagerDuty for production Degraded > 10 minutes.

#### Key Points to Cover:
```
ArgoCD Notifications setup:
  → Install: argocd-notifications controller
  → Config: ConfigMap argocd-notifications-cm
  → Secrets: argocd-notifications-secret (webhook URLs, tokens)

Step 1 — Notification ConfigMap:
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: argocd-notifications-cm
    namespace: argocd
  data:
    # Service (integration) configs:
    service.slack: |
      token: $slack-token

    service.github: |
      appID: <github-app-id>
      installationID: <installation-id>
      privateKey: $github-private-key

    service.pagerduty: |
      token: $pagerduty-token

    # Template 1: Simple Degraded notification
    template.app-degraded: |
      slack:
        attachments: |
          [{
            "title": "⚠️ Application Degraded",
            "color": "danger",
            "fields": [
              {"title": "App", "value": "{{.app.metadata.name}}", "short": true},
              {"title": "Namespace", "value": "{{.app.spec.destination.namespace}}", "short": true},
              {"title": "Status", "value": "{{.app.status.health.status}}", "short": true}
            ]
          }]

    # Template 2: Detailed sync failure
    template.sync-failed: |
      slack:
        attachments: |
          [{
            "title": "❌ Sync Failed",
            "color": "danger",
            "text": "Application {{.app.metadata.name}} sync failed",
            "fields": [
              {"title": "App", "value": "{{.app.metadata.name}}", "short": true},
              {"title": "Revision", "value": "{{.app.status.sync.revision}}", "short": true},
              {"title": "Sync Message", "value": "{{.app.status.operationState.message}}", "short": false},
              {"title": "Failed Resources", "value": "{{range .app.status.resources}}{{if eq .health.status \"Degraded\"}}• {{.kind}}/{{.name}}\n{{end}}{{end}}", "short": false}
            ]
          }]

    # Template 3: GitHub commit status
    template.github-commit-status: |
      github:
        repoURLPath: "{{.app.spec.source.repoURL}}"
        revisionPath: "{{.app.status.sync.revision}}"
        status:
          state: "{{if eq .app.status.sync.status \"Synced\"}}success{{else}}failure{{end}}"
          label: "argocd/{{.app.metadata.name}}"
          targetURL: "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"

    # Template 4: PagerDuty for prolonged Degraded
    template.prod-degraded-critical: |
      pagerduty:
        incidentKey: "{{.app.metadata.name}}-degraded"
        summary: "PROD: {{.app.metadata.name}} has been Degraded for >10 minutes"
        severity: critical
        component: "{{.app.metadata.name}}"
        customDetails:
          health_status: "{{.app.status.health.status}}"
          namespace: "{{.app.spec.destination.namespace}}"
          message: "{{.app.status.health.message}}"

    # Triggers — when to send each notification:
    trigger.on-degraded: |
      - when: app.status.health.status == 'Degraded'
        send: [app-degraded]

    trigger.on-sync-failed: |
      - when: app.status.operationState.phase in ['Error', 'Failed']
        send: [sync-failed]

    trigger.on-sync-any: |
      - when: app.status.operationState.phase in ['Running', 'Succeeded', 'Error', 'Failed']
        send: [github-commit-status]

    # Prolonged Degraded (> 10 minutes):
    trigger.on-degraded-production: |
      - when: app.status.health.status == 'Degraded' and time.Now().Sub(time.Parse(app.status.health.lastTransitionTime)).Minutes() >= 10
        oncePer: app.status.operationState.syncResult.revision
        send: [prod-degraded-critical]

Step 2 — Subscribe applications to notifications:
  # Per-application annotation:
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: payment-service-prod
    annotations:
      notifications.argoproj.io/subscribe.on-degraded.slack: "#alerts-prod"
      notifications.argoproj.io/subscribe.on-sync-failed.slack: "#deployments"
      notifications.argoproj.io/subscribe.on-sync-any.github: ""
      notifications.argoproj.io/subscribe.on-degraded-production.pagerduty: ""
```

> 💡 **Interview tip:** ArgoCD Notifications follows a **trigger → template → service** pattern. Triggers define WHEN to notify, templates define WHAT to send, services define HOW/WHERE to deliver. The most important operational tip: **use `oncePer` on repeated alerts** to prevent notification storms. Without it, a Degraded application sends a notification on every evaluation cycle (every ~1 minute). With `oncePer: app.status.health.status`, it sends once when the status changes. For production PagerDuty alerts, mention that the **incidentKey** de-duplicates: if PagerDuty already has an open incident with that key, new alerts update the existing incident rather than creating a new one.

---

### Q334 — Linux / Bash | Scenario-Based | Advanced

> Write a Bash script for automated incident triage when a high CPU alert fires. Capture diagnostics, identify processes, check recent deployments, check logs, bundle into JSON report, post to Slack, archive data.

#### Key Points to Cover:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Configuration
SLACK_WEBHOOK="${SLACK_WEBHOOK_URL:-}"
APP_LOG_DIR="${APP_LOG_DIR:-/var/log/app}"
ARCHIVE_DIR="/var/incident-archives"
KNOWN_APPS=("java" "python" "node" "nginx" "postgres")
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
INCIDENT_ID="INC-${TIMESTAMP}"
WORK_DIR=$(mktemp -d /tmp/incident-XXXXXX)
REPORT_FILE="${WORK_DIR}/report.json"

cleanup() { rm -rf "${WORK_DIR}" 2>/dev/null; }
trap cleanup EXIT

log() { echo "[$(date '+%H:%M:%S')] $*" >&2; }

# 1. Capture system snapshot
capture_diagnostics() {
  log "Capturing system diagnostics..."

  TOP_PROCS=$(ps aux --sort=-%cpu | head -11 | tail -10 \
    | awk '{printf "{\"pid\":\"%s\",\"user\":\"%s\",\"cpu\":\"%s\",\"mem\":\"%s\",\"cmd\":\"%s\"},", $2,$1,$3,$4,$11}' \
    | sed 's/,$//')

  LOAD_AVG=$(cat /proc/loadavg | awk '{print "{\"1m\":\""$1"\",\"5m\":\""$2"\",\"15m\":\""$3"\"}"}')
  CPU_COUNT=$(nproc)
  MEM_INFO=$(free -m | awk '/^Mem:/{printf "{\"total\":%s,\"used\":%s,\"free\":%s,\"available\":%s}", $2,$3,$4,$7}')
  DISK_IO=$(iostat -x 1 1 2>/dev/null | awk '/^[sd]d/{printf "{\"device\":\"%s\",\"util\":\"%s\"},", $1,$NF}' | sed 's/,$//' || echo "{}") 
  NET_CONNS=$(ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn \
    | awk '{printf "{\"state\":\"%s\",\"count\":%s},", $2,$1}' | sed 's/,$//')
}

# 2. Identify if high-CPU process is known or unknown
classify_process() {
  local top_cmd=$(ps aux --sort=-%cpu | awk 'NR==2{print $11}')
  local top_pid=$(ps aux --sort=-%cpu | awk 'NR==2{print $2}')
  local is_known="false"

  for app in "${KNOWN_APPS[@]}"; do
    if echo "$top_cmd" | grep -qi "$app"; then
      is_known="true"
      break
    fi
  done

  PROCESS_CLASSIFICATION="{\"command\":\"${top_cmd}\",\"pid\":\"${top_pid}\",\"is_known_app\":${is_known}}"
}

# 3. Check recent deployments via git logs
check_recent_deployments() {
  log "Checking recent deployments..."
  local deploy_info="[]"

  if [ -d "/opt/app/.git" ]; then
    deploy_info=$(git -C /opt/app log --oneline --since="1 hour ago" \
      --pretty='{"hash":"%h","message":"%s","author":"%an","time":"%ci"}' \
      2>/dev/null | jq -s '.' 2>/dev/null || echo "[]")
  fi
  RECENT_DEPLOYS="${deploy_info}"
}

# 4. Check application logs for errors in last 5 minutes
check_app_logs() {
  log "Checking application logs..."
  local error_count=0
  local sample_errors="[]"
  local cutoff=$(date -d '5 minutes ago' '+%Y-%m-%d %H:%M' 2>/dev/null || date -v-5M '+%Y-%m-%d %H:%M')

  if [ -d "${APP_LOG_DIR}" ]; then
    local recent_errors
    recent_errors=$(find "${APP_LOG_DIR}" -name "*.log" -newer /tmp/incident-check \
      -exec grep -h "ERROR\|FATAL\|Exception" {} \; 2>/dev/null | head -20 || true)
    error_count=$(echo "${recent_errors}" | grep -c "ERROR\|FATAL" || true)
    # Capture sample errors as JSON array
    sample_errors=$(echo "${recent_errors}" | head -5 \
      | python3 -c "import sys,json; lines=sys.stdin.read().splitlines(); print(json.dumps(lines))" 2>/dev/null || echo "[]")
  fi

  LOG_ANALYSIS="{\"error_count_5min\":${error_count},\"sample_errors\":${sample_errors}}"
}

# 5. Bundle into structured JSON report
generate_report() {
  log "Generating JSON incident report..."
  cat > "${REPORT_FILE}" <<EOF
{
  "incident_id": "${INCIDENT_ID}",
  "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "hostname": "$(hostname)",
  "cpu_count": ${CPU_COUNT},
  "load_average": ${LOAD_AVG},
  "memory": ${MEM_INFO},
  "top_processes": [${TOP_PROCS}],
  "network_connections": [${NET_CONNS}],
  "disk_io": [${DISK_IO}],
  "process_classification": ${PROCESS_CLASSIFICATION},
  "recent_deployments": ${RECENT_DEPLOYS},
  "log_analysis": ${LOG_ANALYSIS}
}
EOF
}

# 6. Post to Slack
post_to_slack() {
  [ -z "${SLACK_WEBHOOK}" ] && { log "No Slack webhook configured"; return; }
  log "Posting to Slack..."

  local top_proc=$(echo "${TOP_PROCS}" | python3 -c "import sys,json; d=json.loads('['+sys.stdin.read()+']'); print(d[0]['cmd'][:50] if d else 'unknown')" 2>/dev/null || echo "unknown")
  local load=$(echo "${LOAD_AVG}" | python3 -c "import sys,json; d=json.loads(sys.stdin.read()); print(d['1m'])" 2>/dev/null || echo "?")

  curl -sS -X POST "${SLACK_WEBHOOK}" \
    -H 'Content-Type: application/json' \
    -d "{
      \"text\": \"🚨 *High CPU Incident: ${INCIDENT_ID}*\",
      \"attachments\": [{
        \"color\": \"danger\",
        \"fields\": [
          {\"title\": \"Host\", \"value\": \"$(hostname)\", \"short\": true},
          {\"title\": \"Load Avg (1m)\", \"value\": \"${load} / ${CPU_COUNT} cores\", \"short\": true},
          {\"title\": \"Top Process\", \"value\": \"${top_proc}\", \"short\": false},
          {\"title\": \"Archive\", \"value\": \"${ARCHIVE_DIR}/${INCIDENT_ID}.tar.gz\", \"short\": false}
        ]
      }]
    }" || log "Slack notification failed"
}

# 7. Archive all captured data
archive_data() {
  mkdir -p "${ARCHIVE_DIR}"
  cp "${REPORT_FILE}" "${WORK_DIR}/"
  # Also save raw ps, top output
  ps aux > "${WORK_DIR}/ps_aux.txt"
  top -b -n1 > "${WORK_DIR}/top_snapshot.txt" 2>/dev/null || true
  tar -czf "${ARCHIVE_DIR}/${INCIDENT_ID}.tar.gz" -C "${WORK_DIR}" .
  log "Archived to: ${ARCHIVE_DIR}/${INCIDENT_ID}.tar.gz"
}

# Main
touch /tmp/incident-check
log "Starting incident triage: ${INCIDENT_ID}"
capture_diagnostics
classify_process
check_recent_deployments
check_app_logs
generate_report
post_to_slack
archive_data
cat "${REPORT_FILE}"
log "Triage complete: ${INCIDENT_ID}"
```

> 💡 **Interview tip:** The key design decisions to explain: (1) **`set -euo pipefail`** at the top — fail fast on errors, catch pipe failures. (2) **`trap cleanup EXIT`** — temp directory always cleaned up even on failure. (3) **JSON output** — structured data can be parsed by downstream automation, dashboards, or incident management tools. (4) **`tempfile` for work directory** — no race conditions with fixed filenames. The most important operational point: **this script should be triggered by your alerting system automatically**, not run manually. Integrate with PagerDuty runbooks or AlertManager webhook so the diagnostic snapshot is captured at the moment of alert, not 10 minutes later when an engineer logs in.

---

### Q335 — AWS | Scenario-Based | Advanced

> Design a hub-and-spoke network architecture with AWS Network Firewall for centralized egress inspection, threat intelligence blocking, east-west traffic control, and denied traffic logging.

#### Key Points to Cover:
```
Architecture Overview:
  Hub VPC (Inspection VPC):
    → Contains: AWS Network Firewall, NAT Gateways
    → Connects to: all spoke VPCs via Transit Gateway
    → All traffic flows through hub for inspection

  Spoke VPCs:
    → Application workloads (VPC-A: Production, VPC-B: Dev, VPC-C: Data)
    → No direct internet access
    → Route all traffic to hub via TGW

Transit Gateway (TGW):
  → Central routing hub
  → Attachments: Hub VPC + all Spoke VPCs
  → Route tables control traffic flow
  → Appliance mode: forces symmetric routing for stateful inspection

  TGW Route Table — Spoke:
    0.0.0.0/0 → Hub VPC attachment  (internet-bound → hub)
    10.0.0.0/8 → TGW (stays in TGW, hub inspects east-west)

  TGW Route Table — Hub:
    10.10.0.0/16 → VPC-A attachment (return to prod)
    10.20.0.0/16 → VPC-B attachment (return to dev)

Hub VPC Layout:
  ┌─────────────────────────────────────────────────────┐
  │  Hub VPC (100.64.0.0/16)                           │
  │  ┌─────────────────────┐  ┌─────────────────────┐  │
  │  │ TGW Attachment      │  │ Firewall Subnet      │  │
  │  │ Subnet              │  │ (Network Firewall)   │  │
  │  └─────────┬───────────┘  └──────────┬──────────┘  │
  │            │ route to FW              │ route to GW  │
  │  ┌─────────▼───────────┐  ┌──────────▼──────────┐  │
  │  │ Firewall Endpoint   │  │ Public Subnet        │  │
  │  │ (GWLBe or native)   │  │ (NAT Gateway)        │  │
  │  └─────────────────────┘  └─────────────────────┘  │
  └─────────────────────────────────────────────────────┘

Network Firewall Configuration:
  resource "aws_networkfirewall_firewall" "hub" {
    name                = "hub-inspection"
    firewall_policy_arn = aws_networkfirewall_firewall_policy.main.arn
    vpc_id              = aws_vpc.hub.id
    subnet_mapping {
      subnet_id = aws_subnet.firewall.id
    }
  }

  # Firewall Policy (stateful rules):
  resource "aws_networkfirewall_firewall_policy" "main" {
    firewall_policy {
      stateless_default_actions          = ["aws:forward_to_sfe"]
      stateless_fragment_default_actions = ["aws:forward_to_sfe"]

      stateful_rule_group_reference {
        resource_arn = aws_networkfirewall_rule_group.block_malicious.arn
      }
      stateful_rule_group_reference {
        resource_arn = aws_networkfirewall_rule_group.east_west_policy.arn
      }
    }
  }

  # Rule Group 1 — Block malicious domains (threat intelligence):
  resource "aws_networkfirewall_rule_group" "block_malicious" {
    type     = "STATEFUL"
    capacity = 1000
    rule_group {
      rules_source {
        rules_source_list {
          generated_rules_type = "DENYLIST"
          target_types         = ["HTTP_HOST", "TLS_SNI"]
          targets = [
            ".malware-c2.com",
            ".known-bad-domain.net",
            # Or: reference AWS managed threat intelligence feeds
          ]
        }
      }
    }
  }

  # Rule Group 2 — East-west traffic control:
  resource "aws_networkfirewall_rule_group" "east_west_policy" {
    type     = "STATEFUL"
    capacity = 500
    rule_group {
      rules_source {
        stateful_rule {
          action = "PASS"
          header {
            protocol         = "TCP"
            source           = "10.10.0.0/16"  # Prod VPC
            source_port      = "ANY"
            destination      = "10.20.0.0/16"  # Dev VPC — blocked
            destination_port = "ANY"
            direction        = "FORWARD"
          }
          rule_option { keyword = "sid:1" }
        }
        # Drop all other east-west not explicitly allowed:
        stateful_rule {
          action = "DROP"
          header {
            protocol         = "IP"
            source           = "10.0.0.0/8"
            source_port      = "ANY"
            destination      = "10.0.0.0/8"
            destination_port = "ANY"
            direction        = "FORWARD"
          }
          rule_option { keyword = "sid:2" }
        }
      }
    }
  }

  # Logging to S3 for denied traffic:
  resource "aws_networkfirewall_logging_configuration" "main" {
    firewall_arn = aws_networkfirewall_firewall.hub.arn
    logging_configuration {
      log_destination_config {
        log_destination_type = "S3"
        log_type             = "ALERT"   # only denied/alerted traffic
        log_destination = {
          bucketName = aws_s3_bucket.firewall_logs.bucket
          prefix     = "network-firewall-alerts"
        }
      }
      log_destination_config {
        log_type             = "FLOW"    # all flows (verbose)
        log_destination_type = "CloudWatchLogs"
        log_destination = {
          logGroup = "/aws/networkfirewall/flows"
        }
      }
    }
  }
```

> 💡 **Interview tip:** The key architectural principle: **all traffic must flow through the hub** before reaching the internet or other VPCs. This is enforced by TGW route tables — there is no route from spoke VPCs directly to an Internet Gateway. TGW's **appliance mode** is critical for stateful inspection: without it, return traffic might take a different path through TGW, and the stateful firewall would see asymmetric flows (request on one path, response on another) and drop them. Also mention: Network Firewall works at **Layer 3/4/7 (with domain-based rules)**. For full Layer 7 inspection (TLS inspection, deep packet inspection), you need to configure TLS inspection certificates — a significant operational overhead that must be planned for.

---

### Q336 — Kubernetes | Troubleshooting | Advanced

> CoreDNS pods running but DNS resolution intermittently takes 5+ seconds. Diagnose CoreDNS performance issues — metrics, ndots, caching, NodeLocal DNSCache.

#### Key Points to Cover:
```
CoreDNS latency root causes:

1. ndots configuration (most common):
  Default ndots:5 in /etc/resolv.conf inside pods
  For a query like "backend":
    → Tries: backend.namespace.svc.cluster.local (fails → wait)
    → Tries: backend.svc.cluster.local (fails → wait)
    → Tries: backend.cluster.local (fails → wait)
    → Tries: backend.company.com (fails → wait)
    → Tries: backend (fails → wait)
    = 5 DNS queries for ONE lookup
  
  Fix: reduce ndots or use FQDN:
    dnsConfig:
      options:
      - name: ndots
        value: "2"    # Only 2 search domains tried first
  
  OR: always use FQDN: "backend.namespace.svc.cluster.local."
    (trailing dot = FQDN, skips search domain expansion)

2. CoreDNS conntrack table overflow:
  → Linux conntrack table full → new connections dropped
  → DNS packets lost → 5s timeout (default resolver wait)
  kubectl -n kube-system exec -it coredns-xxx -- \
    cat /proc/net/nf_conntrack | wc -l
  Fix: increase conntrack table or use CoreDNS with UDP (stateless)

3. CoreDNS not scaled:
  → Default: 2 CoreDNS pods for entire cluster
  → High query load → CoreDNS overloaded → queuing delay
  kubectl -n kube-system top pods -l k8s-app=kube-dns
  Fix: scale CoreDNS replicas
  kubectl -n kube-system scale deployment coredns --replicas=4
  → Or: use HPA for CoreDNS based on CPU/memory

4. Upstream DNS slow:
  → CoreDNS forwards external queries to: 8.8.8.8 (or VPC DNS)
  → If upstream is slow → all external queries slow
  Fix: use DNS caching in CoreDNS (already default via cache plugin)
  Check: CoreDNS cache hit rate metric

5. CoreDNS metrics diagnosis:
  # Query rate:
  coredns_dns_requests_total
  rate(coredns_dns_requests_total[5m])

  # Query latency:
  histogram_quantile(0.99, rate(coredns_dns_request_duration_seconds_bucket[5m]))

  # Cache hit rate:
  rate(coredns_cache_hits_total[5m]) / rate(coredns_dns_requests_total[5m])

  # Errors:
  rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m])

NodeLocal DNSCache (best solution for high-throughput):
  → Runs a DNS cache daemon on EVERY node (as DaemonSet)
  → Pods query local node cache (127.0.0.1:53 or link-local IP)
  → Cache hit → microsecond response
  → Cache miss → NodeLocal DNS queries CoreDNS
  → Eliminates: conntrack issues (local socket, no NAT)
  → Reduces: CoreDNS load by 60-80% for services within cluster

  Install:
  kubectl apply -f https://k8s.io/examples/admin/dns/nodelocaldns.yaml

  Pod DNS config (with NodeLocal):
  dnsConfig:
    nameservers:
    - 169.254.20.10  # NodeLocal DNSCache IP (link-local)
    options:
    - name: ndots
      value: "5"
    - name: ndots
      value: "2"     # reduce search domain attempts

CoreDNS tuning (Corefile):
  cluster.local:53 {
    kubernetes cluster.local in-addr.arpa ip6.arpa {
      pods insecure
      fallthrough in-addr.arpa ip6.arpa
    }
    cache 30              # cache TTL
    autopath @kubernetes  # fix for ndots — sends all search queries as one
    loop
    reload
    loadbalance
  }
```

> 💡 **Interview tip:** The **5-second DNS timeout** is the classic giveaway that DNS queries are being dropped (not just slow). Linux DNS resolver default timeout is 5 seconds per attempt — if a UDP packet is dropped (conntrack overflow, network issue), the resolver waits the full 5 seconds before retrying. This makes intermittent drops appear as consistent 5-second latency. The ndots issue causes **5x the expected DNS queries** — with ndots:5, a single hostname lookup triggers 5 sequential queries. For production Kubernetes with many microservices, **NodeLocal DNSCache is essentially mandatory** — it eliminates the conntrack problem entirely (local socket, no kernel NAT tracking) and dramatically reduces CoreDNS load.

---

### Q337 — Grafana | Conceptual | Advanced

> Explain all 7 Grafana variable types. Write a dashboard with chained variables: cluster → namespace → deployment, all panels filter on all three.

#### Key Points to Cover:
```
7 Grafana Variable Types:

  1. Query:
     → Executes a query against a data source to get values
     → Most powerful and dynamic
     → Example: query all cluster labels from Prometheus
     label_values(kube_pod_info, cluster)

  2. Custom:
     → Static comma-separated list you define manually
     → Example: environment: dev,staging,production
     → Use: fixed set of values that don't change often

  3. Constant:
     → Single fixed value (not user-selectable)
     → Use: URL prefixes, API endpoints, shared config
     → Hidden from dashboard UI by default

  4. Datasource:
     → Select from available Grafana data sources
     → Use: dashboards that can switch between Prometheus instances
     → Example: select prod-prometheus vs dev-prometheus

  5. Interval:
     → Time grouping for aggregations ($__interval)
     → Values: 1m, 5m, 10m, 30m, 1h, 6h, 1d
     → Use: rate() and avg_over_time() windows

  6. Textbox:
     → Free-form text input from user
     → Use: filter by arbitrary string (pod name prefix, user ID)
     → No dropdown — user types value

  7. Ad hoc filters:
     → Dynamic label=value filter pairs
     → User adds/removes filters at runtime
     → Applied to all panel queries automatically
     → Use: exploratory dashboards, log filtering in Loki

Chained Variables (cluster → namespace → deployment):

  Variable 1 — cluster:
    Type: Query
    Data Source: Prometheus
    Query: label_values(kube_pod_info, cluster)
    Multi-value: false (select one cluster)
    Include All: false

  Variable 2 — namespace (filtered by $cluster):
    Type: Query
    Data Source: Prometheus
    Query: label_values(kube_pod_info{cluster="$cluster"}, namespace)
    Refresh: On time range change OR On dashboard load
    Depends on: $cluster — when cluster changes, namespace options update

  Variable 3 — deployment (filtered by $cluster AND $namespace):
    Type: Query
    Data Source: Prometheus
    Query: label_values(kube_deployment_labels{cluster="$cluster", namespace="$namespace"}, deployment)
    Refresh: On time range change
    Depends on: both $cluster and $namespace

  Dashboard panels using all three variables:
  # CPU usage panel:
  sum(rate(container_cpu_usage_seconds_total{
    cluster="$cluster",
    namespace="$namespace",
    pod=~"$deployment-.*"
  }[5m])) by (pod)

  # Memory usage panel:
  sum(container_memory_working_set_bytes{
    cluster="$cluster",
    namespace="$namespace",
    pod=~"$deployment-.*"
  }) by (pod)

  # Request rate panel:
  sum(rate(http_requests_total{
    cluster="$cluster",
    namespace="$namespace",
    deployment="$deployment"
  }[5m])) by (status)

  # Replica panel:
  kube_deployment_status_replicas_available{
    cluster="$cluster",
    namespace="$namespace",
    deployment="$deployment"
  }

  # Repeat panels for multi-value selection:
  # If $deployment is multi-select:
  # Panel → Repeat → Variable: deployment
  # → Creates one panel per selected deployment
```

> 💡 **Interview tip:** The **chained variable** pattern is the most valuable Grafana skill for operational dashboards. The key mechanism: variable 2's query uses `$cluster` in its label filter — when `$cluster` changes, Grafana re-evaluates variable 2's query with the new cluster value, updating the namespace dropdown automatically. This cascades: changing namespace updates the deployment dropdown. Set `Refresh: On Dashboard Load` for variable 2 and 3 so they're populated immediately when the dashboard opens. A common mistake: not enabling `Refresh: On time range change` — if you don't, switching clusters doesn't update the namespace options until you manually refresh.

---

### Q338 — Terraform | Troubleshooting | Advanced

> Terraform wants to destroy and recreate buckets 3, 4, 5 when inserting at position 2 in a count-based resource list. Explain why, show the for_each fix, and migrate using moved blocks.

#### Key Points to Cover:
```
The Problem — count uses indices:

  # Original:
  variable "bucket_names" {
    default = ["logs", "data", "backups", "archives", "reports"]
  }
  resource "aws_s3_bucket" "buckets" {
    count  = length(var.bucket_names)
    bucket = var.bucket_names[count.index]
  }
  
  State addresses:
    aws_s3_bucket.buckets[0] = logs
    aws_s3_bucket.buckets[1] = data
    aws_s3_bucket.buckets[2] = backups
    aws_s3_bucket.buckets[3] = archives
    aws_s3_bucket.buckets[4] = reports

  # Insert "temp-uploads" at position 2:
  default = ["logs", "data", "temp-uploads", "backups", "archives", "reports"]

  # Now Terraform sees:
    [0] = logs       → unchanged
    [1] = data       → unchanged
    [2] = backups    → WAS backups, NOW temp-uploads → REPLACE backups!
    [3] = archives   → WAS archives, NOW backups → REPLACE archives!
    [4] = reports    → WAS reports, NOW archives → REPLACE reports!
    [5] = (new)      → CREATE reports

  → Index-based addressing: inserting shifts all subsequent indices
  → Terraform: "index 2 used to be 'backups', now it's 'temp-uploads'" → destroy + create

The Solution — for_each uses keys (not indices):

  resource "aws_s3_bucket" "buckets" {
    for_each = toset(["logs", "data", "temp-uploads", "backups", "archives", "reports"])
    bucket   = each.value
  }

  State addresses (key-based):
    aws_s3_bucket.buckets["logs"]
    aws_s3_bucket.buckets["data"]
    aws_s3_bucket.buckets["temp-uploads"]  ← only new one
    aws_s3_bucket.buckets["backups"]       ← unchanged
    aws_s3_bucket.buckets["archives"]      ← unchanged
    aws_s3_bucket.buckets["reports"]       ← unchanged

  # Terraform plan: only CREATE ["temp-uploads"] — nothing else changes ✅

Migrating count to for_each using moved blocks (TF 1.1+):

  # Step 1: Add moved blocks BEFORE removing count:
  moved {
    from = aws_s3_bucket.buckets[0]
    to   = aws_s3_bucket.buckets["logs"]
  }
  moved {
    from = aws_s3_bucket.buckets[1]
    to   = aws_s3_bucket.buckets["data"]
  }
  moved {
    from = aws_s3_bucket.buckets[2]
    to   = aws_s3_bucket.buckets["backups"]
  }
  moved {
    from = aws_s3_bucket.buckets[3]
    to   = aws_s3_bucket.buckets["archives"]
  }
  moved {
    from = aws_s3_bucket.buckets[4]
    to   = aws_s3_bucket.buckets["reports"]
  }

  # Step 2: Replace count with for_each in resource block:
  resource "aws_s3_bucket" "buckets" {
    for_each = toset(var.bucket_names)  # changed from count
    bucket   = each.value
  }

  # Step 3: terraform plan → should show: 0 to add, 0 to change, 0 to destroy
  # (moved blocks tell TF: "this is the same resource, just renamed in state")

  # Step 4: Apply, then remove moved blocks in next PR
  # (moved blocks are one-time migration aids, not permanent config)

When to use count vs for_each:
  count:    homogeneous resources (N identical things, order doesn't matter)
  for_each: named resources (order matters, insert/delete doesn't shift others)
  Rule: if you'll ever insert/delete in the middle → use for_each
```

> 💡 **Interview tip:** This is one of the most practical Terraform pitfalls in production. The mental model: `count` resources are like **array elements** — insert at position 2, everything after shifts. `for_each` resources are like **dictionary entries** — keyed by value, insertion of a new key doesn't affect other keys. The `moved` block is the Terraform 1.1+ way to tell the state "this resource is the same thing, just at a new address" — without it, you'd have to use `terraform state mv` commands manually for each resource. For the migration: **always do `terraform plan` after adding moved blocks and before applying** to confirm zero changes are shown (pure rename, no destruction).

---

### Q339 — Python | Scenario-Based | Advanced

> Write a Python log anomaly detector: real-time file tailing, sliding window error counts, Z-score spike detection, new pattern detection, Slack alerts, log rotation handling.

#### Key Points to Cover:
```python
#!/usr/bin/env python3
"""Log anomaly detector with Z-score spike detection and new pattern identification."""

import os
import re
import time
import json
import math
import hashlib
import requests
import collections
from datetime import datetime, timedelta
from pathlib import Path


SLACK_WEBHOOK = os.environ.get("SLACK_WEBHOOK_URL", "")
WINDOW_MINUTES = 5
Z_SCORE_THRESHOLD = 3.0
PATTERN_HISTORY_HOURS = 24
POLL_INTERVAL = 0.5


def extract_pattern(line: str) -> str:
    """Extract a normalized pattern from a log line (remove dynamic values)."""
    # Remove timestamps, IPs, UUIDs, numbers
    pattern = re.sub(r'\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}[\.\d]*', 'TIMESTAMP', line)
    pattern = re.sub(r'\b(?:\d{1,3}\.){3}\d{1,3}\b', 'IP', pattern)
    pattern = re.sub(r'[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}', 'UUID', pattern)
    pattern = re.sub(r'\b\d+\b', 'NUM', pattern)
    pattern = re.sub(r'"[^"]*"', '"STR"', pattern)
    return pattern.strip()[:200]  # limit pattern length


def hash_pattern(pattern: str) -> str:
    return hashlib.md5(pattern.encode()).hexdigest()[:8]


class SlidingWindow:
    """Maintains error counts in a sliding time window."""
    def __init__(self, window_minutes: int):
        self.window = timedelta(minutes=window_minutes)
        self.events: collections.deque = collections.deque()

    def add(self):
        self.events.append(datetime.now())
        self._prune()

    def _prune(self):
        cutoff = datetime.now() - self.window
        while self.events and self.events[0] < cutoff:
            self.events.popleft()

    def count(self) -> int:
        self._prune()
        return len(self.events)


class ZScoreDetector:
    """Detects spikes using Z-score on a rolling history of bucket counts."""
    def __init__(self, history_size: int = 30):
        self.history: collections.deque = collections.deque(maxlen=history_size)

    def is_spike(self, current_count: int) -> tuple[bool, float]:
        if len(self.history) < 5:
            self.history.append(current_count)
            return False, 0.0

        mean = sum(self.history) / len(self.history)
        variance = sum((x - mean) ** 2 for x in self.history) / len(self.history)
        std_dev = math.sqrt(variance) if variance > 0 else 0.1

        z_score = (current_count - mean) / std_dev
        self.history.append(current_count)

        return z_score > Z_SCORE_THRESHOLD, z_score


class PatternTracker:
    """Tracks error patterns seen in the last N hours to detect new patterns."""
    def __init__(self, history_hours: int):
        self.history_hours = history_hours
        self.seen_patterns: dict[str, datetime] = {}  # hash → first_seen

    def is_new(self, pattern: str) -> bool:
        ph = hash_pattern(pattern)
        cutoff = datetime.now() - timedelta(hours=self.history_hours)

        if ph not in self.seen_patterns:
            self.seen_patterns[ph] = datetime.now()
            return True

        if self.seen_patterns[ph] < cutoff:
            # Pattern was seen but older than history window — treat as new
            self.seen_patterns[ph] = datetime.now()
            return True

        return False

    def prune(self):
        cutoff = datetime.now() - timedelta(hours=self.history_hours)
        self.seen_patterns = {k: v for k, v in self.seen_patterns.items() if v >= cutoff}


def send_slack_alert(anomaly_type: str, details: dict, sample_lines: list[str]):
    if not SLACK_WEBHOOK:
        print(f"[ALERT] {anomaly_type}: {json.dumps(details)}")
        return

    fields = [{"title": k, "value": str(v), "short": True} for k, v in details.items()]
    samples_text = "\n".join(f"• {line[:150]}" for line in sample_lines[:3])

    payload = {
        "text": f"🚨 *Log Anomaly Detected: {anomaly_type}*",
        "attachments": [{
            "color": "danger",
            "fields": fields,
            "footer": f"Sample logs:\n{samples_text}"
        }]
    }
    try:
        requests.post(SLACK_WEBHOOK, json=payload, timeout=5)
    except Exception as e:
        print(f"[ERROR] Slack notification failed: {e}")


class LogTailer:
    """Tail a file, handling log rotation (truncation or replacement)."""
    def __init__(self, filepath: str):
        self.filepath = Path(filepath)
        self.file = None
        self.inode = None
        self._open()

    def _open(self):
        try:
            self.file = open(self.filepath, 'r')
            self.file.seek(0, 2)  # seek to end
            self.inode = self.filepath.stat().st_ino
        except FileNotFoundError:
            self.file = None
            self.inode = None

    def readlines(self) -> list[str]:
        """Read new lines, detect rotation."""
        if not self.file:
            self._open()
            return []

        # Detect rotation: inode changed or file truncated
        try:
            current_stat = self.filepath.stat()
            if current_stat.st_ino != self.inode:
                # File replaced (logrotate with copytruncate=false)
                self.file.close()
                self._open()
                return []
            if current_stat.st_size < self.file.tell():
                # File truncated (logrotate with copytruncate=true)
                self.file.seek(0)
        except FileNotFoundError:
            self.file = None
            return []

        lines = []
        while True:
            line = self.file.readline()
            if not line:
                break
            lines.append(line.rstrip())
        return lines

    def __del__(self):
        if self.file:
            self.file.close()


def is_error_line(line: str) -> bool:
    return bool(re.search(r'\b(ERROR|FATAL|CRITICAL|Exception|Traceback)\b', line, re.IGNORECASE))


def monitor(log_file: str):
    print(f"[{datetime.now():%H:%M:%S}] Starting log anomaly detection on: {log_file}")

    tailer = LogTailer(log_file)
    error_window = SlidingWindow(WINDOW_MINUTES)
    spike_detector = ZScoreDetector()
    pattern_tracker = PatternTracker(PATTERN_HISTORY_HOURS)
    last_bucket_check = datetime.now()
    recent_error_lines: collections.deque = collections.deque(maxlen=10)

    while True:
        lines = tailer.readlines()

        for line in lines:
            if is_error_line(line):
                error_window.add()
                recent_error_lines.append(line)

                # New pattern detection
                pattern = extract_pattern(line)
                if pattern_tracker.is_new(pattern):
                    send_slack_alert(
                        "New Error Pattern",
                        {"pattern": pattern[:100], "timestamp": datetime.now().isoformat()},
                        [line]
                    )

        # Check for spikes every minute
        if (datetime.now() - last_bucket_check).seconds >= 60:
            current_count = error_window.count()
            is_spike, z_score = spike_detector.is_spike(current_count)

            if is_spike:
                send_slack_alert(
                    "Error Rate Spike",
                    {
                        "errors_last_5min": current_count,
                        "z_score": f"{z_score:.2f}",
                        "threshold": Z_SCORE_THRESHOLD,
                        "timestamp": datetime.now().isoformat()
                    },
                    list(recent_error_lines)[-5:]
                )

            pattern_tracker.prune()
            last_bucket_check = datetime.now()

        time.sleep(POLL_INTERVAL)


if __name__ == "__main__":
    import sys
    log_path = sys.argv[1] if len(sys.argv) > 1 else "/var/log/app/application.log"
    monitor(log_path)
```

> 💡 **Interview tip:** Three concepts to emphasize: (1) **Z-score anomaly detection** — it's self-calibrating (adapts to the baseline of your specific log), unlike a fixed threshold (e.g., ">100 errors/min") which is meaningless without context. If your normal rate is 200 errors/min, a spike to 201 is noise; if normal is 5, a spike to 50 is an incident. (2) **Pattern normalization** — stripping dynamic values (UUIDs, IPs, timestamps) so "Connection refused to 10.0.0.1" and "Connection refused to 10.0.0.2" map to the same pattern. (3) **Inode-based rotation detection** — checking the inode number detects file replacement (logrotate mv then create), while checking file size detects truncation. Both are needed for robust log tailing.

---

### Q340 — AWS | Conceptual | Advanced

> Explain SQS dead-letter queues in depth. Design a complete error handling strategy: 3-attempt DLQ, depth alerting, DLQ replay mechanism, poison pill handling.

#### Key Points to Cover:
```
SQS DLQ fundamentals:
  maxReceiveCount: how many times a message is received before moving to DLQ
  → Receive count tracked per message (not per consumer)
  → Receive ≠ delete: if consumer reads but doesn't delete → count increments
  → After maxReceiveCount receives → message moved to DLQ automatically
  → DLQ is a separate SQS queue (must be created separately)

Message flow with DLQ:
  Producer → Main Queue → Consumer (attempt 1, fails, doesn't delete)
                       → Consumer (attempt 2, fails)
                       → Consumer (attempt 3, fails)
                       → SQS automatically moves to DLQ (maxReceiveCount=3)
  
  After move to DLQ:
  → Original message attributes preserved
  → Original message body unchanged
  → New attribute: ApproximateFirstReceiveTimestamp (first time it was received)
  → DLQ retention: set LONGER than main queue (diagnose before expiry)

Configuration:
  resource "aws_sqs_queue" "main" {
    name                       = "order-processing"
    visibility_timeout_seconds = 30   # must be >= processing time
    redrive_policy = jsonencode({
      deadLetterTargetArn = aws_sqs_queue.dlq.arn
      maxReceiveCount     = 3
    })
  }

  resource "aws_sqs_queue" "dlq" {
    name                      = "order-processing-dlq"
    message_retention_seconds = 1209600  # 14 days (max) — plenty of time to debug
  }

CloudWatch Alert on DLQ depth:
  resource "aws_cloudwatch_metric_alarm" "dlq_alarm" {
    alarm_name          = "DLQ-depth-high"
    metric_name         = "ApproximateNumberOfMessagesVisible"
    namespace           = "AWS/SQS"
    statistic           = "Sum"
    period              = 60
    threshold           = 10
    comparison_operator = "GreaterThanThreshold"
    dimensions = {
      QueueName = aws_sqs_queue.dlq.name
    }
    alarm_actions = [aws_sns_topic.alerts.arn]
  }

DLQ Replay Mechanism (after fixing the bug):
  Option 1 — AWS Console: SQS → DLQ → "Start DLQ redrive" (built-in feature)
    → Moves messages back to source queue
    → Rate-limited to avoid overwhelming consumer

  Option 2 — Script for controlled replay:
  import boto3, time

  sqs = boto3.client('sqs')

  def replay_dlq(dlq_url, main_url, batch_size=10, delay_between_batches=1):
      while True:
          response = sqs.receive_message(
              QueueUrl=dlq_url,
              MaxNumberOfMessages=batch_size,
              WaitTimeSeconds=5
          )
          messages = response.get('Messages', [])
          if not messages:
              print("DLQ is empty")
              break
          
          for msg in messages:
              # Re-send to main queue
              sqs.send_message(
                  QueueUrl=main_url,
                  MessageBody=msg['Body'],
                  MessageAttributes=msg.get('MessageAttributes', {})
              )
              # Delete from DLQ
              sqs.delete_message(
                  QueueUrl=dlq_url,
                  ReceiptHandle=msg['ReceiptHandle']
              )
          
          time.sleep(delay_between_batches)

Poison Pill Handling (messages that ALWAYS fail):
  Problem: message with corrupt data → always fails → replays forever from DLQ
  
  Detection:
  → If message in DLQ is replayed and fails again → goes back to DLQ
  → Track replayCount attribute (custom, add before replay)
  → If replayCount > N → poison pill detected
  
  Strategy:
  1. Quarantine queue: separate queue for poison pills (human review)
  2. Alert with message ID and body sample (don't include PII)
  3. Parser validation BEFORE processing: schema validation catches poison pills early
  
  code:
  def safe_replay(msg):
      replay_count = int(msg.get('MessageAttributes', {})
                       .get('ReplayCount', {})
                       .get('StringValue', '0'))
      
      if replay_count >= 3:
          # Send to quarantine, do not replay
          sqs.send_message(QueueUrl=QUARANTINE_URL, MessageBody=msg['Body'])
          alert_ops(f"Poison pill detected: {msg['MessageId']}")
      else:
          # Add replay count and re-enqueue
          sqs.send_message(
              QueueUrl=MAIN_URL,
              MessageBody=msg['Body'],
              MessageAttributes={
                  'ReplayCount': {'DataType': 'Number', 'StringValue': str(replay_count + 1)}
              }
          )
```

> 💡 **Interview tip:** The key operational insight about DLQ: **set DLQ message retention to 14 days (the maximum)**, not the same as the main queue. You need time to diagnose why messages failed before they expire. If your main queue has 4-day retention and your DLQ has 4-day retention, by the time you notice the DLQ has messages, investigate, fix the bug, and want to replay — the messages might be gone. Also emphasize: the **SQS DLQ redrive** (Start DLQ redrive) is now a built-in AWS Console/API feature — you don't need to write replay scripts for standard cases. Write custom replay logic only when you need rate control, filtering, or transformation during replay.

---

### Q341 — Kubernetes | Scenario-Based | Advanced

> Implement horizontal pod autoscaling based on SQS queue depth using KEDA on EKS, including installation, ScaledObject, IRSA permissions, and testing.

#### Key Points to Cover:
```
Architecture:
  SQS Queue → KEDA Metrics → HPA → Pod count adjustment
  Worker pods poll SQS, process messages
  KEDA scales to 0 when queue empty, scales up as queue fills

Step 1 — Install KEDA:
  helm repo add kedacore https://kedacore.github.io/charts
  helm install keda kedacore/keda \
    --namespace keda \
    --create-namespace \
    --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT:role/keda-operator-role

Step 2 — IRSA IAM Role for KEDA (AWS permissions):
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "sqs:GetQueueAttributes",    # read queue depth
        "sqs:GetQueueUrl",
        "cloudwatch:GetMetricStatistics"  # alternative metric source
      ],
      "Resource": "arn:aws:sqs:us-east-1:ACCOUNT:order-processing"
    }]
  }

  # Trust policy (allows KEDA's service account to assume the role):
  {
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/CLUSTER_ID"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.us-east-1.amazonaws.com/id/CLUSTER_ID:sub": "system:serviceaccount:keda:keda-operator"
      }
    }
  }

Step 3 — ScaledObject configuration:
  apiVersion: keda.sh/v1alpha1
  kind: ScaledObject
  metadata:
    name: order-worker-scaler
    namespace: production
  spec:
    scaleTargetRef:
      name: order-worker              # Deployment to scale
    minReplicaCount: 0                # scale to ZERO (saves cost when idle)
    maxReplicaCount: 50
    pollingInterval: 15               # check SQS every 15 seconds
    cooldownPeriod: 60               # wait 60s before scaling down
    triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/ACCOUNT/order-processing
        queueLength: "100"           # target: 100 messages per pod
        awsRegion: us-east-1
        identityOwner: operator      # use KEDA's IRSA role

  # How scaling math works:
  # Current queue depth: 350 messages
  # Target per pod: 100
  # Desired replicas: ceil(350/100) = 4 pods

Step 4 — Worker Deployment (handles graceful SIGTERM):
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: order-worker
  spec:
    replicas: 1  # KEDA manages this
    template:
      spec:
        terminationGracePeriodSeconds: 60  # give worker time to finish current message
        containers:
        - name: worker
          image: order-worker:latest
          env:
          - name: QUEUE_URL
            value: https://sqs.us-east-1.amazonaws.com/ACCOUNT/order-processing
          - name: VISIBILITY_TIMEOUT
            value: "30"

Step 5 — Testing scale behavior:
  # Check ScaledObject status:
  kubectl get scaledobject order-worker-scaler -n production
  kubectl describe scaledobject order-worker-scaler -n production

  # Send test messages to trigger scaling:
  for i in $(seq 1 500); do
    aws sqs send-message \
      --queue-url https://sqs.us-east-1.amazonaws.com/ACCOUNT/order-processing \
      --message-body "test-order-$i"
  done

  # Watch scaling:
  watch -n5 kubectl get pods -n production -l app=order-worker
  watch -n5 "aws sqs get-queue-attributes --queue-url $QUEUE_URL --attribute-names ApproximateNumberOfMessages"

  # Verify KEDA is detecting queue depth:
  kubectl get hpa -n production  # KEDA creates an HPA under the hood
```

> 💡 **Interview tip:** The most powerful KEDA feature vs standard HPA: **scale to zero**. Standard HPA has a minimum of 1 replica — you always pay for at least one pod. With KEDA's `minReplicaCount: 0`, when the SQS queue is empty, all workers scale down to 0. This is transformative for batch workloads that run only during business hours or in response to events. The `cooldownPeriod` is critical: without it, KEDA might scale to 0 mid-processing when the queue momentarily empties. Set it long enough (60-300 seconds) that workers finish processing before scaling down. Also mention: KEDA creates a standard Kubernetes HPA under the hood — `kubectl get hpa` shows it — KEDA is essentially a more powerful metrics provider for HPA.

---

### Q342 — ELK Stack | Conceptual | Advanced

> Explain Elasticsearch cross-cluster search (CCS) and cross-cluster replication (CCR). Design an architecture with 3 regional clusters, global search, and EU→US DR replica.

#### Key Points to Cover:
```
Cross-Cluster Search (CCS):
  → Query across multiple Elasticsearch clusters from one request
  → Data remains in each cluster — no replication
  → Federated search: results merged and returned
  → Latency: limited by slowest cluster + network RTT
  → Use case: unified search/analytics without moving data

  Syntax:
  GET us-cluster:logs-*,eu-cluster:logs-*,ap-cluster:logs-*/_search
  {
    "query": { "match": { "message": "payment error" } }
  }
  # Returns: results from all 3 clusters merged

  Configuration (in Kibana/API):
  PUT _cluster/settings
  {
    "persistent": {
      "cluster.remote.eu-cluster.seeds": ["eu-es-1:9300"],
      "cluster.remote.ap-cluster.seeds": ["ap-es-1:9300"]
    }
  }

Cross-Cluster Replication (CCR):
  → Replicates indices from leader cluster to follower cluster
  → Near real-time: typically <1 second lag
  → Follower is READ-ONLY (cannot write to follower)
  → Use case: DR (disaster recovery), local read performance
  → Replicated: index data, mappings, settings, aliases

  CCR setup (follower cluster):
  PUT /eu-logs-replicated/_ccr/follow
  {
    "remote_cluster": "eu-cluster",
    "leader_index": "logs-*"
  }

Architecture — 3 regions + DR:

  Regional Elasticsearch Clusters:
  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
  │  US Cluster      │  │  EU Cluster      │  │  AP Cluster      │
  │  us-east-1       │  │  eu-west-1       │  │  ap-southeast-1  │
  │  logs-us-*       │  │  logs-eu-*       │  │  logs-ap-*       │
  │  (Primary data)  │  │  (Primary data)  │  │  (Primary data)  │
  └──────────────────┘  └──────────────────┘  └──────────────────┘
           ▲ CCS                ▲ CCS                ▲ CCS
           │                   │                     │
  ┌────────┼───────────────────┼─────────────────────┼────────────┐
  │        │    Ops Team Kibana (any cluster as entry point)      │
  │        └───────────────────┴─────────────────────┘           │
  │                Cross-Cluster Search query                      │
  │                us-cluster:logs-*,eu-cluster:*,ap-cluster:*    │
  └────────────────────────────────────────────────────────────────┘

                      │ CCR (replication)
                      ▼
  ┌──────────────────────────────────────────┐
  │  US Cluster (also serves as EU DR)       │
  │  logs-eu-replicated-* (READ-ONLY)        │
  │  → Follows: eu-cluster:logs-eu-*        │
  │  → Lag: <1 second normally              │
  │  → If EU goes down: failover to US copy │
  └──────────────────────────────────────────┘

Key distinctions:
  CCS:  query multiple clusters → merged results → no data duplication
  CCR:  copy data across clusters → local replica → read-only follower

  CCS pros:  no storage duplication, each region manages own data
  CCS cons:  latency = max of all cluster RTTs, all clusters must be up

  CCR pros:  local reads (no cross-region latency for DR scenario)
  CCR cons:  storage duplication, follower is read-only, lag monitoring needed

Kibana global view setup (using CCS):
  # In kibana.yml pointing to US cluster as primary:
  elasticsearch.hosts: ["https://us-es:9200"]
  
  # Index pattern in Kibana:
  us-cluster:logs-us-*,eu-cluster:logs-eu-*,ap-cluster:logs-ap-*
  # → Shows logs from all regions in one Discover view
  
  # Or use alias per cluster, wildcarded:
  *:logs-*  # queries all registered remote clusters
```

> 💡 **Interview tip:** CCS and CCR solve different problems and are often used together. CCS for **unified analytics** (ops team queries all regions from one Kibana), CCR for **disaster recovery** (EU data replicated to US so if EU cluster fails, historical logs are still accessible). The key CCR limitation to mention: **followers are read-only** — you cannot index new data into the follower. This is fine for DR (you'd fail over writes to the leader's region or a new cluster), but means you can't use CCR as an active-active write setup. Also mention: CCR requires the **Platinum license** (or above) in Elastic Stack — this is a common surprise for teams using the free version.

---

### Q343 — Jenkins | Troubleshooting | Advanced

> 150 jobs waiting in queue. Builds stuck waiting for shared resource locks. Explain Jenkins Lockable Resources plugin, deadlock causes, and resolution.

#### Key Points to Cover:
```
Jenkins Lockable Resources Plugin:
  → Defines named resources (e.g., "test-db-1", "staging-env")
  → Pipelines request locks before running critical sections
  → Only ONE pipeline holds a named lock at a time
  → Others queue and wait for lock release
  
  Usage:
  pipeline {
    stages {
      stage('Integration Test') {
        steps {
          lock(resource: 'staging-environment') {
            sh './run-integration-tests.sh'
            sh './deploy-to-staging.sh'
          }  // lock released here
        }
      }
    }
  }

Why deadlock occurs:
  Circular dependency:
  Pipeline A: needs [staging-env] + [test-db-1]
  Pipeline B: needs [test-db-1] + [staging-env]

  Timeline:
  T1: Pipeline A acquires lock on "staging-env"
  T2: Pipeline B acquires lock on "test-db-1"
  T3: Pipeline A requests "test-db-1" → waiting (B holds it)
  T4: Pipeline B requests "staging-env" → waiting (A holds it)
  DEADLOCK: both waiting for each other forever

Diagnosing deadlock:
  1. Jenkins → Manage Jenkins → Lockable Resources
     → See: which resources are locked and by whom
  2. Check waiting builds:
     Pipeline A build log: "Waiting for resource [test-db-1]..."
     Pipeline B build log: "Waiting for resource [staging-env]..."
  3. Build console output timestamps:
     → Both builds stuck for >30 minutes = likely deadlock

Resolving deadlock:
  Immediate fix:
  1. Identify which build to sacrifice (lower priority, re-runnable)
  2. Abort one of the deadlocked builds:
     Jenkins → Build → Abort
  3. Other build gets released lock, continues
  4. Re-trigger aborted build

  Permanent fixes:

  Fix 1 — Resource ordering (prevent circular dependency):
    → Always request resources in the same order
    pipeline {
      steps {
        lock(resource: 'staging-environment') {    // always first
          lock(resource: 'test-database') {         // always second
            sh './run-tests.sh'
          }
        }
      }
    }
    → If ALL pipelines always acquire staging before db → no circular wait

  Fix 2 — Request multiple locks atomically:
    lock(resources: ['staging-environment', 'test-database']) {
      // Both acquired at once or neither — no partial acquisition
      sh './run-tests.sh'
    }

  Fix 3 — Timeout with retry:
    lock(resource: 'staging-environment', timeout: 30) {
      // Gives up after 30 minutes if can't acquire → build fails
      // Prevents infinite queue buildup
    }

  Fix 4 — Resource pools (multiple instances):
    // Define 3 staging environments instead of 1:
    // staging-env-1, staging-env-2, staging-env-3
    lock(label: 'staging', quantity: 1) {
      // Acquires ANY ONE of the resources labeled "staging"
      // 3 pipelines can run in parallel
    }

  Fix 5 — Architecture change:
    → Use Jenkins parallel stages with independent environments
    → Ephemeral test environments (spin up/tear down per build)
    → No shared resources → no locking needed
```

> 💡 **Interview tip:** Deadlocks in Jenkins Lockable Resources are identical to database deadlocks — the same solution applies: **resource ordering**. If every pipeline always acquires resource A before resource B, circular waiting is mathematically impossible. The more scalable production solution: **resource pools** — instead of one "staging-environment" lock, define a pool of 5 identical staging environments labeled "staging". Pipelines grab any available one. This eliminates serialization entirely and is the right architecture for teams at scale. Also mention: **ephemeral environments** (spin up test infra per build, tear down after) is the ultimate solution — no shared state, no locking, infinite parallelism, but requires more infrastructure investment.

---

### Q344 — Linux / Bash | Conceptual | Advanced

> Explain Linux shared libraries (.so files), dynamic linking, LD_LIBRARY_PATH. Difference between ldd and ldconfig. Why do containers avoid shared library errors?

#### Key Points to Cover:
```
Shared Libraries (.so files):
  → .so = Shared Object = Linux equivalent of .dll (Windows)
  → Code compiled once, shared by multiple programs in memory
  → Saves: disk space, memory, enables security patches (update once, all programs benefit)
  → Loaded at runtime (dynamic linking) vs linked at compile time (static linking)

  Example:
  libssl.so.1.1   → OpenSSL shared library
  libz.so.1       → zlib compression
  libc.so.6       → C standard library (virtually every program uses this)

  Naming convention:
  libname.so.MAJOR.MINOR
  Symlinks: libssl.so → libssl.so.1 → libssl.so.1.1
  Program links against: libssl.so (no version) → resolved at runtime

Dynamic Linking process:
  1. Program binary contains: names of needed libraries (not the code itself)
  2. At execution: dynamic linker (ld-linux.so) loads the program
  3. Linker reads: program's .dynamic section → list of required .so files
  4. Linker searches: /etc/ld.so.conf paths, LD_LIBRARY_PATH
  5. Loads: each .so into shared memory
  6. Resolves: symbol addresses (function pointers)
  7. Program runs with all symbols resolved

LD_LIBRARY_PATH:
  → Environment variable: colon-separated list of directories to search FIRST
  → Override: normal library search path
  → Use case: testing new library version without installing system-wide
  → Security risk: never use in SUID programs (privilege escalation)
  
  export LD_LIBRARY_PATH=/opt/custom-libs:$LD_LIBRARY_PATH
  ./myprogram  # will find /opt/custom-libs/libfoo.so first

ldd (List Dynamic Dependencies):
  → Shows: which shared libraries a binary needs and where they'll load from
  → Diagnostic tool: shows if libraries are found or missing
  
  ldd /usr/bin/python3
  # Output:
  # linux-vdso.so.1 => (virtual, kernel-provided)
  # libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0
  # libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2
  # NOT FOUND: libcustom.so.1  ← this causes the error
  
  # NEVER run ldd on untrusted binaries: it executes the binary with RTLD_VERIFY

ldconfig:
  → Updates the dynamic linker cache (/etc/ld.so.cache)
  → Must run: after installing new .so files to /lib or /usr/lib
  → Reads: /etc/ld.so.conf for library search directories
  
  # Add new lib directory:
  echo "/opt/myapp/lib" >> /etc/ld.so.conf.d/myapp.conf
  ldconfig   # rebuild cache — must run as root
  ldconfig -p | grep libssl  # search cache for specific library

The common error and causes:
  error while loading shared libraries: libxxx.so.x: cannot open shared object file
  
  Causes:
  a) Library not installed:
     apt install libssl-dev
  b) Library installed but wrong version:
     libssl.so.3 installed, program wants libssl.so.1.1
     Fix: install correct version or create symlink
  c) Library in non-standard path, not in ld.so.cache:
     ldconfig not run after installation
     Fix: run ldconfig
  d) Wrong architecture (32-bit lib on 64-bit system):
     Need: lib32z1 or multilib packages

Why containers avoid this problem:
  → Container image is a complete, self-contained filesystem
  → All required libraries bundled IN the image
  → No dependency on host system libraries (isolated via Linux namespaces)
  → Library version conflicts between containers: impossible (each has own)
  → Library version conflict with host: impossible (containers don't use host libs)
  
  The tradeoff: image size (each container bundles its own libc, libssl, etc.)
  Solution: shared base images (alpine, ubuntu) — layers shared in Docker layer cache
  
  Container layers:
  ubuntu:22.04 layer (shared across all ubuntu-based images)
  python:3.11 layer (shared across all python apps)
  your-app layer (unique to your app)
  → Only unique layers downloaded per container
```

> 💡 **Interview tip:** Connect shared libraries to the **"works on my machine" problem** — the classic deployment failure where code works locally but breaks on the server because the server has a different version of a shared library. Containers completely eliminate this problem by bundling ALL dependencies in the image. The ldd vs ldconfig distinction: `ldd` is a **diagnostic tool** (what does this binary need?), `ldconfig` is a **system management tool** (rebuild the library cache after installing new libraries). Common mistake: installing a `.so` file to `/usr/local/lib` and wondering why the program still can't find it — `ldconfig` wasn't run to add it to the cache.

---

### Q345 — AWS | Scenario-Based | Advanced

> Walk through AWS AppConfig for feature flag management on ECS: setup, polling without restart, rollout strategy, CloudWatch alarm rollback triggers, and comparison with SSM Parameter Store.

#### Key Points to Cover:
```
AppConfig Overview:
  → Managed feature flag and dynamic configuration service
  → Key advantage: deploy configuration changes WITH validation and rollback
  → Supports: JSON, YAML, feature flags schema
  → Integrates: Lambda, ECS, EC2, Kubernetes via AppConfig agent

Step 1 — AppConfig Setup:
  # Application (logical grouping):
  aws appconfig create-application --name payment-service

  # Environment:
  aws appconfig create-environment \
    --application-id $APP_ID \
    --name production

  # Configuration profile (feature flags):
  aws appconfig create-configuration-profile \
    --application-id $APP_ID \
    --name feature-flags \
    --location-uri hosted    # stored in AppConfig (not SSM/S3)
    --type AWS.AppConfig.FeatureFlags

  # Feature flag content:
  {
    "flags": {
      "new-payment-flow": {
        "name": "New Payment Flow",
        "description": "A/B test new payment UI"
      }
    },
    "values": {
      "new-payment-flow": {
        "enabled": true,
        "percentage": 10    # enable for 10% of requests
      }
    },
    "version": "1"
  }

  # Deployment strategy:
  aws appconfig create-deployment-strategy \
    --name gradual-rollout \
    --deployment-duration-in-minutes 30 \
    --final-bake-time-in-minutes 10 \
    --growth-factor 20 \    # increase by 20% every step
    --growth-type EXPONENTIAL

Step 2 — Application polling (no restart needed):
  # Option A: AppConfig Agent as sidecar (ECS):
  # Add to task definition:
  {
    "name": "appconfig-agent",
    "image": "public.ecr.aws/aws-appconfig/aws-appconfig-agent:2.x",
    "environment": [
      {"name": "SERVICE_REGION", "value": "us-east-1"}
    ]
  }
  
  # Application calls AppConfig Agent (localhost):
  import requests
  
  def get_config():
      resp = requests.get(
          "http://localhost:2772/applications/payment-service/environments/production/configurations/feature-flags"
      )
      return resp.json()
  
  # Agent caches config (default 45s TTL)
  # On TTL expiry: agent checks AppConfig for new version
  # No application restart needed
  
  # Option B: AppConfig SDK (direct polling):
  import boto3, time
  
  client = boto3.client('appconfigdata')
  token = client.start_configuration_session(
      ApplicationIdentifier='payment-service',
      EnvironmentIdentifier='production',
      ConfigurationProfileIdentifier='feature-flags'
  )['InitialConfigurationToken']
  
  while True:
      result = client.get_latest_configuration(ConfigurationToken=token)
      token = result['NextPollConfigurationToken']
      if result['Configuration'].read():  # only non-empty if changed
          config = json.loads(result['Configuration'].read())
          update_local_config(config)
      time.sleep(result['NextPollIntervalInSeconds'])

Step 3 — Rollout Strategy (linear/exponential):
  Linear:
    Deployment duration: 30 minutes
    Growth factor: 25% per step = 25% → 50% → 75% → 100%
    Good for: predictable staged rollout

  Exponential:
    Growth factor: 2 = doubles each step
    25% → 50% → 100% (faster)
    Good for: config changes you're confident in

  Instant (AllAtOnce):
    Immediately 100%
    Use for: non-risky config changes, emergency rollback

Step 4 — CloudWatch Alarm rollback trigger:
  # Create alarm on application error rate:
  aws cloudwatch put-metric-alarm \
    --alarm-name payment-error-rate-high \
    --metric-name ErrorRate \
    --namespace Payment/Service \
    --threshold 5 \
    --comparison-operator GreaterThanThreshold \
    --alarm-actions arn:aws:appconfig:us-east-1:ACCOUNT:deployment-strategy/rollback

  # Associate alarm with AppConfig deployment:
  aws appconfig start-deployment \
    --application-id $APP_ID \
    --environment-id $ENV_ID \
    --configuration-profile-id $PROFILE_ID \
    --configuration-version 2 \
    --deployment-strategy-id $STRATEGY_ID
    # AppConfig watches the alarm during rollout
    # If alarm fires during bake time → automatic rollback to previous version

AppConfig vs SSM Parameter Store:
  SSM Parameter Store:
    → Simple key-value storage
    → No deployment strategies
    → No automatic rollback
    → No validation schemas
    → Immediate effect (no bake time)
    → Cheaper, simpler
    → Use for: static configuration, secrets, connection strings

  AppConfig:
    → Deployment lifecycle management (validate → deploy → bake → rollback)
    → Schema validation before deployment
    → Automatic rollback on CloudWatch alarm
    → Gradual rollout (not all instances at once)
    → Audit trail of all config changes
    → Use for: feature flags, runtime tunable config, A/B testing
    → Feature flags specifically: use AppConfig (it has native FF support)
```

> 💡 **Interview tip:** The key AppConfig differentiator over SSM Parameter Store: **deployment lifecycle**. SSM gives you a new value immediately (all consumers pick it up on next poll). AppConfig gives you control: validate first (JSON schema), roll out gradually (25% → 50% → 100%), monitor during bake time, and roll back automatically if alarms fire. For feature flags specifically: AppConfig has a dedicated **FeatureFlags schema** that supports percentages, segments, and rules — things SSM doesn't understand. The practical recommendation: use SSM for infrastructure config (DB endpoints, connection strings), use AppConfig for application behavior config (feature flags, rate limits, timeouts).

---

### Q346 — Kubernetes | Conceptual | Advanced

> Explain MutatingAdmissionWebhook for sidecar injection. Write a MutatingWebhookConfiguration that injects a logging sidecar into namespaces labeled inject-logging=true. What are failure modes?

#### Key Points to Cover:
```
How MutatingAdmissionWebhook works:
  1. kubectl apply / API request → kube-apiserver
  2. apiserver: authentication → authorization → admission webhooks
  3. MutatingAdmissionWebhook (your webhook): receives AdmissionReview object
  4. Webhook response: allow + JSON patch (mutations)
  5. apiserver applies patch → modified object stored in etcd
  6. ValidatingAdmissionWebhook runs (after mutation)

How Istio sidecar injection works:
  → Webhook watches Pod creation in labeled namespaces
  → Pod spec: nginx container only
  → Webhook response: add istio-proxy sidecar + init containers
  → Pod stored with 2 containers: nginx + istio-proxy
  → User never manually adds sidecar — automatic

MutatingWebhookConfiguration for logging sidecar:

apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: logging-sidecar-injector
webhooks:
- name: sidecar-injector.logging.company.com
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Ignore        # see failure modes below
  namespaceSelector:
    matchLabels:
      inject-logging: "true"   # only applies to labeled namespaces
  objectSelector:
    matchExpressions:
    - key: logging-sidecar-injected
      operator: DoesNotExist   # skip if already injected
  rules:
  - apiGroups:   [""]
    apiVersions: ["v1"]
    resources:   ["pods"]
    operations:  ["CREATE"]    # only on pod creation, not updates
  clientConfig:
    service:
      name:      sidecar-injector
      namespace: logging-system
      path:      /inject
    caBundle: <BASE64_CA_CERT>  # webhook server's CA cert

Webhook Server Response (Go-style pseudo-code):
  func inject(w http.ResponseWriter, r *http.Request) {
    // Decode AdmissionReview
    ar := decodeAdmissionReview(r.Body)
    pod := ar.Request.Object  // the Pod being created
    
    // Check if sidecar already present
    for _, c := range pod.Spec.Containers {
      if c.Name == "logging-sidecar" { return allow(ar, nil) }
    }
    
    // Build JSON patch to add sidecar
    patch := []JSONPatch{
      {Op: "add", Path: "/spec/containers/-", Value: Container{
        Name:  "logging-sidecar",
        Image: "company/log-forwarder:v2",
        VolumeMounts: []VolumeMount{{
          Name: "app-logs", MountPath: "/var/log/app",
        }},
        Resources: ResourceRequirements{
          Limits:   {"memory": "64Mi", "cpu": "50m"},
          Requests: {"memory": "32Mi", "cpu": "10m"},
        },
      }},
      // Also add volume if not present
      {Op: "add", Path: "/spec/volumes/-", Value: Volume{
        Name: "app-logs", EmptyDir: &EmptyDirVolumeSource{},
      }},
      // Label pod to prevent double-injection
      {Op: "add", Path: "/metadata/labels/logging-sidecar-injected", Value: "true"},
    }
    
    return allow(ar, patch)
  }

Failure Modes and failurePolicy:

  failurePolicy: Ignore (recommended for non-critical webhooks):
    → If webhook is unreachable or times out → allow request anyway
    → Pod created WITHOUT sidecar (sidecar injection missed)
    → Acceptable for: logging sidecars, optional telemetry
    → Risk: silent failure — pod runs without expected sidecar

  failurePolicy: Fail (use for security-critical webhooks):
    → If webhook is unreachable or times out → REQUEST DENIED
    → Pod NOT created
    → Critical issue if webhook is down → no pods can be created in labeled NS
    → Required for: PSP replacement, OPA/Gatekeeper, security policies

  Other failure scenarios:
  → Webhook returns non-200: admission denied
  → Webhook timeout (default 10s): failurePolicy determines outcome
  → Invalid JSON patch: pod creation fails
  → CA cert mismatch: TLS handshake fails → webhook unreachable

  HA requirements:
  → Webhook server: minimum 2 replicas (avoid single point of failure)
  → PodAntiAffinity: spread replicas across nodes
  → Not in a namespace that the webhook itself controls (bootstrap problem)
  → Webhook timeout should be generous: 10-30s
```

> 💡 **Interview tip:** The **failurePolicy** setting is the most operationally critical decision for admission webhooks. With `failurePolicy: Fail` on a webhook that is itself deployed via Kubernetes Deployments: if the webhook becomes unavailable (its own pods are evicted, updated, OOMKilled), **no new pods can be created cluster-wide** — including the pods needed to restart the webhook itself. This is a deadlock. Best practice: deploy webhook servers with `hostNetwork: true` or in a separate, privileged namespace, with `failurePolicy: Fail` only for genuinely security-critical checks (like policy enforcement) and `failurePolicy: Ignore` for operational convenience features (like sidecar injection).

---

### Q347 — GitHub Actions | Scenario-Based | Advanced

> Implement GitHub Actions for GitOps: build and push Docker image, update image tag in a separate GitOps repo, create PR for production, comment on original PR.

#### Key Points to Cover:
```yaml
name: GitOps Image Promotion

on:
  push:
    branches: [main]

jobs:
  build-and-promote:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write   # comment on original PR
      id-token: write        # OIDC for ECR

    outputs:
      image-tag: ${{ steps.vars.outputs.image-tag }}
      gitops-pr-url: ${{ steps.create-pr.outputs.pr-url }}

    steps:
    - uses: actions/checkout@v4

    - name: Set variables
      id: vars
      run: |
        SHORT_SHA="${GITHUB_SHA::8}"
        echo "image-tag=${SHORT_SHA}" >> $GITHUB_OUTPUT
        echo "IMAGE_TAG=${SHORT_SHA}" >> $GITHUB_ENV

    # Build and push image to ECR
    - name: Configure AWS credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-actions-ecr
        aws-region: us-east-1

    - name: Login and push to ECR
      run: |
        ECR="123456789012.dkr.ecr.us-east-1.amazonaws.com"
        aws ecr get-login-password | docker login --username AWS --password-stdin $ECR
        docker build -t $ECR/payment-service:$IMAGE_TAG .
        docker push $ECR/payment-service:$IMAGE_TAG
        echo "Pushed: $ECR/payment-service:$IMAGE_TAG"

    # Update GitOps repo
    - name: Clone GitOps repository
      uses: actions/checkout@v4
      with:
        repository: your-org/gitops-manifests
        token: ${{ secrets.GITOPS_TOKEN }}  # PAT with repo write access
        path: gitops-repo

    - name: Update image tag in Helm values
      working-directory: gitops-repo
      run: |
        # Update image tag in values file
        sed -i "s|tag: .*|tag: $IMAGE_TAG|g" \
          apps/payment-service/production/values.yaml
        
        # Verify change:
        grep "tag:" apps/payment-service/production/values.yaml

    - name: Commit and push to feature branch in GitOps repo
      working-directory: gitops-repo
      run: |
        git config user.name "github-actions[bot]"
        git config user.email "github-actions[bot]@users.noreply.github.com"
        
        BRANCH="promote/payment-service-${IMAGE_TAG}"
        git checkout -b $BRANCH
        git add apps/payment-service/production/values.yaml
        git commit -m "chore: promote payment-service to ${IMAGE_TAG}

        Source: ${{ github.repository }}@${GITHUB_SHA}
        Build: ${{ github.run_id }}
        Triggered by: ${{ github.actor }}"
        
        git push origin $BRANCH
        echo "GITOPS_BRANCH=$BRANCH" >> $GITHUB_ENV

    - name: Create PR in GitOps repo for production
      id: create-pr
      uses: actions/github-script@v7
      with:
        github-token: ${{ secrets.GITOPS_TOKEN }}
        script: |
          const { data: pr } = await github.rest.pulls.create({
            owner: 'your-org',
            repo: 'gitops-manifests',
            title: `Promote payment-service to ${process.env.IMAGE_TAG}`,
            body: `## Promotion Request
            
            **Service:** payment-service
            **Image Tag:** \`${process.env.IMAGE_TAG}\`
            **Source Commit:** ${context.sha}
            **Source Repo:** ${context.repo.owner}/${context.repo.repo}
            
            This PR was auto-generated by the CI/CD pipeline.
            Review and merge to deploy to production.`,
            head: process.env.GITOPS_BRANCH,
            base: 'main'
          });
          
          core.setOutput('pr-url', pr.html_url);
          core.setOutput('pr-number', pr.number);
          console.log(`Created PR: ${pr.html_url}`);

    # Comment on the original PR that triggered this workflow
    - name: Comment on source PR with GitOps PR link
      if: github.event.pull_request.number != null
      uses: actions/github-script@v7
      with:
        script: |
          // Find the PR that was merged
          const { data: prs } = await github.rest.repos.listPullRequestsAssociatedWithCommit({
            owner: context.repo.owner,
            repo: context.repo.repo,
            commit_sha: context.sha
          });
          
          if (prs.length > 0) {
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: prs[0].number,
              body: `## 🚀 Deployment Initiated
              
              Docker image \`payment-service:${{ steps.vars.outputs.image-tag }}\` has been built and pushed.
              
              A GitOps PR has been created for production deployment:
              ${{ steps.create-pr.outputs.pr-url }}
              
              Once the GitOps PR is merged, ArgoCD will deploy to production automatically.`
            });
          }
```

> 💡 **Interview tip:** The **separate GitOps repository** pattern (also called "push-based GitOps") is the production standard. Application code lives in the app repo; deployment manifests live in the GitOps repo. Benefits: (1) clear audit trail — who approved what deployment, (2) rollback = revert a commit in GitOps repo, (3) developers don't need prod cluster access — ArgoCD watches the GitOps repo and applies changes. The key security consideration: the PAT (`GITOPS_TOKEN`) needs write access to the GitOps repo — use a **GitHub App** instead of a PAT for better security (no personal token tied to an employee account). Also mention: for production merges, the GitOps PR should require **manual review and approval** — automated everything up to the point of production merge, then require human sign-off.

---

### Q348 — Terraform | Scenario-Based | Advanced

> Write Terraform postconditions to validate RDS Multi-AZ, ALB subnet configuration, ECS desired count, and S3 versioning/encryption. Explain postcondition vs check block behavior.

#### Key Points to Cover:
```hcl
# Postconditions run AFTER resource is created
# If they fail: Terraform marks the resource as tainted + errors
# Unlike preconditions (before creation), postconditions validate the real state

# 1. RDS Multi-AZ validation:
resource "aws_db_instance" "main" {
  identifier     = "prod-db"
  engine         = "postgres"
  multi_az       = var.multi_az_enabled
  instance_class = "db.t3.large"

  lifecycle {
    postcondition {
      condition     = self.multi_az == true
      error_message = "RDS instance ${self.identifier} must be Multi-AZ. Current value: ${self.multi_az}. Set multi_az = true."
    }
    postcondition {
      condition     = self.status == "available"
      error_message = "RDS instance is not in 'available' state after creation. Current state: ${self.status}"
    }
  }
}

# 2. ALB with at least 2 subnets in different AZs:
resource "aws_lb" "main" {
  name               = "prod-alb"
  internal           = false
  load_balancer_type = "application"
  subnets            = var.public_subnet_ids  # at least 2 in different AZs

  lifecycle {
    postcondition {
      condition     = length(self.subnets) >= 2
      error_message = "ALB requires at least 2 subnets for high availability. Currently configured with ${length(self.subnets)} subnet(s)."
    }
    # Validate subnets are in different AZs:
    postcondition {
      condition = length(toset([
        for az in data.aws_subnet.alb_subnets[*].availability_zone : az
      ])) >= 2
      error_message = "ALB subnets must be in at least 2 different Availability Zones."
    }
  }
}

# Helper data source for AZ validation:
data "aws_subnet" "alb_subnets" {
  count = length(var.public_subnet_ids)
  id    = var.public_subnet_ids[count.index]
}

# 3. ECS service desired count matches running count (within timeout):
# NOTE: For time-based validation, use terraform_data + local-exec
# Postconditions can't wait — they evaluate synchronously right after creation

resource "aws_ecs_service" "app" {
  name            = "payment-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count

  lifecycle {
    postcondition {
      # This checks immediately — may be 0 right after creation
      # For production: use terraform_data with aws ecs wait command
      condition     = self.desired_count > 0
      error_message = "ECS service desired count must be > 0. Got: ${self.desired_count}"
    }
  }
}

# Better approach for ECS stability — use terraform_data:
resource "terraform_data" "ecs_health_check" {
  triggers_replace = [aws_ecs_service.app.task_definition]

  provisioner "local-exec" {
    command = <<-EOT
      echo "Waiting for ECS service to stabilize..."
      aws ecs wait services-stable \
        --cluster ${aws_ecs_cluster.main.name} \
        --services ${aws_ecs_service.app.name} \
        --region ${var.aws_region}
      
      RUNNING=$(aws ecs describe-services \
        --cluster ${aws_ecs_cluster.main.name} \
        --services ${aws_ecs_service.app.name} \
        --query 'services[0].runningCount' --output text)
      DESIRED=$(aws ecs describe-services \
        --cluster ${aws_ecs_cluster.main.name} \
        --services ${aws_ecs_service.app.name} \
        --query 'services[0].desiredCount' --output text)
      
      if [ "$RUNNING" != "$DESIRED" ]; then
        echo "ERROR: Running count ($RUNNING) != Desired count ($DESIRED)"
        exit 1
      fi
    EOT
  }
  depends_on = [aws_ecs_service.app]
}

# 4. S3 bucket versioning and encryption:
resource "aws_s3_bucket" "data" {
  bucket = var.bucket_name

  lifecycle {
    postcondition {
      # Verify bucket is in expected region
      condition     = self.region == var.aws_region
      error_message = "S3 bucket must be in ${var.aws_region}. Got: ${self.region}"
    }
  }
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
  }
}

# Validation that versioning is actually enabled (using check block):
check "s3_bucket_versioning_enabled" {
  data "aws_s3_bucket" "data_check" {
    bucket = aws_s3_bucket.data.bucket
  }
  assert {
    condition     = data.aws_s3_bucket.data_check.versioning[0].enabled == true
    error_message = "S3 bucket ${aws_s3_bucket.data.bucket} versioning is not enabled."
  }
}

# Postcondition vs Check Block:
# postcondition: tied to a specific resource lifecycle
#   - Runs after the resource is applied
#   - FAILURE: marks resource as tainted, apply fails
#   - Use for: validating the resource you just created
#
# check block (TF 1.5+): independent assertion
#   - Can reference any data/resource
#   - FAILURE: warning only (apply still succeeds)
#   - Use for: cross-resource invariants, compliance checks
#   - Good for: "this S3 bucket in ANOTHER config has versioning"
```

> 💡 **Interview tip:** The key distinction: **postcondition = fatal, check block = advisory**. A postcondition failure stops the apply and taints the resource — it's a hard stop for "this infrastructure doesn't meet our requirements." A check block failure is a warning that still lets the apply succeed — useful for compliance checks that shouldn't block deployment but should be visible. The practical use case for check blocks: organization-wide compliance assertions in a shared module, where you want to warn teams that their configuration violates a policy without blocking their deployment immediately. For production infrastructure requirements (Multi-AZ, encryption) — use postconditions. For "nice to have" or audit-style checks — use check blocks.

---

### Q349 — Prometheus | Troubleshooting | Advanced

> Your Prometheus metrics show a 15-minute gap between 14:00 and 14:15. Everything is normal after. Walk through diagnosing the cause: WAL recovery, storage issues, network partition, restart.

#### Key Points to Cover:
```
Step 1 — Determine if Prometheus restarted:
  # Check Prometheus uptime metric:
  process_start_time_seconds
  # If value is > 14:00 epoch time → Prometheus restarted between 14:00-14:15
  
  # Check systemd/container logs:
  journalctl -u prometheus --since "14:00" --until "14:20"
  kubectl logs -n monitoring prometheus-0 --since=1h | grep -E "restart|panic|OOM|kill"
  
  # Check Prometheus TSDB status:
  curl http://prometheus:9090/api/v1/status/tsdb
  # Look for: lastBlock.maxTime, headStats.minTime

Step 2 — WAL (Write-Ahead Log) analysis:
  # If Prometheus restarted: WAL must be replayed on startup
  # WAL replay time depends on: WAL size (2 hours of data by default)
  # Large WAL = longer startup = longer gap before data appears
  
  # WAL location: /prometheus/wal/
  ls -la /prometheus/wal/  # check WAL file sizes and modification times
  
  # In Prometheus startup logs look for:
  "replaying WAL, this may take a while"
  "WAL replay completed"
  # Time between these = duration of gap (WAL replay is not collecting data)
  
  # WAL recovery failure shows:
  "error replaying WAL"
  → If WAL corrupted: data before crash permanently lost, Prometheus starts fresh

Step 3 — Check if targets were unreachable (network partition):
  # Prometheus scrape failure metric:
  up{job="my-service"} == 0
  # Or:
  rate(scrape_duration_seconds[5m])  # zero scrapes during gap
  
  # Look at Prometheus targets page:
  http://prometheus:9090/targets
  # Check: lastScrape, lastError for each target
  
  # If targets were down but Prometheus was up:
  # → Metrics show gap for THOSE targets only
  # → Node metrics (node_exporter) might still be present
  # → This is a network partition between Prometheus and targets
  
  # Key differentiator:
  # Prometheus restart → ALL metrics have gap simultaneously
  # Network partition → only affected targets have gap

Step 4 — Storage issues:
  # Check disk space at time of gap (CloudWatch, node exporter):
  node_filesystem_avail_bytes{mountpoint="/prometheus"}
  # If disk hit 0 → Prometheus stops writing → gap
  
  # Check TSDB compaction errors:
  prometheus_tsdb_compaction_failed_total
  # Compaction failure → head block not flushed → gap
  
  # Prometheus OOM during compaction:
  # Large cardinality → OOM killer kills Prometheus → gap
  # Check: container_oom_events_total or kernel logs

Step 5 — Correlate with Prometheus own metrics:
  # These are scraped by Prometheus itself and survive restarts (stored in TSDB):
  prometheus_tsdb_head_min_time     # earliest time in memory
  prometheus_tsdb_head_max_time     # latest time in memory
  prometheus_notifications_dropped_total
  scrape_samples_scraped            # samples collected per scrape
  
  # If you see a gap in prometheus_tsdb_head_max_time progressing:
  # → Prometheus was running but not scraping (network issue)
  
  # If prometheus_tsdb_head_max_time jumps from 14:00 to 14:15:
  # → Prometheus was down, WAL replay covers 14:00-14:15 with old data

Definitive diagnosis:
  Restart + WAL replay:
    → process_start_time_seconds shows 14:00-14:15 start
    → Startup logs show "replaying WAL"
    → ALL metrics have gap simultaneously

  Network partition from targets:
    → Prometheus uptime shows no restart
    → up{} == 0 for affected targets during 14:00-14:15
    → Not all metrics affected (only ones from unreachable targets)

  Disk full:
    → node_filesystem_avail_bytes near 0 at 14:00
    → prometheus_tsdb_head_samples_appended_total stops incrementing

  OOM kill:
    → container_oom_events_total spike at 14:00
    → process_start_time_seconds shows restart
    → Check: prometheus memory limits vs process_resident_memory_bytes
```

> 💡 **Interview tip:** The fastest diagnosis: **check if the gap affects ALL metrics simultaneously or just some targets**. If it's all metrics at once — Prometheus itself was the problem (restart, storage). If only specific targets — network partition between Prometheus and those targets. The WAL replay duration is a common surprise: Prometheus WAL stores 2 hours of data by default. If Prometheus has high cardinality (millions of time series), the WAL can be several GB, and replay takes minutes — during which no new data is collected, creating a gap even though the Prometheus process started quickly. Tuning `--storage.tsdb.min-block-duration` and increasing scrape memory help reduce WAL size.

---

### Q350 — AWS | Conceptual | Advanced

> Explain AWS CloudFront — origin, distribution, behavior, cache policy. TTL vs Cache-Control vs invalidation. Signed URLs for private S3 content. CloudFront vs API Gateway for API entry point.

#### Key Points to Cover:
```
Core Concepts:

  Distribution:
    → The CloudFront deployment (has a domain: xxx.cloudfront.net)
    → Global: 400+ edge locations worldwide
    → Can have multiple origins and behaviors

  Origin:
    → Where CloudFront fetches content FROM (cache miss)
    → Types: S3 bucket, ALB, EC2, API Gateway, custom HTTP
    → Origin Access Control (OAC): only CloudFront can access private S3

  Behavior:
    → Rules matching URL path patterns → different origin + cache settings
    → Default behavior: /∗ → catch-all
    → Example:
      /api/* → origin: ALB (no caching)
      /static/* → origin: S3 (long cache)
      /* → origin: S3 (index.html, short cache)

  Cache Policy:
    → Defines: TTL values, which headers/cookies/query strings affect cache key
    → AWS managed policies: CachingOptimized, CachingDisabled, etc.
    → Custom: fine-grained control

TTL vs Cache-Control vs Invalidation:
  TTL (in CloudFront cache policy):
    → Default TTL: 86400s (1 day) — used when no Cache-Control header
    → Min TTL: minimum even if Cache-Control says shorter
    → Max TTL: maximum even if Cache-Control says longer

  Cache-Control headers (from origin):
    → Cache-Control: max-age=3600 → tells CloudFront cache for 1 hour
    → Cache-Control: no-cache → CloudFront still caches, but revalidates
    → Cache-Control: no-store → CloudFront does NOT cache
    → s-maxage: overrides max-age for shared caches (like CloudFront)

  Priority order (highest wins):
    1. If Cache-Control: no-store → not cached
    2. If s-maxage present → use s-maxage
    3. If max-age present → use max-age (capped by MaxTTL)
    4. If no Cache-Control → use DefaultTTL

  Invalidation:
    → Removes cached objects BEFORE TTL expires
    → Costs: first 1000 paths/month free, then $0.005/path
    → Wildcard: /static/* invalidates all paths under /static/
    → NOT instant: can take up to 60 seconds to propagate globally
    → Use for: emergency fixes, breaking bug deployed to CDN
    → Better pattern: versioned asset filenames (/static/app.v2.js)
      → No invalidation needed — new version = new URL = cache miss naturally

Signed URLs for private S3 content:
  Architecture:
  User → CloudFront signed URL → CloudFront validates signature
       → Origin (S3 with OAC, private) → returns object

  Step 1 — Create CloudFront key pair:
    → Generate RSA key pair
    → Upload public key to CloudFront → get Key ID
    → Store private key securely (Secrets Manager)

  Step 2 — Python: generate signed URL:
  from cryptography.hazmat.primitives import hashes
  from cryptography.hazmat.primitives.asymmetric import padding
  import datetime, base64, json

  def generate_signed_url(url, expire_minutes=60):
      expiry = int((datetime.datetime.now() +
                    datetime.timedelta(minutes=expire_minutes)).timestamp())
      policy = json.dumps({
          "Statement": [{
              "Resource": url,
              "Condition": {"DateLessThan": {"AWS:EpochTime": expiry}}
          }]
      })
      # Sign policy with private key
      signature = private_key.sign(policy.encode(), padding.PKCS1v15(), hashes.SHA1())
      # Encode for URL
      encoded_sig = base64.b64encode(signature).decode()
      return f"{url}?Expires={expiry}&Signature={encoded_sig}&Key-Pair-Id=ABCDEF123456"

  Step 3 — S3 bucket: block all public access, deny direct S3 access
  # S3 bucket policy only allows CloudFront OAC:
  {
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Condition": {"StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT:distribution/DIST_ID"
    }}
  }

CloudFront vs API Gateway (for API entry point):
  CloudFront:
    ✅ Global edge caching for GET responses (reduce origin load)
    ✅ DDoS protection (AWS Shield)
    ✅ WAF integration (Layer 7 filtering)
    ✅ Custom domains with ACM certs — free
    ✅ Lower latency for cacheable content
    ✅ Useful when your API has cacheable endpoints (/products, /catalog)
    ❌ No request transformation, validation, throttling
    ❌ Not serverless-native (no Lambda integration)

  API Gateway:
    ✅ Built-in throttling per-client
    ✅ Request/response transformation
    ✅ Direct Lambda integration
    ✅ Usage plans, API keys
    ✅ Websocket support
    ✅ JWT authorizers (Cognito)
    ❌ Pricing: per-request (expensive at high scale)
    ❌ Limited edge caching (CloudFront needed in front for CDN)

  Combined pattern (production):
  User → CloudFront (WAF, DDoS, cache static responses)
       → API Gateway (throttling, auth, Lambda integration)
       → Backend services
```

> 💡 **Interview tip:** The CloudFront vs API Gateway question tests **architectural judgment**. The key insight: they're not alternatives — **they're complementary layers**. CloudFront provides global edge presence, WAF, and caching. API Gateway provides request lifecycle management (auth, throttling, transformation). The mature production pattern has both: CloudFront in front of API Gateway gives you global DDoS protection + caching for cacheable API responses + API management features for backend logic. Cost optimization: cache `/products` and `/catalog` at CloudFront for 60 seconds — if you have 10,000 rps, that's 10,000 requests saved per minute from hitting API Gateway (which charges per-request).

---

### Q351 — Kubernetes | Scenario-Based | Advanced

> Implement zero-trust networking in Kubernetes using Cilium eBPF CNI. Explain eBPF vs iptables, Cilium's L7 capabilities, Hubble observability, and write a L7 HTTP method restriction policy.

#### Key Points to Cover:
```
What is eBPF:
  → Extended Berkeley Packet Filter
  → Allows running sandboxed programs inside the Linux kernel
  → Without modifying kernel source or loading kernel modules
  → Runs in kernel space: near hardware-speed processing
  → Programs verified by eBPF verifier before execution (safe)

Why Cilium uses eBPF instead of iptables:

  iptables limitations:
    → O(N) rule processing: 10,000 services = 10,000 iptables rules checked per packet
    → Not kernel-native for modern networking: workaround, not designed for K8s scale
    → No L7 awareness: can only filter by IP/port, not HTTP paths/methods
    → Conntrack table: bottleneck at high connection rates
    → Complex debugging: iptables-save output is hard to read

  eBPF advantages:
    → O(1) hash table lookups (not linear rule scanning)
    → Kernel-native: hooks directly into network stack
    → L7 aware: can parse HTTP, gRPC, DNS inside kernel
    → Observable: bpftrace, bpftool for real-time introspection
    → Programmable: modify behavior without kernel patches

Cilium L7 Policy — HTTP method restriction:

  # Standard Kubernetes NetworkPolicy: only L3/L4 (IP + port)
  # Cilium NetworkPolicy: extends to L7 (HTTP methods, paths, headers)

  apiVersion: cilium.io/v2
  kind: CiliumNetworkPolicy
  metadata:
    name: payment-api-l7-policy
    namespace: production
  spec:
    endpointSelector:
      matchLabels:
        app: payment-api     # applies to payment-api pods

    ingress:
    # Allow frontend to GET/POST /payments/* but NOT DELETE
    - fromEndpoints:
      - matchLabels:
          app: frontend
      toPorts:
      - ports:
        - port: "8080"
          protocol: TCP
        rules:
          http:
          - method: "GET"
            path: "/payments/.*"
          - method: "POST"
            path: "/payments"
          # No DELETE allowed — not in the allow list → blocked
          # No PUT allowed → blocked
          # Cilium HTTP L7 = whitelist: only listed methods allowed

    # Allow admin service full access:
    - fromEndpoints:
      - matchLabels:
          app: admin-service
      toPorts:
      - ports:
        - port: "8080"
          protocol: TCP
        rules:
          http:
          - method: ".*"    # allow all methods
            path: "/.*"

    # DNS egress (required for service discovery):
    egress:
    - toEndpoints:
      - matchLabels:
          "k8s:io.kubernetes.pod.namespace": kube-system
    - toPorts:
      - ports:
        - port: "53"
          protocol: UDP
        rules:
          dns:
          - matchPattern: "*.cluster.local"  # only cluster DNS
          # Block: exfiltration via DNS (no external DNS queries from app pods)

Hubble — Cilium's observability:
  → Distributed networking and security observability platform
  → Real-time: which pod talks to which, over which port/protocol
  → Drop visibility: see WHICH policy dropped WHICH packet and why
  → No code changes: eBPF hooks provide visibility transparently

  kubectl exec -n kube-system cilium-xxx -- hubble observe \
    --namespace production \
    --pod payment-api \
    --last 100 \
    --protocol http

  # Output:
  # FORWARDED  frontend → payment-api:8080 GET /payments/123
  # DROPPED    frontend → payment-api:8080 DELETE /payments/123 (reason: L7 policy)
  # FORWARDED  admin-service → payment-api:8080 DELETE /payments/123

  # Hubble UI: graph showing all pod-to-pod communication
  # Can see: which flows are blocked, which are allowed, policy violations
```

> 💡 **Interview tip:** The eBPF vs iptables comparison is the heart of this question. The concrete performance number: iptables with 10,000 services requires checking ~40,000 rules per packet (each service adds multiple rules). eBPF uses O(1) hash maps — same performance for 10,000 services as for 10. This isn't theoretical — Facebook reported 10x better throughput moving from iptables to eBPF (they contributed early Cilium work). For the L7 policy, emphasize: **this is enforcement in the kernel**, not in a proxy sidecar. Istio/Envoy do L7 enforcement in a userspace proxy (sidecar) — each request goes: kernel → envoy sidecar → application → envoy sidecar → kernel. Cilium does it in the kernel itself — no userspace hop, lower latency, no sidecar pod to manage.

---

### Q352 — ArgoCD | Conceptual | Advanced

> Explain ArgoCD Diffing Customization. Global ignoreDifferences in ConfigMap vs per-application. Write configs to globally ignore caBundle, status fields, and rollingUpdate changes.

#### Key Points to Cover:
```
Why ignoreDifferences is needed:
  → Kubernetes often mutates resources after ArgoCD applies them
  → ArgoCD compares: Git state vs live state → sees difference → OutOfSync
  → But the difference is EXPECTED (Kubernetes behavior, not a drift)
  → ignoreDifferences: tell ArgoCD "this specific difference is normal, ignore it"

Two levels of configuration:

Level 1 — Per-Application ignoreDifferences:
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  spec:
    ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
      - /spec/replicas         # HPA manages replicas
    - group: ""
      kind: Service
      jsonPointers:
      - /spec/clusterIP        # K8s assigns clusterIP, GitOps doesn't set it

  Use for: app-specific exceptions that don't apply globally

Level 2 — Global resource.customizations in ArgoCD ConfigMap:
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: argocd-cm
    namespace: argocd
  data:
    # Global ignore: applies to ALL applications across ALL clusters
    
    # 1. Ignore caBundle in all MutatingWebhookConfigurations:
    # (cert-manager or the webhook controller sets this after creation)
    resource.customizations.ignoreDifferences.admissionregistration.k8s.io_MutatingWebhookConfiguration: |
      jqPathExpressions:
      - '.webhooks[]?.clientConfig.caBundle'

    # 2. Ignore status field in all custom resources:
    # (status is controller-managed, never in Git)
    resource.customizations.ignoreDifferences.apiextensions.k8s.io_CustomResourceDefinition: |
      jqPathExpressions:
      - '.status'
    
    # For any specific CRD (e.g., your company's AppConfig CR):
    resource.customizations.ignoreDifferences.mycompany.io_AppConfig: |
      jqPathExpressions:
      - '.status'

    # 3. Ignore rollingUpdate strategy changes in all Deployments:
    # (some admission webhooks normalize rolling update settings)
    resource.customizations.ignoreDifferences.apps_Deployment: |
      jqPathExpressions:
      - '.spec.template.spec.containers[].image'  # might want to track this though!
      - '.spec.strategy.rollingUpdate'

    # More targeted — only the specific fields within rollingUpdate:
    resource.customizations.ignoreDifferences.apps_Deployment: |
      jqPathExpressions:
      - '.spec.strategy.rollingUpdate.maxSurge'
      - '.spec.strategy.rollingUpdate.maxUnavailable'

    # 4. Combined real-world example:
    # Ignore K8s-added fields that ArgoCD doesn't understand:
    resource.customizations.ignoreDifferences.core_Service: |
      jqPathExpressions:
      - '.spec.clusterIP'
      - '.spec.clusterIPs'
      - '.spec.ipFamilies'
      - '.spec.ipFamilyPolicy'

    # 5. Health check customization (bonus):
    # Tell ArgoCD how to determine if a custom resource is Healthy:
    resource.customizations.health.mycompany.io_Database: |
      hs = {}
      if obj.status ~= nil then
        if obj.status.phase == "Running" then
          hs.status = "Healthy"
          hs.message = "Database is running"
        else
          hs.status = "Progressing"
          hs.message = "Database phase: " .. obj.status.phase
        end
      else
        hs.status = "Progressing"
        hs.message = "Status not yet available"
      end
      return hs

When to use global vs per-app:
  Global (resource.customizations):
    → Kubernetes-native behavior (clusterIP, caBundle, status)
    → Applies to all apps — no need to repeat in every Application YAML
    → Examples: MutatingWebhookConfiguration caBundle, Service clusterIP

  Per-app (Application.spec.ignoreDifferences):
    → Application-specific exceptions
    → HPA-managed replica counts (only for apps with HPA)
    → Custom replica or image settings for specific apps
    → When global would be too broad (affects things you DO want to track)
```

> 💡 **Interview tip:** The `caBundle` in MutatingWebhookConfiguration is the most commonly needed global ignoreDifferences in production. Here's why: when you deploy a webhook (like cert-manager, Istio, or your own), the caBundle field contains the TLS CA certificate. This is either injected by cert-manager AFTER ArgoCD creates the resource, or it's managed by the webhook's own controller. ArgoCD's Git source doesn't have this field (or has it empty), so every sync shows a diff. Without `ignoreDifferences`, ArgoCD keeps trying to clear the caBundle, which breaks the webhook. This affects virtually every cluster with admission webhooks — making it a perfect global customization rather than per-application.

---

### Q353 — Linux / Bash | Scenario-Based | Advanced

> Write a Bash script for rolling restart of application servers: primary/secondary ordering, drain connections, health check, pause on failure, rolling log, dry-run mode.

#### Key Points to Cover:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Configuration
INVENTORY_FILE="${1:-inventory.txt}"
HEALTH_CHECK_URL="http://{host}:8080/health"
HEALTH_CHECK_RETRIES=10
HEALTH_CHECK_INTERVAL=5
MAX_ACTIVE_CONNECTIONS=5
DRAIN_TIMEOUT=120
LOG_FILE="/var/log/rolling-restart-$(date +%Y%m%d_%H%M%S).log"
DRY_RUN=false

# Parse flags
for arg in "$@"; do
  case $arg in
    --dry-run) DRY_RUN=true ;;
  esac
done

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Logging
log() {
  local level="$1"; shift
  local message="$*"
  local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
  local line="${timestamp} [${level}] ${message}"
  echo -e "${line}" | tee -a "${LOG_FILE}"
}

info()  { log "INFO " "${GREEN}$*${NC}"; }
warn()  { log "WARN " "${YELLOW}$*${NC}"; }
error() { log "ERROR" "${RED}$*${NC}"; }

# Dry-run wrapper
run_cmd() {
  if [[ "${DRY_RUN}" == "true" ]]; then
    info "[DRY-RUN] Would execute: $*"
    return 0
  fi
  eval "$@"
}

# Parse inventory file
# Format:
#   [secondary]
#   web-01
#   web-02
#   [primary]
#   web-03
declare -a SECONDARY_SERVERS=()
declare -a PRIMARY_SERVERS=()

parse_inventory() {
  local current_group=""
  while IFS= read -r line || [[ -n "$line" ]]; do
    line="${line// /}"  # trim spaces
    [[ -z "$line" || "$line" == "#"* ]] && continue
    if [[ "$line" == "[secondary]" ]]; then
      current_group="secondary"
    elif [[ "$line" == "[primary]" ]]; then
      current_group="primary"
    else
      [[ "$current_group" == "secondary" ]] && SECONDARY_SERVERS+=("$line")
      [[ "$current_group" == "primary"   ]] && PRIMARY_SERVERS+=("$line")
    fi
  done < "$INVENTORY_FILE"
}

# Wait for active connections to drain
drain_connections() {
  local host="$1"
  local elapsed=0
  
  info "Draining connections on ${host}..."
  
  while [[ $elapsed -lt $DRAIN_TIMEOUT ]]; do
    local active_conns
    active_conns=$(ssh -o ConnectTimeout=5 "${host}" \
      "ss -tn state established | wc -l" 2>/dev/null || echo "999")
    
    info "  ${host}: active connections = ${active_conns}"
    
    if [[ "${active_conns}" -le "${MAX_ACTIVE_CONNECTIONS}" ]]; then
      info "  ${host}: connections drained (${active_conns} <= ${MAX_ACTIVE_CONNECTIONS})"
      return 0
    fi
    
    sleep 5
    elapsed=$((elapsed + 5))
  done
  
  warn "${host}: drain timeout after ${DRAIN_TIMEOUT}s — proceeding anyway (${active_conns} connections)"
  return 0
}

# Restart service on a host
restart_service() {
  local host="$1"
  info "Restarting service on ${host}..."
  run_cmd "ssh -o ConnectTimeout=10 '${host}' 'sudo systemctl restart myapp'"
}

# Wait for health check to pass
wait_for_health() {
  local host="$1"
  local url="${HEALTH_CHECK_URL//\{host\}/${host}}"
  local attempt=0
  
  info "Waiting for health check on ${host} (${url})..."
  
  while [[ $attempt -lt $HEALTH_CHECK_RETRIES ]]; do
    attempt=$((attempt + 1))
    
    if [[ "${DRY_RUN}" == "true" ]]; then
      info "  [DRY-RUN] Health check attempt ${attempt}/${HEALTH_CHECK_RETRIES} — simulating pass"
      return 0
    fi
    
    local status
    status=$(curl -s -o /dev/null -w "%{http_code}" \
      --max-time 5 "${url}" 2>/dev/null || echo "000")
    
    if [[ "${status}" == "200" ]]; then
      info "  ${host}: health check passed (HTTP ${status})"
      return 0
    fi
    
    warn "  ${host}: health check attempt ${attempt}/${HEALTH_CHECK_RETRIES} — HTTP ${status}"
    sleep "${HEALTH_CHECK_INTERVAL}"
  done
  
  return 1  # health check failed
}

# Process a single server
process_server() {
  local host="$1"
  
  info "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  info "Processing: ${host}"
  info "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  
  drain_connections "${host}"
  restart_service "${host}"
  
  if ! wait_for_health "${host}"; then
    error "HEALTH CHECK FAILED on ${host} — PAUSING ROLLOUT"
    error "Manual intervention required."
    error "To resume: fix ${host} and re-run script"
    
    # Alert (replace with your alerting mechanism):
    run_cmd "curl -sS -X POST '${SLACK_WEBHOOK:-}' \
      -d '{\"text\":\"🚨 Rolling restart PAUSED — health check failed on ${host}\"}' || true"
    
    exit 1  # halt the entire rollout
  fi
  
  info "✓ ${host}: successfully restarted and healthy"
  echo "${host}: SUCCESS $(date '+%Y-%m-%d %H:%M:%S')" >> "${LOG_FILE}.summary"
}

# Main
main() {
  [[ "${DRY_RUN}" == "true" ]] && warn "=== DRY RUN MODE — no actual changes will be made ==="
  
  parse_inventory
  
  info "Rolling restart plan:"
  info "  Secondary servers (${#SECONDARY_SERVERS[@]}): ${SECONDARY_SERVERS[*]:-none}"
  info "  Primary servers   (${#PRIMARY_SERVERS[@]}): ${PRIMARY_SERVERS[*]:-none}"
  info "  Log file: ${LOG_FILE}"
  info ""
  
  # Phase 1: Secondary servers first
  if [[ ${#SECONDARY_SERVERS[@]} -gt 0 ]]; then
    info "Phase 1: Restarting secondary servers..."
    for host in "${SECONDARY_SERVERS[@]}"; do
      process_server "${host}"
    done
    info "Phase 1 complete: all secondaries restarted"
  fi
  
  # Phase 2: Primary servers
  if [[ ${#PRIMARY_SERVERS[@]} -gt 0 ]]; then
    info "Phase 2: Restarting primary servers..."
    for host in "${PRIMARY_SERVERS[@]}"; do
      process_server "${host}"
    done
    info "Phase 2 complete: all primaries restarted"
  fi
  
  info "Rolling restart complete. Summary:"
  cat "${LOG_FILE}.summary" 2>/dev/null || true
}

main "$@"
```

> 💡 **Interview tip:** The key design decisions to explain: (1) **Halt on failure** — if any server fails its health check, the script exits immediately. You don't want to continue restarting healthy servers when one is already broken. (2) **Connection draining before restart** — reduces in-flight request failures. The `ss -tn state established | wc -l` command is more accurate than `netstat` on modern Linux. (3) **Dry-run mode** — this is the #1 feature SREs ask for in automation scripts. Running `--dry-run` on a production inventory before executing gives confidence without risk. (4) **Separate log file per run** with timestamp — makes post-incident review easy. Always mention that for Kubernetes workloads, `kubectl rollout restart` replaces this pattern — rolling restart scripts are for VM/bare-metal fleets.

---

### Q354 — AWS | Scenario-Based | Advanced

> Design a workflow where Kubernetes ServiceInstance objects (using ACK) provision RDS with approval workflow, inject connection details as Secrets, and deprovision on deletion.

#### Key Points to Cover:
```
Architecture — ACK (AWS Controllers for Kubernetes):
  ACK = Kubernetes operators that create AWS resources from K8s CRDs
  → Developer creates: DBInstance CRD in Kubernetes
  → ACK controller: detects CRD → calls AWS API → provisions RDS
  → Status synced back to CRD (endpoint, port, status)
  → No direct AWS console access for developers

Step 1 — Install ACK RDS Controller:
  helm install ack-rds-controller \
    oci://public.ecr.aws/aws-controllers-k8s/rds-chart \
    --namespace ack-system \
    --set aws.region=us-east-1 \
    --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT:role/ack-rds-role

Step 2 — Developer creates DBInstance CRD (self-service):
  apiVersion: rds.services.k8s.aws/v1alpha1
  kind: DBInstance
  metadata:
    name: team-alpha-postgres
    namespace: team-alpha
    annotations:
      # Request approval before provisioning:
      ack.aws.services/approval-required: "true"
  spec:
    dbInstanceIdentifier: team-alpha-postgres
    dbInstanceClass: db.t3.medium
    engine: postgres
    engineVersion: "15.2"
    masterUsername: admin
    masterUserPassword:
      namespace: team-alpha
      name: db-master-password    # Secret reference
      key: password
    allocatedStorage: 20
    storageEncrypted: true
    dbSubnetGroupName: private-subnet-group
    vpcSecurityGroupIDs: ["sg-xxxx"]

Step 3 — Approval Workflow:
  Option A: OPA/Gatekeeper policy (automated approval):
  → Validate: instance class is in allowed list
  → Validate: storage ≤ 100GB for self-service
  → Reject: anything requiring manual review

  Option B: Kubernetes approval annotation:
  → Webhook intercepts DBInstance creation
  → If annotation "approval-required: true":
    → Creates Approval CRD (custom resource)
    → Platform team reviews and adds annotation: "approved: true"
    → Webhook allows creation
  → Platform team gets Slack notification to review

  Option C: GitHub PR-based approval (GitOps):
  → Developer creates DBInstance YAML in git
  → PR requires platform team approval
  → Merge triggers ArgoCD sync → ACK controller provisions

Step 4 — Connection details injected as K8s Secret:
  # ACK automatically creates a FieldExport or Secret reference:
  apiVersion: services.k8s.aws/v1alpha1
  kind: FieldExport
  metadata:
    name: postgres-endpoint
    namespace: team-alpha
  spec:
    from:
      resource:
        group: rds.services.k8s.aws
        kind: DBInstance
        name: team-alpha-postgres
      fieldPath: ".status.endpoint.address"
    to:
      name: app-db-config    # target Secret name
      namespace: team-alpha
      kind: Secret
      key: DB_HOST

  # After ACK provisions RDS, automatically creates/updates:
  # Secret team-alpha/app-db-config:
  #   DB_HOST: team-alpha-postgres.xxxxxx.us-east-1.rds.amazonaws.com

  # Application uses the Secret:
  envFrom:
  - secretRef:
      name: app-db-config

Step 5 — Deprovisioning on DBInstance deletion:
  → Developer: kubectl delete dbinstance team-alpha-postgres
  → ACK controller: detects deletion → calls RDS DeleteDBInstance API
  → RDS: creates final snapshot (if configured), then deletes
  → K8s: DBInstance CRD removed after AWS deletion confirmed
  
  # Protection against accidental deletion:
  spec:
    deletionProtection: true    # RDS-level: cannot delete with data
    # + Kubernetes finalizer added by ACK controller
    # → DBInstance cannot be deleted from K8s until AWS deletion completes
  
  # Terraform-style prevent_destroy equivalent for ACK:
  metadata:
    annotations:
      rds.services.k8s.aws/deletion-policy: Retain  # don't delete AWS resource

Service Catalog integration pattern:
  # AWS Service Catalog portfolio → approved products → developer self-service
  # Products defined as: CloudFormation templates OR ACK CRDs
  # Approval workflows: Service Catalog supports approval with SNS notifications
```

> 💡 **Interview tip:** ACK vs AWS Service Catalog: ACK is the **lower-level** tool (Kubernetes-native, CRD-per-resource), Service Catalog is the **higher-level** tool (pre-packaged products with approval gates). They're complementary: use ACK as the provisioning engine under the hood, with a Service Catalog product wrapping it for the developer experience. The killer feature of this pattern: developers never need AWS console access or IAM permissions to provision databases — they just create a CRD in their own namespace. The ACK controller (with appropriate IAM via IRSA) handles all AWS API calls. This satisfies both developer self-service AND security team requirements for least-privilege AWS access.

---

### Q355 — Kubernetes | Conceptual | Advanced

> Explain Kubernetes ExternalName services and headless services. How do headless services enable direct pod-to-pod communication for StatefulSets?

#### Key Points to Cover:
```
ExternalName Service:
  → A Service that is just a DNS CNAME alias
  → No selector, no endpoints, no proxying
  → When pod queries the service name → DNS returns the CNAME target
  → Useful for: external services that need a Kubernetes-native DNS name

  apiVersion: v1
  kind: Service
  metadata:
    name: external-database
    namespace: production
  spec:
    type: ExternalName
    externalName: prod-rds.cluster-xxxx.us-east-1.rds.amazonaws.com

  # Now pods use: external-database.production.svc.cluster.local
  # DNS resolves to: prod-rds.cluster-xxxx.us-east-1.rds.amazonaws.com
  # Benefit: if RDS endpoint changes → update one place (Service) not all apps

  Use cases:
  → Abstracting external databases, APIs, services behind K8s DNS
  → Migration path: start with ExternalName pointing to old service,
    migrate to in-cluster service, switch ExternalName → ClusterIP, zero downtime
  → Cross-namespace references: Service in namespace-A as ExternalName
    pointing to service in namespace-B (workaround before cross-namespace support)

  Limitations:
  → Does NOT work with TLS SNI validation (CNAME resolves before TLS)
  → Cannot use with HTTP host-header based routing
  → No load balancing (it's just a DNS alias)

Headless Services (clusterIP: None):
  → NO cluster IP assigned
  → DNS returns: A records for EACH backing pod directly
  → No kube-proxy involvement: traffic goes directly to pod IPs
  → Selector-based headless: returns individual pod IPs
  → No-selector headless: you manage endpoints manually

  apiVersion: v1
  kind: Service
  metadata:
    name: postgres-headless
    namespace: production
  spec:
    clusterIP: None       ← THIS makes it headless
    selector:
      app: postgres
    ports:
    - port: 5432

  DNS behavior comparison:
    Regular ClusterIP Service (my-svc.ns.svc.cluster.local):
      → DNS returns: single stable VIP (e.g., 10.96.45.100)
      → kube-proxy load-balances to one of the pod IPs
      → Client never knows which pod it's talking to

    Headless Service (postgres-headless.ns.svc.cluster.local):
      → DNS returns: ALL pod IPs (A records for each pod)
        10.244.1.5 (postgres-0)
        10.244.2.3 (postgres-1)
        10.244.3.7 (postgres-2)
      → Client directly connects to a specific pod
      → Client does its own load balancing / pod selection

StatefulSet + Headless Service — stable pod DNS:
  StatefulSet "postgres" + headless service "postgres-headless":
  
  Pod DNS names:
    postgres-0.postgres-headless.production.svc.cluster.local
    postgres-1.postgres-headless.production.svc.cluster.local
    postgres-2.postgres-headless.production.svc.cluster.local
  
  Why this matters:
  → Postgres replica knows to replicate FROM: postgres-0.postgres-headless...
  → Even if postgres-0's IP changes (pod rescheduled) → DNS still resolves correctly
  → The NAME is stable, the IP can change
  
  In practice (PostgreSQL Patroni HA):
    Primary:  always postgres-0
    Replicas: postgres-1, postgres-2 connect to postgres-0 for replication
    Failover: if postgres-0 fails, Patroni elects new leader
              DNS updated to point to new leader
              All replicas reconnect using the stable hostname

  In practice (Kafka):
    Brokers: kafka-0, kafka-1, kafka-2
    Zookeeper/KRaft: each broker knows its own stable identity
    Producers/consumers: connect to kafka-0.kafka-headless:9092 etc.
```

> 💡 **Interview tip:** The key insight for headless services: it shifts **load balancing from the network layer (kube-proxy) to the application layer (client)**. For StatefulSets like databases, this is actually what you WANT — the application knows better than the network which pod is the primary and which are replicas. Regular ClusterIP service would randomly route to any pod (bad for a database where writes must go to primary only). Headless service lets the app discover all pods via DNS and make an intelligent routing decision. Also mention: headless services are how **service meshes like Istio** discover pods — Istio uses headless service DNS to find all pod endpoints for its own load balancing.

---

### Q356 — Grafana | Scenario-Based | Advanced

> Set up Grafana alerting with multiple data sources: Loki error rate alert, multi-datasource Prometheus+Loki AND alert, alert grouping, and mute timings for maintenance windows.

#### Key Points to Cover:
```
Grafana Unified Alerting (Grafana 8+):
  → Alerts from ANY data source: Prometheus, Loki, CloudWatch, etc.
  → Alert rules stored in Grafana (not Prometheus AlertManager)
  → Can use AlertManager as notification backend

Alert 1 — Loki-based: error log rate > 10/minute:
  Rule name: HighErrorLogRate
  Data source: Loki
  Query:
    sum(rate({namespace="production", app="payment-service"} |= "ERROR" [1m])) by (app)
  
  Condition:
    WHEN last() OF A IS ABOVE 10
    FOR: 2m  (must be above 10 for 2 minutes)
  
  Labels:
    severity: warning
    team: payment
  
  Annotations:
    summary: "High error log rate for {{ $labels.app }}"
    description: "{{ $value | printf \"%.0f\" }} errors/min in last 2 minutes"

Alert 2 — Multi-datasource (Prometheus CPU AND Loki errors):
  # Grafana supports multi-query alerts where multiple queries combine
  
  Rule with multiple queries:
  Query A (Prometheus):
    sum(rate(container_cpu_usage_seconds_total{pod=~"payment.*"}[5m]))
    / sum(container_spec_cpu_quota{pod=~"payment.*"} / 100000)
    → Returns: CPU utilization ratio (0.0 to 1.0)
  
  Query B (Loki):
    sum(rate({app="payment-service"} |= "ERROR" [5m]))
    → Returns: error logs per second
  
  # Grafana Math expression to combine:
  Query C (Math/Reduce):
    $A > 0.8 && $B > 1
    # True ONLY when CPU > 80% AND error rate > 1/s simultaneously
    # This is the AND logic
  
  Condition: WHEN C IS ABOVE 0 (true = alert)
  
  For AND logic using Classic Condition:
    Condition 1: A (Prometheus CPU) IS ABOVE 0.8 FOR 5m
    Condition 2: B (Loki error rate) IS ABOVE 1 FOR 5m
    WHEN ALL conditions are met → alert fires

Alert Grouping:
  # In Grafana Alerting → Notification Policies:
  # Group alerts by labels
  
  Root policy:
    group_by: [alertname, namespace, team]
    group_wait: 30s
    group_interval: 5m
    repeat_interval: 4h
    receiver: default-slack
  
  # Child policy for payment team:
  - matchers:
    - team = payment
    group_by: [alertname, app]
    group_wait: 10s
    receiver: payment-pagerduty
  
  # Child policy for non-critical:
  - matchers:
    - severity = info
    group_by: [team]
    repeat_interval: 24h
    receiver: team-slack

Mute Timings for maintenance windows:
  # Grafana → Alerting → Mute timings
  
  mute_timing:
    name: weekly-maintenance
    time_intervals:
    - weekdays: [saturday]
      times:
      - start_time: "02:00"
        end_time: "06:00"
    - weekdays: [sunday]
      times:
      - start_time: "02:00"
        end_time: "06:00"
  
  # Apply to notification policy:
  - matchers:
    - severity != critical   # mute warnings, not criticals
    mute_timings: [weekly-maintenance]
    receiver: default-slack
  
  # Via Grafana API (for automation):
  curl -X POST http://grafana:3000/api/v1/provisioning/mute-timings \
    -H "Content-Type: application/json" \
    -d '{
      "name": "scheduled-maintenance",
      "time_intervals": [{
        "weekdays": ["saturday", "sunday"],
        "times": [{"start_time": "02:00", "end_time": "06:00"}]
      }]
    }'
  
  # One-time silence (for ad-hoc maintenance):
  # Grafana → Alerting → Silences → New Silence
  # Start: now, End: 2 hours
  # Matchers: namespace="production"
  # Comment: "Scheduled maintenance - ticket #1234"
```

> 💡 **Interview tip:** The key distinction: **Mute Timings are recurring schedules** (weekly maintenance window), **Silences are one-time suppressions** (this specific deployment, this incident). Use mute timings for predictable recurring windows (weekend maintenance, end-of-month batch jobs), use silences for ad-hoc situations. The multi-datasource alert with AND logic is where Grafana shines over pure Prometheus alerting — Prometheus can't natively correlate Loki log data with metrics. Grafana's Math expressions let you combine signals from any source. The practical value: "CPU is high" is a noisy alert. "CPU is high AND error logs are spiking" is a meaningful incident — the AND logic dramatically reduces false positives.

---

### Q357 — Terraform | Conceptual | Advanced

> Explain the Terraform dependency graph, implicit vs explicit dependencies, and when depends_on causes unnecessary rebuilds vs when it's absolutely necessary.

#### Key Points to Cover:
```
terraform graph command:
  terraform graph | dot -Tpng > graph.png   # requires graphviz
  # Generates: directed acyclic graph (DAG) of all resources
  # Nodes: resources, data sources, variables, outputs
  # Edges: dependency arrows (A must exist before B)

How Terraform determines creation order:
  1. Parse all .tf files → build resource graph
  2. Find resources with no dependencies → create in parallel
  3. As resources complete → unblock dependent resources
  4. Deletion: reverse order (destroy dependents first)

Implicit Dependencies (via references):
  → Created automatically when one resource references another
  → Most common: referencing .id, .arn, .endpoint etc.

  resource "aws_vpc" "main" { cidr_block = "10.0.0.0/16" }
  
  resource "aws_subnet" "private" {
    vpc_id = aws_vpc.main.id  # ← IMPLICIT dependency on aws_vpc.main
    cidr_block = "10.0.1.0/24"
  }
  
  # Terraform sees: aws_subnet.private needs aws_vpc.main.id
  # → Creates aws_vpc.main FIRST, then aws_subnet.private
  # → This is the correct and PREFERRED way to express dependencies

Explicit Dependencies (depends_on):
  → Used when dependency exists but isn't captured by attribute references
  → "This resource depends on that one, but doesn't reference it directly"

  # Necessary example: IAM role policy attachment + resource that needs the permission
  resource "aws_iam_role_policy_attachment" "lambda_s3" {
    role       = aws_iam_role.lambda.name
    policy_arn = aws_iam_policy.s3_read.arn
  }
  
  resource "aws_lambda_function" "processor" {
    function_name = "processor"
    role          = aws_iam_role.lambda.arn
    # Lambda references the ROLE (implicit dep on role)
    # But NOT the policy attachment
    # Without depends_on: Lambda created before policy attached
    # Lambda exists but can't access S3 → runtime error
    
    depends_on = [aws_iam_role_policy_attachment.lambda_s3]
    # Explicit: wait for policy to be attached before creating Lambda
  }

When depends_on causes unnecessary rebuilds:
  # PROBLEM: depends_on on a module creates a whole-module dependency
  # If ANY resource in the depended-on module changes
  # → ALL resources in the dependent module get flagged for possible change

  module "network" { source = "./modules/network" }
  
  module "application" {
    source     = "./modules/application"
    depends_on = [module.network]  # ← PROBLEMATIC if overused
    # If network module changes ANY resource (even unrelated)
    # → application module re-evaluates ALL resources
    # → may trigger unnecessary updates
  }
  
  # BETTER: reference specific outputs instead:
  module "application" {
    source     = "./modules/application"
    vpc_id     = module.network.vpc_id        # implicit dep only on what's needed
    subnet_ids = module.network.subnet_ids
  }

When depends_on is ABSOLUTELY necessary:
  1. IAM propagation (most common):
     → IAM policy attached → AWS takes ~10s to propagate globally
     → Resource using that policy fails if created immediately
     → depends_on ensures sequential creation

  2. Null resource / local-exec ordering:
     resource "terraform_data" "db_init" {
       depends_on = [aws_db_instance.main]
       # aws_db_instance.main doesn't appear in any attribute reference
       # but db_init must run AFTER db is created
     }

  3. Bootstrap ordering with no attribute reference:
     resource "aws_s3_bucket_notification" "config" {
       bucket = aws_s3_bucket.config.id
       # References bucket.id (implicit dep on bucket)
       # But needs SQS queue to exist too:
       queue { queue_arn = aws_sqs_queue.notify.arn }
       # Wait — queue IS referenced via .arn, so implicit dep exists
       # depends_on NOT needed here
     }

Performance impact of depends_on:
  → Prevents parallelism: A depends_on B → B must complete before A starts
  → Without depends_on: A and B can create in parallel (if no attribute dep)
  → Over-using depends_on: forces sequential execution, slows apply time
  → Module-level depends_on: worst case — serializes entire modules

terraform graph debugging:
  terraform graph -type=plan | grep "resource_name"
  # Find: what depends on what in your specific plan
  # Useful when: apply is unexpectedly slow (too many sequential operations)
```

> 💡 **Interview tip:** The IAM propagation case is the most common production scenario requiring `depends_on`. You create an IAM role, attach a policy, create a resource that uses the role — without `depends_on`, Terraform creates the resource and the policy attachment in parallel (they don't reference each other), and sometimes the resource gets created before the policy attachment completes and propagates globally, causing runtime failures. The `depends_on` ensures sequential creation. But — the **overuse anti-pattern**: putting `depends_on = [module.entire_network_module]` when you only need the VPC ID means ANY change to the network module triggers full re-evaluation of the application module, even if the VPC didn't change. Use specific attribute references instead.

---

### Q358 — Git | Scenario-Based | Advanced

> Branches diverged for 3 months — 150 commits on feature, 200 on main. Walk through merge strategy: merge vs rebase vs squash, handling conflicts in 30 files, visualization tools.

#### Key Points to Cover:
```
Assessment first — understand the scale:
  git log --oneline main..feature/payment-v2 | wc -l  # 150 feature commits
  git log --oneline feature/payment-v2..main | wc -l  # 200 main commits
  git diff --stat main...feature/payment-v2            # which files changed?
  git diff --name-only main feature/payment-v2         # file list

Strategy decision — merge vs rebase vs squash:

  Merge (recommended for this scenario):
    git checkout main
    git merge feature/payment-v2 --no-ff
    → Preserves: entire feature branch history
    → Creates: one merge commit
    → History: non-linear but honest (shows when branches diverged)
    → Risk: conflicts appear all at once (from 3 months of divergence)
    → Use when: feature work has meaningful commit history worth preserving

  Rebase (NOT recommended for 3-month divergence):
    git checkout feature/payment-v2
    git rebase main
    → Replays: each of 150 commits on top of 200 new main commits
    → Each commit potentially has conflicts → resolve 150 times
    → Linear history (clean) but misleading (looks like no branch ever happened)
    → Risk: extremely painful, many conflict rounds
    → Use when: small number of commits, fresh divergence

  Squash Merge (recommended for this scenario + large teams):
    git checkout main
    git merge --squash feature/payment-v2
    git commit -m "feat: payment-v2 complete implementation"
    → Compresses: 150 commits into 1
    → Clean history on main
    → Feature branch history lost (only exists if you keep the branch)
    → Use when: feature commits are "work in progress" noise,
      final commit message is more useful than individual steps

Handling 30-file conflicts:

  Step 1 — Use a merge tool:
    git config merge.tool vimdiff   # or code, IntelliJ, kdiff3
    git mergetool                    # opens each conflicting file in tool

  Step 2 — Prioritize conflicts by risk:
    git diff --stat main...feature/payment-v2 | sort -k3 -rn | head -10
    # Fix high-churn files first (most likely to have real conflicts)
    # Pure formatting conflicts: accept one side entirely
    # Logic conflicts: require careful review

  Step 3 — Accept one side wholesale where safe:
    # For files that only changed in feature (main didn't touch):
    git checkout feature/payment-v2 -- path/to/only-feature-changed.py
    
    # For files that only changed in main (feature didn't touch):
    git checkout main -- path/to/only-main-changed.py

  Step 4 — Use git rerere (record and reuse resolutions):
    git config rerere.enabled true
    # If you've resolved this conflict before, git rerere reapplies it automatically
    # Extremely useful for long-running feature branches that need periodic rebasing

  Step 5 — Divide and conquer for large conflicts:
    # If the branch is too large, split the merge:
    git checkout -b intermediate-merge
    git merge --strategy-option=ours main  # keep our version first
    # Then selectively apply main's changes section by section

Visualization tools:
  CLI:
    git log --oneline --graph --all --decorate
    # Shows: branch topology as ASCII art
    
    git log --left-right --cherry-pick main...feature/payment-v2
    # Shows: commits unique to each side (< = main, > = feature)

  GUI tools:
    gitk --all                          # built-in Git GUI
    git log --graph | tig               # TUI (terminal UI)
    GitHub/GitLab merge request diff    # PR interface shows all conflicts
    GitKraken, Sourcetree               # graphical diff/merge tools
    IntelliJ/VSCode                     # built-in 3-way merge view

Post-merge verification:
  # Run full test suite immediately after merge:
  make test
  
  # Check nothing unexpected changed:
  git diff main~1 main --stat     # what the merge commit actually changed
  
  # Verify commit count:
  git log --oneline ORIG_HEAD..HEAD  # commits added by merge
```

> 💡 **Interview tip:** For 3 months of divergence, the honest answer is: **merge, not rebase**. Rebasing 150 commits onto 200 new commits is asking for suffering — each commit replay is a potential conflict resolution. A merge resolves everything once. The decision framework: "Is the feature branch history meaningful to preserve for future debugging?" If yes, merge with `--no-ff`. If the commits are messy WIP snapshots, squash merge gives you a clean single entry in main's history. Also mention: **this situation (3-month divergence) is itself a process failure** — the right practice is to merge main into the feature branch weekly, resolving small conflicts incrementally. Bring this up in the postmortem, not during the emergency merge.

---

### Q359 — AWS + Kubernetes | Scenario-Based | Advanced

> Design S3 video transcoding architecture with KEDA for SQS-based scaling, Spot instances for cost, checkpoint/resume for interrupted jobs, and graceful SIGTERM handling.

#### Key Points to Cover:
```
Complete Architecture:

  S3 (uploads) → Lambda (trigger) → SQS Queue
  KEDA → reads SQS depth → scales Worker Deployment
  Worker Deployment → Spot EC2 node group via Karpenter
  Worker → downloads from S3 → transcodes → uploads to S3 output bucket
  Worker → checkpoints progress to DynamoDB every 30 seconds

Step 1 — SQS trigger on S3 upload (Lambda → SQS):
  # S3 event notification → Lambda:
  def lambda_handler(event, context):
      for record in event['Records']:
          s3_key = record['s3']['object']['key']
          sqs.send_message(
              QueueUrl=TRANSCODE_QUEUE_URL,
              MessageBody=json.dumps({
                  "video_key": s3_key,
                  "bucket": record['s3']['bucket']['name'],
                  "timestamp": datetime.utcnow().isoformat()
              })
          )

Step 2 — KEDA ScaledObject for SQS:
  apiVersion: keda.sh/v1alpha1
  kind: ScaledObject
  metadata:
    name: video-transcoder-scaler
  spec:
    scaleTargetRef:
      name: video-transcoder
    minReplicaCount: 0       # scale to zero when idle — cost saving
    maxReplicaCount: 100     # max concurrent transcoding jobs
    pollingInterval: 10
    cooldownPeriod: 300      # 5 min cooldown (jobs take 10-30 min)
    triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/ACCOUNT/video-transcode
        queueLength: "1"     # 1 message = 1 pod (1:1 ratio)
        awsRegion: us-east-1
        identityOwner: operator

Step 3 — Spot instances with Karpenter:
  apiVersion: karpenter.sh/v1alpha5
  kind: Provisioner
  metadata:
    name: spot-transcoder
  spec:
    labels:
      workload: video-transcoder
    taints:
    - key: workload
      value: video-transcoder
      effect: NoSchedule
    requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["c5.2xlarge", "c5a.2xlarge", "m5.2xlarge"]  # multiple types = better spot availability
    limits:
      resources:
        cpu: "200"
        memory: 400Gi
    ttlSecondsAfterEmpty: 60  # terminate node 60s after all pods leave

  # Worker pod tolerates the taint:
  spec:
    tolerations:
    - key: workload
      value: video-transcoder
      effect: NoSchedule
    nodeSelector:
      workload: video-transcoder

Step 4 — Checkpoint/Resume with DynamoDB:
  # DynamoDB table: VideoJobs
  # PK: video_key
  # Attributes: status, checkpoint_time, chunk_number, progress_pct

  import boto3, signal, sys, time

  dynamodb = boto3.resource('dynamodb')
  table = dynamodb.Table('VideoJobs')
  s3 = boto3.client('s3')
  shutdown_requested = False

  def save_checkpoint(video_key, chunk_number, total_chunks):
      table.put_item(Item={
          'video_key': video_key,
          'status': 'IN_PROGRESS',
          'checkpoint_chunk': chunk_number,
          'total_chunks': total_chunks,
          'checkpoint_time': int(time.time()),
          'worker_pod': os.environ.get('POD_NAME', 'unknown')
      })

  def get_checkpoint(video_key):
      result = table.get_item(Key={'video_key': video_key})
      item = result.get('Item')
      if item and item['status'] == 'IN_PROGRESS':
          return item.get('checkpoint_chunk', 0)
      return 0  # start from beginning

Step 5 — Graceful SIGTERM handling:
  def handle_sigterm(signum, frame):
      global shutdown_requested
      print("SIGTERM received — finishing current chunk then saving checkpoint")
      shutdown_requested = True

  signal.signal(signal.SIGTERM, handle_sigterm)

  def transcode_video(video_key):
      # Get existing checkpoint (resume from where we left off)
      start_chunk = get_checkpoint(video_key)
      print(f"Starting/resuming from chunk {start_chunk}")
      
      chunks = split_video_into_chunks(video_key)
      
      for i, chunk in enumerate(chunks[start_chunk:], start=start_chunk):
          if shutdown_requested:
              save_checkpoint(video_key, i, len(chunks))
              print(f"Saved checkpoint at chunk {i} — pod terminating gracefully")
              sys.exit(0)    # exits cleanly, SQS message NOT deleted → re-queued
          
          process_chunk(chunk)
          
          # Checkpoint every 10 chunks:
          if i % 10 == 0:
              save_checkpoint(video_key, i, len(chunks))
      
      # Job complete:
      table.put_item(Item={'video_key': video_key, 'status': 'COMPLETE'})
      # Now delete from SQS:
      sqs.delete_message(QueueUrl=QUEUE_URL, ReceiptHandle=receipt_handle)
  
  # Pod spec: terminationGracePeriodSeconds: 120
  # Gives worker 2 minutes to finish current chunk + save checkpoint
```

> 💡 **Interview tip:** The **checkpoint frequency trade-off** is worth mentioning: checkpointing every second minimizes data loss on interruption but adds DynamoDB write overhead. Checkpointing every 10 chunks reduces writes but means re-processing up to 10 chunks on restart. For 10-30 minute transcoding jobs, checkpointing every 30-60 seconds of wall clock time (or every N chunks) is the right balance. Also mention: **don't delete the SQS message until the job is fully complete** — SQS visibility timeout must be set longer than the max expected job duration (e.g., 35 minutes) plus checkpointing overhead. If the pod is killed and the message becomes visible again, the new worker picks it up, reads the checkpoint, and resumes from where the last worker left off.

---

### Q360 — All Topics | System Design | Advanced

> Design a complete Internal Developer Platform (IDP) using Backstage for 500 engineers across 20 teams: service catalog, self-service infra, golden paths, developer portal, DORA metrics. How does each tool integrate?

#### Key Points to Cover:
```
IDP Architecture Overview:

  Developer Experience Layer:
    Backstage (developer portal) → unified UI for everything

  Source of Truth:
    GitHub (code, policies, approval workflows)
    Terraform modules (approved infrastructure patterns)

  CI/CD Layer:
    GitHub Actions (build, test, push image)
    ArgoCD (GitOps deploy to Kubernetes)

  Infrastructure Layer:
    Terraform Cloud / Atlantis (infra provisioning)
    EKS (container runtime)
    AWS (cloud provider)

  Observability Layer:
    Prometheus + Grafana (metrics)
    Loki (logs)
    Jaeger/Tempo (traces)
    PagerDuty (alerting)

Component 1 — Service Catalog (Backstage):
  → All 200 microservices registered in Backstage catalog
  → Each service: backstage.yaml in repo root
  
  catalog-info.yaml in each repo:
  apiVersion: backstage.io/v1alpha1
  kind: Component
  metadata:
    name: payment-service
    annotations:
      github.com/project-slug: myorg/payment-service
      grafana/dashboard-selector: "app=payment-service"
      pagerduty.com/integration-key: xxxxx
      argocd/app-name: payment-service-prod
    tags: [payment, critical, tier-1]
  spec:
    type: service
    lifecycle: production
    owner: group:payment-team
    system: payments-platform
    dependsOn:
    - component:auth-service
    - resource:postgres-prod
    providesApis:
    - payments-api
  
  # Backstage reads all catalog-info.yaml from GitHub automatically
  # Developers search: "show me all services owned by payment-team"
  # Ops: "show me all services that depend on auth-service"

Component 2 — Self-Service Infrastructure (Backstage Software Templates):
  # Backstage Templates = self-service forms that provision infra
  # Developer fills out form → Template creates GitHub PR → Atlantis applies Terraform

  apiVersion: scaffolder.backstage.io/v1beta3
  kind: Template
  metadata:
    name: new-rds-database
    title: "Provision RDS Database"
  spec:
    parameters:
    - title: "Database Configuration"
      properties:
        dbName:
          type: string
        instanceClass:
          type: string
          enum: ["db.t3.micro", "db.t3.medium", "db.r5.large"]
          # Restricted choices = guardrails
        teamName:
          type: string
    steps:
    - id: create-pr
      name: "Create Terraform PR"
      action: publish:github:pull-request
      input:
        repoUrl: github.com?repo=infra-terraform&owner=myorg
        title: "feat: provision RDS for ${{ parameters.teamName }}"
        sourcePath: ./rds-module/
        targetPath: ./environments/prod/databases/${{ parameters.dbName }}/

  # Atlantis picks up PR → runs terraform plan → posts plan to PR
  # Platform team reviews → approves PR → Atlantis applies

Component 3 — Golden Paths (CI/CD templates):
  # Golden paths = opinionated, pre-approved ways to do things
  # Reduces cognitive load: "don't think about CI/CD, just use this"
  
  GitHub Actions reusable workflows:
  # .github/workflows/golden-path-deploy.yml (in shared repo)
  on:
    workflow_call:
      inputs:
        service-name: {type: string, required: true}
        environment: {type: string, default: "production"}
  
  jobs:
    test: (standard test step)
    security-scan: (standard SAST/DAST)
    build-push: (standard ECR push)
    deploy: (standard ArgoCD sync)
    notify: (standard Slack notification)
  
  # Each service's workflow is just:
  uses: myorg/.github/.github/workflows/golden-path-deploy.yml@main
  with:
    service-name: payment-service
  
  # Platform team updates golden path once → all services benefit automatically

Component 4 — Developer Portal (unified view in Backstage):
  Backstage plugins for unified view:
  
  → GitHub plugin: recent commits, open PRs, CI status
  → ArgoCD plugin: deployment status, sync status, last deployment
  → Grafana plugin: service health dashboard embedded in Backstage
  → PagerDuty plugin: on-call schedule, recent incidents
  → Kubernetes plugin: pod count, restarts, resource usage
  → Lighthouse plugin: performance score for web services
  
  # All on one page: payment-service catalog page shows:
  # - Last 5 deployments (ArgoCD)
  # - Current error rate (Grafana)
  # - On-call engineer (PagerDuty)
  # - Recent PRs (GitHub)
  # - Live pod status (Kubernetes)

Component 5 — DORA Metrics:
  Four DORA metrics and how to collect each:

  Deployment Frequency:
    → Count: git tags to production per week
    → Source: ArgoCD sync events + Prometheus counter
    → metric: argocd_app_sync_total{dest_namespace="production"}
    → Dashboard: Grafana graph of deployments/week per service

  Lead Time for Changes:
    → Time from: first commit in PR → production deployment
    → Source: GitHub PR creation time + ArgoCD sync completion time
    → Collect: GitHub webhook → Lambda → store in TimescaleDB
    → Dashboard: histogram of lead times per team

  Mean Time to Recovery (MTTR):
    → Time from: incident created → service restored
    → Source: PagerDuty API → incident open time to resolve time
    → Backstage plugin: PagerDuty incident history per service
    → Calculate: avg(resolve_time - trigger_time) per service

  Change Failure Rate:
    → Percentage of deployments causing incidents/rollbacks
    → Source: ArgoCD rollback events + PagerDuty incidents
    → Correlate: deployment time vs incident open time (within 1 hour window)
    → metric: rollbacks_triggered / total_deployments

  DORA dashboard (Grafana):
    → Top-level: org-wide DORA scores (color coded: Elite/High/Medium/Low)
    → Drill-down: per team, per service
    → Trend: are metrics improving or degrading?
    → Target: Deployment Frequency > 1/day (Elite), Lead Time < 1 day (Elite)

Integration Flow Summary:
  New service onboarding (self-service, 1 day instead of 2 weeks):
  1. Developer: Backstage "Create Service" template
  2. Template: creates GitHub repo with golden path CI/CD, dockerfile, backstage.yaml
  3. Developer: writes code, opens PR
  4. GitHub Actions golden path: test + security scan automatically
  5. ArgoCD: automatically creates Application for new service
  6. Backstage: automatically discovers new service via catalog-info.yaml
  7. Developer: requests RDS via Backstage template → Terraform PR → approved
  8. DORA metrics: start tracking automatically from first deployment
```

> 💡 **Interview tip:** The IDP question tests **platform engineering thinking**, not just tool knowledge. The key principle: an IDP is not about the tools — it's about **reducing cognitive load for developers while maintaining governance**. The worst IDP is one that forces developers to know 10 different tools to deploy one service. The best IDP has **one entry point (Backstage)** that abstracts everything else. Emphasize: **golden paths aren't mandates — they're the path of least resistance**. Teams can deviate, but the golden path is so much easier that most don't. Also connect to business value: DORA metrics are the feedback loop that proves the IDP is working. If deployment frequency increases and MTTR decreases after IDP adoption, you have data to show leadership the platform is delivering ROI.

---

## Coverage Summary — Q316–Q360 Enhanced

| Topic | Questions |
|---|---|
| Kubernetes | Q316, Q321, Q326, Q332, Q336, Q341, Q346, Q351, Q355 |
| AWS | Q317, Q323, Q330, Q335, Q340, Q345, Q350, Q354, Q359 |
| Terraform | Q319, Q328, Q338, Q348, Q357 |
| Prometheus | Q320, Q331, Q349 |
| Grafana | Q337, Q356 |
| ELK Stack | Q325, Q342 |
| Linux / Bash | Q318, Q327, Q334, Q344, Q353 |
| ArgoCD | Q333, Q352 |
| Jenkins | Q324, Q343 |
| GitHub Actions | Q329, Q347 |
| Git | Q322, Q358 |
| Python | Q339 |
| System Design | Q360 |

---

*Part 8 Enhanced — DevOps/SRE Interview Questions Q316–Q360*
*Format matches Q1–Q45 Enhanced: Key Points + Interview Tips for every question*
