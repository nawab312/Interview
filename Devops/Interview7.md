# DevOps / SRE Interview Questions — Q271–Q315 Enhanced
## Key Points + Interview Tips Added to Every Question

---

### Q271 — Kubernetes | Conceptual | Advanced

> Explain Kubernetes subresources — which built-in subresources exist, why `/status` is separated, and write a Role that allows reading pod logs but not exec.

#### Key Points to Cover:
```
What are subresources:
  → Sub-paths of a Kubernetes resource that expose specific operations
  → Format: resource/subresource (e.g., pods/exec, pods/log, pods/status)
  → Independently controllable via RBAC
  → Some are read-only, some trigger server-side actions

Built-in subresources:

  pods/log:
    → Read container logs (kubectl logs)
    → HTTP GET to /api/v1/namespaces/{ns}/pods/{name}/log

  pods/exec:
    → Execute commands inside containers (kubectl exec)
    → Establishes a WebSocket connection
    → Dangerous: full shell access to container

  pods/portforward:
    → Forward local port to container port (kubectl port-forward)

  pods/attach:
    → Attach to running container's stdin/stdout

  pods/status:
    → Read/write just the status subresource
    → Only controllers should write status

  deployments/scale:
    → Read/write .spec.replicas without full deployment access
    → HPA uses this to scale deployments

  deployments/status:
    → controllers write observed state here

  nodes/proxy:
    → Proxy HTTP requests to kubelet API on the node

Why /status is separated from main resource:
  → Controllers manage status (what IS) separately from spec (what SHOULD BE)
  → A developer can update spec (desired state) but not fake status
  → Prevents: application lying about its own health
  → Example: your Deployment controller sets status.readyReplicas
    but a developer editing the deployment YAML can't forge that value
  → RBAC separation: CI/CD system gets deployments update
    but NOT deployments/status update
  → Pattern: spec = intent (human-managed), status = observed (controller-managed)

Role — read logs but NOT exec:
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: pod-log-reader
    namespace: production
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]    # see pod names/status
  - apiGroups: [""]
    resources: ["pods/log"]            # read logs
    verbs: ["get"]
  # NOTE: pods/exec is NOT listed → exec is blocked
  # NOTE: pods/portforward is NOT listed → port-forward is blocked

  # RoleBinding:
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: pod-log-reader-binding
    namespace: production
  subjects:
  - kind: User
    name: alice@company.com
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: Role
    name: pod-log-reader
    apiGroup: rbac.authorization.k8s.io

  # Verify:
  kubectl auth can-i get pods/log -n production --as alice@company.com
  # yes
  kubectl auth can-i create pods/exec -n production --as alice@company.com
  # no

Common subresource RBAC patterns:
  SRE (read-only + logs):
    pods: get, list, watch
    pods/log: get
    pods/exec: NONE

  Developer (full access):
    pods: get, list, watch
    pods/log: get
    pods/exec: create   ← exec requires "create" verb
    pods/portforward: create

  HPA controller:
    deployments/scale: get, update
    (doesn't need full deployment update)
```

> 💡 **Interview tip:** The exec verb is the one most candidates get wrong — `pods/exec` requires the `create` verb, not `get`. This trips up even experienced engineers. The reason: exec creates a new WebSocket session. Also nail the **spec/status separation rationale**: it's the Kubernetes implementation of the separation between desired state and observed state. Controllers write status; humans write spec. Without this separation, a rogue process could mark itself as healthy while actually broken.

---

### Q272 — AWS | Scenario-Based | Advanced

> Design immutable infrastructure on AWS with Packer AMI baking, testing, blue/green ASG swap, graceful drain, and rollback strategy.

#### Key Points to Cover:
```
Immutable Infrastructure principle:
  → Never modify running servers — replace them
  → Every deployment = new AMI → new ASG → traffic swap → old ASG teardown
  → Benefits: reproducible, no config drift, fast rollback (just route to old ASG)

Step 1 — AMI baking with Packer:
  # packer.pkr.hcl:
  source "amazon-ebs" "app" {
    ami_name      = "myapp-${formatdate("YYYY-MM-DD-hhmm", timestamp())}-${var.git_sha}"
    instance_type = "t3.medium"
    source_ami    = data.amazon-ami.amazon_linux_2.id
    ssh_username  = "ec2-user"
  }

  build {
    sources = ["source.amazon-ebs.app"]

    provisioner "shell" {
      scripts = [
        "./scripts/install-dependencies.sh",
        "./scripts/configure-app.sh",
        "./scripts/harden-os.sh"
      ]
    }

    # Bake app binary directly into AMI:
    provisioner "file" {
      source      = "./dist/app"
      destination = "/opt/app/app"
    }

    post-processor "manifest" {
      output = "manifest.json"   # captures AMI ID for pipeline use
    }
  }

Step 2 — AMI Testing before promotion:
  # Launch test instance from new AMI:
  TEST_INSTANCE=$(aws ec2 run-instances \
    --image-id $NEW_AMI_ID \
    --instance-type t3.medium \
    --security-group-ids $TEST_SG \
    --query 'Instances[0].InstanceId' --output text)

  # Wait for running state:
  aws ec2 wait instance-running --instance-ids $TEST_INSTANCE

  # Run smoke tests:
  TEST_IP=$(aws ec2 describe-instances --instance-ids $TEST_INSTANCE \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
  
  curl -f http://$TEST_IP:8080/health || {
    echo "AMI health check failed — not promoting"
    aws ec2 terminate-instances --instance-ids $TEST_INSTANCE
    exit 1
  }

  # Tag as tested:
  aws ec2 create-tags --resources $NEW_AMI_ID \
    --tags Key=Status,Value=tested Key=TestDate,Value=$(date +%Y-%m-%d)
  
  # Terminate test instance:
  aws ec2 terminate-instances --instance-ids $TEST_INSTANCE

Step 3 — Blue/Green ASG swap with Launch Template:
  # Create new Launch Template version with new AMI:
  NEW_LT_VERSION=$(aws ec2 create-launch-template-version \
    --launch-template-id $LT_ID \
    --source-version '$Latest' \
    --launch-template-data "{\"ImageId\":\"$NEW_AMI_ID\"}" \
    --query 'LaunchTemplateVersion.VersionNumber' --output text)

  # Create GREEN ASG (new):
  aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name "myapp-green-$(date +%Y%m%d%H%M)" \
    --launch-template "LaunchTemplateId=$LT_ID,Version=$NEW_LT_VERSION" \
    --min-size 3 --max-size 10 --desired-capacity 3 \
    --target-group-arns $TARGET_GROUP_ARN \
    --health-check-type ELB \
    --health-check-grace-period 120
  
  # Wait for green instances healthy in target group:
  aws elbv2 wait target-in-service \
    --target-group-arn $TARGET_GROUP_ARN

  # Blue ASG is the current production one (BLUE_ASG_NAME)

Step 4 — Graceful drain of Blue ASG:
  # Detach Blue instances from target group (ALB stops routing):
  aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name $BLUE_ASG_NAME \
    --min-size 0 --max-size 0 --desired-capacity 0

  # ALB waits deregistration delay (default 300s) before closing connections
  # Adjust for faster draining:
  aws elbv2 modify-target-group-attributes \
    --target-group-arn $TARGET_GROUP_ARN \
    --attributes Key=deregistration_delay.timeout_seconds,Value=30

  # Wait for connections to drain:
  sleep 60

  # Delete old Blue ASG:
  aws autoscaling delete-auto-scaling-group \
    --auto-scaling-group-name $BLUE_ASG_NAME \
    --force-delete

Step 5 — Rollback strategy:
  # If green is bad: route traffic back to blue (if blue ASG still exists)
  # NEVER delete blue ASG until green is fully validated

  # Emergency rollback: scale blue back up, deregister green:
  aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name $BLUE_ASG_NAME \
    --min-size 3 --max-size 10 --desired-capacity 3
  
  aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name $GREEN_ASG_NAME \
    --min-size 0 --max-size 0 --desired-capacity 0

  # Keep last 2 AMIs tagged (never deregister until 2 deployments later)
  # → Allows rollback to N-1 or N-2

Key validation window:
  Green deployed → monitor 10 minutes → errors acceptable → delete Blue
                 → errors spike → rollback to Blue
```

> 💡 **Interview tip:** The critical operational detail: **never delete the Blue ASG until Green has been validated**. Keep it at desired=0 (no instances, no cost) but ready to scale back up instantly. Many teams make the mistake of deleting Blue immediately, then scrambling to rebuild it during a rollback. Also mention the **deregistration delay** — ALB's default 300-second wait for connections to drain means a "fast" rollback still takes 5 minutes. For latency-sensitive services, lower this to 30 seconds during deploys.

---

### Q273 — Linux / Bash | Conceptual | Advanced

> Explain Linux network namespaces, `ip netns`, and the veth pair plumbing for Docker container networking with exact commands.

#### Key Points to Cover:
```
What is a network namespace:
  → An isolated copy of the Linux network stack
  → Each namespace has: its own network interfaces, routing table,
    iptables rules, sockets, netfilter hooks
  → Process inside namespace only sees its own interfaces
  → Cannot see host interfaces or other namespace interfaces

  Default namespace: all processes start here
  New namespace: container runtime creates one per container

ip netns commands:
  # Create a new named network namespace:
  ip netns add myns

  # List all named namespaces:
  ip netns list

  # Run command inside namespace:
  ip netns exec myns ip addr show
  ip netns exec myns ping 8.8.8.8

  # Delete namespace:
  ip netns delete myns

  # NOTE: Docker/K8s namespaces are NOT in /var/run/netns (named)
  # They are anonymous — referenced by /proc/<PID>/ns/net
  # To access: nsenter --target <PID> --net ip addr

How Docker creates container networking (veth pair):

  Step 1 — Create network namespace for container:
  # Docker does this internally, equivalent to:
  ip netns add container-ns

  Step 2 — Create veth pair (virtual ethernet cable — 2 ends):
  ip link add veth0 type veth peer name veth1
  # veth0 and veth1 are connected — packet in one end comes out the other

  Step 3 — Move one end into the container namespace:
  ip link set veth1 netns container-ns
  # veth0 stays on host, veth1 is inside the container
  # Inside container: veth1 renamed to eth0

  Step 4 — Configure IPs:
  # Host side:
  ip addr add 172.17.0.1/16 dev veth0
  ip link set veth0 up
  
  # Container side:
  ip netns exec container-ns ip addr add 172.17.0.2/16 dev veth1
  ip netns exec container-ns ip link set veth1 up
  ip netns exec container-ns ip link set lo up
  ip netns exec container-ns ip route add default via 172.17.0.1

  Step 5 — Connect to docker0 bridge (multi-container):
  # Docker uses a bridge (docker0) instead of direct veth to host:
  ip link set veth0 master docker0
  # Now: container ↔ veth pair ↔ docker0 bridge ↔ NAT ↔ host NIC ↔ internet

  Full path for container → internet:
  Container eth0 → veth1 → veth0 → docker0 bridge → iptables MASQUERADE → eth0 (host) → internet

Verifying the plumbing:
  # On host: see all veth interfaces:
  ip link show type veth

  # Find which veth belongs to which container:
  # 1. Get container PID:
  docker inspect --format '{{.State.Pid}}' my-container
  # 2. See namespace links:
  nsenter --target <PID> --net ip link
  # Shows: eth0 with an index (e.g., index 5)
  # 3. On host: find veth with peer index 5:
  ip link show | grep "^5:"

Kubernetes veth plumbing:
  → Same pattern but at larger scale
  → Each pod gets its own network namespace
  → CNI plugin (Calico, Flannel, aws-vpc-cni) creates veth pairs
  → aws-vpc-cni: each pod gets a real ENI IP (no NAT needed)
  → Calico: veth pairs + BGP routing between nodes
```

> 💡 **Interview tip:** The veth pair is the fundamental building block of container networking — everything else (bridges, CNI plugins, overlay networks) builds on top of it. The key mental model: a veth pair is like a **patch cable** between the container's network namespace and the host. One end lives in the container (eth0), one end lives on the host or a bridge. If you can explain this plumbing and then draw how packets flow from container to internet (through bridge, NAT, host NIC), you've demonstrated deeper knowledge than most candidates who can only describe "Docker uses a virtual network."

---

### Q274 — Terraform | Conceptual | Advanced

> Explain Terraform backends in depth — local vs remote, 5 backend types, state locking, and backend partial configuration.

#### Key Points to Cover:
```
Backend fundamentals:
  → Defines: WHERE state is stored + HOW operations execute
  → Every config has a backend (default: local if not declared)
  → Two categories: standard (state only) and enhanced (state + remote ops)

Local backend:
  → State at ./terraform.tfstate
  → No locking — concurrent runs corrupt state
  → No sharing — each developer has their own state
  → Acceptable for: learning, solo projects

5 Backend Types:

  1. local:
     terraform { backend "local" {
       path = "relative/path/to/terraform.tfstate"
     }}
     Use: dev/learning only

  2. s3 (most common AWS):
     terraform {
       backend "s3" {
         bucket         = "company-tfstate"
         key            = "prod/services/api/terraform.tfstate"
         region         = "us-east-1"
         encrypt        = true
         dynamodb_table = "terraform-lock"   # locking via DynamoDB
         role_arn       = "arn:aws:iam::ACCOUNT:role/TerraformRole"
       }
     }
     Use: AWS teams, fine-grained IAM per bucket prefix

  3. gcs (Google Cloud):
     backend "gcs" { bucket = "my-tfstate"; prefix = "prod" }
     → Native locking (no separate lock resource needed)
     Use: GCP teams

  4. azurerm:
     backend "azurerm" {
       resource_group_name  = "tfstate-rg"
       storage_account_name = "tfstate"
       container_name       = "tfstate"
       key                  = "prod.terraform.tfstate"
     }
     Use: Azure teams

  5. terraform cloud / terraform enterprise (enhanced):
     terraform {
       cloud {
         organization = "my-org"
         workspaces { name = "prod-api" }
       }
     }
     → Stores state AND executes plan/apply remotely
     → Includes: Sentinel policies, cost estimation, SSO, RBAC
     Use: enterprises needing governance + approval workflows

State locking mechanics:
  S3 + DynamoDB:
    → Before plan: create item in DynamoDB table (lock ID + timestamp)
    → If item exists: another apply is running → wait or fail
    → After apply: delete DynamoDB item (release lock)
    → Stuck lock: terraform force-unlock <lock-id>
    → Dangerous: only force-unlock if the process is genuinely dead
      (not just slow) — otherwise two applies run simultaneously → corruption

Backend partial configuration:
  Problem:
    → backend block is evaluated at init time — before variables resolve
    → Cannot use var.*, local.*, or data.* inside backend block
    → Sensitive values (bucket names, account IDs) shouldn't be in git
    → Different environments need different buckets

  Solution — partial configuration:
    Check in: structural/non-sensitive config only
    Supply at init: sensitive/env-specific values via -backend-config

  In version control (main.tf):
    terraform {
      backend "s3" {
        encrypt = true   # only non-sensitive structural config
      }
    }

  Outside version control (backend-prod.hcl — in .gitignore):
    bucket         = "company-tfstate-prod"
    key            = "services/payment/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-lock-prod"
    role_arn       = "arn:aws:iam::PROD_ACCOUNT:role/TerraformRole"

  Init with partial config:
    terraform init -backend-config=backend-prod.hcl
    # OR: per-value flags:
    terraform init \
      -backend-config="bucket=company-tfstate-prod" \
      -backend-config="key=payment/terraform.tfstate"

  CI/CD pipeline pattern:
    → Secrets Manager stores backend config values
    → Pipeline: aws secretsmanager get-secret-value → write backend.hcl
    → Pipeline: terraform init -backend-config=backend.hcl
    → backend.hcl never committed, never persists after pipeline

State security:
  → State contains: resource IDs, IPs, sometimes PLAINTEXT secrets
  → Must: encrypt at rest (S3 SSE-KMS), restrict IAM, enable versioning
  → Never: commit to git, log to CloudWatch, share publicly
```

> 💡 **Interview tip:** The constraint that makes partial configuration necessary: **backend blocks are resolved at `terraform init` time, before any variables are loaded**. This is why `var.bucket_name` inside a backend block throws an error — variables aren't available yet. Know this, and the partial config pattern makes complete sense. Also, walk through the **DynamoDB lock failure scenario**: if a CI/CD runner crashes mid-apply, the lock stays. The next pipeline run hits a locked state and fails. You must `terraform force-unlock` with the lock ID — but only after confirming the previous runner is truly dead, not just slow.

---

### Q275 — Prometheus | Scenario-Based | Advanced

> Implement complete Prometheus monitoring for Redis on Kubernetes: exporter, ServiceMonitor, recording rules, and 3 alerting rules covering memory, hit rate, and evictions.

#### Key Points to Cover:
```
Step 1 — Redis Exporter deployment:
  # redis_exporter reads Redis INFO command and exposes as Prometheus metrics
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: redis-exporter
    namespace: production
  spec:
    template:
      spec:
        containers:
        - name: redis-exporter
          image: oliver006/redis_exporter:latest
          env:
          - name: REDIS_ADDR
            value: "redis://redis-headless.production.svc.cluster.local:6379"
          - name: REDIS_PASSWORD
            valueFrom:
              secretKeyRef:
                name: redis-secret
                key: password
          ports:
          - containerPort: 9121  # metrics port

  # Service for scraping:
  apiVersion: v1
  kind: Service
  metadata:
    name: redis-exporter
    labels:
      app: redis-exporter
  spec:
    ports:
    - name: metrics
      port: 9121

Step 2 — ServiceMonitor:
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: redis-exporter
    labels:
      release: prometheus
  spec:
    selector:
      matchLabels:
        app: redis-exporter
    endpoints:
    - port: metrics
      interval: 30s

Key Redis Metrics:
  redis_memory_used_bytes          # current memory usage
  redis_memory_max_bytes           # maxmemory config (0 = no limit)
  redis_keyspace_hits_total        # cache hits (counter)
  redis_keyspace_misses_total      # cache misses (counter)
  redis_connected_clients          # current client connections
  redis_config_maxclients          # maxclients config
  redis_evicted_keys_total         # evicted keys (counter)
  redis_replication_backlog_bytes  # replication backlog size
  redis_connected_slaves           # number of replicas
  redis_master_last_io_seconds_ago # replication lag proxy metric

Step 3 — Recording Rules:
  groups:
  - name: redis_recording
    rules:
    # Memory utilization %:
    - record: redis:memory_utilization:ratio
      expr: |
        redis_memory_used_bytes / redis_memory_max_bytes
        # NOTE: if maxmemory = 0 (no limit), this returns +Inf — handle with clamp

    # Cache hit rate:
    - record: redis:hit_rate:rate5m
      expr: |
        rate(redis_keyspace_hits_total[5m])
        /
        (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))

    # Client connection utilization:
    - record: redis:client_utilization:ratio
      expr: redis_connected_clients / redis_config_maxclients

    # Eviction rate per second:
    - record: redis:eviction_rate:rate5m
      expr: rate(redis_evicted_keys_total[5m])

Step 4 — Alerting Rules:
  groups:
  - name: redis_alerts
    rules:
    # Alert 1: High memory usage
    - alert: RedisHighMemoryUsage
      expr: |
        redis:memory_utilization:ratio > 0.85
        and redis_memory_max_bytes > 0  # skip if no limit set
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Redis memory usage > 85% on {{ $labels.instance }}"
        description: "Memory: {{ $value | humanizePercentage }}. Consider increasing maxmemory or adding more data eviction."

    # Alert 2: Low cache hit rate (cache becoming ineffective)
    - alert: RedisLowHitRate
      expr: redis:hit_rate:rate5m < 0.5
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Redis cache hit rate below 50%"
        description: "Hit rate: {{ $value | humanizePercentage }}. Keys may be expiring too fast or cache is undersized."

    # Alert 3: Keys being evicted (memory pressure indicator)
    - alert: RedisKeysEvicting
      expr: redis:eviction_rate:rate5m > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Redis is evicting keys — memory limit reached"
        description: "Eviction rate: {{ $value | humanize }} keys/sec. Increase maxmemory or reduce dataset size."

    # Alert 4: Replication lag
    - alert: RedisReplicationLag
      expr: redis_master_last_io_seconds_ago > 30
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Redis replica lagging behind master"
        description: "Last replication: {{ $value }}s ago. Network or master overload."
```

> 💡 **Interview tip:** The **cache hit rate formula** is the most important Redis metric — it tells you if your cache is actually working. A hit rate below 70% often means TTLs are too short, the cache is undersized, or access patterns don't match what's cached. The alert on **eviction rate > 0** is the key production signal: if Redis is evicting keys, it means the dataset is larger than maxmemory — you're in memory pressure territory. This is always a P1/P2 issue for cache-dependent applications because evictions cause unexpected cache misses and latency spikes.

---

### Q276 — Kubernetes | Troubleshooting | Advanced

> Hundreds of Evicted pods accumulating. How to identify root cause, bulk clean up, and prevent recurrence.

#### Key Points to Cover:
```
Why evicted pods accumulate:
  → Kubelet evicts pods when a node experiences resource pressure
  → Eviction policies: DiskPressure, MemoryPressure, PIDPressure
  → Evicted pods stay in FAILED/Evicted state (not auto-deleted)
  → Kubernetes keeps them for: kubectl describe, event history, debugging
  → No automatic cleanup mechanism in default config

Why evictions are happening — diagnosis:
  # Check node conditions:
  kubectl get nodes -o custom-columns=NAME:.metadata.name,MEMORY_PRESSURE:.status.conditions[1].status,DISK_PRESSURE:.status.conditions[0].status,PID_PRESSURE:.status.conditions[2].status

  # Check kubelet eviction thresholds:
  kubectl describe node <node> | grep -A20 "Conditions:"

  # Check recent eviction events:
  kubectl get events -A | grep -i evict | sort -k1,1 | head -20

  # Find most commonly evicted namespaces/pods:
  kubectl get pods -A | grep Evicted | awk '{print $1}' | sort | uniq -c | sort -rn

  # Look at why a specific pod was evicted:
  kubectl describe pod <evicted-pod> -n <ns> | grep -A5 "Message:"
  # Shows: "The node was low on resource: memory. Threshold quantity: 100Mi, available: 50Mi"

Common eviction causes:
  Memory pressure:
    → Pods without memory limits → one pod consumes all node memory
    → Fix: set memory limits on all pods; use LimitRange defaults

  Disk pressure (most common!):
    → /var/lib/docker or /var/lib/containerd filling up
    → Large container image layers, excessive logs, leftover temp files
    → Fix: log rotation, node image cleanup, larger EBS volumes
    → docker system prune -f on nodes

  PID pressure:
    → Process count approaching kernel pid_max
    → Fix: see Q321 (fork bomb, bad scripts)

Bulk cleanup of evicted pods:

  # Option 1: kubectl (works across all namespaces):
  kubectl get pods -A | grep Evicted | awk '{print $1 " " $2}' | \
    xargs -n 2 sh -c 'kubectl delete pod $2 -n $1' _

  # Option 2: cleaner one-liner:
  kubectl get pods --all-namespaces --field-selector=status.phase=Failed \
    -o json | kubectl delete -f -

  # Option 3: per namespace:
  kubectl delete pods -n production --field-selector=status.phase=Failed

  # Option 4: loop with safety check:
  for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
    count=$(kubectl get pods -n $ns | grep Evicted | wc -l)
    echo "$ns: $count evicted pods"
    kubectl delete pods -n $ns --field-selector=status.phase=Failed 2>/dev/null
  done

Prevention strategies:

  1. Set resource limits on all pods (most important):
     → Without limits: one pod can consume all node memory → evicts others
     → Use LimitRange to set defaults

  2. CronJob for automatic evicted pod cleanup:
     apiVersion: batch/v1
     kind: CronJob
     metadata:
       name: cleanup-evicted-pods
     spec:
       schedule: "*/15 * * * *"   # every 15 minutes
       jobTemplate:
         spec:
           template:
             spec:
               serviceAccountName: pod-cleaner
               containers:
               - name: kubectl
                 image: bitnami/kubectl
                 command:
                 - /bin/sh
                 - -c
                 - kubectl delete pods --all-namespaces --field-selector=status.phase=Failed

  3. Alert before eviction threshold is reached:
     # Alert at 80% memory/disk → fix before eviction starts
     - alert: NodeMemoryPressureImminent
       expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.15
       for: 5m

  4. PodDisruptionBudget + priorityClasses:
     → High priority pods survive eviction; low priority pods evicted first
     → Protect critical services with PriorityClass
```

> 💡 **Interview tip:** The most often missed root cause: **disk pressure from container image layers and log files**, not memory. Nodes that run many different containers accumulate image layers and log files. The cleanup command `docker system prune` or `crictl rmi --prune` (for containerd) frees space immediately. For prevention, the most impactful change: **set memory limits on every pod** and use LimitRange to enforce it. A single pod without limits that has a memory leak will evict all other pods on its node. The CronJob cleanup pattern is a pragmatic short-term fix — but investigating and eliminating the root cause of evictions is the real goal.

---

### Q277 — Git | Conceptual | Advanced

> Explain `git rerere` and `git notes`. When is rerere most useful? Give a DevOps use case for git notes.

#### Key Points to Cover:
```
git rerere (Reuse Recorded Resolution):
  → Records how you resolved a merge conflict
  → When the same conflict appears again, automatically re-applies your resolution
  → "rere" = "reuse recorded resolution"

  Enable:
    git config --global rerere.enabled true
    # Or per repo: git config rerere.enabled true
    # Stored in: .git/rr-cache/

  How it works:
    1. You hit a conflict during merge/rebase
    2. You manually resolve it
    3. git rerere records: (pre-conflict state, resolution)
    4. Next time same conflict appears: rerere auto-applies resolution
    5. You still review and commit — rerere doesn't commit automatically

  Most useful when:
    → Long-running feature branches that need periodic rebasing onto main
    → Same file changed in both feature and main repeatedly
    → Release branches: cherry-picking many commits across releases
    → Rebase-heavy workflows where same conflict appears multiple times

  Workflow:
    git config rerere.enabled true
    git rebase main  # conflict in config.py
    # resolve conflict...
    git add config.py
    git rebase --continue
    # rerere recorded the resolution

    # Next week, rebase again:
    git rebase main  # same conflict in config.py
    # rerere: automatically applies your previous resolution
    # Review, then: git add && git rebase --continue

  Check recorded resolutions:
    git rerere status    # what conflicts would rerere resolve
    git rerere diff      # show what rerere would apply
    ls .git/rr-cache/    # all recorded resolutions

git notes:
  → Attach additional information to a commit WITHOUT modifying it
  → Notes are stored separately from commits (in refs/notes/commits)
  → The commit hash stays the same (no history rewrite)
  → Not included in regular git log by default

  Add a note:
    git notes add -m "Deployed to production at 14:32 UTC" abc1234
    git notes append -m "Hotfix applied at 15:00 UTC" abc1234

  View notes:
    git log --show-notes       # includes notes in log output
    git notes show abc1234     # notes for specific commit

  Push/share notes (not automatic!):
    git push origin refs/notes/commits   # must explicitly push
    git fetch origin refs/notes/commits  # and explicitly fetch

DevOps use case — attaching CI metadata to commits:
  # In GitHub Actions or Jenkins, after a build:
  git notes add -m "{
    'build_id': '$BUILD_NUMBER',
    'test_results': 'passed',
    'coverage': '87%',
    'image': 'myapp:abc1234',
    'deployed_to': 'staging',
    'deploy_time': '$(date -u +%Y-%m-%dT%H:%M:%SZ)'
  }" $GITHUB_SHA

  # When investigating a production issue:
  git log --show-notes origin/main
  # See: exactly which build, test results, and deployment info
  # for each commit — without modifying commit history

  Other use cases:
  → Code review approvals: "Approved by: alice, bob"
  → Security scan results: "SAST: pass, CVE-2024-xxxx: N/A"
  → Release notes: "Included in release v2.1.0"
  → Performance regression test results per commit
```

> 💡 **Interview tip:** `git rerere` is the answer to "how do you handle the same merge conflict over and over?" — it's essential for teams that maintain long-lived feature branches or release branches. Most candidates have never heard of it, so knowing it signals you've worked in complex branching environments. For `git notes`, the key insight: they solve the **metadata problem without history pollution** — you want to know which CI build created a commit, but adding that to the commit message would require amending the commit (changing its hash, breaking history). Notes attach after the fact without touching the original commit.

---

### Q278 — AWS | Conceptual | Advanced

> Explain Savings Plans vs Reserved Instances vs Spot vs On-Demand. Compute vs EC2 Instance Savings Plans. What tools recommend the right mix?

#### Key Points to Cover:
```
On-Demand:
  → Pay per hour/second, no commitment
  → Most expensive, most flexible
  → Use: unpredictable workloads, dev/test, short-lived jobs
  → No upfront cost, can stop anytime

Reserved Instances (RI):
  → Commit to specific: instance type + region + OS + tenancy
  → 1-year or 3-year term
  → Discount: up to 72% vs On-Demand
  → Three payment options:
    - All Upfront: best discount (~60-72%)
    - Partial Upfront: moderate discount
    - No Upfront: lowest commitment, least discount
  → Convertible RIs: can change instance type (smaller discount ~54%)
  → Limitation: locked to specific instance family + size + region

Savings Plans (newer, more flexible than RIs):
  → Commit to $/hour spend (not specific instances)
  → 1-year or 3-year term
  → Two types:

  Compute Savings Plans (most flexible):
    → Discount: up to 66% vs On-Demand
    → Applies to: ANY EC2 instance (any family, any size, any region)
    → Also: Lambda (up to 17%), Fargate (up to 52%)
    → Best for: organizations that change instance types frequently
    → Can switch region, OS, tenancy freely

  EC2 Instance Savings Plans (higher discount, less flexible):
    → Discount: up to 72% vs On-Demand
    → Locked to: specific instance family in specific region
    → Can change: size, OS, tenancy (within family + region)
    → Example: commit to "m5 in us-east-1" → can use m5.large, m5.xlarge etc.
    → Best for: stable workloads with fixed instance family

Spot Instances:
  → AWS's spare capacity, up to 90% discount
  → Can be interrupted with 2-minute warning when AWS needs capacity back
  → Use: fault-tolerant, stateless, batch, CI/CD workers, ML training
  → NEVER use for: databases, stateful services, anything requiring uptime

Decision framework for production workloads:
  Baseline steady state (always on):
    → 60-70% On-Demand equivalent → buy Compute Savings Plans (3-year)
    → Covers: ECS Fargate, Lambda, EC2 with flexibility

  Predictable peaks:
    → Remaining ~20% → EC2 Instance Savings Plans (1-year)
    → For known workloads (nightly batch, specific service fleet)

  Variable/burst work:
    → Remaining ~20% → On-Demand

  Batch / CI / fault-tolerant:
    → 100% Spot for these workloads specifically
    → Use Spot with diversified instance types (multiple families)

AWS tools for optimization:
  AWS Cost Explorer:
    → Savings Plans Recommendations tab
    → Shows: how much you'd save with specific plan purchases
    → Based on: your last 7/30/60 days of On-Demand usage

  AWS Trusted Advisor:
    → Cost Optimization checks
    → Identifies: underutilized RIs, idle EC2 instances

  AWS Compute Optimizer:
    → ML-based right-sizing recommendations
    → Analyzes: CloudWatch metrics to suggest instance type changes
    → Identifies: over-provisioned instances wasting money

  Third-party: CloudHealth, Spot.io, Apptio Cloudability
```

> 💡 **Interview tip:** The key interviewer trap: confusing Compute Savings Plans with EC2 Instance Savings Plans. Remember: **Compute = most flexible (any instance, any region, includes Lambda/Fargate)**, **EC2 Instance = higher discount but locked to family + region**. The production recommendation: buy Compute Savings Plans for baseline because they're flexible as your architecture evolves, then add EC2 Instance Savings Plans for specific stable workloads. Spot is a completely separate category — it's not a discount tier for normal workloads, it's for workloads that can be interrupted. Mixing Spot with ASG and diversified instance types is the right pattern.

---

### Q279 — Jenkins | Scenario-Based | Advanced

> Write a Jenkins pipeline for progressive feature flag rollout with Prometheus monitoring and automatic rollback.

#### Key Points to Cover:
```groovy
pipeline {
  agent { kubernetes { yaml '''
    spec:
      containers:
      - name: curl
        image: curlimages/curl:latest
        command: [cat]
        tty: true
  ''' }}

  environment {
    FEATURE_FLAG_API   = "https://unleash.company.internal/api"
    UNLEASH_TOKEN      = credentials('unleash-api-token')
    PROMETHEUS_URL     = "http://prometheus.monitoring.svc.cluster.local:9090"
    ERROR_THRESHOLD    = "0.05"  // 5% error rate
    SLACK_CHANNEL      = "#deployments"
    DEPLOYER           = "${currentBuild.rawBuild.getCause(hudson.model.Cause.UserIdCause)?.userId ?: 'automated'}"
  }

  stages {
    stage('Deploy Application') {
      steps {
        echo "Deploying new version by ${DEPLOYER}..."
        // Standard deployment step
        sh 'kubectl set image deployment/payment-service payment=payment:${GIT_COMMIT[0..7]}'
        sh 'kubectl rollout status deployment/payment-service --timeout=5m'
      }
    }

    stage('Enable Canary — 1%') {
      steps {
        script {
          setFeatureFlag('payment-new-flow', 1)
          logFlagChange('payment-new-flow', 0, 1)
        }
        sleep(time: 2, unit: 'MINUTES')
        script { checkErrorRate('payment-new-flow', 1) }
      }
    }

    stage('Expand to 10%') {
      steps {
        script {
          setFeatureFlag('payment-new-flow', 10)
          logFlagChange('payment-new-flow', 1, 10)
        }
        sleep(time: 5, unit: 'MINUTES')
        script { checkErrorRate('payment-new-flow', 10) }
      }
    }

    stage('Expand to 50%') {
      steps {
        script {
          setFeatureFlag('payment-new-flow', 50)
          logFlagChange('payment-new-flow', 10, 50)
        }
        sleep(time: 5, unit: 'MINUTES')
        script { checkErrorRate('payment-new-flow', 50) }
      }
    }

    stage('Full Rollout — 100%') {
      steps {
        script {
          setFeatureFlag('payment-new-flow', 100)
          logFlagChange('payment-new-flow', 50, 100)
        }
        sleep(time: 10, unit: 'MINUTES')
        script { checkErrorRate('payment-new-flow', 100) }
      }
    }
  }

  post {
    failure {
      script {
        echo "Rolling back feature flag due to pipeline failure"
        setFeatureFlag('payment-new-flow', 0)
        logFlagChange('payment-new-flow', -1, 0)
      }
      slackSend channel: env.SLACK_CHANNEL, color: 'danger',
        message: "❌ Feature flag rollback triggered by ${DEPLOYER}: payment-new-flow disabled"
    }
    success {
      slackSend channel: env.SLACK_CHANNEL, color: 'good',
        message: "✅ Feature flag fully rolled out by ${DEPLOYER}: payment-new-flow at 100%"
    }
  }
}

// ──────────────── Helper functions ────────────────

def setFeatureFlag(String flagName, int percentage) {
  container('curl') {
    sh """
      curl -s -X POST ${FEATURE_FLAG_API}/admin/features/${flagName}/percentage \
        -H 'Authorization: ${UNLEASH_TOKEN}' \
        -H 'Content-Type: application/json' \
        -d '{"percentage": ${percentage}}'
    """
  }
  echo "Feature flag '${flagName}' set to ${percentage}%"
}

def logFlagChange(String flagName, int from, int to) {
  // Audit log: who changed what and when
  def logEntry = """
    {
      "timestamp": "${new Date().format("yyyy-MM-dd'T'HH:mm:ss'Z'")}",
      "flag": "${flagName}",
      "from": ${from},
      "to": ${to},
      "operator": "${DEPLOYER}",
      "build": "${env.BUILD_NUMBER}",
      "pipeline": "${env.JOB_NAME}"
    }
  """
  writeFile file: 'flag-audit.log', text: logEntry
  archiveArtifacts 'flag-audit.log'
}

def checkErrorRate(String flagName, int currentPercentage) {
  container('curl') {
    def query = URLEncoder.encode(
      "rate(http_requests_total{status=~'5..',feature_flag='${flagName}'}[5m]) / rate(http_requests_total{feature_flag='${flagName}'}[5m])",
      'UTF-8'
    )
    def response = sh(
      script: "curl -s '${PROMETHEUS_URL}/api/v1/query?query=${query}'",
      returnStdout: true
    )
    def result = readJSON text: response
    def errorRate = result.data?.result?.getAt(0)?.value?.getAt(1)?.toDouble() ?: 0.0

    echo "Current error rate at ${currentPercentage}%: ${errorRate}"

    if (errorRate > ERROR_THRESHOLD.toDouble()) {
      error("Error rate ${errorRate} exceeds threshold ${ERROR_THRESHOLD} — triggering rollback")
    }
  }
}
```

> 💡 **Interview tip:** The **audit log** requirement is what separates a production pipeline from a demo. Regulators and post-incident reviews need to know: who changed the flag, when, and to what value. Archiving the audit log as a build artifact makes it retrievable long after the build. Also highlight: the `post { failure }` block is critical — without it, a pipeline failure at 50% leaves the flag at 50% forever. The rollback must be in the `post` block, not in the stage, so it runs regardless of which stage failed.

---

### Q280 — ELK Stack | Conceptual | Advanced

> Explain Elasticsearch aggregation categories. Write queries for top slow endpoints, error rate per service per hour, and geographic user distribution.

#### Key Points to Cover:
```
4 Categories of Elasticsearch Aggregations:

  1. Metric Aggregations:
     → Calculate a single numeric value from a set of documents
     → Examples: avg, sum, min, max, percentiles, cardinality, value_count
     → Used for: response times, sizes, counts

  2. Bucket Aggregations:
     → Group documents into "buckets" based on field values
     → Examples: terms, date_histogram, range, geo_distance, histogram
     → Each bucket can have sub-aggregations
     → Used for: group by service, time ranges, IP ranges

  3. Pipeline Aggregations:
     → Operate on the OUTPUT of other aggregations (not documents)
     → Examples: derivative, moving_avg, bucket_sort, bucket_selector
     → Used for: rate of change, anomaly detection, sorting buckets

  4. Matrix Aggregations:
     → Operate across multiple fields simultaneously
     → Example: matrix_stats (correlation between fields)
     → Less commonly used

Query 1 — Top 5 slowest API endpoints (last hour):
  POST /logs-*/_search
  {
    "query": {
      "range": {
        "@timestamp": { "gte": "now-1h", "lte": "now" }
      }
    },
    "aggs": {
      "by_endpoint": {
        "terms": {
          "field": "request.path.keyword",
          "size": 5,
          "order": { "avg_response_time": "desc" }  // sort by slowest
        },
        "aggs": {
          "avg_response_time": {
            "avg": { "field": "response_time_ms" }
          },
          "p99_response_time": {
            "percentiles": {
              "field": "response_time_ms",
              "percents": [99]
            }
          },
          "request_count": {
            "value_count": { "field": "response_time_ms" }
          }
        }
      }
    },
    "size": 0  // don't return documents, only aggregation results
  }

Query 2 — Error rate per service per 1-hour bucket (last 24 hours):
  POST /logs-*/_search
  {
    "query": {
      "range": {
        "@timestamp": { "gte": "now-24h", "lte": "now" }
      }
    },
    "aggs": {
      "by_service": {
        "terms": {
          "field": "service.name.keyword",
          "size": 50
        },
        "aggs": {
          "by_hour": {
            "date_histogram": {
              "field": "@timestamp",
              "calendar_interval": "1h",
              "format": "yyyy-MM-dd HH:mm"
            },
            "aggs": {
              "total_requests": { "value_count": { "field": "status_code" } },
              "error_requests": {
                "filter": {
                  "range": { "status_code": { "gte": 500 } }
                }
              },
              "error_rate": {
                "bucket_script": {                      // pipeline agg
                  "buckets_path": {
                    "errors": "error_requests._count",
                    "total": "total_requests"
                  },
                  "script": "params.errors / params.total"
                }
              }
            }
          }
        }
      }
    },
    "size": 0
  }

Query 3 — Geographic distribution by country:
  POST /logs-*/_search
  {
    "aggs": {
      "by_country": {
        "terms": {
          "field": "geoip.country_name.keyword",
          "size": 20,
          "order": { "_count": "desc" }
        },
        "aggs": {
          "unique_users": {
            "cardinality": { "field": "user.id.keyword" }
          }
        }
      }
    },
    "size": 0
  }

  // For geo_point heatmap (Kibana Maps):
  "aggs": {
    "geo_centroid": {
      "geohash_grid": {
        "field": "geoip.location",
        "precision": 3
      }
    }
  }
```

> 💡 **Interview tip:** The `size: 0` at the top level is critical and often forgotten — without it, Elasticsearch returns documents AND aggregation results, wasting bandwidth and compute for pure analytics queries. The **bucket_script pipeline aggregation** for error rate is the key advanced concept — it calculates a derived metric (error rate) from two other aggregations (errors, total) within each bucket. Without this, you'd need to calculate error rate client-side. Mention that for performance, aggregations on **`.keyword` fields** (not analyzed) are dramatically faster than on text fields.

---

### Q281 — Kubernetes | Scenario-Based | Advanced

> Write NetworkPolicies for gRPC inter-service communication (port 50051), Prometheus scraping (port 9090), and explain gRPC/HTTP2 considerations.

#### Key Points to Cover:
```
gRPC over HTTP/2 vs HTTP/1.1 for NetworkPolicy:
  → NetworkPolicy operates at Layer 3/4 (IP + TCP port)
  → For gRPC: same as any TCP policy — specify port 50051
  → HTTP/2 multiplexes multiple streams over ONE connection
    → One TCP connection for many gRPC calls (unlike HTTP/1.1 per-request)
    → NetworkPolicy still works: it controls TCP connections not streams
  → Special consideration: HTTP/2 connection persistence
    → If policy changes while connection is open, existing connection not immediately affected
    → New connections immediately subject to new policy
  → Use standard NetworkPolicy — no special gRPC handling needed at L4

NetworkPolicy Setup:

  # Default deny all ingress in production namespace first:
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: default-deny-all
    namespace: production
  spec:
    podSelector: {}  # applies to ALL pods
    policyTypes:
    - Ingress
    - Egress

  # Allow service-a to call service-b on 50051:
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-service-a-to-service-b
    namespace: production
  spec:
    podSelector:
      matchLabels:
        app: service-b       # DESTINATION: applies to service-b pods
    policyTypes:
    - Ingress
    ingress:
    - from:
      - podSelector:
          matchLabels:
            app: service-a   # SOURCE: only from service-a
      ports:
      - port: 50051
        protocol: TCP

  # Allow service-b to call service-c on 50051:
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-service-b-to-service-c
    namespace: production
  spec:
    podSelector:
      matchLabels:
        app: service-c
    ingress:
    - from:
      - podSelector:
          matchLabels:
            app: service-b
      ports:
      - port: 50051
        protocol: TCP

  # Allow Prometheus to scrape all services on port 9090:
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-prometheus-scraping
    namespace: production
  spec:
    podSelector: {}   # applies to ALL pods in namespace
    policyTypes:
    - Ingress
    ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: monitoring
        podSelector:
          matchLabels:
            app: prometheus    # AND logic: from monitoring NS AND prometheus pod
      ports:
      - port: 9090
        protocol: TCP

  # Also need egress policies for services to call each other:
  # Allow service-a egress to service-b:
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: service-a-egress
    namespace: production
  spec:
    podSelector:
      matchLabels:
        app: service-a
    policyTypes:
    - Egress
    egress:
    - to:
      - podSelector:
          matchLabels:
            app: service-b
      ports:
      - port: 50051
    # Also allow DNS (required for service discovery):
    - to:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: kube-system
        podSelector:
          matchLabels:
            k8s-app: kube-dns
      ports:
      - port: 53
        protocol: UDP
      - port: 53
        protocol: TCP

Testing policies:
  # Verify connectivity:
  kubectl exec -it service-a-pod -- grpc_health_probe -addr=service-b:50051
  # Should: succeed
  
  # Verify blocking:
  kubectl exec -it service-c-pod -- grpc_health_probe -addr=service-b:50051
  # Should: timeout (policy blocks this)
```

> 💡 **Interview tip:** The most common NetworkPolicy mistake: **forgetting egress policies**. Ingress policies on service-b allow service-a to connect, but if service-a has a default-deny-egress policy, the connection is still blocked (from service-a's side). You need both: ingress on the destination AND egress on the source. The DNS egress allowance is the other frequent miss — after applying default-deny-all egress, DNS breaks, and services can't resolve each other's names. Always add the DNS egress rule explicitly.

---

### Q282 — Linux / Bash | Troubleshooting | Advanced

> NTP is out of sync by 45 seconds causing TLS failures and JWT errors. Diagnose, fix, and set up monitoring for clock drift.

#### Key Points to Cover:
```
Immediate impact of 45-second clock skew:
  TLS: certificates have notBefore/notAfter fields checked against system clock
    → 45s skew: might not be enough to fail TLS itself (tolerance varies)
    → But: OCSP stapling, HSTS, etc. have stricter requirements

  JWT: iat (issued at) and exp (expiry) claims checked against clock
    → Token issued by service A, validated by service B
    → If B's clock is 45s ahead: token looks expired before it should be
    → If B's clock is 45s behind: token looks issued in the future → rejected

  Distributed tracing: timestamps from different services don't align
    → Spans appear in wrong order → misleading traces

Diagnosis:
  # Check current time sync status:
  timedatectl status
  # Look for: "System clock synchronized: yes/no"
  # Look for: "NTP service: active/inactive"

  # Modern systems (chrony — default on RHEL/Amazon Linux):
  chronyc tracking
  # Shows: Reference ID (NTP server), System time offset, Stratum
  # Key field: System time — how far off current is

  chronyc sources -v
  # Shows: all configured NTP servers, reachability, last offset
  # '*' prefix = currently selected source
  # '?' = unreachable source

  # Older systems (ntpd):
  ntpq -p
  # Shows: NTP peers, offset (ms), jitter
  ntpstat
  # Shows: synchronized yes/no

  # Check if NTP service is running:
  systemctl status chronyd   # RHEL/Amazon Linux
  systemctl status ntp       # Ubuntu
  systemctl status systemd-timesyncd  # minimal systems

  # Check firewall (NTP uses UDP 123):
  iptables -L INPUT | grep 123
  nc -zu pool.ntp.org 123 && echo "NTP reachable"

Fix — immediate correction:
  # Force sync immediately (chrony):
  chronyc makestep
  # → Steps the clock immediately (can go backward!)
  # → Use for large offsets (>1s): gradual adjustment takes too long

  # For small offsets: chrony adjusts gradually (slews) automatically
  # Once reachable NTP servers are found, chrony adjusts ~0.5ms/s

  # Restart chrony if stuck:
  systemctl restart chronyd

  # If NTP server unreachable (check /etc/chrony.conf):
  cat /etc/chrony.conf | grep server
  # In AWS: should use 169.254.169.123 (AWS Time Sync Service)
  # Or: pool.ntp.org
  
  # AWS recommended config:
  echo "server 169.254.169.123 prefer iburst" >> /etc/chrony.conf
  systemctl restart chronyd

Monitoring for clock drift:
  # Prometheus Node Exporter exports NTP metrics:
  node_timex_offset_seconds    # current offset in seconds
  node_timex_sync_status       # 1 = in sync, 0 = not
  
  # Alert rule:
  - alert: ClockSkewTooHigh
    expr: abs(node_timex_offset_seconds) > 0.5  # 500ms threshold
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Clock skew on {{ $labels.instance }} is {{ $value }}s"

  - alert: NTPNotSynchronized
    expr: node_timex_sync_status == 0
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "NTP not synchronized on {{ $labels.instance }}"

  # Kubernetes/EKS: container clocks inherit from host
  # → Fix NTP on the node, not inside containers
  # → AWS EKS nodes should always use 169.254.169.123
```

> 💡 **Interview tip:** The critical distinction: **`chronyc makestep` vs slewing**. For a 45-second offset, `makestep` is necessary — gradual slewing at 0.5ms/s would take 25 hours to correct a 45-second offset. But stepping the clock backward can cause problems: timestamps in databases go backward, cron jobs fire at wrong times, Kerberos tickets may be invalidated. In AWS environments, always configure **169.254.169.123** (AWS Time Sync Service) as the primary NTP source — it's a local hypervisor-level time source with microsecond accuracy and no external network dependency.

---

### Q283 — Terraform | Scenario-Based | Advanced

> Implement Terraform policy as code using OPA/Conftest to enforce S3 encryption, EC2 tags, SSH security groups, and RDS deletion protection in CI/CD.

#### Key Points to Cover:
```
Conftest + OPA workflow:
  1. terraform plan -out=tfplan
  2. terraform show -json tfplan > tfplan.json
  3. conftest test tfplan.json --policy ./policies/
  4. If violations: fail the pipeline, show violations
  5. If all pass: allow terraform apply

Project structure:
  infrastructure/
  ├── main.tf
  ├── policies/
  │   ├── s3_encryption.rego
  │   ├── ec2_tagging.rego
  │   ├── security_group.rego
  │   └── rds_protection.rego
  └── .github/workflows/terraform.yml

Policy 1 — S3 encryption required (s3_encryption.rego):
  package main

  deny[msg] {
    resource := input.planned_values.root_module.resources[_]
    resource.type == "aws_s3_bucket"
    
    # Check if server_side_encryption_configuration is missing:
    not resource.values.server_side_encryption_configuration
    
    msg := sprintf(
      "S3 bucket '%s' must have server-side encryption enabled",
      [resource.address]
    )
  }

  # Also check for separate encryption resource:
  deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket_server_side_encryption_configuration"
    resource.change.after.rule[_].apply_server_side_encryption_by_default[_].sse_algorithm != "aws:kms"
    
    msg := sprintf(
      "S3 bucket '%s' must use aws:kms encryption, not AES256",
      [resource.address]
    )
  }

Policy 2 — EC2 must have CostCenter tag (ec2_tagging.rego):
  package main

  required_tags := {"CostCenter", "Environment", "Owner"}

  deny[msg] {
    resource := input.planned_values.root_module.resources[_]
    resource.type == "aws_instance"
    
    missing := required_tags - {tag | resource.values.tags[tag]}
    count(missing) > 0
    
    msg := sprintf(
      "EC2 instance '%s' is missing required tags: %v",
      [resource.address, missing]
    )
  }

Policy 3 — No SSH open to world (security_group.rego):
  package main

  deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_security_group"
    
    ingress := resource.change.after.ingress[_]
    ingress.from_port <= 22
    ingress.to_port >= 22
    ingress.cidr_blocks[_] == "0.0.0.0/0"
    
    msg := sprintf(
      "Security group '%s' allows SSH (port 22) from 0.0.0.0/0 — use VPN/bastion instead",
      [resource.address]
    )
  }

  # Also check IPv6:
  deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_security_group"
    ingress := resource.change.after.ingress[_]
    ingress.from_port <= 22
    ingress.to_port >= 22
    ingress.ipv6_cidr_blocks[_] == "::/0"
    msg := sprintf("Security group '%s' allows SSH from ::/0", [resource.address])
  }

Policy 4 — RDS deletion protection (rds_protection.rego):
  package main

  deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_db_instance"
    resource.change.after.deletion_protection != true
    
    msg := sprintf(
      "RDS instance '%s' must have deletion_protection = true",
      [resource.address]
    )
  }

GitHub Actions integration:
  - name: Terraform Plan
    run: |
      terraform plan -out=tfplan
      terraform show -json tfplan > tfplan.json

  - name: Install conftest
    run: |
      curl -LO https://github.com/open-policy-agent/conftest/releases/download/v0.46.0/conftest_0.46.0_Linux_x86_64.tar.gz
      tar xzf conftest_0.46.0_Linux_x86_64.tar.gz
      chmod +x conftest && mv conftest /usr/local/bin/

  - name: Run OPA Policies
    run: |
      conftest test tfplan.json \
        --policy policies/ \
        --output table
      # Exits non-zero if any deny rules fire → pipeline fails
```

> 💡 **Interview tip:** The key insight about policy-as-code placement: run policies **on the terraform plan output** (JSON), not on `.tf` files. Plan JSON contains the RESOLVED values including defaults and interpolations — checking `.tf` files directly misses dynamic values. The `conftest test` command exits non-zero on violations, which naturally fails CI/CD pipelines. Also mention that this is a **shift-left** approach — catching policy violations at PR time (before production) rather than auditing after-the-fact with AWS Config rules.

---

### Q284 — GitHub Actions | Scenario-Based | Advanced

> Write a GitHub Actions workflow for a monorepo with 5 services: detect changed services, parallel pipelines, deploy only changed+tested services, PR summary comment.

#### Key Points to Cover:
```yaml
name: Monorepo CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # Job 1: Detect which services changed
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      frontend:     ${{ steps.filter.outputs.frontend }}
      backend:      ${{ steps.filter.outputs.backend }}
      auth:         ${{ steps.filter.outputs.auth }}
      payments:     ${{ steps.filter.outputs.payments }}
      notifications: ${{ steps.filter.outputs.notifications }}
      changed-matrix: ${{ steps.matrix.outputs.services }}
    steps:
    - uses: actions/checkout@v4
    - uses: dorny/paths-filter@v3
      id: filter
      with:
        filters: |
          frontend:
            - 'services/frontend/**'
            - 'shared/**'          # shared code change = all affected
          backend:
            - 'services/backend/**'
            - 'shared/**'
          auth:
            - 'services/auth/**'
          payments:
            - 'services/payments/**'
          notifications:
            - 'services/notifications/**'

    - name: Build changed services matrix
      id: matrix
      run: |
        SERVICES=()
        [[ "${{ steps.filter.outputs.frontend }}" == "true" ]]     && SERVICES+=("frontend")
        [[ "${{ steps.filter.outputs.backend }}" == "true" ]]      && SERVICES+=("backend")
        [[ "${{ steps.filter.outputs.auth }}" == "true" ]]         && SERVICES+=("auth")
        [[ "${{ steps.filter.outputs.payments }}" == "true" ]]     && SERVICES+=("payments")
        [[ "${{ steps.filter.outputs.notifications }}" == "true" ]] && SERVICES+=("notifications")
        
        # Build JSON array for matrix
        JSON=$(printf '"%s",' "${SERVICES[@]}" | sed 's/,$//')
        echo "services=[${JSON}]" >> $GITHUB_OUTPUT
        echo "Changed services: ${SERVICES[*]}"

  # Job 2: Test all changed services in parallel (matrix)
  test:
    needs: detect-changes
    if: needs.detect-changes.outputs.changed-matrix != '[]'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.changed-matrix) }}
      fail-fast: false   # don't cancel others if one fails
    steps:
    - uses: actions/checkout@v4
    - name: Run tests for ${{ matrix.service }}
      run: |
        cd services/${{ matrix.service }}
        npm ci && npm test
    - name: Upload test results
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: test-results-${{ matrix.service }}
        path: services/${{ matrix.service }}/test-results/

  # Job 3: Build and push Docker images (only changed services)
  build-push:
    needs: [detect-changes, test]
    if: |
      github.event_name == 'push' &&
      github.ref == 'refs/heads/main' &&
      needs.detect-changes.outputs.changed-matrix != '[]'
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.changed-matrix) }}
    steps:
    - uses: actions/checkout@v4
    - name: Configure AWS (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-actions
        aws-region: us-east-1
    - name: Build and push ${{ matrix.service }}
      run: |
        ECR="123456789012.dkr.ecr.us-east-1.amazonaws.com"
        aws ecr get-login-password | docker login --username AWS --password-stdin $ECR
        docker build -t $ECR/${{ matrix.service }}:${GITHUB_SHA::8} \
          services/${{ matrix.service }}/
        docker push $ECR/${{ matrix.service }}:${GITHUB_SHA::8}

  # Job 4: PR Summary comment
  pr-summary:
    needs: [detect-changes, test]
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
    - name: Post deployment summary
      uses: actions/github-script@v7
      with:
        script: |
          const changed = ${{ needs.detect-changes.outputs.changed-matrix }};
          const all = ['frontend', 'backend', 'auth', 'payments', 'notifications'];
          
          const rows = all.map(svc => {
            const wasChanged = changed.includes(svc);
            const status = wasChanged ? '✅ Tested' : '⏭️ Skipped (no changes)';
            return `| ${svc} | ${status} |`;
          }).join('\n');
          
          await github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: `## 🔍 CI/CD Summary\n\n| Service | Status |\n|---|---|\n${rows}\n\n**Changed services:** ${changed.join(', ') || 'none'}`
          });
```

> 💡 **Interview tip:** The `fail-fast: false` in the matrix strategy is critical for monorepos — without it, if the `auth` service tests fail, GitHub Actions cancels tests for `payments` and `notifications` even if they're fine. With `fail-fast: false`, all services are tested independently. Also explain the **shared code problem**: if `shared/` changes, ALL services need to be tested (since they all import shared code). The paths-filter `shared/**` under every service handles this — any change to shared triggers all service pipelines.

---

### Q285 — AWS | Troubleshooting | Advanced

> CloudFormation stack stuck in UPDATE_ROLLBACK_FAILED. Cannot update, delete, or roll back. How to safely recover.

#### Key Points to Cover:
```
What causes UPDATE_ROLLBACK_FAILED:
  → CloudFormation attempted an update
  → Update partially succeeded (some resources changed)
  → Rollback started (to undo the partial update)
  → Rollback itself failed (couldn't undo some changes)
  → Stack is now in a limbo state — CloudFormation can't proceed

  Common rollback failure causes:
  → Resource was manually modified outside CloudFormation
    (e.g., someone manually deleted a resource CloudFormation was trying to rollback)
  → Dependency change: resource A depends on B, B was already deleted manually
  → IAM permission removed during rollback
  → S3 bucket with objects (can't delete non-empty bucket during rollback)
  → RDS with deletion protection enabled
  → EC2 instance in stopped state that CloudFormation tried to restart

  State diagram:
  UPDATE_IN_PROGRESS → UPDATE_FAILED → UPDATE_ROLLBACK_IN_PROGRESS
                                      → UPDATE_ROLLBACK_FAILED ← stuck here

Recovery options:

Option 1 — Continue Rollback with Skip Resources:
  → Most common fix
  → Tell CloudFormation: "skip these resources during rollback"
  → CloudFormation rolls back everything else, marks skipped resources as failed
  
  aws cloudformation continue-update-rollback \
    --stack-name my-production-stack \
    --resources-to-skip LogicalResourceId1 LogicalResourceId2
  
  # Find which resources failed in rollback:
  aws cloudformation describe-stack-events \
    --stack-name my-production-stack \
    --query 'StackEvents[?ResourceStatus==`UPDATE_ROLLBACK_FAILED`]'
  
  # After continue-rollback:
  # Stack goes to ROLLBACK_COMPLETE (stable) but skipped resources may be wrong
  # Then: update the stack to fix the skipped resources

Option 2 — Stabilize the resource manually, then continue rollback:
  → If rollback failed because resource is in wrong state
  → Manually fix the resource to match what CloudFormation expects
  → Then retry rollback:
    aws cloudformation continue-update-rollback \
      --stack-name my-production-stack
  
  Example: S3 bucket with objects (CloudFormation tried to delete it):
  → Delete the objects manually
  → Retry continue-update-rollback (now it can delete the empty bucket)

Option 3 — Update stack to target state:
  → Instead of rolling back to old, move forward to new desired state
  → Delete the stuck resources manually
  → Call continue-update-rollback --resources-to-skip to skip them
  → After ROLLBACK_COMPLETE: do a new UPDATE with correct config

Option 4 — Delete and recreate (last resort):
  → If no production data is at risk
  → aws cloudformation delete-stack --stack-name my-stack
  → May fail if stack has DeletionPolicy: Retain resources
  → Retain policy: CloudFormation removes from stack but keeps the resource
  → Then recreate stack from scratch

Protecting against this situation:
  → Add DeletionPolicy: Retain to critical resources (RDS, S3)
  → Use CloudFormation drift detection regularly
  → Never modify CloudFormation resources outside of CloudFormation
  → Stack policies: prevent updates to critical resources without explicit override
```

> 💡 **Interview tip:** The `--resources-to-skip` flag is the key to unlocking UPDATE_ROLLBACK_FAILED safely. The trick: it tells CloudFormation to skip rolling back specific resources, complete the rollback for everything else, and leave the stack in ROLLBACK_COMPLETE state. This gets the stack to a stable state where you can then do a normal UPDATE to fix those skipped resources. The alternative of just deleting and recreating should be a last resort for production stacks — mention the **DeletionPolicy: Retain** protection so that even if the stack is deleted, the actual RDS/S3 resources survive.

---

### Q286 — Prometheus | Conceptual | Advanced

> Explain Prometheus exemplars — how they link high-latency metrics to trace IDs, what changes are needed end-to-end.

#### Key Points to Cover:
```
What are exemplars:
  → Specific data points attached to a metric observation
  → Contains: the sample value + labels including a trace ID
  → Purpose: when a metric shows a spike, find the SPECIFIC REQUEST that caused it
  → Standard: OpenMetrics (CNCF specification)
  → Supported: Prometheus 2.26+, Grafana 7.4+

Without exemplars:
  You see: P99 latency spike at 14:35
  You know: SOMETHING was slow at 14:35
  You don't know: WHICH request, WHICH user, WHAT trace

With exemplars:
  You see: P99 latency spike at 14:35
  You click the spike → exemplar shows: traceID=abc123
  You follow link to Jaeger/Tempo → see exact slow trace
  You know: EXACTLY which request, all service hops, where time was spent

How exemplars work technically:
  1. Application instrument with exemplar-aware library:
     # Python (prometheus_client with exemplar support):
     REQUEST_LATENCY.labels(endpoint='/api/orders').observe(
         value,
         exemplar={'traceID': get_current_trace_id()}
     )
     
     # Go (prometheus/client_golang):
     histogram.With(labels).ObserveWithExemplar(duration, prometheus.Labels{
         "traceID": span.SpanContext().TraceID().String(),
     })

  2. Prometheus scrapes: stores exemplar alongside histogram bucket
     # OpenMetrics text format:
     http_request_duration_seconds_bucket{le="0.5"} 1234 # {traceID="abc123"} 0.234 1640000000.000

  3. Prometheus must enable exemplar storage:
     # prometheus.yml:
     storage:
       exemplars:
         max_exemplars: 100000

  4. Grafana queries exemplars:
     → Enable "Enable exemplars" in data source config
     → Set exemplar label: "traceID"
     → Set trace data source: Jaeger or Tempo
     → Grafana shows diamonds on histogram panels at high values
     → Click diamond → follows traceID to Tempo/Jaeger automatically

OpenTelemetry standard:
  → Exemplars part of OpenMetrics specification (not original Prometheus format)
  → OTel SDK automatically propagates trace context
  → Bridge: OTel → Prometheus format includes exemplars if configured
  → Requires: W3C TraceContext or B3 propagation in headers

Requirements checklist:
  ✅ Application: using exemplar-aware metrics library
  ✅ Application: trace context propagated (OTel/Jaeger agent)
  ✅ Prometheus: OpenMetrics scrape format (scrape_configs: honor_exemplars: true)
  ✅ Prometheus: exemplar storage enabled (max_exemplars)
  ✅ Grafana: exemplar enabled in Prometheus data source
  ✅ Grafana: trace data source configured (Jaeger/Tempo URL)
```

> 💡 **Interview tip:** Exemplars solve the **metrics-to-traces correlation problem** — the most common observability gap. You can see that something is slow (metric) but can't see WHY it's slow (trace). The key limitation: exemplars are sampled — you don't get one for every request, only for notable ones (typically the slowest in a histogram bucket). This is by design for efficiency. In practice, exemplars pointing to the P99 or P999 latency samples are exactly the traces you want to investigate. Mention that this is part of the **three pillars of observability**: metrics, logs, and traces — and exemplars are the bridge between the first and third pillars.

---

### Q287 — Kubernetes | Conceptual | Advanced

> Explain LimitRange and ResourceQuota edge cases: pod > LimitRange max, quota without LimitRange, scopeSelector for BestEffort pods.

#### Key Points to Cover:
```
LimitRange — sets per-object limits within a namespace:
  → min/max: floor and ceiling for any container's requests/limits
  → default/defaultRequest: applied when container doesn't specify
  → maxLimitRequestRatio: max(limit/request) ratio

ResourceQuota — sets aggregate limits for entire namespace:
  → Hard limits: total resources all objects can consume
  → Soft limits: not supported natively (only hard)

Edge Case 1 — Pod requests more than LimitRange max:
  LimitRange: max.memory = 512Mi
  Pod spec: resources.limits.memory = 1Gi  ← exceeds max
  Result: pod REJECTED by admission controller immediately
  Error: "maximum memory usage per Container is 512Mi, but limit is 1Gi"
  
  → LimitRange is enforced at ADMISSION time (before scheduling)
  → Pod never reaches etcd or scheduler
  → Fix: lower pod's memory limit to ≤ 512Mi

Edge Case 2 — ResourceQuota without LimitRange:
  ResourceQuota: limits.memory = 4Gi (namespace total)
  No LimitRange exists
  
  Result: a container WITHOUT a memory limit = INFINITE memory request
  → Pod creation FAILS because ResourceQuota requires all containers
    to have explicit limits when quota is set for that resource
  Error: "must specify limits.memory for container"
  
  Solution: add LimitRange with defaults → new containers get defaults
  → LimitRange provides per-container defaults
  → ResourceQuota enforces namespace-wide aggregate
  → Both together: sensible defaults + hard ceiling

Edge Case 3 — LimitRange default higher than ResourceQuota hard limit:
  LimitRange: default.memory = 2Gi
  ResourceQuota: limits.memory = 4Gi (namespace total)
  
  What happens when you create 3 pods:
  Pod 1: gets default limit 2Gi (no explicit limit set) → quota used: 2Gi
  Pod 2: gets default limit 2Gi → quota used: 4Gi (at hard limit)
  Pod 3: FAILS — quota exceeded (2Gi would put it over 4Gi total)
  
  This is a misconfiguration: default per-pod is too high for the namespace quota
  Fix: lower LimitRange default to allow reasonable number of pods

scopeSelector in ResourceQuota:
  → Applies quota only to pods matching a specific QoS class or scope
  → BestEffort: no requests or limits set
  → NotBestEffort: has at least one request or limit
  
  # Quota only for BestEffort pods (limit their count):
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: besteffort-quota
  spec:
    hard:
      pods: "10"    # max 10 BestEffort pods in this namespace
    scopeSelector:
      matchExpressions:
      - operator: In
        scopeName: BestEffort
  
  # Quota only for Burstable/Guaranteed pods:
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: notbesteffort-quota
  spec:
    hard:
      pods: "50"
      requests.cpu: "20"
      requests.memory: "40Gi"
      limits.cpu: "40"
      limits.memory: "80Gi"
    scopeSelector:
      matchExpressions:
      - operator: NotIn
        scopeName: BestEffort
  
  Use case: allow batch jobs (BestEffort) but cap them separately
  from production services (Burstable/Guaranteed)
```

> 💡 **Interview tip:** The **ResourceQuota without LimitRange** edge case is a production trap. Teams add a ResourceQuota to a namespace to limit total memory, then wonder why all new pods fail with "must specify limits.memory." The reason: ResourceQuota for limits.memory requires every container to have an explicit limit — otherwise Kubernetes can't verify the quota isn't exceeded. The fix is always a LimitRange with sensible defaults, which ensures every container has a limit even if the developer didn't set one. This is the canonical reason LimitRange and ResourceQuota are almost always deployed together.

---

### Q288 — ArgoCD | Conceptual | Advanced

> Explain ArgoCD resource health assessment. Write a custom Lua health check for a DatabaseCluster CRD.

#### Key Points to Cover:
```
ArgoCD Health Status Values:
  Healthy:     resource is in desired state, fully operational
  Progressing: resource is changing/updating (transitional state)
  Degraded:    resource is not meeting desired state (partially failed)
  Missing:     resource doesn't exist in the cluster (not yet deployed)
  Unknown:     health cannot be determined (no health check defined)
  Suspended:   intentionally paused (CronJob suspended, etc.)

Built-in health checks (ArgoCD knows these natively):
  Deployment:  Healthy when readyReplicas == replicas
  StatefulSet: Healthy when readyReplicas == replicas
  DaemonSet:   Healthy when desiredNumberScheduled == numberReady
  Job:         Healthy when complete, Degraded when failed
  PVC:         Healthy when Bound, Progressing when Pending
  Service:     Healthy when has endpoints (ClusterIP/type)
  Ingress:     Healthy when has load balancer IP
  CertManager: Healthy when certificate is Ready

Custom health checks via Lua:
  → Written in Lua (sandboxed scripting language)
  → Access: obj (the full resource as a table)
  → Must return: hs.status and hs.message
  → Configured in argocd-cm ConfigMap

Custom health check for DatabaseCluster CRD:
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: argocd-cm
    namespace: argocd
  data:
    resource.customizations.health.mycompany.io_DatabaseCluster: |
      hs = {}
      
      -- Check if status exists
      if obj.status == nil then
        hs.status = "Progressing"
        hs.message = "Status not yet available"
        return hs
      end
      
      -- Check readyReplicas matches spec.replicas
      local desiredReplicas = obj.spec.replicas or 1
      local readyReplicas = obj.status.readyReplicas or 0
      
      -- Check phase
      if obj.status.phase == "Running" then
        if readyReplicas >= desiredReplicas then
          hs.status = "Healthy"
          hs.message = string.format(
            "DatabaseCluster is running (%d/%d replicas ready)",
            readyReplicas, desiredReplicas
          )
        else
          hs.status = "Progressing"
          hs.message = string.format(
            "Waiting for replicas (%d/%d ready)",
            readyReplicas, desiredReplicas
          )
        end
      elseif obj.status.phase == "Initializing" or obj.status.phase == "Starting" then
        hs.status = "Progressing"
        hs.message = string.format("DatabaseCluster is %s", obj.status.phase)
      elseif obj.status.phase == "Failed" then
        hs.status = "Degraded"
        hs.message = obj.status.message or "DatabaseCluster is in Failed state"
      elseif obj.status.phase == "Updating" then
        hs.status = "Progressing"
        hs.message = "DatabaseCluster is updating"
      else
        hs.status = "Unknown"
        hs.message = string.format("Unknown phase: %s", obj.status.phase or "nil")
      end
      
      return hs

  # Testing custom health check locally:
  argocd admin settings resource-overrides health <path-to-resource.yaml>

  # Health status affects ArgoCD behavior:
  # Healthy → sync complete (if spec matches)
  # Progressing → ArgoCD waits (won't mark sync as complete yet)
  # Degraded → sync failed (ArgoCD shows error in UI)
  # Using health with sync waves:
  # Wave 1 resources must be Healthy before Wave 2 syncs
```

> 💡 **Interview tip:** Custom health checks are what make ArgoCD useful for operators who deploy CRDs. Without them, ArgoCD marks every CRD as "Healthy" as soon as it's applied — even if the controller hasn't finished provisioning. This is misleading and breaks sync wave ordering. The Lua script has access to the full resource object — use `obj.status.phase`, `obj.spec.replicas`, `obj.status.conditions` etc. The most important pattern: **check that readyReplicas equals spec.replicas** — this ensures the health check reflects actual pod availability, not just whether the controller accepted the resource.

---

### Q289 — Linux / Bash | Scenario-Based | Advanced

> Write a Bash certificate expiry monitor: check SSL for list of domains, Slack alerts at 30/7 days, sorted report, skip unreachable domains.

#### Key Points to Cover:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Configuration
DOMAINS_FILE="${1:-domains.txt}"
SLACK_WEBHOOK="${SLACK_WEBHOOK_URL:-}"
WARN_DAYS=30
CRITICAL_DAYS=7
TIMEOUT=10
REPORT_FILE="/tmp/cert-report-$(date +%Y%m%d).txt"

# Colors
RED='\033[0;31m'
YELLOW='\033[1;33m'
GREEN='\033[0;32m'
NC='\033[0m'

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

send_slack() {
  local color="$1"
  local title="$2"
  local message="$3"
  [[ -z "${SLACK_WEBHOOK}" ]] && { echo "[ALERT] $title: $message"; return; }
  curl -sS -X POST "${SLACK_WEBHOOK}" \
    -H 'Content-Type: application/json' \
    -d "{
      \"attachments\": [{
        \"color\": \"${color}\",
        \"title\": \"${title}\",
        \"text\": \"${message}\"
      }]
    }" > /dev/null
}

# Check a single domain, return days until expiry
# Arguments: domain [port]
check_cert() {
  local domain="$1"
  local port="${2:-443}"
  local days_left

  # Get certificate expiry date
  local expiry_date
  expiry_date=$(echo | timeout "${TIMEOUT}" openssl s_client \
    -servername "${domain}" \
    -connect "${domain}:${port}" \
    2>/dev/null | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)

  if [[ -z "${expiry_date}" ]]; then
    echo "UNREACHABLE"
    return
  fi

  # Calculate days remaining
  local expiry_epoch
  expiry_epoch=$(date -d "${expiry_date}" +%s 2>/dev/null || \
    date -j -f "%b %d %T %Y %Z" "${expiry_date}" +%s 2>/dev/null)
  local now_epoch
  now_epoch=$(date +%s)
  days_left=$(( (expiry_epoch - now_epoch) / 86400 ))

  echo "${days_left}"
}

# Collect results
declare -A RESULTS
declare -A STATUSES

log "Checking certificates from: ${DOMAINS_FILE}"

while IFS= read -r line || [[ -n "$line" ]]; do
  # Skip empty lines and comments
  [[ -z "$line" || "$line" =~ ^# ]] && continue

  # Parse domain and optional port (format: domain or domain:port)
  domain="${line%%:*}"
  port="${line##*:}"
  [[ "$port" == "$domain" ]] && port="443"

  log "Checking: ${domain}:${port}"

  days=$(check_cert "${domain}" "${port}")

  if [[ "${days}" == "UNREACHABLE" ]]; then
    STATUSES["${domain}"]="UNREACHABLE"
    RESULTS["${domain}"]="9999"  # sort last
    log "  SKIP: ${domain} is unreachable"
    continue
  fi

  RESULTS["${domain}"]="${days}"

  if [[ "${days}" -le "${CRITICAL_DAYS}" ]]; then
    STATUSES["${domain}"]="CRITICAL"
    send_slack "danger" \
      "🚨 CRITICAL: Certificate Expiring Very Soon" \
      "${domain} expires in *${days} days* (${expiry_date})"
  elif [[ "${days}" -le "${WARN_DAYS}" ]]; then
    STATUSES["${domain}"]="WARNING"
    send_slack "warning" \
      "⚠️ WARNING: Certificate Expiring Soon" \
      "${domain} expires in *${days} days*"
  else
    STATUSES["${domain}"]="OK"
  fi

done < "${DOMAINS_FILE}"

# Generate sorted report
log "Generating report: ${REPORT_FILE}"

{
  echo "========================================"
  echo "  SSL Certificate Expiry Report"
  echo "  Generated: $(date '+%Y-%m-%d %H:%M:%S')"
  echo "========================================"
  printf "%-40s %10s %12s\n" "DOMAIN" "DAYS LEFT" "STATUS"
  echo "----------------------------------------"

  # Sort by days remaining (ascending)
  for domain in $(for k in "${!RESULTS[@]}"; do
    echo "${RESULTS[$k]} $k"
  done | sort -n | awk '{print $2}'); do
    days="${RESULTS[$domain]}"
    status="${STATUSES[$domain]}"

    if [[ "${days}" == "9999" ]]; then
      printf "%-40s %10s %12s\n" "${domain}" "N/A" "UNREACHABLE"
    elif [[ "${days}" -le "${CRITICAL_DAYS}" ]]; then
      printf "%-40s %10s %12s\n" "${domain}" "${days}d" "⚠️ CRITICAL"
    elif [[ "${days}" -le "${WARN_DAYS}" ]]; then
      printf "%-40s %10s %12s\n" "${domain}" "${days}d" "⚠️  WARNING"
    else
      printf "%-40s %10s %12s\n" "${domain}" "${days}d" "✅ OK"
    fi
  done
  echo "========================================"
} | tee "${REPORT_FILE}"

# Summary Slack notification
total=${#RESULTS[@]}
critical=$(for s in "${STATUSES[@]}"; do echo "$s"; done | grep -c CRITICAL || true)
warning=$(for s in "${STATUSES[@]}"; do echo "$s"; done | grep -c WARNING || true)

if [[ "${critical}" -gt 0 || "${warning}" -gt 0 ]]; then
  send_slack "warning" \
    "📊 Certificate Expiry Summary" \
    "Checked ${total} domains: *${critical} CRITICAL*, *${warning} WARNING*. See full report."
fi

log "Certificate check complete. Report: ${REPORT_FILE}"
```

> 💡 **Interview tip:** The `timeout` command wrapping `openssl s_client` is critical for production scripts — without it, an unreachable host causes `openssl` to hang until TCP timeout (~2 minutes), blocking the entire script. With `timeout 10`, it fails fast and moves on. Also highlight the `sort -n` by days-remaining — the report should show the most urgent certificates first. The script handles custom ports (format: `domain:port`) which is needed for non-standard HTTPS deployments (internal APIs, Kubernetes ingresses).

---

### Q290 — AWS | Scenario-Based | Advanced

> Design AWS Lake Formation data lake: Glue Catalog, column-level security by team, Athena query optimization, date partitioning.

#### Key Points to Cover:
```
Architecture Overview:
  Raw data → S3 (raw zone) → Glue Crawler → Glue Data Catalog
  → Lake Formation (access control) → Athena (query)
  → Team A (can see all columns)
  → Team B (PII columns masked/restricted)

S3 Structure (zone-based):
  s3://company-datalake/
    raw/                        # unprocessed data
      sales/year=2024/month=01/day=01/
    processed/                  # cleaned, partitioned
      sales/
        year=2024/month=01/day=01/sales.parquet
    curated/                    # aggregated for analysis

Step 1 — Glue Crawler + Data Catalog:
  # Crawler discovers schema and partitions automatically:
  aws glue create-crawler \
    --name sales-crawler \
    --role arn:aws:iam::ACCOUNT:role/GlueRole \
    --database-name sales_db \
    --targets '{"S3Targets": [{"Path": "s3://company-datalake/processed/sales/"}]}' \
    --schedule '{"ScheduleExpression": "cron(0 * * * ? *)"}'  # hourly
  
  # Hive-style partitioning: Glue auto-discovers
  # year=2024/month=01/day=01 → partition columns: year, month, day
  # Athena uses partitions to skip scanning irrelevant data

Step 2 — Lake Formation permissions setup:
  # Register S3 location with Lake Formation:
  aws lakeformation register-resource \
    --resource-arn arn:aws:s3:::company-datalake
  
  # Grant Data Engineers: full access:
  aws lakeformation grant-permissions \
    --principal DataLakePrincipalIdentifier=arn:aws:iam::ACCOUNT:role/DataEngineers \
    --permissions SELECT DESCRIBE \
    --resource '{"Table": {"DatabaseName": "sales_db", "Name": "sales"}}'

Step 3 — Column-level security (restrict PII for Team B):
  # Lake Formation column-level permissions:
  # Allow Team B to see only non-PII columns:
  
  aws lakeformation grant-permissions \
    --principal DataLakePrincipalIdentifier=arn:aws:iam::ACCOUNT:role/TeamB \
    --permissions SELECT \
    --resource '{
      "TableWithColumns": {
        "DatabaseName": "sales_db",
        "Name": "customers",
        "ColumnNames": ["customer_id", "purchase_amount", "product_id", "region"],
        "ColumnWildcard": null
      }
    }'
  # Team B CANNOT SELECT: email, phone, credit_card, address (PII columns)
  
  # Grant Team A full columns:
  aws lakeformation grant-permissions \
    --principal DataLakePrincipalIdentifier=arn:aws:iam::ACCOUNT:role/TeamA \
    --permissions SELECT \
    --resource '{
      "TableWithColumns": {
        "DatabaseName": "sales_db",
        "Name": "customers",
        "ColumnWildcard": {}   # all columns
      }
    }'

Step 4 — Athena query optimization:
  # Create Athena workgroup with limits:
  aws athena create-work-group \
    --name production \
    --configuration '{
      "ResultConfiguration": {
        "OutputLocation": "s3://company-datalake/athena-results/"
      },
      "EnforceWorkGroupConfiguration": true,
      "PublishCloudWatchMetricsEnabled": true,
      "BytesScannedCutoffPerQuery": 10737418240  # 10GB max scan
    }'
  
  # Partition projection (faster than Glue Catalog lookups):
  # Athena table property:
  "projection.enabled" = "true"
  "projection.year.type" = "integer"
  "projection.year.range" = "2020,2030"
  "projection.month.type" = "integer"
  "projection.month.range" = "1,12"
  "projection.month.digits" = "2"
  
  # Query pattern — always include partition filter:
  SELECT customer_id, SUM(amount) 
  FROM sales 
  WHERE year = 2024 AND month = 01  -- uses partition pruning
  GROUP BY customer_id
  
  # Use Parquet format (columnar = only reads needed columns)
  # Compress with Snappy or ZSTD
  # Parquet + partitioning can reduce scan by 99% vs CSV

  # Glue Data Quality for validation before queries
```

> 💡 **Interview tip:** Lake Formation vs S3 bucket policies: **S3 policies control who can access S3 objects** (coarse-grained), **Lake Formation controls who can access specific columns and rows** (fine-grained). Before Lake Formation, column-level security required either Athena views (cumbersome) or separate S3 paths per team. Lake Formation's column-level grants work transparently — Team B runs normal Athena queries and simply can't see PII columns. Also mention: **Athena cost is proportional to bytes scanned** — partitioning and columnar formats (Parquet) directly reduce your bill. A query scanning 1TB of CSVs vs 10GB of partitioned Parquet = 100x cost difference.

---

### Q291 — Kubernetes | Troubleshooting | Advanced

> kube-scheduler logs "0/10 nodes: 10 had taint not-ready" but all nodes show Ready in kubectl. What causes this timing/race condition?

#### Key Points to Cover:
```
What's happening:
  → kubectl get nodes shows: Ready (current state)
  → Scheduler logs show: not-ready taint (past state or stale view)
  → Apparent contradiction: how can nodes be Ready but scheduler sees them as tainted?

Understanding the not-ready taint:
  → When a node goes NotReady: taint added: node.kubernetes.io/not-ready:NoExecute
  → When node returns to Ready: taint removed
  → Pod being scheduled RIGHT when node is transitioning:
    scheduler sees stale cache with the taint

The Informer Cache Lag:
  → kube-scheduler uses an informer cache (not direct etcd reads)
  → Cache is updated asynchronously via watch events from kube-apiserver
  → Cache update lag: typically milliseconds, but can be up to a few seconds
  → During lag: scheduler cache has old (tainted) node state
  → By the time you run kubectl get nodes: cache has updated

Race condition scenario:
  T+0:   Node restarts (kubelet restarts)
  T+1s:  Node condition: NotReady (kubelet hasn't sent first heartbeat)
  T+2s:  Node controller adds taint: node.kubernetes.io/not-ready:NoExecute
  T+5s:  kubelet sends heartbeat, Node becomes Ready again
  T+5.1s: Taint removed from Node
  T+5.2s: Taint removal event sent to kube-apiserver
  T+5.3s: Pod creation request arrives at scheduler
           Scheduler cache: still has taint (event not yet processed)
           Scheduler: "can't schedule — node has not-ready taint"
  T+5.5s: Scheduler cache processes taint removal event
  T+5.6s: Pod retried → now schedules successfully

Other causes of the same message:
  1. Node conditions not fully propagated yet:
     → Node just joined cluster (kubelet started, heartbeat sent, but
       node condition not yet processed by node lifecycle controller)

  2. Node taints manually added:
     kubectl describe node <node> | grep -i taint
     # Check if there's a not-ready taint still present

  3. Node lifecycle controller configuration:
     --node-monitor-grace-period: default 40s (time before marking not-ready)
     → If a node had a brief blip, it took 40s to recover the taint

  4. Cluster Autoscaler newly added node:
     → Node joins, kubelet starts, readiness takes 30-60s
     → During startup: legitimately has not-ready taint
     → Pod scheduling fails until node is fully ready

Diagnosis:
  # Check if nodes actually have the taint now:
  kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, taints: .spec.taints}'
  
  # Check node events:
  kubectl describe node <node> | grep -A20 "Events:"
  
  # Check scheduler logs with timestamp:
  kubectl logs -n kube-system kube-scheduler-master -f | grep "not-ready"
  # Compare timestamps with kubectl get nodes output
  
  # If persistent (not just transient):
  # → Node has a real problem (disk full, kubelet crash, etc.)
  # → taint is legitimately there, not a race condition

Fix:
  → Transient (race condition): pods will retry and succeed shortly
  → Increase scheduler retry backoff if needed
  → If persistent: investigate why node is cycling between ready/not-ready
  → Set node monitor grace period appropriate to your environment
```

> 💡 **Interview tip:** This question tests deep understanding of Kubernetes **controller architecture** and **eventual consistency**. The key insight: kubectl and the scheduler both read from the same API server, but through different paths. kubectl makes a direct API call (real-time). The scheduler uses an **informer with a local cache** that processes watch events asynchronously. During the lag between a state change and cache update, they see different views of the world. This is by design (performance) but creates these confusing race conditions. The practical takeaway: transient "can't schedule" messages during node restarts are expected — the concern is if they persist, indicating a real node health issue.

---

### Q292 — Grafana | Conceptual | Advanced

> Explain Grafana Transformations — joining queries, wide-to-long format, filtering, renaming, calculated fields.

#### Key Points to Cover:
```
What are Grafana Transformations:
  → Post-processing steps applied to query results BEFORE visualization
  → Run client-side in the browser (not server-side)
  → Can be chained: output of one transformation = input of next
  → Available in Edit Panel → Transform tab
  → Work on the data "frame" (table of values)

Common Transformations:

  1. Merge (Join two queries into one table):
     Problem: Query A returns CPU per pod, Query B returns Memory per pod
     → Two separate tables with pod label
     Transform: Merge
     → Combines into: pod | CPU | Memory
     
     Config: Join by field "pod" (the common key)
     Result: One row per pod with all metrics

  2. Convert field type:
     → Change string timestamps to time fields
     → Change numeric strings to numbers

  3. Filter data by values:
     Problem: Show only rows where CPU > 80%
     Transform: Filter data by values
     → Field: cpu_usage
     → Condition: greater than 0.8
     → Result: table shows only overloaded pods

  4. Organize fields (rename columns):
     Problem: Prometheus returns "container_cpu_usage_seconds_total"
     Transform: Organize fields
     → Rename: container_cpu_usage_seconds_total → CPU Usage
     → Hide: __name__, job, instance (internal Prometheus labels)
     → Result: clean human-readable column names

  5. Reduce (collapse time series to a single value):
     → Reduce time series to: last, mean, max, min, sum
     → Use: stat panels showing "current memory usage"

  6. Calculate field (derived column):
     Problem: have requests and errors, want error rate %
     Transform: Add field from calculation
     → Mode: Binary operation
     → Field A: errors
     → Operation: divide by
     → Field B: requests
     → Alias: Error Rate
     → Result: new column with error_rate = errors/requests

  7. Sort by:
     → Sort rows by a specific column (ascending or descending)
     → Use: show highest CPU consumers first

  8. Limit:
     → Show only first N rows
     → Use with Sort: Top 10 by CPU

  9. Group by:
     → Aggregate rows by a field value
     → Similar to SQL GROUP BY

Practical pipeline (Top 10 pods by CPU):
  Query → [Reduce: last value] → [Sort: CPU desc] → [Limit: 10] → [Organize: rename]
  → Shows: clean top-10 table with human readable names

Wide vs Long format:
  Wide (default Prometheus format):
    time | pod-a_cpu | pod-b_cpu | pod-c_cpu
    14:00 | 0.5       | 0.8       | 0.3

  Long format (needed for some visualizations):
    time | pod   | cpu
    14:00 | pod-a | 0.5
    14:00 | pod-b | 0.8
    14:00 | pod-c | 0.3

  Transform: Convert wide to long
  Use: when visualization expects one row per data point
```

> 💡 **Interview tip:** Transformations are the difference between "a dashboard that shows data" and "a dashboard that shows insights." The **Merge transformation** is the most powerful and least known — it solves the problem of needing data from two different queries in one table. For example, joining CPU from container metrics with the deployment owner label from kube_deployment_labels requires merging two queries on the pod label. Without Merge, you'd need to write a single complex PromQL query or show two separate panels. Also: transformations run client-side, so for large datasets they can be slow — use Prometheus recording rules to pre-aggregate heavy queries instead.

---

### Q293 — Terraform | Troubleshooting | Advanced

> Fix: (1) checksum mismatch in dependency lock file, (2) cannot connect to registry.terraform.io.

#### Key Points to Cover:
```
Error 1 — Checksum mismatch in lock file:
  Full error:
  "the local package for registry.terraform.io/hashicorp/aws doesn't match
   any of the checksums previously recorded in the dependency lock file"

  What caused this:
    → .terraform.lock.hcl records checksums for downloaded provider binaries
    → Mismatch happens when:
      a) Provider binary was manually replaced/corrupted
      b) Different OS/architecture from when lock was created
         (Mac developer locked it, Linux CI is using it)
      c) Provider version updated but lock file wasn't regenerated

  Platform checksum issue (most common in teams):
    # Lock file contains h1: checksums for ONE platform
    # CI runner needs Linux checksum, lock has only Darwin checksum
    
    Error: "the local package doesn't match any checksums"
    Reason: lock file has darwin_amd64 hash, CI needs linux_amd64 hash

  Fix A — Regenerate lock file with multiple platforms:
    terraform providers lock \
      -platform=linux_amd64 \
      -platform=linux_arm64 \
      -platform=darwin_amd64 \
      -platform=windows_amd64
    # Updates .terraform.lock.hcl with hashes for all platforms
    # Commit the updated lock file

  Fix B — Quick fix (re-init, accepts new checksums):
    terraform init -upgrade
    # Re-downloads provider, updates lock file
    # WARNING: only do this if you trust the provider version hasn't been tampered with

  Fix C — Delete and regenerate lock file:
    rm .terraform.lock.hcl
    terraform init
    # Regenerates lock file from scratch
    # Then: terraform providers lock for multi-platform hashes

Error 2 — Cannot connect to registry.terraform.io:
  Full error:
  "could not connect to registry.terraform.io: failed to request discovery document"

  Causes and fixes:

  A) No internet access (air-gapped environment):
     Fix: Use a private mirror (Nexus, Artifactory, custom registry)
     terraform {
       required_providers {
         aws = {
           source  = "my-nexus.company.com/hashicorp/aws"
           version = "~> 5.0"
         }
       }
     }
     # Or: use filesystem mirror
     # terraform init -plugin-dir=/path/to/local/providers

  B) Corporate proxy not configured:
     export HTTPS_PROXY=http://proxy.company.com:8080
     export NO_PROXY=".internal,.local,169.254.169.254"
     terraform init

  C) TLS inspection proxy replacing certificates:
     export SSL_CERT_FILE=/etc/ssl/certs/company-ca.crt
     terraform init

  D) DNS resolution failure:
     dig registry.terraform.io   # test DNS
     # If fails: check /etc/resolv.conf, VPN connected?

  E) Temporary registry outage:
     # Check: https://status.hashicorp.com
     # Wait and retry

  F) Firewall blocking egress to registry.terraform.io:443:
     curl -v https://registry.terraform.io/v1/providers/hashicorp/aws/versions
     # If timeout: firewall issue
     # Fix: open egress on port 443 to registry.terraform.io
     # Or: use private registry mirror

  G) CI/CD without internet (common):
     # Best practice: pre-install providers in Docker image
     # Or: use .terraform.d/plugins/ directory
     # Or: use private Terraform registry
```

> 💡 **Interview tip:** The **multi-platform lock file** issue is the most common checksum mismatch in real teams — a developer runs `terraform init` on their Mac, the lock file gets committed with only macOS checksums, then the Linux CI runner fails because the Linux binary has a different hash. The fix is `terraform providers lock -platform=linux_amd64 -platform=darwin_amd64` which populates the lock file with hashes for all platforms. Always commit the `.terraform.lock.hcl` file and use `providers lock` with all needed platforms as part of your setup.

---

### Q294 — Python | Scenario-Based | Advanced

> Write a Kubernetes cost analyzer: list deployments, calculate CPU/memory cost, identify over-provisioned pods, sort by estimated cost.

#### Key Points to Cover:
```python
#!/usr/bin/env python3
"""Kubernetes cost analyzer — identifies over-provisioned deployments."""

from kubernetes import client, config
from kubernetes.client.rest import ApiException
import sys

# Cost configuration ($/hour per resource unit)
COST_CONFIG = {
    "cpu_cost_per_core_per_month": 30.0,    # ~$0.04/core/hour
    "memory_cost_per_gb_per_month": 4.0,    # ~$0.005/GB/hour
    "over_provision_threshold": 0.5,        # 50% over-provisioned
}

def get_resource_value(quantity_str: str | None, resource_type: str) -> float:
    """Convert Kubernetes resource quantity string to float (cores or GB)."""
    if not quantity_str:
        return 0.0
    
    q = quantity_str.strip()
    
    if resource_type == "cpu":
        if q.endswith("m"):
            return float(q[:-1]) / 1000  # millicores to cores
        return float(q)
    
    elif resource_type == "memory":
        units = {
            "Ki": 1/1024/1024,
            "Mi": 1/1024,
            "Gi": 1.0,
            "Ti": 1024.0,
            "k":  1/1024/1024,
            "M":  1/1024,
            "G":  1.0,
        }
        for suffix, multiplier in units.items():
            if q.endswith(suffix):
                return float(q[:-len(suffix)]) * multiplier
        return float(q) / (1024**3)  # bytes to GB

def get_actual_usage(metrics_api: client.CustomObjectsApi,
                     namespace: str, pod_prefix: str) -> tuple[float, float]:
    """Get actual CPU and memory usage from metrics API."""
    try:
        pod_metrics = metrics_api.list_namespaced_custom_object(
            group="metrics.k8s.io",
            version="v1beta1",
            namespace=namespace,
            plural="pods"
        )
        
        total_cpu = 0.0
        total_mem = 0.0
        pod_count = 0
        
        for pod in pod_metrics.get("items", []):
            if not pod["metadata"]["name"].startswith(pod_prefix):
                continue
            
            for container in pod.get("containers", []):
                total_cpu += get_resource_value(container["usage"]["cpu"], "cpu")
                total_mem += get_resource_value(container["usage"]["memory"], "memory")
            pod_count += 1
        
        if pod_count == 0:
            return 0.0, 0.0
        
        return total_cpu / pod_count, total_mem / pod_count  # per-pod average
    
    except ApiException:
        return 0.0, 0.0  # metrics not available

def analyze_deployments():
    """Main analyzer function."""
    try:
        config.load_incluster_config()  # when running in-cluster
    except config.ConfigException:
        config.load_kube_config()       # when running locally
    
    apps_v1 = client.AppsV1Api()
    metrics_api = client.CustomObjectsApi()
    
    results = []
    
    # List all deployments across all namespaces
    deployments = apps_v1.list_deployment_for_all_namespaces()
    
    for deploy in deployments.items:
        name = deploy.metadata.name
        namespace = deploy.metadata.namespace
        replicas = deploy.spec.replicas or 1
        
        # Aggregate requested resources across all containers
        total_cpu_req = 0.0
        total_mem_req = 0.0
        
        containers = deploy.spec.template.spec.containers
        for container in containers:
            if container.resources and container.resources.requests:
                total_cpu_req += get_resource_value(
                    container.resources.requests.get("cpu"), "cpu"
                )
                total_mem_req += get_resource_value(
                    container.resources.requests.get("memory"), "memory"
                )
        
        # Per replica cost (requests × replicas)
        total_cpu = total_cpu_req * replicas
        total_mem = total_mem_req * replicas
        
        # Estimate monthly cost
        monthly_cost = (
            total_cpu * COST_CONFIG["cpu_cost_per_core_per_month"] +
            total_mem * COST_CONFIG["memory_cost_per_gb_per_month"]
        )
        
        # Get actual usage from metrics API
        actual_cpu, actual_mem = get_actual_usage(metrics_api, namespace, name)
        
        # Calculate over-provisioning
        cpu_over = ((total_cpu_req - actual_cpu) / total_cpu_req) if total_cpu_req > 0 else 0
        mem_over = ((total_mem_req - actual_mem) / total_mem_req) if total_mem_req > 0 else 0
        is_over_provisioned = (
            cpu_over > COST_CONFIG["over_provision_threshold"] or
            mem_over > COST_CONFIG["over_provision_threshold"]
        )
        
        results.append({
            "namespace":       namespace,
            "name":            name,
            "replicas":        replicas,
            "cpu_requested":   total_cpu,
            "mem_requested_gb": total_mem,
            "cpu_actual":      actual_cpu * replicas,
            "mem_actual_gb":   actual_mem * replicas,
            "monthly_cost":    monthly_cost,
            "cpu_over_pct":    cpu_over * 100,
            "mem_over_pct":    mem_over * 100,
            "is_over_provisioned": is_over_provisioned,
        })
    
    # Sort by monthly cost (highest first)
    results.sort(key=lambda x: x["monthly_cost"], reverse=True)
    
    # Print report
    print(f"\n{'='*90}")
    print(f"{'Kubernetes Deployment Cost Analysis':^90}")
    print(f"{'='*90}")
    print(f"{'NAMESPACE':<20} {'DEPLOYMENT':<30} {'REPLICAS':>8} {'CPU':>6} {'MEM(GB)':>8} {'COST/MO':>10} {'STATUS'}")
    print(f"{'-'*90}")
    
    total_monthly = 0.0
    
    for r in results:
        status = "⚠️ OVER" if r["is_over_provisioned"] else "✅ OK"
        print(
            f"{r['namespace']:<20} {r['name']:<30} {r['replicas']:>8} "
            f"{r['cpu_requested']:>6.2f} {r['mem_requested_gb']:>8.2f} "
            f"${r['monthly_cost']:>9.2f} {status}"
        )
        
        if r["is_over_provisioned"]:
            print(f"  └─ CPU over-provisioned: {r['cpu_over_pct']:.0f}%  "
                  f"Memory over-provisioned: {r['mem_over_pct']:.0f}%")
        
        total_monthly += r["monthly_cost"]
    
    print(f"{'='*90}")
    print(f"Total estimated monthly cost: ${total_monthly:.2f}")
    
    over_count = sum(1 for r in results if r["is_over_provisioned"])
    print(f"Over-provisioned deployments: {over_count}/{len(results)}")
    
    potential_savings = sum(
        r["monthly_cost"] * max(r["cpu_over_pct"], r["mem_over_pct"]) / 100
        for r in results if r["is_over_provisioned"]
    )
    print(f"Potential monthly savings:    ${potential_savings:.2f}")
    print(f"{'='*90}\n")
    
    return results

if __name__ == "__main__":
    analyze_deployments()
```

> 💡 **Interview tip:** The **over-provisioning threshold** calculation is the key analytical insight. Comparing requested resources (what the pod CLAIMS it needs) vs actual usage (what it's REALLY using via metrics API) identifies waste. A pod requesting 4 CPUs but using 0.1 CPUs is a prime candidate for right-sizing. In production, you'd run this weekly and feed results into a ticket system. Also mention VPA (Vertical Pod Autoscaler) in recommendation mode — it does this analysis continuously and suggests right-sized requests automatically.

---

### Q295 — AWS | Conceptual | Advanced

> Explain AWS Macie, Inspector, and Security Hub. How does Security Hub aggregate findings? What is ASFF?

#### Key Points to Cover:
```
AWS Macie:
  → Scans S3 buckets for sensitive data (PII, credentials, financial data)
  → Uses ML to detect: credit cards, SSNs, passwords in files
  → Finds: public S3 buckets, unencrypted sensitive data
  → Reports: S3 bucket exposure, sensitive data location
  → Use case: compliance (GDPR, HIPAA, PCI-DSS) — "do we store PII we shouldn't?"
  → Pricing: per GB of data scanned

AWS Inspector:
  → Vulnerability assessment for EC2 instances and container images
  → Scans: OS packages, application libraries for known CVEs
  → Integrates with: ECR (scan on push), EC2 SSM agent
  → Reports: CVE details, severity (CRITICAL/HIGH/MEDIUM), fix available
  → Continuous: rescans when new CVEs published (not just on-demand)
  → Use case: "are our servers running vulnerable software?"

  Inspector vs Macie comparison:
    Macie:    What data do we have? (data classification)
    Inspector: Are our systems vulnerable? (vulnerability management)

AWS Security Hub:
  → Aggregates findings from: GuardDuty, Macie, Inspector, Config, Access Analyzer
  → Plus: third-party tools (Splunk, Palo Alto, CrowdStrike via ASFF)
  → Provides: security score, compliance checks (CIS, PCI-DSS, NIST)
  → Single pane of glass for security posture across all accounts
  → Integrates: AWS Organizations (multi-account view)

AWS Security Finding Format (ASFF):
  → Standardized JSON schema for security findings
  → Any tool can submit findings to Security Hub using ASFF
  → Every finding includes:
    - SchemaVersion, Id (unique finding ID)
    - ProductArn (which tool created it)
    - Types (classification: Software and Configuration Checks/CVE)
    - Severity (CRITICAL/HIGH/MEDIUM/LOW/INFORMATIONAL)
    - Resources (affected AWS resource ARN)
    - Remediation (how to fix it)
    - UpdatedAt, CreatedAt

  # Custom tool sending finding to Security Hub:
  import boto3
  securityhub = boto3.client('securityhub')
  
  securityhub.batch_import_findings(Findings=[{
    "SchemaVersion": "2018-10-08",
    "Id": "finding-unique-id-123",
    "ProductArn": "arn:aws:securityhub:us-east-1:ACCOUNT:product/ACCOUNT/default",
    "GeneratorId": "my-custom-scanner",
    "AwsAccountId": "123456789012",
    "Types": ["Software and Configuration Checks/Vulnerabilities/CVE"],
    "CreatedAt": "2024-01-15T10:00:00Z",
    "UpdatedAt": "2024-01-15T10:00:00Z",
    "Severity": {"Label": "HIGH"},
    "Title": "SSH key not rotated in 90 days",
    "Description": "IAM user alice@company.com has an SSH key 91 days old",
    "Resources": [{
      "Type": "AwsIamUser",
      "Id": "arn:aws:iam::123456789012:user/alice"
    }],
    "Remediation": {
      "Recommendation": {
        "Text": "Rotate the SSH key and update key rotation policy"
      }
    }
  }])

Security Hub workflow:
  Findings → Security Hub → EventBridge rules → Lambda → Jira/PagerDuty/Slack
  → Automate: create tickets for CRITICAL findings
  → Automate: block IAM user if GuardDuty detects compromise
```

> 💡 **Interview tip:** The ASFF is what makes Security Hub genuinely useful as a platform — it's not just an aggregator of AWS tools, it's a **universal security event bus** that any tool can send findings to. If you have a custom compliance checker, a third-party SAST tool, or an internal policy engine, you can send their findings to Security Hub using ASFF and get a unified view. The key business value: instead of security team checking 6 different dashboards (GuardDuty, Macie, Inspector, Config, your tool, partner tool), they check one. Connect findings to EventBridge to automate responses.

---

### Q296 — Kubernetes | Scenario-Based | Advanced

> Configure PodDisruptionBudget for Cassandra with 2/3 availability, application-aware drain blocking via PreStop hook.

#### Key Points to Cover:
```
PodDisruptionBudget for Cassandra:
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata:
    name: cassandra-pdb
    namespace: production
  spec:
    # Option A: minAvailable (minimum pods that must be Running):
    minAvailable: 2   # out of 3 = can drain 1 at a time
    
    # Option B: maxUnavailable (max pods that can be down at once):
    # maxUnavailable: 1  # same effect, different expression
    
    selector:
      matchLabels:
        app: cassandra

  # Effect on cluster upgrade (kubectl drain):
  # Node drain evicts pods one at a time
  # If evicting Cassandra pod would violate PDB (drop below 2): drain BLOCKS
  # Drain waits until evicted pod is rescheduled and ready elsewhere
  # Only then does it continue to next pod

Verifying PDB is working:
  kubectl get pdb cassandra-pdb
  # Shows: ALLOWED-DISRUPTIONS column
  # If 3/3 running: can disrupt 1 (ALLOWED-DISRUPTIONS = 1)
  # If 2/3 running: can disrupt 0 (ALLOWED-DISRUPTIONS = 0)

Application-aware drain blocking with PreStop hook:
  Problem: PDB only protects against VOLUNTARY disruptions
           Can't know if Cassandra data replication is complete
  Solution: Custom preStop hook that waits for data safety

  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: cassandra
  spec:
    template:
      spec:
        terminationGracePeriodSeconds: 600  # 10 minutes max
        containers:
        - name: cassandra
          image: cassandra:4.1
          lifecycle:
            preStop:
              exec:
                command:
                - /bin/bash
                - -c
                - |
                  #!/bin/bash
                  # Application-aware drain blocking script
                  
                  echo "PreStop: Starting graceful Cassandra drain..."
                  
                  # Step 1: Mark node as leaving (stop accepting new writes):
                  nodetool drain
                  echo "PreStop: nodetool drain complete"
                  
                  # Step 2: Wait for pending repairs/compactions to complete:
                  MAX_WAIT=300  # 5 minutes max
                  ELAPSED=0
                  
                  while true; do
                    # Check if compaction queue is empty:
                    COMPACTIONS=$(nodetool compactionstats 2>/dev/null | grep -c "CompactionTask" || echo "0")
                    
                    # Check if repairs are in progress:
                    REPAIRS=$(nodetool netstats 2>/dev/null | grep "Repair" | wc -l || echo "0")
                    
                    if [[ "$COMPACTIONS" -eq 0 && "$REPAIRS" -eq 0 ]]; then
                      echo "PreStop: Queue empty — safe to terminate"
                      break
                    fi
                    
                    if [[ $ELAPSED -ge $MAX_WAIT ]]; then
                      echo "PreStop: Timeout after ${MAX_WAIT}s — proceeding with termination"
                      break
                    fi
                    
                    echo "PreStop: Waiting... compactions=$COMPACTIONS repairs=$REPAIRS (${ELAPSED}s)"
                    sleep 10
                    ELAPSED=$((ELAPSED + 10))
                  done
                  
                  # Step 3: Final decommission signal:
                  # Note: full decommission takes too long — just drain
                  # Data replication continues on other nodes
                  echo "PreStop: Complete"

  # PDB + PreStop together:
  # 1. Drain requested for node
  # 2. Kubernetes sends SIGTERM to Cassandra pod
  # 3. preStop hook runs (waits for safety)
  # 4. After preStop completes: SIGTERM becomes effective
  # 5. Cassandra process receives SIGTERM, shuts down
  # 6. PDB was already checking: remaining pods meet minAvailable
  # 7. If PDB check would fail: drain blocked before step 2

  # Combine with topologySpreadConstraints for distribution:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: cassandra
  # Ensures: one Cassandra pod per AZ
  # PDB + AZ spread = no single AZ failure takes Cassandra down
```

> 💡 **Interview tip:** PDB and PreStop hook solve **different but complementary problems**. PDB is a Kubernetes-level guarantee: "never voluntarily disrupt more than N pods at once." The PreStop hook is an application-level guarantee: "when this specific pod is being terminated, wait until it's safe." The key limitation of PDB alone: Kubernetes doesn't know if Cassandra is in the middle of a crucial data replication operation. The PreStop hook bridges this gap — the application itself knows when it's safe to terminate. The `terminationGracePeriodSeconds: 600` gives the hook enough time to complete; without it, Kubernetes kills the container 30 seconds after SIGTERM regardless.

---

### Q297 — ELK Stack | Troubleshooting | Advanced

> Logstash pipeline.workers threads all stuck, throughput zero, CPU 0%, input queue growing. Diagnose and recover without losing events.

#### Key Points to Cover:
```
What happened:
  → All worker threads blocked (waiting for I/O or lock)
  → CPU 0%: threads are not executing, just waiting
  → Input queue growing: events arriving but nothing processing them
  → This is a thread deadlock or blocking I/O situation

Diagnosis tools:

Step 1 — Check Logstash Node Stats API:
  curl http://logstash:9600/_node/stats/pipelines
  # Look for: pipeline.events.duration_in_millis vs events.in
  # If duration is huge but events.out is near zero: workers are stalling

  # Check worker thread state:
  curl http://logstash:9600/_node/stats/jvm
  # Look for: threads section — how many are blocked vs waiting vs runnable

Step 2 — Thread dump (identify where threads are stuck):
  # Get Logstash PID:
  ps aux | grep logstash
  
  # Java thread dump (shows stack trace of every thread):
  kill -3 <logstash-pid>         # SIGQUIT → dumps to stdout/logfile
  # OR:
  jstack <logstash-pid> > /tmp/thread-dump.txt
  
  # Look for: "BLOCKED" or "WAITING" thread states
  # Common stack traces indicating blocking:
  #   at java.net.Socket.connect (TCP connection waiting)
  #   at java.io.OutputStreamWriter.write (waiting on output)
  #   at org.jruby.RubyThread (Logstash/JRuby thread)

Step 3 — Identify which stage is blocking:

  Input blocking (Beats/Kafka):
    → Thread stuck at: org.logstash.beats.BeatsChannelHandler
    → Cause: too many pending connections, backpressure from filter/output
    → The output is slow → queue fills → input slows → pressure

  Filter blocking:
    → Thread stuck at: grok pattern with catastrophic backtracking
    → Cause: complex regex on long log lines → exponential backtrack time
    → Test: logstash -e 'filter { grok { match => { "message" => "pattern" } } }'

  Output blocking (most common):
    → Thread stuck at: elasticsearch HTTP client socket write
    → Cause: Elasticsearch is overloaded or unresponsive
    → All output threads blocked waiting for ES response
    → → all filter threads blocked waiting for output queue
    → → all input threads blocked waiting for filter queue
    → Full pipeline stall

Step 4 — Check Elasticsearch health (most likely cause):
  curl http://elasticsearch:9200/_cluster/health
  # If RED: ES has primary shard issues — Logstash can't index
  curl http://elasticsearch:9200/_cat/thread_pool/write?v
  # If write queue full: ES is overwhelmed

Step 5 — Recovery WITHOUT losing events:

  A) If using Persistent Queue (recommended):
     → Events are stored on disk in Logstash PQ
     → Safe to restart Logstash: PQ replays on startup
     
     systemctl restart logstash
     # Events resume from where they left off

  B) If using in-memory queue (risky):
     → Restarting WILL lose unprocessed in-memory events
     → Option: fix the blocking output first (fix ES)
     → Then wait for workers to unblock naturally
     # Never restart with in-memory queue if you can avoid it

  C) Fix the root cause first:
     # If ES is overloaded: add bulk.actions settings to reduce load
     output {
       elasticsearch {
         bulk_max_size => 500   # reduce from default 125 (actually increase)
         # Actually REDUCE if ES is overwhelmed:
         bulk_max_size => 100
         flush_size => 50
       }
     }
     
     # If grok is catastrophically backtracking:
     filter {
       grok {
         timeout_millis => 1000  # kill grok if it takes > 1 second
         tag_on_timeout => "_groktimeout"
       }
     }

  D) Enable persistent queue (prevent future data loss):
     # logstash.yml:
     queue.type: persisted
     queue.max_bytes: 1gb
     path.queue: /var/lib/logstash/queue
```

> 💡 **Interview tip:** The **cascading backpressure** pattern is the key insight: output blocks → filter queue fills → input blocks → external queue fills. It always starts at the output. The fastest diagnosis: check if Elasticsearch is healthy FIRST (`/_cluster/health`). If ES is RED or write thread pool is full, that's almost certainly why Logstash is stalled. The permanent fix: **enable persistent queues** in Logstash — this decouples the input from the output, allowing Logstash to continue ingesting even when output is temporarily unavailable, and ensures no events are lost on restart.

---

### Q298 — Jenkins | Conceptual | Advanced

> Explain Jenkins Groovy CPS execution model. Why can't all Groovy features be used? What is @NonCPS? How to fix NotSerializableException.

#### Key Points to Cover:
```
CPS (Continuation Passing Style) — why it exists:
  Problem: Jenkins must be able to:
    1. Pause a pipeline (waiting for input, sleep)
    2. Survive Jenkins master restart mid-pipeline
    3. Resume from where it left off
  
  Solution: CPS transformation
    → All pipeline code is transformed by CPS compiler at load time
    → CPS transforms code into a state machine that can be serialized
    → Pipeline state saved to disk at every "step"
    → If Jenkins restarts: loads state from disk, continues execution

What CPS means for your code:
  → The CPS compiler transforms Groovy closures into continuations
  → ALL variables in pipeline scope must be serializable
  → NOT all Groovy objects are serializable (Java Map.Entry, Iterators, etc.)

Methods that DON'T work in CPS:
  1. Non-serializable objects:
     // FAILS: Map.Entry is not serializable
     def map = [a: 1, b: 2]
     map.each { k, v ->    // ← this iterator state can't be serialized
       echo "${k}=${v}"
     }

  2. Some Groovy built-in methods:
     def sorted = [3,1,2].sort()  // ← might fail depending on version
     
  3. Java streams / complex iterators

@NonCPS annotation:
  → Marks a method as NOT CPS-transformed
  → Method runs as normal Groovy/Java (no serialization requirement)
  → BUT: cannot call CPS methods from @NonCPS (no pipeline steps)
  → Cannot pause or resume within @NonCPS method
  
  @NonCPS
  def sortList(List input) {
    return input.sort { it.name }  // can use any Groovy features
  }
  
  // In pipeline (CPS context):
  def sorted = sortList(myList)  // call from pipeline (CPS) to @NonCPS (ok)
  echo "First: ${sorted[0].name}"  // back in CPS

NotSerializableException — common causes and fixes:
  Cause 1 — Non-serializable object in closure variable:
    // FAILS:
    def matcher = (someString =~ /pattern/)
    if (matcher) {
      // matcher is a Matcher object — not serializable
    }
    
    // FIX: Extract what you need, don't keep matcher in scope:
    @NonCPS
    def findMatch(String text, String pattern) {
      def m = text =~ pattern
      return m ? m[0] : null
    }
    def result = findMatch(someString, /pattern/)

  Cause 2 — Storing non-serializable in map:
    // FAILS:
    def config = [
      connection: new java.sql.Connection(...)  // not serializable!
    ]
    
    // FIX: Never store connections, streams, or resources as pipeline vars
    // Use them locally within @NonCPS methods

  Cause 3 — Using .collect{} with complex objects:
    // RISKY:
    def names = deployments.collect { it.metadata.name }
    
    // SAFER:
    @NonCPS
    def extractNames(List items) {
      items.collect { it.metadata.name }
    }
    def names = extractNames(deployments)

  General rules:
    → Keep pipeline variables: simple types (String, Integer, List, Map)
    → Complex logic: put in @NonCPS methods
    → regex Matcher, Java iterators, streams: ALWAYS @NonCPS
    → @NonCPS methods: can't call steps (sh, echo, input, etc.)
    → Think: CPS = "pipeline state checkpoints", @NonCPS = "pure computation"
```

> 💡 **Interview tip:** The underlying reason for CPS is **pipeline durability** — Jenkins saves the pipeline state to disk after every step so it can survive restarts. To save state, all variables must be serializable to disk. A Java Iterator or regex Matcher can't be serialized. This constraint is the root of most confusing Jenkins pipeline failures. The practical rule: if you're doing complex data manipulation (sorting, regex, string processing), put it in a `@NonCPS` method and return only serializable results back to the CPS pipeline context. The common gotcha: `@NonCPS` methods can call `@NonCPS` methods but cannot call pipeline steps (`sh`, `echo`, `input`).

---

### Q299 — Linux / Bash | Conceptual | Advanced

> Explain Linux inodes — what they store, what they don't. Inode exhaustion scenario in DevOps context. Diagnose and fix without unmounting.

#### Key Points to Cover:
```
What an inode stores:
  → Every file/directory has ONE inode (index node)
  → Inode contains:
    ✅ File type (regular file, directory, symlink, socket, etc.)
    ✅ Permissions (owner, group, mode: rwxr-xr-x)
    ✅ Owner UID and GID
    ✅ File size
    ✅ Timestamps: atime (last access), mtime (last modified), ctime (last status change)
    ✅ Link count (number of hard links)
    ✅ Pointers to data blocks (where the actual content is on disk)

What an inode does NOT store:
  ❌ Filename — stored in the DIRECTORY that contains the file
  ❌ File path — directories hold the name-to-inode mapping
  ❌ File content — stored in data blocks, pointed to by inode
  ❌ Parent directory reference

Why filenames aren't in inodes:
  → Hard links: one file can have multiple names (both names → same inode)
  → The inode is "the file", the directory entry is "the name"
  → mv: just changes directory entry, doesn't touch inode (fast!)
  → Deleting a file: removes directory entry, decrements inode link count
    → When link count = 0 AND no open file descriptors: inode is freed

Inode exhaustion:
  → Each filesystem has a fixed inode table (set at mkfs time)
  → Default ratio: ~1 inode per 4KB of space (but varies by mkfs options)
  → If you create more files than inodes: "No space left on device"
  → BUT: df -h might show 50% disk space free — confusing!
  
  # Check inode usage:
  df -i                        # shows inode usage per filesystem
  df -ih /var                  # human-readable for /var

DevOps scenario — inode exhaustion:
  Common cause: application or system creating millions of small files
  
  Real-world examples:
  → PHP sessions: /var/lib/php/sessions/sess_XXXXXXXX (one per user session)
  → Node.js/npm cache: many small files in node_modules
  → Java: many .class files in build directory
  → Kubernetes: ephemeral storage — each pod creates many small tmp files
  → Log rotation: many small rotated log files (.log.1, .log.2, etc.)
  → Ansible: creates thousands of .retry files

Diagnosis without unmounting:
  # Identify which directory has most files:
  find /var -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head -20
  # Shows: which parent directory has the most files
  
  # More targeted (find the worst offender):
  for dir in /var/*/; do
    count=$(find "$dir" -maxdepth 1 | wc -l)
    echo "$count $dir"
  done | sort -rn | head -10
  
  # Check total file count in a directory:
  find /var/lib/php/sessions -type f | wc -l

Fix without unmounting:
  # Delete old files (PHP sessions older than 1 day):
  find /var/lib/php/sessions -type f -mtime +1 -delete
  
  # Delete Node.js build artifacts:
  find /opt/app -name "node_modules" -type d -prune -exec rm -rf {} +
  
  # Delete all files in a directory quickly (faster than rm -rf for millions):
  cd /path/to/dir && find . -type f -delete

Prevention:
  # Use ramfs/tmpfs for temp files (no persistent inodes):
  mount -t tmpfs tmpfs /tmp -o size=512m
  
  # Configure PHP session cleanup:
  session.gc_maxlifetime = 1440  # in php.ini
  
  # Monitor inode usage:
  - alert: FilesystemInodeUsageHigh
    expr: node_filesystem_files_free / node_filesystem_files < 0.1
    annotations:
      summary: "Inode usage > 90% on {{ $labels.mountpoint }}"
```

> 💡 **Interview tip:** Inode exhaustion is a **classic "disk full but not full" trap**. The system returns "No space left on device" but `df -h` shows free space. The missing command: `df -i`. For a DevOps context, the most common trigger is PHP sessions, CI/CD temporary files, or any system that creates many small files. The fast cleanup: `find . -type f -delete` is significantly faster than `rm -rf` for directories with millions of files because `rm -rf` does more work per file. Also mention: you can create a filesystem with more inodes at `mkfs` time using `-N` flag — relevant when designing filesystems that will store many small files.

---

### Q300 — AWS | Scenario-Based | Advanced

> Walk through complete AWS CodePipeline setup: GitHub source, CodeBuild with tests, ECR scanning, ECS blue/green CodeDeploy, manual approval, SNS notifications. Compare to GitHub Actions.

#### Key Points to Cover:
```
Complete CodePipeline Structure:
  Stage 1: Source → Stage 2: Build+Test → Stage 3: Scan → Stage 4: Approve → Stage 5: Deploy

Step 1 — CodeStar Connection (GitHub):
  # In AWS Console: Settings → Connections → GitHub
  # Or via CLI (must complete OAuth in console):
  aws codestar-connections create-connection \
    --provider-type GitHub \
    --connection-name github-myorg

Step 2 — CodeBuild project (tests + Docker build):
  # buildspec.yml (in repo root):
  version: 0.2
  phases:
    pre_build:
      commands:
      - echo Logging in to ECR
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
    build:
      commands:
      - npm ci && npm test          # unit tests
      - docker build -t $IMAGE_URI:$COMMIT_HASH .
      - docker push $IMAGE_URI:$COMMIT_HASH
    post_build:
      commands:
      # Generate ECS task def with new image:
      - envsubst < taskdef.json.template > taskdef.json
      - cat appspec.yaml
  artifacts:
    files:
    - taskdef.json
    - appspec.yaml

Step 3 — ECR Image Scanning:
  # Enable enhanced scanning on ECR repo:
  aws ecr put-registry-scanning-configuration \
    --scan-type ENHANCED \
    --rules '[{"repositoryFilters":[{"filter":"*","filterType":"WILDCARD"}],"scanFrequency":"SCAN_ON_PUSH"}]'
  
  # CodePipeline action: check scan results before proceeding
  # Add Lambda function action that queries ECR scan results:
  # If CRITICAL vulnerabilities found → fail pipeline

Step 4 — Pipeline definition (CloudFormation):
  Type: AWS::CodePipeline::Pipeline
  Properties:
    Stages:
    - Name: Source
      Actions:
      - Name: GitHub
        ActionTypeId:
          Category: Source
          Owner: AWS
          Provider: CodeStarSourceConnection
        Configuration:
          ConnectionArn: !Ref GitHubConnection
          FullRepositoryId: myorg/myapp
          BranchName: main
    
    - Name: Build
      Actions:
      - Name: BuildAndTest
        ActionTypeId:
          Category: Build
          Owner: AWS
          Provider: CodeBuild
        InputArtifacts: [{Name: SourceArtifact}]
        OutputArtifacts: [{Name: BuildArtifact}]
    
    - Name: ManualApproval
      Actions:
      - Name: ApproveProduction
        ActionTypeId:
          Category: Approval
          Owner: AWS
          Provider: Manual
        Configuration:
          NotificationArn: !Ref ApprovalSNSTopic
          CustomData: "Review changes before deploying to production"
    
    - Name: Deploy
      Actions:
      - Name: DeployToECS
        ActionTypeId:
          Category: Deploy
          Owner: AWS
          Provider: CodeDeployToECS
        Configuration:
          ApplicationName: !Ref CodeDeployApp
          DeploymentGroupName: !Ref DeploymentGroup
          TaskDefinitionTemplateArtifact: BuildArtifact
          AppSpecTemplateArtifact: BuildArtifact

SNS Notifications:
  # EventBridge rule for pipeline state changes:
  aws events put-rule \
    --name codepipeline-notifications \
    --event-pattern '{"source":["aws.codepipeline"],
                      "detail-type":["CodePipeline Pipeline Execution State Change"]}'
  
  aws events put-targets \
    --rule codepipeline-notifications \
    --targets '[{"Id":"SNS","Arn":"'$SNS_ARN'"}]'

CodePipeline vs GitHub Actions comparison:

  CodePipeline advantages:
    ✅ Fully AWS-native: deep ECS/CodeDeploy integration
    ✅ Manual approval stage: built-in, integrated with SNS
    ✅ Audit trail in AWS CloudTrail
    ✅ No GitHub secrets in AWS
    ✅ Works without internet (VPC endpoints)
    ✅ Fine-grained IAM control per stage

  GitHub Actions advantages:
    ✅ Much simpler config (YAML vs multiple AWS services)
    ✅ Better developer experience (PR comments, status checks)
    ✅ Rich marketplace (1000s of actions)
    ✅ Platform-agnostic (deploy to any cloud)
    ✅ Lower cost for small teams (free minutes)
    ✅ Easier local testing (act tool)
    ❌ Secrets stored in GitHub (not in AWS Secrets Manager)
    ❌ Less AWS-native integration

  Production recommendation:
    → GitHub Actions for build/test (better DX, market actions)
    → CodeDeploy for actual ECS blue/green (AWS-native, rollback)
    → Combine: GitHub Actions → push to ECR → trigger CodeDeploy
```

> 💡 **Interview tip:** In interviews, the real question is "which would you choose and why?" The honest answer: **GitHub Actions for most teams** — it's simpler, has better developer experience, and the ecosystem is much richer. CodePipeline makes sense when: you have strict AWS-only requirements, need deep AWS governance (Audit Manager, CloudTrail), or you're in a heavily regulated environment where GitHub Actions secrets are a compliance concern. The hybrid approach (GitHub Actions for CI, CodeDeploy for deployment) is increasingly the production standard.

---

### Q301 — Kubernetes | Conceptual | Advanced

> Explain EndpointSlice topology hints and topology-aware routing for multi-AZ cost and latency reduction. Trade-offs and when it causes problems.

#### Key Points to Cover:
```
Problem being solved:
  Multi-AZ Kubernetes cluster: nodes in us-east-1a, us-east-1b, us-east-1c
  Service with 3 pods (one in each AZ)
  Default: kube-proxy routes randomly to any pod regardless of AZ
  
  Result:
  → Pod in us-east-1a calling service → might hit pod in us-east-1c
  → Cross-AZ traffic: $0.01/GB (not free!)
  → Higher latency: cross-AZ > intra-AZ
  → Solution: prefer routing to pods in the SAME AZ

EndpointSlice Topology Hints:
  → kube-proxy uses EndpointSlices for routing tables (since K8s 1.21)
  → Each endpoint can have: forZones hints (which AZ it prefers to serve)
  → EndpointSlice controller calculates and sets hints automatically
  
  apiVersion: discovery.k8s.io/v1
  kind: EndpointSlice
  metadata:
    name: payment-service-abc
  addressType: IPv4
  endpoints:
  - addresses: ["10.244.1.5"]
    conditions: {ready: true}
    hints:
      forZones: [{name: "us-east-1a"}]   # hint: prefer to route from AZ-a
    zone: "us-east-1a"                    # where this endpoint lives
  - addresses: ["10.244.2.3"]
    hints:
      forZones: [{name: "us-east-1b"}]
    zone: "us-east-1b"

Enable topology-aware routing:
  # Service annotation:
  apiVersion: v1
  kind: Service
  metadata:
    name: payment-service
    annotations:
      service.kubernetes.io/topology-mode: "Auto"  # K8s 1.27+
      # Old annotation (deprecated): service.kubernetes.io/topology-aware-hints: "Auto"
  spec:
    selector:
      app: payment-service

  # kube-proxy: when routing, prefer endpoints with forZones hint matching current zone
  # Node knows its own zone via: topology.kubernetes.io/zone label

Trade-offs:

  Benefits:
    ✅ Reduced cross-AZ data transfer costs (significant for high-traffic services)
    ✅ Lower latency (same-AZ networking faster)
    ✅ Better resilience (AZ failure less likely to cascade)

  When topology hints cause problems:
    ❌ Uneven load distribution:
       AZ-a: 10 pods, AZ-b: 2 pods, AZ-c: 2 pods
       If all requests from AZ-a pods: overloads AZ-a endpoints
       Controller: falls back to random routing if imbalance > 20%
    
    ❌ Pod count not balanced across AZs:
       Hints require: roughly equal pods in each AZ
       If one AZ has no pods: topology hints disabled automatically
    
    ❌ Slow auto-scaling response:
       AZ-a gets traffic spike → AZ-a pods scale up → other AZs idle
       (less of an issue with cross-AZ balancing)
    
    ❌ StatefulSet with zone affinity:
       If you WANT cross-AZ for DR purposes, topology hints work against you

  Controller safeguards:
    → Automatically disables hints if:
      - Zone imbalance > 20% of proportional allocation
      - Less than 3 endpoints in the slice
      - Not all zones have endpoints
    → Falls back to normal (random) routing in these cases
```

> 💡 **Interview tip:** The business case for topology hints: at $0.01/GB for cross-AZ traffic, a service handling 10TB/month of inter-service calls = $100/month in cross-AZ costs. For large clusters with hundreds of services, topology-aware routing can save thousands of dollars monthly. The caveat: **it only helps when pods are roughly evenly distributed across AZs**. Use `topologySpreadConstraints` with `whenUnsatisfiable: DoNotSchedule` to ensure even distribution — this makes topology hints effective and prevents the automatic fallback to random routing.

---

### Q302 — GitHub Actions | Troubleshooting | Advanced

> Fix GitHub API rate limiting in workflows and Docker Hub pull rate limiting in CI/CD.

#### Key Points to Cover:
```
Issue 1 — GitHub API Rate Limits in Workflows:

  Rate limits:
    Unauthenticated: 60 requests/hour
    GITHUB_TOKEN (default): 1000 requests/hour per repo
    GitHub App token: 5000 requests/hour
    Enterprise: 15000 requests/hour

  Common causes:
    → Many parallel workflow runs
    → Workflows calling GitHub API directly (creating issues, labels, comments)
    → actions/cache hitting rate limits during peak hours
    → Third-party actions making many API calls

  Symptoms:
    - error: "API rate limit exceeded for..."
    - "You have exceeded a secondary rate limit"
    - HTTP 403 with "rate limit" in response body

  Fixes:
    1. Use GITHUB_TOKEN for all API calls (not PAT):
       env:
         GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    
    2. Add retry logic for rate-limited steps:
       - name: API call with retry
         uses: nick-fields/retry@v3
         with:
           max_attempts: 3
           retry_wait_seconds: 60
           command: gh api repos/owner/repo/issues
    
    3. Reduce parallel workflow runs:
       concurrency:
         group: ${{ github.workflow }}-${{ github.ref }}
         cancel-in-progress: true   # cancel older runs for same branch
    
    4. Use GitHub App instead of GITHUB_TOKEN for higher limits:
       - uses: tibdex/github-app-token@v1
         id: app-token
         with:
           app_id: ${{ secrets.APP_ID }}
           private_key: ${{ secrets.APP_PRIVATE_KEY }}
       
       - name: Higher rate limit with App token
         env:
           GH_TOKEN: ${{ steps.app-token.outputs.token }}
    
    5. Cache responses where possible:
       → Don't call same API endpoint in every job
       → Pass data between jobs via outputs instead of API calls

Issue 2 — Docker Hub Pull Rate Limits:

  Rate limits (as of 2023):
    Unauthenticated: 100 pulls/6 hours per IP
    Free account: 200 pulls/6 hours
    Pro/Team: unlimited
  
  CI/CD impact:
    → Multiple workflow runners share the same egress IP
    → 100 pulls / 6 hours exhausted quickly with parallel builds
    → Error: "toomanyrequests: You have reached your pull rate limit"

  Fix 1 — Authenticate to Docker Hub (Pro account):
    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}  # use token not password
  
  Fix 2 — Use GitHub Container Registry (ghcr.io) instead:
    # Replace: FROM node:20
    # With: FROM ghcr.io/node:20  (if available)
    # Or: mirror to your own registry
    
    - name: Login to GHCR
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}  # free, built-in
  
  Fix 3 — Mirror base images to ECR (best for teams):
    # Set up ECR pull-through cache:
    aws ecr create-pull-through-cache-rule \
      --ecr-repository-prefix dockerhub \
      --upstream-registry-url registry-1.docker.io
    
    # Use ECR mirror in Dockerfile:
    # FROM 123456789.dkr.ecr.us-east-1.amazonaws.com/dockerhub/node:20
    # → ECR pulls from Docker Hub once, caches for your account
    # → All subsequent pulls: from ECR (no rate limit, faster)
  
  Fix 4 — Cache Docker layers in GitHub Actions:
    - uses: docker/build-push-action@v5
      with:
        cache-from: type=gha      # GitHub Actions cache
        cache-to: type=gha,mode=max
        # → Reuses cached layers from previous builds
        # → Fewer pulls from Docker Hub
```

> 💡 **Interview tip:** The Docker Hub rate limit problem has the best permanent solution: **ECR pull-through cache**. Once set up, your entire organization shares the cache — one pull from Docker Hub, then everyone uses ECR. It's faster (regional), cheaper (no rate limits), and works with private images too. For GitHub API rate limits: `concurrency` groups are underused but powerful — they serialize workflows for the same branch, reducing parallel API calls and preventing the cascade of retries that amplifies rate limit issues.

---

### Q303 — Terraform | Conceptual | Advanced

> Explain Terraform `moved` block for 4 use cases: rename resource, move into module, move out of module, rename module. Why prefer it over `terraform state mv`?

#### Key Points to Cover:
```
What moved block does:
  → Tells Terraform: "this resource at OLD address is the same as NEW address"
  → Updates the state: renames the address WITHOUT destroying/recreating
  → Declarative: committed to version control (reviewable, auditable)
  → Applies once, then can be removed (or kept as documentation)

Use Case 1 — Rename a resource within same module:
  # Before: aws_instance.webserver
  # After:  aws_instance.api_server
  
  resource "aws_instance" "api_server" {  # changed name
    # ... same config ...
  }
  
  moved {
    from = aws_instance.webserver
    to   = aws_instance.api_server
  }
  
  # terraform plan: shows 0 to add, 0 to change, 0 to destroy
  # Just a rename in state — no actual AWS changes

Use Case 2 — Move resource INTO a module:
  # Before: standalone resource at root
  # aws_s3_bucket.logs (in root module)
  
  # After: moved into module
  module "storage" {
    source = "./modules/storage"
  }
  # Inside modules/storage/main.tf:
  # resource "aws_s3_bucket" "logs" { ... }
  
  moved {
    from = aws_s3_bucket.logs                    # root module address
    to   = module.storage.aws_s3_bucket.logs     # module address
  }
  
  # Result: state updated, S3 bucket not touched

Use Case 3 — Move resource OUT of a module:
  # Opposite of above:
  moved {
    from = module.storage.aws_s3_bucket.logs
    to   = aws_s3_bucket.logs
  }
  # Use case: decomposing a module, pulling resources to root

Use Case 4 — Rename a module:
  # Before: module "legacy_vpc"
  # After:  module "main_vpc"
  
  module "main_vpc" {
    source = "./modules/vpc"
    # ... same config ...
  }
  
  moved {
    from = module.legacy_vpc
    to   = module.main_vpc
  }
  
  # ALL resources inside the module get their addresses updated:
  # module.legacy_vpc.aws_vpc.main → module.main_vpc.aws_vpc.main
  # module.legacy_vpc.aws_subnet.public[0] → module.main_vpc.aws_subnet.public[0]
  # etc. (cascading rename)

moved block vs terraform state mv:

  terraform state mv:
    Command line only (not tracked in code)
    Requires: manual execution for every environment
    Risk: forgot to run on prod, dev, staging separately
    No code review (just a command someone ran)
    Easy to make mistakes in command syntax
    Doesn't handle for_each/count well (must run for each instance)
  
  moved block:
    Code change (goes through PR review)
    Applied automatically by terraform apply in all environments
    Version controlled: auditable, reversible
    Handles for_each/count elegantly:
      moved {
        from = aws_instance.servers["old-key"]
        to   = aws_instance.servers["new-key"]
      }
    Self-documenting: team members understand what changed and why

  Lifecycle of moved block:
    1. Add moved block + rename resource
    2. Commit, PR review
    3. Run terraform apply in all environments (moved block auto-applies)
    4. Keep moved block for historical documentation
       OR remove after all environments are updated
    5. Future engineers can see: "webserver was renamed to api_server on 2024-01-15"
```

> 💡 **Interview tip:** The key advantage of `moved` blocks over `terraform state mv` is **multi-environment safety**. If you have 5 environments (dev/staging/prod-us/prod-eu/dr), you'd need to run `terraform state mv` in all 5 separately and hope you don't forget one. With a `moved` block in code, every `terraform apply` across every environment applies the rename. This is especially critical for resources managed by Atlantis or Terraform Cloud where `terraform state mv` isn't easily accessible.

---

### Q304 — Prometheus | Troubleshooting | Advanced

> AlertManager shows alerts as silenced but you didn't create silences. Inhibition rules are suppressing alerts. Debug unexpected inhibition.

#### Key Points to Cover:
```
Inhibition vs Silence — difference:
  Silence: manually created, time-bounded, matches specific labels
           → You create it, you know about it
  
  Inhibition rule: configured in alertmanager.yml
                   → Automatic: fires when source alert matches
                   → Can suppress unexpectedly if misconfigured
                   → Alert shows as "inhibited" in UI

How inhibition rules work:
  inhibit_rules:
  - source_match:         # the SUPPRESSOR alert
      alertname: NodeDown
    target_match:         # alerts being SUPPRESSED
      severity: warning
    equal:                # labels that must MATCH between source and target
    - node
  
  Effect: when NodeDown fires for node "worker-1"
          suppress ALL warning alerts for node "worker-1"
  Reason: if node is down, warning alerts from that node are redundant

Common misconfiguration — overly broad inhibition:
  # BAD: this suppresses ALL warnings when ANY critical fires
  inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    # Missing "equal" clause = no label matching required
    # → ANY critical alert suppresses ALL warnings
    # → Even if critical is for service-A, it suppresses warning for service-B!

  # FIX: add equal to scope the inhibition:
  inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal:
    - alertname      # only suppress if SAME alertname
    - cluster        # only suppress if SAME cluster
    - namespace      # only suppress if SAME namespace

Debugging unexpected inhibition:

Step 1 — Check AlertManager UI:
  http://alertmanager:9093/
  → Alerts tab → click an "inhibited" alert
  → Shows: which inhibition rule is suppressing it
  → Shows: which SOURCE alert is causing the inhibition

Step 2 — AlertManager API:
  # List all currently active alerts including inhibited:
  curl http://alertmanager:9093/api/v2/alerts?silenced=true&inhibited=true&active=true
  
  # Look for: "inhibitedBy": ["<source-alert-fingerprint>"]

Step 3 — Check current inhibition rules:
  curl http://alertmanager:9093/api/v2/status | jq .config.original

Step 4 — Analyze equal clause gaps:
  # Find alerts that SHOULDN'T be inhibited:
  # Check their labels vs the source alert labels
  # Verify: does the equal clause actually scope correctly?

Step 5 — Test inhibition logic:
  # Use amtool to test routing/inhibition:
  amtool --alertmanager.url=http://alertmanager:9093 \
    alert query --inhibited
  
  # Test what would happen for specific labels:
  amtool config routes test \
    --config.file alertmanager.yml \
    alertname="HighErrorRate" severity="warning" namespace="production"

Common debugging scenarios:

  Scenario: NodeDown critical suppresses warnings from DIFFERENT nodes:
    Cause: equal clause missing node label
    Fix: add "equal: [node]"
  
  Scenario: Production critical suppresses dev warnings:
    Cause: equal clause missing environment label
    Fix: add "equal: [environment]"
  
  Scenario: All warnings silenced during business hours:
    Cause: time-based inhibition using time_intervals
    Check: alertmanager.yml for time_intervals + inhibit_rules
```

> 💡 **Interview tip:** The **missing `equal` clause** is the most common inhibition misconfiguration and the hardest to debug without knowing where to look. Without `equal`, the inhibition is cluster-wide: "if any alert with severity=critical fires, suppress all alerts with severity=warning." This sounds intentional until you realize a database critical alert is suppressing unrelated frontend warnings. The `equal` list is what scopes inhibition to the same service/node/namespace. The debugging shortcut: AlertManager UI shows exactly which inhibition rule is active and which source alert triggered it — start there before diving into config files.

---

### Q305 — AWS | Conceptual | Advanced

> Explain AWS PrivateLink in depth — endpoint services, interface endpoints, gateway endpoints. SaaS provider pattern without VPC peering.

#### Key Points to Cover:
```
PrivateLink fundamentals:
  → Connect to services WITHOUT traffic going through internet
  → No internet gateway, no NAT gateway, no public IPs required
  → Traffic stays entirely within AWS backbone network
  → More secure, lower latency, no internet exposure

Two types of VPC endpoints:

  Interface Endpoints (PrivateLink):
    → Creates: ENI (Elastic Network Interface) in your VPC subnet
    → ENI gets: private IP from your subnet's CIDR
    → DNS: service hostname resolves to the ENI's private IP
    → Works with: most AWS services + any PrivateLink-enabled service
    → Cost: $0.01/hour + $0.01/GB processed
    → Supports: Security groups (you control which resources can access)

  Gateway Endpoints:
    → Route table entry pointing to the endpoint (not an ENI)
    → No private IP needed
    → Supported services: S3 and DynamoDB ONLY
    → Why free? S3 and DynamoDB are so widely used + gateway is simpler
    → Works at: route table level (affects entire subnet or VPC)
    → Does NOT support: Security groups (just route table control)
    → DNS: service hostname unchanged, route table directs traffic

Technical flow — Interface Endpoint:
  EC2 in private subnet → DNS lookup for sqs.amazonaws.com
  → Route 53 returns: ENI private IP in same VPC
  → EC2 → ENI (same VPC, private) → AWS SQS service
  Never leaves AWS network, never touches internet

Technical flow — Gateway Endpoint (S3):
  EC2 in private subnet → wants s3.amazonaws.com
  → Route table has: pl-xxxxxxxx (S3 prefix list) → vpce-xxxxxxxx
  → Traffic routed to gateway endpoint → S3
  No internet gateway involved, traffic stays in AWS

SaaS Provider pattern with PrivateLink (no VPC peering):
  
  Problem with VPC peering:
  → Non-overlapping CIDR required
  → Transitive peering not allowed
  → Customer can see all resources in provider VPC
  → Management complexity at scale (N² connections)

  PrivateLink solution:
  
  SaaS Provider setup:
    → Deploy service behind Network Load Balancer (NLB)
    → Create: VPC Endpoint Service (exposes NLB via PrivateLink)
    → Configure: which AWS accounts can access
    
    aws ec2 create-vpc-endpoint-service-configuration \
      --network-load-balancer-arns arn:aws:elasticloadbalancing:...:loadbalancer/net/saas-nlb/... \
      --acceptance-required   # provider must approve each customer
    
    # Returns: Service Name = com.amazonaws.vpce.us-east-1.vpce-svc-xxxxxxxx

  Customer setup:
    → Create Interface Endpoint pointing to provider's service name
    
    aws ec2 create-vpc-endpoint \
      --vpc-id vpc-customer-xxxx \
      --service-name com.amazonaws.vpce.us-east-1.vpce-svc-xxxxxxxx \
      --vpc-endpoint-type Interface \
      --subnet-ids subnet-customer-yyyy \
      --security-group-ids sg-customer-zzzz
    
    # Customer gets: private IP in their own VPC
    # Customer traffic: stays in their VPC → PrivateLink → provider VPC
    # Provider: never sees customer's VPC CIDR or other resources

  Benefits over VPC peering:
    ✅ Customer VPC CIDR can be anything (no overlap constraint)
    ✅ Customer cannot see provider's other resources
    ✅ Provider can serve 1000s of customers with one service
    ✅ No transitive peering needed
    ✅ Provider controls which customers can connect (approval required)
    
  Real example: Snowflake, Databricks, Datadog all use PrivateLink
  for enterprise customers who require all traffic to stay off internet
```

> 💡 **Interview tip:** The **gateway vs interface endpoint** distinction is frequently tested. Remember: **Gateway = S3 + DynamoDB only, free, route table entry**. **Interface = everything else, costs money, creates ENI with private IP**. The key insight on gateway endpoints: they're free because S3 and DynamoDB are high-volume services that AWS wants to keep off the internet for performance/security, and the gateway mechanism (route tables) is simpler than creating ENIs at scale. For the SaaS PrivateLink pattern: this is how AWS Marketplace software works behind the scenes — vendors expose their services via PrivateLink, customers connect without VPC peering complexity.

---

### Q306 — Kubernetes | Scenario-Based | Advanced

> Implement Kubernetes RuntimeClass for gVisor on sensitive workloads alongside runc. Walk through installation, RuntimeClass objects, scheduling, and performance trade-offs.

#### Key Points to Cover:
```
What is RuntimeClass:
  → Kubernetes mechanism to select the container runtime
  → Different pods can use different runtimes on the same cluster
  → Default: runc (standard OCI runtime, used by containerd/CRI-O)
  → Alternative: gVisor (runsc), Kata Containers, Firecracker

Why gVisor (runsc):
  → Additional security isolation between container and host kernel
  → Normal containers: share host kernel (syscalls go to host kernel)
  → gVisor: syscalls intercepted and handled by gVisor's sandbox kernel
  → Attack surface: if container is compromised, attacker hits gVisor's kernel
    → Much harder to escape to host vs standard runc

gVisor vs runc trade-offs:
  runc:
    ✅ Fast startup (microseconds)
    ✅ Full kernel feature compatibility
    ✅ Near-native performance
    ❌ Container escape = host compromise

  gVisor (runsc):
    ✅ Strong isolation (separate kernel space)
    ✅ Harder container escapes
    ❌ 2-5x slower for syscall-heavy workloads
    ❌ Not all syscalls supported (compatibility issues)
    ❌ Slower startup (~100ms)
    ❌ Higher memory overhead
    ❌ Network performance degraded

Step 1 — Install gVisor on nodes:
  # On each node that will run gVisor workloads:
  curl -fsSL https://gvisor.dev/archive.key | apt-key add -
  add-apt-repository "deb [arch=amd64,arm64] https://storage.googleapis.com/gvisor/releases release main"
  apt-get install runsc

  # Configure containerd to use runsc:
  # /etc/containerd/config.toml:
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
    runtime_type = "io.containerd.runsc.v1"
  
  systemctl restart containerd

Step 2 — Label nodes that have gVisor installed:
  kubectl label node worker-1 node.kubernetes.io/runtime=gvisor
  kubectl label node worker-2 node.kubernetes.io/runtime=gvisor
  # Other nodes: default runc only

Step 3 — Create RuntimeClass objects:
  # Default (runc):
  apiVersion: node.k8s.io/v1
  kind: RuntimeClass
  metadata:
    name: runc
  handler: runc         # must match containerd runtime handler name
  # No scheduling constraint needed (all nodes support runc)
  
  # gVisor:
  apiVersion: node.k8s.io/v1
  kind: RuntimeClass
  metadata:
    name: gvisor
  handler: runsc        # must match containerd config name
  scheduling:
    nodeSelector:
      node.kubernetes.io/runtime: gvisor  # only schedule on gVisor nodes
    tolerations:
    - effect: NoSchedule
      key: runtime
      value: gvisor

Step 4 — Use RuntimeClass in sensitive pods:
  # Normal workload (uses default runc):
  apiVersion: v1
  kind: Pod
  spec:
    containers:
    - name: api
      image: my-api:latest
    # runtimeClassName not set = default runc

  # Sensitive workload (uses gVisor):
  apiVersion: v1
  kind: Pod
  metadata:
    name: payment-processor
  spec:
    runtimeClassName: gvisor   # ← select gVisor sandbox
    containers:
    - name: payment
      image: payment-processor:latest

Step 5 — Taint gVisor nodes (optional, dedicated nodes):
  kubectl taint node worker-1 runtime=gvisor:NoSchedule
  # Prevents normal workloads from landing on gVisor nodes
  # gVisor pods tolerate this taint (configured in RuntimeClass scheduling)

When to use gVisor:
  ✅ Multi-tenant clusters (untrusted workloads)
  ✅ Processing untrusted user input or code
  ✅ Compliance requirements for workload isolation
  ✅ Cryptographic operations, secrets processing
  ❌ High-throughput databases (performance hit too high)
  ❌ Networking-intensive workloads
  ❌ Applications using unusual syscalls (may fail)
```

> 💡 **Interview tip:** RuntimeClass is the Kubernetes answer to the question "what if I need stronger isolation for some workloads but not all?" The key insight: you don't need a separate cluster for sensitive workloads — RuntimeClass lets you mix runtimes on the same cluster. The practical deployment pattern: **taint gVisor nodes** so only workloads that explicitly tolerate it (via the RuntimeClass scheduling config) land there. This prevents normal workloads from accidentally running on slower gVisor nodes. Mention Kata Containers as an alternative — it uses hardware virtualization (VM per pod) for even stronger isolation than gVisor, at even higher performance cost.

---

### Q307 — ArgoCD | Troubleshooting | Advanced

> ArgoCD SyncFailed: admission webhook for Nginx Ingress timing out. Diagnose and fix webhook-related sync failures.

#### Key Points to Cover:
```
Error analysis:
  "one or more objects failed to apply, reason: Internal error occurred:
   failed calling webhook 'validate.nginx.ingress.kubernetes.io':
   the server is currently unable to handle the request"

  This means:
  → ArgoCD ran kubectl apply for an Ingress resource
  → kube-apiserver tried to call the Nginx webhook for validation
  → Webhook (Nginx Ingress pod) didn't respond within timeout
  → API server: "Internal error occurred" (not the resource being wrong)

Root causes of webhook timeout:

  1. Nginx Ingress pods are down/unhealthy:
     kubectl get pods -n ingress-nginx
     # If 0/1 Ready → webhook server unreachable
  
  2. Nginx Ingress pods are being updated during ArgoCD sync:
     → Rolling update temporarily removes webhook pods
     → ArgoCD syncs Ingress resources at exact wrong time
  
  3. Webhook pods exist but are overloaded:
     → High CPU, OOM pending
  
  4. Network policy blocking apiserver → webhook:
     → Webhook is on TCP 8443 (default Nginx)
     → apiserver must reach it directly

Diagnosis:
  # Check webhook config:
  kubectl get validatingwebhookconfigurations
  kubectl describe validatingwebhookconfiguration ingress-nginx-admission
  # Look for: failurePolicy, timeoutSeconds, service reference

  # Check if webhook endpoint is reachable:
  kubectl get svc -n ingress-nginx
  kubectl describe svc ingress-nginx-controller-admission -n ingress-nginx

  # Test webhook directly:
  kubectl get endpoints -n ingress-nginx ingress-nginx-controller-admission
  # If no endpoints: pods not ready

  # Check Nginx Ingress pods:
  kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller

Immediate fix — set failurePolicy to Ignore:
  # Patch the webhook to not block on timeout:
  kubectl patch validatingwebhookconfiguration ingress-nginx-admission \
    --type='json' \
    -p='[{"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"}]'
  
  # WARNING: Ignore means invalid Ingress objects can be created
  # Use only as temporary fix until webhook is healthy

Proper fix — ensure webhook HA:
  # Scale up Nginx Ingress controller (minimum 2 replicas):
  kubectl scale deployment ingress-nginx-controller -n ingress-nginx --replicas=2
  
  # Add PodDisruptionBudget:
  kubectl create poddisruptionbudget nginx-pdb -n ingress-nginx \
    --selector app.kubernetes.io/component=controller \
    --min-available=1

ArgoCD-specific mitigation:
  # Use sync waves to control order:
  # Wave -1: deploy Nginx Ingress controller first
  # Wave 0: deploy Ingress resources (webhook is already up)
  
  metadata:
    annotations:
      argocd.argoproj.io/sync-wave: "-1"  # on IngressClass/controller resources
  
  # OR: add retry logic to ArgoCD sync:
  spec:
    syncPolicy:
      retry:
        limit: 5
        backoff:
          duration: 5s
          factor: 2
          maxDuration: 3m
  # ArgoCD retries sync on transient failures

  # OR: use ignoreDifferences + manual sync for Ingress objects
  # while webhook is being fixed

General webhook best practices:
  → Always: failurePolicy: Fail for security webhooks (OPA, PSP replacements)
  → Always: failurePolicy: Ignore for operational webhooks (Nginx, sidecar injection)
  → Always: run at least 2 webhook pod replicas
  → Always: PDB with minAvailable=1
  → Always: set timeoutSeconds (default 10s — sufficient for most cases)
  → Never: deploy webhook in same namespace it's controlling (bootstrap problem)
```

> 💡 **Interview tip:** This scenario highlights a fundamental webhook design tension: **failurePolicy: Fail** means strong enforcement but creates availability risk. **failurePolicy: Ignore** means no availability risk but validation can be bypassed. For Nginx Ingress validation: Ignore is appropriate (invalid Ingress just doesn't route correctly — not a security issue). For OPA/Gatekeeper policies: Fail is required (if the policy webhook is down, you need to block all changes, not silently allow them). The ArgoCD sync wave pattern is the most elegant fix — deploy the controller in wave -1 so it's always running before wave 0 resources (which need the webhook) are applied.

---

### Q308 — Linux / Bash | Scenario-Based | Advanced

> Write a Bash script for automated SSH key rotation across a fleet: generate 4096-bit RSA key, distribute, verify before removing old key, audit report.

#### Key Points to Cover:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Configuration
INVENTORY_FILE="${1:-inventory.txt}"
LOG_FILE="/var/log/ssh-key-rotation-$(date +%Y%m%d_%H%M%S).log"
AUDIT_REPORT="/var/log/ssh-rotation-audit-$(date +%Y%m%d_%H%M%S).txt"
NEW_KEY_FILE="/tmp/ssh-rotation-key-$$"
OPERATOR=$(whoami)
SUCCESS_COUNT=0
FAIL_COUNT=0
declare -a FAILED_SERVERS=()

# Cleanup on exit
cleanup() {
  rm -f "${NEW_KEY_FILE}" "${NEW_KEY_FILE}.pub" 2>/dev/null || true
  log "Cleanup complete"
}
trap cleanup EXIT

log() {
  local msg="[$(date '+%Y-%m-%d %H:%M:%S')] [$OPERATOR] $*"
  echo "$msg" | tee -a "$LOG_FILE"
}

# Step 1: Generate new 4096-bit RSA key pair
generate_key() {
  log "Generating 4096-bit RSA key pair..."
  ssh-keygen -t rsa -b 4096 \
    -f "${NEW_KEY_FILE}" \
    -N "" \
    -C "rotation-$(date +%Y%m%d)-${OPERATOR}" \
    -q
  
  NEW_PUBLIC_KEY=$(cat "${NEW_KEY_FILE}.pub")
  log "New public key fingerprint: $(ssh-keygen -lf "${NEW_KEY_FILE}.pub")"
}

# Step 2: Distribute new key to a server
distribute_key() {
  local server="$1"
  log "  Distributing new public key to ${server}..."
  
  ssh -o StrictHostKeyChecking=accept-new \
      -o ConnectTimeout=10 \
      -o BatchMode=yes \
      "${server}" \
    "mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
     echo '${NEW_PUBLIC_KEY}' >> ~/.ssh/authorized_keys && \
     chmod 600 ~/.ssh/authorized_keys && \
     echo 'Key added'"
}

# Step 3: Verify new key works BEFORE removing old key
verify_new_key() {
  local server="$1"
  log "  Verifying new key authentication on ${server}..."
  
  # Test: connect using ONLY the new private key
  ssh -i "${NEW_KEY_FILE}" \
      -o IdentitiesOnly=yes \
      -o StrictHostKeyChecking=yes \
      -o ConnectTimeout=10 \
      -o BatchMode=yes \
      "${server}" \
    "echo 'verification-success'" 2>/dev/null
}

# Step 4: Remove OLD public key (only after verification passes)
remove_old_key() {
  local server="$1"
  local old_key="$2"
  log "  Removing old key from ${server}..."
  
  # Escape the old key for use in sed
  local escaped_key
  escaped_key=$(echo "$old_key" | sed 's/[[\.*^$()+?{|]/\\&/g')
  
  ssh -i "${NEW_KEY_FILE}" \
      -o IdentitiesOnly=yes \
      -o BatchMode=yes \
      "${server}" \
    "sed -i '/${escaped_key}/d' ~/.ssh/authorized_keys && \
     echo 'Old key removed'"
}

# Process a single server
process_server() {
  local server="$1"
  local old_public_key="$2"
  
  log "━━━━ Processing: ${server} ━━━━"
  
  # Distribute new key
  if ! distribute_key "${server}"; then
    log "  ERROR: Failed to distribute key to ${server}"
    FAILED_SERVERS+=("${server}:distribute_failed")
    FAIL_COUNT=$((FAIL_COUNT + 1))
    return 1
  fi
  
  # Verify new key works
  if ! verify_new_key "${server}"; then
    log "  ERROR: New key verification FAILED on ${server} — keeping old key"
    # Remove the new key we just added (rollback):
    ssh -o BatchMode=yes "${server}" \
      "sed -i '/rotation-$(date +%Y%m%d)/d' ~/.ssh/authorized_keys" 2>/dev/null || true
    FAILED_SERVERS+=("${server}:verification_failed")
    FAIL_COUNT=$((FAIL_COUNT + 1))
    return 1
  fi
  
  # Only remove old key after successful verification
  if ! remove_old_key "${server}" "${old_public_key}"; then
    log "  WARNING: Failed to remove old key from ${server} (new key still works)"
    FAILED_SERVERS+=("${server}:old_key_removal_failed")
    FAIL_COUNT=$((FAIL_COUNT + 1))
    return 1
  fi
  
  log "  ✓ ${server}: rotation SUCCESSFUL"
  SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
  return 0
}

# Main
main() {
  [[ ! -f "$INVENTORY_FILE" ]] && { echo "Inventory file not found: $INVENTORY_FILE"; exit 1; }
  
  log "=== SSH Key Rotation Started by ${OPERATOR} ==="
  log "Inventory: ${INVENTORY_FILE}"
  
  # Get current (old) public key
  local old_key_file="${HOME}/.ssh/id_rsa.pub"
  [[ ! -f "$old_key_file" ]] && { log "ERROR: No existing public key found at $old_key_file"; exit 1; }
  local OLD_PUBLIC_KEY
  OLD_PUBLIC_KEY=$(cat "$old_key_file")
  log "Old key fingerprint: $(ssh-keygen -lf "$old_key_file")"
  
  # Generate new key pair
  generate_key
  
  # Process each server
  while IFS= read -r server || [[ -n "$server" ]]; do
    [[ -z "$server" || "$server" =~ ^# ]] && continue
    process_server "$server" "$OLD_PUBLIC_KEY" || true  # continue on failure
  done < "$INVENTORY_FILE"
  
  # Generate audit report
  {
    echo "============================================"
    echo "  SSH Key Rotation Audit Report"
    echo "  Date: $(date '+%Y-%m-%d %H:%M:%S')"
    echo "  Operator: ${OPERATOR}"
    echo "  New Key Fingerprint: $(ssh-keygen -lf "${NEW_KEY_FILE}.pub")"
    echo "============================================"
    echo "  Successful: ${SUCCESS_COUNT}"
    echo "  Failed:     ${FAIL_COUNT}"
    echo "============================================"
    if [[ ${#FAILED_SERVERS[@]} -gt 0 ]]; then
      echo "  Failed Servers:"
      for s in "${FAILED_SERVERS[@]}"; do
        echo "    - $s"
      done
    fi
    echo "============================================"
    echo "  Log file: ${LOG_FILE}"
  } | tee "$AUDIT_REPORT"
  
  log "=== Rotation complete. Audit: ${AUDIT_REPORT} ==="
  
  # Exit with failure if any servers failed
  [[ $FAIL_COUNT -gt 0 ]] && exit 1
  return 0
}

main "$@"
```

> 💡 **Interview tip:** The **verify-before-remove** pattern is the critical safety feature. Without it, distributing the new key and immediately removing the old key risks a lockout if the new key distribution succeeded but something else prevents the new key from working (permissions issue, SSH daemon config). By testing the new key with `IdentitiesOnly=yes` first, you confirm it actually works before removing the only other access method. The `IdentitiesOnly=yes` SSH option is the key flag — it forces SSH to use ONLY the specified identity file and not try any other keys (like ssh-agent keys).

---

### Q309 — AWS | Scenario-Based | Advanced

> AWS account consolidation during acquisition: migrate 3 accounts with production workloads, IAM, VPCs, RDS, S3 petabyte data. Strategy, sequence, and risks.

#### Key Points to Cover:
```
Migration phases and sequence:

Phase 1 — Assessment (weeks 1-2):
  → Inventory: all AWS resources across 3 accounts
  → Identify: dependencies between resources
  → Check: overlapping VPC CIDRs (must resolve before anything else)
  → Review: IAM policies, roles, permission boundaries
  → Compliance: data residency requirements (can data cross regions?)
  → Cost: data transfer charges for migration
  
  aws resource-explorer-2 search --query-string "resourcetype:*"

Phase 2 — Organization setup (week 3):
  → Invite existing accounts into new parent Organization
  → Apply: SCPs (Service Control Policies) at OU level
  → Don't: move accounts into production OUs until ready
  → Set up: AWS Control Tower for guardrails
  → Enable: CloudTrail, Config, SecurityHub across all accounts

Phase 3 — IAM federation (weeks 4-5):
  → Set up: AWS SSO / IAM Identity Center with parent company IdP
  → Map: existing IAM roles to permission sets
  → Enable: cross-account access via IAM roles (not IAM users)
  → Deprecate: long-lived IAM user credentials
  
  # Cross-account role assumption:
  aws sts assume-role \
    --role-arn arn:aws:iam::TARGET_ACCOUNT:role/CrossAccountAdmin \
    --role-session-name migration

Phase 4 — Networking (weeks 5-8):
  → Resolve CIDR conflicts:
    # If 10.0.0.0/16 overlaps across accounts → must re-address one VPC
    # Or: use separate address spaces per account (10.1.0.0/16, 10.2.0.0/16)
  
  → Set up: Transit Gateway in parent account (hub)
  → Attach: all VPCs to Transit Gateway
  → Remove: VPC peering (replace with TGW routes)
  
  → DNS: Route 53 Private Hosted Zones
    # Share PHZ across accounts via RAM (Resource Access Manager)

Phase 5 — Data migration (weeks 8-16):
  S3 data (petabyte scale):
    → Use: S3 Batch Replication (most efficient for cross-account S3)
    → OR: S3 Cross-Region Replication if changing regions
    → Enable: S3 Replication on source buckets → destination
    → Use: S3 Inventory to verify all objects copied
    → DO NOT: just change bucket policies — create new buckets in new account
    
    aws s3 sync s3://source-bucket s3://dest-bucket \
      --source-region us-east-1 \
      --region us-east-1 \
      --acl bucket-owner-full-control  # critical for cross-account

  RDS databases:
    → Take: final snapshot
    → Share: snapshot with destination account
    → Restore: in destination account
    → Use: AWS DMS for live migration (minimal downtime)
    → Cutover: update connection strings, test, switch DNS
    
    # Share snapshot:
    aws rds modify-db-snapshot-attribute \
      --db-snapshot-identifier prod-snapshot \
      --attribute-name restore \
      --values-to-add DEST_ACCOUNT_ID

Phase 6 — Application migration (weeks 12-20):
  → Re-deploy: application in new account (Infrastructure as Code)
  → Or: if using containers → push images to new ECR, update task defs
  → DNS cutover: Route 53 record change (low TTL → switch → raise TTL)
  → Validate: smoke tests, integration tests in new account

Key risks and mitigations:

  Risk: S3 object ownership confusion
  → Cross-account uploaded objects owned by uploader
  → Mitigation: S3 Bucket Owner Enforced (ACLs disabled)

  Risk: Data transfer costs ($0.09/GB for internet-bound)
  → Mitigation: use within-AWS transfer ($0.02/GB) or S3 batch replication

  Risk: IAM role trust policy gaps
  → Mitigation: audit cross-account assumes with IAM Access Analyzer

  Risk: RDS downtime during migration
  → Mitigation: AWS DMS for near-zero downtime cutover

  Risk: Compliance — data must stay in specific region
  → Mitigation: migrate within same region (no cross-region needed)
```

> 💡 **Interview tip:** The S3 migration is the hardest problem at petabyte scale. Key mistakes to avoid: (1) `aws s3 sync` is fine for small buckets but at petabyte scale with millions of objects, **S3 Batch Replication** is the right tool — it handles checkpointing and is resumable. (2) Always add `--acl bucket-owner-full-control` to cross-account syncs or objects will be owned by the source account and the destination can't access them. (3) For RDS, the snapshot share + restore method causes downtime — for production, **AWS DMS** runs a live replication that keeps the source and destination in sync until you're ready to cut over.

---

### Q310 — Kubernetes | Conceptual | Advanced

> Explain Service session affinity, ExternalTrafficPolicy: Local vs Cluster, and the deprecated topologyKeys.

#### Key Points to Cover:
```
Service Session Affinity (ClientIP mode):
  → Routes: requests from the same client IP always to the same pod
  → Implementation: kube-proxy maintains a mapping table (client IP → pod)
  → Timeout: configurable (default 10800s = 3 hours)
  
  apiVersion: v1
  kind: Service
  spec:
    sessionAffinity: ClientIP
    sessionAffinityConfig:
      clientIP:
        timeoutSeconds: 3600  # 1 hour
  
  Use cases:
    ✅ Applications with in-memory session state
    ✅ Maintaining long-lived connections (WebSocket, gRPC streams)
  
  Limitations:
    ❌ IP-based only: doesn't work behind shared NAT (many users, one IP)
    ❌ New pod = users get redistributed (pod scale-up breaks affinity)
    ❌ Node failure = all affected clients lose affinity
    ❌ Not real sticky sessions: client IP changes (mobile, VPN) = different pod
    ❌ Load imbalance: one IP sending all traffic → one pod overloaded
    
  Better alternative: use Redis/Memcached for session state → stateless pods

ExternalTrafficPolicy: Local vs Cluster:
  Applies to: NodePort and LoadBalancer services
  Controls: what happens when traffic arrives at a node from external source

  ExternalTrafficPolicy: Cluster (default):
    → External traffic → any node → kube-proxy → ANY pod (any node)
    → Load balanced across ALL pods regardless of which node traffic arrived at
    → Source IP is SNAT'd (pod sees node IP, not client IP)
    → Pro: even distribution, works even if node has no pods
    → Con: potential extra hop (external → node-A → pod on node-B)
    → Con: source IP masqueraded (can't see client IP in app)

  ExternalTrafficPolicy: Local:
    → External traffic → node → ONLY pods on THAT node
    → Source IP preserved (pod sees real client IP)
    → Pro: no extra hop (direct: node → pod on same node)
    → Pro: client IP visible (important for auth, rate limiting, geo-routing)
    → Con: uneven load if pods not equally distributed across nodes
    → Con: if node has NO pods for this service: request DROPPED (health check fails → LB removes node)
    
  Use case for Local:
    → You need real client IP (for logging, rate limiting, geo-blocking)
    → Combined with: PodAntiAffinity to spread pods evenly (avoid uneven load)
    → Combined with: AWS NLB (which passes through source IP natively)

topologyKeys (deprecated, why it failed):
  → Original attempt at topology-aware routing (K8s 1.17-1.21)
  → Field in Service spec: topologyKeys (list of topology domains)
  → Worked: route to pods in same zone if available, else same region, else any
  → Problems:
    1. Complex: had to specify the full preference ordering in every Service
    2. kube-proxy-based: performance overhead on every node
    3. Non-standard: each CNI implemented it differently
    4. Maintenance burden: had to update every Service
  → Replaced by: EndpointSlice topology hints (automatic, controlled by annotation)
  → Topology hints: simpler, set once on the Service, controller calculates hints
```

> 💡 **Interview tip:** The `ExternalTrafficPolicy: Local` source IP preservation is the most operationally important concept here. Many teams enable it specifically to get real client IPs in their application logs for security analytics (who is attacking us?) and compliance (GDPR requires knowing which IP accessed PII). The caveat: **pods must be evenly distributed across nodes** or load will be uneven. Combine with `podAntiAffinity` requiring one pod per node, and with a minimum node count matching desired replicas, to ensure even distribution. Without even distribution, some nodes serve all traffic (those with pods) while others drop it (those without).

---

### Q311 — Grafana | Scenario-Based | Advanced

> Build a Grafana K8s cluster overview dashboard: PromQL for CPU/memory utilization, capacity planning projection, top namespaces, pod health, node pressure, OOMKills/evictions.

#### Key Points to Cover:
```
Panel 1 — Cluster CPU Utilization:
  # Current total CPU usage across all nodes:
  sum(rate(node_cpu_seconds_total{mode!="idle"}[5m]))
  /
  sum(kube_node_status_allocatable{resource="cpu"})

  # Capacity planning: linear projection 7 days out:
  predict_linear(
    sum(rate(node_cpu_seconds_total{mode!="idle"}[5m]))[1h:5m],
    7 * 24 * 3600  # 7 days in seconds
  )
  / sum(kube_node_status_allocatable{resource="cpu"})

Panel 2 — Cluster Memory Utilization:
  # Current memory in use (working set, not just requested):
  sum(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
  /
  sum(node_memory_MemTotal_bytes)

  # Capacity prediction:
  predict_linear(
    (sum(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes))[1h:5m],
    7 * 24 * 3600
  )
  / sum(node_memory_MemTotal_bytes)

Panel 3 — Top 10 Namespaces by CPU consumption:
  topk(10,
    sum by(namespace) (
      rate(container_cpu_usage_seconds_total{container!=""}[5m])
    )
  )

Panel 4 — Top 10 Namespaces by Memory:
  topk(10,
    sum by(namespace) (
      container_memory_working_set_bytes{container!=""}
    )
  )

Panel 5 — Pod Health Summary (stat panel per namespace):
  # Running pods:
  sum by(namespace) (kube_pod_status_phase{phase="Running"})
  # Pending pods:
  sum by(namespace) (kube_pod_status_phase{phase="Pending"})
  # Failed pods:
  sum by(namespace) (kube_pod_status_phase{phase="Failed"})

  # CrashLoopBackOff specifically:
  sum by(namespace) (
    kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}
  )

Panel 6 — Node Pressure Conditions:
  # DiskPressure:
  kube_node_status_condition{condition="DiskPressure", status="true"}
  # MemoryPressure:
  kube_node_status_condition{condition="MemoryPressure", status="true"}
  # PIDPressure:
  kube_node_status_condition{condition="PIDPressure", status="true"}
  # NotReady:
  kube_node_status_condition{condition="Ready", status="false"}

Panel 7 — OOMKills in last hour:
  # OOM kills (container restarts due to memory limit):
  sum(
    increase(kube_pod_container_status_restarts_total[1h])
  ) by(namespace, pod, container)
  # Filter to OOM specifically (check container status reason):
  # OR use: container_oom_events_total from cAdvisor

  # OOM events directly:
  increase(container_oom_events_total[1h])

Panel 8 — Evictions in last hour:
  # Count evicted pod events:
  sum(
    increase(kube_pod_status_phase{phase="Failed"}[1h])
  ) by(namespace)

  # Better: use eviction event metric if available:
  increase(kube_evictions_total[1h])

Panel 9 — Resource Requests vs Limits utilization:
  # CPU request utilization ratio:
  sum(kube_pod_container_resource_requests{resource="cpu"})
  /
  sum(kube_node_status_allocatable{resource="cpu"})
  
  # Flag over-committed nodes:
  sum by(node) (kube_pod_container_resource_requests{resource="memory"})
  /
  sum by(node) (kube_node_status_allocatable{resource="memory"})
```

> 💡 **Interview tip:** The `predict_linear` function is the capacity planning secret weapon in Grafana. It fits a linear regression to the last N hours of data and projects forward. `predict_linear(metric[1h], 7*24*3600)` means: "based on the last hour's trend, where will this metric be in 7 days?" If projected utilization crosses 80%, it's time to add nodes. The practical limitation: linear projection works well for steady growth but misses sudden spikes (seasonal traffic, viral events). Combine with business knowledge ("we double traffic every Black Friday") for real capacity planning.

---

### Q312 — Terraform | Scenario-Based | Advanced

> Use Terraform `for` expressions, `flatten`, and `setproduct` to generate all combinations of teams/environments/services for S3 buckets and IAM roles.

#### Key Points to Cover:
```hcl
# Input variable — complex nested structure:
variable "teams" {
  type = map(object({
    environments = list(string)
    services     = list(string)
  }))
  default = {
    "platform" = {
      environments = ["dev", "staging", "prod"]
      services     = ["api", "worker"]
    }
    "payment" = {
      environments = ["dev", "prod"]
      services     = ["gateway", "processor"]
    }
  }
}

# Goal: create S3 bucket for every team/env/service combination:
# platform-dev-api, platform-dev-worker, platform-staging-api,
# platform-prod-api, payment-dev-gateway, payment-prod-gateway, etc.

# Step 1: Generate all combinations using for + flatten:
locals {
  # Flatten nested structure to list of objects:
  all_combinations = flatten([
    for team_name, team in var.teams : [
      for environment in team.environments : [
        for service in team.services : {
          team        = team_name
          environment = environment
          service     = service
          key         = "${team_name}-${environment}-${service}"
        }
      ]
    ]
  ])

  # Convert to map (keyed by combination key) for for_each:
  combinations_map = {
    for combo in local.all_combinations :
    combo.key => combo
  }
}

# Preview:
output "all_combinations_preview" {
  value = keys(local.combinations_map)
  # ["payment-dev-gateway", "payment-dev-processor",
  #  "payment-prod-gateway", "payment-prod-processor",
  #  "platform-dev-api", "platform-dev-worker", ...]
}

# Step 2: Create S3 buckets for all combinations:
resource "aws_s3_bucket" "team_buckets" {
  for_each = local.combinations_map

  bucket = "company-${each.value.team}-${each.value.environment}-${each.value.service}"

  tags = {
    Team        = each.value.team
    Environment = each.value.environment
    Service     = each.value.service
    ManagedBy   = "terraform"
  }
}

# Step 3: IAM roles with naming convention:
resource "aws_iam_role" "team_roles" {
  for_each = local.combinations_map

  name = "role-${each.value.team}-${each.value.environment}-${each.value.service}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
      Action = "sts:AssumeRole"
      Condition = {
        StringEquals = {
          "aws:RequestedRegion": var.aws_region
        }
      }
    }]
  })

  tags = {
    Team        = each.value.team
    Environment = each.value.environment
  }
}

# Step 4: S3 bucket policy per role (each role can only access its bucket):
resource "aws_iam_role_policy" "s3_access" {
  for_each = local.combinations_map

  name = "s3-access-${each.value.team}-${each.value.environment}-${each.value.service}"
  role = aws_iam_role.team_roles[each.key].id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
      Resource = [
        aws_s3_bucket.team_buckets[each.key].arn,
        "${aws_s3_bucket.team_buckets[each.key].arn}/*"
      ]
    }]
  })
}

# BONUS: setproduct alternative (when lists are independent, not nested in a map):
# If teams, environments, and services were separate lists:
locals {
  teams_list        = ["platform", "payment"]
  environments_list = ["dev", "staging", "prod"]
  services_list     = ["api", "worker"]

  # setproduct creates cartesian product:
  all_via_setproduct = {
    for combo in setproduct(local.teams_list, local.environments_list, local.services_list) :
    "${combo[0]}-${combo[1]}-${combo[2]}" => {
      team        = combo[0]
      environment = combo[1]
      service     = combo[2]
    }
  }
  # Creates ALL combinations regardless of which services belong to which teams
  # Use flatten when teams have different environments/services
  # Use setproduct when all dimensions are independent
}
```

> 💡 **Interview tip:** The key distinction: **`flatten`** is for when your input is a nested map where each team has DIFFERENT environments/services (custom combination per team). **`setproduct`** is for when you want ALL possible combinations of independent lists (every team × every env × every service). In the real world, teams usually have different service sets, making `flatten` the more realistic approach. The two-step pattern — flatten to list, then `for combo in list : combo.key => combo` to convert to map — is the standard pattern for `for_each` with complex inputs. The key insight: `for_each` requires a map or set, never a list of objects.

---

### Q313 — Git | Scenario-Based | Advanced

> Implement Conventional Commits: lint commit messages in CI, automatic versioning with semantic-release, CHANGELOG generation, git tag creation in GitHub Actions.

#### Key Points to Cover:
```
Conventional Commits format:
  <type>(<scope>): <description>

  Types:
    feat:     new feature → MINOR version bump
    fix:      bug fix → PATCH version bump
    feat!:    breaking change → MAJOR version bump
    docs:     documentation only
    style:    formatting (no logic change)
    refactor: code restructuring
    test:     adding tests
    chore:    maintenance (deps, config)

  Examples:
    feat(payments): add PayPal integration
    fix(auth): fix JWT expiry validation
    feat!: remove deprecated v1 API
    docs(api): update authentication guide

Step 1 — Commit lint on every PR (GitHub Actions):
  # .github/workflows/commit-lint.yml
  name: Commit Lint
  on:
    pull_request:
      types: [opened, synchronize, reopened, edited]
  
  jobs:
    commitlint:
      runs-on: ubuntu-latest
      steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # need full history for commit range
      
      - name: Install commitlint
        run: |
          npm install -g @commitlint/cli @commitlint/config-conventional
      
      - name: Create commitlint config
        run: |
          echo "module.exports = { extends: ['@commitlint/config-conventional'] };" \
            > commitlint.config.js
      
      - name: Lint commit messages
        run: |
          # Lint all commits in this PR:
          npx commitlint \
            --from ${{ github.event.pull_request.base.sha }} \
            --to ${{ github.event.pull_request.head.sha }} \
            --verbose

Step 2 — semantic-release configuration:
  # .releaserc.json:
  {
    "branches": ["main"],
    "plugins": [
      "@semantic-release/commit-analyzer",      // determine version bump
      "@semantic-release/release-notes-generator",  // generate changelog
      "@semantic-release/changelog",             // update CHANGELOG.md
      [
        "@semantic-release/git",                 // commit CHANGELOG.md
        {
          "assets": ["CHANGELOG.md"],
          "message": "chore(release): ${nextRelease.version} [skip ci]"
        }
      ],
      "@semantic-release/github"                 // create GitHub release + tags
    ]
  }

Step 3 — GitHub Actions release workflow:
  # .github/workflows/release.yml
  name: Release
  on:
    push:
      branches: [main]
  
  jobs:
    release:
      runs-on: ubuntu-latest
      permissions:
        contents: write      # create releases, push tags, write CHANGELOG.md
        issues: write        # comment on issues closed by this release
        pull-requests: write # comment on PRs merged in this release
        id-token: write      # for npm provenance (if publishing npm package)
      steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0       # semantic-release needs full history
          persist-credentials: false  # use custom token for push
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
      
      - name: Install semantic-release
        run: |
          npm install -g \
            semantic-release \
            @semantic-release/changelog \
            @semantic-release/git \
            @semantic-release/github \
            @semantic-release/commit-analyzer \
            @semantic-release/release-notes-generator
      
      - name: Run semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npx semantic-release

Step 4 — What semantic-release does automatically:
  1. Analyzes commits since last release using @semantic-release/commit-analyzer
  2. Determines version bump:
     fix: → PATCH (1.0.0 → 1.0.1)
     feat: → MINOR (1.0.0 → 1.1.0)
     feat!: or BREAKING CHANGE: → MAJOR (1.0.0 → 2.0.0)
  3. Generates CHANGELOG.md with grouped commits
  4. Creates git tag: v1.1.0
  5. Creates GitHub Release with release notes
  6. Commits CHANGELOG.md update (with [skip ci] to prevent loop)

CHANGELOG.md example output:
  ## [1.1.0] - 2024-01-15
  
  ### Features
  - **payments**: add PayPal integration (abc1234)
  - **auth**: add OAuth2 PKCE flow (def5678)
  
  ### Bug Fixes
  - **auth**: fix JWT expiry validation (ghi9012)
  
  ## [1.0.1] - 2024-01-10
  
  ### Bug Fixes
  - **api**: fix null pointer in /users endpoint (jkl3456)
```

> 💡 **Interview tip:** The `fetch-depth: 0` in `actions/checkout` is critical for semantic-release — it needs the complete git history to determine what changed since the last tag. Without it (default `fetch-depth: 1`), semantic-release can't find previous tags and either fails or creates a version 1.0.0 every time. Also highlight the `[skip ci]` in the commit message for the CHANGELOG update — without it, committing back to main triggers the CI pipeline again, creating an infinite loop of releases.

---

### Q314 — AWS + Kubernetes | Scenario-Based | Advanced

> EKS pod-to-pod throughput between nodes is only 1 Gbps instead of 10 Gbps. Diagnose VPC CNI config, EC2 networking limits, security groups, MTU settings.

#### Key Points to Cover:
```
Baseline diagnosis:
  # Measure current throughput:
  # On pod-1 (server):
  kubectl exec -it pod-1 -- iperf3 -s
  # On pod-2 (different node, client):
  kubectl exec -it pod-2 -- iperf3 -c <pod-1-ip> -t 30 -P 8

  # Check: is same-node communication fast? (rules out app issue)
  # Check: is node-to-node communication fast? (rules out K8s overhead)
  iperf3 between node IPs (not pod IPs) — if ALSO 1 Gbps, it's EC2/network issue

Possible root causes:

1. EC2 Instance Type Network Limits (most common):
  Each EC2 instance type has a maximum network bandwidth
  t3.medium: up to 5 Gbps (burstable)
  m5.large: up to 10 Gbps (but shared with all traffic)
  
  # Check: are you actually using the right instance type?
  kubectl get nodes -o json | jq '.items[].metadata.labels["beta.kubernetes.io/instance-type"]'
  
  # Check: are you hitting burst limit?
  aws cloudwatch get-metric-statistics \
    --namespace AWS/EC2 \
    --metric-name NetworkIn \
    --dimensions Name=InstanceId,Value=i-xxxx
  
  # ENA (Elastic Network Adapter) supports enhanced networking:
  ethtool -i eth0  # should show: driver ena (not ixgbevf)
  # If not ena: enable enhanced networking on instance

2. VPC CNI — too few ENIs/IPs per node:
  # vpc-cni (aws-node) manages ENI attachment
  # If ENIs are exhausted, pods can't get IPs → scheduling fails (not a throughput issue)
  # But: wrong ENI configuration can cause indirect issues
  
  kubectl describe ds aws-node -n kube-system | grep -A5 "Env:"
  # Check: WARM_ENI_TARGET, MINIMUM_IP_TARGET, MAXIMUM_ENI
  
  # For throughput: each ENI adds bandwidth capacity
  # Multiple ENIs don't aggregate bandwidth by default in VPC CNI

3. MTU mismatch (major throughput killer):
  # Pod-to-pod MTU must account for encapsulation overhead
  # EKS default: 9001 bytes (jumbo frames)
  # If MTU too small → packet fragmentation → throughput drops
  
  # Check pod MTU:
  kubectl exec -it pod-1 -- ip link show eth0
  # Should show: mtu 9001 (for EKS with VPC CNI)
  # If showing 1500: MTU not configured for jumbo frames
  
  # Test actual MTU (ping with specific size):
  kubectl exec -it pod-1 -- ping -M do -s 8972 <pod-2-ip>
  # 8972 + 28 (header) = 9000 bytes
  # If this FAILS: somewhere in path drops large packets
  
  # Fix: ensure EC2 VPC has jumbo frames enabled
  # EKS: VPC CNI sets MTU automatically if nodes support it

4. Security Group rules causing CPU overhead:
  # Stateful SG tracking adds CPU overhead at high packet rates
  # At 1 Gbps with many small packets: CPU can become bottleneck
  
  # Check: is CPU saturated during iperf3 test?
  kubectl exec -it pod-1 -- top
  # Also: kernel softirq (si) in top output — high si = network interrupt overhead

5. Placement Group not used:
  # EC2 Cluster Placement Group: physical proximity for lowest latency + highest throughput
  # Without it: instances might be far apart in the datacenter
  # With cluster placement group: 10 Gbps between instances guaranteed (not shared)
  
  aws ec2 create-placement-group --group-name eks-high-perf --strategy cluster
  # Add to EKS node group launch template

6. Network policy overhead (Calico/Cilium):
  # If using network policies: each packet checked against policy rules
  # iptables-based policies: iptables rule evaluation per packet
  # At high throughput: iptables becomes bottleneck
  
  # Check: how many iptables rules?
  iptables -L | wc -l
  # Thousands of rules → CPU overhead per packet
  
  # Fix: Cilium eBPF (O(1) lookup vs O(N) iptables scan)

Systematic approach:
  1. Measure node-to-node bandwidth (bypasses K8s/CNI)
  2. If node-to-node is also slow → EC2 instance type, placement, MTU
  3. If node-to-node is fast but pod-to-pod is slow → CNI overhead (MTU, iptables)
  4. Check: MTU (jumbo frames configured?)
  5. Check: CPU usage during test (interrupt bottleneck?)
  6. Check: EC2 network baseline vs burst bandwidth
```

> 💡 **Interview tip:** Start with the simplest diagnosis: **iperf3 between node IPs (not pod IPs)**. If node-to-node is also 1 Gbps, the problem is below Kubernetes — EC2 instance networking limits, MTU configuration, or placement. If node-to-node is 10 Gbps but pod-to-pod is 1 Gbps, the problem is in the CNI layer — MTU mismatch is the most common culprit. A 9001-byte MTU allows jumbo frames; if something in the path doesn't support it, packets are fragmented, and throughput drops dramatically. The MTU ping test (`ping -M do -s 8972`) is the definitive test for MTU issues.

---

### Q315 — All Topics | System Design | Advanced

> Design a self-healing infrastructure platform: automatic detection, remediation, circuit breakers, escalation, audit log, runbook automation.

#### Key Points to Cover:
```
Platform Architecture — 5 Layers:

Layer 1 — Detection:
  Prometheus + AlertManager:
    → Metrics: pod restarts, error rate, CPU/memory, disk, queue depth
    → Alert rules: codified failure conditions
    → Alert grouping: related alerts bundled
    
  Custom health checks:
    → Kubernetes readiness/liveness probes (application-level)
    → Synthetic monitoring: cron jobs checking endpoints
    → Database connectivity checks
    → Queue depth checks

  Kubernetes events:
    → Node pressure events
    → Pod eviction events
    → OOMKill events

Layer 2 — Decision Engine:
  Rule-based engine (not ML initially):
    → Input: alert type + severity + affected resource + history
    → Output: remediation action + confidence level
    
  Rules examples:
    PodCrashLoopBackOff → restart pod (if <3 restarts in 1h)
    NodeMemoryPressure → evict low-priority pods
    HighCPU + healthy replicas → scale HPA
    DatabaseConnectionExhausted → restart connection pool
    QueueBacklog > threshold → scale workers (KEDA)
    
  Rule evaluation:
    is_known_pattern(alert) ? execute_runbook(alert) : escalate_to_human(alert)

Layer 3 — Execution:
  Argo Workflows or custom controller:
    → Runs remediation steps as DAG
    → Each step: idempotent action
    → Timeout on each step
    → Rollback if step fails
    
  Runbook steps example (PodCrashLoop):
    1. Get pod logs → store for debugging
    2. Check recent deployments (new version?)
    3. If new deployment → initiate rollback
    4. Else → delete pod (kubelet recreates it)
    5. Wait for pod to be Running
    6. Run health check
    7. If healthy → close alert, log success
    8. If still failing → escalate to human

  Actions available:
    kubectl rollout undo → rollback deployment
    kubectl delete pod → force restart
    aws autoscaling set-desired-capacity → scale ASG
    kubectl scale → scale K8s deployment
    aws rds failover-db-cluster → RDS failover
    aws sqs purge-queue → drain dead queue
    Execute runbook script → custom remediation

Layer 4 — Circuit Breakers:
  Prevent remediation from making things worse:
  
  Circuit breaker rules:
    → Max remediations per resource per hour: 3
    → If 3 failed remediations in 1 hour → OPEN circuit → escalate
    → If remediation itself causes errors → OPEN circuit
    → Global: if >5 remediations firing simultaneously → PAUSE all (cascade failure?)
    → Time-based: no auto-remediation during deployments (suppress window)
    
  Implementation:
    Redis counter: remediation_count:{resource}:{hour}
    If count >= threshold: skip remediation, escalate instead
    
  Example:
    Pod "payment-api" restarts 3 times
    Runbook triggers 3 times (pod delete + restart)
    Still crashing → circuit opens → PagerDuty alert to on-call engineer

Layer 5 — Escalation:
  PagerDuty integration:
    → Auto-remediation attempted → failed → page on-call
    → Include: what was tried, what failed, current state
    → Severity based on: service criticality + duration
    
  Runbook documentation:
    → Each escalation: link to Confluence runbook
    → Engineer sees: "attempted X, Y, Z. These failed. Here's the runbook."
    → Reduces MTTR even when human needed

Audit Log:
  Every remediation event stored in:
    → Elasticsearch (searchable)
    → DynamoDB (queryable by resource/time)
    
  Log entry includes:
    {
      "timestamp": "2024-01-15T14:35:00Z",
      "alert": "PodCrashLoopBackOff",
      "resource": "payment-api-7d9f8b-xkz4p",
      "namespace": "production",
      "runbook": "pod-restart-v2",
      "steps_executed": ["get_logs", "delete_pod", "health_check"],
      "result": "SUCCESS",
      "duration_seconds": 45,
      "circuit_breaker_state": "CLOSED",
      "escalated": false
    }
  
  Compliance report:
    "Show all auto-remediation actions on production resources in Q1"
    "How many times did the payment service self-heal vs require human intervention?"
    "What percentage of incidents are resolved without human involvement?"

Tools and integration:
  Detection:    Prometheus + custom exporters + kube-state-metrics
  Alerting:     AlertManager → Webhook receiver
  Decision:     Custom controller (Go/Python) OR Argo Events + Argo Workflows
  Execution:    Argo Workflows (DAG-based runbooks)
  Circuit BK:   Redis counters + Go controller
  Escalation:   PagerDuty API
  Audit:        Elasticsearch + Kibana
  Runbooks:     Git repo of YAML runbook definitions
```

> 💡 **Interview tip:** The circuit breaker layer is what separates a thoughtful self-healing design from a naive "just restart everything" approach. Without circuit breakers, a platform that auto-remediates can amplify failures: restart a crashed pod → crashes again → restart → crash → 30 restarts in 10 minutes → node overwhelmed. The key operational insight: **auto-remediation should escalate when it fails**, not retry infinitely. Also highlight the **audit log requirement** — many organizations need it for compliance ("who authorized this action?"). Frame your answer: auto-remediation doesn't remove human oversight, it handles known-pattern failures automatically so engineers focus on novel problems.

---

## Coverage Summary — Q271–Q315 Enhanced

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

*Part 7 Enhanced — DevOps/SRE Interview Questions Q271–Q315*
*Format matches Q1–Q45 Enhanced: Key Points + Interview Tips for every question*
