# DevOps / SRE / Cloud Interview Questions (526–575)
### 50 Most Commonly Asked — 100% Unique, Not Repeated From Q1–Q525

> Every question here is **brand new** — zero overlap with Q1–Q525.
> Covers the most frequently asked gaps: Prometheus Architecture, 502/503/504,
> Aurora, Route53, IAM Permission Boundaries, Prometheus Operator,
> SonarQube, Ansible Vault, Terragrunt, 12-Factor App, Postmortem,
> EC2 Placement Groups, API Gateway, ECR Lifecycle, and more.
> All questions include **Key Points to Cover** + **💡 Interview Tips**.

---

## Topic Coverage Map

| Area | Questions | Gap Filled |
|---|---|---|
| **Prometheus Architecture** | Q526–Q530 | 🔴 Complete architecture, data model, Operator, ServiceMonitor |
| **HTTP Error Codes** | Q531–Q533 | 🔴 502 vs 503 vs 504 conceptual + ALB + Nginx |
| **AWS — Aurora** | Q534–Q535 | 🔴 Aurora vs RDS, Aurora Serverless |
| **AWS — Route53** | Q536–Q537 | 🔴 Routing policies, health checks |
| **AWS — IAM** | Q538–Q539 | 🔴 Permission Boundaries, IAM deep dive |
| **AWS — API Gateway** | Q540–Q541 | 🔴 REST vs HTTP vs WebSocket, throttling |
| **AWS — EC2** | Q542–Q543 | 🔴 Placement Groups, EC2 instance types |
| **AWS — ECR** | Q544 | 🔴 Lifecycle policies |
| **AWS — SQS Deep Dive** | Q545–Q546 | 🔴 FIFO vs Standard, DLQ internals |
| **CI/CD** | Q547–Q549 | 🔴 SonarQube, Nexus/Artifactory, GitHub Reusable Workflows |
| **Terraform** | Q550–Q551 | 🔴 Terragrunt, Terraform Cloud |
| **Ansible** | Q552–Q553 | 🔴 Ansible Vault, Dynamic Inventory |
| **Kubernetes** | Q554–Q557 | 🔴 PV Access Modes, Velero, Helmfile, Kustomize deep dive |
| **SRE Culture** | Q558–Q561 | 🔴 12-Factor App, Postmortem, Toil, On-call best practices |
| **Observability** | Q562–Q565 | 🔴 Prometheus Pushgateway deep, PromQL label ops, Mimir, Logstash Grok |
| **Networking** | Q566–Q568 | 🔴 gRPC deep, WebSocket on AWS, Load balancer algorithms |
| **Docker** | Q569–Q570 | 🔴 Docker Compose, ENTRYPOINT vs CMD |
| **Linux** | Q571–Q572 | 🔴 Linux performance tools, /proc filesystem |
| **AWS — Cost** | Q573–Q574 | 🔴 Savings Plans vs Reserved Instances deep dive, FinOps |
| **Capstone** | Q575 | 🔴 SRE interview system design |

---

## Questions

---

### Q526 — Prometheus Architecture | Conceptual | Advanced

> Explain the **complete internal architecture of a Prometheus server**. Walk through every component — from how Prometheus discovers targets to how a metric ends up stored, queried, and how an alert reaches Alertmanager.
>
> Cover: **data model**, **service discovery**, **target manager**, **scrape loop**, **relabeling pipeline**, **TSDB**, **PromQL engine**, **rule manager**, **notifier**, and **web handler**.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — architecture, internals sections

#### Key Points to Cover in Your Answer:

**Prometheus Data Model — foundation of everything:**
```
A sample(data point) is one measurement at one moment in time.
Structure: (timestamp, value), Example: (1710500000, 1547)
1710500000 → Unix timestamp (when Prometheus scraped, 1547 → metric value (stored as float64)

A time series is all samples belonging to the same metric + label combination.
Example metric: http_requests_total{method="GET", status="200", service="payment"}
Prometheus scrapes every 15 seconds:
(10:00:00, 1547)
(10:00:15, 1551)
(10:00:30, 1558)
(10:00:45, 1563)

Labels are KEY to Prometheus:
- Identify the time series uniquely
- Enable filtering and aggregation
- High cardinality labels = millions of time series = OOM
```

**Complete architecture flow:**
```
┌─────────────────────────────────────────────────────────────┐
│                    PROMETHEUS SERVER                         │
│                                                             │
│  Service Discovery ──► Target Manager ──► Scrape Loop       │
│         │                                     │             │
│  (k8s_sd, ec2_sd,                    relabel_configs        │
│   file_sd, static)                   metric_relabel_configs │
│                                             │               │
│                                          ┌──▼──┐            │
│                                          │ WAL │            │
│                                          └──┬──┘            │
│                                          ┌──▼──┐            │
│                                   TSDB   │Head │ (in memory)│
│                                          │Block│            │
│                                          └──┬──┘            │
│                                    compact  │               │
│                                          ┌──▼──┐            │
│                                          │2hr  │ (on disk)  │
│                                          │Block│            │
│                                          └─────┘            │
│                                                             │
│  Rule Manager ──► Notifier ──► Alertmanager                 │
│  (every eval_interval:                                      │
│   recording rules → TSDB                                    │
│   alerting rules → check                                    │
│   → fire → Notifier queue)                                  │
│                                                             │
│  PromQL Engine ◄── HTTP API (Grafana, Web UI)               │
└─────────────────────────────────────────────────────────────┘
```

**1. Service Discovery:**
```yaml
# Prometheus supports multiple SD mechanisms:
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:       # k8s_sd: watch K8s API
      - role: pod
    # Also: ec2_sd, consul_sd, file_sd, dns_sd, static_configs
```

**2. Target Manager:**
```
Maintains list of all active scrape targets
Refreshes from service discovery every sync_period (default: 5m)
Each target = {address, labels, metadata}
```

**3. Scrape Loop (Retrieval):**
```
Every scrape_interval (default: 1m):
1. HTTP GET http://<target>/metrics
2. Parse OpenMetrics/text exposition format
3. Apply metric_relabel_configs (drop/rename/keep metrics)
4. Store samples in WAL (Write Ahead Log) then Head Block
```

**4. TSDB (Time Series Database):**
```
WAL (Write Ahead Log):
  → All new samples written here FIRST
  → Protects against data loss on crash
  → Replayed on startup

Head Block (in-memory, 2h window):
  → Active time series kept in RAM
  → Allows fast writes and recent queries
  → Chunks of 120 samples each

Persistent Blocks (on disk, 2h each):
  → Head block compacted to disk every 2h
  → Further compacted: 2h → 6h → 24h → 72h
  → Immutable once written
  → Default retention: 15 days
```

**5. PromQL Engine:**
```
Query API request → parse PromQL expression → 
scan relevant blocks → return time series → 
apply functions/aggregations → return result
```

**6. Rule Manager:**
```
Every evaluation_interval (default: 1m):
Recording rules → compute → store back in TSDB as new metrics
Alerting rules → compute → if result non-empty → create Alert
```

**7. Notifier:**
```
Alerts queue → batch → HTTP POST to Alertmanager /api/v2/alerts
Retries on failure
Multiple Alertmanager instances supported (HA)
```

> 💡 **Interview tip:** The **WAL (Write Ahead Log)** is the most important reliability concept — it ensures no data is lost even if Prometheus crashes. On restart, Prometheus replays the WAL to recover the Head Block. The separation of **Head Block (in-memory, fast writes)** and **Persistent Blocks (on-disk, immutable, compressed)** is why Prometheus is both fast and efficient. The compaction cascade (2h → 6h → 24h → 72h) reduces the number of files over time and improves query performance for historical data.

---

### Q527 — Prometheus Architecture | Conceptual | Advanced

> Explain the **Prometheus data model** in depth — what is a **time series**, **sample**, **metric name**, and **labels**?
>
> What is **cardinality** and why is high cardinality dangerous? What is the **OpenMetrics exposition format**? What is the difference between **`/metrics`** (pull) and **PushGateway** (push) and when is each appropriate?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — data model, cardinality sections

#### Key Points to Cover in Your Answer:

**Data Model:**
```
Metric name:  http_requests_total
Labels:       {method="GET", status="200", handler="/api/users"}
Timestamp:    1705276800000 (Unix ms)
Value:        15234.0

Full time series identity:
http_requests_total{method="GET",status="200",handler="/api/users"}

Every unique combination of labels = separate time series stored separately
```

**Exposition format (what /metrics returns):**
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 15234
http_requests_total{method="POST",status="201"} 3891
http_requests_total{method="GET",status="404"} 42
# EOF
```

**Cardinality explained:**
```
Low cardinality:  status_code label → values: 200, 201, 400, 404, 500
                  = 5 time series per metric (safe)

High cardinality: user_id label → values: 1, 2, 3...1,000,000 users
                  = 1,000,000 time series per metric (DANGEROUS)

High cardinality effects:
1. RAM: each active time series = ~3-4KB in Head Block
   1M series × 4KB = 4GB RAM just for one metric
2. Scrape timeout: /metrics endpoint generates millions of lines → slow
3. Query slowness: aggregations scan all series
4. Crash: OOM kill

Safe label values: bounded, low-count, not user-generated
Unsafe: user_id, request_id, IP address, UUID
```

**Pull vs Push model:**
```
PULL (Prometheus default):
  + Prometheus controls scrape frequency
  + Easy to detect dead targets (scrape fails)
  + No agent needed, just expose /metrics
  + Centralized config in Prometheus
  - Hard for short-lived jobs (job finishes before scrape)
  - Hard across firewalls (Prometheus needs network access to targets)

PUSH (PushGateway):
  + Works for batch jobs, cron jobs
  + Works behind firewalls
  - Prometheus cannot detect if job died (metrics persist forever)
  - Single point of failure
  - Anti-pattern for long-running services (use pull instead)

Rule: Use PULL for everything. Use PUSH only for batch jobs.
```

> 💡 **Interview tip:** The most critical cardinality rule to mention: **never use user_id, session_id, request_id, or IP addresses as label values** — they create unbounded cardinality. The correct approach is to put high-cardinality data in **log fields** (ELK/Loki) and use **Prometheus only for aggregated metrics**. When an interviewer asks "how do you find which specific user caused high load?" — the answer is "use tracing (X-Ray/Jaeger) or logs, not Prometheus metrics".

---

### Q528 — Prometheus Architecture | Conceptual | Advanced

> Explain **Prometheus Operator** — what problem does it solve and how does it work?
>
> What are **ServiceMonitor**, **PodMonitor**, and **PrometheusRule** CRDs? How do they differ from manually configuring `scrape_configs` in `prometheus.yml`? Write a complete ServiceMonitor for a payment service.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — Prometheus Operator, ServiceMonitor sections

#### Key Points to Cover in Your Answer:

**Problem Prometheus Operator solves:**
```
Without Operator:
  Add new service → manually edit prometheus.yml → reload Prometheus
  → ConfigMap change → Prometheus restart → brief monitoring gap
  → 50 teams, 200 services → ops team bottleneck

With Prometheus Operator:
  Add new service → developer creates ServiceMonitor CRD → done
  → Operator watches ServiceMonitor CRDs
  → Automatically updates Prometheus config
  → No Prometheus restart needed (hot reload)
  → Teams self-service their own monitoring
```

**Prometheus Operator CRDs:**
```
Prometheus:        defines a Prometheus instance (replicas, storage, rules)
Alertmanager:      defines an Alertmanager instance
ServiceMonitor:    tells Prometheus how to scrape a Service
PodMonitor:        tells Prometheus how to scrape Pods directly (no Service needed)
PrometheusRule:    defines alerting + recording rules
ScrapeConfig:      advanced scrape config (Prometheus Operator 0.64+)
```

**ServiceMonitor vs PodMonitor:**
```
ServiceMonitor:
  → Selects Services by label → discovers endpoints behind the Service
  → Best for: standard services with a Kubernetes Service resource
  → Port specified by name (not number)

PodMonitor:
  → Selects Pods directly (bypasses Service layer)
  → Best for: pods without a Service, DaemonSets, sidecar scrapers
  → More flexible label selection
```

**Complete ServiceMonitor example:**
```yaml
# Step 1: Service must have matching labels and named port
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: production
  labels:
    app: payment-service
    team: payments           # ← ServiceMonitor selector matches this
spec:
  selector:
    app: payment-service
  ports:
  - name: http-metrics       # ← named port (ServiceMonitor references by name)
    port: 9090
    targetPort: 9090
---
# Step 2: ServiceMonitor tells Prometheus to scrape this Service
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payment-service
  namespace: production
  labels:
    release: prometheus      # ← must match Prometheus CRD's serviceMonitorSelector
spec:
  selector:
    matchLabels:
      team: payments         # ← select Services with this label
  namespaceSelector:
    matchNames:
    - production             # ← only watch production namespace
  endpoints:
  - port: http-metrics       # ← named port from Service spec
    interval: 30s            # override default scrape_interval
    scrapeTimeout: 10s
    path: /metrics           # default
    scheme: http
    relabelings:             # add extra labels to scraped metrics
    - sourceLabels: [__meta_kubernetes_pod_name]
      targetLabel: pod
    - sourceLabels: [__meta_kubernetes_namespace]
      targetLabel: namespace
    metricRelabelings:       # drop metrics you don't need
    - sourceLabels: [__name__]
      regex: go_.*           # drop all Go runtime metrics
      action: drop
---
# Step 3: Prometheus CRD must select this ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  serviceMonitorSelector:
    matchLabels:
      release: prometheus    # ← matches ServiceMonitor label above
  serviceMonitorNamespaceSelector:
    matchLabels:
      monitoring: "true"     # ← only watch namespaces with this label
```

**PrometheusRule for alerting:**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: payment-alerts
  namespace: production
  labels:
    release: prometheus      # must match Prometheus ruleSelector
spec:
  groups:
  - name: payment.rules
    rules:
    - alert: PaymentHighErrorRate
      expr: rate(http_requests_total{status=~"5..",service="payment"}[5m]) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Payment service error rate above 5%"
```

> 💡 **Interview tip:** The **most common Prometheus Operator misconfiguration** is label mismatch — the ServiceMonitor has label `release: prometheus` but the Prometheus CRD's `serviceMonitorSelector` doesn't match it, so Prometheus never picks up the ServiceMonitor. Always check: ServiceMonitor labels → match → Prometheus `serviceMonitorSelector`. Similarly for `ruleSelector` and PrometheusRule. Run `kubectl get prometheuses -n monitoring -o yaml | grep -A5 serviceMonitorSelector` to debug.

---

### Q529 — Prometheus Architecture | Scenario-Based | Advanced

> Explain **PromQL label manipulation** — `label_replace()`, `label_join()`, `group_left`, `group_right`.
>
> Write PromQL queries using these for real-world scenarios:
> - Join node CPU metrics with node instance labels from another metric
> - Add a synthesized label from an existing label value using regex
> - Many-to-one join — enrich pod metrics with deployment-level information

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — PromQL advanced, label manipulation sections

#### Key Points to Cover in Your Answer:

**`label_replace()` — create/modify labels with regex:**
```promql
# Syntax: label_replace(v, dst_label, replacement, src_label, regex)

# Example 1: Extract just the hostname from pod name
# pod name: payment-service-7d9f8b-xkz4p → want: payment-service
label_replace(
  up{job="kubernetes-pods"},
  "service_name",             # destination label
  "$1",                       # replacement (capture group 1)
  "pod",                      # source label
  "^(.+)-[a-z0-9]+-[a-z0-9]+$"  # regex with capture group
)

# Example 2: Normalize region labels
# us-east-1 → us_east_1 (replace hyphens with underscores)
label_replace(
  aws_ec2_cpu_utilization,
  "region_normalized",
  "$1",
  "region",
  "(.+)"
)
# Then use string replacement in Grafana template variable
```

**`label_join()` — combine multiple labels:**
```promql
# Syntax: label_join(v, dst_label, separator, src_label_1, src_label_2, ...)

# Create a human-readable identifier from namespace + pod
label_join(
  kube_pod_info,
  "full_name",    # destination label
  "/",            # separator
  "namespace",    # source labels to join
  "pod"
)
# Result: full_name="production/payment-service-7d9f8b"
```

**`group_left` / `group_right` — many-to-one joins:**
```promql
# Problem: container metrics have pod label,
#          but you want deployment label (from kube_pod_owner)
# One deployment has MANY pods → many-to-one join

# Step 1: Get pod → deployment mapping
# kube_pod_owner{owner_kind="ReplicaSet"} → gives pod + replicaset

# Step 2: Join container CPU with deployment info
container_cpu_usage_seconds_total
* on(pod, namespace) group_left(owner_name)  # group_left: many-to-one
  kube_pod_owner{owner_kind="ReplicaSet"}

# group_left: LEFT side has many rows per join key (many pods per deployment)
# group_right: RIGHT side has many rows per join key (rare)
# on(): which labels to join on
# group_left(owner_name): carry over owner_name from right side

# Real-world: CPU usage per Deployment (not per pod)
sum by(owner_name, namespace) (
  container_cpu_usage_seconds_total
  * on(pod, namespace) group_left(owner_name)
    kube_pod_owner{owner_kind="ReplicaSet"}
)
```

**`topk()`, `bottomk()`, `sort()` — ranking:**
```promql
# Top 5 pods by memory usage
topk(5, container_memory_usage_bytes{container!=""})

# Bottom 3 nodes by available disk
bottomk(3, node_filesystem_avail_bytes{mountpoint="/"})

# Sort all services by error rate (descending)
sort_desc(
  sum by(service) (rate(http_requests_total{status=~"5.."}[5m]))
)
```

> 💡 **Interview tip:** `group_left` is the most powerful and most misunderstood PromQL operator. The key mental model: **`group_left` means "the left side has more rows"**. In Kubernetes monitoring, you almost always use `group_left` because you have many pods (left) joining to one deployment (right). The `on()` clause specifies which labels must match — forgetting this causes cross-product explosions. Always test joins in Prometheus UI before putting in dashboards.

---

### Q530 — Prometheus Architecture | Scenario-Based | Advanced

> Explain **Grafana Mimir** — what is it and how does it compare to **Thanos** for Prometheus long-term storage?
>
> When would you choose Mimir over Thanos? What is the **Prometheus Operator + Mimir** integration and how does `remote_write` connect them?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — Mimir, long-term storage, remote_write sections

#### Key Points to Cover in Your Answer:

**Why long-term storage is needed:**
```
Default Prometheus: 15 days retention
Problems:
  - Can't see trends over months/quarters
  - Prometheus restarts lose recent data (before WAL flush)
  - Single instance = single point of failure
  - Can't query across multiple Prometheus instances
```

**Thanos vs Mimir:**

| Feature | Thanos | Mimir |
|---|---|---|
| Architecture | Sidecar + Store Gateway | Microservices (fully distributed) |
| Installation complexity | High (many components) | Medium (single binary or microservices) |
| Scalability | Good | Excellent (designed for massive scale) |
| Query performance | Good | Better (sharding built-in) |
| Storage | Object storage (S3/GCS) | Object storage (S3/GCS) |
| Multi-tenancy | Limited | Built-in (tenant isolation) |
| Maintained by | Open source community | Grafana Labs (actively developed) |
| Best for | Existing Thanos users, flexible setups | New deployments, Grafana-centric stacks |

**Mimir architecture:**
```
Prometheus (remote_write) ──► Mimir Distributor
                                    │
                         ┌──────────┴──────────┐
                         ▼                     ▼
                    Mimir Ingester        Mimir Ingester
                         │                     │
                         └──────────┬──────────┘
                                    ▼
                              Object Storage
                              (S3/GCS/Azure)
                                    │
                         ┌──────────┴──────────┐
                         ▼                     ▼
                   Mimir Store-             Mimir
                   Gateway                 Compactor
                         │
                         ▼
                   Mimir Querier ◄── Grafana (PromQL queries)
```

**Remote write config:**
```yaml
# prometheus.yml — send all metrics to Mimir
remote_write:
  - url: http://mimir-distributor:8080/api/v1/push
    headers:
      X-Scope-OrgID: production    # Mimir tenant ID
    queue_config:
      capacity: 10000
      max_shards: 50               # parallel write workers
      max_samples_per_send: 5000
      batch_send_deadline: 5s
    write_relabel_configs:          # filter what to send remotely
      - source_labels: [__name__]
        regex: "go_.*"
        action: drop               # don't send Go runtime metrics to long-term
```

**When to choose each:**
```
Choose Thanos if:
  - Already running Thanos
  - Need flexible query federation across many Prometheus
  - Complex multi-cluster setup already in place

Choose Mimir if:
  - New setup (Mimir is simpler to start with)
  - Using Grafana Cloud (Mimir is the backend)
  - Need multi-tenancy (different teams isolated)
  - Very high metric volume (Mimir scales better)
  - Want single binary for dev/staging
```

> 💡 **Interview tip:** Mimir is increasingly replacing Thanos in new deployments because it was purpose-built by Grafana Labs to address Thanos pain points. The key selling point: Mimir can run as a **single binary** for development and a **microservices deployment** for production using the same codebase. Also mention that **Grafana Cloud** runs on Mimir under the hood — so if your team uses Grafana Cloud for metrics, you are already using Mimir.

---

### Q531 — HTTP Error Codes | Conceptual | Advanced

> Explain the difference between **502 Bad Gateway**, **503 Service Unavailable**, and **504 Gateway Timeout** from the perspective of an **AWS ALB** and **Nginx Ingress Controller**.
>
> For each error code: what exactly causes it, what ALB/Nginx does when it encounters this situation, and what is the first thing you check to diagnose it?

📁 **Reference:** `nawab312/AWS` — ALB error codes, Nginx Ingress sections

#### Key Points to Cover in Your Answer:

**The key distinction — who is at fault:**
```
502 = Backend connected but gave BAD/no response
      "I reached you but you replied with garbage"

503 = No healthy backend EXISTS to connect to
      "I have no one to send this request to"

504 = Backend connected but TOO SLOW to respond
      "I reached you but you took too long"
```

**502 Bad Gateway:**
```
ALB perspective:
  ALB connected to backend target (TCP handshake succeeded)
  BUT backend:
    - Sent invalid HTTP response
    - Closed connection before sending response  
    - Sent response with wrong format

Common causes:
  - App crashed AFTER accepting connection (race condition during deploy)
  - Backend running wrong protocol (HTTPS on HTTP port)
  - App responding to health check but crashing on real requests
  - Pod shutting down, received request, closed connection

First check:
  → ALB access logs: target_status_code field
  → kubectl logs <pod> (app crash logs)
  → Is the backend actually sending valid HTTP?

Nginx Ingress 502:
  - upstream prematurely closed connection
  - upstream sent invalid header
  - First check: nginx ingress controller logs
    kubectl logs -n ingress-nginx <ingress-pod> | grep "upstream"
```

**503 Service Unavailable:**
```
ALB perspective:
  ALB has NO healthy targets in target group
  All targets are failing health checks OR
  Target group is empty (no instances registered)

Common causes:
  - All pods failing readiness probe (bad deployment)
  - All pods restarting simultaneously (OOMKill)
  - Target group empty (autoscaling scaled to 0)
  - Health check path wrong (returns 404 → ALB marks unhealthy)
  - Security group blocks health check port

First check:
  → AWS Console: Target Group → Targets tab → check Health Status
  → aws elbv2 describe-target-health --target-group-arn <arn>
  → kubectl get pods → are pods Running/Ready?
  → kubectl describe pod → readiness probe failures?

Nginx Ingress 503:
  - No endpoints available for the service
  - Service has no ready pods
  - First check: kubectl get endpoints <service>
    → if NONE: no pods matching selector
```

**504 Gateway Timeout:**
```
ALB perspective:
  ALB connected to backend
  Backend took longer than ALB idle timeout (default 60s) to respond

Common causes:
  - Backend processing slow query (DB timeout)
  - Backend waiting on downstream service (cascade timeout)
  - Backend CPU throttled (hitting CPU limits)
  - Large file upload/download exceeds timeout

First check:
  → ALB access logs: target_processing_time field
    If close to 60s → timeout issue (increase or fix backend)
  → Check backend application logs for slow operations
  → CloudWatch: TargetResponseTime metric histogram

Nginx Ingress 504:
  - proxy_read_timeout exceeded (default 60s)
  - First check: nginx ingress annotation:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
```

**Quick diagnostic matrix:**
```
Error  │ Target registered? │ Target healthy? │ Backend responding?
───────┼────────────────────┼─────────────────┼───────────────────
503    │ Maybe No           │ NO              │ N/A (never reached)
502    │ Yes                │ Yes             │ YES but invalid
504    │ Yes                │ Yes             │ YES but too slow
```

> 💡 **Interview tip:** The single fastest way to distinguish 502 vs 503 in production: check **Target Group health** in AWS console. If targets show **"unhealthy"** → 503. If targets show **"healthy"** but you're getting errors → 502 or 504. For 502 vs 504: check `target_processing_time` in ALB access logs — if it's close to 60 seconds → 504. If it's near-zero → 502 (backend rejected immediately).

---

### Q532 — HTTP Error Codes | Troubleshooting | Advanced

> Your Nginx Ingress Controller is returning **502 Bad Gateway** errors for a specific microservice but not others. The pods are healthy, pass readiness probes, and respond correctly via `kubectl port-forward`.
>
> Walk through the complete diagnosis — covering keepalive mismatch, request timeout, large headers, upstream connect errors, and buffer size issues.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — Nginx Ingress 502 troubleshooting sections

#### Key Points to Cover in Your Answer:

**Step 1 — Read Nginx Ingress controller logs:**
```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=100 | \
  grep "upstream\|error\|502"

# Key error messages and meanings:
# "upstream prematurely closed connection"  → backend closed before response done
# "upstream connect() failed"              → TCP connection failed
# "connect() failed (111: Connection refused)" → backend not listening
# "no live upstreams while connecting to upstream" → all pods unhealthy
# "upstream sent invalid header"           → backend sent non-HTTP response
```

**Step 2 — Keepalive mismatch (most common cause):**
```
Problem:
  Nginx keeps TCP connection open to backend (keepalive)
  Backend closes connection after N requests or N seconds
  Nginx sends request on "stale" closed connection → 502

Fix — add keepalive annotations:
nginx.ingress.kubernetes.io/upstream-keepalive-connections: "32"
nginx.ingress.kubernetes.io/upstream-keepalive-requests: "100"
nginx.ingress.kubernetes.io/upstream-keepalive-timeout: "60"

OR tell Nginx not to use keepalive for this service:
nginx.ingress.kubernetes.io/upstream-keepalive-connections: "0"
```

**Step 3 — Request/response timeout:**
```yaml
# Default: 60 seconds — may not be enough for slow backends
annotations:
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
```

**Step 4 — Large headers (413/502 from header overflow):**
```yaml
# Default buffer too small for JWT tokens, large cookies
annotations:
  nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"
  nginx.ingress.kubernetes.io/proxy-buffers-number: "4"
  nginx.ingress.kubernetes.io/large-client-header-buffers: "4 16k"
```

**Step 5 — Large request body:**
```yaml
# Default: 1MB — increase for file uploads
annotations:
  nginx.ingress.kubernetes.io/proxy-body-size: "50m"
```

**Step 6 — Pod startup race condition:**
```yaml
# Pods marked Ready but app not fully initialized
# Fix: proper readinessProbe
readinessProbe:
  httpGet:
    path: /readyz       # endpoint that returns 200 ONLY when truly ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

**Step 7 — Verify with curl through Nginx:**
```bash
# Test directly through ingress (reproduces the 502)
curl -v -H "Host: payment.company.com" http://<ingress-ip>/api/endpoint

# Compare with direct pod access (no ingress)
kubectl port-forward pod/payment-pod 8080:8080 &
curl -v http://localhost:8080/api/endpoint
# If direct works but ingress fails → Nginx config issue
```

> 💡 **Interview tip:** The **keepalive mismatch** is the most common cause of intermittent 502s that are hard to reproduce — they happen randomly because the stale connection is only hit occasionally. The telltale sign is that the 502 rate is low (1-2%) and not correlated with load. The immediate fix is to disable upstream keepalive for the affected service. The proper fix is to ensure the backend's keepalive timeout is LONGER than Nginx's keepalive timeout.

---

### Q533 — HTTP Error Codes | Scenario-Based | Advanced

> Your AWS ALB is showing **503 errors** spiking to 100% during a deployment. The deployment uses a **rolling update** strategy. Everything was working before the deployment started.
>
> Explain why rolling deployments cause 503s and how to configure ALB, Kubernetes, and the application to achieve **zero-downtime deployments** with zero 503s.

📁 **Reference:** `nawab312/AWS` — ALB health checks, deregistration delay, Kubernetes rolling update sections

#### Key Points to Cover in Your Answer:

**Why rolling updates cause 503s:**
```
Timeline of a bad rolling update:
T+0s:  New pod starts, passes readiness probe
T+1s:  Old pod receives SIGTERM (starts 30s graceful shutdown)
T+2s:  ALB still sends traffic to old pod (doesn't know it's shutting down)
T+3s:  Old pod stops accepting new connections
T+4s:  Requests to old pod → Connection refused → ALB returns 502/503
T+30s: Old pod terminated
...
Repeat for each pod in rolling update
```

**Fix 1 — ALB Deregistration Delay:**
```
When pod is removed from target group:
  ALB sends "draining" signal
  ALB waits deregistration_delay (default 300s!) before stopping traffic
  During wait: ALB still sends some traffic → 502

Fix: Reduce deregistration delay to match app shutdown time
  aws elbv2 modify-target-group-attributes \
    --target-group-arn <arn> \
    --attributes Key=deregistration_delay.timeout_seconds,Value=30
```

**Fix 2 — preStop hook to delay termination:**
```yaml
# Tell Kubernetes to wait before sending SIGTERM
# This gives ALB time to deregister the pod
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 15"]
# Pod receives SIGTERM → preStop runs (15s sleep) → then SIGTERM to app
# During 15s sleep: ALB finishes deregistering pod
# After 15s: app starts graceful shutdown
# Result: zero dropped requests
```

**Fix 3 — Proper terminationGracePeriodSeconds:**
```yaml
spec:
  terminationGracePeriodSeconds: 60  # must be > preStop duration + app shutdown time
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 15"]  # wait for ALB deregistration
```

**Fix 4 — Readiness probe that fails before SIGTERM:**
```yaml
# Better approach: app marks itself not-ready on shutdown signal
# Kubernetes removes from endpoints → ALB stops sending traffic
# THEN app shuts down gracefully

readinessProbe:
  httpGet:
    path: /readyz
    port: 8080
  periodSeconds: 5
  failureThreshold: 1   # fail fast when app starts shutting down

# In application code:
# On SIGTERM: set ready = false → /readyz returns 503
# Wait 10s for connections to drain
# Then shutdown
```

**Fix 5 — minReadySeconds to validate new pods:**
```yaml
spec:
  minReadySeconds: 10     # pod must be Ready for 10s before old pod removed
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0   # NEVER remove old pod before new is ready
```

**Complete zero-downtime configuration:**
```yaml
spec:
  minReadySeconds: 15
  terminationGracePeriodSeconds: 60
  strategy:
    rollingUpdate:
      maxUnavailable: 0     # never take pod down before replacement ready
      maxSurge: 1           # add one new pod before removing old
  containers:
  - lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 20"]
    readinessProbe:
      httpGet:
        path: /readyz
        port: 8080
      periodSeconds: 5
      failureThreshold: 1
      successThreshold: 3   # must pass 3 times before considered ready
```

> 💡 **Interview tip:** The **preStop sleep** is the single most impactful fix for zero-downtime deployments on Kubernetes with ALB — it costs 15-20 seconds per pod during rollout but eliminates all 503s. Without it, there is an unavoidable race condition between Kubernetes removing the pod from endpoints and ALB finishing deregistration. The sleep bridges this gap. Also mention `maxUnavailable: 0` — this ensures you never have fewer pods than desired, only more (via maxSurge), which maintains full capacity during rollout.

---

### Q534 — AWS Aurora | Conceptual | Advanced

> Explain **Amazon Aurora** — what makes it different from standard **RDS PostgreSQL/MySQL**? What is the **Aurora storage architecture** and why is it faster and more resilient than standard RDS?
>
> What is **Aurora Serverless v2**? When would you choose Aurora over standard RDS and when would you NOT?

📁 **Reference:** `nawab312/AWS` — Aurora, Aurora Serverless, RDS comparison sections

#### Key Points to Cover in Your Answer:

**Aurora vs Standard RDS Architecture:**
```
Standard RDS:
  - One primary instance + EBS volume (tightly coupled)
  - Read replica = separate EBS copy → replication lag
  - Failover = DNS change to standby (60-120 seconds)
  - Storage = single AZ EBS (Multi-AZ = synchronous to standby)

Aurora:
  - Compute (instance) SEPARATED from storage
  - Shared distributed storage across 6 copies in 3 AZs
  - Storage auto-grows in 10GB increments (no pre-provisioning)
  - Replication at storage layer (not at compute layer)
  - Failover: 30 seconds (not 120s like RDS)
```

**Aurora Storage Architecture:**
```
Aurora Storage = distributed, fault-tolerant, self-healing

6 copies across 3 AZs (2 per AZ):
  - Writes: quorum of 4/6 copies (tolerates 2 AZ failures for writes)
  - Reads:  quorum of 3/6 copies (tolerates 3 AZ failures for reads)
  - Data automatically repaired if a copy fails

Write path:
  1. Application writes to Aurora primary
  2. Aurora sends only REDO LOG to storage layer (not full page writes)
  3. 4/6 storage nodes acknowledge
  4. Return success to application
  5. Storage nodes apply logs asynchronously

Why faster than RDS:
  - Reduces network I/O by 7/8 (only redo logs, not full pages)
  - Parallel writes across distributed storage
  - No double-write buffer (MySQL optimization not needed)
```

**Aurora Read Replicas vs RDS Read Replicas:**
```
RDS Read Replica:
  - Full data copy on separate EBS
  - Replication via binlog (asynchronous)
  - Lag: 10ms to minutes
  - Each replica = full storage cost

Aurora Read Replica:
  - SHARED same storage as primary
  - Replication lag: typically < 10ms (often < 1ms)
  - Adding replica = no additional storage cost
  - Up to 15 replicas (vs 5 for RDS)
  - Replica can be promoted to primary in < 30s
```

**Aurora Serverless v2:**
```
Automatically scales compute:
  - Scale range: 0.5 to 128 ACUs (Aurora Capacity Units)
  - Scales in increments of 0.5 ACU
  - Scales up in seconds (not minutes like v1)
  - Scales to zero when idle (after configurable pause time)

Use cases:
  ✅ Variable/unpredictable workloads
  ✅ Dev/staging (scale to zero = cost savings)
  ✅ Multi-tenant apps with bursty customers
  ❌ Sustained high-throughput workloads (provisioned cheaper)
  ❌ Latency-critical apps (scale-up adds ~1s latency)
```

**When Aurora vs RDS:**
```
Choose Aurora when:
  ✅ Need < 30s failover (banking, payments)
  ✅ Multiple read replicas needed (> 5)
  ✅ Storage > 16TB (Aurora auto-grows to 128TB)
  ✅ Global database (Aurora Global spans regions)
  ✅ Variable load (Aurora Serverless v2)

Choose Standard RDS when:
  ✅ Cost is primary concern (Aurora ~20% more expensive)
  ✅ Oracle or SQL Server (Aurora only supports MySQL/PostgreSQL)
  ✅ Need specific PostgreSQL extensions not supported by Aurora
  ✅ Steady, predictable workload (provisioned RDS = cheaper)
```

> 💡 **Interview tip:** The most impressive Aurora fact to mention: **Aurora does not write full data pages to storage — only redo logs**. This is why Aurora is 5x faster than standard MySQL on the same hardware. The storage layer reconstructs pages from the redo log. This also means Aurora backup is near-instant (no data to copy — just log metadata) and point-in-time recovery is precise to the second.

---

### Q535 — AWS Aurora | Scenario-Based | Advanced

> Your Aurora PostgreSQL cluster is experiencing **high replication lag** and **read replica falling behind**. You also notice that **failover is taking longer than expected** (3+ minutes vs the advertised 30 seconds).
>
> Walk through diagnosing and fixing each issue, and explain what **Aurora Global Database** is and when you would use it.

📁 **Reference:** `nawab312/AWS` — Aurora replication, failover, Global Database sections

#### Key Points to Cover in Your Answer:

**Aurora Replica Lag diagnosis:**
```bash
# Check replica lag metric
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name AuroraReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=my-replica \
  --period 60 --statistics Maximum

# Common causes of Aurora replica lag:
# 1. Large transactions on primary (long-running writes)
# 2. Replica instance undersized vs primary
# 3. Heavy read workload on replica consuming its resources

# Fix 1: Ensure replica is same or larger instance class as primary
# Fix 2: Reduce write transaction size on primary
# Fix 3: Add more replicas (up to 15) to distribute read load
```

**Slow failover diagnosis:**
```bash
# Aurora failover should be < 30s — if 3+ minutes, investigate:

# Cause 1: Application not respecting DNS TTL
# Aurora failover = DNS change from primary to replica
# If app caches DNS for 5 minutes → 5 minute outage!
# Fix: Set DNS TTL to 5 seconds in application:
jdbc:postgresql://cluster.cluster-xyz.us-east-1.rds.amazonaws.com:5432/db
# Use cluster endpoint (not instance endpoint)

# Cause 2: Application not retrying connections
# After failover, existing connections are dropped
# Fix: implement connection retry with exponential backoff

# Cause 3: RDS Proxy not configured
# RDS Proxy caches connections and handles failover transparently
# Reduces failover impact from 30s to < 5s
aws rds create-db-proxy \
  --db-proxy-name aurora-proxy \
  --engine-family POSTGRESQL \
  --target-group-name default

# Cause 4: No read replica in cluster
# If no replica exists → Aurora must promote a new instance (~3min)
# Always maintain at least one replica for fast failover
```

**Aurora Global Database:**
```
What it is:
  Single Aurora cluster spanning MULTIPLE AWS regions
  Primary region: read/write
  Secondary regions: read-only replicas (up to 5)
  Replication lag: typically < 1 second (uses dedicated replication infrastructure)

Failover (managed):
  1. Detach secondary region from global cluster
  2. Promote it to standalone primary
  3. RTO: typically < 1 minute
  4. RPO: typically < 5 seconds

Use cases:
  ✅ Globally distributed applications needing low read latency
  ✅ Disaster recovery across regions (RPO < 5s)
  ✅ Regulatory: data must be readable in specific regions
  ✅ Read scaling across regions

vs Multi-AZ:
  Multi-AZ: HA within same region (automatic failover, same endpoint)
  Global: DR across regions (manual failover, read in all regions)
```

> 💡 **Interview tip:** The **DNS caching problem** is the most commonly missed Aurora failover issue. Applications using JDBC or other database drivers often cache DNS results for minutes. After Aurora failover, the DNS record changes but the application keeps connecting to the old (now replica/failed) instance. The fix is to use the **cluster endpoint** (not instance endpoint) and configure DNS TTL properly, or better — use **RDS Proxy** which completely abstracts away failover from the application.

---

### Q536 — AWS Route53 | Conceptual | Advanced

> Explain **AWS Route53 routing policies** — what are the 7 routing policies and when would you use each?
>
> What is the difference between **active-active** and **active-passive** failover? Write a Route53 configuration for a **multi-region active-active setup** with automatic failover.

📁 **Reference:** `nawab312/AWS` — Route53, routing policies, DNS failover sections

#### Key Points to Cover in Your Answer:

**7 Route53 Routing Policies:**

**1. Simple:**
```
→ One record → one value (or multiple values returned randomly)
→ No health checks, no routing logic
→ Use: single resource, development, static sites
```

**2. Weighted:**
```
→ Route X% to resource A, Y% to resource B
→ Adds up to any total (not necessarily 100)
→ Use: canary deployments, A/B testing, gradual traffic migration
Example: 90% → v1, 10% → v2 during canary release
```

**3. Latency-based:**
```
→ Routes to region with lowest latency FOR THAT USER
→ Route53 measures latency from user location to each region
→ Use: global apps where performance matters
→ NOT the geographically closest — the LOWEST LATENCY
```

**4. Failover:**
```
→ Primary + Secondary records
→ Route53 health checks primary continuously
→ If primary unhealthy → automatically route to secondary
→ Active-passive: only primary receives traffic when healthy
→ Use: DR, active-passive HA
```

**5. Geolocation:**
```
→ Route based on USER's geographic location
→ Country, continent, or US state level
→ Use: content localization, compliance (data sovereignty), language
→ Must set "default" record for unmatched locations
```

**6. Geoproximity (Traffic Flow only):**
```
→ Route based on location of resources AND users
→ Can adjust with "bias" (+/-) to expand/shrink coverage area
→ Use: shift traffic gradually between regions
```

**7. Multi-value answer:**
```
→ Return up to 8 records (like Simple but with health checks)
→ Only returns healthy records
→ Client-side load balancing (client picks one)
→ Use: simple load balancing without a load balancer, better than Simple
```

**Active-Active Multi-Region Setup:**
```
active-active = traffic goes to BOTH regions simultaneously
(vs active-passive = traffic only to primary, secondary is standby)

Route53 config (Latency-based + Health Checks):
```
```bash
# Region 1: us-east-1
aws route53 change-resource-record-sets --hosted-zone-id Z123 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.company.com",
        "Type": "A",
        "SetIdentifier": "us-east-1",
        "Region": "us-east-1",                     # latency routing
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "us-east-1-alb.amazonaws.com",
          "EvaluateTargetHealth": true              # health check via ALB
        }
      }
    }]
  }'

# Region 2: eu-west-1 (same record name, different region)
# Route53 automatically routes each user to lowest-latency healthy region

# Result:
# US users → us-east-1 (low latency)
# EU users → eu-west-1 (low latency)
# If us-east-1 ALB unhealthy → ALL users route to eu-west-1
```

> 💡 **Interview tip:** The most commonly confused pair is **Geolocation vs Latency-based**. Geolocation routes based on WHERE the user IS (their country/continent). Latency-based routes to the LOWEST LATENCY region for that user — which is usually the closest geographically but not always (network path matters). For compliance (GDPR: EU users must stay in EU), use **Geolocation**. For performance (lowest latency worldwide), use **Latency-based**.

---

### Q537 — AWS Route53 | Troubleshooting | Advanced

> Explain **Route53 Health Checks** in depth — what types exist, how do they work, and what is the difference between **endpoint health checks**, **calculated health checks**, and **CloudWatch alarm health checks**?
>
> Your Route53 failover is not working — traffic is not switching to the secondary even though the primary ALB is returning 503s. Walk through diagnosing this.

📁 **Reference:** `nawab312/AWS` — Route53 health checks, failover diagnosis sections

#### Key Points to Cover in Your Answer:

**3 Health Check Types:**
```
1. Endpoint health check:
   → Route53 sends HTTP/HTTPS/TCP requests to your endpoint
   → From: 15 global Route53 health checker locations
   → Healthy if: X of 18 checkers report healthy
   → Interval: 30s (standard) or 10s (fast, extra cost)
   → String matching: optional (check for specific text in response)

2. Calculated health check:
   → Combines multiple health checks with AND/OR logic
   → Use: "healthy if at least 2 of 3 endpoints are healthy"
   → Parent health check aggregates children

3. CloudWatch alarm health check:
   → Healthy/unhealthy based on CloudWatch alarm state
   → Use: when you want custom metric-based health (CPU, error rate)
   → Can alarm on anything CloudWatch monitors
```

**Common failover not working diagnosis:**
```bash
# Step 1: Verify health check is actually failing
# Route53 console → Health checks → check status
aws route53 get-health-check-status \
  --health-check-id <hc-id>
# Look for: StatusReport → Status (Healthy/Unhealthy)
# Look for: CheckedTime (when last checked)

# Step 2: Check if health check is associated with the record
aws route53 list-resource-record-sets \
  --hosted-zone-id Z123 \
  --query "ResourceRecordSets[?Name=='api.company.com']"
# Must show: "HealthCheckId": "<hc-id>"
# If HealthCheckId missing → failover NEVER happens (no check = always healthy)

# Step 3: Check security groups
# Route53 health checkers have specific IP ranges
# Must allow inbound from: 54.183.255.128/26, 54.228.16.0/26, etc.
# (full list: https://ip-ranges.amazonaws.com/ip-ranges.json → filter "ROUTE53_HEALTHCHECKS")

# Step 4: Check health check path returns correct status code
curl -v https://api.company.com/health
# Must return 2xx for Route53 to consider healthy
# If returns 200 with error body → Route53 thinks it's healthy (checks status only by default)
# Fix: enable string matching to check response body

# Step 5: DNS TTL still propagating
aws route53 get-hosted-zone --id Z123 \
  --query "HostedZone.Config.PrivateZone"
# Check record TTL — if TTL=300 and failover happened 1 min ago
# → clients still have cached old record for 4 more minutes
# Fix: set TTL=60 or TTL=30 for failover records
```

> 💡 **Interview tip:** The most common Route53 failover bug: **health check not attached to the DNS record**. Without `HealthCheckId` on the record, Route53 never checks health and never fails over — it just keeps the record active forever regardless of endpoint status. Always verify the health check ID is actually on the failover record. The second most common issue: **security groups blocking Route53 health checker IPs** — Route53 checkers connect from specific AWS IP ranges that must be allowed through security groups and NACLs.

---

### Q538 — AWS IAM | Conceptual | Advanced

> Explain **IAM Permission Boundaries**. What problem do they solve that regular IAM policies cannot?
>
> What is the difference between a **Permission Boundary**, **SCP (Service Control Policy)**, and **Resource-based policy** in terms of where they apply and how they interact during policy evaluation? Write a Permission Boundary that prevents a developer role from escalating privileges.

📁 **Reference:** `nawab312/AWS` — IAM Permission Boundaries, policy evaluation sections

#### Key Points to Cover in Your Answer:

**What Permission Boundaries solve:**
```
Problem: You want to allow developers to create IAM roles
         for their Lambda functions or ECS tasks.
         But: if developer can create ANY IAM role with ANY permissions
         → developer can create admin role → privilege escalation!

Permission Boundary solution:
         You allow developers to create IAM roles
         BUT: every role they create must have your boundary attached
         → They can create roles, but those roles can never exceed
           the permissions defined in the boundary
         → Privilege escalation prevented ✅
```

**Policy evaluation order:**
```
For an IAM principal to perform an action, ALL of these must ALLOW it:
(and NONE of the explicit DENYs must fire)

1. SCPs (Service Control Policies) — org-level ceiling
2. Identity-based policies (what the user/role has)
3. Permission Boundary — applied to user/role, limits #2
4. Resource-based policies (S3 bucket policy, KMS key policy)
5. Session policies (for assumed roles via STS)

Effective permissions = intersection of all that apply
```

**Permission Boundary example:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowServiceActions",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "dynamodb:*",
        "lambda:*",
        "logs:*",
        "cloudwatch:*",
        "xray:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowPassRoleOnlyWithBoundary",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": [
            "lambda.amazonaws.com",
            "ecs-tasks.amazonaws.com"
          ]
        }
      }
    },
    {
      "Sid": "AllowCreateRoleWithBoundary",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:AttachRolePolicy"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary":
            "arn:aws:iam::123456789012:policy/DeveloperBoundary"
        }
      }
    },
    {
      "Sid": "DenyPrivilegeEscalation",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:DeletePermissionsBoundary",
        "iam:PutUserPermissionsBoundary"
      ],
      "Resource": "*"
    }
  ]
}
```

**Key rule:** Developer can create roles but ONLY if those roles have `DeveloperBoundary` attached. Any role they create inherits the boundary → cannot exceed boundary permissions → no privilege escalation.

> 💡 **Interview tip:** Permission Boundaries are a **delegated administration** mechanism — not a security control in the traditional sense. They don't grant permissions, they only limit them. The mental model: identity policy = "what you CAN do", permission boundary = "the ceiling of what you can do". The effective permissions are the intersection. This is why a boundary alone grants nothing — you still need identity policies to actually allow actions within the boundary.

---

### Q539 — AWS IAM | Scenario-Based | Advanced

> Explain the complete **IAM policy evaluation logic** — in what order does AWS evaluate policies and when does an explicit DENY win vs an implicit DENY?
>
> Walk through this scenario: an IAM user has an identity policy allowing `s3:*`, the S3 bucket has a resource policy allowing the user `s3:GetObject`, there's an SCP denying `s3:DeleteObject`, and a Permission Boundary allowing only `s3:GetObject` and `s3:PutObject`. What can the user actually do?

📁 **Reference:** `nawab312/AWS` — IAM policy evaluation, explicit vs implicit deny sections

#### Key Points to Cover in Your Answer:

**Policy evaluation flowchart:**
```
1. Is there an explicit DENY anywhere? → YES → DENY (full stop)
2. Is this a request within an AWS Organization?
   → YES → Does SCP allow it? → NO → DENY
3. Is there a resource-based policy that allows it?
   → YES (and it's the same account) → ALLOW (can skip identity policy check)
4. Does identity-based policy allow it? → NO → DENY
5. Is there a Permission Boundary?
   → YES → Does boundary allow it? → NO → DENY
6. Is there a session policy? → YES → Does it allow? → NO → DENY
7. All checks passed → ALLOW
```

**Scenario walkthrough:**
```
User has:
  Identity policy:    s3:* (allows everything on S3)
  S3 bucket policy:   Allow s3:GetObject for this user
  SCP:                Deny s3:DeleteObject
  Permission Boundary: Allow s3:GetObject, s3:PutObject only

What can the user do?

s3:GetObject:
  SCP: no deny → ✅
  Identity policy: s3:* → ✅
  Boundary: allows GetObject → ✅
  Result: ALLOWED ✅

s3:PutObject:
  SCP: no deny → ✅
  Identity policy: s3:* → ✅
  Boundary: allows PutObject → ✅
  Result: ALLOWED ✅

s3:DeleteObject:
  SCP: EXPLICIT DENY → ❌
  Result: DENIED (SCP wins, even though identity policy allows it)

s3:ListBucket:
  SCP: no deny → ✅
  Identity policy: s3:* → ✅
  Boundary: does NOT allow ListBucket → ❌
  Result: DENIED (boundary acts as ceiling)

s3:DeleteBucket:
  SCP: no deny for DeleteBucket → ✅
  Identity policy: s3:* → ✅
  Boundary: does NOT allow DeleteBucket → ❌
  Result: DENIED (boundary)

Summary: user can ONLY do GetObject and PutObject
```

**Explicit vs Implicit DENY:**
```
Explicit DENY: a policy statement with "Effect": "Deny"
  → Always wins, overrides any Allow
  → Even if 10 other policies allow it, one explicit Deny wins

Implicit DENY: no policy grants the permission
  → Default state for all actions
  → Not the same as explicit Deny (cannot be "overridden" — there's nothing to override)
  → Cross-account: resource policy must grant access (implicit deny applies cross-account even with identity policy)
```

> 💡 **Interview tip:** The **cross-account IAM rule** is a common gotcha: when accessing resources CROSS-ACCOUNT, you need BOTH an identity policy on the requester AND a resource policy on the resource (or assume-role trust). Within the same account, resource policies can grant access without identity policies. This is why S3 bucket policies can allow public access without any IAM user policies.

---

### Q540 — AWS API Gateway | Conceptual | Advanced

> Explain **AWS API Gateway** — what is the difference between **REST API**, **HTTP API**, and **WebSocket API**?
>
> When would you choose each? What are the key differences in features, pricing, and latency? What is **API Gateway throttling** and how does it protect your backend?

📁 **Reference:** `nawab312/AWS` — API Gateway, REST vs HTTP API, throttling sections

#### Key Points to Cover in Your Answer:

**REST API vs HTTP API vs WebSocket API:**

| Feature | REST API | HTTP API | WebSocket API |
|---|---|---|---|
| Protocol | HTTP/1.1 | HTTP/1.1 + HTTP/2 | WebSocket |
| Latency | Higher (~6ms) | Lower (~1ms) | N/A (persistent) |
| Price | $3.50/million | $1.00/million | $0.80/million + connection fee |
| Auth | IAM, Cognito, Lambda authorizer, API keys | IAM, Cognito, Lambda authorizer, JWT | IAM, Lambda authorizer |
| Request/response transform | ✅ Full (mapping templates) | ❌ No (passthrough only) | ❌ No |
| Usage plans + API keys | ✅ Yes | ❌ No | ❌ No |
| Private integrations | ✅ VPC Link | ✅ VPC Link | ❌ No |
| WAF integration | ✅ Yes | ✅ Yes | ❌ Limited |
| Mock integration | ✅ Yes | ❌ No | ❌ No |

**When to choose:**
```
REST API:
  → Need request/response transformation (mapping templates)
  → Need usage plans and API keys for B2B APIs
  → Need mock integrations for testing
  → Need WAF, resource policies, client certificates

HTTP API:
  → OIDC/JWT auth (Cognito, Auth0)
  → Cost-sensitive (70% cheaper than REST)
  → Lower latency requirement
  → Simple proxy to Lambda or HTTP backend
  → Most new APIs should use HTTP API

WebSocket API:
  → Real-time bidirectional communication
  → Chat applications, live dashboards, gaming
  → IoT device communication
  → Long-lived connections
```

**Throttling — protecting your backend:**
```
Two levels of throttling:

Account-level default:
  → 10,000 req/s burst, 5,000 req/s steady state per region

Stage-level:
  → Set per API stage (dev/prod)
  → Overrides account default for that API

Method-level:
  → Set per individual endpoint
  → Most granular control

Usage Plans (REST API only):
  → Assign to API key
  → Define: throttle rate (per second) + burst + quota (per day/week/month)
  → Use: B2B APIs, different tiers (free/basic/premium)

What happens when throttled:
  → API Gateway returns 429 Too Many Requests
  → Backend Lambda/ECS NEVER receives the request
  → Protects backend from being overwhelmed
```

> 💡 **Interview tip:** For any new API Gateway use case, the answer should almost always be **HTTP API** (not REST API) — it is 70% cheaper, has lower latency, supports all modern auth methods, and handles the vast majority of use cases. The only reasons to still use REST API are: request/response transformation (mapping templates), usage plans with API keys, or mock integrations. Knowing this distinction immediately signals you have up-to-date AWS knowledge.

---

### Q541 — AWS API Gateway | Troubleshooting | Advanced

> Your API Gateway is returning **429 Too Many Requests** to legitimate users even though the backend Lambda is not being throttled. Walk through diagnosing and fixing API Gateway throttling issues.
>
> Also explain **API Gateway caching** — what it is, how it works, and when you should use it.

📁 **Reference:** `nawab312/AWS` — API Gateway throttling, caching, 429 troubleshooting sections

#### Key Points to Cover in Your Answer:

**Diagnosing 429 errors:**
```bash
# Step 1: Check which level is throttling
# CloudWatch metrics:
# 4XXError (client errors including 429)
# ThrottleCount (explicit throttle rejections)

aws cloudwatch get-metric-statistics \
  --namespace AWS/ApiGateway \
  --metric-name ThrottleCount \
  --dimensions Name=ApiName,Value=MyAPI Name=Stage,Value=prod \
  --period 60 --statistics Sum

# Step 2: Identify the scope
# Account level? Stage level? Method level?
aws apigateway get-stage \
  --rest-api-id <api-id> \
  --stage-name prod | jq '.defaultRouteSettings'

# Step 3: Check method-level throttling
aws apigateway get-method \
  --rest-api-id <api-id> \
  --resource-id <resource-id> \
  --http-method GET | jq '.methodIntegration.requestParameters'
```

**Fixing throttling:**
```bash
# Increase stage-level throttle
aws apigateway update-stage \
  --rest-api-id <api-id> \
  --stage-name prod \
  --patch-operations \
    op=replace,path=defaultRouteSettings/throttlingBurstLimit,value=5000 \
    op=replace,path=defaultRouteSettings/throttlingRateLimit,value=2000

# Request account-level limit increase
aws service-quotas request-service-quota-increase \
  --service-code apigateway \
  --quota-code L-8A5B8E43 \
  --desired-value 20000

# If using Usage Plans — check API key quota not exhausted
aws apigateway get-usage \
  --usage-plan-id <plan-id> \
  --key-id <api-key-id> \
  --start-date 2024-01-15 \
  --end-date 2024-01-15
```

**API Gateway Caching:**
```
What it is:
  → API Gateway caches backend responses
  → Subsequent identical requests served from cache (no Lambda/backend call)
  → Cache key: URL path + query params + headers (configurable)
  → TTL: 0-3600 seconds (default 300s)

When to use:
  ✅ Responses don't change frequently (reference data, catalogs)
  ✅ Backend is expensive to call (RDS, slow API)
  ✅ High traffic, repetitive requests
  ❌ User-specific data (must invalidate per-user)
  ❌ Real-time data
  ❌ POST/PUT/DELETE (only GET cached by default)

Cost: ~$0.020/hour per GB of cache capacity
Cache invalidation:
  → Per-request: header Cache-Control: max-age=0 (requires IAM permission)
  → Flush entire stage cache: Console or API call
```

> 💡 **Interview tip:** API Gateway caching and Lambda Provisioned Concurrency solve different problems that look the same — **high latency**. Caching reduces calls to Lambda entirely. Provisioned Concurrency eliminates Lambda cold starts. If you see high p99 latency on an API: (1) check if cache hit rate is low → enable/tune caching, (2) check if Lambda cold starts spike → enable provisioned concurrency. Use CloudWatch `CacheHitCount` and `CacheMissCount` metrics to measure cache effectiveness.

---

### Q542 — AWS EC2 | Conceptual | Advanced

> Explain **EC2 Placement Groups** — what are the 3 types (**Cluster**, **Spread**, **Partition**) and when would you use each?
>
> What is the trade-off between **Cluster placement** (performance) and **Spread placement** (availability)? Which placement group type would you use for a **Hadoop/Spark cluster** vs a **high-frequency trading system** vs a **critical 3-tier web application**?

📁 **Reference:** `nawab312/AWS` — EC2 Placement Groups, networking sections

#### Key Points to Cover in Your Answer:

**3 Placement Group Types:**

**1. Cluster Placement Group:**
```
What: All instances placed on same physical rack (or close racks)
      within a single AZ

Network:
  → Instances get 10Gbps or 25Gbps enhanced networking
  → Ultra-low latency between instances (<1ms)
  → High bandwidth inter-instance communication

Limitations:
  → All instances must be in SAME AZ
  → Higher risk: rack failure = ALL instances affected
  → Capacity constraints: must launch all at once or use reserved capacity

Use for:
  ✅ HPC (High Performance Computing) workloads
  ✅ Machine learning training (GPU clusters, all-reduce communication)
  ✅ Tightly coupled parallel processing
  ✅ Low latency, high throughput applications
  ❌ NOT for HA (single point of failure)
```

**2. Spread Placement Group:**
```
What: Each instance placed on DIFFERENT physical hardware (different rack)
      Can span multiple AZs

Availability:
  → Maximum isolation: one hardware failure affects only ONE instance
  → Best for small numbers of critical instances

Limitations:
  → Max 7 instances per AZ per placement group
  → Network performance same as normal (no enhanced networking benefit)

Use for:
  ✅ Small number of critical instances that must not fail together
  ✅ ZooKeeper, etcd, Kafka brokers (must stay up if one fails)
  ✅ Primary + standby database pair
  ❌ NOT for large clusters (7 instance limit)
  ❌ NOT for performance (no network benefit)
```

**3. Partition Placement Group:**
```
What: Instances divided into "partitions" (groups)
      Each partition = separate rack/hardware
      Instances within same partition share hardware
      Different partitions do NOT share hardware

Characteristics:
  → Up to 7 partitions per AZ
  → Hundreds of instances per partition
  → Partition topology visible via metadata (which partition am I in?)

Use for:
  ✅ Large distributed systems: HDFS, HBase, Cassandra
  ✅ Workloads needing rack-awareness (replicate across partitions)
  ✅ Hadoop/Spark clusters (spread data blocks across partitions)
```

**Use case answers:**
```
Hadoop/Spark cluster:
  → PARTITION placement group
  → Can specify rack-aware HDFS replication across partitions
  → Hundreds of instances supported
  → Balance: performance within partition + isolation across partitions

High-Frequency Trading system:
  → CLUSTER placement group
  → Ultra-low latency between matching engine and risk system
  → 10Gbps+ networking for tick data
  → Performance > availability for HFT

3-tier web application (web/app/db):
  → SPREAD placement group
  → Only 3 critical instances (one per tier)
  → Maximum isolation: web server failure doesn't affect app/db
  → Well within 7-instance limit
```

> 💡 **Interview tip:** Placement groups are **free** — there is no charge to use them, only potential cost for the enhanced networking capability on the instances. The most commonly forgotten limitation is the **7-instance-per-AZ limit on Spread groups** — if you need 10 instances with maximum hardware isolation, Spread cannot help you (use Partition instead). Also mention that Cluster groups have **capacity reservation risks** — if you try to launch 100 instances into a Cluster group sequentially, later launches may fail due to capacity constraints on that specific rack.

---

### Q543 — AWS EC2 | Conceptual | Advanced

> Explain **EC2 instance families and types** — what do the letters in `m5.2xlarge` mean? What are the key instance families and their use cases?
>
> What is **Nitro hypervisor** and what performance improvements does it bring? What is **EC2 Enhanced Networking (ENA)** and what are the network performance limits of different instance types?

📁 **Reference:** `nawab312/AWS` — EC2 instance types, Nitro, Enhanced Networking sections

#### Key Points to Cover in Your Answer:

**Instance naming convention:**
```
m  5  .  2  x  large
│  │      │  │  └── Size: nano/micro/small/medium/large/xlarge/2xlarge...
│  │      │  └── "x" = extra (2xlarge = 2× xlarge)
│  │      └── Separator
│  └── Generation: 5 (higher = newer hardware)
└── Family: m (general), c (compute), r (memory), g (GPU), etc.

Qualifiers (may follow family letter):
  a = AMD processor (e.g., m5a)
  g = AWS Graviton ARM processor (e.g., m6g, m7g)
  n = network optimized
  d = NVMe local instance store
  z = high frequency
```

**Key instance families:**
```
General Purpose: m (balanced CPU/RAM)
  m5, m6i, m7i → for most workloads
  t3, t4g → burstable (cheap, good for dev/staging)

Compute Optimized: c (high CPU:RAM ratio)
  c5, c6i, c7g → web servers, batch, gaming, ML inference

Memory Optimized: r (high RAM:CPU ratio)
  r5, r6i → in-memory databases, caching, real-time analytics
  x2idn → extreme memory (up to 6TB RAM)

Storage Optimized: i/d (NVMe local storage)
  i3en, i4i → Cassandra, Hadoop, high IOPS NoSQL
  d3 → dense HDD storage, Hadoop data lakes

GPU: g/p/trn/inf
  g4dn, g5 → ML inference, video transcoding
  p4d, p4de → ML training (A100 GPUs)
  trn1 → AWS Trainium chips (cheaper ML training)
  inf1, inf2 → AWS Inferentia (cheapest ML inference)

ARM/Graviton:
  m7g, c7g, r7g → up to 40% better price/performance vs x86
  Requires: ARM-compatible software (most Linux software works)
```

**Nitro hypervisor:**
```
Traditional hypervisor (Xen): hypervisor competes with instances for CPU
Nitro: hardware-based virtualization
  → All I/O operations offloaded to dedicated Nitro hardware
  → Instance gets near bare-metal performance
  → Security: hardware-enforced isolation between instances
  → Enables: enhanced networking, EFA (Elastic Fabric Adapter), NVMe

Performance improvement:
  → Near-100% CPU dedicated to instance (vs ~90% on Xen)
  → Network: up to 100Gbps (vs ~25Gbps on older Xen instances)
  → EBS: up to 160,000 IOPS (vs 80,000 on older instances)
```

**Enhanced Networking (ENA):**
```
Standard networking: software-based, higher CPU overhead
Enhanced Networking (ENA): hardware-based, lower latency, higher throughput

Performance by instance type (examples):
  t3.medium:    up to 5 Gbps
  m5.xlarge:    up to 10 Gbps
  m5.4xlarge:   up to 10 Gbps
  m5.8xlarge:   10 Gbps dedicated
  m5.16xlarge:  20 Gbps
  m5.24xlarge:  25 Gbps

EFA (Elastic Fabric Adapter):
  → For HPC workloads needing MPI communication
  → Bypass OS networking (OS-bypass)
  → Achieves: up to 400 Gbps on hpc7g, sub-microsecond latency
  → Use: ML training, HPC clusters
```

> 💡 **Interview tip:** For EKS/Kubernetes workloads, **Graviton (m7g, c7g, r7g)** instances are increasingly the default choice — 40% better price/performance and AWS is investing heavily in Graviton. The main consideration is ensuring your container images are built for `linux/arm64`. Multi-arch Docker images (built with `docker buildx`) solve this automatically. In Kubernetes, node selectors or Karpenter node pools can mix x86 and ARM nodes seamlessly.

---

### Q544 — AWS ECR | Conceptual | Advanced

> Explain **Amazon ECR (Elastic Container Registry)** — what features does it provide beyond just storing images?
>
> What are **ECR Lifecycle Policies** and how do you use them to automatically clean up old images? What is **ECR image scanning** and how does it integrate with CI/CD? What is **ECR image replication**?

📁 **Reference:** `nawab312/AWS` — ECR, lifecycle policies, image scanning sections

#### Key Points to Cover in Your Answer:

**ECR key features beyond storage:**
```
1. Image scanning (basic + enhanced with Inspector)
2. Lifecycle policies (auto-delete old images)
3. Cross-region/cross-account replication
4. Image immutability (prevent tag overwrites)
5. Pull through cache (proxy Docker Hub, public ECR)
6. Resource-based policies (control who can push/pull)
7. Encryption at rest (KMS)
8. VPC endpoint (pull images without internet)
```

**ECR Lifecycle Policies:**
```json
// Auto-clean rules — evaluated in priority order (lower = higher priority)
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep only 10 tagged release images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],          // only images tagged v1.0, v2.3, etc.
        "countType": "imageCountMoreThan",
        "countNumber": 10                 // keep latest 10 release tags
      },
      "action": {"type": "expire"}
    },
    {
      "rulePriority": 2,
      "description": "Delete untagged images older than 1 day",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 1
      },
      "action": {"type": "expire"}
    },
    {
      "rulePriority": 3,
      "description": "Keep only 30 days of branch builds",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["sha-"],        // git SHA tags from CI
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 30
      },
      "action": {"type": "expire"}
    }
  ]
}
```

**ECR Image Scanning:**
```bash
# Basic scanning: uses CVE database, runs on push
aws ecr put-image-scanning-configuration \
  --repository-name payment-service \
  --image-scanning-configuration scanOnPush=true

# Enhanced scanning: uses AWS Inspector, continuous scanning
# Detects new CVEs even in images already pushed (not just on push)
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{
    "scanFrequency": "CONTINUOUS_SCAN",
    "repositoryFilters": [{
      "filter": "*",
      "filterType": "WILDCARD"
    }]
  }]'

# In CI/CD: fail build if CRITICAL CVEs found
aws ecr describe-image-scan-findings \
  --repository-name payment-service \
  --image-id imageTag=v1.2.3 \
  --query 'imageScanFindings.findingSeverityCounts'
# If CRITICAL > 0 → fail pipeline
```

**ECR Cross-region replication:**
```bash
# Replicate to DR region automatically
aws ecr put-replication-configuration \
  --replication-configuration '{
    "rules": [{
      "destinations": [{
        "region": "eu-west-1",
        "registryId": "123456789012"
      }],
      "repositoryFilters": [{
        "filter": "production/*",
        "filterType": "PREFIX_MATCH"
      }]
    }]
  }'
# All images pushed to production/* repos automatically replicated
# Use: DR, multi-region deployments, latency reduction
```

> 💡 **Interview tip:** **ECR lifecycle policies** are one of the highest-ROI optimizations teams forget about. Without them, ECR repositories grow indefinitely — we have seen teams spending $500+/month on ECR storage for old images from CI/CD builds. A simple lifecycle policy keeping only 30 days of SHA builds and 10 release tags reduces storage 90%+ with zero operational impact. Always set lifecycle policies when creating new ECR repositories.

---

### Q545 — AWS SQS | Conceptual | Advanced

> Explain the difference between **SQS Standard** and **SQS FIFO** queues. When does ordering and exactly-once delivery actually matter?
>
> What is the **visibility timeout** and what happens if a consumer crashes before deleting the message? What is **long polling** vs **short polling** and which should you always use?

📁 **Reference:** `nawab312/AWS` — SQS Standard vs FIFO, visibility timeout sections

#### Key Points to Cover in Your Answer:

**Standard vs FIFO:**

| Feature | Standard | FIFO |
|---|---|---|
| Ordering | Best-effort (roughly in order) | Strict FIFO within message group |
| Delivery | At-least-once (occasional duplicates) | Exactly-once (deduplication) |
| Throughput | Nearly unlimited | 3,000 msg/s with batching (300 without) |
| Price | Cheaper | ~10% more expensive |
| DLQ support | ✅ Yes | ✅ Yes |
| Content deduplication | ❌ No | ✅ Yes (5-min dedup window) |

**When ordering matters:**
```
Matters:
  - Financial transactions for same account (debit before credit)
  - Event sourcing (events must replay in order)
  - Database CDC (change data capture must be in order)
  - Sequential workflow steps

Does NOT matter:
  - Email notifications (order irrelevant)
  - Image processing (each image independent)
  - Log ingestion (reordering acceptable)
  - Notification fanout (parallel processing fine)
```

**Visibility Timeout:**
```
When consumer receives message:
  1. Message becomes INVISIBLE to other consumers for visibility_timeout period
  2. Consumer processes message
  3. Consumer sends DeleteMessage → message permanently deleted
  4. OR: timeout expires → message becomes VISIBLE again (redelivered)

If consumer crashes:
  → Timeout expires (default 30 seconds)
  → Message reappears in queue
  → Another consumer picks it up
  → This is SQS's resilience mechanism

Important: set visibility_timeout > max processing time
  Too short: message redelivered while still being processed → duplicate processing
  Too long: if consumer crashes, message stuck invisible for too long

Dynamic extension:
  consumer.change_message_visibility(
    receipt_handle=msg.receipt_handle,
    visibility_timeout=60  # extend if processing takes longer than expected
  )
```

**Long polling vs Short polling:**
```
Short polling (default, bad):
  → SQS returns immediately even if queue is empty
  → Returns from subset of servers only
  → Problem: empty responses cost money ($0.40/million requests)
  → Problem: may miss messages on first poll
  → NEVER use short polling in production

Long polling (WaitTimeSeconds=20, always use this):
  → SQS waits up to 20s for a message
  → Only returns when message available (or timeout)
  → Queries all SQS servers (no missed messages)
  → 20x reduction in API calls → 20x cost reduction
  → ALWAYS set WaitTimeSeconds=20

aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123/my-queue \
  --wait-time-seconds 20 \    # long polling
  --max-number-of-messages 10  # batch up to 10 messages
```

> 💡 **Interview tip:** **Long polling is not an optimization — it is the correct way to use SQS**. Short polling (the default) is almost always wrong for production. The cost difference is dramatic: at 100 consumers polling continuously, short polling costs ~$1,500/month in API calls alone. Long polling reduces this to ~$75/month. Always set `WaitTimeSeconds=20` in consumer code or `ReceiveMessageWaitTimeSeconds=20` on the queue attribute.

---

### Q546 — AWS SQS | Scenario-Based | Advanced

> Explain **SQS Dead Letter Queues (DLQ)** in depth — how do they work and what should you do with messages in the DLQ?
>
> Also explain **SQS message deduplication** in FIFO queues and **SQS message groups** — how do they enable parallel processing while maintaining order within a group?

📁 **Reference:** `nawab312/AWS` — SQS DLQ, FIFO deduplication, message groups sections

#### Key Points to Cover in Your Answer:

**Dead Letter Queue (DLQ):**
```
What: separate queue for messages that fail processing repeatedly

How it works:
  1. Message received by consumer
  2. Consumer fails to process (exception, timeout)
  3. Message visibility timeout expires
  4. Message returned to queue (retry #1)
  5. Repeat until maxReceiveCount reached
  6. SQS moves message to DLQ automatically

maxReceiveCount:
  How many times a message can be received before moving to DLQ
  Recommended: 3-5 (allows retries but doesn't loop forever)

aws sqs set-queue-attributes \
  --queue-url <main-queue-url> \
  --attributes '{
    "RedrivePolicy": "{
      \"deadLetterTargetArn\": \"<dlq-arn>\",
      \"maxReceiveCount\": \"3\"
    }"
  }'
```

**What to do with DLQ messages:**
```
1. Alert on DLQ message count (CloudWatch alarm)
   → DLQ messages = production errors requiring attention

2. Inspect messages:
   aws sqs receive-message --queue-url <dlq-url>
   → Read the actual message + SQS metadata
   → Find: why did processing fail?

3. Fix the bug in consumer

4. Redrive messages back to main queue (DLQ redrive):
   aws sqs start-message-move-task \
     --source-arn <dlq-arn> \
     --destination-arn <main-queue-arn>
   → Moves messages back for reprocessing after fix

5. Dead letter queue retention:
   → Set DLQ retention to MAX (14 days) — give yourself time to investigate
   → Main queue retention can be shorter (4 days)
```

**FIFO Message Deduplication:**
```
Problem: network error causes message sent twice
         FIFO queue must deliver exactly once

Content-based deduplication:
  → SHA-256 hash of message body = deduplication ID
  → Within 5-minute window: same hash = same message = deduplicated
  → Enable: ContentBasedDeduplication=true on queue

Message deduplication ID (explicit):
  → You provide unique ID with each send
  → Useful when same payload sent legitimately at different times
  aws sqs send-message \
    --queue-url <fifo-queue-url> \
    --message-body "payment-123-retry" \
    --message-deduplication-id "payment-123-$(date +%s)" \
    --message-group-id "user-456"
```

**Message Groups — parallel processing with order:**
```
Without message groups: FIFO = one consumer at a time (not scalable)

With message groups:
  → Each group processed strictly in order
  → DIFFERENT groups processed in parallel
  → Multiple consumers each handle different groups

Example: payment processing
  message-group-id = customer_id
  → Customer A's payments: processed in order by Consumer 1
  → Customer B's payments: processed in order by Consumer 2
  → Customer C's payments: processed in order by Consumer 3
  → All happening simultaneously = scalable + ordered

aws sqs send-message \
  --message-group-id "customer-456"  # all messages for customer 456 in order
  --message-body '{"action":"debit","amount":100}'
```

> 💡 **Interview tip:** A very common interview question: "how do you handle SQS consumer poison pills?" — messages that always fail and block the queue. The answer is DLQ with `maxReceiveCount=3`. Without DLQ, a bad message keeps cycling forever and prevents other messages from being processed (especially in FIFO queues where one failing group blocks that group). Always configure DLQ when creating any SQS queue — it is the production safety net.

---

### Q547 — CI/CD | Conceptual | Advanced

> Explain **SonarQube** — what is it, what does it analyze, and how do you integrate it into a CI/CD pipeline?
>
> What are **Quality Gates** and what happens when a PR fails a quality gate? What metrics does SonarQube track and what is **technical debt**?

📁 **Reference:** `nawab312/CI_CD` — SonarQube, code quality, CI integration sections

#### Key Points to Cover in Your Answer:

**What SonarQube analyzes:**
```
Static code analysis — examines SOURCE CODE without executing it

Categories:
1. Bugs: code that is likely to behave incorrectly
   e.g., null pointer dereference, resource leak

2. Vulnerabilities: security issues
   e.g., SQL injection, hardcoded credentials, insecure deserialization

3. Code Smells: maintainability issues
   e.g., too-complex functions, duplicated code, dead code

4. Security Hotspots: code that needs manual review
   e.g., cryptographic operations, user input handling

5. Coverage: what % of code is covered by unit tests

6. Duplications: copy-pasted code (increases maintenance burden)

Languages supported: Java, Python, JavaScript, TypeScript, Go,
                     C/C++, C#, Kotlin, Ruby, PHP, and 30+ more
```

**Quality Gates:**
```
Quality Gate = pass/fail threshold for merging code

Default "Sonar Way" gate fails if any of:
  - New bugs > 0
  - New vulnerabilities > 0  
  - New security hotspots not reviewed
  - New code coverage < 80%
  - New duplicated lines > 3%
  - New reliability rating < A
  - New security rating < A

Integration with PR:
  → PR created → CI triggers SonarQube analysis
  → SonarQube sends status back to GitHub
  → If Quality Gate FAILS → PR blocked from merging
  → Developer must fix issues before merging
```

**CI/CD Integration:**
```yaml
# GitHub Actions integration
- name: SonarQube Scan
  uses: SonarSource/sonarqube-scan-action@master
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
  with:
    args: >
      -Dsonar.projectKey=payment-service
      -Dsonar.sources=src
      -Dsonar.tests=tests
      -Dsonar.python.coverage.reportPaths=coverage.xml
      -Dsonar.qualitygate.wait=true    # fail if quality gate fails

- name: Quality Gate check
  uses: SonarSource/sonarqube-quality-gate-action@master
  timeout-minutes: 5
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

**Technical Debt:**
```
Technical debt = estimated time to fix all code smells
SonarQube measures in hours/days

Example output:
  Debt: 3 days 4 hours
  Reliability rating: B (1 bug)
  Security rating: A (0 vulnerabilities)
  Coverage: 76% (4% below gate threshold)
  Duplication: 2.1%

Debt ratio = technical_debt / time_to_develop_project
  A (excellent): 0-5%
  B (good):      6-10%
  C (fair):      11-20%
  D (poor):      21-50%
  E (very poor): >50%
```

> 💡 **Interview tip:** The most practical SonarQube configuration decision is **what to put in the Quality Gate**. "New code" gates (only fail on issues in new/changed code) are much more practical than gates on the entire codebase — you cannot fix 5 years of technical debt before merging a bug fix. Always configure the Quality Gate to focus on **new code only**, then gradually improve the existing codebase separately as a tech debt reduction initiative.

---

### Q548 — CI/CD | Conceptual | Advanced

> Explain **Nexus Repository Manager** and **JFrog Artifactory** — what problems do artifact repositories solve?
>
> What types of artifacts can they store? What is a **proxy repository** vs **hosted repository** vs **group repository**? How do they integrate with Maven, npm, Docker, and Python?

📁 **Reference:** `nawab312/CI_CD` — artifact management, Nexus, Artifactory sections

#### Key Points to Cover in Your Answer:

**Why artifact repositories are needed:**
```
Without artifact repository:
  - CI downloads dependencies from internet on EVERY build (slow, unreliable)
  - Published artifacts scattered across different registries
  - No control over what 3rd party libraries are used
  - Vulnerability in public registry affects all teams immediately

With artifact repository:
  - Proxy: cache public packages internally (fast, offline-capable)
  - Host: publish internal packages (Maven JARs, npm packages, Docker images)
  - Control: approve/block specific packages
  - Audit: know exactly which packages each team uses
  - Security: scan all packages for CVEs before teams use them
```

**3 Repository Types:**
```
Proxy Repository:
  → Caches artifacts from remote registries (Maven Central, npm registry, Docker Hub)
  → First request: downloads from internet, caches locally
  → Subsequent requests: served from cache (fast, no internet needed)
  → Example: mvn.proxy pointing to repo1.maven.org

Hosted Repository:
  → Stores YOUR internally built artifacts
  → Snapshot: development versions (often overwritten)
  → Release: stable versions (immutable)
  → Example: company-releases, company-snapshots

Group Repository:
  → Virtual repository combining multiple proxy + hosted repos
  → Single URL for all artifacts
  → Build tools configured with one URL (the group)
  → Nexus searches group members in order: hosted first, then proxies
```

**Artifact types supported:**
```
Maven/Gradle:  JARs, WARs, POM files (Java ecosystem)
npm:           JavaScript packages
PyPI:          Python packages (pip)
Docker:        Container images (acts like private Docker registry)
Helm:          Kubernetes Helm charts
Apt/Yum:       Linux packages (deb, rpm)
Go:            Go modules
NuGet:         .NET packages
Raw:           Any file (ZIP, tar, binary releases)
```

**CI/CD integration:**
```yaml
# Maven - settings.xml
<settings>
  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>http://nexus:8081/repository/maven-group/</url>
    </mirror>
  </mirrors>
</settings>

# Docker
docker login nexus:8082
docker tag myapp:v1.0 nexus:8082/myapp:v1.0
docker push nexus:8082/myapp:v1.0

# pip
pip install --index-url http://nexus:8081/repository/pypi-proxy/simple/ requests
```

> 💡 **Interview tip:** The key value of artifact repositories in an enterprise context is **supply chain security** — you control EXACTLY which third-party packages your teams can use. By configuring builds to use only the Nexus/Artifactory proxy (and blocking direct internet access from build agents), you can: (1) scan all packages for CVEs before they reach developers, (2) approve/block specific package versions, (3) ensure reproducible builds (cached packages don't change). This is increasingly required for SOC2 and NIST compliance.

---

### Q549 — GitHub Actions | Conceptual | Advanced

> Explain **GitHub Actions Reusable Workflows** and **Composite Actions** — what are they and how do they differ?
>
> When would you use a reusable workflow vs composite action vs a custom Docker action? Write a reusable workflow that other repositories can call to build and push Docker images.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — reusable workflows, composite actions sections

#### Key Points to Cover in Your Answer:

**Reusable Workflow vs Composite Action vs Docker Action:**

| Feature | Reusable Workflow | Composite Action | Docker Action |
|---|---|---|---|
| Scope | Full workflow (jobs + steps) | Multiple steps in one action | Single step, full Docker container |
| Calling syntax | `uses: org/repo/.github/workflows/build.yml` | `uses: org/repo/.github/actions/build` | `uses: org/repo/.github/actions/build` |
| Inputs | `inputs:` + `secrets:` | `inputs:` | `inputs:` |
| Can use `secrets:` | ✅ Yes | ❌ No (secrets passed as inputs) | ❌ No |
| Can define multiple jobs | ✅ Yes | ❌ No (steps only) | ❌ No (single step) |
| Best for | Complex multi-job pipelines | Reusable step sequences | Portable, environment-isolated actions |

**Reusable Workflow — build and push Docker:**
```yaml
# .github/workflows/docker-build.yml (in shared repo)
name: Reusable Docker Build

on:
  workflow_call:                    # this makes it reusable
    inputs:
      image_name:
        required: true
        type: string
      dockerfile_path:
        required: false
        type: string
        default: "Dockerfile"
      push:
        required: false
        type: boolean
        default: true
    secrets:
      aws_role_arn:                 # secrets passed explicitly
        required: true
    outputs:
      image_tag:                    # outputs returned to caller
        description: "Built image tag"
        value: ${{ jobs.build.outputs.image_tag }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.version }}
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.aws_role_arn }}
          aws-region: us-east-1

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ steps.login-ecr.outputs.registry }}/${{ inputs.image_name }}
          tags: |
            type=sha,prefix=sha-
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ${{ inputs.dockerfile_path }}
          push: ${{ inputs.push }}
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Calling the reusable workflow:**
```yaml
# .github/workflows/ci.yml (in calling repo)
name: CI

on:
  push:
    branches: [main]

jobs:
  build-docker:
    uses: my-org/shared-workflows/.github/workflows/docker-build.yml@main
    with:
      image_name: payment-service
      push: true
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}

  deploy:
    needs: build-docker
    runs-on: ubuntu-latest
    steps:
      - name: Deploy with built image
        run: |
          echo "Image tag: ${{ needs.build-docker.outputs.image_tag }}"
```

> 💡 **Interview tip:** Reusable workflows are the **DRY principle applied to CI/CD** — instead of copying the same Docker build + push logic into 50 repositories, you define it once and call it from everywhere. The key advantage over composite actions: reusable workflows can have **multiple jobs with dependencies**, secrets passing, and environment protection rules. When a security fix is needed in the Docker build process, you change it in ONE place and all 50 repos benefit immediately.

---

### Q550 — Terraform | Conceptual | Advanced

> Explain **Terragrunt** — what problem does it solve that Terraform alone cannot?
>
> What is the **DRY (Don't Repeat Yourself)** problem in Terraform at scale? Show how Terragrunt's `terragrunt.hcl` reduces boilerplate across 10+ environments. What is `dependency {}` in Terragrunt and when do you use it?

📁 **Reference:** `nawab312/Terraform` — Terragrunt, DRY Terraform, multi-environment sections

#### Key Points to Cover in Your Answer:

**Problem Terragrunt solves:**
```
Terraform at scale (10+ environments × 10+ modules):

Without Terragrunt:
environments/
  dev/
    vpc/
      main.tf        # same content as staging/vpc/main.tf
      backend.tf     # slightly different bucket/key/region
      provider.tf    # slightly different region
    eks/
      main.tf        # same content
      backend.tf     # different key
  staging/
    vpc/
      main.tf        # COPY of dev/vpc/main.tf
      backend.tf     # different bucket/key
    eks/
      main.tf        # COPY of dev/eks/main.tf

Problem: 100+ directories, 60% is identical boilerplate
         Change one module → update 10 copies → drift → bugs
```

**Terragrunt solution:**
```
With Terragrunt: DRY via inheritance
environments/
  terragrunt.hcl          # ROOT: shared remote_state, provider config
  dev/
    terragrunt.hcl        # ENV: dev-specific vars
    vpc/
      terragrunt.hcl      # MODULE: just the inputs, no boilerplate
    eks/
      terragrunt.hcl
  staging/
    terragrunt.hcl        # ENV: staging-specific vars
    vpc/
      terragrunt.hcl      # ONLY inputs differ from dev!
    eks/
      terragrunt.hcl
```

**Root `terragrunt.hcl`:**
```hcl
# environments/terragrunt.hcl
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"     # auto-generate backend.tf
    if_exists = "overwrite"
  }
  config = {
    bucket         = "my-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

generate "provider" {             # auto-generate provider.tf
  path      = "provider.tf"
  if_exists = "overwrite"
  contents  = <<EOF
provider "aws" {
  region = var.aws_region
}
EOF
}
```

**Module-level `terragrunt.hcl`:**
```hcl
# environments/dev/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()   # inherit root config
}

terraform {
  source = "github.com/my-org/terraform-modules//vpc?ref=v1.2.0"
}

inputs = {
  vpc_cidr    = "10.0.0.0/16"
  environment = "dev"
}

# That's it! Backend and provider auto-generated from root.
# staging/vpc/terragrunt.hcl is identical except vpc_cidr = "10.1.0.0/16"
```

**`dependency {}` block:**
```hcl
# environments/dev/eks/terragrunt.hcl
dependency "vpc" {
  config_path = "../vpc"    # depend on vpc module in same environment
}

inputs = {
  vpc_id          = dependency.vpc.outputs.vpc_id
  private_subnets = dependency.vpc.outputs.private_subnet_ids
  # EKS reads VPC outputs automatically
}

# Terragrunt run-all apply:
# → detects dependency graph
# → applies vpc FIRST, then eks
# terragrunt run-all apply  ← applies all modules in correct order
```

> 💡 **Interview tip:** The **`path_relative_to_include()`** function is the magic that makes Terragrunt work — it dynamically generates the correct S3 key for each module's state file based on the directory path. So `dev/vpc` gets key `dev/vpc/terraform.tfstate` and `staging/eks` gets `staging/eks/terraform.tfstate` automatically, with zero per-module configuration. This is what eliminates the backend.tf boilerplate across all modules.

---

### Q551 — Terraform | Conceptual | Advanced

> Explain **Terraform Cloud** and **Terraform Enterprise** — what features do they add over open-source Terraform?
>
> What is **Terraform Sentinel** and how does it enforce policies before `apply`? Compare Terraform Cloud with running Terraform in GitHub Actions — when would you choose one over the other?

📁 **Reference:** `nawab312/Terraform` — Terraform Cloud, Sentinel, policy enforcement sections

#### Key Points to Cover in Your Answer:

**Features Terraform Cloud adds:**
```
Remote State Management:
  → Free remote state storage (vs managing S3 + DynamoDB yourself)
  → State locking built-in
  → State history and versioning

Remote Execution:
  → Plan + apply runs on Terraform Cloud agents
  → Consistent environment (no "works on my machine")
  → Run history and audit trail

VCS Integration:
  → Connect GitHub/GitLab/Bitbucket
  → Auto-plan on PR (like Atlantis)
  → Auto-apply on merge to main

Private Module Registry:
  → Share Terraform modules across teams
  → Semantic versioning for modules
  → Module documentation

Cost Estimation:
  → Shows estimated monthly cost BEFORE apply
  → Based on resources being created/modified

Sentinel Policy as Code (Enterprise):
  → Enforce organizational policies BEFORE apply
  → Mandatory, Advisory, or Soft-mandatory enforcement
```

**Terraform Sentinel policies:**
```python
# sentinel/restrict-instance-types.sentinel
import "tfplan/v2" as tfplan

# Policy: only allow approved EC2 instance types
allowed_types = ["t3.micro", "t3.small", "t3.medium", "m5.large", "m5.xlarge"]

ec2_violations = filter tfplan.resource_changes as address, changes {
    changes.type is "aws_instance" and
    changes.change.actions contains "create" and
    changes.change.after.instance_type not in allowed_types
}

main = rule {
    length(ec2_violations) is 0
}
```

**Terraform Cloud vs GitHub Actions:**

| Factor | Terraform Cloud | GitHub Actions |
|---|---|---|
| State management | Built-in | Manual (S3 + DynamoDB) |
| Policy enforcement | Sentinel (Enterprise) | OPA/Conftest (DIY) |
| Cost estimation | Built-in | DIY (infracost action) |
| Audit trail | Built-in | GitHub Actions logs |
| VCS integration | Native | Native |
| Cost | Free tier available / $20/user/month | GitHub Actions minutes |
| Learning curve | Low | Medium |

**When to choose:**
```
Terraform Cloud:
  ✅ Team is new to Terraform (easier setup)
  ✅ Need Sentinel for compliance (Enterprise)
  ✅ Want managed state without S3/DynamoDB setup
  ✅ Cost estimation built-in is important

GitHub Actions:
  ✅ Already have complex GitHub Actions pipelines
  ✅ Want full control over execution environment
  ✅ Using Atlantis or Terragrunt (GitHub Actions integrates better)
  ✅ Cost-sensitive (GitHub Actions minutes cheaper)
```

> 💡 **Interview tip:** Terraform Cloud's **cost estimation** feature is underrated — seeing "this plan will increase your monthly bill by $2,400" before applying is extremely valuable for preventing surprise AWS bills. It requires no configuration beyond connecting Terraform Cloud to your AWS credentials. In organizations without FinOps maturity, this single feature often justifies the Terraform Cloud subscription.

---

### Q552 — Ansible | Conceptual | Advanced

> Explain **Ansible Vault** — what does it encrypt and how does it work? What are the different ways to store and provide the vault password?
>
> Show how to encrypt a variable file, use it in a playbook, and integrate Ansible Vault with CI/CD pipelines securely.

📁 **Reference:** `nawab312/AWS` — Ansible Vault, secrets management sections

#### Key Points to Cover in Your Answer:

**What Ansible Vault encrypts:**
```
Ansible Vault uses AES-256 encryption to protect:
  - Variable files (group_vars/, host_vars/)
  - Individual variables within a file
  - Task files
  - Any file in a playbook (templates, handlers)

Most common use: encrypting group_vars/all/vault.yml
containing passwords, API keys, certificates
```

**Working with Vault:**
```bash
# Create encrypted file
ansible-vault create group_vars/production/vault.yml
# → opens editor → type variables → saves encrypted

# Encrypt existing file
ansible-vault encrypt group_vars/production/secrets.yml

# Decrypt to view (temporary)
ansible-vault view group_vars/production/vault.yml

# Edit encrypted file
ansible-vault edit group_vars/production/vault.yml

# Encrypt single variable (inline)
ansible-vault encrypt_string 'MySecretPassword123!' --name 'db_password'
# Output: db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   ...encrypted content...
```

**Variable file structure (best practice):**
```yaml
# group_vars/production/vars.yml (plain text)
db_host: prod-db.company.com
db_port: 5432
db_user: app_user
db_password: "{{ vault_db_password }}"    # reference to vault variable

# group_vars/production/vault.yml (encrypted)
vault_db_password: MySecretPassword123!   # actual secret
vault_api_key: sk-live-abc123...
```

**Running with vault password:**
```bash
# Method 1: interactive prompt
ansible-playbook site.yml --ask-vault-pass

# Method 2: password file
echo "MyVaultPassword" > ~/.vault_pass.txt
chmod 600 ~/.vault_pass.txt
ansible-playbook site.yml --vault-password-file ~/.vault_pass.txt

# Method 3: environment variable (CI/CD)
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass.txt
ansible-playbook site.yml

# Method 4: vault password from AWS Secrets Manager (production best practice)
#!/bin/bash
# vault_pass_script.sh
aws secretsmanager get-secret-value \
  --secret-id ansible-vault-password \
  --query SecretString --output text

ansible-playbook site.yml \
  --vault-password-file vault_pass_script.sh  # Ansible runs this script, uses stdout
```

**CI/CD Integration (GitHub Actions):**
```yaml
- name: Run Ansible Playbook
  env:
    ANSIBLE_VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}
  run: |
    echo "$ANSIBLE_VAULT_PASSWORD" > /tmp/.vault_pass
    chmod 600 /tmp/.vault_pass
    ansible-playbook \
      -i inventory/production \
      site.yml \
      --vault-password-file /tmp/.vault_pass
    rm -f /tmp/.vault_pass  # cleanup
```

> 💡 **Interview tip:** The **vault password script** approach (Method 4) is the production best practice — store the vault password itself in AWS Secrets Manager (or HashiCorp Vault), and have Ansible retrieve it at runtime via a script. This means: no vault password in environment variables, no vault password files on disk, full audit trail of when the vault password was accessed. The vault password is only in one place (Secrets Manager) and rotatable without touching any Ansible files.

---

### Q553 — Ansible | Conceptual | Advanced

> Explain **Ansible Dynamic Inventory** — what problem does it solve and how does it work?
>
> Write a dynamic inventory configuration for **AWS EC2** that groups instances by tag, region, and environment. How do you combine dynamic inventory with static inventory?

📁 **Reference:** `nawab312/AWS` — Ansible Dynamic Inventory, AWS EC2 plugin sections

#### Key Points to Cover in Your Answer:

**Problem static inventory has:**
```
Static inventory (hosts.ini):
  [web_servers]
  10.0.1.10
  10.0.1.11

Problems:
  → New EC2 instance launched → must manually add to hosts file
  → Instance terminated → must remove from hosts file
  → Auto-scaling → hosts file always wrong
  → Multiple regions/environments → huge hosts files to maintain
```

**Dynamic inventory solution:**
```
Dynamic inventory = script or plugin that generates inventory
                    automatically by querying AWS API, GCP, Azure, etc.

Ansible runs inventory script/plugin → gets JSON → uses as inventory
Every ansible run = fresh inventory from AWS (always current)
```

**AWS EC2 dynamic inventory config:**
```yaml
# aws_ec2.yml (file must end in aws_ec2.yml or aws_ec2.yaml)
plugin: amazon.aws.aws_ec2    # requires amazon.aws collection

regions:
  - us-east-1
  - eu-west-1

# Filter: only running instances
filters:
  instance-state-name: running

# Automatic grouping by EC2 attributes
keyed_groups:
  - key: tags.Environment         # group by Environment tag
    prefix: env
  - key: tags.Role                # group by Role tag
    prefix: role
  - key: placement.region         # group by region
    prefix: region
  - key: instance_type            # group by instance type
    prefix: type

# Hostnames: use private IP (for VPC) or public IP
hostnames:
  - private-ip-address
  # or: public-ip-address, private-dns-name, tag:Name

# Add EC2 metadata as hostvars
compose:
  ansible_host: private_ip_address  # connect via private IP
  environment: tags.Environment      # make tag accessible as variable
  server_role: tags.Role

# Cache inventory for 5 minutes (reduces AWS API calls)
cache: true
cache_plugin: jsonfile
cache_connection: /tmp/ansible_inventory_cache
cache_timeout: 300
```

**Running with dynamic inventory:**
```bash
# Test inventory
ansible-inventory -i aws_ec2.yml --list | jq .
ansible-inventory -i aws_ec2.yml --graph

# Shows groups:
# @all:
#   |--@env_production:
#   |  |--10.0.1.10
#   |  |--10.0.1.11
#   |--@env_staging:
#   |  |--10.0.2.10
#   |--@role_web:
#   |  |--10.0.1.10

# Run playbook against specific dynamic group
ansible-playbook -i aws_ec2.yml site.yml \
  --limit env_production   # only production instances
```

**Combining static + dynamic:**
```bash
# Use directory as inventory source
mkdir -p inventory/
cp aws_ec2.yml inventory/
cat > inventory/static_hosts.yml << EOF
all:
  children:
    load_balancers:
      hosts:
        nlb.internal.company.com:
          ansible_user: ec2-user
EOF

# Ansible merges all inventory sources in directory
ansible-playbook -i inventory/ site.yml
# Uses both EC2 dynamic + static load_balancers group
```

> 💡 **Interview tip:** The **cache** setting in dynamic inventory is critical for large AWS environments — without it, every ansible command makes AWS API calls which adds 5-15 seconds of latency. With cache enabled, the inventory is fetched once and reused for the cache_timeout period. For CI/CD pipelines where freshness is critical, set cache_timeout=0. For interactive ad-hoc commands, 5-15 minutes of cache is acceptable.

---

### Q554 — Kubernetes | Conceptual | Advanced

> Explain **PersistentVolume access modes** — what are **ReadWriteOnce (RWO)**, **ReadOnlyMany (ROX)**, **ReadWriteMany (RWX)**, and **ReadWriteOncePod (RWOP)**?
>
> What storage backends support RWX? When does your application NEED RWX vs RWO? What are the common mistakes when choosing access modes?

📁 **Reference:** `nawab312/Kubernetes` → `04_STORAGE.md` — PersistentVolume access modes sections

#### Key Points to Cover in Your Answer:

**4 Access Modes:**
```
ReadWriteOnce (RWO):
  → Mounted as read-write by ONE NODE at a time
  → Multiple pods on the SAME node can use it
  → Most common mode (EBS, regular block storage)

ReadOnlyMany (ROX):
  → Mounted as read-only by MANY nodes simultaneously
  → Use: shared configuration files, reference data
  → Supported by: EFS, NFS, GCS

ReadWriteMany (RWX):
  → Mounted as read-write by MANY nodes simultaneously
  → RARE — only specific backends support it
  → Supported by: EFS, NFS, Azure Files, CephFS, GlusterFS
  → NOT supported by: EBS, most block storage

ReadWriteOncePod (RWOP) — new in K8s 1.22:
  → Mounted as read-write by ONE POD only (stronger than RWO)
  → Prevents multiple pods even on same node from using it
  → Use: single-instance databases, exclusive access needed
```

**When you NEED RWX:**
```
Need RWX when:
  ✅ Multiple pods need to read AND write to shared storage
  ✅ Shared upload directory (multiple web servers writing uploads)
  ✅ Shared log directory (multiple instances writing to same location)
  ✅ CMS media storage (multiple app instances reading/writing assets)
  ✅ Legacy apps requiring shared filesystem semantics

Can use RWO when:
  ✅ Single-instance databases (always RWO)
  ✅ Stateful applications with one replica (RWO)
  ✅ Workloads that don't actually need to share files (most apps!)
```

**Common mistakes:**
```
Mistake 1: Requesting RWX when RWO would work
  → RWX not supported by EBS
  → Team requests RWX for "safety" → gets error (no RWX storage available)
  → Fix: design app to not need shared writable storage (use S3 instead)

Mistake 2: Using EBS (RWO) with Deployment replicas > 1
  → Pod 1 on Node A binds the EBS volume
  → Pod 2 scheduled on Node B → cannot attach (different node!)
  → Pods in Pending state with "Multi-Attach error"
  → Fix: Use StatefulSet (one PVC per pod) or switch to EFS (RWX)

Mistake 3: Assuming EBS supports RWX
  → EBS = block storage = RWO only
  → RWX requires shared filesystem (EFS, NFS)
  → Common error: spec.accessModes: [ReadWriteMany] with EBS storage class
  → Result: PVC stuck in Pending (no volume can satisfy)
```

**EFS StorageClass for RWX:**
```yaml
# AWS EFS StorageClass (RWX supported)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-12345678
  directoryPerms: "700"

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage
spec:
  accessModes:
    - ReadWriteMany     # works with EFS, NOT with EBS
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

> 💡 **Interview tip:** **EBS supports only RWO** — this is one of the most common Kubernetes+AWS storage mistakes. When teams migrate from on-premises NFS shares to AWS, they often try to use EBS for shared storage and hit the "Multi-Attach error" when scaling deployments to multiple replicas. The AWS answer for RWX is **EFS (Elastic File System)** — it is NFS-based and natively supports ReadWriteMany. The application answer is to not need shared writable storage: use S3 for shared files, keep each pod's state local.

---

### Q555 — Kubernetes | Conceptual | Advanced

> Explain **Velero** — what is it, what does it back up, and how does it work?
>
> Walk through: backing up an entire namespace, restoring to a different namespace, and setting up **scheduled backups** to S3. What is the difference between **Velero file system backup** and **volume snapshot backup**?

📁 **Reference:** `nawab312/Kubernetes` — Velero, disaster recovery, cluster backup sections

#### Key Points to Cover in Your Answer:

**What Velero backs up:**
```
Kubernetes resources:
  → All API resources: Deployments, Services, ConfigMaps, Secrets, etc.
  → Custom resources (CRDs and CR instances)
  → RBAC resources (Roles, ClusterRoles, Bindings)
  → Namespace resources

Persistent Volumes (optional):
  → Method 1: File system backup (Restic/Kopia) → backs up actual data
  → Method 2: Volume snapshots (CSI) → creates EBS/EFS snapshots
```

**How Velero works:**
```
Architecture:
  Velero server (Deployment) running in cluster
  BackupStorageLocation: S3 bucket where backups stored
  VolumeSnapshotLocation: where PV snapshots stored

Backup process:
  1. velero backup create my-backup --include-namespaces production
  2. Velero queries Kubernetes API for all resources
  3. Serializes resources to JSON
  4. Uploads to S3: s3://my-bucket/backups/my-backup/
  5. For volumes: creates EBS snapshot or copies data with Restic

Restore process:
  1. velero restore create --from-backup my-backup
  2. Velero reads JSON from S3
  3. Re-creates Kubernetes resources in cluster
  4. For volumes: restores from snapshot or Restic data
```

**File system backup vs Volume snapshot:**
```
Volume Snapshot (CSI snapshots):
  ✅ Fast (instant snapshot)
  ✅ Application-consistent (if hooks configured)
  ✅ Native to storage provider (EBS snapshot)
  ❌ Provider-specific (EBS snapshots don't restore to GCP)
  ❌ Costs money for snapshot storage
  Best for: production databases, large volumes

File system backup (Restic/Kopia):
  ✅ Provider-agnostic (data stored in S3 as files)
  ✅ Portable (restore to any cloud/on-prem)
  ✅ Incremental (only changed files)
  ❌ Slow for large volumes (reads all files)
  ❌ Application may need to be quiesced first
  Best for: smaller volumes, cross-provider migration
```

**Common Velero operations:**
```bash
# Install Velero with AWS plugin
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-velero-backups \
  --secret-file ./credentials-velero \
  --backup-location-config region=us-east-1 \
  --use-node-agent                          # enable Restic/Kopia

# Create backup of specific namespace
velero backup create prod-backup \
  --include-namespaces production \
  --default-volumes-to-fs-backup            # include volumes

# Scheduled backup (daily at 2am)
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces production \
  --ttl 720h                               # keep for 30 days

# Restore to different namespace (migration)
velero restore create --from-backup prod-backup \
  --namespace-mappings production:production-restored

# Check backup status
velero backup describe prod-backup
velero backup logs prod-backup
```

> 💡 **Interview tip:** Velero is essential for **Kubernetes DR** but it only backs up resources and PV data — it does NOT back up the etcd database (cluster state). A complete Kubernetes DR strategy needs both: Velero for namespaced application data AND etcd snapshots for cluster configuration (control plane nodes, CRDs, RBAC). Also mention that Velero backups are tested by actually doing a restore in a test namespace — "untested backups are not backups."

---

### Q556 — Kubernetes | Conceptual | Advanced

> Explain **Helmfile** — what problem does it solve and how does it differ from using Helm directly?
>
> Show a Helmfile configuration that manages 5 Helm releases across 3 environments with different values per environment. What is `helmfile sync` vs `helmfile diff` vs `helmfile apply`?

📁 **Reference:** `nawab312/Kubernetes` — Helmfile, multi-release Helm management sections

#### Key Points to Cover in Your Answer:

**Problem Helmfile solves:**
```
Helm alone:
  helm upgrade --install prometheus prometheus-community/prometheus \
    -f values/base.yaml -f values/production.yaml --version 25.0.0

  helm upgrade --install grafana grafana/grafana \
    -f values/grafana.yaml --version 7.0.0

  helm upgrade --install cert-manager cert-manager/cert-manager \
    --set installCRDs=true --version 1.13.0

  # Problems:
  # - Must run multiple commands (one per release)
  # - No declarative definition of ALL releases
  # - Easy to forget which release needs which flags
  # - Different commands per environment (different values files)
  # - No easy "diff all changes before applying"

Helmfile:
  helmfile diff    # show ALL changes across ALL releases
  helmfile apply   # apply ONLY changed releases (idempotent)
  helmfile sync    # force sync ALL releases
```

**Complete Helmfile configuration:**
```yaml
# helmfile.yaml
environments:
  dev:
    values:
    - environments/dev.yaml
  staging:
    values:
    - environments/staging.yaml
  production:
    values:
    - environments/production.yaml
    kubeContext: production-eks

repositories:
- name: prometheus-community
  url: https://prometheus-community.github.io/helm-charts
- name: grafana
  url: https://grafana.github.io/helm-charts
- name: cert-manager
  url: https://charts.jetstack.io

releases:
- name: prometheus
  namespace: monitoring
  createNamespace: true
  chart: prometheus-community/prometheus
  version: "25.0.0"
  values:
  - values/prometheus/base.yaml
  - values/prometheus/{{ .Environment.Name }}.yaml  # env-specific values
  set:
  - name: server.retention
    value: {{ .Values.prometheus.retention }}        # from environment values

- name: grafana
  namespace: monitoring
  chart: grafana/grafana
  version: "7.0.0"
  values:
  - values/grafana/base.yaml
  needs:
  - monitoring/prometheus                            # deploy after prometheus

- name: cert-manager
  namespace: cert-manager
  createNamespace: true
  chart: cert-manager/cert-manager
  version: "1.13.0"
  set:
  - name: installCRDs
    value: "true"

- name: payment-service
  namespace: production
  chart: ./charts/payment-service
  version: "{{ .Values.payment_service.version }}"  # version from env values
  values:
  - values/payment-service/base.yaml
  - values/payment-service/{{ .Environment.Name }}.yaml
```

**Key commands:**
```bash
# Show all changes before applying (like terraform plan)
helmfile diff --environment production

# Apply only changed releases (fast, idempotent)
helmfile apply --environment production

# Force sync all releases (like terraform apply -refresh)
helmfile sync --environment production

# Only deploy specific releases
helmfile -l name=payment-service apply --environment production

# Destroy specific release
helmfile -l name=payment-service destroy --environment production
```

> 💡 **Interview tip:** `helmfile apply` vs `helmfile sync`: **apply** is smarter — it runs `helm diff` first and only upgrades releases that have actual changes. **sync** forces upgrade of ALL releases regardless. In production, always use `helmfile apply` — it is faster and safer (less chance of unintended changes to stable releases). Think of `apply` as idempotent and `sync` as forceful. For CI/CD pipelines, `helmfile apply` is the correct command.

---

### Q557 — Kubernetes | Conceptual | Advanced

> Explain **Kustomize** in depth — what is it and how does it differ from Helm? What are **bases**, **overlays**, **patches**, and **transformers**?
>
> Show a Kustomize structure that deploys the same application to dev, staging, and production with different replicas, resource limits, and environment-specific config.

📁 **Reference:** `nawab312/Kubernetes` — Kustomize, overlay management sections

#### Key Points to Cover in Your Answer:

**Kustomize vs Helm:**
```
Helm:
  → Template engine (Go templates with {{ }})
  → Chart = templates + values
  → Packaged, versioned, distributed via registries
  → Full-featured: hooks, lifecycle, rollback tracking
  → Learning curve: Helm template syntax

Kustomize:
  → Pure YAML patching (no templating)
  → Overlay pattern (base + patches)
  → Built into kubectl (kubectl apply -k)
  → No package distribution model (just Git)
  → Simpler: just YAML

When to use Kustomize:
  ✅ Your own application manifests (not third-party charts)
  ✅ Teams that want "just YAML" without template syntax
  ✅ When already in ArgoCD (native support)
  ✅ Simple environment differences

When to use Helm:
  ✅ Third-party software (Prometheus, Grafana, cert-manager)
  ✅ Complex parameterization needed
  ✅ Want versioned packages and rollback history
```

**Kustomize directory structure:**
```
k8s/
├── base/                          # shared resources
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── replicas.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── replicas.yaml
    └── production/
        ├── kustomization.yaml
        └── patches/
            ├── replicas.yaml
            └── resources.yaml
```

**base/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:
  app: payment-service
  managed-by: kustomize
```

**overlays/production/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base                       # inherit base resources

namespace: production

# Image tag override (common for CI/CD)
images:
- name: payment-service
  newTag: v1.2.3                   # override image tag

# ConfigMap override
configMapGenerator:
- name: payment-config
  behavior: merge
  literals:
  - LOG_LEVEL=info                 # override dev's LOG_LEVEL=debug

# Strategic merge patches (merge with base)
patches:
- path: patches/replicas.yaml      # patch: set replicas to 5
- path: patches/resources.yaml     # patch: increase resource limits
```

**patches/replicas.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 5                      # production: 5 replicas (dev: 1)
```

**patches/resources.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      containers:
      - name: payment-service
        resources:
          requests:
            cpu: "500m"            # production: more resources
            memory: "512Mi"
          limits:
            cpu: "2"
            memory: "2Gi"
```

**Deploy:**
```bash
# Preview production manifests
kubectl kustomize overlays/production

# Apply production
kubectl apply -k overlays/production

# In ArgoCD:
# source:
#   path: overlays/production
#   repoURL: https://github.com/company/k8s-manifests
#   targetRevision: main
```

> 💡 **Interview tip:** Kustomize's **strategic merge patch** is smarter than a simple JSON merge — it understands Kubernetes resource structures. For example, when patching a container in a Deployment, Kustomize finds the right container by name and merges only the specified fields, leaving other container settings unchanged. A simple JSON merge would replace the entire containers array. This is why `patches` in Kustomize work intuitively — you only specify what changes.

---

### Q558 — SRE Culture | Conceptual | Advanced

> Explain the **12-Factor App** methodology — what are the 12 factors and why do they matter for DevOps and cloud-native applications?
>
> For a banking microservice you are deploying, which factors are most critical and which are most commonly violated?

📁 **Reference:** `nawab312/Kubernetes` and `nawab312/CI_CD` — 12-factor app, cloud-native principles sections

#### Key Points to Cover in Your Answer:

**The 12 Factors:**
```
I.   Codebase:        One codebase, tracked in VCS, many deploys
                      (not multiple repos for same app, not no VCS)

II.  Dependencies:    Explicitly declare and isolate dependencies
                      (requirements.txt, pom.xml, package.json — no system packages)

III. Config:          Store config in environment (not code)
                      (no hardcoded URLs, ports, credentials in source code)

IV.  Backing services: Treat backing services as attached resources
                      (DB, cache, queue = interchangeable, configured via env var)

V.   Build/Release/Run: Strictly separate build, release, and run stages
                      (immutable builds, release = build + config, run = execute)

VI.  Processes:       Execute app as stateless processes
                      (no sticky sessions, no local file storage for state)

VII. Port binding:    Export services via port binding
                      (app is self-contained, binds its own port)

VIII. Concurrency:    Scale out via the process model
                      (horizontal scaling, not vertical)

IX.  Disposability:   Maximize robustness with fast startup and graceful shutdown
                      (preStop hooks, SIGTERM handling, short startup time)

X.   Dev/prod parity: Keep dev, staging, production as similar as possible
                      (same services, same versions, Docker Compose for local)

XI.  Logs:            Treat logs as event streams
                      (write to stdout, never files, let infra aggregate)

XII. Admin processes: Run admin tasks as one-off processes
                      (database migrations = one-off Job, not app startup)
```

**Most critical for banking microservice:**
```
Factor III (Config): CRITICAL
  → Never hardcode DB credentials, API keys
  → Use env vars from Kubernetes Secrets/ConfigMaps
  → Violation = credentials in Git = security incident

Factor VI (Processes): CRITICAL
  → Banking app must be stateless
  → No in-memory session state (lost on crash)
  → Use Redis/DB for session storage
  → Enables zero-downtime rolling deployments

Factor IX (Disposability): CRITICAL
  → Must handle SIGTERM gracefully (complete in-flight transactions)
  → Fast startup (< 30s) for auto-scaling to be effective
  → Violation = dropped transactions during deployment

Factor XI (Logs): CRITICAL
  → Write to stdout only (never files)
  → Kubernetes/Fluentbit picks up stdout automatically
  → Violation = logs lost when pod restarts
```

**Most commonly violated in practice:**
```
Factor III (Config): most violated
  → Teams hardcode "localhost" URLs, commit env files
  → Fix: code review, pre-commit hooks for secret detection

Factor X (Dev/prod parity): commonly violated
  → "Works on my laptop" with SQLite but prod uses PostgreSQL
  → Different versions locally vs production
  → Fix: Docker Compose with same images as production

Factor IX (Disposability): often missed
  → App ignores SIGTERM, gets SIGKILL after 30s
  → Fix: implement signal handler, use preStop hook
```

> 💡 **Interview tip:** Factor XI (logs as event streams) is the factor most directly impacted by containerization — containers that write to files need volume mounts and log rotation, while containers that write to stdout get automatic log collection by the container runtime. This is why `docker logs` works: it captures stdout/stderr. On Kubernetes, Fluentbit/Fluentd collects `/var/log/containers/*.log` which is the container runtime writing stdout to disk — the application never writes to disk.

---

### Q559 — SRE Culture | Conceptual | Advanced

> Explain the **blameless postmortem** process — what makes a postmortem blameless and what makes it effective?
>
> Walk through writing a complete postmortem for a real incident: a database connection pool exhaustion that caused 15 minutes of downtime in a banking application. Include: timeline, contributing factors, root causes, action items.

📁 **Reference:** SRE practices, incident management, postmortem culture sections

#### Key Points to Cover in Your Answer:

**Why blameless:**
```
Blame-ful culture:
  → Engineers hide mistakes
  → Root cause analysis stops at "human error"
  → Same incidents repeat
  → Engineers leave (psychological safety gone)

Blameless culture:
  → People operated with best information available at the time
  → Systems/processes failed, not individuals
  → Rich root cause analysis (why was it possible to make this mistake?)
  → Action items prevent recurrence
  → Engineers feel safe reporting issues
```

**Complete postmortem structure:**
```
Incident: Database Connection Pool Exhaustion
Severity: P1
Duration: 14:05 - 14:20 UTC (15 minutes)
Impact: 100% of payment API requests failed with "FATAL: connection pool exhausted"
Written by: On-call SRE
Date: 2024-01-15

SUMMARY
At 14:05 UTC, payment service became completely unavailable due to RDS connection
pool exhaustion. Root cause: a slow database query caused connection pool threads
to block, accumulating until the pool was exhausted. Service restored at 14:20
after increasing connection pool size and killing the blocking query.

TIMELINE
13:45 UTC - Batch analytics job started (scheduled, unmonitored)
14:02 UTC - Analytics job runs expensive full-table scan on payments table
14:03 UTC - RDS CPU increases to 95% (no alert configured for analytics queries)
14:04 UTC - API service connections start queueing (pool threads waiting for query)
14:05 UTC - Connection pool exhausted, new API requests fail immediately
14:05 UTC - PagerDuty fires: "payment-service error rate 100%"
14:06 UTC - On-call engineer paged
14:09 UTC - On-call identifies connection pool exhaustion in logs
14:11 UTC - Identifies blocking query from RDS Performance Insights
14:14 UTC - Kills blocking analytics query
14:17 UTC - Connection pool begins recovering
14:20 UTC - Service fully restored

CONTRIBUTING FACTORS
1. Analytics job had no connection limit (used same pool as API)
2. No monitoring on analytics query duration
3. No RDS connection pool utilization alert
4. Connection pool size (10) too small for sustained load
5. Analytics job had no query timeout configured

ROOT CAUSES (5 Whys)
Why did the service fail? → Connection pool exhausted
Why was pool exhausted? → Slow query holding connections
Why was the query slow? → Full table scan, no query timeout
Why no timeout? → Analytics job config never reviewed for production impact
Why never reviewed? → Analytics and API teams don't have joint review process

ACTION ITEMS
1. [CRITICAL - 3 days] Add 1-second timeout to all analytics queries
2. [HIGH - 1 week] Alert on RDS connection pool utilization > 70%
3. [HIGH - 1 week] Separate analytics connection pool from API pool
4. [MEDIUM - 2 weeks] Add query timeout at RDS parameter group level (max 30s)
5. [MEDIUM - 2 weeks] Require analytics team review for queries touching large tables
6. [LOW - 1 month] Implement circuit breaker in API to degrade gracefully during DB pressure
```

> 💡 **Interview tip:** The **5 Whys technique** is what separates a good postmortem from a bad one. "Root cause: database was slow" is bad (descriptive, not actionable). "Root cause: no process for analytics teams to get approval before running queries that touch production tables" is good (systemic, actionable). The test of a good root cause: could the same incident happen again if we only fixed the immediate technical issue? If yes, keep asking why.

---

### Q560 — SRE Culture | Conceptual | Advanced

> Explain **SRE Toil** — what is it, how do you measure it, and why does Google's SRE model suggest keeping toil below 50%?
>
> Give 5 real examples of toil in a DevOps/SRE role and for each one, describe how you would eliminate it through automation.

📁 **Reference:** SRE principles, toil elimination, automation sections

#### Key Points to Cover in Your Answer:

**Definition of Toil:**
```
Toil = manual, repetitive, automatable work that scales with service growth

Properties of Toil (must have most of these):
  ✅ Manual: requires human action
  ✅ Repetitive: done again and again
  ✅ Automatable: a machine could do it
  ✅ Reactive: triggered by interruption, not planned
  ✅ No lasting value: doing it doesn't improve the service
  ✅ O(n) with service growth: more services = more toil

NOT toil:
  - On-call for novel incidents (requires engineering judgment)
  - Code reviews (judgment required)
  - Project planning
  - Capacity planning (intellectual work, not mechanical)
```

**Why < 50% toil matters:**
```
50% rule:
  → SREs must spend < 50% time on toil
  → > 50% = "ops team with fancy title" (no engineering done)
  → Remainder: engineering work that permanently reduces future toil

If toil is 80%:
  → No time to automate toil → toil grows with scale
  → Engineers burn out (repetitive work)
  → Service reliability doesn't improve
  → Talent retention problem (engineers want to build, not click)
```

**5 Toil Examples + Elimination:**

```
Toil 1: Manual EC2 patching
  Current: SSH into 200 servers monthly, run yum update, reboot
  Elimination: AWS Systems Manager Patch Manager
    → automated patching with maintenance windows
    → zero manual SSH required
    → patch compliance report generated automatically

Toil 2: Manually scaling services for traffic events
  Current: Before each marketing campaign, manually scale ECS tasks
  Elimination: Scheduled scaling in ECS / Application Auto Scaling
    → Define: Fri 6pm UTC = scale to 20 tasks
    → Scale back: Mon 6am UTC = scale to 5 tasks
    → Zero manual intervention

Toil 3: Responding to CloudWatch "low disk" alerts
  Current: Alert fires → SSH → find large files → delete → close alert
  Elimination:
    → CloudWatch alarm → Lambda → auto-cleanup of known safe directories
    → EBS storage autoscaling for non-root volumes
    → Logrotate properly configured (most root cause)

Toil 4: Creating DNS records for new services
  Current: Developer deploys service → tickets Ops → Ops creates DNS
  Elimination: External DNS controller in Kubernetes
    → Developer adds annotation to Ingress
    → ExternalDNS automatically creates Route53 record
    → Zero Ops involvement

Toil 5: Renewing SSL certificates
  Current: Calendar reminder → generate CSR → email CA → install cert → restart
  Elimination: cert-manager + Let's Encrypt
    → Certificates auto-renewed 30 days before expiry
    → Zero manual intervention
    → Never expires silently
```

> 💡 **Interview tip:** When asked about toil in interviews, always quantify it: "This took 2 hours per week × 52 weeks = 104 engineer-hours per year. I automated it, saving 100 hours/year which I redirected to reliability engineering." This shows you think like an SRE (measuring, optimizing) rather than just "I automated stuff." The business case for toil elimination is always: engineering time saved = engineering time for reliability improvement.

---

### Q561 — SRE Culture | Conceptual | Advanced

> Explain **on-call best practices** — what makes an on-call rotation healthy vs toxic?
>
> What is **alert fatigue** and how do you fix it? What should an **on-call runbook** contain? What is the difference between **pages** (P1), **tickets** (P2), and **FYI (P3)** in an alerting strategy?

📁 **Reference:** SRE on-call practices, alerting strategy, runbook design sections

#### Key Points to Cover in Your Answer:

**Healthy vs Toxic On-Call:**
```
Healthy on-call:
  ✅ < 2 pages per shift (on average)
  ✅ Alerts are actionable (on-call can DO something)
  ✅ Runbooks exist for all pages
  ✅ < 30% of pages at night (not chronically disrupting sleep)
  ✅ On-call rotation: 1 week on, N weeks off (N >= 4)
  ✅ Escalation path exists (on-call can call for backup)
  ✅ Time in lieu after rough on-call

Toxic on-call:
  ❌ 20+ pages per shift
  ❌ "Low disk space" pages at 3am (not urgent, just noise)
  ❌ No runbooks (on-call improvises every time)
  ❌ Same engineer on-call every week
  ❌ Pages that require escalating immediately (why page the SRE?)
  ❌ Alert flapping (same alert fires/resolves repeatedly)
```

**Alert fatigue and fixes:**
```
Alert fatigue: on-call receives so many alerts they start ignoring them
               → a real P1 alert gets ignored → user-impacting outage

Signs of alert fatigue:
  - On-call acknowledges alerts without investigating
  - Alerts frequently "accidentally" resolved without fixing root cause
  - On-call complains every shift
  - p50 alert volume: 20+ per shift

Fixes:
1. Delete alerts nobody acts on (if ignored 5 shifts in a row → delete or demote)
2. Raise thresholds (CPU 80% → alert, not CPU 60%)
3. Increase evaluation windows (5-minute alert, not 1-minute spike)
4. Demote noisy alerts to Slack (ticket) instead of page
5. Fix root causes (why does low disk keep happening? → fix, not alert)
6. Monthly alert review: every alert must have an owner and a runbook
```

**Alerting severity strategy:**
```
P1 (Page — wake you up):
  → Service completely down or severely degraded
  → Revenue/data loss occurring
  → SLA breach occurring
  → Requires immediate action
  → Channel: PagerDuty → SMS + phone call
  Examples: payment API error rate > 10%, cluster unreachable

P2 (Ticket — next business day):
  → Warning signs that may become P1
  → Needs attention but not immediately
  → Channel: Jira ticket auto-created + Slack notification
  Examples: disk usage > 70%, replica lag > 30s

P3 (FYI — awareness only):
  → Informational
  → No action required right now
  → Channel: Slack #monitoring channel only
  Examples: deployment completed, weekly cost report, certificate expiry in 60 days
```

**What a good runbook contains:**
```
Title: High Error Rate on Payment Service
Severity: P1
Owner: Payments Team SRE
Last updated: 2024-01-15

WHAT DOES THIS ALERT MEAN?
The payment service error rate exceeds 5% over 5 minutes.
This means customers cannot complete payments.

IMMEDIATE ACTIONS (first 5 minutes):
1. Check pods: kubectl get pods -n production -l app=payment-service
2. Check recent deployments: argocd app history payment-service
3. Check DB connections: [link to CloudWatch dashboard]

COMMON CAUSES AND FIXES:
1. Recent bad deployment → rollback: argocd app rollback payment-service
2. DB connection exhaustion → [runbook link for DB issues]
3. Downstream service failure → check fraud-service status

ESCALATION:
If not resolved in 15 minutes → page Tech Lead at +1-555-xxx
If DB issue → page DBA on-call at +1-555-yyy

DASHBOARDS:
- Payment service: [Grafana link]
- DB performance: [link]
```

> 💡 **Interview tip:** The best measure of on-call health is **mean alerts per on-call shift**. Google's SRE book suggests fewer than 2 actionable alerts per 12-hour shift is the target. If you are getting 20 alerts per shift, 18 of them are noise and are burning out your engineers. The fix is NOT to hire more people — it is to fix the root causes driving the alerts and delete/demote the rest. Always cite this metric when discussing on-call improvement initiatives.

---

### Q562 — Observability | Conceptual | Advanced

> Explain **Prometheus PushGateway** in depth — not just what it is, but when you should use it vs NOT use it.
>
> What are the **known anti-patterns** with PushGateway? What happens to metrics when the job that pushed them crashes? How do you implement **metric staleness** handling for batch jobs?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — PushGateway, batch monitoring sections

#### Key Points to Cover in Your Answer:

**When PushGateway IS the right tool:**
```
✅ Batch jobs: complete in < scrape_interval (60s)
   → Job finishes before Prometheus scrapes it
   → Must push metrics before exiting

✅ Jobs behind firewall: Prometheus cannot reach them
   → Push to PushGateway (in accessible network)
   → Prometheus scrapes PushGateway instead

✅ Cron jobs needing last-success tracking
   → "When did this job last succeed?"
   → Push: job_last_success_timestamp{job="nightly-backup"} = $(date +%s)
```

**Known anti-patterns (when NOT to use):**
```
❌ Long-running services:
   → Use pull model instead
   → PushGateway cannot detect if service is down
   → Metrics persist even after service crashes

❌ Multiple instances of same service:
   → PushGateway stores ONE value per metric+label combo
   → Instance 1 pushes: http_requests_total = 100
   → Instance 2 pushes: http_requests_total = 200
   → PushGateway stores LAST PUSH = 200 (instance 1 silently lost)
   → Use: distinct job label per instance (not ideal)
   → Better: pull model with separate targets per instance

❌ Replacing the pull model for convenience:
   → "It's easier to push than expose /metrics"
   → You lose: target health detection, scrape failure detection
```

**Metric staleness — critical problem:**
```
Problem: batch job completes (success) → pushes metrics
         Next run fails → pushes nothing (or pushes failure status)
         Prometheus still reads OLD success metrics from PushGateway!
         → Looks like job is still succeeding

Solution 1: Always push final status metric (success/failure)
  #!/bin/bash
  START_TIME=$(date +%s)

  run_backup() { ... }  # your job logic

  if run_backup; then
    STATUS=1  # success
  else
    STATUS=0  # failure
  fi

  END_TIME=$(date +%s)
  DURATION=$((END_TIME - START_TIME))

  # Push BOTH success status AND completion time
  cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/backup
  job_success{job="nightly-backup"} $STATUS
  job_duration_seconds{job="nightly-backup"} $DURATION
  job_last_run_timestamp{job="nightly-backup"} $END_TIME
  EOF

Solution 2: Delete metrics after job completes (prevents stale reads)
  curl -X DELETE http://pushgateway:9091/metrics/job/nightly-backup

Solution 3: Alert on last success timestamp being too old
  - alert: BackupJobNotRun
    expr: time() - job_last_run_timestamp{job="nightly-backup"} > 86400
    for: 0m
    annotations:
      summary: "Backup job not run in 24 hours"
```

> 💡 **Interview tip:** The **DELETE after push** pattern is the most production-safe PushGateway usage — push metrics at job start and end, then DELETE the entire metric group after completion. If the job crashes, Prometheus will still see the "started but not finished" state. If the job completes, the metrics are cleaned up. The alternative (not deleting) leaves metrics lingering forever, giving false confidence that jobs are running even when they are not.

---

### Q563 — Observability | Conceptual | Advanced

> Explain **PromQL `topk()`, `bottomk()`, `sort()`, `sort_desc()`** functions. When would you use each?
>
> Also explain **`changes()`**, **`resets()`**, **`deriv()`** functions and give real-world use cases for each.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — PromQL functions, ranking sections

#### Key Points to Cover in Your Answer:

**Ranking functions:**
```promql
# topk(): return k elements with HIGHEST values
# Use: find top N worst-performing services/pods/nodes

# Top 5 pods by memory usage
topk(5, container_memory_usage_bytes{container!=""})

# Top 3 endpoints by error rate
topk(3,
  sum by(endpoint) (rate(http_requests_total{status=~"5.."}[5m]))
)

# bottomk(): return k elements with LOWEST values
# Use: find underutilized resources, lowest-performing

# Bottom 3 nodes by CPU utilization (least busy)
bottomk(3, avg by(node) (rate(node_cpu_seconds_total{mode="idle"}[5m])))

# sort() / sort_desc(): order time series by value
# Note: returns instant vector, used in Grafana table panels

# Sort services by request rate (ascending)
sort(sum by(service) (rate(http_requests_total[5m])))

# Sort by error rate (descending) - highest error rate first
sort_desc(
  sum by(service) (rate(http_requests_total{status=~"5.."}[5m]))
)
```

**`changes()` — count value changes:**
```promql
# changes(v range-vector): count how many times value changed in window
# Use: detect flapping, frequent state changes

# Alert if pod restarted more than 3 times in 1 hour
kube_pod_container_status_restarts_total
and
changes(kube_pod_container_status_restarts_total[1h]) > 3

# Count how many times an alert went from 0 to 1 (alert flapping)
changes(ALERTS{alertname="HighCPU"}[1h]) > 5
```

**`resets()` — count counter resets:**
```promql
# resets(v range-vector): count how many times a counter reset to zero
# Use: detect process restarts, crashes

# Count how many times nginx restarted (counter reset = restart)
resets(nginx_http_requests_total[1h]) > 0

# Alert on any process that restarted in last 15 minutes
resets(process_cpu_seconds_total[15m]) > 0
```

**`deriv()` — derivative (rate of change):**
```promql
# deriv(v range-vector): per-second derivative using linear regression
# Use: for GAUGES (not counters) - how fast is a gauge changing?
# Note: rate() is for counters, deriv() is for gauges

# How fast is memory growing (bytes per second)?
deriv(process_resident_memory_bytes[1h])
# Positive value = memory growing (possible leak)
# Negative value = memory shrinking

# Alert: memory leak (growing > 1MB/second for 30 minutes)
- alert: MemoryLeak
  expr: deriv(container_memory_usage_bytes[30m]) > 1048576
  for: 30m
  annotations:
    summary: "Container {{ $labels.container }} memory growing at {{ $value | humanize }}B/s"
```

> 💡 **Interview tip:** `topk()` and `sort_desc()` look similar but behave differently: **`topk(5, expr)`** returns exactly 5 time series (the top 5), while **`sort_desc(expr)`** returns ALL time series sorted by value. In Grafana tables, use `sort_desc()` to show all services sorted by error rate. Use `topk()` when you specifically want "just the top 5 worst" without seeing everything else.

---

### Q564 — Observability | Conceptual | Advanced

> Explain how **Logstash Grok patterns** work. What is a Grok pattern, what is the Grok syntax, and how do you parse a custom application log format?
>
> Write Grok patterns to parse: an Nginx access log, a Java stack trace (multiline), and a custom banking transaction log. What do you do when a Grok pattern fails to match?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Logstash Grok, log parsing sections

#### Key Points to Cover in Your Answer:

**What Grok is:**
```
Grok = named regex patterns for parsing unstructured log text into fields
Pre-built patterns for common formats (IP, hostname, date, HTTP method, etc.)
Logstash ships with 120+ built-in Grok patterns

Syntax: %{PATTERN_NAME:field_name}
  %{IP:client_ip}          → matches IP address, stores in field "client_ip"
  %{NUMBER:response_time}  → matches number, stores in "response_time"
  %{WORD:method}           → matches word, stores in "method"
  %{GREEDYDATA:message}    → matches everything remaining
```

**Nginx access log parsing:**
```
Log format:
192.168.1.100 - user1 [15/Jan/2024:14:05:23 +0000] "GET /api/payments HTTP/1.1" 200 1247

Grok pattern:
%{IPORHOST:client_ip} - %{USER:username} \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{URIPATH:request_path} HTTP/%{NUMBER:http_version}" %{NUMBER:response_code:int} %{NUMBER:bytes:int}

Result fields:
  client_ip:     "192.168.1.100"
  username:      "user1"
  timestamp:     "15/Jan/2024:14:05:23 +0000"
  method:        "GET"
  request_path:  "/api/payments"
  response_code: 200
  bytes:         1247
```

**Java stack trace (multiline):**
```ruby
# Logstash config for multiline Java stack traces
input {
  file {
    path => "/var/log/payment-service/*.log"
    codec => multiline {
      pattern => "^%{TIMESTAMP_ISO8601}"   # new log starts with timestamp
      negate => true                        # if NOT timestamp → continuation
      what => "previous"                    # append to previous event
    }
  }
}

filter {
  grok {
    match => {
      "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} \[%{DATA:thread}\] %{JAVACLASS:class} - %{GREEDYDATA:log_message}"
    }
  }
  if [level] == "ERROR" {
    grok {
      match => {
        "log_message" => "%{JAVACLASS:exception_class}: %{GREEDYDATA:exception_message}"
      }
    }
  }
}
```

**Custom banking transaction log:**
```
Log format:
TXN|2024-01-15T14:05:23Z|DEBIT|ACC001234|USD|500.00|SUCCESS|REF-ABC123

Grok pattern:
TXN\|%{TIMESTAMP_ISO8601:timestamp}\|%{WORD:transaction_type}\|%{WORD:account_id}\|%{WORD:currency}\|%{NUMBER:amount:float}\|%{WORD:status}\|%{WORD:reference}
```

**Handling Grok failures:**
```ruby
filter {
  grok {
    match => { "message" => "%{NGINX_ACCESS_LOG}" }
    tag_on_failure => ["_grokparsefailure"]   # tag failed parses (default)
    break_on_match => false                    # try all patterns, not just first
  }

  # Route failed parses to separate index for investigation
  if "_grokparsefailure" in [tags] {
    mutate {
      add_field => { "[@metadata][index]" => "parse-failures" }
    }
  }
}

# Test patterns without running Logstash:
# Kibana Dev Tools → Grok Debugger
# OR: online tool: grokconstructor.appspot.com
```

> 💡 **Interview tip:** **Always test Grok patterns in Kibana's Grok Debugger** before deploying to production — a wrong Grok pattern will silently fail and tag all events as `_grokparsefailure`, sending everything to a failure index while production logs are unparsed. The `_grokparsefailure` tag is your early warning system — alert on it: if `_grokparsefailure` count > 0, investigate immediately. Also mention that for high-throughput pipelines, simpler is better: plain JSON logs (`json {}` filter) are 10x faster than Grok parsing.

---

### Q565 — Observability | Conceptual | Advanced

> Explain **Grafana Mimir's ruler component** and how it handles **Prometheus rules at scale** across multiple tenants.
>
> Also explain **Grafana Tempo** — what is it, how does it differ from Jaeger/Zipkin, and how does the **TraceQL** query language work?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — Tempo, TraceQL, Mimir ruler sections

#### Key Points to Cover in Your Answer:

**Mimir Ruler:**
```
Purpose: evaluates Prometheus alerting and recording rules
         at scale across multiple tenants

How it works:
  - Rules stored per-tenant in object storage (S3)
  - Ruler instances shard rule groups across themselves
  - Each ruler evaluates its shard of rules
  - Recording rule results → pushed back to Ingesters → stored
  - Alerting rule results → sent to Alertmanager

Multi-tenant rule management:
  # Each team manages their own rules
  # Upload via Mimirtool:
  mimirtool rules load rules.yaml \
    --address http://mimir:8080 \
    --id payment-team    # tenant ID

  # Rules only affect that tenant's metrics
  # No cross-tenant interference
```

**Grafana Tempo:**
```
What: distributed tracing backend (stores and queries traces)

How it differs from Jaeger/Zipkin:
  Jaeger/Zipkin:
    → Own storage format
    → UI tightly coupled to backend
    → Separate deployment

  Grafana Tempo:
    → Stores traces in object storage (S3/GCS) — cheap at massive scale
    → No indexing by default (full scan for queries) → use exemplars for access
    → Integrates with Grafana UI natively
    → Receives traces from: Jaeger, Zipkin, OpenTelemetry, Zipkin
    → "Just works" with Grafana stack (Prometheus + Loki + Tempo = 3 pillars)

Architecture:
  Application (OTEL SDK) → Tempo Distributor → Ingester → S3 storage
  Query: Tempo Query Frontend → Querier → S3
```

**TraceQL — Tempo's query language:**
```
Like PromQL for traces:

# Find all traces with a specific span attribute
{ span.http.status_code = 500 }

# Find slow traces (> 1 second)
{ duration > 1s }

# Find traces where payment service had errors
{ resource.service.name = "payment-service" && status = error }

# Find traces spanning multiple services
{ span.db.system = "postgresql" } | count() > 5

# Pipeline: filter then aggregate
{ resource.service.name = "payment-service" }
| avg(duration) by(span.http.route)

# Compare with specific trace ID (for drilling into a specific trace)
{ traceID = "abc123def456" }
```

**Connecting Prometheus metrics to Tempo traces (exemplars):**
```promql
# In Grafana: show metrics + traces together
# Click a spike in error rate → jump to a trace from that exact moment

# Enable exemplars in prometheus.yml:
scrape_configs:
  - job_name: payment-service
    exemplars:
      enabled: true

# Application emits exemplars with trace ID:
# http_request_duration_seconds{method="POST"} 0.45 # {traceID="abc123"} 0.45
```

> 💡 **Interview tip:** The key Tempo advantage that impresses interviewers: **Tempo stores traces in object storage at minimal cost** — unlike Jaeger which requires Elasticsearch (expensive at scale), Tempo stores raw traces in S3 for pennies. The trade-off is that without indexes, searching traces is slow (full scan). The solution is **exemplars** — Prometheus metrics link to specific trace IDs, so you navigate to traces from the metric spike rather than searching. This is the "metrics → traces" correlation that makes the Grafana stack powerful.

---

### Q566 — Networking | Conceptual | Advanced

> Explain **gRPC** in depth — how does it work, what are the 4 communication patterns, and why is it better than REST for microservice communication?
>
> What is **Protocol Buffers (protobuf)** and why is it more efficient than JSON? How do you expose gRPC services on Kubernetes with a gRPC-aware load balancer?

📁 **Reference:** `nawab312/Kubernetes` and networking sections — gRPC, protobuf, service communication

#### Key Points to Cover in Your Answer:

**What gRPC is:**
```
gRPC = Google Remote Procedure Call
  → Application calls a FUNCTION on a remote server as if it were local
  → Abstraction: you don't write HTTP requests — you call methods
  → Built on HTTP/2 (multiplexing, streaming, binary)
  → Uses Protocol Buffers (protobuf) for serialization
```

**Protocol Buffers vs JSON:**
```
JSON:
  {"user_id": 123, "name": "John", "amount": 100.50}
  → 45 bytes as text
  → Schema-less (any client can add fields)
  → Slow to parse (string parsing)
  → Human readable

Protobuf:
  → Same data: ~12 bytes (binary encoding)
  → Schema required (defined in .proto file)
  → 5-10x faster to parse (binary)
  → Not human readable
  → Strong typing (schema enforced)

Performance impact:
  → 50-80% less bandwidth
  → 5-10x faster serialization
  → Critical for high-throughput inter-service communication
```

**4 gRPC communication patterns:**
```proto
service PaymentService {
  // 1. Unary (request-response, like REST)
  rpc ProcessPayment(PaymentRequest) returns (PaymentResponse);

  // 2. Server streaming (client sends one, server streams many)
  rpc StreamTransactions(AccountRequest) returns (stream Transaction);

  // 3. Client streaming (client streams many, server replies once)
  rpc BatchImport(stream ImportRecord) returns (ImportSummary);

  // 4. Bidirectional streaming (both sides stream simultaneously)
  rpc LiveFraudDetection(stream Transaction) returns (stream FraudAlert);
}
```

**gRPC on Kubernetes — load balancing challenge:**
```
Problem with standard Kubernetes Service + gRPC:
  → gRPC uses HTTP/2 which multiplexes many calls over ONE TCP connection
  → Standard K8s Service (kube-proxy) does L4 load balancing
  → Result: ALL requests go to the FIRST pod (the one that opened the connection)
  → No actual load balancing!

Solutions:

1. Headless Service + client-side load balancing (gRPC built-in):
   spec:
     clusterIP: None    # headless - returns all pod IPs
   # gRPC client does round-robin across all pod IPs

2. Service Mesh (Istio/Linkerd):
   → Envoy sidecar intercepts gRPC traffic
   → Performs L7 load balancing at the request level (not connection level)
   → No application changes needed

3. gRPC-specific load balancer:
   → Nginx with grpc_pass directive
   → AWS ALB with gRPC routing (HTTP/2 support required)
   annotations:
     kubernetes.io/ingress.class: alb
     alb.ingress.kubernetes.io/backend-protocol: GRPC
```

> 💡 **Interview tip:** The **gRPC load balancing problem on Kubernetes** is one of the most common production issues teams hit when adopting gRPC. The symptom: one pod is at 100% CPU while others are idle during high traffic. The root cause: all requests going through one long-lived HTTP/2 connection to one pod. Always mention this in interviews when discussing gRPC — it shows you have real production experience. The service mesh solution (Istio) is most common because it requires zero application changes.

---

### Q567 — Networking | Conceptual | Advanced

> Explain **WebSocket** support in AWS — how do you set up WebSocket connections through **API Gateway**, **ALB**, and **NLB**?
>
> What are the challenges of WebSockets in containerized environments? How do you handle **sticky sessions** and **connection persistence** in Kubernetes?

📁 **Reference:** `nawab312/AWS` and `nawab312/Kubernetes` — WebSocket, load balancing, sticky sessions sections

#### Key Points to Cover in Your Answer:

**WebSocket basics:**
```
HTTP:        request → response (connection closes)
WebSocket:   initial HTTP upgrade → persistent bidirectional connection
             Client and server can send messages at any time
             Connection persists for minutes/hours/days

Use cases:
  → Real-time dashboards (live stock prices, metrics)
  → Chat applications
  → Collaborative editing (Google Docs style)
  → Gaming (real-time multiplayer)
  → IoT (device telemetry streams)
```

**WebSocket on AWS API Gateway (WebSocket API):**
```
API Gateway WebSocket API:
  → Manages WebSocket connections for you (up to 10 minutes timeout)
  → Routes messages to Lambda functions or HTTP backends
  → Connection ID assigned per connection
  → State management: DynamoDB stores connection IDs

Routes:
  $connect    → Lambda runs when client connects
  $disconnect → Lambda runs when client disconnects
  $default    → Lambda handles all messages
  custom      → route by message attribute

Pushing messages to clients (from backend):
  aws apigatewaymanagementapi post-to-connection \
    --connection-id "abc123" \
    --data '{"type":"update","value":42}' \
    --endpoint-url https://xyz.execute-api.us-east-1.amazonaws.com/prod
```

**WebSocket on ALB:**
```
ALB natively supports WebSocket (HTTP → WS upgrade):
  → No special configuration needed
  → Connection stays open to backend target
  → ALB idle timeout must be ≥ expected connection duration

Critical config:
  # Default idle timeout: 60 seconds (kills most WebSocket connections!)
  aws elbv2 modify-load-balancer-attributes \
    --attributes Key=idle_timeout.timeout_seconds,Value=3600  # 1 hour
```

**WebSocket challenges in Kubernetes:**
```
Problem 1: Long-lived connections and pod restarts
  → Rolling update terminates pod → all WS connections dropped
  → Fix: preStop hook + graceful drain period
         terminationGracePeriodSeconds: 3600  # 1 hour (long!)
         lifecycle:
           preStop:
             exec:
               command: ["/bin/sh", "-c", "sleep 30"]  # let clients reconnect

Problem 2: Sticky sessions (connection affinity)
  → WebSocket client connects to pod A
  → New request to same server must reach pod A
  → Without sticky sessions: load balancer routes to pod B → connection lost

  Fix: enable session affinity on Service
  spec:
    sessionAffinity: ClientIP   # hash client IP → always route to same pod
    sessionAffinityConfig:
      clientIP:
        timeoutSeconds: 3600    # maintain affinity for 1 hour

Problem 3: Horizontal scaling
  → 10,000 clients on pod A, 0 on pod B (no rebalancing)
  → Fix: use pub/sub (Redis) for message routing
         → Any pod can receive message and forward to Redis
         → Redis broadcasts to all connected pods
         → Any pod holding the target connection delivers it
```

> 💡 **Interview tip:** The **idle timeout** mismatch is the most common WebSocket issue on ALB — default 60-second timeout silently kills WebSocket connections that have been quiet for 60 seconds (even though the client is still connected). The client typically implements a reconnect, but this creates a poor user experience. Always set ALB idle timeout to match your maximum expected WebSocket session duration, or implement **heartbeat pings** every 30 seconds from client to server to keep the connection alive.

---

### Q568 — Networking | Conceptual | Advanced

> Explain **load balancing algorithms** — what are **Round Robin**, **Least Connections**, **IP Hash**, **Random**, and **Weighted** algorithms? When does the choice of algorithm matter?
>
> How does Kubernetes kube-proxy choose which pod to send traffic to? How does this change with **IPVS mode** vs **iptables mode**?

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` and networking sections — load balancing algorithms

#### Key Points to Cover in Your Answer:

**Load Balancing Algorithms:**

```
Round Robin:
  → Requests sent to each backend in sequence: A, B, C, A, B, C...
  → Assumption: all requests equal cost, all backends equal capacity
  → Problem: some requests are 100x more expensive (long queries)
  → Best for: stateless services with roughly equal request cost

Least Connections:
  → Send new request to backend with FEWEST active connections
  → Better than Round Robin for variable-cost requests
  → Expensive requests naturally go to less-busy backends
  → Best for: variable-duration requests (DB queries, API calls)

IP Hash (Sticky):
  → hash(client_IP) % N_backends = consistent backend assignment
  → Same client always reaches same backend
  → No state sharing needed between backends
  → Best for: stateful sessions, WebSockets, connection affinity
  → Problem: hot spots if many clients from same IP (corporate NAT)

Weighted Round Robin:
  → Backend A gets 3× more requests than Backend B
  → Use: heterogeneous backends (faster server gets more traffic)
  → Use: canary deployments (10% to new version)

Random:
  → Purely random backend selection
  → Statistically equivalent to Round Robin at scale
  → Simpler to implement, surprisingly effective
  → Used by some modern load balancers (Netflix Ribbon default)
```

**Kubernetes kube-proxy — iptables mode (default):**
```
Service with 3 pods:
  kube-proxy creates iptables rules:
    25% chance → Pod A (1st rule, 1/4 match)
    33% chance → Pod B (1/3 of remaining)
    100% → Pod C (remaining)
  
  Result: statistically round-robin across pods
  
  Problems:
    - O(n) rules for n pods (10,000 pods = huge iptables ruleset)
    - Not truly "least connections"
    - Sequential rule matching = slow with many services
    - No health awareness (sends to pods even during termination)
```

**IPVS mode (production recommendation for large clusters):**
```
Benefits over iptables:
  → Hash table lookup O(1) vs O(n) → much faster at scale
  → Supports multiple algorithms: rr, lc, dh, sh, sed, nq
  → Native kernel load balancer (Linux Virtual Server)

Enable:
  kubectl edit configmap kube-proxy -n kube-system
  # mode: "ipvs"
  # ipvs.scheduler: "lc"  # least-connections

When to use IPVS:
  → Cluster with > 1,000 services
  → Need non-round-robin algorithms (least connections)
  → High connection rate (iptables processing becomes bottleneck)

Check current mode:
  kubectl describe pod kube-proxy -n kube-system | grep mode
```

> 💡 **Interview tip:** For Kubernetes at scale (> 1,000 services or > 10,000 pods), **IPVS mode is strongly recommended** over iptables. The iptables approach processes rules sequentially — adding a new rule at the END of a 10,000-rule chain means every packet traverses all 10,000 rules first. IPVS uses a hash table (O(1) lookup). In practice, IPVS mode reduces kube-proxy CPU usage by 60-70% in large clusters. This is a signal of deep Kubernetes networking knowledge that impresses interviewers.

---

### Q569 — Docker | Conceptual | Advanced

> Explain **Docker Compose** — what is it, what problems does it solve, and how does it differ from Kubernetes?
>
> Write a production-like Docker Compose file for a 3-tier application (React frontend + Node.js API + PostgreSQL). Explain the key directives: `depends_on`, `healthcheck`, `volumes`, `networks`, and `profiles`.

📁 **Reference:** `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md` — Docker Compose sections

#### Key Points to Cover in Your Answer:

**What Docker Compose solves:**
```
Problem: running multiple containers with correct networking and config
         is complex with individual docker run commands

docker run -d --name postgres -e POSTGRES_PASSWORD=secret -v pgdata:/var/lib/postgresql/data postgres
docker run -d --name api --link postgres -e DB_HOST=postgres -p 3000:3000 my-api
docker run -d --name frontend --link api -p 80:80 my-frontend

Compose solution: declare everything in one YAML file
docker-compose up    # starts all services with correct networking
docker-compose down  # stops and removes all containers
```

**When Docker Compose vs Kubernetes:**
```
Docker Compose:
  ✅ Local development (fast, simple)
  ✅ Single server deployment (no orchestration needed)
  ✅ CI/CD testing (spin up dependencies for integration tests)
  ✅ Small team, simple app

Kubernetes:
  ✅ Multi-node, high availability
  ✅ Auto-scaling, self-healing
  ✅ Rolling updates, rollback
  ✅ Complex microservices (> 10 services)
```

**Production-like Docker Compose:**
```yaml
version: '3.9'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: payments
      POSTGRES_USER: app
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
    - pgdata:/var/lib/postgresql/data    # named volume (persists)
    - ./init-scripts:/docker-entrypoint-initdb.d  # SQL init scripts
    secrets:
    - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d payments"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
    - backend
    restart: unless-stopped

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
      target: production           # multi-stage: use production stage
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: payments
      NODE_ENV: production
    secrets:
    - db_password
    ports:
    - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy  # wait for postgres healthcheck to pass
    healthcheck:
      test: ["CMD", "wget", "-q", "-O-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
    - backend
    - frontend
    restart: unless-stopped
    deploy:                        # resource limits
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M

  frontend:
    build: ./frontend
    ports:
    - "80:80"
    - "443:443"
    volumes:
    - ./ssl:/etc/nginx/ssl:ro      # bind mount (read-only)
    depends_on:
      api:
        condition: service_healthy
    networks:
    - frontend
    restart: unless-stopped
    profiles:               # only starts with --profile prod
    - prod

  # Development-only: pgAdmin
  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@company.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
    - "5050:80"
    profiles:
    - dev                   # only starts with --profile dev

volumes:
  pgdata:                   # named volume managed by Docker

networks:
  backend:                  # postgres ↔ api (no external access)
  frontend:                 # api ↔ frontend (no postgres access)

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

**Key directives explained:**
```
depends_on + healthcheck:
  → condition: service_healthy = wait for healthcheck to pass
  → Default (no condition): just wait for container to start (not ready)
  → Always use service_healthy for databases

profiles:
  → docker-compose --profile dev up  → starts dev profile services too
  → docker-compose --profile prod up → starts prod profile services
  → Default: starts services with NO profile (api, frontend, postgres)

networks:
  → frontend network: api + frontend communicate
  → backend network: postgres + api communicate
  → postgres CANNOT reach frontend (network isolation!)
```

> 💡 **Interview tip:** `depends_on` without `condition: service_healthy` is a very common mistake — it only waits for the container to **start**, not for the service inside to be **ready**. PostgreSQL takes 10-15 seconds to initialize after the container starts. Without `service_healthy`, the API starts, tries to connect to PostgreSQL, fails, and crashes. The fix is always: `depends_on: postgres: condition: service_healthy` combined with a proper `healthcheck` on the postgres service.

---

### Q570 — Docker | Conceptual | Advanced

> Explain the difference between **ENTRYPOINT** and **CMD** in a Dockerfile. What happens when you combine them? What is the **exec form** vs **shell form** and why does the exec form matter for signal handling?

📁 **Reference:** `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md` — Dockerfile ENTRYPOINT CMD sections

#### Key Points to Cover in Your Answer:

**CMD vs ENTRYPOINT:**
```
CMD:
  → Default command to run when container starts
  → Easily overridden: docker run myimage /bin/bash
  → Provides defaults for an executable

ENTRYPOINT:
  → The main executable — always runs, cannot be easily overridden
  → docker run myimage arg1 → appends arg1 to ENTRYPOINT
  → Override requires: docker run --entrypoint /bin/bash myimage
  → Makes container behave like an executable
```

**Combination patterns:**
```dockerfile
# Pattern 1: ENTRYPOINT as executable, CMD as default args
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
→ docker run myimage             = nginx -g "daemon off;"
→ docker run myimage -p          = nginx -p (override CMD, keep ENTRYPOINT)

# Pattern 2: ENTRYPOINT as wrapper script, CMD as the app command
ENTRYPOINT ["/docker-entrypoint.sh"]  # setup script
CMD ["postgres"]                       # default app to run
→ entrypoint.sh receives "postgres" as argument
→ entrypoint.sh sets up environment, then exec "$@" (runs postgres)
→ docker run myimage redis         = runs redis instead (still through entrypoint.sh)

# Pattern 3: CMD only (flexible, common for dev images)
CMD ["python", "app.py"]
→ docker run myimage              = python app.py
→ docker run myimage bash         = bash (completely overrides CMD)
```

**Exec form vs Shell form — CRITICAL for signals:**
```dockerfile
# SHELL FORM (BAD for production):
CMD python app.py
# Becomes: /bin/sh -c "python app.py"
# PID 1 = /bin/sh
# PID 2 = python
# docker stop → SIGTERM sent to PID 1 (/bin/sh)
# /bin/sh does NOT forward SIGTERM to python!
# Python never receives SIGTERM → 10 second wait → SIGKILL
# = ungraceful shutdown, dropped requests

# EXEC FORM (CORRECT):
CMD ["python", "app.py"]
# PID 1 = python (runs directly, no shell wrapper)
# docker stop → SIGTERM sent to PID 1 (python)
# Python receives SIGTERM → handles gracefully → exits cleanly

# Rule: ALWAYS use exec form for CMD and ENTRYPOINT
# ["executable", "arg1", "arg2"]  ← exec form (JSON array)
# executable arg1 arg2            ← shell form (string, BAD)
```

**tini as PID 1 init process:**
```dockerfile
# For apps that don't handle signals well or create zombie processes
FROM ubuntu:22.04
RUN apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]    # tini as init process
CMD ["python", "app.py"]

# tini:
# → Forwards signals properly to child processes
# → Reaps zombie processes
# → Properly handles PID 1 responsibilities
```

> 💡 **Interview tip:** **Shell form vs exec form** is one of the most commonly tested Docker questions because it directly impacts production reliability. Shell form means graceful shutdown NEVER works — `docker stop` waits 10 seconds then sends SIGKILL, causing all in-flight requests to be dropped. This is a very common root cause of 502 errors during Kubernetes rolling updates. The fix is always: convert shell form CMD/ENTRYPOINT to exec form (JSON array syntax).

---

### Q571 — Linux | Conceptual | Advanced

> Explain **Linux performance analysis tools** — what is the correct order to investigate a performance problem?
>
> Walk through using **`top`, `vmstat`, `iostat`, `netstat`/`ss`, `strace`, `lsof`, `perf`** to diagnose these scenarios: (1) high CPU with low throughput, (2) high I/O wait, (3) process stuck in D state.

📁 **Reference:** `nawab312/DSA` → `Linux` — performance tools, analysis sections

#### Key Points to Cover in Your Answer:

**Performance investigation order (Brendan Gregg's USE method):**
```
For each resource (CPU, memory, storage, network):
  U — Utilization (how busy is it?)
  S — Saturation (how much extra work is queued?)
  E — Errors (are there errors occurring?)

Tools in order:
1. uptime/top/htop     → quick overview (load average, CPU, memory)
2. vmstat 1            → CPU, memory, I/O, context switches per second
3. iostat -x 1         → disk I/O breakdown per device
4. netstat -s / ss -s  → network statistics, connection states
5. top -H              → thread-level CPU breakdown
6. perf top            → kernel-level CPU profiling
7. strace -p PID       → what syscalls is a process making?
8. lsof -p PID         → what files/sockets does a process have open?
```

**Scenario 1: High CPU, low throughput:**
```bash
# Step 1: Which process?
top                       # find PID with highest CPU
top -H -p <PID>           # which THREAD in that process?

# Step 2: What is it doing? (syscall level)
strace -p <PID> -c        # summary of syscalls (what's taking most time)
strace -p <PID>           # live syscall trace

# Common findings:
# Many futex() calls → lock contention (mutex/semaphore)
# Many read()/write() → I/O bound
# Many clone() → spawning too many threads
# Tight loop (no blocking syscalls) → CPU-bound computation

# Step 3: Function-level profiling
perf top -p <PID>         # which functions are hot?
perf record -p <PID> -g   # record with call graph
perf report               # show flame graph data
```

**Scenario 2: High I/O wait:**
```bash
# Step 1: Confirm I/O wait
vmstat 1
# CPU: wa column (I/O wait %) - if > 20% consistently → I/O bottleneck

# Step 2: Which device?
iostat -x 1
# Look for: %util (100% = device saturated), await (request wait time)
# High r/s or w/s = high request rate
# High avgqu-sz = queue building up = saturation

# Step 3: Which process?
iotop                    # which processes are doing most I/O
pidstat -d 1             # I/O statistics per process

# Step 4: What files?
lsof -p <PID>            # which files the high-I/O process has open
```

**Scenario 3: Process stuck in D state (uninterruptible sleep):**
```bash
# D state = waiting for I/O that never completes

# Find D state processes
ps aux | grep "^[^ ]* D"
# OR
for pid in /proc/[0-9]*; do
  state=$(cat $pid/status 2>/dev/null | grep "^State" | awk '{print $2}')
  if [ "$state" = "D" ]; then
    echo "D state: $pid $(cat $pid/comm)"
  fi
done

# What is it waiting for?
cat /proc/<PID>/wchan     # kernel function where process is sleeping
# "nfs_execute_read" → NFS hang
# "jbd2_journal_commit_transaction" → filesystem journal hang
# "io_schedule" → generic I/O wait

# Stack trace
cat /proc/<PID>/stack     # kernel stack showing wait path

# Solutions:
# NFS hang → unmount/remount NFS, fix NFS server
# Disk hang → check dmesg for I/O errors, disk failing
# Cannot kill -9 (D state immune to signals)
# → Reboot may be required if disk truly failed
```

> 💡 **Interview tip:** The **D state process** question is a classic Linux interview topic because it catches engineers who think `kill -9` kills everything. D state processes are immune to ALL signals including SIGKILL — the process is waiting in kernel space for I/O, and the kernel won't interrupt it until the I/O completes. The only ways to resolve it: fix the underlying I/O issue (NFS server, disk), or reboot. In Kubernetes, this manifests as pods stuck in `Terminating` state forever.

---

### Q572 — Linux | Conceptual | Advanced

> Explain the **`/proc` filesystem** — what is it and why is it useful for system debugging?
>
> Walk through key `/proc` paths every SRE should know: `/proc/meminfo`, `/proc/cpuinfo`, `/proc/net/tcp`, `/proc/<pid>/fd`, `/proc/<pid>/maps`, and `/proc/sys/kernel` tuning parameters.

📁 **Reference:** `nawab312/DSA` → `Linux` — /proc filesystem, kernel parameters sections

#### Key Points to Cover in Your Answer:

**What /proc is:**
```
/proc = virtual filesystem (not on disk)
        Kernel exposes live system state as "files"
        Reading a file = querying the kernel in real time
        Writing to some files = configuring kernel parameters

Everything in /proc is generated dynamically on read
No actual disk I/O — pure kernel memory access
```

**Key /proc paths:**
```bash
# System-wide memory info
cat /proc/meminfo
# MemTotal:      16384000 kB  (total RAM)
# MemFree:        2048000 kB  (completely unused)
# MemAvailable:  10240000 kB  ← THIS is what matters (free + reclaimable cache)
# Cached:         8192000 kB  (page cache - normal, good)
# SwapUsed:        512000 kB  (if non-zero: RAM pressure)
# Dirty:           102400 kB  (waiting to be written to disk)
# Writeback:           0 kB  (being written now)

# CPU information
cat /proc/cpuinfo | grep -E "processor|model name|cpu MHz|cache size" | head -20
# Shows: physical/logical cores, current frequency, cache size
# CPU throttling: compare cpu MHz vs max frequency

# Network connections (raw - ss/netstat is easier but /proc is the source)
cat /proc/net/tcp
# Columns: sl local_address rem_address st tx_queue rx_queue
# State 0A = LISTEN, 01 = ESTABLISHED, 06 = TIME_WAIT

# All open file descriptors for a process
ls -la /proc/<PID>/fd/
# Each symlink = one open file descriptor
# Count: ls /proc/<PID>/fd | wc -l
# Too many = file descriptor leak

# Memory map of process (virtual memory layout)
cat /proc/<PID>/maps
# Shows: memory regions, permissions, mapped files
# Find: which shared libraries loaded, anonymous mmap allocations

# Process status and resource usage
cat /proc/<PID>/status
# VmRSS: physical memory used
# VmPeak: peak virtual memory
# Threads: number of threads
# FDSize: file descriptor table size
```

**Kernel parameter tuning via /proc/sys:**
```bash
# Network tuning
echo 65535 > /proc/sys/net/core/somaxconn          # max listen backlog
echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse           # reuse TIME_WAIT sockets
echo "1024 65535" > /proc/sys/net/ipv4/ip_local_port_range  # ephemeral ports

# Memory tuning
echo 1 > /proc/sys/vm/overcommit_memory            # allow memory overcommit
echo 10 > /proc/sys/vm/swappiness                  # avoid swap (0-100)
echo 500 > /proc/sys/vm/dirty_writeback_centisecs  # write dirty pages sooner

# Make persistent (survives reboot) via sysctl.conf:
echo "net.core.somaxconn = 65535" >> /etc/sysctl.conf
sysctl -p   # apply immediately
```

**Useful /proc debugging commands:**
```bash
# Find which process has a port open
grep -l "0000:1F90" /proc/*/net/tcp 2>/dev/null  # port 8080 in hex
# Or easier: ss -tlnp | grep :8080

# Check process namespaces (useful for containers)
ls -la /proc/<PID>/ns/
# Shows: ipc, mnt, net, pid, uts namespaces
# Containers have different ns than host

# Check if process is inside a container
cat /proc/<PID>/cgroup
# Contains: /kubepods/... → Kubernetes container
# Contains: /docker/...  → Docker container
# Contains: root cgroup → bare metal process
```

> 💡 **Interview tip:** `/proc/<PID>/fd` is your best friend for debugging **"too many open files"** errors — `ls /proc/<PID>/fd | wc -l` shows exactly how many file descriptors a process currently has open. If it's close to `ulimit -n`, you have a file descriptor leak. `ls -la /proc/<PID>/fd` shows WHAT is open (sockets, files, pipes). This is faster and more accurate than lsof for quick debugging.

---

### Q573 — AWS Cost | Conceptual | Advanced

> Explain **AWS Savings Plans** vs **Reserved Instances** vs **Spot Instances** vs **On-Demand**. What is the actual pricing difference and when would you use each?
>
> For an EKS cluster running 24/7 with predictable workloads, design the **optimal cost strategy** mixing all three purchasing options.

📁 **Reference:** `nawab312/AWS` — Cost optimization, purchasing options sections

#### Key Points to Cover in Your Answer:

**Purchasing options comparison:**

| Option | Commitment | Discount | Flexibility | Use case |
|---|---|---|---|---|
| On-Demand | None | 0% | Full | Unpredictable, short-term |
| Spot | None | 70-90% | Instance can be reclaimed | Fault-tolerant, batch |
| Reserved Instances | 1 or 3 years | 40-75% | Fixed instance type/region | Stable, predictable baseline |
| Savings Plans | 1 or 3 years | 40-66% | Any instance in family (Compute SP) | Flexible long-term commitment |

**Savings Plans types:**
```
Compute Savings Plans (most flexible):
  → Applies to: EC2, Fargate, Lambda
  → Any instance family, size, region, OS
  → Discount: up to 66%
  → Commitment: $/hour (e.g., $5/hour)

EC2 Instance Savings Plans:
  → Specific instance family + region
  → Any size and OS within that family
  → Discount: up to 72%

Recommendation: Compute Savings Plans for EKS
  → As Karpenter changes instance types for cost → savings plan still applies
```

**Reserved Instances:**
```
Standard RI:
  → Fixed: instance type + region + OS
  → Maximum discount (75%)
  → Can be sold in RI Marketplace if no longer needed

Convertible RI:
  → Can exchange for different instance type (same value)
  → Lower discount (54%)
  → Cannot sell in Marketplace

Savings Plans vs RI:
  Savings Plans: more flexible (any instance in family)
  RI: higher discount but less flexible
  Recommendation: Savings Plans unless you know EXACTLY what instances you'll run
```

**Optimal EKS cluster cost strategy:**
```
Assumption: 24/7 cluster, 3 node groups

Step 1: Analyze baseline (minimum nodes always running)
  → 3 nodes always on (control plane + critical services)
  → Purchase: 3-year Compute Savings Plan for baseline
  → Discount: ~60%

Step 2: Predictable scale
  → 5 additional nodes during business hours (8am-8pm weekdays)
  → Purchase: 1-year Compute Savings Plan for average usage
  → Discount: ~40%

Step 3: Variable/burst capacity
  → Karpenter with Spot instances for remaining scale
  → Mix: 5 instance families (m5, m5a, m6i, c5, c6i)
  → Discount: ~70-80% vs On-Demand

Result (100 nodes example):
  20 nodes: 3-year Savings Plan → $6,000/month → saves $9,000/month
  30 nodes: 1-year Savings Plan → $10,500/month → saves $5,500/month
  50 nodes: Spot instances → $5,000/month → saves $12,000/month
  Total savings: ~$26,500/month vs all On-Demand
```

**Key tools:**
```bash
# AWS Cost Explorer Savings Plan recommendations
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days THIRTY_DAYS

# AWS Compute Optimizer for right-sizing
aws compute-optimizer get-ec2-instance-recommendations \
  --filters name=Finding,values=OVER_PROVISIONED

# Kubecost for namespace-level Kubernetes cost breakdown
# Shows: which team/namespace is spending what → accountability
```

> 💡 **Interview tip:** **Never purchase Reserved Instances for Kubernetes nodes** — Karpenter can change instance types based on availability and cost, so a reserved t3.xlarge is useless if Karpenter launches m5.xlarge instead. The correct tool for Kubernetes long-term commitments is **Compute Savings Plans** — they apply to ANY EC2 instance regardless of type, so Karpenter's flexibility is preserved while still getting the discount.

---

### Q574 — AWS Cost | Scenario-Based | Advanced

> Your company's AWS bill is **$200,000/month** and the CTO wants to reduce it by 30% ($60,000/month) within 6 months.
>
> Walk through your **FinOps approach** — how do you identify waste, prioritize actions, implement tagging for accountability, and build a culture of cost ownership across engineering teams.

📁 **Reference:** `nawab312/AWS` — FinOps, cost optimization, tagging strategy sections

#### Key Points to Cover in Your Answer:

**Phase 1: Visibility (Month 1)**
```
1. Enable AWS Cost Explorer + CUR (Cost and Usage Report)
   → CUR → S3 → Athena → query: which service/account/team costs what?

2. Implement tagging strategy (mandatory):
   Required tags:
     Environment:  dev/staging/production
     Team:         payments/auth/platform
     Service:      payment-api/auth-service
     CostCenter:   CC-001/CC-002
   
   Enforce via:
     → SCP: deny resource creation without required tags
     → AWS Config rule: required-tags

3. Activate Compute Optimizer
   → Run for 14 days → generates right-sizing recommendations
   → Typically: 20-40% of instances are over-provisioned

4. Run Trusted Advisor
   → Identify: idle EC2, unused EBS, underutilized RDS, unused EIPs
```

**Phase 2: Quick wins (Months 1-2, target $20K/month savings)**
```
Unattached resources (typically $5-15K/month in large orgs):
  → Unattached EBS volumes: aws ec2 describe-volumes --filter State=available
  → Unused Elastic IPs: aws ec2 describe-addresses (attached=false)
  → Idle load balancers: 0 requests/day
  → Stopped instances still paying for EBS
  Action: delete or resize

Right-sizing EC2 (typically 15-30% savings):
  → Compute Optimizer shows: "this m5.xlarge uses avg 8% CPU → recommend t3.medium"
  → Schedule: dev/staging instances off nights and weekends (saves 65%)
    aws scheduler create-schedule --name stop-dev-instances \
      --schedule-expression "cron(0 20 ? * MON-FRI *)"

Storage optimization ($5-10K/month):
  → Move S3 data to Intelligent Tiering (auto-moves to cheaper tiers)
  → Delete old EBS snapshots (retain last 30 days only)
  → Enable S3 Lifecycle policies (30 days → Standard-IA, 90 days → Glacier)
```

**Phase 3: Architectural changes (Months 3-4, target $25K/month savings)**
```
Spot instances for eligible workloads ($15K/month):
  → EKS worker nodes → Karpenter with Spot + fallback On-Demand
  → Batch jobs → 100% Spot
  → Dev/staging environments → 100% Spot

Savings Plans purchase ($15K/month):
  → Analyze 14-day Compute Optimizer data
  → Purchase 1-year Compute Savings Plans for baseline
  → Start with 1-year (lower risk than 3-year)

NAT Gateway optimization ($3-5K/month):
  → Add VPC endpoints for S3, DynamoDB, ECR (free gateway endpoints)
  → Reduces NAT Gateway data processing by 40-60%
```

**Phase 4: Culture (Months 5-6)**
```
Cost ownership by team:
  → Monthly cost report per team (based on tags)
  → Each team has cost budget and sees their spend
  → Engineering managers own team AWS budgets

Show back / charge back:
  → Internal invoicing per team
  → Makes engineers CARE about resource waste
  → "Your team spent $15,000 on idle dev instances last month"

FinOps champions:
  → One engineer per team as FinOps champion
  → Reviews team costs monthly
  → Raises optimization opportunities
```

> 💡 **Interview tip:** The **tagging + chargeback** combination is the long-term solution for sustainable cost optimization. Technical optimizations (right-sizing, Spot) deliver one-time savings. Culture change (teams owning their costs) delivers ongoing optimization. When engineers see their team's monthly bill, they naturally stop leaving dev environments running 24/7. Quantify: "after implementing team cost dashboards, dev environment costs dropped 60% in 3 months as teams started using scheduling and right-sizing."

---

### Q575 — All Topics | System Design | Advanced

> You are interviewing for a **Senior SRE role at a fintech company**. The panel asks:
>
> *"Walk us through how you would design and implement complete observability, reliability, and operational excellence for a microservices platform — including monitoring strategy, incident management, capacity planning, disaster recovery, and how you measure the success of your SRE function."*
>
> This is a 30-minute open-ended system design question. Give a comprehensive answer.

📁 **Reference:** All repositories — comprehensive SRE system design

#### Key Points to Cover in Your Answer:

**1. Observability Stack (Metrics + Logs + Traces):**
```
Metrics: Prometheus + Grafana
  → Prometheus Operator for auto-discovery (ServiceMonitor)
  → Thanos/Mimir for long-term retention (1 year)
  → SLO dashboards per service (error budget burning rate)
  → Alerting: multi-window burn rate (Google SRE method)

Logs: ELK Stack (or Loki for K8s-native)
  → Fluentbit DaemonSet → Elasticsearch
  → Structured JSON logging (no Grok parsing needed)
  → Log retention: hot (7 days ES) → warm (30 days) → cold (S3, 1 year)
  → Index per service per day

Traces: Jaeger/Tempo + OpenTelemetry
  → OTel SDK in each service (language-agnostic)
  → Trace sampling: 100% for errors, 1% for success
  → Link metrics → traces via exemplars
```

**2. SLO Definition and Error Budgets:**
```
Per service SLOs:
  Payment API:   99.99% availability, p99 < 200ms
  Auth Service:  99.95% availability, p99 < 100ms
  Notification:  99.9% availability, p99 < 500ms

Error budget policy:
  Budget > 50%: full feature velocity, can take risks
  Budget 25-50%: slow down, focus on reliability
  Budget < 25%: freeze feature work, only reliability improvements
  Budget 0%: incident response, postmortem, root cause fix

Burn rate alerting: multi-window (2h/5m, 6h/30m, 1d/2h)
```

**3. Incident Management:**
```
Severity levels:
  P0: complete outage (all engineers paged, war room)
  P1: degraded service, SLA at risk (on-call paged)
  P2: partial degradation (ticket, next business day)

On-call:
  → PagerDuty rotations: 1 week on, 4 weeks off
  → Escalation: on-call → team lead → engineering director
  → Runbooks for all P1 alerts
  → Target: < 2 P1 pages per shift

Postmortem:
  → Required for all P0/P1 incidents
  → Blameless, within 3 business days
  → Action items tracked in Jira, reviewed monthly
```

**4. Reliability Architecture:**
```
Multi-AZ deployment (every service):
  → minReplicas: 3 (one per AZ)
  → PodDisruptionBudget: maxUnavailable: 1
  → TopologySpreadConstraints: maxSkew: 1

Circuit breakers:
  → Istio: outlier detection (eject failing pods)
  → Application: Hystrix/Resilience4j for downstream calls

Graceful degradation:
  → Feature flags (ops toggles) for every risky feature
  → Fallback responses for non-critical downstream failures
  → Rate limiting (API Gateway + application-level)
```

**5. Capacity Planning:**
```
Monthly capacity review:
  → predict_linear() on key metrics (CPU, connections, storage)
  → 3-month ahead projections for infrastructure decisions
  → Correlate with business growth projections

Auto-scaling:
  → HPA: CPU + custom metrics (SQS queue depth via KEDA)
  → Karpenter: node auto-provisioning
  → RDS storage autoscaling enabled

Load testing:
  → Quarterly GameDays: test failover, load, chaos
  → k6/Gatling for performance testing in CI/CD
```

**6. Measuring SRE Success (DORA + SRE metrics):**
```
Engineering metrics (DORA):
  Deployment frequency: > 3/day (elite)
  Lead time: < 1 hour (elite)
  Change failure rate: < 5%
  MTTR: < 30 minutes

Reliability metrics:
  SLO compliance: all services > 99.9%
  Error budget remaining: > 25% each month
  Alert-to-page ratio: < 20% (80% of alerts are tickets, not pages)
  On-call page rate: < 2/shift

Toil metrics:
  Toil ratio: < 50% of SRE time
  Automation additions: > 2 toil eliminations per sprint
```

> 💡 **Interview tip:** The best way to answer this question is with the **"crawl-walk-run"** framing: (1) first establish visibility and stop the bleeding (monitoring, on-call), (2) then improve reliability (SLOs, error budgets, auto-scaling), (3) then optimize and scale (capacity planning, chaos engineering, FinOps). This shows you have a practical, phased approach rather than trying to implement everything at once. Also always tie SRE work back to business outcomes: "error budget policy means engineers can move fast when the service is healthy, and are forced to slow down when it is fragile — this creates the right incentives for both velocity AND reliability."

---

## Coverage Summary — Q526–Q575

| Topic | Questions | Status |
|---|---|---|
| Prometheus Architecture (data model, Operator, PromQL joins, Mimir) | Q526–Q530 | ✅ Complete |
| HTTP 502/503/504 (conceptual + ALB + Nginx + rolling update fix) | Q531–Q533 | ✅ Complete |
| Aurora (vs RDS, failover, Global DB) | Q534–Q535 | ✅ Complete |
| Route53 (routing policies, health checks, failover debug) | Q536–Q537 | ✅ Complete |
| IAM Permission Boundaries + policy evaluation | Q538–Q539 | ✅ Complete |
| API Gateway REST vs HTTP vs WebSocket + 429 debug | Q540–Q541 | ✅ Complete |
| EC2 Placement Groups + instance families + Nitro | Q542–Q543 | ✅ Complete |
| ECR lifecycle policies + scanning + replication | Q544 | ✅ Complete |
| SQS FIFO vs Standard + DLQ + message groups | Q545–Q546 | ✅ Complete |
| SonarQube + Nexus + GitHub Reusable Workflows | Q547–Q549 | ✅ Complete |
| Terragrunt + Terraform Cloud/Sentinel | Q550–Q551 | ✅ Complete |
| Ansible Vault + Dynamic Inventory | Q552–Q553 | ✅ Complete |
| PV Access Modes + Velero + Helmfile + Kustomize | Q554–Q557 | ✅ Complete |
| 12-Factor App + Postmortem + Toil + On-call | Q558–Q561 | ✅ Complete |
| PushGateway + PromQL ranking + Logstash Grok + Mimir/Tempo | Q562–Q565 | ✅ Complete |
| gRPC + WebSocket + Load balancer algorithms | Q566–Q568 | ✅ Complete |
| Docker Compose + ENTRYPOINT vs CMD | Q569–Q570 | ✅ Complete |
| Linux perf tools + /proc filesystem | Q571–Q572 | ✅ Complete |
| Savings Plans vs RI + FinOps approach | Q573–Q574 | ✅ Complete |
| SRE system design capstone | Q575 | ✅ Complete |

---

## Final Question Bank: **Q1–Q575 = 575 Questions**

**Zero repetition. 100% unique. Every major topic covered.**

---

*Most Commonly Asked Gaps — Q526–Q575*
*DevOps / SRE Interview Preparation — nawab312 GitHub repositories*
