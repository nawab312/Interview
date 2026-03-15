# DevOps / SRE Interview Questions — Q1–Q45 Enhanced
## Key Points + Interview Tips Added to Every Question

---

### Q1 — Kubernetes | Conceptual | Beginner

> When a Pod is scheduled on a Node and the kubelet starts it, the Pod goes through several phases. Can you walk me through the **Pod lifecycle phases** — from `Pending` to `Running` to `Succeeded/Failed` — and explain what happens at each stage?

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`

#### Key Points to Cover:
```
Pending:
  → Pod accepted by API server, stored in etcd
  → Scheduler not yet assigned a node OR image being pulled
  → Sub-states: Unschedulable (no node fits), ContainerCreating (pulling image)

Running:
  → At least one container running, starting, or restarting
  → Pod is bound to a node, all containers created

Succeeded:
  → ALL containers exited with code 0
  → Will not be restarted
  → Typical for Jobs/batch workloads

Failed:
  → At least one container exited with non-zero code OR was terminated
  → restartPolicy determines what happens next

Unknown:
  → Node lost contact with API server
  → Node not responding to kubelet heartbeats

Container states (inside Running phase):
  Waiting:    container not running yet (pulling image, initializing)
  Running:    executing normally
  Terminated: finished (success or failure)

restartPolicy:
  Always:    always restart (default for Deployment)
  OnFailure: restart only on non-zero exit (for Jobs)
  Never:     never restart
```

> 💡 **Interview tip:** Most candidates only list Pending → Running → Succeeded/Failed. Stand out by explaining the **container states within a Running pod** (Waiting/Running/Terminated) and the **restartPolicy impact**. Also mention `Unknown` — it shows you understand what happens during node failures, which is a real production concern.

---

### Q2 — Kubernetes | Scenario-Based | Beginner-Intermediate

> You deploy an application and the Pod shows status `Running`, but your users are still unable to reach the service. You check and the Pod has been Running for 2 minutes. **What are the possible reasons, and how would you diagnose this?**

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md` and `03_NETWORKING.md`

#### Key Points to Cover:
```
Step 1 — Check Service selector matches Pod labels:
  kubectl get svc my-service -o yaml | grep selector
  kubectl get pods --show-labels
  → Most common cause: selector mismatch (typo in label)

Step 2 — Check Endpoints populated:
  kubectl get endpoints my-service
  → If "none": selector mismatch or no ready pods

Step 3 — Check readiness probe:
  kubectl describe pod <pod> | grep -A10 Readiness
  → Pod Running but readiness failing → not added to endpoints

Step 4 — Check Service port vs container port:
  Service targetPort must match containerPort in Pod spec

Step 5 — Check NetworkPolicy:
  kubectl get networkpolicy -A
  → Policy might be blocking traffic

Step 6 — Test connectivity directly:
  kubectl exec -it debug-pod -- curl http://<pod-ip>:<port>
  kubectl port-forward pod/<pod> 8080:80

Step 7 — Check Ingress (if external):
  kubectl describe ingress my-ingress
  → Backend service name/port correct?
  → Ingress controller running?
```

> 💡 **Interview tip:** Always start with `kubectl get endpoints` — this single command immediately tells you if the Service is routing to any pods. Empty endpoints = selector problem or readiness probe failing. This is the fastest diagnostic. Mention the **selector mismatch** as the #1 cause — it's responsible for ~50% of these issues in real environments.

---

### Q3 — Kubernetes | Conceptual | Intermediate

> Can you explain the difference between a **Deployment** and a **StatefulSet**? When would you choose one over the other? Give a real-world example for each.

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`

#### Key Points to Cover:
```
Deployment:
  → Stateless workloads
  → Pods are interchangeable — any pod can serve any request
  → Pods get random names (my-app-7d9f8b-xkz4p)
  → PVCs shared or not used
  → Scaling: add/remove any pod
  → Rolling update: random pod order
  → Use: web servers, APIs, microservices

StatefulSet:
  → Stateful workloads needing stable identity
  → Pods have fixed ordinal names (my-db-0, my-db-1, my-db-2)
  → Stable network identity: my-db-0.my-db-svc.namespace.svc.cluster.local
  → Each pod gets its OWN PVC (volumeClaimTemplates)
  → Scaling: pods created/deleted in order (0,1,2... or 2,1,0)
  → Rolling update: in reverse order (highest ordinal first)
  → Use: databases (PostgreSQL, MySQL), Kafka, Elasticsearch, Redis Sentinel

Key differences table:
  Feature          | Deployment      | StatefulSet
  Pod names        | Random          | Ordered (app-0, app-1)
  PVC per pod      | Shared          | Separate per pod
  Network identity | Random          | Stable DNS per pod
  Scale order      | Random          | Sequential
  Use case         | Stateless       | Stateful

Real examples:
  Deployment: nginx web server, payment-api, auth-service
  StatefulSet: PostgreSQL cluster, Kafka brokers, Elasticsearch nodes
```

> 💡 **Interview tip:** The **stable network identity** of StatefulSet is what makes it critical for databases. In a PostgreSQL cluster, the primary is always `postgres-0` and replicas are `postgres-1`, `postgres-2`. If pods had random names (like Deployments), replicas wouldn't know where to connect for replication. This stable DNS is the core reason StatefulSets exist — not just for persistent storage.

---

### Q4 — Kubernetes | Troubleshooting | Intermediate

> You have a Pod stuck in **`CrashLoopBackOff`**. Walk me through your **step-by-step troubleshooting process** to identify and fix the root cause.

📁 **Reference:** `nawab312/Kubernetes` → `14_TROUBLESHOOTING.md`

#### Key Points to Cover:
```
Step 1 — Get events and status:
  kubectl describe pod <pod-name> -n <namespace>
  → Look at: Events section (image pull errors, OOMKilled, etc.)
  → Look at: Last State (previous container exit code)
  → Exit code 1: application error
  → Exit code 137: OOMKilled (memory limit exceeded)
  → Exit code 139: segfault
  → Exit code 143: SIGTERM (graceful shutdown signal)

Step 2 — Check current logs:
  kubectl logs <pod-name> -n <namespace>

Step 3 — Check previous container logs (most useful):
  kubectl logs <pod-name> -n <namespace> --previous
  → Shows logs from the container before it crashed
  → This is the most important command for CrashLoopBackOff

Step 4 — Common root causes:
  a) Application error: bad config, missing env var, DB connection failed
     Fix: check env vars, ConfigMap, Secrets
  b) OOMKilled: memory limit too low or memory leak
     Fix: increase memory limit or fix leak
  c) Liveness probe failing: app not responding to health check
     Fix: adjust probe settings or fix app
  d) Missing required file/config:
     kubectl exec -it <pod> -- sh (if pod is briefly up)
  e) Wrong image or entrypoint:
     Check image name, tag, and command in spec

Step 5 — Override entrypoint to debug:
  spec:
    containers:
    - command: ["sleep", "3600"]  # override to keep pod running
  Then: kubectl exec -it <pod> -- sh
```

> 💡 **Interview tip:** `kubectl logs --previous` is the single most important command for CrashLoopBackOff — without it you only see logs from the current (just-started) container which may be empty. The previous container's logs show WHY it crashed. Also, **exit code 137 = OOMKilled** is critical to memorize — it means the container was killed by the Linux OOM killer because it exceeded its memory limit, not because of an application error.

---

### Q5 — Terraform | Conceptual | Intermediate

> What is the difference between **`terraform plan`** and **`terraform apply`**? Also, what is the purpose of the **Terraform state file** and what risks come with it?

📁 **Reference:** `nawab312/Terraform` — state management and core workflow sections

#### Key Points to Cover:
```
terraform plan:
  → Read-only operation (never modifies infrastructure)
  → Compares: desired state (.tf files) vs current state (state file)
  → Shows: what will be created, changed, or destroyed
  → Output: +, ~, - symbols for add, change, destroy
  → Safe to run anytime
  → Use: review before applying, CI/CD PR checks
  terraform plan -out=tfplan    # save plan to file
  terraform show tfplan         # review saved plan

terraform apply:
  → Executes changes shown in plan
  → Without -auto-approve: shows plan and asks for confirmation
  → With saved plan: applies exactly what was planned (no surprises)
  terraform apply tfplan        # apply saved plan (no confirmation prompt)
  terraform apply -auto-approve # skip confirmation (CI/CD only)

State file (terraform.tfstate):
  Purpose:
    → Maps Terraform resources to real infrastructure
    → Tracks metadata (IDs, attributes) of created resources
    → Enables: plan diffs, imports, destroy operations
    → Without it: Terraform doesn't know what exists

  Risks:
    → Contains sensitive data (passwords, keys) in plaintext
    → Must be shared across team (remote backend needed)
    → State corruption if two applies run simultaneously (use locking)
    → Manual edits can break everything

  Best practices:
    → Remote backend: S3 + DynamoDB (AWS)
    → Never commit to Git
    → Enable S3 versioning for state backup
    → Enable DynamoDB locking to prevent concurrent applies
```

> 💡 **Interview tip:** The most important point about state: **`terraform plan` reads from state file, not from real AWS**. If someone manually changed a resource in AWS without updating state, `terraform plan` won't detect it (until `terraform refresh`). This is called **drift** and is a common production issue. Mention `terraform plan -refresh=false` for faster plans when drift is not a concern, and `terraform plan -refresh=true` (default) when you want to detect drift.

---

### Q6 — Prometheus | Conceptual | Intermediate

> What are the **4 core metric types** in Prometheus? Explain each with a real-world example of when you would use it.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`

#### Key Points to Cover:
```
1. Counter:
   → Monotonically increasing value (only goes up, never down)
   → Resets to 0 on process restart
   → Use: counting things that happen
   → Examples: http_requests_total, errors_total, bytes_sent_total
   → Query: always use rate() or increase() (never raw value)
   rate(http_requests_total[5m])  # requests per second

2. Gauge:
   → Can go up AND down
   → Represents current state/value at a point in time
   → Examples: memory_usage_bytes, active_connections, cpu_temperature
   → Query: use directly or with avg_over_time()
   container_memory_usage_bytes  # current memory

3. Histogram:
   → Samples observations into configurable buckets
   → Tracks: count of observations, sum, and bucket counts
   → Use: latency, request size, response size
   → Enables: percentile calculations
   → Examples: http_request_duration_seconds (buckets: 0.1s, 0.5s, 1s, 5s)
   histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

4. Summary:
   → Like histogram but calculates quantiles on client side
   → Pre-calculated percentiles (cannot aggregate across instances)
   → Use: when you know exactly which quantiles you need
   → Disadvantage: cannot aggregate across multiple pods
   → Histogram is almost always preferred over Summary

Key difference Histogram vs Summary:
  Histogram: server-side quantile calculation, CAN aggregate
  Summary:   client-side, CANNOT aggregate (don't use for K8s)
```

> 💡 **Interview tip:** The **Counter vs Gauge** distinction trips up many candidates. A common mistake: using a Gauge for request counts. Always use Counter for things that accumulate (requests, errors, bytes). The `rate()` function only works correctly on Counters. Also mention that **Histogram is almost always preferred over Summary** for Kubernetes workloads because with multiple pods, you need to aggregate metrics across instances — Summary calculates percentiles per-instance (client-side) which cannot be aggregated correctly.

---

### Q7 — AWS | Scenario-Based | Intermediate

> Your application is running on an EC2 instance and it needs to read files from an S3 bucket. A junior engineer suggests hardcoding AWS Access Keys inside the application code to authenticate. **Why is this a bad idea, and what is the correct AWS-native way to handle this?**

📁 **Reference:** `nawab312/AWS` — IAM, IAM Roles, EC2 Instance Profiles sections

#### Key Points to Cover:
```
Why hardcoding is dangerous:
  1. Credentials in code → committed to Git → exposed publicly
  2. Credentials don't expire → stolen keys work forever
  3. Shared across all environments (dev key = prod key)
  4. No audit trail (all API calls look the same)
  5. Rotation requires code change + redeploy
  6. violates AWS security best practices (will fail audits)

Correct solution: IAM Roles + Instance Profile
  Step 1: Create IAM Role with S3 permission
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-bucket/*"]
    }

  Step 2: Attach role to EC2 as Instance Profile
    aws ec2 associate-iam-instance-profile \
      --instance-id i-1234567890 \
      --iam-instance-profile Name=my-role

  Step 3: Application uses role automatically (no keys needed)
    boto3.client('s3')  # SDK automatically gets temp credentials
                        # from EC2 metadata service (169.254.169.254)

How it works internally:
  → EC2 metadata service: http://169.254.169.254/latest/meta-data/iam/security-credentials/
  → Returns temporary credentials (rotated every ~1 hour automatically)
  → AWS SDK reads these automatically (no code change needed)

For EKS: use IRSA (IAM Roles for Service Accounts)
  → Each Kubernetes pod gets its own IAM role
  → Zero static credentials anywhere
```

> 💡 **Interview tip:** This is one of the most commonly asked AWS security questions. The answer structure that impresses: (1) explain WHY it's dangerous with specific consequences, (2) explain the correct solution (IAM role + instance profile), (3) mention HOW it works (metadata service provides temp creds), (4) extend to containers (IRSA for EKS). Mention that AWS SDKs have a **credential provider chain** — they automatically check environment variables → instance profile → container credentials in order, so the application code needs zero changes.

---

### Q8 — Linux / Bash | Scenario-Based | Intermediate

> You are on-call and get an alert that a production server is consuming unexpectedly **high CPU**. Walk me through how you would **identify the process** and **investigate the root cause** using command line tools.

📁 **Reference:** `nawab312/DSA` → `Linux` — process management and system monitoring sections

#### Key Points to Cover:
```
Step 1 — Quick overview (first 30 seconds):
  top             # interactive, see top CPU consumers
  uptime          # check load average (1m, 5m, 15m)
                  # load avg > CPU count = overloaded

Step 2 — Identify the process:
  top -b -n1 | head -20        # snapshot
  ps aux --sort=-%cpu | head -10  # top CPU processes
  pidstat -u 1 5               # per-process CPU every 1s for 5s

Step 3 — Identify what it's doing:
  strace -p <PID> -c           # syscall summary (what system calls?)
  strace -p <PID>              # live syscall trace
  lsof -p <PID>                # open files/sockets
  cat /proc/<PID>/cmdline      # exact command with arguments
  cat /proc/<PID>/status       # memory, threads, state

Step 4 — Thread level (for multi-threaded apps):
  top -H -p <PID>              # show threads of specific process
  ps -T -p <PID>               # list threads

Step 5 — CPU profiling (if still unclear):
  perf top -p <PID>            # which function consuming CPU?
  perf record -p <PID> -g sleep 30
  perf report                  # flame graph data

Common root causes:
  → Infinite loop in application code
  → CPU-bound computation (compression, encryption, ML)
  → Excessive garbage collection (Java/Python)
  → Fork bomb (rapid process spawning)
  → Runaway cron job
```

> 💡 **Interview tip:** Show the **investigation hierarchy**: `top` → `ps` → `strace` → `perf`. Most candidates stop at `top` (identifying the process). Senior engineers go further: `strace -c` to see what syscalls it's making (is it I/O? locking? computation?), then `perf top` to pinpoint the exact function. Also mention checking `uptime` load average first — a load average higher than your CPU count tells you immediately that CPUs are saturated. Load 45 on 8 cores = serious problem.

---

### Q9 — ArgoCD | Conceptual | Intermediate

> What is the difference between ArgoCD's **Sync** and **Refresh** operations? Also explain what **App of Apps** pattern is and why it is used.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — Sync, Refresh, and App of Apps sections

#### Key Points to Cover:
```
Refresh:
  → Re-reads manifests from Git repository
  → Updates ArgoCD's cached view of desired state
  → Does NOT apply anything to the cluster
  → Like: "check what's in Git now"
  → Triggers: every 3 minutes (default) OR webhook push

Sync:
  → Applies Git state to the cluster (kubectl apply)
  → Makes cluster match what's in Git
  → Requires: refresh must happen first (to know what to apply)
  → Can be: manual or automatic (auto-sync policy)
  → Like: "make the cluster match Git"

Hard Refresh:
  → Clears the manifest cache
  → Forces re-render from source (Helm/Kustomize templates)
  → Use when: template rendering changed but Git content didn't

App of Apps pattern:
  Problem: 50 microservices = 50 ArgoCD Application CRDs to manage
  Solution: One "root" Application manages all other Applications

  Root App → Git repo containing Application CRDs
           → ArgoCD syncs root app
           → Root app creates child Application resources
           → ArgoCD syncs each child application

  Structure:
  gitops-repo/
    apps/
      root-app.yaml          ← manually created once
      payment-service.yaml   ← Application CRD for payment
      auth-service.yaml      ← Application CRD for auth
      api-gateway.yaml       ← Application CRD for api-gw

  Benefits:
    → Declarative management of all applications in Git
    → Add new app = add YAML file to apps/ directory
    → Bootstrap entire cluster from one ArgoCD Application
```

> 💡 **Interview tip:** The **App of Apps** question tests architectural knowledge, not just ArgoCD mechanics. The key insight: it solves the "who manages the managers?" problem. When you have 50 services, creating 50 ArgoCD Applications manually is itself manual toil. App of Apps makes the Application definitions themselves GitOps-managed — your Git repo is the source of truth for what ArgoCD manages, not just what ArgoCD deploys.

---

### Q10 — Jenkins / CI-CD | Troubleshooting | Intermediate

> Your Jenkins pipeline was working fine yesterday, but today it is failing at the build stage with the error: `ERROR: Could not resolve dependencies for project: Connection timed out`. **How would you troubleshoot this, and what are the possible root causes?**

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — pipeline troubleshooting and build failures sections

#### Key Points to Cover:
```
Root causes (check in this order):

1. Network connectivity from Jenkins agent:
   curl -v https://repo1.maven.org/maven2/  # test Maven Central
   curl -v https://registry.npmjs.org/      # test npm
   → Firewall rule changed? Egress blocked?

2. Proxy configuration:
   → Corporate proxy required but not configured
   → Check: JAVA_OPTS, MAVEN_OPTS, npm config
   → Add: -Dhttps.proxyHost=proxy.company.com -Dhttps.proxyPort=8080

3. Nexus/Artifactory down (if using internal mirror):
   → Your internal artifact repository is unavailable
   → Check: http://nexus:8081/
   → Fallback: temporarily point to public Maven Central

4. DNS resolution failure:
   nslookup repo1.maven.org                 # from Jenkins agent
   → DNS server changed or unreachable?

5. Dependency repository changed:
   → External library removed from Maven Central (rare)
   → Version no longer available
   → Check: exact dependency + version in pom.xml/build.gradle

6. SSL/TLS certificate issue:
   → Certificate expired on artifact repository
   → Jenkins JVM doesn't trust new cert

Troubleshooting commands:
  # On Jenkins agent:
  curl --max-time 10 -v https://repo1.maven.org/maven2/
  ping repo1.maven.org
  nslookup repo1.maven.org
  traceroute repo1.maven.org
  
  # Check proxy settings:
  env | grep -i proxy
```

> 💡 **Interview tip:** "Connection timed out" almost always means **network-level issue** (firewall, routing) while "Connection refused" means the server is up but the port is blocked. Start with: can the Jenkins agent reach the internet at all? (`curl google.com`). Then narrow down to the specific artifact repository. Mention that in production environments, **Nexus/Artifactory as a proxy** is the right long-term solution — it caches dependencies internally so your builds don't depend on external internet connectivity.

---

### Q11 — Git | Conceptual | Intermediate

> What is the difference between **`git merge`** and **`git rebase`**? When would you use one over the other in a team environment?

📁 **Reference:** `nawab312/CI_CD` → `Git` — merge vs rebase section

#### Key Points to Cover:
```
git merge:
  → Creates a NEW merge commit combining two branches
  → Preserves complete history (when branches diverged)
  → Non-destructive (existing commits unchanged)
  → Results in: non-linear history (graph with branches)

  main: A - B - C
  feat:         D - E
  after merge:  A - B - C - M  (M has two parents: C and E)
                         \   /
                          D-E

git rebase:
  → Re-applies commits on TOP of another branch
  → Rewrites commit history (new commit hashes)
  → Results in: linear history (straight line)
  → NEVER rebase shared/public branches

  main: A - B - C
  feat:         D - E
  after rebase: A - B - C - D' - E'  (D' and E' are new commits)

When to use which:
  Use merge for:
    → Integrating completed features to main
    → Public/shared branches
    → When you want to preserve exact history
    → Pull requests (git merge --no-ff for visibility)

  Use rebase for:
    → Updating your local feature branch with latest main
    → Before creating a PR (clean linear history)
    → Local commits not yet pushed
    → git pull --rebase (cleaner than merge commits)

Golden rule: Never rebase commits that have been pushed to a shared branch
```

> 💡 **Interview tip:** The question teams always debate is "merge or rebase?" The senior answer: **use both strategically**. Rebase your local feature branch onto main before opening a PR (keeps history clean). Use merge (or squash merge) when merging the PR into main (creates clear feature boundaries). The rule: "rebase before you share, merge after you share." Also mention that `git merge --no-ff` (no fast-forward) forces a merge commit even when it's not needed, making feature integrations visible in history.

---

### Q12 — ELK Stack | Conceptual | Intermediate

> Can you explain the **role of each component** in the ELK Stack — Elasticsearch, Logstash, and Kibana? Also, what is **Beats** and where does it fit in the pipeline?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack`

#### Key Points to Cover:
```
Complete pipeline:
  Application → Beats → Logstash → Elasticsearch → Kibana

Elasticsearch:
  → Distributed search and analytics engine
  → Stores and indexes logs
  → Provides: full-text search, aggregations, real-time indexing
  → Based on: Apache Lucene
  → Data model: indices → documents (JSON)
  → Cluster: multiple nodes for HA and scale

Logstash:
  → Data processing pipeline (ETL for logs)
  → Input plugins: Beats, Kafka, S3, Syslog, JDBC
  → Filter plugins: grok (parse), mutate (transform), geoip
  → Output plugins: Elasticsearch, S3, Kafka, stdout
  → Heavy: JVM-based, resource intensive
  → Use when: complex transformation needed

Kibana:
  → Web UI for Elasticsearch
  → Dashboards, visualizations, Discover (log search)
  → Alerting (Kibana Alerting or Watcher)
  → Dev Tools (run Elasticsearch queries)
  → Machine learning (anomaly detection)

Beats (lightweight shippers):
  → Small Go agents deployed on every server
  → Filebeat: log files (most common)
  → Metricbeat: system metrics
  → Packetbeat: network data
  → Heartbeat: uptime monitoring
  → Much lighter than Logstash (no JVM)

Modern architecture (Filebeat → Elasticsearch, skip Logstash):
  Application → Filebeat → Elasticsearch → Kibana
  → Filebeat has basic processing (modules for nginx, apache, etc.)
  → Use Logstash only when complex processing is needed
```

> 💡 **Interview tip:** Many teams today skip Logstash entirely and use **Filebeat → Elasticsearch** directly with Elasticsearch Ingest Pipelines for basic processing. Logstash is reserved for complex transformations. Know both patterns. Also mention **OpenSearch** (AWS fork of Elasticsearch) — if interviewing at an AWS-heavy company, they may use OpenSearch instead of Elasticsearch. The concepts are identical, just different branding.

---

### Q13 — GitHub Actions | Scenario-Based | Intermediate

> You have a GitHub Actions workflow that builds and deploys your application to AWS. You want to make sure the deployment to production only happens when a PR is merged into `main`, and it should **never trigger on a push to a feature branch**. How would you structure your workflow file to achieve this?

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — workflow triggers and event filters sections

#### Key Points to Cover:
```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main          # ONLY trigger on push to main
                      # NOT on feature branches
  pull_request:
    branches:
      - main          # Trigger on PRs targeting main (for tests only)

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: npm test
    # This job runs on BOTH push to main AND pull_request

  deploy:
    runs-on: ubuntu-latest
    needs: test
    # Only deploy when merged to main (not on PRs)
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        run: ./deploy.sh

# Key concepts:
# github.event_name: 'push' | 'pull_request' | 'schedule' etc.
# github.ref: 'refs/heads/main' | 'refs/heads/feature/xyz'
# github.base_ref: target branch for PRs

# Alternative: use environment protection rules
# Set 'production' environment to require manual approval
# deploy job: environment: production
```

> 💡 **Interview tip:** There are two layers of protection here: (1) the `on: push: branches: [main]` trigger ensures the workflow only runs for main branch pushes, and (2) the `if: github.event_name == 'push'` condition on the deploy job prevents deployment when the workflow runs for pull_request events. **Both are needed** — the trigger controls when the workflow starts, the condition controls which jobs run. Also mention using **GitHub Environments** with required reviewers for an additional approval gate before production deployments.

---

### Q14 — Grafana | Conceptual | Intermediate

> What is the difference between a **Grafana Dashboard** and a **Grafana Alert**? Also explain what a **Data Source** is in Grafana and name at least **4 data sources** Grafana supports.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana`

#### Key Points to Cover:
```
Dashboard:
  → Visual display of metrics/logs over time
  → Panels: graphs, tables, gauges, stat, heatmaps
  → Passive: shows current state, requires human to look at it
  → Use: situational awareness, investigation, capacity planning
  → Time range selector, variables for dynamic filtering

Alert:
  → Automated condition check that sends notifications
  → Active: fires when threshold breached (no human needed)
  → Alert Rule: query + condition + evaluation interval
  → Contact Points: where to send (Slack, PagerDuty, email)
  → Notification Policies: routing rules (which alerts → which contacts)
  → Use: on-call notifications, automated incident detection

Data Source:
  → Connection to a data backend
  → Grafana queries the data source and visualizes results
  → Each panel can use different data source

Supported data sources (name 6+):
  1. Prometheus    — metrics (most common for K8s)
  2. Loki          — logs (Grafana's own log aggregation)
  3. Elasticsearch — logs and metrics
  4. CloudWatch    — AWS metrics
  5. InfluxDB      — time series metrics
  6. Jaeger/Tempo  — distributed traces
  7. MySQL/PostgreSQL — relational databases
  8. Datadog       — commercial monitoring
  9. Graphite      — metrics
  10. Azure Monitor — Azure metrics
```

> 💡 **Interview tip:** The key distinction interviewers test: **dashboard = reactive** (human must look at it), **alert = proactive** (notifies you). A dashboard without alerts means you only know about problems when you happen to look. In Grafana 9+, alerts are "unified" — alert rules can be defined in dashboards OR independently. Mention that Grafana supports over **150 data sources** via plugins — knowing the major ones (Prometheus, Loki, CloudWatch, Elasticsearch) shows breadth of observability knowledge.

---

### Q15 — Python | Scenario-Based | Intermediate

> You are given a log file where each line looks like: `2024-01-15 10:23:45 ERROR Service unavailable`. Write a **Python script** that reads this log file, counts the occurrences of each log level, and prints a summary.

📁 **Reference:** `nawab312/DSA` → `Python` — file handling sections

#### Key Points to Cover:
```python
#!/usr/bin/env python3
from collections import defaultdict
import sys
import re

def parse_log_file(filename):
    """Parse log file and count occurrences of each log level."""
    counts = defaultdict(int)
    total_lines = 0
    unmatched = 0
    
    # Pattern: date time LEVEL message
    pattern = re.compile(
        r'\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2} (\w+) '
    )
    
    try:
        with open(filename, 'r') as f:
            for line in f:
                total_lines += 1
                match = pattern.match(line.strip())
                if match:
                    level = match.group(1).upper()
                    counts[level] += 1
                else:
                    unmatched += 1
    except FileNotFoundError:
        print(f"Error: File '{filename}' not found", file=sys.stderr)
        sys.exit(1)
    
    return counts, total_lines, unmatched

def print_summary(counts, total, unmatched):
    """Print formatted summary."""
    print(f"\n{'='*40}")
    print(f"LOG LEVEL SUMMARY")
    print(f"{'='*40}")
    print(f"Total lines: {total}")
    print(f"Unmatched:   {unmatched}")
    print(f"{'-'*40}")
    
    for level in sorted(counts.keys()):
        pct = (counts[level] / total * 100) if total > 0 else 0
        print(f"{level:<10}: {counts[level]:>6} ({pct:.1f}%)")

if __name__ == "__main__":
    filename = sys.argv[1] if len(sys.argv) > 1 else "app.log"
    counts, total, unmatched = parse_log_file(filename)
    print_summary(counts, total, unmatched)

# Key concepts used:
# defaultdict(int)     → auto-initializes missing keys to 0
# re.compile()         → pre-compile regex for efficiency
# context manager      → with open() for safe file handling
# sys.argv             → command line arguments
# f-strings            → formatted output
# sys.exit(1)          → non-zero exit code on error
```

> 💡 **Interview tip:** Go beyond the basic solution. Show: (1) **regex** for robust parsing instead of split(), (2) **error handling** with try/except and proper exit codes, (3) **defaultdict** instead of checking `if key in dict`, (4) **percentage calculation** for context. The interviewer is testing Python proficiency — these extras demonstrate real-world coding habits (error handling, clean code) vs just making it work.

---

### Q16 — Kubernetes | Scenario-Based | Advanced

> Your production cluster has a **memory leak** in one of your microservices. The Pod keeps consuming more and more memory until it gets OOMKilled and restarts. **How would you handle this both immediately and long-term?**

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

#### Key Points to Cover:
```
Immediate response (stop the bleeding):

1. Confirm OOMKill:
   kubectl describe pod <pod> | grep -i "OOMKilled\|exit code"
   kubectl top pods -n <namespace>

2. Increase memory limit temporarily:
   kubectl set resources deployment/<name> \
     --limits=memory=1Gi --requests=memory=512Mi

3. Add liveness probe restart threshold:
   → If memory exceeds X → liveness fails → pod restarts before crash
   → Reduces user impact by restarting before OOM

4. Scale out (more replicas = more total capacity):
   kubectl scale deployment/<name> --replicas=5
   → Distribute load, each pod handles less traffic

5. Alert on memory growth trend:
   container_memory_usage_bytes > 0.8 * container_spec_memory_limit_bytes
   → Alert before OOMKill, not after

Long-term fixes:

1. Profile the application:
   → Java: heap dump, jmap, async-profiler
   → Python: memory_profiler, tracemalloc
   → Node.js: --inspect, Chrome DevTools heap snapshot
   → Go: pprof, runtime.ReadMemStats

2. Fix the leak in code:
   → Unclosed connections (DB, HTTP)
   → Growing caches without eviction
   → Event listener accumulation
   → Static lists/maps never cleared

3. Set proper resource limits:
   → Based on: kubectl top pods over 1 week (p99 usage)
   → Set limit = 1.5x normal peak usage

4. Implement circuit breaker:
   → Limit concurrent requests per pod
   → Prevent memory spike under load

5. VPA (Vertical Pod Autoscaler) recommendation mode:
   → Suggests correct resource values based on actual usage
```

> 💡 **Interview tip:** Structure your answer as **immediate (stabilize) then long-term (fix)**. The immediate fix is NOT "restart the pod" — that just delays the problem. The immediate fix is **increasing the limit** so it can operate while you investigate AND **alerting on memory growth trend** so you know BEFORE it crashes next time. The long-term fix requires profiling the actual leak. Mention **VPA in recommendation mode** — it observes real memory usage and suggests correct limits without changing anything, which helps right-size resources.

---

### Q17 — Terraform | Troubleshooting | Advanced

> You run `terraform apply` and it fails halfway through — some resources got created, some didn't. The state file is out of sync. How would you **recover without destroying and recreating everything**?

📁 **Reference:** `nawab312/Terraform` — state management, `terraform import`, and `terraform state` commands

#### Key Points to Cover:
```
Step 1 — Assess the situation:
  terraform show                    # what's in state
  terraform plan                    # what Terraform thinks needs doing
  # Compare with what actually exists in AWS console

Step 2 — For resources created but NOT in state:
  terraform import aws_instance.web i-1234567890
  # Import existing resource into state
  # After import: terraform plan should show no changes for it

Step 3 — For resources in state but NOT created (failed):
  terraform apply                   # re-run apply — idempotent
  # Terraform only creates what's missing (doesn't duplicate)
  # OR target specific resource:
  terraform apply -target=aws_db_instance.main

Step 4 — For partially created resources (e.g., RDS with no DB):
  terraform taint aws_db_instance.main  # mark for recreation
  terraform apply                       # recreates the resource

Step 5 — For corrupted state:
  # Restore from S3 versioned backup
  aws s3 cp s3://bucket/terraform.tfstate.backup ./terraform.tfstate
  # Or list S3 versions and restore specific version

Step 6 — Manual state manipulation (advanced):
  terraform state list              # list all resources in state
  terraform state rm aws_instance.old  # remove resource from state
  terraform state mv old_name new_name # rename resource in state

Prevention:
  1. prevent_destroy = true on critical resources
  2. Separate state per environment
  3. S3 versioning on state bucket
  4. Never run apply without plan review first
  5. -target for surgical applies during incidents
```

> 💡 **Interview tip:** The key insight: **Terraform apply is idempotent** — running it again after a partial failure will only create what's missing and skip what already exists. You don't need to start from scratch. The most useful command for recovery is `terraform import` — use it to pull manually-created or partially-created resources into the state so Terraform knows about them. Also mention `-target` for surgical applies that only affect one resource, which limits blast radius during recovery.

---

### Q18 — AWS | Conceptual | Advanced

> Explain the difference between **Security Groups** and **Network ACLs (NACLs)** in AWS. In what order are they evaluated, and what happens if they conflict?

📁 **Reference:** `nawab312/AWS` — VPC Security, Security Groups, and Network ACLs sections

#### Key Points to Cover:
```
Security Groups (SG):
  Level:     Instance level (attached to EC2, RDS, Lambda ENI)
  Stateful:  YES — return traffic automatically allowed
  Rules:     ALLOW only (no deny rules — whitelist approach)
  Source:    IP CIDR or another Security Group
  Evaluation: ALL rules evaluated, any match = allow
  Default:   deny all inbound, allow all outbound

Network ACLs (NACL):
  Level:     Subnet level (all traffic in/out of subnet)
  Stateful:  NO — must explicitly allow both directions
  Rules:     ALLOW and DENY (explicit deny possible)
  Source:    IP CIDR only (not SG reference)
  Evaluation: Rules evaluated in order (lowest number first)
              First match wins — STOP processing
  Default:   allow all (default NACL)

Evaluation order:
  INBOUND traffic to EC2:
  1. NACL inbound rules evaluated first
  2. If NACL allows → Security Group inbound evaluated
  3. Both must allow → traffic reaches EC2

  OUTBOUND response from EC2:
  1. Security Group: STATEFUL → automatically allowed
  2. NACL: STATELESS → outbound rule must exist
     (allow ephemeral ports 1024-65535 for return traffic)

When they conflict:
  → NACL denies traffic → traffic blocked regardless of SG
  → SG allows traffic but NACL denies → blocked
  → NACL allows but SG denies → blocked
  → BOTH must allow for traffic to flow

Use case for NACL:
  → Block specific IP ranges (impossible with SG — no deny)
  → Quick emergency block (affects all instances in subnet)
  → Additional layer of defense in depth
```

> 💡 **Interview tip:** The most commonly forgotten NACL detail: **ephemeral ports**. When a client connects to your server on port 443, the server's response goes back to the client's random ephemeral port (1024-65535). Security Groups handle this automatically (stateful). NACLs don't — you MUST add an outbound rule allowing ports 1024-65535 or your responses will be silently blocked. This is the #1 NACL misconfiguration in real environments.

---

### Q19 — Kubernetes | Scenario-Based | Advanced

> Your production microservice experiences high latency during peak traffic. The HPA is configured but Pods are scaling up too late — by the time new Pods are ready, the spike has already impacted users. **What are the possible reasons and how would you fix it?**

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

#### Key Points to Cover:
```
Root causes and fixes:

1. HPA metrics lag (most common):
   → HPA evaluates every 15s (default)
   → By the time metric is high enough → already overloaded
   Fix:
     kubectl patch hpa my-app -p '{"spec":{"scaleTargetRef":{},"behavior":{"scaleUp":{"stabilizationWindowSeconds":0}}}}'
   → Set stabilizationWindowSeconds: 0 for scale-up (immediate)
   → Reduce sync period: --horizontal-pod-autoscaler-sync-period=10s

2. Slow pod startup (new pods not ready in time):
   → App takes 60s to start → users wait during scale-up
   Fix:
     → Optimize startup time (lazy loading, faster init)
     → Add minReadySeconds to avoid premature load balancing
     → Warm-up via init containers or readiness probe tuning

3. Target utilization too high:
   → targetCPUUtilizationPercentage: 90 → scaling starts too late
   Fix:
     → Lower target to 60-70% → more headroom, earlier scaling
     → Scale up before hitting limits, not when already saturated

4. minReplicas too low:
   → minReplicas: 1 → single pod must handle traffic until scale completes
   Fix:
     → Set minReplicas: 3 (spread across AZs)
     → Pre-scale: scheduled scaling before known peak hours

5. Custom metrics not available fast enough:
   → Prometheus → custom metrics API → HPA
   → 30-60s delay in metric propagation
   Fix:
     → Use predictive scaling (Karpenter/KEDA)
     → Use Kubernetes Event Driven Autoscaler (KEDA) for SQS depth

6. Nodes not available for scheduling (Cluster Autoscaler delay):
   → No nodes available → pods pending → 2-3 min for new node
   Fix:
     → Karpenter (faster than Cluster Autoscaler)
     → Keep spare node capacity (overprovisioner pod)
```

> 💡 **Interview tip:** The best answer addresses **both HPA lag AND pod startup time** — they compound each other. If HPA takes 30s to decide to scale, and pods take 60s to start, users experience 90 seconds of degraded service. The senior SRE fix: (1) lower target utilization (scale earlier), (2) set `stabilizationWindowSeconds: 0` for scale-up, (3) reduce pod startup time (the most impactful improvement), (4) use pre-scaling for known traffic peaks (scheduled scaling). Mention **Karpenter over Cluster Autoscaler** for faster node provisioning.

---

### Q20 — Linux / Bash | Troubleshooting | Advanced

> You are on-call and a production server's root partition is at **95% disk usage**. Walk me through your complete step-by-step process to identify what is consuming the disk and free it up **without causing downtime**.

📁 **Reference:** `nawab312/DSA` → `Linux` — disk management sections

#### Key Points to Cover:
```
Step 1 — Confirm and identify the partition:
  df -h                              # which partition is full?
  df -i                              # check inodes too (can be "full" with 0% used)

Step 2 — Find what's consuming space:
  du -sh /var/log/* | sort -rh | head -20    # logs usually culprit
  du -sh /tmp/* | sort -rh | head -10
  du -sh /home/* | sort -rh | head -10
  du -sh /* 2>/dev/null | sort -rh | head -10  # top-level dirs

Step 3 — Check for deleted files still open:
  lsof | grep deleted | sort -k7 -rn | head -10
  # Process has deleted file open → space not freed
  # Fix: restart that process OR kill it

Step 4 — Quick safe cleanup:
  # Clear old logs:
  find /var/log -name "*.gz" -mtime +7 -delete   # compressed logs > 7 days
  journalctl --vacuum-size=500M                  # limit journald logs
  
  # Clear package cache:
  apt-get clean                  # Ubuntu/Debian
  yum clean all                  # RHEL/CentOS
  
  # Clear temp files:
  find /tmp -mtime +7 -delete    # tmp files older than 7 days
  
  # Docker cleanup (if Docker host):
  docker system prune -f         # remove unused images/containers
  docker volume prune -f

Step 5 — Find and truncate growing log files (safely):
  # DO NOT: rm application.log (breaks app file descriptor)
  # DO: truncate -s 0 /var/log/app/application.log (safe)
  # OR: > /var/log/app/application.log (same effect)

Step 6 — Fix root cause:
  → Configure logrotate for the growing log
  → Fix application writing excessive debug logs
  → Add disk space monitoring + alert at 80%
```

> 💡 **Interview tip:** The **deleted-but-open file** scenario is the classic trap: `df` shows 95% full but `du /` shows only 50% used. The gap is deleted files still held open by processes. `lsof | grep deleted` reveals them. The fix: restart or signal the process to reopen its log files. Also critical: **never `rm` an active log file** — the application keeps writing to the deleted file descriptor, consuming disk space that `df` counts but `ls` doesn't show. Use `> /path/to/log` (truncate) instead.

---

### Q21 — Kubernetes | Conceptual | Advanced

> Explain how **RBAC** works in Kubernetes. What is the difference between a **Role** and a **ClusterRole**? A **RoleBinding** and a **ClusterRoleBinding**? Give a real-world example of when you would use each.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md`

#### Key Points to Cover:
```
4 RBAC objects:
  Role:               permissions scoped to ONE namespace
  ClusterRole:        permissions cluster-wide OR cluster-scoped resources
  RoleBinding:        grants Role/ClusterRole to subject in ONE namespace
  ClusterRoleBinding: grants ClusterRole to subject cluster-wide

Role vs ClusterRole:
  Role:
    → Defined in and applies to one namespace only
    → Cannot grant access to cluster-scoped resources (nodes, PVs)
    → Example: "read pods in production namespace"

  ClusterRole:
    → Applies across all namespaces OR for cluster-scoped resources
    → Can be bound to a namespace (via RoleBinding) or cluster-wide (via ClusterRoleBinding)
    → Example: "read nodes anywhere", "read pods in all namespaces"

RoleBinding vs ClusterRoleBinding:
  RoleBinding:
    → Grants permissions in ONE namespace only
    → Can reference Role OR ClusterRole (limits cluster role to one namespace)
    → Example: grant "developer" ClusterRole only in namespace "team-a"

  ClusterRoleBinding:
    → Grants permissions across ALL namespaces
    → Must reference ClusterRole
    → Example: cluster-admin binding for platform team

Real-world examples:
  Developer access to one namespace:
    → ClusterRole "pod-reader" (reusable definition)
    → RoleBinding in "production" namespace → grants to dev-team group

  Prometheus scraping all pods:
    → ClusterRole with get/list/watch on pods, endpoints, services
    → ClusterRoleBinding → grants to prometheus ServiceAccount

  Platform admin:
    → ClusterRole "admin" (built-in)
    → ClusterRoleBinding → grants to platform-team group

Verify RBAC:
  kubectl auth can-i list pods -n production --as developer@company.com
```

> 💡 **Interview tip:** The **ClusterRole + RoleBinding combination** is the most elegant RBAC pattern and often catches candidates off guard. You define a ClusterRole ONCE (e.g., "developer" with read access to pods, logs, exec), then bind it to different namespaces via RoleBindings. Team A gets the "developer" ClusterRole in namespace "team-a", Team B in "team-b". One ClusterRole definition, many namespace-scoped bindings — DRY principle applied to RBAC.

---

### Q22 — AWS | Scenario-Based | Advanced

> Your multi-tier AWS application has EC2 in a public subnet (frontend), EC2 in a private subnet (backend API), and RDS in a private subnet. The backend API needs to download packages from the internet during deployment but has no direct internet access. **How do you architect this?**

📁 **Reference:** `nawab312/AWS` — VPC, NAT Gateway, Internet Gateway sections

#### Key Points to Cover:
```
Solution: NAT Gateway in public subnet

Architecture:
  VPC (10.0.0.0/16)
  ├── Public Subnet (10.0.1.0/24)  AZ-a
  │   ├── Internet Gateway (IGW) attached to VPC
  │   ├── Frontend EC2                      ← has public IP
  │   └── NAT Gateway                       ← outbound only
  ├── Private Subnet (10.0.2.0/24)  AZ-a
  │   └── Backend API EC2                   ← no public IP
  └── Private Subnet (10.0.3.0/24)  AZ-a
      └── RDS PostgreSQL                    ← no public IP

Routing tables:
  Public subnet:
    0.0.0.0/0 → Internet Gateway
    (direct internet access both directions)

  Private subnet:
    0.0.0.0/0 → NAT Gateway (in public subnet)
    10.0.0.0/16 → local
    (outbound internet via NAT, no inbound from internet)

How NAT Gateway works:
  Backend API → NAT Gateway → IGW → Internet
  → Translates private IP to NAT Gateway's public IP
  → Internet sees: NAT Gateway IP (not backend IP)
  → Return traffic: Internet → NAT → Backend (stateful)
  → Result: backend can download packages, but internet CANNOT initiate connections to backend

Security:
  Frontend SG: allow 80/443 from 0.0.0.0/0
  Backend SG:  allow 8080 from Frontend SG only
  RDS SG:      allow 5432 from Backend SG only
  NAT GW:      no SG (managed service)

Cost: NAT Gateway = $0.045/hr + $0.045/GB processed
Alternative: NAT Instance (EC2) — cheaper but less reliable
```

> 💡 **Interview tip:** The follow-up question is always "what's the difference between NAT Gateway and Internet Gateway?" Remember: **IGW is bidirectional** (inbound + outbound internet), **NAT Gateway is outbound only** (private resources can reach internet, but internet cannot initiate connections to private resources). Also mention HA: deploy **one NAT Gateway per AZ** — if you have one NAT Gateway in AZ-a and the AZ goes down, all private subnets in other AZs lose internet access. One NAT GW per AZ costs more but is production-grade.

---

### Q23 — Prometheus | Conceptual | Advanced

> What is the difference between **Recording Rules** and **Alerting Rules** in Prometheus? When would you use a Recording Rule? Write an example of both.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`

#### Key Points to Cover:
```
Recording Rules:
  → Pre-compute expensive PromQL expressions
  → Store result as NEW metric (faster queries later)
  → Use when: query is expensive, frequently used, or powers dashboards
  → Naming convention: level:metric:operation

  Example:
  - record: job:http_requests:rate5m
    expr: sum by(job)(rate(http_requests_total[5m]))
  # Now dashboards query: job:http_requests:rate5m (fast)
  # Instead of: sum by(job)(rate(http_requests_total[5m])) (slow, computed each time)

Alerting Rules:
  → Define conditions that trigger alerts
  → Evaluated every evaluation_interval (default: 1m)
  → "for" duration: must be true for this long before firing (avoids flapping)
  → Labels: severity, team, service (for routing in Alertmanager)

  Example:
  - alert: HighErrorRate
    expr: rate(http_errors_total[5m]) / rate(http_requests_total[5m]) > 0.05
    for: 5m        # must be >5% for 5 consecutive minutes
    labels:
      severity: critical
    annotations:
      summary: "Error rate {{ $value | humanizePercentage }}"

Combined (best practice — alert on recording rule):
  # Step 1: Recording rule (fast pre-computation)
  - record: job:http_error_rate:rate5m
    expr: rate(http_errors_total[5m]) / rate(http_requests_total[5m])

  # Step 2: Alert on the recording rule (fast evaluation)
  - alert: HighErrorRate
    expr: job:http_error_rate:rate5m > 0.05
    for: 5m

Rule file organization:
  /etc/prometheus/rules/
    recording_rules.yml   ← pre-computations
    alerting_rules.yml    ← alert conditions
```

> 💡 **Interview tip:** The **"for" duration** in alerting rules is critical — without it, a 1-second spike triggers an alert (flapping). With `for: 5m`, the condition must be true for 5 continuous minutes. This eliminates transient spikes. The rule of thumb: use `for: 0m` only for critical binary states (node down), use `for: 5m` or longer for rate-based metrics (error rate, CPU usage). Also emphasize alerting ON recording rules — this is the production best practice since it makes alert evaluation fast and the recording rule computation is reused by both dashboards and alerts.

---

### Q24 — ArgoCD | Troubleshooting | Advanced

> ArgoCD shows an application as `OutOfSync` even after Sync completes successfully. It goes back to OutOfSync within minutes. **What are the possible reasons and how do you investigate and fix it?**

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — Sync Status, ignoreDifferences sections

#### Key Points to Cover:
```
Causes of persistent OutOfSync:

1. Kubernetes mutates resources after ArgoCD applies them:
   → K8s adds default values ArgoCD didn't set (defaultMode, imagePullPolicy)
   → Admission webhooks mutate resources (Istio injects sidecar annotations)
   → Check: argocd app diff my-app | grep "^[<>]"
   Fix: ignoreDifferences for those fields

2. HPA modifying replicas:
   → ArgoCD wants: replicas: 3 (from Git)
   → HPA changes to: replicas: 5
   → ArgoCD sees: difference → marks OutOfSync
   Fix:
     ignoreDifferences:
     - group: apps
       kind: Deployment
       jsonPointers:
       - /spec/replicas

3. Server-side annotation drift:
   → AWS Load Balancer Controller adds annotations
   → ArgoCD doesn't know about them
   Fix: ignoreDifferences for those annotations

4. List ordering differences:
   → Env vars in different order in Git vs live
   → ArgoCD treats as different
   Fix: ignoreDifferences with JQ path expressions

5. ResourceVersion/Generation fields:
   → These K8s internal fields always differ
   → Usually auto-handled but custom resources might expose them

Debugging:
  argocd app diff my-app          # shows exact diff
  argocd app sync my-app --dry-run  # what would sync do?
  kubectl get deployment my-app -o yaml  # live state

Fix template:
  spec:
    ignoreDifferences:
    - group: apps
      kind: Deployment
      jqPathExpressions:
      - .spec.replicas         # managed by HPA
      - .spec.template.metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"]
```

> 💡 **Interview tip:** Persistent OutOfSync almost always means **Kubernetes is mutating resources** after ArgoCD applies them. The diagnostic command: `argocd app diff my-app` — this shows exactly what ArgoCD thinks is different. Usually you'll see ArgoCD wanting to change something that Kubernetes automatically set. The fix is `ignoreDifferences` — not to force-sync — because force-syncing would fight against the legitimate mutation (like HPA replica management). Never set `syncPolicy.automated.selfHeal: true` without first reviewing what's causing the OutOfSync.

---

### Q25 — Terraform | Scenario-Based | Advanced

> Multiple engineers work on the same Terraform codebase. Problems: code duplication across environments, inconsistency, and state corruption from concurrent applies. **How would you restructure the codebase to solve all these problems?**

📁 **Reference:** `nawab312/Terraform` — Modules, Remote State, State Locking sections

#### Key Points to Cover:
```
Problem 1: Code duplication
Solution: Terraform Modules

  modules/
    vpc/         ← define VPC once
    eks/         ← define EKS once
    rds/         ← define RDS once

  environments/
    dev/
      main.tf    ← just calls modules with dev variables
    staging/
      main.tf    ← calls same modules with staging variables
    prod/
      main.tf    ← calls same modules with prod variables

Problem 2: State corruption (concurrent applies)
Solution: Remote backend with locking

  terraform {
    backend "s3" {
      bucket         = "company-terraform-state"
      key            = "environments/prod/terraform.tfstate"
      region         = "us-east-1"
      encrypt        = true
      dynamodb_table = "terraform-state-lock"
    }
  }
  → DynamoDB lock prevents two engineers applying simultaneously
  → S3 versioning enables state rollback

Problem 3: Inconsistency between environments
Solution: Same modules + environment-specific variables

  # environments/prod/terraform.tfvars
  environment   = "prod"
  instance_type = "m5.xlarge"
  min_nodes     = 5
  
  # environments/dev/terraform.tfvars  
  environment   = "dev"
  instance_type = "t3.medium"
  min_nodes     = 1

Advanced: Terragrunt for DRY backend config
  → Root terragrunt.hcl defines S3 backend once
  → Each module dir inherits it automatically
  → Key auto-set to ${path_relative_to_include()}/terraform.tfstate

CI/CD for Terraform:
  → PR: terraform plan runs automatically
  → Merge to main: terraform apply runs automatically
  → Tools: Atlantis, GitHub Actions, Terraform Cloud
```

> 💡 **Interview tip:** The **state locking** explanation impresses most interviewers: without it, Engineer A and Engineer B running `terraform apply` simultaneously will both read the same state, both compute the same plan, and both apply — resulting in duplicate resources or a corrupted state file. DynamoDB provides atomic locking: the first apply grabs the lock, the second waits. This is the same problem databases solve with transactions. Mention that lock timeout (default 20 min) is important — a crashed apply holds the lock until it expires.

---

### Q26 — Kubernetes Networking | Scenario-Based | Advanced

> Frontend pod in namespace `web` cannot reach backend Service in namespace `api`. NetworkPolicies are enabled. **Walk through diagnosis and write the NetworkPolicy to allow the traffic.**

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md`

#### Key Points to Cover:
```
Step 1 — Test connectivity:
  kubectl exec -it frontend-pod -n web -- \
    curl -v http://backend-svc.api.svc.cluster.local:8080
  # DNS format: <service>.<namespace>.svc.cluster.local

Step 2 — Check DNS resolution:
  kubectl exec -it frontend-pod -n web -- \
    nslookup backend-svc.api.svc.cluster.local
  # If DNS fails: CoreDNS issue, not NetworkPolicy

Step 3 — Check existing NetworkPolicies:
  kubectl get networkpolicy -n api
  kubectl get networkpolicy -n web
  kubectl describe networkpolicy <name> -n api

Step 4 — Verify service and endpoints:
  kubectl get svc backend-svc -n api
  kubectl get endpoints backend-svc -n api

NetworkPolicy solution:

# Allow frontend (in 'web' namespace) to reach backend (in 'api' namespace)

# Policy on the BACKEND side (ingress rule in 'api' namespace):
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: api
spec:
  podSelector:
    matchLabels:
      app: backend              # applies to backend pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:        # from namespace 'web'
        matchLabels:
          kubernetes.io/metadata.name: web
      podSelector:              # from pods with label app=frontend
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080

# IMPORTANT: namespace must have label for namespaceSelector to work
kubectl label namespace web kubernetes.io/metadata.name=web
```

> 💡 **Interview tip:** The most common NetworkPolicy mistake: using `namespaceSelector` without labeling the namespace. By default, namespaces don't have the `kubernetes.io/metadata.name` label (Kubernetes 1.22+ adds it automatically — but older clusters don't). Always verify namespace labels with `kubectl get namespace web --show-labels`. Also critical: **`namespaceSelector` and `podSelector` together (AND logic)** vs **listing them separately (OR logic)** — putting them under the same `from` item means both must match (AND). Separate `from` items = OR.

---

### Q27 — Git | Conceptual | Advanced

> Explain the difference between **`git reset`** and **`git revert`**. When would you use one over the other in a shared branch? Explain the 3 modes of `git reset`.

📁 **Reference:** `nawab312/CI_CD` → `Git` — undoing changes sections

#### Key Points to Cover:
```
git reset:
  → Moves the branch pointer backward (rewrites history)
  → 3 modes:

  --soft:  HEAD moves, staging preserved, working dir preserved
    Use: undo last commit but keep changes staged (ready to recommit)
    git reset --soft HEAD~1

  --mixed (default): HEAD moves, staging cleared, working dir preserved
    Use: undo commit, unstage changes (files still modified)
    git reset HEAD~1

  --hard: HEAD moves, staging cleared, working dir WIPED
    Use: completely abandon last N commits (DESTRUCTIVE)
    git reset --hard HEAD~3
    WARNING: uncommitted changes are LOST permanently

git revert:
  → Creates a NEW commit that undoes changes
  → History preserved (old commit still there)
  → Safe for shared branches
  git revert abc1234

When to use each:
  reset:
    ✅ Local commits not yet pushed
    ✅ Cleaning up your own work before sharing
    ✅ Squashing multiple local commits into one
    ❌ NEVER on shared/pushed branches (breaks others' history)

  revert:
    ✅ Shared branches (main, develop)
    ✅ After pushing to remote
    ✅ When you need audit trail ("this was intentionally undone")
    ✅ Undoing a specific commit in the middle of history

Real scenario:
  Bad commit pushed to main:
    → git revert <bad-commit-hash>
    → git push origin main
    → Others pull normally — no history conflict

  Bad local commit (not pushed):
    → git reset --soft HEAD~1
    → Fix the issue
    → git commit (clean commit)
```

> 💡 **Interview tip:** The golden rule: **`reset` for local, `revert` for shared**. A great way to explain reset: think of the 3 modes as "how far back do you rewind?". `--soft` rewinds only the commit pointer. `--mixed` rewinds the pointer AND empties the staging area. `--hard` rewinds everything including your working files. Also mention `git reflog` as the safety net — even after `--hard`, commits are recoverable for 90 days via reflog as long as you haven't garbage-collected.

---

### Q28 — ELK Stack | Troubleshooting | Advanced

> Application logs are not appearing in Kibana. The application is running fine and generating logs. **Walk through your complete troubleshooting process** from application to Kibana.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack`

#### Key Points to Cover:
```
Pipeline: Application → Filebeat → Logstash → Elasticsearch → Kibana

Step 1 — Check application is writing logs:
  tail -f /var/log/app/application.log
  # Confirm logs are actually being written

Step 2 — Check Filebeat:
  systemctl status filebeat
  journalctl -u filebeat -f          # real-time logs
  filebeat test config               # validate config syntax
  filebeat test output               # test Logstash/ES connection

  # Check Filebeat registry (tracks position in files):
  cat /var/lib/filebeat/registry/filebeat/log.json
  # If position = end of file → Filebeat processed everything already
  # If stale: reset registry position or use 'close_eof: false'

Step 3 — Check Logstash:
  systemctl status logstash
  tail -f /var/log/logstash/logstash-plain.log
  # Look for: pipeline errors, grok failures, ES connection errors

  curl http://logstash:9600/_node/stats  # pipeline stats
  # Check: events.in vs events.out vs events.filtered

Step 4 — Check Elasticsearch:
  curl http://elasticsearch:9200/_cluster/health
  # GREEN: all good, YELLOW: some replicas unassigned, RED: data loss

  curl http://elasticsearch:9200/app-logs-*/_count
  # Are documents actually in ES?

  curl http://elasticsearch:9200/_cat/indices?v
  # Is the index being created?

Step 5 — Check Kibana:
  # Is the index pattern correct? (app-logs-* not app-log-*)
  # Is the time filter too narrow? (set to last 7 days)
  # Are documents in ES? → problem is Kibana config
  # No documents in ES? → problem is ingestion pipeline

Common root causes:
  → Filebeat can't reach Logstash (firewall, wrong port)
  → Grok parse failure → _grokparsefailure tag → routed to error index
  → Index pattern mismatch in Kibana
  → Time filter too narrow (correct data, wrong time window)
```

> 💡 **Interview tip:** Always **start from the source and work downstream** — Application → Filebeat → Logstash → Elasticsearch → Kibana. Each step is a potential failure point. The fastest diagnostic: `curl elasticsearch:9200/app-logs-*/_count` — if this returns 0, the problem is before Elasticsearch. If it returns documents, the problem is Kibana configuration (usually wrong index pattern or time filter). This binary search approach halves the problem space immediately.

---

### Q29 — AWS | Scenario-Based | Advanced

> Your entire application went down because it was deployed in a single AZ and that AZ had an outage. **How would you redesign the architecture to be highly available across multiple AZs?**

📁 **Reference:** `nawab312/AWS` — High Availability, Multi-AZ sections

#### Key Points to Cover:
```
Multi-AZ architecture (3 AZs minimum):

  Region: us-east-1
  ┌─────────────────────────────────────────────────────┐
  │                      VPC                            │
  │                                                     │
  │  AZ-a (us-east-1a)  AZ-b (us-east-1b)  AZ-c       │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │
  │  │ Public       │  │ Public       │  │ Public   │  │
  │  │ NAT GW       │  │ NAT GW       │  │ NAT GW   │  │
  │  └──────────────┘  └──────────────┘  └──────────┘  │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │
  │  │ Private      │  │ Private      │  │ Private  │  │
  │  │ EC2/ECS/EKS  │  │ EC2/ECS/EKS  │  │ EC2      │  │
  │  └──────────────┘  └──────────────┘  └──────────┘  │
  │  ┌──────────────┐  ┌──────────────┐                 │
  │  │ DB Private   │  │ DB Private   │                 │
  │  │ RDS Primary  │  │ RDS Standby  │                 │
  │  └──────────────┘  └──────────────┘                 │
  │                                                     │
  │  ALB: spans all 3 AZs                               │
  └─────────────────────────────────────────────────────┘

Key components for HA:

1. ALB across all AZs:
   → Single DNS entry → routes to healthy AZs
   → If AZ fails → ALB stops routing to that AZ automatically

2. ASG with multi-AZ:
   → min: 3, max: 9 (one per AZ)
   → AZRebalancing: automatically replaces instances in failed AZ

3. RDS Multi-AZ:
   → Primary in AZ-a, standby in AZ-b
   → Automatic failover: 60-120 seconds
   → OR: Aurora (faster failover: 30 seconds)

4. ElastiCache Multi-AZ (if using):
   → Redis with replication group across AZs

5. Stateless application tier:
   → No session state on EC2 instances
   → Sessions in ElastiCache (any node can serve any user)

6. NAT Gateway per AZ:
   → One NAT GW per AZ (avoid single NAT GW = single point of failure)

Testing: Use AWS Fault Injection Simulator to test AZ failure
```

> 💡 **Interview tip:** The key principle: **every stateful component needs AZ redundancy separately**. ALB, ASG, RDS, ElastiCache, EFS — each needs multi-AZ config independently. Missing one creates a hidden single point of failure. Also mention: **stateless application tier is prerequisite** — if your EC2 instances store session state locally, you can't distribute across AZs (user's next request might go to different AZ). Always mention ElastiCache for session storage as part of the HA redesign.

---

### Q30 — Kubernetes | Conceptual | Advanced

> Explain how **Kubernetes Ingress** works end-to-end. What is the difference between an Ingress Resource and an Ingress Controller? Write an Ingress manifest that routes `/api` to a backend service and `/` to frontend, with TLS.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md`

#### Key Points to Cover:
```
Two distinct components:

Ingress Resource:
  → Kubernetes object (YAML spec)
  → Defines routing rules (host/path → service)
  → Does nothing by itself without a controller

Ingress Controller:
  → Pod(s) running in cluster (nginx, traefik, AWS ALB)
  → Watches Ingress resources via Kubernetes API
  → Configures actual load balancer based on rules
  → Handles: TLS termination, routing, rate limiting

Request flow:
  User → DNS → ALB/LoadBalancer → Nginx Ingress Controller Pod
       → reads Ingress rules → routes to Service → Pod

Complete Ingress manifest:
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx       # which Ingress Controller handles this
  tls:
  - hosts:
    - api.company.com
    secretName: api-tls-secret  # Secret with tls.crt and tls.key
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80

TLS secret creation:
  kubectl create secret tls api-tls-secret \
    --cert=tls.crt --key=tls.key -n production
  
  # OR use cert-manager for automatic Let's Encrypt:
  metadata:
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
  spec:
    tls:
    - hosts: [api.company.com]
      secretName: api-tls-secret  # cert-manager creates this automatically
```

> 💡 **Interview tip:** Candidates often confuse Ingress Resource with Ingress Controller. The analogy: **Ingress Resource = traffic rules written on paper. Ingress Controller = the actual traffic cop that reads and enforces those rules**. Without a controller, the Ingress resource does absolutely nothing — traffic still goes nowhere. On AWS EKS, you have a choice: Nginx Ingress Controller (software in cluster) vs AWS Load Balancer Controller (creates actual ALB from Ingress resources). AWS LBC is preferred for EKS because it integrates natively with ALB features (WAF, ACM certificates, target groups).

---

### Q31 — GitHub Actions | Scenario-Based | Advanced

> Write a complete GitHub Actions workflow that: runs tests on every PR, builds and pushes Docker image to ECR on merge to main, deploys to Kubernetes, and uses OIDC authentication (no stored AWS credentials).

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — OIDC, ECR, kubectl deploy sections

#### Key Points to Cover:
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REGISTRY: 123456789012.dkr.ecr.us-east-1.amazonaws.com
  IMAGE_NAME: payment-service
  EKS_CLUSTER: production-cluster

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Run tests
      run: npm ci && npm test

  build-push:
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      id-token: write    # Required for OIDC
      contents: read
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials via OIDC
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
        aws-region: ${{ env.AWS_REGION }}
        # NO access keys stored! OIDC exchanges GitHub token for AWS creds

    - name: Login to ECR
      uses: aws-actions/amazon-ecr-login@v2

    - name: Docker metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.ECR_REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,prefix=sha-
          type=raw,value=latest

    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy:
    runs-on: ubuntu-latest
    needs: build-push
    environment: production    # requires manual approval if configured
    permissions:
      id-token: write
      contents: read
    steps:
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
        aws-region: ${{ env.AWS_REGION }}

    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig --name ${{ env.EKS_CLUSTER }} --region ${{ env.AWS_REGION }}

    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/${{ env.IMAGE_NAME }} \
          ${{ env.IMAGE_NAME }}=${{ env.ECR_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-push.outputs.image-tag }}
        kubectl rollout status deployment/${{ env.IMAGE_NAME }} --timeout=5m
```

> 💡 **Interview tip:** The **OIDC authentication** is the key security improvement over storing AWS credentials as GitHub secrets. With OIDC: GitHub generates a short-lived token, GitHub Actions exchanges it with AWS STS via AssumeRoleWithWebIdentity, AWS returns temporary 1-hour credentials. No long-lived secrets stored anywhere. The AWS IAM role's trust policy specifies which GitHub repo/branch can assume it — so `feature/*` branches cannot assume the production role even if they try. This is the modern, production-grade approach.

---

### Q32 — Terraform | Conceptual | Advanced

> What is the difference between **`terraform taint`** and **`terraform import`**? Also explain **`terraform refresh`** and when you'd use it. Give a real-world scenario for each.

📁 **Reference:** `nawab312/Terraform` — state management, taint, import, refresh

#### Key Points to Cover:
```
terraform taint (deprecated in TF 1.0+, replaced by -replace):
  Purpose: Mark a resource for recreation on next apply
  When: Resource is degraded but Terraform doesn't know it
  
  Old: terraform taint aws_instance.web
  New: terraform apply -replace=aws_instance.web
  
  Scenario: EC2 instance has corrupted filesystem. Terraform thinks it's
  healthy (state says "running"). Mark for replacement:
  terraform apply -replace=aws_instance.web
  → Terraform destroys old instance, creates new one

terraform import:
  Purpose: Bring existing AWS resources under Terraform management
  When: Resource was created manually (console/CLI) but not in state
  
  terraform import aws_s3_bucket.logs my-logs-bucket
  → Reads the real resource from AWS
  → Adds it to terraform.tfstate
  → Now Terraform manages it (plan shows no changes if TF config matches)
  
  Scenario: 200 EC2 instances created manually before Terraform adoption.
  Write the TF resource config, then import each one. Now IaC manages them.
  
  Note: import only updates state, does NOT generate .tf config.
  Must write the .tf resource block manually FIRST.
  TF 1.5+: import blocks in config (generate TF code automatically)

terraform refresh:
  Purpose: Update state file to match real AWS resources
  When: Resources changed outside Terraform (manual console edits)
  
  terraform refresh
  → Queries every resource in state from AWS
  → Updates state with current real values
  → Does NOT apply any changes
  
  Scenario: Someone manually changed an EC2 security group in console.
  refresh updates state to reflect the change.
  Next plan shows the drift.
  
  Note: terraform plan runs refresh automatically by default.
  Use -refresh=false for faster plans when drift not expected.
```

> 💡 **Interview tip:** **`terraform import` is not magic** — it only imports the state, it does NOT write the `.tf` configuration file. You must write the resource block manually FIRST (matching the real resource), THEN run import. If your .tf doesn't match reality, `terraform plan` will show changes even after import. Terraform 1.5+ introduced **import blocks** in config files, and `terraform plan` can now generate the configuration automatically — mention this improvement as it shows up-to-date knowledge.

---

### Q33 — Kubernetes | Troubleshooting | Advanced

> A rolling update Deployment rollout is stuck — new Pods are in `Pending` state and old Pods are not terminating. `kubectl rollout status` shows "waiting for 2 of 5 new replicas". **What are the causes and how do you fix it?**

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`, `14_TROUBLESHOOTING.md`

#### Key Points to Cover:
```
Step 1 — Check pending pod reason:
  kubectl describe pod <new-pod> | grep -A20 "Events:"
  # Common events:
  # "Insufficient memory"  → resource request too high
  # "No nodes available"   → all nodes full or tainted
  # "Unschedulable"        → PodAntiAffinity conflict

Step 2 — Check rolling update strategy:
  kubectl describe deployment my-app | grep -A5 "Strategy"
  # maxUnavailable: 0, maxSurge: 1
  # → Never removes old pods until new ones are ready
  # → If new pods can't start → rollout stuck forever

Root causes:

1. Insufficient cluster resources:
   kubectl describe nodes | grep -A5 "Allocated resources"
   Fix: Add nodes (Cluster Autoscaler/Karpenter) or
        reduce resource requests in deployment

2. Image pull failure:
   kubectl describe pod <new-pod> | grep "Failed to pull\|ImagePullBackOff"
   Fix: Correct image name/tag, fix ECR permissions

3. New ConfigMap/Secret doesn't exist:
   kubectl get events -n production | grep "secret\|configmap"
   Fix: Create the referenced ConfigMap/Secret first

4. Readiness probe failing (pod Running but not Ready):
   kubectl describe pod <new-pod> | grep -A10 "Readiness"
   Fix: Fix application startup issue or adjust probe
        initialDelaySeconds, periodSeconds

5. PodAntiAffinity prevents scheduling:
   → All nodes already have one pod of this deployment
   → antiAffinity says only one per node
   → Not enough nodes for all pods
   Fix: Add nodes or loosen affinity rules

Recovery:
  kubectl rollout undo deployment/my-app  # rollback immediately
  kubectl rollout status deployment/my-app
  kubectl rollout history deployment/my-app
```

> 💡 **Interview tip:** `maxUnavailable: 0` combined with a stuck new pod is the classic rollout deadlock: "I won't remove old pods until new pods are Ready, but new pods can't become Ready." Always check BOTH the pending pod events AND the rollout strategy. The immediate recovery is `kubectl rollout undo` — don't spend 20 minutes debugging while prod is degraded. Undo first, investigate second. Also mention: running `kubectl rollout status --timeout=5m` in CI/CD and failing the pipeline if it times out — this prevents silent stuck rollouts.

---

### Q34 — Jenkins | Scenario-Based | Advanced

> Design a complete Jenkins pipeline (Jenkinsfile) for a microservice that: runs on Kubernetes pod agents, has 4 stages (Test → Build → Push to ECR → Deploy to K8s), uses different containers per stage, reads secrets from Jenkins Credentials, and notifies Slack on failure.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — Kubernetes agents, multi-container sections

#### Key Points to Cover:
```groovy
pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: node
            image: node:20-alpine
            command: [cat]
            tty: true
          - name: docker
            image: docker:24-dind
            securityContext:
              privileged: true   # Docker-in-Docker needs this
          - name: kubectl
            image: bitnami/kubectl:latest
            command: [cat]
            tty: true
      '''
    }
  }

  environment {
    ECR_REGISTRY = credentials('ecr-registry-url')
    AWS_REGION   = 'us-east-1'
    IMAGE_TAG    = "${env.GIT_COMMIT[0..7]}"
  }

  stages {
    stage('Test') {
      steps {
        container('node') {
          sh 'npm ci && npm test'
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        container('docker') {
          sh "docker build -t ${ECR_REGISTRY}/payment-service:${IMAGE_TAG} ."
        }
      }
    }

    stage('Push to ECR') {
      steps {
        container('docker') {
          withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'aws-credentials'
          ]]) {
            sh '''
              aws ecr get-login-password --region ${AWS_REGION} | \
                docker login --username AWS --password-stdin ${ECR_REGISTRY}
              docker push ${ECR_REGISTRY}/payment-service:${IMAGE_TAG}
            '''
          }
        }
      }
    }

    stage('Deploy to K8s') {
      steps {
        container('kubectl') {
          withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
            sh '''
              kubectl set image deployment/payment-service \
                payment-service=${ECR_REGISTRY}/payment-service:${IMAGE_TAG}
              kubectl rollout status deployment/payment-service --timeout=5m
            '''
          }
        }
      }
    }
  }

  post {
    failure {
      slackSend(
        channel: '#alerts',
        color: 'danger',
        message: "Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n${env.BUILD_URL}"
      )
    }
    success {
      slackSend(channel: '#deployments', color: 'good',
        message: "Deployed ${env.IMAGE_TAG} to production")
    }
  }
}
```

> 💡 **Interview tip:** The **multi-container pod agent** pattern is what distinguishes modern Jenkins from legacy. Each container provides a specific tool (node for tests, docker for builds, kubectl for deploys) without installing everything in one bloated image. The `container('name')` step switches which container executes the commands. Key detail: all containers share the **same workspace** (volume mount) so files created in the `node` container are accessible in the `docker` container. This is because Kubernetes pod containers share the same pod storage.

---

### Q35 — AWS | Conceptual | Advanced

> Explain the difference between **SQS** and **SNS**. When would you use one over the other? Explain the **Fan-out pattern** and give a real DevOps/SRE use case.

📁 **Reference:** `nawab312/AWS` — SQS, SNS, and messaging patterns sections

#### Key Points to Cover:
```
SQS (Simple Queue Service):
  → Queue: messages stored until a consumer reads and deletes them
  → Pull-based: consumers poll for messages
  → One consumer processes each message (one-to-one)
  → Messages persist until deleted (up to 14 days)
  → Decouples producer from consumer (async processing)
  → Use: task queues, job processing, rate limiting downstream

SNS (Simple Notification Service):
  → Topic: publish-subscribe messaging
  → Push-based: SNS pushes to all subscribers immediately
  → Many subscribers receive each message (one-to-many)
  → Messages not stored (fire and forget)
  → Subscribers: SQS, Lambda, HTTP, email, SMS
  → Use: notifications, broadcasting events, fan-out

Key comparison:
  SQS: async task queue (process each message once)
  SNS: broadcast notification (all subscribers get it)
  SQS: consumer pulls when ready (flow control)
  SNS: immediate push to all (no flow control)

Fan-out pattern (SNS + SQS together):
  Problem: Order placed → need to: process payment + update inventory + send email + audit log
  All independently, in parallel

  Solution:
  Order Service → SNS Topic "order-placed"
                  ├── SQS Queue "payment-processing"   → Payment Lambda
                  ├── SQS Queue "inventory-updates"    → Inventory Service
                  ├── SQS Queue "email-notifications"  → Email Lambda
                  └── SQS Queue "audit-log"            → Audit Service

  Benefits:
  → All 4 services process in parallel (not sequential)
  → If email service is down: messages queue in SQS until it recovers
  → Add new consumer: subscribe new SQS queue to SNS (no code change)
  → Each service has its own DLQ for failures

  Real DevOps use case:
  New EC2 instance launched → SNS event → 
    SQS → Lambda: register in monitoring (Prometheus)
    SQS → Lambda: add to Ansible inventory
    SQS → Lambda: configure log shipping (Filebeat)
```

> 💡 **Interview tip:** The fan-out pattern is the most powerful SQS+SNS concept. The key advantage: **adding a new consumer requires zero code change** — just subscribe a new SQS queue to the SNS topic. This is the Open/Closed Principle applied to infrastructure. Also critical for production: each SQS queue in a fan-out has its own DLQ — if one downstream service fails, only its messages go to DLQ, not all messages. Other services continue processing normally.

---

### Q36 — Prometheus + Grafana | Scenario-Based | Advanced

> You get a PagerDuty alert at 3 AM: `FIRING: HighErrorRate — HTTP 5xx errors > 5% for payment-service for 5 minutes`. Walk through your complete incident investigation process.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`, `Grafana`, `ELK_Stack`

#### Key Points to Cover:
```
Phase 1 — Acknowledge and assess (first 2 minutes):
  → Acknowledge PagerDuty (stop escalation)
  → Check: is this a real incident or false alarm?
  → Open Grafana payment-service dashboard

Phase 2 — Check current state (minutes 2-5):
  PromQL queries to run immediately:
  
  # Error rate over time
  sum(rate(http_requests_total{service="payment",status=~"5.."}[5m]))
  / sum(rate(http_requests_total{service="payment"}[5m]))
  
  # Which endpoints are erroring?
  sum by(endpoint) (rate(http_requests_total{service="payment",status=~"5.."}[5m]))
  
  # Error rate vs traffic (is this a traffic spike?)
  sum(rate(http_requests_total{service="payment"}[5m]))
  
  # Pod health
  kube_pod_container_status_restarts_total{namespace="production", pod=~"payment.*"}
  
  # Recent deployments (Grafana annotations)
  → Did error rate start after a deployment?

Phase 3 — Correlate with logs (minutes 5-10):
  Kibana query:
  kubernetes.labels.app: "payment-service" AND level: "ERROR"
  → Sort by time → what error messages appear?
  → Same error repeated? → likely a bug or config issue

Phase 4 — Identify root cause:
  Common findings:
  → New deployment introduced bug → rollback
  → Database connection exhausted → scale or fix connection pool
  → Downstream service down → implement circuit breaker
  → Memory pressure causing crashes → check OOMKilled

Phase 5 — Mitigate:
  kubectl rollout undo deployment/payment-service   # if deployment caused it
  kubectl scale deployment/payment-service --replicas=10  # if traffic spike

Phase 6 — Document:
  → Create incident ticket
  → Note: when started, what you found, what you did, current status
```

> 💡 **Interview tip:** The structure of your incident response shows maturity. Senior SREs follow: **1) Don't panic** (acknowledge → assess → mitigate → investigate, in that order — not investigate first). **2) Check recent changes first** — "did anything change in the last 2 hours?" answers 80% of incidents. Grafana deployment annotations make this visual. **3) Separate mitigation from investigation** — rollback immediately if a bad deployment is likely, then investigate why while users are unaffected. Mention **incident timeline documentation** from minute one — it becomes the postmortem.

---

### Q37 — Linux / Bash | Scenario-Based | Advanced

> Write a Bash script that monitors a directory for new files, logs filename/size/timestamp, alerts if file > 100MB, runs continuously, and handles SIGINT gracefully with a summary on exit.

📁 **Reference:** `nawab312/DSA` → `Linux` — Bash scripting, signals, traps sections

#### Key Points to Cover:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Configuration
WATCH_DIR="${1:-/tmp/watched}"
LOG_FILE="/var/log/file-monitor.log"
ALERT_THRESHOLD=$((100 * 1024 * 1024))  # 100MB in bytes
POLL_INTERVAL=5

# Counters for summary
total_files=0
alert_count=0
start_time=$(date +%s)

# Create watch directory if it doesn't exist
mkdir -p "$WATCH_DIR"

# Logging function
log() {
    local level="$1"
    local message="$2"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "${timestamp} [${level}] ${message}" | tee -a "$LOG_FILE"
    [[ "$level" == "ALERT" ]] && echo "${timestamp} [${level}] ${message}" >&2
}

# Cleanup function called on SIGINT
cleanup() {
    local end_time=$(date +%s)
    local duration=$((end_time - start_time))
    echo ""
    echo "========== MONITOR SUMMARY =========="
    echo "Duration:     ${duration} seconds"
    echo "Total files:  ${total_files}"
    echo "Alerts:       ${alert_count} (files > 100MB)"
    echo "Log file:     ${LOG_FILE}"
    echo "====================================="
    exit 0
}

# Register signal handler
trap cleanup SIGINT SIGTERM

log "INFO" "Starting monitor on: ${WATCH_DIR}"

# Track known files
declare -A known_files
for f in "$WATCH_DIR"/*; do
    [[ -f "$f" ]] && known_files["$f"]=1
done

# Main loop
while true; do
    for filepath in "$WATCH_DIR"/*; do
        [[ ! -f "$filepath" ]] && continue
        
        # Skip if already known
        [[ -v known_files["$filepath"] ]] && continue
        
        # New file detected
        known_files["$filepath"]=1
        filename=$(basename "$filepath")
        size=$(stat -c%s "$filepath" 2>/dev/null || echo 0)
        timestamp=$(stat -c%y "$filepath" 2>/dev/null || date)
        total_files=$((total_files + 1))
        
        log "INFO" "New file: ${filename} | Size: ${size} bytes | Time: ${timestamp}"
        
        # Alert if > 100MB
        if [[ $size -gt $ALERT_THRESHOLD ]]; then
            alert_count=$((alert_count + 1))
            log "ALERT" "Large file detected: ${filename} | Size: $((size / 1024 / 1024))MB"
        fi
    done
    
    sleep "$POLL_INTERVAL"
done
```

> 💡 **Interview tip:** Three things separate a production Bash script from a basic one: (1) **`trap` for signal handling** — without it, SIGINT just kills the script abruptly. `trap cleanup SIGINT` ensures graceful exit with summary. (2) **`set -euo pipefail`** at the top — `-e` exits on error, `-u` errors on undefined variables, `-o pipefail` catches pipe failures. (3) **Logging with timestamps to a file AND stderr for alerts** — in production you need both. Also mention `inotifywait` as a more efficient alternative to polling — it uses kernel events instead of checking every 5 seconds.

---

### Q38 — Kubernetes | Conceptual | Advanced

> What is **etcd** in Kubernetes and what role does it play? If etcd goes down, what happens to running Pods, new deployments, scaling, and kubectl commands? How do you backup and restore etcd?

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md`

#### Key Points to Cover:
```
What is etcd:
  → Distributed key-value store (Raft consensus algorithm)
  → The ONLY persistent storage in Kubernetes
  → Stores: all cluster state (pods, services, secrets, configs, nodes)
  → Only kube-apiserver reads/writes to etcd directly
  → HA: typically 3 or 5 instances (odd numbers for quorum)

What happens when etcd goes down:

  Running Pods:          CONTINUE RUNNING ✅
    → kubelet manages pods locally
    → kubelet doesn't need etcd to keep running containers
    → Pods keep serving traffic

  kubectl commands:      FAIL ❌
    → kubectl → kube-apiserver → etcd (cannot read/write)
    → kubectl get pods: "Error: etcdserver: request timed out"
    → Cannot view cluster state

  New deployments:       FAIL ❌
    → kubectl apply → kube-apiserver → etcd (write fails)
    → New objects cannot be created

  Scaling operations:    FAIL ❌
    → HPA scaling, manual scale: cannot update state in etcd

  Self-healing:          STOPS ❌
    → Controllers read desired state from etcd
    → If pod crashes while etcd is down: not restarted
    → Reconciliation loop stalled

Backup:
  ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
    --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

Restore:
  # Stop kube-apiserver first
  ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-backup.db \
    --data-dir=/var/lib/etcd-restore
  # Update etcd config to use new data-dir
  # Restart etcd, then kube-apiserver

Best practices:
  → Backup every 30 minutes (cron job)
  → Store in S3 (encrypted, versioned)
  → Test restore in staging quarterly
  → Monitor: etcd_server_has_leader metric
```

> 💡 **Interview tip:** The most impressive point: **running Pods continue running when etcd is down** — this is because kubelet is autonomous and doesn't need the API server to keep containers running. The cluster looks dead (you can't see anything with kubectl) but traffic is still being served. This distinction matters operationally: if etcd goes down in production, your SLA is not immediately broken — existing traffic continues. The risk is pod crashes during the outage won't be restarted. Always mention the **3-node etcd quorum** requirement: with 3 nodes, you can lose 1 and maintain quorum. With 5 nodes, you can lose 2.

---

### Q39 — Terraform + AWS | Scenario-Based | Advanced

> A developer ran `terraform destroy` in the production environment, destroying RDS databases, EC2 instances, and S3 buckets. **How do you (1) immediately respond, (2) recover the data, and (3) prevent this from ever happening again?**

📁 **Reference:** `nawab312/Terraform` and `nawab312/AWS` — disaster recovery sections

#### Key Points to Cover:
```
Phase 1 — Immediate response (first 30 minutes):
  1. Declare incident, notify stakeholders (CTO, on-call team)
  2. Stop all ongoing work that might interfere
  3. Check S3 versioning → deleted objects still in versioned state?
     aws s3api list-object-versions --bucket critical-bucket
     aws s3api restore-object (if glaciered)
  4. Check RDS automated backups:
     aws rds describe-db-snapshots --snapshot-type automated
     → Automated snapshots retained even after instance deleted (7-35 days)
  5. Document timeline and actions taken (for postmortem)

Phase 2 — Recovery:
  RDS:
    → Restore from latest automated snapshot
    aws rds restore-db-instance-from-db-snapshot \
      --db-instance-identifier prod-db-restored \
      --db-snapshot-identifier <latest-snapshot-id>
    → Update application connection strings
    → Verify data integrity

  EC2:
    → Restore from AMI backup (if you have golden AMI pipeline)
    → OR: re-run Terraform apply (recreates instances, lose runtime data)
    → Data: if on EBS → check EBS snapshots

  S3:
    → Versioning enabled → restore deleted objects
    aws s3api delete-object --bucket my-bucket --key myfile.txt --version-id DeleteMarkerVersionId
    → Versioning NOT enabled → data is GONE

Phase 3 — Prevention (permanent fixes):
  Terraform:
    resource "aws_db_instance" "production" {
      lifecycle { prevent_destroy = true }
    }
    → terraform plan will FAIL if destroy is attempted

  AWS:
    → Enable S3 Object Lock (WORM — cannot delete)
    → Enable RDS deletion protection
    → Enable S3 MFA Delete
    → Separate prod AWS account (prevent devs from having access)

  Process:
    → Require terraform plan review before apply (Atlantis/PR workflow)
    → Never give developers prod AWS credentials
    → CI/CD for prod applies only (no manual runs)
    → workspace/directory separation with separate IAM roles per env
```

> 💡 **Interview tip:** This question tests both **technical recovery knowledge** AND **process/prevention thinking**. The prevention answer is what separates senior candidates: `prevent_destroy = true` in lifecycle blocks is the Terraform safeguard. But the deeper answer is **IAM separation** — if developers don't have IAM permissions to run `terraform destroy` on production, they physically cannot do it regardless of what Terraform allows. The combination: `prevent_destroy` (Terraform safeguard) + separate production AWS account with restricted IAM (infrastructure safeguard) + Atlantis/CI-CD for production applies (process safeguard).

---

### Q40 — Kubernetes + Prometheus + Grafana | Scenario-Based | Advanced

> Set up complete observability for a new Kubernetes microservice: custom metrics, Prometheus auto-discovery, alerting rules for error rate/latency, Grafana RED metrics dashboard, and Slack notifications. Walk through the complete end-to-end setup.

📁 **Reference:** `nawab312/Kubernetes` → `08_OBSERVABILITY.md`, `nawab312/Monitoring-and-Observability`

#### Key Points to Cover:
```
Step 1 — Instrument the application (expose /metrics):
  # Python Flask example:
  from prometheus_client import Counter, Histogram, start_http_server
  
  REQUEST_COUNT = Counter('http_requests_total', 
    'Total requests', ['method', 'status', 'endpoint'])
  REQUEST_DURATION = Histogram('http_request_duration_seconds',
    'Request duration', ['endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0])
  
  # In request handler:
  REQUEST_COUNT.labels(method='GET', status='200', endpoint='/api').inc()
  with REQUEST_DURATION.labels(endpoint='/api').time():
      return handle_request()

Step 2 — Kubernetes Service with named port:
  spec:
    ports:
    - name: http-metrics    # named port (required for ServiceMonitor)
      port: 9090

Step 3 — ServiceMonitor (Prometheus Operator):
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: payment-service
    labels:
      release: prometheus   # must match Prometheus selector
  spec:
    selector:
      matchLabels:
        app: payment-service
    endpoints:
    - port: http-metrics
      interval: 30s

Step 4 — PrometheusRule for alerting:
  groups:
  - name: payment_alerts
    rules:
    - alert: HighErrorRate
      expr: rate(http_requests_total{status=~"5..",service="payment"}[5m]) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Payment error rate {{ $value | humanizePercentage }}"

Step 5 — Grafana dashboard (RED metrics):
  Row 1: Rate      → sum(rate(http_requests_total[5m]))
  Row 2: Errors    → rate(http_requests_total{status=~"5.."}[5m]) / rate(...)
  Row 3: Duration  → histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

Step 6 — AlertManager Slack config:
  receivers:
  - name: slack
    slack_configs:
    - channel: '#alerts'
      api_url: $SLACK_WEBHOOK
      title: '{{ .CommonAnnotations.summary }}'
```

> 💡 **Interview tip:** The **RED methodology** (Rate, Errors, Duration) is the standard for service monitoring — mention it explicitly. These three metrics answer: "Is the service being used? Is it broken? Is it slow?" For Kubernetes workloads, also mention **USE methodology** (Utilization, Saturation, Errors) for infrastructure metrics. Having both RED (service health) and USE (infrastructure health) dashboards is the complete observability picture. Also mention that ServiceMonitor labels must match the Prometheus CRD's `serviceMonitorSelector` — this is the #1 misconfiguration with Prometheus Operator.

---

### Q41 — AWS | Conceptual | Advanced

> Explain the difference between **AWS Lambda** and **AWS ECS**. Your team is debating which to use for a new microservice. What questions would you ask to decide, and what are the trade-offs?

📁 **Reference:** `nawab312/AWS` — Lambda, ECS, serverless sections

#### Key Points to Cover:
```
Lambda:
  → Serverless: no infrastructure to manage
  → Event-driven: triggered by events (API GW, S3, SQS, etc.)
  → Scales to zero: $0 when no traffic
  → Billing: per invocation + per 100ms execution time
  → Limits: 15 min max duration, 10GB memory, 10GB ephemeral storage
  → Cold starts: 100ms-2s (language dependent)
  → Stateless by design

ECS (Fargate or EC2):
  → Container orchestration
  → Long-running services (always on)
  → No time limit per request
  → Billing: per hour of compute (vCPU + memory)
  → No cold starts (containers always running)
  → More control: networking, filesystem, custom runtimes

Decision questions:
  1. How long does each request take?
     < 15 min → Lambda possible
     > 15 min → ECS required

  2. How variable is the traffic?
     Highly variable/bursty → Lambda (scale to zero, scales instantly)
     Steady/predictable → ECS (cheaper for sustained load)

  3. Are there cold start concerns?
     User-facing API → ECS (Lambda cold starts hurt UX)
     Background processing → Lambda (cold start acceptable)

  4. Does it maintain state?
     Stateless → Lambda
     Stateful (long connections, WebSockets) → ECS

  5. What's the request duration distribution?
     P99 < 10s → Lambda
     Some requests can take 5+ minutes → ECS

Choose Lambda:
  ✅ Event-driven processing (S3 uploads, DynamoDB streams)
  ✅ Scheduled tasks (cron-like)
  ✅ Traffic: 0 to millions (auto-scale)
  ✅ Cost: pay per use (bursty or low traffic)

Choose ECS:
  ✅ Long-running HTTP services (APIs)
  ✅ WebSocket connections
  ✅ High-traffic steady workloads
  ✅ Custom runtimes / OS-level config needed
```

> 💡 **Interview tip:** The key differentiator often missed: **ECS is cheaper at sustained high traffic, Lambda is cheaper at bursty/low traffic**. A service processing 10 requests/day → Lambda is essentially free. A service processing 10M requests/day → ECS on Fargate is significantly cheaper. Also mention the hybrid approach: use Lambda for async background processing (resize images, send emails) while ECS serves the synchronous API. This shows architectural maturity — not everything must be one or the other.

---

### Q42 — AWS + Kubernetes | Troubleshooting | Advanced

> EKS Pods are running but cannot reach an RDS database in a private subnet. A junior engineer says "nothing changed." **Walk through your complete troubleshooting process** from Pod to RDS.

📁 **Reference:** `nawab312/AWS` — VPC, Security Groups, RDS, `nawab312/Kubernetes` → `03_NETWORKING.md`

#### Key Points to Cover:
```
Step 1 — Verify from inside the Pod:
  kubectl exec -it <pod> -n production -- sh
  
  # Test DNS resolution
  nslookup mydb.cluster-xyz.us-east-1.rds.amazonaws.com
  
  # Test TCP connectivity
  nc -zv mydb.cluster-xyz.us-east-1.rds.amazonaws.com 5432
  # "succeeded" → network OK, problem is auth/app level
  # "timed out" → network/SG problem

Step 2 — Check RDS Security Group:
  aws rds describe-db-instances --db-instance-identifier mydb \
    --query 'DBInstances[0].VpcSecurityGroups'
  
  aws ec2 describe-security-groups --group-ids sg-xxxx
  # Does it allow inbound 5432 from EKS node/pod security group?
  
  # Common mistake: SG allows from specific node SG
  # but pods got new SG with AWS VPC CNI (aws-node assigns pod SG)

Step 3 — Check EKS Node security group:
  aws ec2 describe-instances --instance-ids <node-id> \
    --query 'Reservations[0].Instances[0].SecurityGroups'
  # Is this SG allowed in RDS inbound rules?

Step 4 — Check subnet routing:
  # Pod is in private subnet → can it reach RDS subnet?
  # Are they in same VPC? Different VPC (needs VPC peering)?
  aws ec2 describe-route-tables

Step 5 — Check NACLs:
  # NACL on RDS subnet: allows inbound 5432 from pod CIDR?
  # NACL on pod subnet: allows outbound to RDS subnet on 5432?
  # NACL on RDS subnet: allows outbound to ephemeral ports (1024-65535)?

Step 6 — Check "nothing changed" (it always changed):
  aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=ModifyDBInstance \
    --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ)
  # Find who changed what in last 24 hours

Common root causes:
  → SG rule accidentally removed
  → EKS node rolled (new nodes have different SG)
  → RDS rotated to different AZ (different subnet)
  → Pod security groups feature enabled (pods now need their own SG rule)
```

> 💡 **Interview tip:** "Nothing changed" is almost never true. Your first action on hearing this: **check CloudTrail** for the last 24 hours of changes to security groups, RDS, and VPC. CloudTrail is the audit trail that proves what changed. In real production, the most common "nothing changed" causes are: (1) an EKS node group was updated (new nodes with new instance IDs don't match old SG rules), (2) someone modified the RDS security group to add a rule but accidentally removed an existing one, or (3) RDS automated maintenance caused an AZ failover to a subnet the pod SG doesn't cover.

---

### Q43 — Kubernetes | Conceptual | Advanced

> Explain the difference between **HPA**, **VPA**, and **KEDA**. When would you use each? Can you use HPA and VPA together? What happens if you try?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

#### Key Points to Cover:
```
HPA (Horizontal Pod Autoscaler):
  → Adds/removes PODS (horizontal)
  → Metrics: CPU%, memory%, custom metrics, external metrics
  → Min/max replicas configured
  → Built into Kubernetes (no install needed, but needs metrics-server)
  → Cannot scale to zero
  → Best for: stateless services with variable load

VPA (Vertical Pod Autoscaler):
  → Adjusts CPU/memory REQUESTS on existing pods (vertical)
  → Modes: Off (recommend only), Auto (apply), Initial (at pod creation)
  → Requires pod restart to apply (disruptive)
  → Best for: stateful workloads, right-sizing, batch jobs
  → Separate install (not built-in)

KEDA (Kubernetes Event Driven Autoscaler):
  → Extends HPA with event sources (60+ scalers)
  → CAN scale to ZERO (killer feature vs HPA)
  → Sources: SQS depth, Kafka lag, Prometheus metrics, Cron, Redis, etc.
  → Replaces HPA for event-driven workloads
  → Best for: queue consumers, batch processors, event-driven workloads

Can HPA + VPA be used together?
  Short answer: NOT on CPU (conflict), YES for memory

  Conflict:
  → HPA scales pods based on CPU usage
  → VPA changes CPU requests (the denominator of CPU%)
  → VPA increases CPU request → CPU% appears to drop → HPA scales down
  → HPA scales down → fewer pods → CPU% rises → HPA scales up
  → Creates oscillation/thrashing

  Safe combinations:
  ✅ HPA on CPU + VPA in recommendation mode (Off) — VPA just recommends
  ✅ HPA on custom metrics + VPA on CPU/memory (no conflict)
  ✅ KEDA (replaces HPA) + VPA (no conflict — different dimensions)

Real production setup:
  Stateless API:    HPA (CPU 60%) + KEDA if SQS-driven
  Batch processor:  KEDA (SQS depth) + VPA (right-size each pod)
  Database:         VPA only (StatefulSet, no HPA on stateful workloads)
```

> 💡 **Interview tip:** The **HPA + VPA conflict** is a frequently asked advanced question. The root cause is that HPA uses CPU utilization percentage (currentCPU / requestedCPU × 100%), and VPA changes the denominator (requestedCPU). When VPA increases CPU requests, the percentage drops, causing HPA to scale down — even though the pods are just as busy. The safe solution: use HPA on custom metrics (requests per second, queue depth) rather than CPU, then VPA can safely manage CPU requests without interfering with HPA's scaling decisions.

---

### Q44 — Linux + Bash | Scenario-Based | Advanced

> You receive an alert: `High number of CLOSE_WAIT connections on production server — current count: 850`. The server runs a Java web application. **What is CLOSE_WAIT, why does it happen, what is the risk, and how do you investigate and fix it?**

📁 **Reference:** `nawab312/DSA` → `Linux` — TCP states, socket management sections

#### Key Points to Cover:
```
What CLOSE_WAIT means:
  TCP 4-way close sequence:
  Remote → FIN → Server: remote wants to close
  Server → ACK → Remote: acknowledged (server now in CLOSE_WAIT)
  ... server application should close its end ...
  Server → FIN → Remote: server closes its end
  Server → TIME_WAIT → CLOSED

  CLOSE_WAIT = server received remote's FIN, acknowledged it,
               but application has NOT called close() on the socket yet

Why it's a problem:
  → 850 CLOSE_WAIT sockets = 850 sockets not being closed by the app
  → Each socket consumes: file descriptor, kernel memory, port
  → File descriptors limited: default 1024 (or 65535 if raised)
  → At 65535 sockets: new connections get EMFILE error → app crash

Root cause (almost always Java application bug):
  → Connection pool not returning connections to pool
  → HTTP client not calling response.close() after reading
  → No timeout on idle connections
  → Exception path that bypasses close() call

Diagnosis:
  # Count CLOSE_WAIT sockets
  ss -tan state close-wait | wc -l
  ss -tan state close-wait | grep :8080  # on which port?
  
  # Which process owns them?
  ss -tanp state close-wait | grep java
  
  # Check file descriptor usage
  cat /proc/$(pgrep java)/fd | wc -l  # FDs this process has open
  cat /proc/$(pgrep java)/limits | grep "open files"  # limit
  
  # Remote IP causing most CLOSE_WAIT?
  ss -tan state close-wait | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn

Fix:
  Immediate: restart Java application (CLOSE_WAIT sockets cleared)
  
  Long-term (code fix):
    // Always use try-with-resources in Java:
    try (HttpResponse response = httpClient.execute(request)) {
        // response automatically closed
    }
    
    // OR: configure connection pool timeouts:
    connectionPool.setKeepAliveTime(60, TimeUnit.SECONDS)
    
    # TCP keepalive to detect dead connections:
    sysctl -w net.ipv4.tcp_keepalive_time=60
    sysctl -w net.ipv4.tcp_keepalive_intvl=10
```

> 💡 **Interview tip:** CLOSE_WAIT vs TIME_WAIT is a classic confusion: **TIME_WAIT is normal** (it's the INITIATOR of close waiting before fully closing — prevents duplicate packet issues). **CLOSE_WAIT is a bug** — it means the application received a close request but never acted on it. Thousands of TIME_WAIT = normal healthy server. Thousands of CLOSE_WAIT = connection leak in application code. Also mention that this is especially common with HTTP/1.1 keep-alive connections — the client closes the connection but the Java application holds the socket open because it didn't call `close()` on the response body.

---

### Q45 — CI-CD + Kubernetes + AWS | System Design | Advanced

> You are the first DevOps/SRE engineer at a startup with a monolithic app on a single EC2 with no CI/CD, no monitoring, and manual SSH deployments. The CTO asks you to design a complete DevOps platform. **What would you build, in what order, and why?**

📁 **Reference:** All repositories — capstone system design question

#### Key Points to Cover:
```
Priority order (business impact per effort):

Phase 1 — Stop the bleeding (Week 1-2):
  1. Source control hygiene:
     → Protect main branch (no direct push)
     → Require PR reviews
     → Reason: foundational, free, immediate

  2. Basic CI (GitHub Actions):
     → Run tests on every PR
     → Block merge if tests fail
     → Reason: catch bugs before production, 2-3 hours to set up

  3. Basic monitoring:
     → CloudWatch alarms (CPU, memory, disk)
     → Set up PagerDuty
     → Reason: currently flying blind — know when things break

Phase 2 — Reliability (Week 3-4):
  4. Automated deployments (no more SSH):
     → GitHub Actions: push to main → deploy to EC2 via AWS CodeDeploy
     → Or: containerize + ECS Fargate (skip EC2 management entirely)
     → Reason: manual deploys = inconsistent, slow, risky

  5. Multi-AZ deployment:
     → ALB + ASG across 2-3 AZs
     → RDS Multi-AZ
     → Reason: current single EC2 = single point of failure

  6. Secrets management:
     → Move hardcoded secrets to AWS Secrets Manager/Parameter Store
     → Reason: security requirement, quick win

Phase 3 — Scalability (Month 2):
  7. Containerize application:
     → Dockerfile → ECR → ECS Fargate OR EKS
     → Reason: reproducible deployments, easy scaling

  8. Infrastructure as Code:
     → Terraform for all infrastructure
     → Reason: reproducible, reviewable, version-controlled

  9. Full observability:
     → Prometheus + Grafana OR CloudWatch + Grafana
     → Centralized logging (CloudWatch Logs or ELK)
     → Reason: cannot improve what you cannot measure

Phase 4 — Maturity (Month 3+):
  10. Kubernetes (EKS) if warranted:
      → Only after: containerized, Terraform managed, monitoring in place
      → Reason: K8s adds operational complexity — only worth it at scale

  11. GitOps with ArgoCD
  12. SLOs and error budgets
  13. Chaos engineering

Key principle: each phase provides immediate value AND enables the next.
Don't jump to K8s before having basic CI/CD working.
```

> 💡 **Interview tip:** This question tests **engineering judgment and prioritization** more than technical knowledge. The answer that impresses: acknowledge that **order matters** — Kubernetes before CI/CD is backwards. The most common mistake new DevOps engineers make: jumping to K8s/microservices before solving the fundamentals (automated deployments, monitoring, secrets management). The senior answer: "Monitoring and CI before containerization, containerization before Kubernetes, all infrastructure as code from day one." Also mention **business continuity**: Phase 1 improvements should not break existing deployments — strangler fig pattern where new infrastructure runs alongside old until ready to cut over.

---

*Q1–Q45 Enhanced with Key Points + Interview Tips*
*DevOps / SRE Interview Preparation — nawab312 GitHub repositories*
