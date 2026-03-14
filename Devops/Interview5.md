# DevOps / SRE / Cloud Interview Questions (181–225)
### Enhanced Edition — With Key Points & Interview Tips

> Part 5 of interview preparation questions based on your GitHub repositories.
> Every question includes: **Key Points to Cover** + **💡 Interview Tip**

---

## Table of Contents

| # | Topic | Type | Difficulty |
|---|---|---|---|
| 181 | Kubernetes | Conceptual | Advanced |
| 182 | AWS | Scenario-Based | Advanced |
| 183 | Linux / Bash | Scenario-Based | Advanced |
| 184 | Terraform | Conceptual | Advanced |
| 185 | Prometheus | Conceptual | Advanced |
| 186 | Kubernetes | Troubleshooting | Advanced |
| 187 | Git | Conceptual | Advanced |
| 188 | AWS | Conceptual | Advanced |
| 189 | Jenkins | Conceptual | Advanced |
| 190 | ELK Stack | Troubleshooting | Advanced |
| 191 | Kubernetes | Scenario-Based | Advanced |
| 192 | Linux / Bash | Conceptual | Advanced |
| 193 | Terraform | Troubleshooting | Advanced |
| 194 | GitHub Actions | Troubleshooting | Advanced |
| 195 | AWS | Troubleshooting | Advanced |
| 196 | Prometheus | Scenario-Based | Advanced |
| 197 | Kubernetes | Conceptual | Advanced |
| 198 | ArgoCD | Conceptual | Advanced |
| 199 | Linux / Bash | Troubleshooting | Advanced |
| 200 | AWS | Scenario-Based | Advanced |
| 201 | Kubernetes | Troubleshooting | Advanced |
| 202 | Grafana | Conceptual | Advanced |
| 203 | Terraform | Scenario-Based | Advanced |
| 204 | Python | Scenario-Based | Advanced |
| 205 | AWS | Conceptual | Advanced |
| 206 | Kubernetes | Scenario-Based | Advanced |
| 207 | ELK Stack | Scenario-Based | Advanced |
| 208 | Jenkins | Scenario-Based | Advanced |
| 209 | Linux / Bash | Scenario-Based | Advanced |
| 210 | AWS | Scenario-Based | Advanced |
| 211 | Kubernetes | Conceptual | Advanced |
| 212 | GitHub Actions | Scenario-Based | Advanced |
| 213 | Terraform | Conceptual | Advanced |
| 214 | Prometheus | Troubleshooting | Advanced |
| 215 | AWS | Conceptual | Advanced |
| 216 | Kubernetes | Scenario-Based | Advanced |
| 217 | ArgoCD | Troubleshooting | Advanced |
| 218 | Linux / Bash | Conceptual | Advanced |
| 219 | AWS | Scenario-Based | Advanced |
| 220 | Kubernetes | Conceptual | Advanced |
| 221 | Grafana | Scenario-Based | Advanced |
| 222 | Terraform | Scenario-Based | Advanced |
| 223 | Git | Scenario-Based | Advanced |
| 224 | AWS + Kubernetes | Scenario-Based | Advanced |
| 225 | All Topics | System Design | Advanced |

---

## Questions

---

### Q181 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `leaderElection`** mechanism. Why is leader election needed in Kubernetes controllers?
>
> Which Kubernetes control plane components use leader election? What happens if leader election fails? How would you debug a situation where multiple controller-manager Pods think they are the leader?

📁 **Reference:** `nawab312/Kubernetes` → `01_CORE_ARCHITECTURE_FUNDAMENTALS.md` — control plane HA, leader election sections

#### Key Points to Cover in Your Answer:

**Why leader election is needed:**
- In HA control plane setups, multiple replicas of `kube-controller-manager` and `kube-scheduler` run simultaneously
- Only ONE should be active at a time — others must be on standby
- Without leader election → split-brain → multiple controllers reconciling same resources → conflicts

**How it works:**
- Uses a **Kubernetes `Lease` object** (in `kube-system` namespace)
- The active leader **continuously renews** the Lease (every `leaseDuration` seconds)
- If leader fails to renew → Lease expires → another instance acquires it
- Key parameters: `leaseDuration` (15s default), `renewDeadline` (10s), `retryPeriod` (2s)

**Components using leader election:**
- `kube-controller-manager` → `kube-controller-manager` Lease
- `kube-scheduler` → `kube-scheduler` Lease
- `cloud-controller-manager` → its own Lease
- Custom controllers built with controller-runtime → use leader election automatically

**Debugging multiple leaders:**
```bash
# Check who holds the Lease
kubectl get lease kube-controller-manager -n kube-system -o yaml
# Look at: spec.holderIdentity and spec.renewTime

# If renewTime is old → leader crashed but Lease not released yet
# Wait for leaseDuration to expire → new leader takes over

# Force release stale Lease (emergency only)
kubectl patch lease kube-controller-manager -n kube-system \
  --type=json -p='[{"op":"remove","path":"/spec/holderIdentity"}]'
```

**What happens if leader election fails:**
- No controller-manager active → no reconciliation → Deployments don't scale, crashed Pods not replaced
- Scheduler not active → new Pods stuck in `Pending`

> 💡 **Interview tip:** Interviewers love asking "what happens if etcd is healthy but controller-manager is down?" — the answer is: **Pods keep running** (kubelet manages them locally) but **no self-healing** — crashed pods stay down, deployments don't scale. Leader election failure has the same effect as controller-manager being down.

---

### Q182 — AWS | Scenario-Based | Advanced

> Your team has just enabled **AWS GuardDuty** and within the first hour you receive a critical finding:
>
> `UnauthorizedAccess:IAMUser/MaliciousIPCaller — API call from known malicious IP`
>
> Walk me through your **complete incident response process** — from the GuardDuty finding to containment, investigation, and remediation.

📁 **Reference:** `nawab312/AWS` — GuardDuty, IAM, CloudTrail, incident response sections

#### Key Points to Cover in Your Answer:

**Immediate Containment (first 15 minutes):**
```bash
# Step 1: Identify the IAM entity involved
# GuardDuty finding shows: which IAM user/role, which IP, which API calls

# Step 2: Immediately disable/quarantine the IAM entity
aws iam update-user --user-name <compromised-user> \
  --no-enable-mfa  # revoke MFA
aws iam attach-user-policy --user-name <compromised-user> \
  --policy-arn arn:aws:iam::aws:policy/AWSDenyAll  # deny all actions

# If it is an IAM role being assumed:
# Revoke all active sessions by updating trust policy to deny all
aws iam update-assume-role-policy --role-name <role> \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Deny","Principal":"*","Action":"sts:AssumeRole"}]}'

# Step 3: Block the malicious IP at WAF/Security Group level
aws ec2 create-security-group-rule ... # block IP
```

**Investigation:**
```bash
# Query CloudTrail for all actions by this entity in last 24h
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=<user> \
  --start-time $(date -d '24 hours ago' -u +%Y-%m-%dT%H:%M:%SZ)

# Look for:
# - What resources were accessed?
# - Were any new IAM users/roles created?
# - Were any S3 buckets made public?
# - Were any EC2 instances launched?
# - Were any credentials exfiltrated?
```

**Blast Radius Assessment:**
- Check if new IAM users/roles were created (attacker persistence)
- Check S3 GetObject calls → data exfiltration
- Check EC2 RunInstances → crypto mining
- Check CloudTrail → were logs disabled?

**Remediation:**
- Rotate all access keys for the compromised user
- Enable MFA enforcement for all users via SCP
- Review and tighten IAM policies (least privilege)
- Enable GuardDuty automated response via EventBridge + Lambda

> 💡 **Interview tip:** The most important thing to mention first is **"contain before investigate"** — immediately revoke permissions before spending time understanding what happened. An attacker with active credentials can do more damage while you are investigating. Attach `AWSDenyAll` policy as the fastest containment action — it does not delete the user (preserving forensic evidence) but stops all further damage.

---

### Q183 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that implements a **TCP port scanner**:
> - Takes a hostname and port range as arguments (e.g., `./scan.sh 192.168.1.1 1-1024`)
> - Scans ports **in parallel** (max 50 concurrent)
> - Shows **open ports** with service name if identifiable
> - Shows **scan progress** as a percentage
> - Outputs results to both **stdout** and a timestamped log file
> - Has a **configurable timeout** per port (default 1 second)

📁 **Reference:** `nawab312/DSA` → `Linux` — networking, bash scripting, parallel execution, /dev/tcp sections

#### Key Points to Cover in Your Answer:

```bash
#!/bin/bash
# TCP Port Scanner using /dev/tcp

HOST="${1}"
RANGE="${2:-1-1024}"
TIMEOUT="${3:-1}"
MAX_PARALLEL=50
LOGFILE="/tmp/portscan_$(date +%Y%m%d_%H%M%S).log"

# Parse port range
START_PORT=$(echo "$RANGE" | cut -d'-' -f1)
END_PORT=$(echo "$RANGE" | cut -d'-' -f2)
TOTAL_PORTS=$((END_PORT - START_PORT + 1))

# Common service names
get_service() {
    local port=$1
    case $port in
        21) echo "FTP" ;; 22) echo "SSH" ;; 23) echo "Telnet" ;;
        25) echo "SMTP" ;; 53) echo "DNS" ;; 80) echo "HTTP" ;;
        443) echo "HTTPS" ;; 3306) echo "MySQL" ;; 5432) echo "PostgreSQL" ;;
        6379) echo "Redis" ;; 8080) echo "HTTP-Alt" ;; *) echo "Unknown" ;;
    esac
}

# Scan a single port
scan_port() {
    local host="$1"
    local port="$2"
    if timeout "$TIMEOUT" bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null; then
        service=$(get_service "$port")
        result="OPEN  Port $port ($service)"
        echo "$result" | tee -a "$LOGFILE"
    fi
}

export -f scan_port get_service
export TIMEOUT LOGFILE

# Progress tracking
SCANNED=0
echo "Scanning $HOST ports $RANGE (timeout: ${TIMEOUT}s)"
echo "Log file: $LOGFILE"
echo "---"

# Parallel scan with progress
for port in $(seq "$START_PORT" "$END_PORT"); do
    # Limit concurrent jobs
    while [[ $(jobs -r | wc -l) -ge $MAX_PARALLEL ]]; do
        sleep 0.1
    done

    scan_port "$HOST" "$port" &

    SCANNED=$((SCANNED + 1))
    PERCENT=$((SCANNED * 100 / TOTAL_PORTS))
    printf "\rProgress: %d%% (%d/%d ports)" "$PERCENT" "$SCANNED" "$TOTAL_PORTS" >&2
done

# Wait for all background jobs
wait

echo ""
echo "Scan complete. Results saved to: $LOGFILE"
```

**Key concepts used:**
- `/dev/tcp/host/port` — bash built-in TCP connection (no ncat/nc required)
- `jobs -r` — count running background jobs for concurrency control
- `timeout` command — per-port timeout
- `export -f` — export functions to subshell for parallel execution
- `printf "\r"` — overwrite progress line in terminal

> 💡 **Interview tip:** Using `/dev/tcp` is the **pure bash approach** — no external tools needed. Mention that in production you would use `nmap` for proper scanning, but knowing `/dev/tcp` shows deep bash knowledge. Also mention that port scanning your own infrastructure is fine but scanning external systems without permission is illegal — always qualify this in an interview.

---

### Q184 — Terraform | Conceptual | Advanced

> Explain **Terraform `provider` configuration** in depth. What is the difference between **provider aliasing** and **multiple provider configurations**?
>
> Write a Terraform configuration that deploys resources in **3 different AWS regions simultaneously** using provider aliases, and explain how modules consume aliased providers.

📁 **Reference:** `nawab312/Terraform` — provider configuration, aliasing, multi-region sections

#### Key Points to Cover in Your Answer:

**Provider aliasing — when you need multiple instances of the same provider:**

```hcl
# Default provider (us-east-1)
provider "aws" {
  region = "us-east-1"
}

# Aliased providers for other regions
provider "aws" {
  alias  = "us-west-2"
  region = "us-west-2"
}

provider "aws" {
  alias  = "eu-west-1"
  region = "eu-west-1"
}

# Using aliased provider directly on a resource
resource "aws_s3_bucket" "us_east" {
  bucket = "my-bucket-us-east"
  # uses default provider (no alias specified)
}

resource "aws_s3_bucket" "us_west" {
  provider = aws.us-west-2   # uses aliased provider
  bucket   = "my-bucket-us-west"
}

resource "aws_s3_bucket" "eu" {
  provider = aws.eu-west-1
  bucket   = "my-bucket-eu"
}
```

**Passing aliased providers to modules:**
```hcl
# Root module passes provider to child module
module "us_west_vpc" {
  source = "./modules/vpc"
  providers = {
    aws = aws.us-west-2   # pass aliased provider to module
  }
  cidr_block = "10.1.0.0/16"
}

module "eu_vpc" {
  source = "./modules/vpc"
  providers = {
    aws = aws.eu-west-1
  }
  cidr_block = "10.2.0.0/16"
}

# In the child module — no alias needed, uses whatever was passed
# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  # automatically uses the provider passed from parent
}
```

**Key rules:**
- Default provider = no alias specified
- Aliased provider MUST be explicitly passed to modules via `providers` block
- Child modules declare `required_providers` to document expected providers
- `terraform providers` command shows provider dependency graph

> 💡 **Interview tip:** The most common mistake with provider aliases is forgetting to pass them to modules via the `providers` block — the module silently uses the default provider instead of the aliased one, deploying resources in the wrong region. Always use `terraform providers` to verify which provider each module is using before applying.

---

### Q185 — Prometheus | Conceptual | Advanced

> Explain **PromQL functions** in depth. What is the difference between:
> - `rate()` vs `irate()`
> - `increase()` vs `delta()`
> - `avg_over_time()` vs `avg()`
>
> When would `irate()` give misleading results? Give a concrete example showing when each function is the correct choice.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — PromQL functions sections

#### Key Points to Cover in Your Answer:

**`rate()` vs `irate()`:**

| Function | Calculation | Best For |
|---|---|---|
| `rate()` | Average rate over ENTIRE range window | Dashboards, slow-changing metrics, alerting |
| `irate()` | Rate between LAST 2 data points only | Spiky real-time metrics, debugging |

```promql
# rate() — smoothed average, good for dashboards
rate(http_requests_total[5m])
# Returns: average requests/second over last 5 minutes

# irate() — instantaneous, only uses last 2 scrapes
irate(http_requests_total[5m])
# Returns: rate based only on most recent 2 data points within 5m window
# The [5m] is just the lookback window to find those 2 points
```

**When `irate()` gives misleading results:**
```
Scenario: Metrics scraped every 15s
irate uses only the last 2 scrapes (30 seconds of data)

If a scrape is missed (timeout):
  irate jumps dramatically because it uses stale data point
  → Graphs show wild spikes that don't reflect reality
  → Alerting based on irate fires incorrectly

Scenario: Low-traffic service (5 requests/minute)
  irate might show 0 (no requests in last 2 scrapes) then spike to 60/s
  → Very noisy, not useful

Use rate() for stable dashboards, use irate() only for real-time debugging
```

**`increase()` vs `delta()`:**
```promql
# increase() — for COUNTERS (only increases)
# Returns: total increase over the range window
increase(http_requests_total[1h])
# = number of requests in the last hour
# Automatically handles counter resets (process restarts)

# delta() — for GAUGES (can go up or down)
# Returns: difference between first and last value in range
delta(node_memory_MemFree_bytes[1h])
# = how much free memory changed in the last hour (can be negative)
# Does NOT handle counter resets → wrong for counters
```

**`avg_over_time()` vs `avg()`:**
```promql
# avg_over_time() — averages a SINGLE metric over TIME
avg_over_time(node_cpu_usage[1h])
# = average CPU usage of this instance over the last hour (time aggregation)

# avg() — averages across MULTIPLE TIME SERIES (label aggregation)
avg(node_cpu_usage)
# = average CPU usage across ALL instances at this moment
avg by(region) (node_cpu_usage)
# = average per region across instances
```

> 💡 **Interview tip:** The `rate()` vs `irate()` question is asked very frequently. The key answer is: **always use `rate()` for alerting rules** — `irate()` is too sensitive to missed scrapes and will cause false alerts. `irate()` is only useful for real-time debugging in Grafana Explore mode where you want to see the most current activity.

---

### Q186 — Kubernetes | Troubleshooting | Advanced

> Your Kubernetes cluster shows **`ImagePullBackOff`** on ALL pods across ALL nodes suddenly — not just one service. This was working fine 10 minutes ago.
>
> What are the cluster-wide causes of ImagePullBackOff (as opposed to per-pod causes) and how would you diagnose and fix this quickly?

📁 **Reference:** `nawab312/Kubernetes` → `14_TROUBLESHOOTING.md` — ImagePullBackOff, registry connectivity, credentials sections

#### Key Points to Cover in Your Answer:

**Cluster-wide vs per-pod causes:**
- Per-pod: wrong image name, wrong tag → only affects that service
- **Cluster-wide: affects ALL pods → infrastructure or credential issue**

**Cluster-wide causes:**

**1. Container Registry is DOWN or unreachable:**
```bash
# Test registry connectivity from a node
kubectl debug node/<node-name> -it --image=busybox
curl -v https://123456.dkr.ecr.us-east-1.amazonaws.com/v2/

# Check ECR status / Docker Hub status page
# Check security group / NACL outbound rules
# Check NAT Gateway is working
```

**2. ECR Authentication expired (most common for AWS):**
```bash
# ECR tokens expire every 12 hours
# Check if the imagePullSecret is expired
kubectl get secret ecr-secret -n default -o jsonpath='{.data.\.dockerconfigjson}' | \
  base64 -d | jq '.auths'

# Check token expiry
kubectl describe secret ecr-secret | grep "Created"

# Refresh ECR token
kubectl delete secret ecr-secret
kubectl create secret docker-registry ecr-secret \
  --docker-server=<ecr-registry> \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1)
```

**3. Node IAM role lost ECR permissions:**
```bash
# Check if node IAM role has ECR read policy
aws iam list-attached-role-policies --role-name <node-role>
# Must have: AmazonEC2ContainerRegistryReadOnly or equivalent

# Test from node
aws ecr get-login-password --region us-east-1
# If this fails → IAM issue
```

**4. Network policy blocking registry traffic:**
```bash
kubectl get networkpolicy --all-namespaces
# Check if any policy blocks egress on port 443
```

**5. DNS resolution failure (cannot resolve registry hostname):**
```bash
kubectl exec -it <any-pod> -- nslookup 123456.dkr.ecr.us-east-1.amazonaws.com
# If this fails → CoreDNS issue
kubectl get pods -n kube-system | grep coredns
```

**Quick diagnosis flow:**
```
kubectl describe pod <any-failing-pod> | grep -A5 "Events"
→ Shows exact error: "no basic auth credentials" vs "connection refused" vs "not found"
```

> 💡 **Interview tip:** The **most common cluster-wide ImagePullBackOff** cause in AWS environments is **ECR token expiration** — ECR tokens are only valid for 12 hours. If you are not using an auto-refreshing mechanism (like the `amazon-ecr-credential-helper` or a CronJob to refresh the imagePullSecret), you will see this every 12 hours. The fix is to use the `ecr-credential-helper` sidecar or IRSA for node-level ECR access instead of static imagePullSecrets.

---

### Q187 — Git | Conceptual | Advanced

> Explain **Git `reflog`** — what it tracks, how long it keeps data, and how it saves you from disaster.
>
> Give **3 real-world scenarios** where `git reflog` would be the only way to recover lost work. Write the exact commands you would use in each scenario to recover.

📁 **Reference:** `nawab312/CI_CD` → `Git` — reflog, recovery, advanced Git sections

#### Key Points to Cover in Your Answer:

**What reflog tracks:**
- Every movement of HEAD — commits, resets, checkouts, rebases, merges
- Stored locally only (not pushed to remote)
- Kept for **90 days** by default (`gc.reflogExpire`)
- Even "deleted" commits are recoverable while in reflog

```bash
# View reflog
git reflog
# Output:
# abc1234 HEAD@{0}: reset: moving to HEAD~3
# def5678 HEAD@{1}: commit: add payment feature
# ghi9012 HEAD@{2}: commit: fix auth bug
# jkl3456 HEAD@{3}: checkout: moving from main to feature/payments
```

**Scenario 1 — Accidental `git reset --hard`:**
```bash
# Oops — reset --hard removed 3 commits
git reset --hard HEAD~3

# Recovery:
git reflog                          # find the commit before the reset
# def5678 HEAD@{1}: commit: add payment feature
git reset --hard def5678            # restore to that commit
# OR create a branch to be safe:
git checkout -b recovery-branch def5678
```

**Scenario 2 — Deleted a branch that was not merged:**
```bash
# Accidentally deleted feature branch
git branch -D feature/important-work

# Recovery:
git reflog | grep "feature/important-work"
# abc9999 HEAD@{5}: checkout: moving from feature/important-work to main
git checkout -b feature/important-work abc9999
```

**Scenario 3 — Bad rebase destroyed commit history:**
```bash
# Rebase went wrong — commits look mangled
git rebase main  # something went wrong

# Recovery:
git reflog
# Find the ORIG_HEAD entry (set automatically before rebase)
# bcd1111 HEAD@{3}: rebase (start): checkout main
git reset --hard ORIG_HEAD
# OR find specific pre-rebase commit:
git reset --hard HEAD@{4}
```

> 💡 **Interview tip:** Always mention that `git reflog` is **local only** — if you lose your local repository entirely (disk failure), reflog cannot save you. That is why remote backups and pushing frequently matters. Also mention `git fsck --lost-found` as a complementary tool that finds dangling commits not referenced by reflog.

---

### Q188 — AWS | Conceptual | Advanced

> Explain **AWS Step Functions**. What problem does it solve that Lambda alone cannot?
>
> What is the difference between **Standard** and **Express** workflows? Give a real-world DevOps use case — design a Step Functions workflow for a **multi-stage deployment pipeline** with approval gates, rollback on failure, and notification steps.

📁 **Reference:** `nawab312/AWS` — Step Functions, workflow orchestration sections

#### Key Points to Cover in Your Answer:

**What Lambda alone cannot do:**
- Lambda has a **15-minute timeout** — cannot orchestrate long workflows
- Lambda has no built-in **retry with exponential backoff** for sequences of steps
- Lambda has no **visual workflow** or audit trail of step execution
- Lambda has no built-in **human approval gates**
- Orchestrating multiple Lambdas from one Lambda = complex, fragile code

**Step Functions solves this with:**
- Visual state machine definition (JSON/YAML)
- Built-in retry and error handling per step
- Human approval via `taskToken` pattern
- Full execution history and audit trail
- Max execution: 1 year (Standard), 5 minutes (Express)

**Standard vs Express:**

| Feature | Standard | Express |
|---|---|---|
| Duration | Up to 1 year | Up to 5 minutes |
| Execution rate | 2,000/second | 100,000/second |
| Pricing | Per state transition | Per execution duration |
| Exactly-once | ✅ Yes | ❌ At-least-once |
| Use case | Long workflows, human approval | High-volume event processing |

**Multi-stage deployment pipeline:**
```json
{
  "Comment": "Deployment Pipeline with Approval",
  "StartAt": "RunTests",
  "States": {
    "RunTests": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:::function:run-tests",
      "Retry": [{"ErrorEquals": ["States.ALL"], "MaxAttempts": 2}],
      "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "NotifyFailure"}],
      "Next": "DeployToStaging"
    },
    "DeployToStaging": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:::function:deploy-staging",
      "Next": "WaitForApproval"
    },
    "WaitForApproval": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
      "Parameters": {
        "FunctionName": "send-approval-email",
        "Payload": {"taskToken.$": "$$.Task.Token"}
      },
      "HeartbeatSeconds": 86400,
      "Next": "DeployToProduction"
    },
    "DeployToProduction": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:::function:deploy-prod",
      "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "RollbackProduction"}],
      "Next": "NotifySuccess"
    },
    "RollbackProduction": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:::function:rollback-prod",
      "Next": "NotifyFailure"
    },
    "NotifySuccess": {"Type": "Task", "Resource": "arn:aws:lambda:::function:notify-slack", "End": true},
    "NotifyFailure": {"Type": "Task", "Resource": "arn:aws:lambda:::function:notify-pagerduty", "End": true}
  }
}
```

> 💡 **Interview tip:** The **`waitForTaskToken` pattern** is the key to human approval gates in Step Functions. The workflow pauses indefinitely until a Lambda sends back the task token via `SendTaskSuccess` or `SendTaskFailure` API. An approver clicks "Approve" in a web UI → Lambda calls `SendTaskSuccess` → workflow resumes. This is a very elegant pattern that impresses interviewers.

---

### Q189 — Jenkins | Conceptual | Advanced

> Explain **Jenkins Pipeline `when` directive** and **`input` step** in detail.
>
> Write a Jenkinsfile that:
> - Runs unit tests on every branch
> - Runs integration tests **only on `main` and `release/*` branches**
> - Requires **manual approval with a timeout of 30 minutes** before deploying to production
> - If approval times out, automatically **aborts the deployment** and sends a Slack notification

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — when directive, input step, pipeline conditions sections

#### Key Points to Cover in Your Answer:

```groovy
pipeline {
    agent any

    environment {
        SLACK_WEBHOOK = credentials('slack-webhook')
    }

    stages {
        // Stage 1: Runs on EVERY branch
        stage('Unit Tests') {
            steps {
                sh 'npm ci && npm test'
            }
            post {
                always {
                    junit '**/test-results/*.xml'
                }
            }
        }

        // Stage 2: Only on main and release/* branches
        stage('Integration Tests') {
            when {
                anyOf {
                    branch 'main'
                    branch pattern: 'release/.*', comparator: 'REGEXP'
                }
            }
            steps {
                sh 'npm run test:integration'
            }
        }

        // Stage 3: Build Docker image (main and release only)
        stage('Build & Push') {
            when {
                anyOf {
                    branch 'main'
                    branch pattern: 'release/.*', comparator: 'REGEXP'
                }
            }
            steps {
                sh 'docker build -t my-app:${GIT_COMMIT} .'
                sh 'docker push my-app:${GIT_COMMIT}'
            }
        }

        // Stage 4: Manual approval with timeout
        stage('Approval') {
            when {
                branch 'main'
            }
            steps {
                script {
                    try {
                        timeout(time: 30, unit: 'MINUTES') {
                            input(
                                message: 'Deploy to Production?',
                                ok: 'Deploy',
                                submitter: 'senior-devops,tech-lead',
                                parameters: [
                                    string(
                                        name: 'REASON',
                                        description: 'Reason for deployment'
                                    )
                                ]
                            )
                        }
                    } catch (org.jenkinsci.plugins.workflow.steps.FlowInterruptedException e) {
                        // Timeout or abort
                        slackSend(
                            webhookSecretTokenId: 'slack-webhook',
                            color: 'warning',
                            message: "⏱️ Deployment approval TIMED OUT for build #${BUILD_NUMBER}. Deployment aborted."
                        )
                        currentBuild.result = 'ABORTED'
                        error('Approval timed out after 30 minutes')
                    }
                }
            }
        }

        // Stage 5: Deploy to Production
        stage('Deploy Production') {
            when {
                branch 'main'
            }
            steps {
                sh 'kubectl set image deployment/my-app my-app=my-app:${GIT_COMMIT}'
                sh 'kubectl rollout status deployment/my-app --timeout=300s'
            }
        }
    }

    post {
        success {
            slackSend(color: 'good',
                message: "✅ Pipeline succeeded: ${JOB_NAME} #${BUILD_NUMBER}")
        }
        failure {
            slackSend(color: 'danger',
                message: "❌ Pipeline FAILED: ${JOB_NAME} #${BUILD_NUMBER}")
        }
    }
}
```

**`when` directive conditions:**
```groovy
when { branch 'main' }                           // exact branch
when { branch pattern: 'release/.*', comparator: 'REGEXP' }  // regex
when { anyOf { branch 'main'; branch 'develop' } }            // OR
when { allOf { branch 'main'; environment name: 'DEPLOY', value: 'true' } } // AND
when { not { branch 'main' } }                   // NOT
when { changeRequest() }                         // only on PR/change request
when { tag "v*" }                                // only on tags
```

> 💡 **Interview tip:** The `FlowInterruptedException` catch is critical — without it, `timeout()` around `input` will fail the build rather than gracefully aborting it. Always catch this exception and set `currentBuild.result = 'ABORTED'` to distinguish timeouts from actual failures in build history.

---

### Q190 — ELK Stack | Troubleshooting | Advanced

> Your **Kibana** dashboard is showing `No results found` for a time range where you know logs exist in Elasticsearch. Direct Elasticsearch queries confirm the data is there.
>
> What are all the possible reasons Kibana would show no results despite data existing, and how do you fix each one?

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Kibana index patterns, time fields, mapping sections

#### Key Points to Cover in Your Answer:

**Cause 1 — Wrong time field in index pattern:**
```
Kibana uses a specific field for time filtering
If index pattern uses @timestamp but logs use event.created → no results

Fix: Stack Management → Index Patterns → edit → change time field
```

**Cause 2 — Time field has wrong mapping type:**
```
@timestamp stored as keyword (string) instead of date
Kibana time filter cannot work with keyword fields

Fix: Check mapping:
GET /app-logs-*/_mapping
→ @timestamp must be type: date

If wrong: reindex with correct mapping or use ingest pipeline to cast
```

**Cause 3 — Kibana time zone mismatch:**
```
Logs in UTC, Kibana showing in UTC+5:30
Time range 10:00-11:00 in Kibana = 04:30-05:30 UTC → no logs

Fix: Kibana → Management → Advanced Settings → dateFormat:tz → Browser or UTC
```

**Cause 4 — Index pattern not matching the indices:**
```
Index pattern: app-logs-*
Actual indices: application-logs-2024-01-15

Fix: Stack Management → Index Patterns → delete and recreate with correct pattern
Check: GET /_cat/indices?v | grep app
```

**Cause 5 — Kibana index pattern cache is stale:**
```
New index created but Kibana doesn't know about it

Fix: Stack Management → Index Patterns → refresh field list
```

**Cause 6 — KQL/Lucene query filtering out results:**
```
A saved search filter is excluding the data
Dashboard has a global filter active

Fix: Check filter bar in Kibana → look for active filters (blue chips)
Click X to remove all filters and test
```

**Cause 7 — Alias pointing to wrong index:**
```
Index alias used in index pattern
Alias pointing to old index, new data in new index

Fix:
GET /_alias/app-logs
→ Verify alias points to correct indices
```

> 💡 **Interview tip:** The **most common cause** in production is the time zone mismatch or wrong time field — always check these two first. A quick test: in Kibana Discover, change the time range to "Last 15 years" — if logs appear then it is a time-related issue. If still empty, it is a query/filter or index pattern issue.

---

### Q191 — Kubernetes | Scenario-Based | Advanced

> You are implementing **Kubernetes cluster upgrades** in production. The cluster has:
> - 50 worker nodes
> - 500 running pods
> - Zero-downtime requirement
> - Mixed workloads (stateless and stateful)
>
> Walk me through your **complete upgrade strategy** from Kubernetes 1.28 to 1.30 — including control plane, worker nodes, add-ons, and validation at each step.

📁 **Reference:** `nawab312/Kubernetes` → `09_CLUSTER_OPERATIONS.md` — cluster upgrade, rolling upgrade sections

#### Key Points to Cover in Your Answer:

**Pre-upgrade preparation (1-2 weeks before):**
```bash
# 1. Read release notes for 1.29 AND 1.30 (must upgrade one minor version at a time)
# Check for: API deprecations, removed features, breaking changes

# 2. Check deprecated API usage in cluster
kubectl api-versions
kubectl get --raw /apis | jq '.groups[].preferredVersion'

# Use kubent (kube no trouble) to find deprecated APIs
kubent

# 3. Verify etcd health
ETCDCTL_API=3 etcdctl endpoint health

# 4. Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/pre-upgrade-snapshot.db

# 5. Check PodDisruptionBudgets
kubectl get pdb --all-namespaces
# Ensure PDBs don't block node draining

# 6. Identify StatefulSets — need special care during upgrade
kubectl get statefulsets --all-namespaces
```

**Phase 1: Upgrade Control Plane (1.28 → 1.29):**
```bash
# For EKS:
aws eks update-cluster-version --name my-cluster --kubernetes-version 1.29

# Wait for control plane upgrade
aws eks wait cluster-active --name my-cluster

# Verify control plane
kubectl version
kubectl get nodes  # worker nodes still on 1.28 — this is OK (skew policy)
```

**Phase 2: Upgrade Add-ons (after control plane):**
```bash
# Upgrade in order: CoreDNS, kube-proxy, VPC CNI, then others
aws eks update-addon --cluster-name my-cluster --addon-name coredns --addon-version v1.10.1
aws eks update-addon --cluster-name my-cluster --addon-name kube-proxy --addon-version v1.29.0
aws eks update-addon --cluster-name my-cluster --addon-name vpc-cni --addon-version v1.16.0
```

**Phase 3: Upgrade Worker Nodes (rolling — 5 nodes at a time):**
```bash
# For each batch of nodes:
# Step 1: Cordon the node (no new pods scheduled)
kubectl cordon <node-name>

# Step 2: Drain the node (evict pods gracefully)
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=120 \
  --timeout=300s

# Step 3: Upgrade node (EKS: replace node group AMI)
# For managed node groups:
aws eks update-nodegroup-version --cluster-name my-cluster \
  --nodegroup-name workers --kubernetes-version 1.29

# Step 4: Verify new node is Ready before proceeding
kubectl get nodes | grep <new-node>

# Step 5: Repeat for next batch
```

**Phase 4: 1.29 → 1.30 (same process):**
- Kubernetes only supports upgrading ONE minor version at a time
- Must complete 1.28→1.29 fully before starting 1.29→1.30

**Validation at each step:**
```bash
kubectl get nodes  # all nodes Ready, correct version
kubectl get pods --all-namespaces | grep -v Running  # no stuck pods
kubectl top nodes  # resource usage normal
# Run smoke tests against application
```

> 💡 **Interview tip:** The two things interviewers most want to hear are: (1) **one minor version at a time** — you cannot jump from 1.28 to 1.30 directly, and (2) **always backup etcd before upgrading** — if the control plane upgrade fails, you can restore from the snapshot. Also mention checking `kubent` for deprecated API usage before upgrading — deploying manifests with deprecated APIs after upgrade can break CI/CD pipelines.

---

### Q192 — Linux / Bash | Conceptual | Advanced

> Explain **Linux memory management** concepts:
> - What is the difference between **virtual memory** and **physical memory**?
> - What are **huge pages** and when would you use them?
> - What is **swap** and what does high swap usage indicate?
> - Explain **memory overcommit** — what are the 3 overcommit modes in Linux?
> - What is **page cache** and how does it affect `free -h` output?

📁 **Reference:** `nawab312/DSA` → `Linux` — memory management, `/proc/meminfo`, vm settings sections

#### Key Points to Cover in Your Answer:

**Virtual vs Physical memory:**
```
Physical memory: actual RAM chips (e.g., 16 GB)
Virtual memory: address space each process sees (can be larger than physical RAM)
  → Each process thinks it has its own large memory space
  → Kernel manages mapping of virtual → physical pages via page table
  → Processes can allocate more virtual memory than physical RAM exists
     (most allocated memory is never actually used — lazy allocation)
```

**Huge Pages:**
```
Default page size: 4KB
Huge pages: 2MB or 1GB pages

Why use them?
- Less TLB (Translation Lookaside Buffer) pressure
- Fewer page table entries for large memory regions
- Reduces CPU overhead for memory-intensive apps

When to use:
- Databases (PostgreSQL, MySQL, Oracle) — suggest enabling hugepages
- JVM applications with large heap
- In-memory caches (Redis with large datasets)

Configure:
echo 1024 > /proc/sys/vm/nr_hugepages  # 1024 × 2MB = 2GB huge pages
```

**Swap:**
```
Swap: disk space used as overflow when RAM is exhausted
High swap usage indicates:
  1. RAM is genuinely exhausted → need more RAM or optimize apps
  2. OOM killer may fire soon
  3. Performance degradation (disk I/O much slower than RAM)
  4. Memory leaks in application

vm.swappiness (0-100):
  0 = avoid swap as long as possible (recommended for most servers)
  10 = swap only when RAM very low (recommended for databases)
  60 = default Linux value
  100 = aggressive swapping
```

**Memory Overcommit:**
```
Linux allows allocating more memory than physically available

3 modes (vm.overcommit_memory):
0 = HEURISTIC (default) — kernel estimates safe allocation, allows moderate overcommit
1 = ALWAYS — allow ALL allocations regardless of available memory
    → Used in Java apps that allocate large virtual spaces
    → Risk: OOM killer fires when memory actually needed
2 = NEVER — only allow allocation if: swap + (RAM × vm.overcommit_ratio / 100)
    → Conservative, reduces OOM kill risk
    → May cause malloc() to fail → application crashes gracefully

Check: cat /proc/sys/vm/overcommit_memory
Set:   echo 1 > /proc/sys/vm/overcommit_memory
```

**Page Cache and `free -h`:**
```bash
free -h
#               total    used    free    shared  buff/cache  available
# Mem:           15Gi    8.2Gi   1.1Gi   512Mi   6.1Gi       6.4Gi

# "available" is what matters — not "free"
# available = free + buff/cache that can be reclaimed
# buff/cache = Linux using "spare" RAM to cache disk files (normal!)
# High buff/cache = good! Linux is using RAM efficiently
# Low "available" = real memory pressure → check for leaks

# Clear page cache (for testing only — never in production normally):
echo 3 > /proc/sys/vm/drop_caches
```

> 💡 **Interview tip:** The most common misconception interviewers test is people panicking about high `buff/cache` in `free -h` output. Explain that this is **normal and beneficial** — Linux uses spare RAM as file system cache to speed up I/O. The number that actually matters is `available`. Only worry when `available` approaches zero.

---

### Q193 — Terraform | Troubleshooting | Advanced

> A colleague ran `terraform apply` and it succeeded, but now `terraform plan` shows the same resources as **changed again** even though nothing in the `.tf` files changed.
>
> What causes **perpetual drift** in Terraform where plan always shows changes even after apply? List at least 5 root causes and the fix for each.

📁 **Reference:** `nawab312/Terraform` — state drift, `ignore_changes`, perpetual diff sections

#### Key Points to Cover in Your Answer:

**Root Cause 1 — AWS adds default values not in your config:**
```hcl
# Problem: AWS adds default tags, encryption settings, etc.
# Plan shows: ~ tags = {} → tags = {aws:cloudformation:stack-name = "auto-added"}

# Fix: Use ignore_changes
resource "aws_security_group" "web" {
  lifecycle {
    ignore_changes = [tags["aws:cloudformation:stack-name"]]
  }
}
```

**Root Cause 2 — Non-deterministic ordering:**
```hcl
# Problem: Security group ingress rules show as changed due to ordering
# AWS returns rules in different order each time

# Fix: Use ignore_changes for the entire block
lifecycle {
  ignore_changes = [ingress, egress]
}
# OR restructure using aws_security_group_rule resources (ordered)
```

**Root Cause 3 — Computed values not stored in state:**
```hcl
# Problem: Attribute computed by AWS (like ARN, ID) changes on each read
# Some providers re-compute values differently

# Fix: Check provider version — often fixed in newer versions
# Workaround: Store computed value in a data source instead
```

**Root Cause 4 — Provider bug or version mismatch:**
```hcl
# Problem: Provider serializes/deserializes differently between versions
# e.g., JSON policy formatting differs between AWS provider 4.x and 5.x

# Fix:
terraform providers lock  # lock provider versions
terraform init -upgrade   # upgrade to version with fix
```

**Root Cause 5 — Encoded values (base64, JSON) formatted differently:**
```hcl
# Problem: JSON policy in state vs fresh read has different whitespace
resource "aws_iam_role_policy" "example" {
  policy = jsonencode({...})  # terraform normalizes JSON
}
# AWS returns policy with different formatting → perpetual diff

# Fix: Use jsonencode() consistently — Terraform normalizes it
# Avoid: file("policy.json") — preserves formatting exactly
```

**Root Cause 6 — Ignore case sensitivity:**
```hcl
# Problem: Terraform stores "us-east-1" but AWS returns "US-EAST-1"
# Fix: Check provider version, use lowercase consistently
```

**Root Cause 7 — External tool modifying the resource:**
```hcl
# Problem: Another tool (Ansible, console) modifies the resource
# Terraform sees the change and wants to revert it

# Fix: Use ignore_changes for fields managed by other tools
lifecycle {
  ignore_changes = [user_data, tags["last-modified-by"]]
}
```

> 💡 **Interview tip:** Always start the diagnosis with `terraform plan -out=tfplan && terraform show -json tfplan | jq '.resource_changes'` — this shows the exact field causing the perpetual diff. Once you know the specific field, the fix is almost always either `ignore_changes` or upgrading the provider version. Mention that `ignore_changes = all` is a last resort that hides all drift — use it only for resources you intentionally manage outside Terraform.

---

### Q194 — GitHub Actions | Troubleshooting | Advanced

> Your GitHub Actions workflow is **consuming GitHub Actions minutes extremely fast**. A workflow that should take 5 minutes is taking 45 minutes and running on every commit.
>
> Walk me through how you would **audit and optimize** the workflow — covering: unnecessary triggers, missing caching, redundant steps, parallelization opportunities, and job concurrency settings.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — workflow optimization, caching, concurrency sections

#### Key Points to Cover in Your Answer:

**Step 1 — Audit triggers (most impactful fix):**
```yaml
# BAD — triggers on every push to every branch
on: push

# GOOD — only trigger when relevant files change
on:
  push:
    branches: [main, develop]
    paths:
      - 'src/**'
      - 'package*.json'
      - '.github/workflows/**'
  pull_request:
    branches: [main]
    paths:
      - 'src/**'
```

**Step 2 — Add dependency caching:**
```yaml
# Node.js caching
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'          # caches node_modules automatically

# Manual cache for other tools
- uses: actions/cache@v4
  with:
    path: |
      ~/.cache/pip
      ~/.gradle/caches
    key: ${{ runner.os }}-deps-${{ hashFiles('**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-deps-
```

**Step 3 — Parallelize independent jobs:**
```yaml
# BAD — sequential
jobs:
  lint:     ...
  test:     needs: lint    # test waits for lint unnecessarily
  build:    needs: test

# GOOD — parallel where possible
jobs:
  lint:     ...    # runs in parallel
  test:     ...    # runs in parallel
  build:
    needs: [lint, test]    # waits for both, then builds
```

**Step 4 — Cancel redundant workflow runs:**
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # cancel old run when new commit pushed
```

**Step 5 — Skip unchanged components:**
```yaml
- uses: dorny/paths-filter@v3
  id: changes
  with:
    filters: |
      backend:
        - 'backend/**'
      frontend:
        - 'frontend/**'

- name: Build backend
  if: steps.changes.outputs.backend == 'true'
  run: cd backend && npm run build

- name: Build frontend
  if: steps.changes.outputs.frontend == 'true'
  run: cd frontend && npm run build
```

**Step 6 — Optimize Docker builds:**
```yaml
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha          # use GitHub Actions cache
    cache-to: type=gha,mode=max
    context: .
```

**Step 7 — Use `timeout-minutes` to catch hanging steps:**
```yaml
jobs:
  build:
    timeout-minutes: 15    # kill job if it runs more than 15 min
    steps:
      - name: Run tests
        timeout-minutes: 10
        run: npm test
```

> 💡 **Interview tip:** The **biggest wins** in order are: (1) fix triggers with `paths` filter — most workflows run unnecessarily on doc/config changes, (2) add caching — `npm ci` downloading 500MB every run is the most common time waster, (3) add `cancel-in-progress: true` — without this, 10 quick commits create 10 parallel workflows all competing for runners. These three changes alone often cut workflow time by 70%.

---

### Q195 — AWS | Troubleshooting | Advanced

> Your **AWS ALB** is returning `504 Gateway Timeout` errors for about 5% of requests. The backend EC2 instances appear healthy in the target group.
>
> Walk me through the **complete troubleshooting process** for ALB 504 errors — what metrics to check, what logs to look at, and what are all the possible root causes?

📁 **Reference:** `nawab312/AWS` — ALB, access logs, target group health, idle timeout sections

#### Key Points to Cover in Your Answer:

**Understanding ALB 504:**
- ALB waits for backend to respond
- If backend takes longer than **ALB idle timeout** (default 60 seconds) → 504
- Backend IS receiving the request — it is just too slow to respond

**Step 1 — Enable and check ALB access logs:**
```bash
# Enable access logs to S3
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn <alb-arn> \
  --attributes Key=access_logs.s3.enabled,Value=true \
               Key=access_logs.s3.bucket,Value=my-alb-logs

# ALB access log fields relevant to 504:
# elb_status_code: 504 (ALB gave up)
# target_status_code: - (backend never responded) OR 504
# request_processing_time: time ALB spent processing
# target_processing_time: time backend took to respond ← THIS is the problem
# response_processing_time: time ALB spent sending response
```

**Step 2 — Check CloudWatch metrics:**
```bash
# TargetResponseTime p99 — is backend slow?
# HTTPCode_Target_5XX_Count — is backend erroring?
# HealthyHostCount — are all targets healthy?
# RequestCountPerTarget — is one target overloaded?
# TargetConnectionErrorCount — can ALB connect to backend?
```

**Possible root causes:**

| Root Cause | Evidence | Fix |
|---|---|---|
| Backend too slow (genuine timeout) | TargetResponseTime > 60s | Optimize backend or increase ALB idle timeout |
| ALB idle timeout too low | target_processing_time close to 60s | Increase: `idle_timeout.timeout_seconds` |
| Backend thread pool exhausted | Consistent slow responses under load | Scale backend, increase thread pool |
| Long-running database queries | Backend waiting on DB | Query optimization, read replicas |
| Backend out of memory / GC pause | JVM GC logs show long pauses | Tune JVM, increase memory |
| Unhealthy targets still receiving traffic | Some targets failing health checks | Fix unhealthy targets |
| Keep-alive mismatch | ALB closes connection before backend responds | Ensure backend keep-alive > ALB idle timeout |

**Fix for most common cause (backend slow):**
```bash
# Increase ALB idle timeout (max 4000 seconds)
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn <alb-arn> \
  --attributes Key=idle_timeout.timeout_seconds,Value=120

# Also ensure application server keep-alive > ALB idle timeout
# Nginx: keepalive_timeout 120s;
# Apache: KeepAliveTimeout 120
```

> 💡 **Interview tip:** The **keep-alive mismatch** is a subtle but common cause that many engineers miss. If your application server's keep-alive timeout is LESS than the ALB idle timeout, the backend closes the connection while ALB thinks it is still alive → ALB sends next request on dead connection → 502/504 error. Always set application server keep-alive slightly HIGHER than ALB idle timeout to prevent this.

---

### Q196 — Prometheus | Scenario-Based | Advanced

> You need to monitor **PostgreSQL** running on Kubernetes using Prometheus. Set up monitoring that covers:
> - Active connections vs max connections
> - Query execution time (slow queries)
> - Replication lag (for replicas)
> - Transaction rate and deadlocks
> - Database size growth
>
> Write the complete setup including exporter deployment, ServiceMonitor, recording rules, and alerting rules.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — postgres_exporter, database monitoring sections

#### Key Points to Cover in Your Answer:

**1. Deploy postgres_exporter:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-exporter
  namespace: production
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: postgres-exporter
        image: prometheuscommunity/postgres-exporter:v0.15.0
        env:
        - name: DATA_SOURCE_NAME
          valueFrom:
            secretKeyRef:
              name: postgres-exporter-secret
              key: dsn
          # dsn format: postgresql://user:password@host:5432/dbname?sslmode=disable
        ports:
        - containerPort: 9187
          name: metrics
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-exporter
  labels:
    app: postgres-exporter
spec:
  ports:
  - name: metrics
    port: 9187
```

**2. ServiceMonitor:**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: postgres-exporter
spec:
  selector:
    matchLabels:
      app: postgres-exporter
  endpoints:
  - port: metrics
    interval: 30s
    scrapeTimeout: 20s
```

**3. Key metrics and Recording Rules:**
```yaml
groups:
- name: postgres_recording
  rules:
  # Connection utilization %
  - record: pg:connection_utilization
    expr: |
      pg_stat_activity_count / pg_settings_max_connections * 100

  # Transaction rate per second
  - record: pg:transaction_rate:rate5m
    expr: |
      sum(rate(pg_stat_database_xact_commit[5m]) +
          rate(pg_stat_database_xact_rollback[5m]))

  # Replication lag in seconds
  - record: pg:replication_lag_seconds
    expr: |
      pg_replication_lag
```

**4. Alerting Rules:**
```yaml
groups:
- name: postgres_alerts
  rules:
  # High connection utilization
  - alert: PostgreSQLHighConnections
    expr: pg:connection_utilization > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "PostgreSQL connections at {{ $value }}% of max"

  # Replication lag > 30 seconds
  - alert: PostgreSQLReplicationLag
    expr: pg:replication_lag_seconds > 30
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "PostgreSQL replica lagging by {{ $value }}s"

  # Deadlocks increasing
  - alert: PostgreSQLDeadlocks
    expr: rate(pg_stat_database_deadlocks[5m]) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "PostgreSQL deadlocks detected"

  # No active connections (service might be down)
  - alert: PostgreSQLNoConnections
    expr: absent(pg_stat_activity_count)
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "PostgreSQL exporter unreachable"
```

> 💡 **Interview tip:** The **connection utilization alert** is the most important PostgreSQL metric to monitor. PostgreSQL has a hard limit on connections (`max_connections`, default 100). When connections are exhausted, new clients get `FATAL: sorry, too many clients already`. Alerting at 80% gives you time to investigate before hitting 100%. The long-term fix is **PgBouncer** (connection pooler) or **RDS Proxy** — always mention this as the architectural solution.

---

### Q197 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `topology spread constraints`**. What problem do they solve that `podAntiAffinity` cannot solve efficiently at scale?
>
> Write a topology spread constraint that ensures pods are spread evenly across:
> - Availability zones (max skew of 1)
> - Nodes within each AZ (max skew of 2)
>
> What happens when the constraint cannot be satisfied?

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — topology spread constraints sections

#### Key Points to Cover in Your Answer:

**Problem with podAntiAffinity at scale:**
```yaml
# podAntiAffinity approach — O(n²) performance problem
affinity:
  podAntiAffinity:
    preferredDuringScheduling...:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels: {app: myapp}
        topologyKey: topology.kubernetes.io/zone

# Problem:
# For 100 pods → scheduler checks 100×100 = 10,000 combinations
# For 1000 pods → 1,000,000 combinations
# Performance degrades badly at scale
# Also: only hard/soft — cannot express "max 1 more than minimum"
```

**Topology Spread Constraints — O(n) performance:**
```yaml
spec:
  topologySpreadConstraints:

  # Constraint 1: Spread across AZs (max 1 skew)
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule   # hard constraint
    labelSelector:
      matchLabels:
        app: my-app

  # Constraint 2: Spread across nodes within AZ (max 2 skew)
  - maxSkew: 2
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway  # soft constraint
    labelSelector:
      matchLabels:
        app: my-app
```

**MaxSkew explained:**
```
3 AZs: us-east-1a (3 pods), us-east-1b (3 pods), us-east-1c (2 pods)
Skew = max(3,3,2) - min(3,3,2) = 3 - 2 = 1 ✅ (maxSkew: 1 satisfied)

If us-east-1a has 4, others have 2:
Skew = 4 - 2 = 2 ❌ (violates maxSkew: 1)
→ Scheduler will not place next pod in us-east-1a
```

**`whenUnsatisfiable` options:**
| Option | Behavior |
|---|---|
| `DoNotSchedule` | Pod stays Pending if constraint cannot be met |
| `ScheduleAnyway` | Schedule anyway but prefer satisfying constraint |

**What happens when constraint cannot be satisfied:**
- `DoNotSchedule` → Pod goes `Pending` with event: `0/10 nodes are available: topology constraint not satisfied`
- Common cause: not enough nodes in all topology zones
- Fix: add nodes, or change `whenUnsatisfiable: ScheduleAnyway`

> 💡 **Interview tip:** The key advantage of topology spread constraints over podAntiAffinity is **proportional scaling** — if you have 3 AZs and scale from 3 to 30 pods, the constraints automatically ensure 10 pods per AZ. With podAntiAffinity you could only guarantee "not on same node" — not "evenly distributed across zones". This matters a lot for availability during AZ failures.

---

### Q198 — ArgoCD | Conceptual | Advanced

> Explain **ArgoCD `syncWaves`** and **`syncPhases`**. What problem do they solve?
>
> Give a real-world example where you have these resources that must be applied in a specific order:
> - Namespace and RBAC (must be first)
> - ConfigMaps and Secrets (must be second)
> - Database migration Job (must complete before app starts)
> - Application Deployment (must be last)
>
> Write the ArgoCD annotations to enforce this order.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — sync waves, sync phases, resource ordering sections

#### Key Points to Cover in Your Answer:

**Problem they solve:**
- By default ArgoCD applies all resources simultaneously
- Some resources MUST exist before others (e.g., Namespace before Deployment)
- Database migrations MUST complete before app starts
- Without ordering → race conditions → deployments fail randomly

**SyncWaves concept:**
- Resources with lower wave number applied FIRST
- ArgoCD waits for all resources in wave N to be healthy before starting wave N+1
- Default wave = 0 if not specified
- Negative waves (-1, -2) run before wave 0

**Annotations for ordered deployment:**

```yaml
# Wave -1: Namespace and RBAC (must exist before everything)
apiVersion: v1
kind: Namespace
metadata:
  name: production
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
---

# Wave 0: ConfigMaps and Secrets (before app needs them)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  annotations:
    argocd.argoproj.io/sync-wave: "0"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  annotations:
    argocd.argoproj.io/sync-wave: "0"
---

# Wave 1: Database migration Job (MUST complete before app)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/sync-wave: "1"
    argocd.argoproj.io/hook: PreSync          # runs before main sync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: my-app:latest
        command: ["./migrate.sh"]
---

# Wave 2: Application Deployment (last — after everything is ready)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  ...
---
# Wave 2: Service (same wave as deployment is fine)
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

**SyncPhases:**
- `PreSync` → runs before main sync (hooks like migrations)
- `Sync` → main resource application (all waves happen here)
- `PostSync` → runs after sync completes (smoke tests, notifications)
- `SyncFail` → runs if sync fails (cleanup, rollback notification)

> 💡 **Interview tip:** The combination of **sync waves + PreSync hooks** for database migrations is a very robust pattern. The PreSync hook runs the migration Job before ANY resources are applied. If the migration fails, the sync is aborted — the old app version keeps running with the old schema. This prevents the catastrophic scenario of new code deployed with incompatible database schema.

---

### Q199 — Linux / Bash | Troubleshooting | Advanced

> A production server is experiencing **network packet loss**. Users are reporting intermittent connection drops. `ping` to the server shows 15% packet loss.
>
> Walk me through the **complete network troubleshooting process** — which commands you would use, what you are looking for at each layer (physical, network, transport), and what the most common causes of packet loss are on Linux.

📁 **Reference:** `nawab312/DSA` → `Linux` — network troubleshooting, tc, ethtool, netstat sections

#### Key Points to Cover in Your Answer:

**Layer 1 — Physical/Driver (check hardware first):**
```bash
# Check NIC errors and dropped packets
ethtool -S eth0 | grep -E "drop|error|miss|overflow"
# rx_dropped: packets dropped at NIC ring buffer (NIC overloaded)
# tx_errors: transmit errors (cable/switch issues)

# Check NIC link status
ethtool eth0 | grep -E "Speed|Duplex|Link"
# Half-duplex + auto-negotiation failure = packet loss

# Check dmesg for NIC errors
dmesg | grep -E "eth0|eno|ens" | grep -i "error\|reset\|timeout"
```

**Layer 2 — Network Interface:**
```bash
# Check interface statistics
ip -s link show eth0
# RX/TX errors, drops, overruns

# Check if NIC ring buffer is too small
ethtool -g eth0
# Pre-set maximums: RX 4096 / TX 4096
# Current: RX 256 / TX 256  ← might be too small for high traffic

# Increase ring buffer
ethtool -G eth0 rx 4096 tx 4096
```

**Layer 3 — Network (routing/firewall):**
```bash
# Check iptables drop rules
iptables -L -n -v | grep -v "0     0"  # show non-zero counters

# Check conntrack table (connection tracking)
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
# If count = max → new connections dropped
# Fix: echo 524288 > /proc/sys/net/netfilter/nf_conntrack_max

# Check route table
ip route show
traceroute <destination>
mtr <destination>  # continuous traceroute showing per-hop packet loss
```

**Layer 4 — Transport (TCP/UDP):**
```bash
# Check socket buffer overflows
ss -s
netstat -s | grep -E "failed|overflow|drop|reject"

# Check kernel network statistics
cat /proc/net/softnet_stat
# Column 3 (throttled) being non-zero = CPU cannot keep up with network
# Fix: increase net.core.netdev_max_backlog

# Check for SYN flood (connection table full)
netstat -n | awk '{print $6}' | sort | uniq -c | sort -rn
# Many SYN_RECV = possible SYN flood or slow backend
```

**Common causes and fixes:**
| Cause | Evidence | Fix |
|---|---|---|
| NIC ring buffer too small | `ethtool -S` shows rx_dropped | `ethtool -G eth0 rx 4096` |
| conntrack table full | count = max_connections | Increase `nf_conntrack_max` |
| CPU too slow (softirq) | softnet_stat column 3 | Enable RSS/RPS, more CPUs |
| Network congestion | `mtr` shows packet loss at specific hop | ISP issue or bandwidth saturation |
| Half-duplex mismatch | `ethtool` shows Half-duplex | Force full-duplex on both ends |
| iptables dropping | High DROP counter in iptables | Review firewall rules |

> 💡 **Interview tip:** Start with **`mtr` (My Traceroute)** — it shows both traceroute path AND packet loss per hop continuously. If loss starts at hop 3 and persists → problem is at hop 3 or beyond. If loss is only at the destination → problem is on the server itself. This immediately narrows down whether it is a server-side or network path issue.

---

### Q200 — AWS | Scenario-Based | Advanced

> Your company is implementing **AWS Cost Anomaly Detection**. You receive an alert:
>
> `Cost anomaly detected: $15,000 unexpected spend in the last 24 hours on EC2`
>
> Walk me through how you would **investigate the cost spike**, identify the root cause, and implement controls to **prevent runaway costs** in future.

📁 **Reference:** `nawab312/AWS` — Cost Explorer, Cost Anomaly Detection, billing alerts, SCPs sections

#### Key Points to Cover in Your Answer:

**Immediate Investigation:**
```bash
# Step 1: Cost Explorer → EC2 → filter by last 24 hours → group by instance type
# Look for: new instance types you don't normally use (p3, g4, large counts)

# Step 2: Filter by region → is the spike in an unexpected region?
# Often: developer accidentally launched instances in wrong region

# Step 3: Filter by account (if multi-account) → which account?
# Step 4: Filter by tags → which team/service owns these instances?

# Step 5: EC2 console → Running Instances → sort by Launch Time
# Find instances launched in last 24 hours
aws ec2 describe-instances \
  --filter "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[?LaunchTime>=`2024-01-15T00:00:00`].[InstanceId,InstanceType,LaunchTime,Tags]' \
  --output table
```

**Common root causes:**
1. **Forgotten development instances** — developer launched GPU instances for testing
2. **Autoscaling gone wrong** — ASG scaling policy misconfigured, scaled to max (1000 instances)
3. **Crypto mining** — compromised credentials used to launch GPU instances
4. **Data transfer charges** — spike in cross-region or internet data transfer
5. **Spot instance switch to On-Demand** — spot capacity unavailable, application fell back to On-Demand

**Immediate action if crypto mining suspected:**
```bash
# Check CloudTrail for unauthorized RunInstances
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --start-time $(date -d '24 hours ago' -u +%Y-%m-%dT%H:%M:%SZ) \
  --query 'Events[*].[EventTime,Username,CloudTrailEvent]'

# Terminate all suspicious instances
aws ec2 terminate-instances --instance-ids <instance-ids>
```

**Prevention controls:**
```json
// SCP: Limit allowed instance types (no GPU instances in non-ML accounts)
{
  "Effect": "Deny",
  "Action": "ec2:RunInstances",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringNotLike": {
      "ec2:InstanceType": ["t3.*", "t4g.*", "m5.*", "c5.*"]
    }
  }
}

// SCP: Prevent launching instances in unapproved regions
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
    }
  }
}
```

**Budget alerts:**
```bash
aws budgets create-budget \
  --account-id <account-id> \
  --budget '{"BudgetName":"EC2-Daily","BudgetLimit":{"Amount":"1000","Unit":"USD"},"TimeUnit":"DAILY","BudgetType":"COST"}' \
  --notifications-with-subscribers '[{"Notification":{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80},"Subscribers":[{"SubscriptionType":"EMAIL","Address":"ops@company.com"}]}]'
```

> 💡 **Interview tip:** Always mention **checking CloudTrail for `RunInstances` events from unexpected IAM users/roles** when investigating cost spikes — this immediately tells you whether it is an operational mistake vs a security breach. Crypto mining via compromised credentials is one of the most common AWS security incidents and it always shows up as unexpected EC2 spend (usually GPU instances like p3.2xlarge).

---

### Q201 — Kubernetes | Troubleshooting | Advanced

> Your **Kubernetes DNS resolution is broken** — pods cannot resolve service names. `nslookup kubernetes.default` from a pod fails, but pods can communicate via IP addresses.
>
> Walk me through diagnosing and fixing Kubernetes DNS issues — covering CoreDNS, ConfigMap, network connectivity, and common failure modes.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` — CoreDNS, DNS troubleshooting sections

#### Key Points to Cover in Your Answer:

**Step 1 — Confirm DNS is the issue (not app config):**
```bash
# Test from inside a pod
kubectl exec -it <pod-name> -- nslookup kubernetes.default
# If this fails → DNS is broken
# If this works → app is hardcoding IPs or wrong service name

# Check resolv.conf inside pod
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
# Should show:
# nameserver 10.96.0.10  (CoreDNS ClusterIP)
# search default.svc.cluster.local svc.cluster.local cluster.local
```

**Step 2 — Check CoreDNS pods:**
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
# All should be Running 1/1

kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
# Look for: SERVFAIL, connection refused, OOMKilled
```

**Step 3 — Check CoreDNS Service:**
```bash
kubectl get svc -n kube-system kube-dns
# Should show ClusterIP matching /etc/resolv.conf nameserver

kubectl get endpoints -n kube-system kube-dns
# Must have endpoints — if empty → CoreDNS pods not healthy
```

**Step 4 — Test CoreDNS directly:**
```bash
# Test from a debug pod
kubectl run dns-test --image=busybox --rm -it -- sh
nslookup kubernetes.default 10.96.0.10  # test specific CoreDNS IP
nslookup google.com 10.96.0.10          # test external resolution
```

**Common failure modes and fixes:**

**Failure 1: CoreDNS OOMKilled:**
```bash
kubectl describe pod coredns-xxx -n kube-system | grep OOMKilled
# Fix: increase CoreDNS memory limit
kubectl edit deployment coredns -n kube-system
# resources.limits.memory: 170Mi → 500Mi
```

**Failure 2: CoreDNS ConfigMap misconfigured:**
```bash
kubectl get configmap coredns -n kube-system -o yaml
# Check: forward . /etc/resolv.conf  (should be present for external DNS)
# Check: kubernetes cluster.local in-addr.arpa ip6.arpa  (for service discovery)
```

**Failure 3: Network Policy blocking DNS:**
```bash
kubectl get networkpolicy --all-namespaces
# Check if any policy blocks egress on UDP/TCP port 53
# Fix: add explicit allow for DNS
```

**Failure 4: CoreDNS not matching kube-dns service:**
```bash
kubectl get deployment coredns -n kube-system -o jsonpath='{.spec.template.metadata.labels}'
kubectl get svc kube-dns -n kube-system -o jsonpath='{.spec.selector}'
# Labels must match selector
```

**Failure 5: All CoreDNS pods on same node that is down:**
```bash
# CoreDNS should have pod anti-affinity to spread across nodes
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
# Check all pods are NOT on same node
```

> 💡 **Interview tip:** The fastest DNS debug command is: `kubectl run -it dns-debug --image=infoblox/dnstools --rm -- host kubernetes.default` — the `dnstools` image has comprehensive DNS debugging tools. Also mention **NodeLocal DNSCache** as a production solution for DNS performance — it runs a DNS cache on each node, reducing latency and CoreDNS load by 60-80%.

---

### Q202 — Grafana | Conceptual | Advanced

> Explain **Grafana alerting architecture** in Grafana 9+ (unified alerting) vs the old **legacy alerting**.
>
> What is a **contact point**, **notification policy**, and **silence** in unified alerting? How does the **alert routing tree** work? Write a notification policy that routes:
> - P1 alerts → PagerDuty + Slack #incidents
> - P2 alerts → Slack #alerts only
> - All resolved alerts → Slack #resolved

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — unified alerting, notification policies sections

#### Key Points to Cover in Your Answer:

**Legacy alerting problems:**
- Alert rules tied to specific dashboard panels
- No routing tree — every alert went to same notification channel
- Could not use multiple data sources in same alert
- No alert state management across Grafana restarts

**Unified alerting (Grafana 9+) improvements:**
- Alert rules independent of dashboards
- **Alert routing tree** — route different alerts to different channels
- Multi-datasource alerts (Prometheus + Loki + CloudWatch in one rule)
- Integrated Alertmanager (Grafana-managed or external)
- Alert state persisted in database

**Key concepts:**

**Contact Point** = Where notifications go (Slack, PagerDuty, email, webhook)
```yaml
# Example contact points
- name: pagerduty-critical
  type: pagerduty
  settings:
    integrationKey: <key>
    severity: critical

- name: slack-incidents
  type: slack
  settings:
    webhookURL: https://hooks.slack.com/xxx
    channel: "#incidents"

- name: slack-resolved
  type: slack
  settings:
    webhookURL: https://hooks.slack.com/xxx
    channel: "#resolved"
```

**Notification Policy** = Routing rules (which alerts go to which contact point)
```yaml
# Root policy
- receiver: slack-alerts        # default for unmatched
  group_by: [alertname, grafana_folder]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
  # P1: PagerDuty + Slack #incidents
  - matchers:
    - severity = critical
    - priority = P1
    receiver: pagerduty-critical
    continue: true              # keep matching other routes
  - matchers:
    - severity = critical
    receiver: slack-incidents

  # P2: Slack #alerts only
  - matchers:
    - severity = warning
    - priority = P2
    receiver: slack-warnings

  # Resolved alerts → #resolved channel
  - matchers:
    - alertstate = resolved     # special resolved state matcher
    receiver: slack-resolved
```

**Silence** = Temporarily suppress alerts (maintenance windows)
```
- Duration: specific time range
- Matchers: which alerts to suppress
- Creator + Comment: audit trail
- Does NOT delete alerts — just suppresses notifications
```

> 💡 **Interview tip:** The `continue: true` setting in notification policies is critical for sending P1 alerts to BOTH PagerDuty AND Slack simultaneously. Without `continue: true`, routing stops at the first matching route. This is one of the most common misconfiguration issues in Grafana alerting setups.

---

### Q203 — Terraform | Scenario-Based | Advanced

> Your Terraform configuration manages **AWS IAM policies** but the policy documents are becoming very complex — 500+ line JSON documents that are hard to read and maintain.
>
> How would you refactor the IAM policy management to be more maintainable? Show how to use **`aws_iam_policy_document` data source**, **heredoc**, and **templatefile()** function to manage complex IAM policies cleanly.

📁 **Reference:** `nawab312/Terraform` — IAM management, templatefile, policy documents sections

#### Key Points to Cover in Your Answer:

**Approach 1 — `aws_iam_policy_document` data source (BEST):**
```hcl
# Readable, validates JSON structure, supports conditions natively
data "aws_iam_policy_document" "s3_access" {
  statement {
    sid    = "AllowS3Read"
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:ListBucket"
    ]
    resources = [
      "arn:aws:s3:::${var.bucket_name}",
      "arn:aws:s3:::${var.bucket_name}/*"
    ]
    condition {
      test     = "StringEquals"
      variable = "s3:prefix"
      values   = ["${var.team}/"]
    }
  }

  statement {
    sid    = "DenyDeleteObject"
    effect = "Deny"
    actions   = ["s3:DeleteObject"]
    resources = ["arn:aws:s3:::${var.bucket_name}/*"]
    principals {
      type        = "AWS"
      identifiers = ["*"]
    }
  }
}

resource "aws_iam_role_policy" "s3_access" {
  name   = "s3-access"
  role   = aws_iam_role.my_role.id
  policy = data.aws_iam_policy_document.s3_access.json
}
```

**Approach 2 — `templatefile()` for dynamic policies:**
```hcl
# templates/s3_policy.json.tpl
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ${jsonencode(actions)},
      "Resource": [
        "arn:aws:s3:::${bucket_name}",
        "arn:aws:s3:::${bucket_name}/*"
      ]
    }
  ]
}

# main.tf
resource "aws_iam_role_policy" "dynamic" {
  name = "dynamic-s3-policy"
  role = aws_iam_role.my_role.id
  policy = templatefile("${path.module}/templates/s3_policy.json.tpl", {
    bucket_name = var.bucket_name
    actions     = ["s3:GetObject", "s3:ListBucket"]
  })
}
```

**Approach 3 — Heredoc (simple, readable for static policies):**
```hcl
resource "aws_iam_role_policy" "inline" {
  name = "inline-policy"
  role = aws_iam_role.my_role.id
  policy = <<-EOF
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": ["s3:GetObject"],
          "Resource": "arn:aws:s3:::${var.bucket_name}/*"
        }
      ]
    }
  EOF
}
```

**Comparison:**
| Approach | Validation | Dynamic | Readability |
|---|---|---|---|
| `aws_iam_policy_document` | ✅ Terraform validates | ✅ Variables, loops | ✅ Best |
| `templatefile()` | ❌ No validation | ✅ Template syntax | ⚠️ Medium |
| Heredoc | ❌ No validation | ⚠️ Basic interpolation | ⚠️ Medium |
| `file()` | ❌ No validation | ❌ Static | ❌ Worst |

> 💡 **Interview tip:** Always recommend `aws_iam_policy_document` over raw JSON — it **validates IAM policy syntax at plan time** (catches mistakes before apply), supports Terraform variables natively, and produces a consistent JSON format that prevents perpetual diffs. The biggest benefit: if you write invalid JSON in a heredoc, you only find out when AWS rejects it during apply. With `aws_iam_policy_document`, Terraform catches it at plan time.

---

### Q204 — Python | Scenario-Based | Advanced

> Write a **Python script** that implements a **Terraform drift detector**:
> - Runs `terraform plan -json` and parses the JSON output
> - Identifies resources that have **drifted** (changed outside Terraform)
> - Groups drift by **resource type** and **change type** (update/delete/create)
> - Outputs a **summary report** with counts and details
> - Sends the report to a **Slack webhook** if drift is detected
> - Exits with code `1` if drift found (useful in CI/CD)

📁 **Reference:** `nawab312/DSA` → `Python` — JSON parsing, subprocess, Slack webhooks sections

#### Key Points to Cover in Your Answer:

```python
#!/usr/bin/env python3
"""Terraform Drift Detector — detects infrastructure drift and reports to Slack"""

import subprocess
import json
import sys
import os
from collections import defaultdict
from datetime import datetime
import urllib.request
import urllib.parse

SLACK_WEBHOOK = os.environ.get("SLACK_WEBHOOK_URL", "")
WORKING_DIR   = os.environ.get("TF_WORKING_DIR", ".")


def run_terraform_plan() -> dict:
    """Run terraform plan -json and return parsed output."""
    try:
        result = subprocess.run(
            ["terraform", "plan", "-json", "-detailed-exitcode"],
            capture_output=True,
            text=True,
            cwd=WORKING_DIR
        )
        # Exit codes: 0=no changes, 1=error, 2=changes present
        if result.returncode == 1:
            print(f"ERROR: terraform plan failed:\n{result.stderr}")
            sys.exit(1)

        # Parse JSON lines (terraform plan -json outputs one JSON per line)
        events = []
        for line in result.stdout.strip().split("\n"):
            if line:
                try:
                    events.append(json.loads(line))
                except json.JSONDecodeError:
                    continue

        return {"events": events, "has_changes": result.returncode == 2}

    except FileNotFoundError:
        print("ERROR: terraform binary not found")
        sys.exit(1)


def extract_drift(events: list) -> list:
    """Extract resource changes from terraform plan JSON output."""
    changes = []

    for event in events:
        if event.get("type") == "resource_drift":
            # resource_drift events indicate external changes
            change = event.get("change", {})
            changes.append({
                "type":          "DRIFT",
                "resource_type": event.get("resource", {}).get("resource_type", "unknown"),
                "resource_name": event.get("resource", {}).get("resource_name", "unknown"),
                "action":        change.get("action", "unknown"),
                "before":        change.get("before", {}),
                "after":         change.get("after", {})
            })

        elif event.get("type") == "planned_change":
            # planned_change events indicate Terraform wants to make changes
            change = event.get("change", {})
            actions = change.get("actions", [])
            if actions and actions != ["no-op"]:
                changes.append({
                    "type":          "PLANNED",
                    "resource_type": event.get("resource", {}).get("resource_type", "unknown"),
                    "resource_name": event.get("resource", {}).get("resource_name", "unknown"),
                    "action":        "+".join(actions),
                    "before":        change.get("before", {}),
                    "after":         change.get("after", {})
                })

    return changes


def group_changes(changes: list) -> dict:
    """Group changes by resource type and action."""
    grouped = defaultdict(lambda: defaultdict(list))
    for change in changes:
        grouped[change["resource_type"]][change["action"]].append(change)
    return grouped


def format_report(changes: list, grouped: dict) -> str:
    """Format a human-readable drift report."""
    if not changes:
        return "✅ No drift detected — infrastructure matches Terraform state."

    lines = [
        f"🚨 *Terraform Drift Detected* — {datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S UTC')}",
        f"*Total changes:* {len(changes)}",
        ""
    ]

    # Summary by resource type
    lines.append("*Summary by resource type:*")
    for resource_type, actions in grouped.items():
        action_summary = ", ".join(
            f"{action}: {len(items)}"
            for action, items in actions.items()
        )
        lines.append(f"  • `{resource_type}` → {action_summary}")

    lines.append("")
    lines.append("*Details:*")
    for change in changes[:10]:  # limit to first 10 for Slack
        lines.append(
            f"  • [{change['type']}] `{change['resource_type']}.{change['resource_name']}` "
            f"— action: `{change['action']}`"
        )

    if len(changes) > 10:
        lines.append(f"  • ... and {len(changes) - 10} more changes")

    return "\n".join(lines)


def send_slack_notification(message: str) -> None:
    """Send notification to Slack webhook."""
    if not SLACK_WEBHOOK:
        print("WARNING: SLACK_WEBHOOK_URL not set — skipping Slack notification")
        return

    payload = json.dumps({"text": message}).encode("utf-8")
    req = urllib.request.Request(
        SLACK_WEBHOOK,
        data=payload,
        headers={"Content-Type": "application/json"}
    )

    try:
        with urllib.request.urlopen(req, timeout=10) as response:
            if response.status == 200:
                print("Slack notification sent successfully")
            else:
                print(f"WARNING: Slack returned status {response.status}")
    except Exception as e:
        print(f"WARNING: Failed to send Slack notification: {e}")


def main():
    print(f"Running terraform plan in: {WORKING_DIR}")

    # Run terraform plan
    result = run_terraform_plan()

    # Extract and group drift
    changes = extract_drift(result["events"])
    grouped = group_changes(changes)

    # Format report
    report = format_report(changes, grouped)
    print(report)

    # Send to Slack if drift detected
    if changes:
        send_slack_notification(report)
        print(f"\nDrift detected: {len(changes)} changes found")
        sys.exit(1)   # exit code 1 for CI/CD to fail the pipeline
    else:
        print("No drift detected")
        sys.exit(0)


if __name__ == "__main__":
    main()
```

**Usage:**
```bash
# Basic usage
export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/xxx"
export TF_WORKING_DIR="/path/to/terraform"
python3 drift_detector.py

# In GitHub Actions (scheduled)
- name: Check for drift
  run: python3 drift_detector.py
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
    TF_WORKING_DIR: environments/prod
```

> 💡 **Interview tip:** The **exit code 1 on drift detected** is the most important feature for CI/CD integration — it causes the pipeline to fail, which triggers a notification in GitHub/Jenkins and creates a clear audit trail. Also mention that `resource_drift` events in the JSON output specifically indicate changes made **outside Terraform** (external drift), while `planned_change` events indicate Terraform wants to make changes to match your config.

---

### Q205 — AWS | Conceptual | Advanced

> Explain **AWS Secrets Manager** vs **AWS Parameter Store (SSM)**.
>
> What are the key differences in terms of **cost**, **rotation support**, **size limits**, **cross-account access**, and **integration with services**? When would you choose one over the other? How do you **rotate database credentials** automatically using Secrets Manager?

📁 **Reference:** `nawab312/AWS` — Secrets Manager, SSM Parameter Store, credential rotation sections

#### Key Points to Cover in Your Answer:

**Comparison Table:**

| Feature | Secrets Manager | SSM Parameter Store |
|---|---|---|
| **Cost** | $0.40/secret/month + API calls | Free (Standard) / $0.05/param/month (Advanced) |
| **Max size** | 64KB per secret | 4KB (Standard) / 8KB (Advanced) |
| **Auto rotation** | ✅ Built-in (Lambda-based) | ❌ Manual / custom Lambda |
| **Cross-account** | ✅ Resource-based policy | ❌ Not natively |
| **Versioning** | ✅ Up to 100 versions | ✅ Up to 100 versions |
| **Encryption** | ✅ KMS (mandatory) | ✅ KMS (SecureString type) |
| **Hierarchy** | ❌ Flat | ✅ Path-based (/app/prod/db) |
| **Service integrations** | RDS, Redshift, DocumentDB | EC2, ECS, Lambda, SSM Agent |

**When to use Secrets Manager:**
- Database credentials that need **automatic rotation**
- Secrets shared **across accounts**
- Secrets requiring **compliance auditing** (who accessed what, when)
- Integration with RDS/Aurora where AWS manages rotation

**When to use Parameter Store:**
- **Configuration values** alongside secrets (mixed config+secrets)
- **Hierarchical configuration** (organized by environment/app)
- Cost-sensitive environments (free for standard tier)
- Simple secrets that don't need rotation

**Automatic RDS credential rotation:**
```python
# Secrets Manager rotation Lambda flow:
# 1. createSecret: Generate new random password
# 2. setSecret: Update RDS with new password (while keeping old one active)
# 3. testSecret: Verify new credentials work
# 4. finishSecret: Mark new version as CURRENT, old as PREVIOUS

# Configure rotation in Terraform:
resource "aws_secretsmanager_secret_rotation" "rds" {
  secret_id           = aws_secretsmanager_secret.rds_password.id
  rotation_lambda_arn = "arn:aws:lambda:::function:SecretsManagerRDSPostgreSQLRotationSingleUser"
  rotation_rules {
    automatically_after_days = 30
  }
}

# Application fetches secret dynamically (not cached):
import boto3, json
client = boto3.client('secretsmanager')
secret = json.loads(client.get_secret_value(SecretId='prod/rds/password')['SecretString'])
connection = psycopg2.connect(
    host=secret['host'],
    user=secret['username'],
    password=secret['password']
)
```

**Key rotation considerations:**
- Applications must fetch secrets **dynamically** (not cache long-term)
- Or implement a **rotation notification** via EventBridge → refresh cached secret
- AWS provides pre-built rotation Lambdas for RDS, Redshift, DocumentDB

> 💡 **Interview tip:** The most important distinction interviewers want to hear is: use **Secrets Manager for anything that needs automatic rotation** (database passwords, API keys) and **Parameter Store for configuration that doesn't rotate** (feature flags, connection strings, non-secret config). Also mention that Secrets Manager's **cross-account access** via resource-based policies is a key advantage in multi-account AWS Organizations setups.

---

### Q206 — Kubernetes | Scenario-Based | Advanced

> You need to implement **Kubernetes node auto-provisioning** (Karpenter) to replace the Cluster Autoscaler. Your cluster has:
> - Workloads with varying CPU/memory requirements (1 CPU to 96 CPU jobs)
> - Need to use **Spot instances** for cost savings with fallback to On-Demand
> - Different instance families preferred for different workloads (compute-optimized, memory-optimized)
>
> Walk me through setting up Karpenter with these requirements.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md` — Cluster Autoscaler, node provisioning sections

#### Key Points to Cover in Your Answer:

**Karpenter vs Cluster Autoscaler:**
| Feature | Cluster Autoscaler | Karpenter |
|---|---|---|
| Node selection | Predefined node groups | Any EC2 instance type |
| Speed | 3-5 minutes | 30-60 seconds |
| Bin packing | Limited | Optimal |
| Spot diversification | Manual config | Automatic |
| Right-sizing | Fixed node sizes | Pod-request-based sizing |

**Installation:**
```bash
# Install Karpenter via Helm
helm install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --set settings.clusterName=my-cluster \
  --set settings.interruptionQueue=karpenter-interruption-queue \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi
```

**NodePool for general workloads (Spot with On-Demand fallback):**
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: general
spec:
  template:
    spec:
      nodeClassRef:
        name: default
      requirements:
        # Allow multiple instance families for Spot diversification
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["m5", "m5a", "m6i", "m6a", "m7i"]
        # Prefer Spot, fallback to On-Demand
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        # Reasonable instance sizes
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: ["xlarge", "2xlarge", "4xlarge"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]

  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s       # remove empty nodes quickly
    expireAfter: 168h            # replace nodes weekly (fresh AMIs)

  limits:
    cpu: "1000"
    memory: 4000Gi
```

**NodePool for ML/GPU workloads:**
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: ml-compute
spec:
  template:
    metadata:
      labels:
        workload-type: ml
    spec:
      nodeClassRef:
        name: gpu-nodeclass
      taints:
        - key: nvidia.com/gpu
          value: "true"
          effect: NoSchedule    # only GPU pods scheduled here
      requirements:
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["p3", "p4d", "g4dn"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
```

**EC2NodeClass for AWS-specific config:**
```yaml
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  role: KarpenterNodeRole
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        encrypted: true
```

**Pod using Karpenter node selection:**
```yaml
spec:
  nodeSelector:
    workload-type: ml           # target ML node pool
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
  resources:
    requests:
      nvidia.com/gpu: "1"
      cpu: "8"
      memory: "32Gi"
```

> 💡 **Interview tip:** The biggest advantage of Karpenter over Cluster Autoscaler is **just-in-time node provisioning** — instead of pre-provisioning node groups of specific sizes, Karpenter launches the **exact instance type** that fits the pending pod's requests. A pod requesting 47 CPU cores gets a 48-core instance, not a 64-core instance that wastes 17 cores. This bin-packing optimization alone can reduce EC2 costs by 20-30%.

---

### Q207 — ELK Stack | Scenario-Based | Advanced

> You need to implement **log-based alerting** using the ELK stack. When more than **100 ERROR logs appear within 5 minutes** from the payment service, an alert should:
> - Send a **Slack notification** with a sample of the error messages
> - Create a **PagerDuty incident** if it persists for more than 10 minutes
> - **Auto-resolve** when error rate drops below 10/minute
>
> Walk me through implementing this using **Elasticsearch Watcher** or **Kibana Alerting**.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack` — Kibana alerting, Watcher, connectors sections

#### Key Points to Cover in Your Answer:

**Approach 1 — Kibana Alerting (modern, UI-based):**

```
Stack Management → Rules → Create Rule → Elasticsearch query
```

```json
{
  "name": "Payment Service High Error Rate",
  "rule_type_id": "es_query",
  "consumer": "alerts",
  "schedule": {"interval": "1m"},
  "params": {
    "index": ["app-logs-*"],
    "timeField": "@timestamp",
    "timeWindowSize": 5,
    "timeWindowUnit": "m",
    "thresholdComparator": ">",
    "threshold": [100],
    "searchType": "esQuery",
    "esQuery": {
      "query": {
        "bool": {
          "must": [
            {"term": {"service.name": "payment-service"}},
            {"term": {"log.level": "ERROR"}}
          ],
          "filter": [
            {"range": {"@timestamp": {"gte": "now-5m"}}}
          ]
        }
      }
    }
  },
  "actions": [
    {
      "id": "slack-connector",
      "group": "threshold met",
      "params": {
        "message": "🚨 High error rate on payment-service: {{context.value}} errors in 5 minutes\nSample errors:\n{{context.hits}}"
      }
    },
    {
      "id": "pagerduty-connector",
      "group": "threshold met",
      "frequency": {"notifyWhen": "activeAlert", "throttle": "10m"},
      "params": {
        "summary": "Payment service critical error rate",
        "severity": "critical"
      }
    }
  ]
}
```

**Approach 2 — Elasticsearch Watcher (API-based, more powerful):**
```json
PUT _watcher/watch/payment_error_alert
{
  "trigger": {
    "schedule": {"interval": "1m"}
  },
  "input": {
    "search": {
      "request": {
        "indices": ["app-logs-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                {"term": {"service.name": "payment-service"}},
                {"term": {"log.level": "ERROR"}}
              ],
              "filter": [{"range": {"@timestamp": {"gte": "now-5m"}}}]
            }
          },
          "aggs": {
            "error_samples": {
              "top_hits": {"size": 3, "_source": ["message", "@timestamp"]}
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {"ctx.payload.hits.total.value": {"gt": 100}}
  },
  "actions": {
    "slack_alert": {
      "slack": {
        "message": {
          "to": ["#incidents"],
          "text": "🚨 {{ctx.payload.hits.total.value}} errors from payment-service in 5 mins",
          "attachments": [{
            "text": "{{#ctx.payload.aggregations.error_samples.hits.hits}}{{_source.message}}\n{{/ctx.payload.aggregations.error_samples.hits.hits}}"
          }]
        }
      }
    }
  },
  "throttle_period": "10m"
}
```

**Auto-resolve:**
```json
// Add resolve condition in Kibana alerting
"recoveryActionGroup": {
  "id": "recovered",
  "name": "Recovered"
},
"actions": [
  {
    "group": "recovered",
    "id": "slack-connector",
    "params": {
      "message": "✅ Payment service error rate normalized — alert resolved"
    }
  }
]
```

> 💡 **Interview tip:** For production log-based alerting, **Kibana Alerting** is preferred over Elasticsearch Watcher — it has a better UI, tighter integration with Elastic Security, and is actively developed. Watcher is still useful for complex scenarios (cross-index joins, complex aggregations) that Kibana Alerting cannot express. Always include **sample error messages** in the alert notification — an alert saying "100 errors" is much less actionable than one showing the actual error message.

---

### Q208 — Jenkins | Scenario-Based | Advanced

> You need to implement a **Jenkins pipeline** that performs **database schema migrations** safely:
> - Runs **Flyway** migrations before deploying the new application version
> - If migration **fails** — abort deployment, do NOT deploy new code
> - If migration **succeeds** — deploy new application version
> - If application deployment **fails** after migration — the migration is NOT rolled back (forward-only)
> - Sends detailed **notification** of migration results (which scripts ran, duration)
>
> Write the complete Jenkinsfile.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — database migration pipeline, sequential stages, notifications sections

#### Key Points to Cover in Your Answer:

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: flyway
    image: flyway/flyway:10-alpine
    command: ['cat']
    tty: true
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['cat']
    tty: true
"""
        }
    }

    environment {
        DB_URL      = credentials('prod-db-url')
        DB_USER     = credentials('prod-db-user')
        DB_PASSWORD = credentials('prod-db-password')
        IMAGE_TAG   = "${env.GIT_COMMIT[0..7]}"
        MIGRATION_START_TIME = ''
        MIGRATION_RESULT     = ''
        SCRIPTS_APPLIED      = ''
    }

    stages {
        stage('Run Database Migrations') {
            steps {
                container('flyway') {
                    script {
                        env.MIGRATION_START_TIME = new Date().format("HH:mm:ss")

                        // Capture Flyway output for notification
                        def flywayOutput = sh(
                            script: """
                                flyway \
                                  -url="${DB_URL}" \
                                  -user="${DB_USER}" \
                                  -password="${DB_PASSWORD}" \
                                  -locations=filesystem:/flyway/sql \
                                  -outOfOrder=false \
                                  migrate
                            """,
                            returnStdout: true
                        ).trim()

                        echo "Flyway output:\n${flywayOutput}"

                        // Extract scripts that were applied
                        env.SCRIPTS_APPLIED = flywayOutput
                            .split('\n')
                            .findAll { it.contains('Successfully applied') || it.contains('V') }
                            .join('\n')

                        env.MIGRATION_RESULT = 'SUCCESS'
                    }
                }
            }
            post {
                failure {
                    script {
                        env.MIGRATION_RESULT = 'FAILED'
                        // Notify migration failure
                        slackSend(
                            color: 'danger',
                            message: """
❌ *Database Migration FAILED* — Deployment ABORTED
*Build:* #${BUILD_NUMBER}
*Time:* ${env.MIGRATION_START_TIME}
*Reason:* Migration script failed — check Flyway logs
*Action:* Application NOT deployed — database is safe
*Logs:* ${BUILD_URL}console
                            """.stripIndent()
                        )
                    }
                }
            }
        }

        // Only reached if migration SUCCEEDED
        stage('Deploy Application') {
            steps {
                container('kubectl') {
                    script {
                        sh """
                            kubectl set image deployment/payment-service \
                              payment-service=${ECR_REGISTRY}/payment-service:${IMAGE_TAG} \
                              -n production

                            kubectl rollout status deployment/payment-service \
                              -n production \
                              --timeout=300s
                        """
                    }
                }
            }
            post {
                failure {
                    script {
                        // Migration succeeded, deployment failed
                        // DO NOT rollback migration — forward-only
                        slackSend(
                            color: 'danger',
                            message: """
⚠️ *Deployment FAILED — Migration was SUCCESSFUL*
*Build:* #${BUILD_NUMBER}
*Migration:* ✅ Completed successfully
*Deployment:* ❌ Failed
*IMPORTANT:* Database migration is NOT rolled back
*Action Required:* Fix forward — deploy a compatible version
*Logs:* ${BUILD_URL}console
                            """.stripIndent()
                        )
                        // Rollback deployment to previous version
                        sh "kubectl rollout undo deployment/payment-service -n production"
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                def endTime = new Date().format("HH:mm:ss")
                slackSend(
                    color: 'good',
                    message: """
✅ *Deployment Successful*
*Build:* #${BUILD_NUMBER}
*Image:* ${IMAGE_TAG}
*Migration:* ✅ ${MIGRATION_RESULT}
*Scripts applied:*
${env.SCRIPTS_APPLIED ?: 'No new migrations'}
*Deployed at:* ${endTime}
                    """.stripIndent()
                )
            }
        }
    }
}
```

**Key design decisions:**
1. Flyway runs in dedicated container with DB credentials
2. Migration failure → `post { failure }` block sends notification → stage fails → pipeline aborts → deploy stage never runs
3. Deploy failure → rollback deployment ONLY (not migration) — schema is forward-only
4. Migration output captured for detailed Slack notification

> 💡 **Interview tip:** The **forward-only migration** principle is critical — database migrations should never be rolled back automatically because data may have already been written using the new schema. If deployment fails after migration, the fix is to deploy a new version that is compatible with both old and new schema (expand-contract pattern). Always mention this design principle in interviews — it shows you understand the coupling between schema changes and code deployments.

---

### Q209 — Linux / Bash | Scenario-Based | Advanced

> Write a **Bash script** that implements a **simple service discovery** mechanism:
> - Reads a `services.conf` file with format: `service_name:host:port:health_check_path`
> - Checks health of each service by hitting the health check endpoint
> - Maintains a **`healthy_services.txt`** and **`unhealthy_services.txt`** file
> - Runs every 30 seconds in a loop
> - When a service transitions from healthy→unhealthy or vice versa, **logs the transition** with timestamp
> - Exposes a simple **HTTP response** via netcat showing current service status

📁 **Reference:** `nawab312/DSA` → `Linux` — bash scripting, curl, netcat, service health sections

#### Key Points to Cover in Your Answer:

```bash
#!/bin/bash
# Simple Service Discovery & Health Checker

SERVICES_CONF="services.conf"
HEALTHY_FILE="healthy_services.txt"
UNHEALTHY_FILE="unhealthy_services.txt"
TRANSITION_LOG="transitions.log"
HTTP_PORT=8888
CHECK_INTERVAL=30
TIMEOUT=5

# Initialize files
> "$HEALTHY_FILE"
> "$UNHEALTHY_FILE"
touch "$TRANSITION_LOG"

# Store previous states for transition detection
declare -A PREV_STATES

log_transition() {
    local service="$1"
    local old_state="$2"
    local new_state="$3"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] TRANSITION: $service → $old_state → $new_state" | \
        tee -a "$TRANSITION_LOG"
}

check_service_health() {
    local name="$1"
    local host="$2"
    local port="$3"
    local path="$4"

    local url="http://${host}:${port}${path}"
    local http_code

    http_code=$(curl -s -o /dev/null -w "%{http_code}" \
        --connect-timeout "$TIMEOUT" \
        --max-time "$TIMEOUT" \
        "$url" 2>/dev/null)

    if [[ "$http_code" =~ ^2 ]]; then
        echo "healthy"
    else
        echo "unhealthy"
    fi
}

update_service_files() {
    local name="$1"
    local state="$2"
    local host="$3"
    local port="$4"

    # Remove from both files first
    sed -i "/^${name}:/d" "$HEALTHY_FILE" 2>/dev/null
    sed -i "/^${name}:/d" "$UNHEALTHY_FILE" 2>/dev/null

    # Add to appropriate file
    if [[ "$state" == "healthy" ]]; then
        echo "${name}:${host}:${port}" >> "$HEALTHY_FILE"
    else
        echo "${name}:${host}:${port}" >> "$UNHEALTHY_FILE"
    fi
}

run_http_server() {
    while true; do
        {
            # Read request (ignore it)
            read -r _

            # Build response
            healthy_count=$(wc -l < "$HEALTHY_FILE")
            unhealthy_count=$(wc -l < "$UNHEALTHY_FILE")
            response="Service Discovery Status\n"
            response+="========================\n"
            response+="Healthy ($healthy_count):\n"
            while IFS=: read -r name host port; do
                response+="  ✅ $name ($host:$port)\n"
            done < "$HEALTHY_FILE"
            response+="\nUnhealthy ($unhealthy_count):\n"
            while IFS=: read -r name host port; do
                response+="  ❌ $name ($host:$port)\n"
            done < "$UNHEALTHY_FILE"

            printf "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n${response}"
        } | nc -l -p "$HTTP_PORT" -q 1 2>/dev/null
    done
}

# Start HTTP server in background
run_http_server &
HTTP_PID=$!
echo "Health status HTTP server started on port $HTTP_PORT (PID: $HTTP_PID)"

# Graceful shutdown
cleanup() {
    echo "Shutting down..."
    kill "$HTTP_PID" 2>/dev/null
    exit 0
}
trap cleanup SIGINT SIGTERM

# Main health check loop
echo "Starting service health checks (interval: ${CHECK_INTERVAL}s)"

while true; do
    if [[ ! -f "$SERVICES_CONF" ]]; then
        echo "ERROR: $SERVICES_CONF not found"
        sleep "$CHECK_INTERVAL"
        continue
    fi

    while IFS=: read -r name host port path; do
        # Skip comments and empty lines
        [[ "$name" =~ ^#.*$ || -z "$name" ]] && continue

        current_state=$(check_service_health "$name" "$host" "$port" "$path")
        prev_state="${PREV_STATES[$name]:-unknown}"

        # Detect state transitions
        if [[ "$prev_state" != "unknown" && "$prev_state" != "$current_state" ]]; then
            log_transition "$name" "$prev_state" "$current_state"
        fi

        update_service_files "$name" "$current_state" "$host" "$port"
        PREV_STATES[$name]="$current_state"

    done < "$SERVICES_CONF"

    sleep "$CHECK_INTERVAL"
done
```

**Example `services.conf`:**
```
# service_name:host:port:health_check_path
payment-api:10.0.1.10:8080:/healthz
auth-service:10.0.1.11:8080:/health
user-service:10.0.1.12:3000:/api/health
```

**Test the HTTP server:**
```bash
curl http://localhost:8888
# Service Discovery Status
# ========================
# Healthy (2):
#   ✅ payment-api (10.0.1.10:8080)
#   ✅ auth-service (10.0.1.11:8080)
# Unhealthy (1):
#   ❌ user-service (10.0.1.12:3000)
```

> 💡 **Interview tip:** Mention that this is a **demonstration of the concept** — in production, use proper service discovery tools (Consul, Kubernetes Services, AWS Cloud Map). The value of knowing how to build this from scratch is understanding what these tools are doing internally: periodic health checks, maintaining healthy/unhealthy registries, and exposing the current state via an API.

---

### Q210 — AWS | Scenario-Based | Advanced

> Your company's **AWS bill has a $30,000 charge for data transfer** this month — 5x higher than usual. You need to identify the source.
>
> Walk me through how you would use **AWS Cost Explorer**, **VPC Flow Logs**, **CloudWatch**, and other tools to identify exactly which resources are generating the unexpected data transfer costs and how to reduce them.

📁 **Reference:** `nawab312/AWS` — Cost Explorer, VPC Flow Logs, data transfer costs, VPC endpoints sections

#### Key Points to Cover in Your Answer:

**Data transfer cost categories (AWS charges for):**
- Data out to internet (EC2 → internet)
- Cross-region data transfer
- Cross-AZ data transfer (between AZs in same region)
- NAT Gateway data processing
- Data out from CloudFront origin

**Step 1 — Cost Explorer investigation:**
```bash
# Filter Cost Explorer:
# Service = EC2-Other (data transfer shows here, not EC2)
# Group by: Usage Type → look for:
#   DataTransfer-Out-Bytes (internet egress)
#   DataTransfer-Regional-Bytes (cross-AZ)
#   NatGateway-Bytes (NAT processing)

# Also check:
# Service = CloudFront → DataTransfer-Out
# Service = S3 → DataTransfer-OUT
```

**Step 2 — Identify top offenders with VPC Flow Logs:**
```bash
# Enable VPC Flow Logs to S3 or CloudWatch Logs
# Query with Athena:
SELECT
  srcaddr,
  dstaddr,
  SUM(bytes) as total_bytes,
  SUM(bytes)/1073741824.0 as total_gb
FROM vpc_flow_logs
WHERE
  start >= 1705276800  -- start of billing period
  AND action = 'ACCEPT'
GROUP BY srcaddr, dstaddr
ORDER BY total_bytes DESC
LIMIT 20;

# This shows: which IPs are sending/receiving the most data
```

**Step 3 — Identify cross-AZ traffic:**
```bash
# In Cost Explorer: Usage Type = "DataTransfer-Regional-Bytes"
# This is usually cross-AZ traffic (ELB → EC2 in different AZ, etc.)

# Common culprit: Kubernetes with pods in AZ-a calling service in AZ-b
# Solution: topology-aware routing (EndpointSlice topology hints)
```

**Common root causes and fixes:**

| Root Cause | Evidence | Fix |
|---|---|---|
| S3 → Internet (no endpoint) | S3 DataTransfer-OUT high | Add S3 Gateway VPC Endpoint (free) |
| EC2 → Internet directly | DataTransfer-Out-Bytes high | Use CloudFront for content delivery |
| Cross-AZ traffic | DataTransfer-Regional high | Topology-aware routing, keep traffic in AZ |
| NAT Gateway overuse | NatGateway-Bytes high | Add VPC endpoints for AWS services |
| EC2 → S3 via internet | S3 + EC2 data transfer both high | S3 VPC Endpoint (traffic stays in AWS) |
| CloudFront → Origin | Origin fetches high | Increase CloudFront cache TTL |

**Biggest wins — VPC Endpoints (free for gateway endpoints):**
```bash
# S3 Gateway Endpoint (FREE, reduces NAT + internet charges)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-12345

# DynamoDB Gateway Endpoint (FREE)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --service-name com.amazonaws.us-east-1.dynamodb \
  --route-table-ids rtb-12345
```

> 💡 **Interview tip:** The **S3 VPC Gateway Endpoint** is one of the highest-ROI AWS cost optimizations — it is completely **free** and immediately eliminates data transfer charges for EC2→S3 traffic (which was going via internet/NAT Gateway). In clusters with heavy S3 usage, this alone can save thousands per month. Always check this first when investigating unexpected data transfer costs.

---

### Q211 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `projected volumes`**. What types of data can be combined into a projected volume?
>
> Give a real-world example of a Pod that needs:
> - A **ServiceAccount token** with a specific audience and expiry
> - A **ConfigMap** value
> - A **Secret** value
>
> All mounted at different paths within the same volume mount. Write the Pod spec.

📁 **Reference:** `nawab312/Kubernetes` → `04_STORAGE.md` and `05_CONFIGURATION_SECRETS.md` — projected volumes, token projection sections

#### Key Points to Cover in Your Answer:

**What projected volumes do:**
- Combine multiple volume sources into a **single directory mount**
- Instead of mounting 3 separate volumes at 3 paths, mount 1 projected volume with all 3 sources
- Cleaner Pod spec, single volume mount point

**Supported sources in projected volumes:**
1. `serviceAccountToken` — projected SA token with custom audience/expiry
2. `configMap` — ConfigMap data
3. `secret` — Secret data
4. `downwardAPI` — pod metadata (labels, annotations, resource limits)

**Real-world Pod spec:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payment-service
spec:
  serviceAccountName: payment-sa

  volumes:
  - name: combined-config
    projected:
      sources:

      # Source 1: ServiceAccount token for calling AWS APIs (IRSA)
      # Custom audience ensures token only valid for AWS STS
      - serviceAccountToken:
          path: token                    # mounted at /var/run/secrets/token
          expirationSeconds: 3600        # rotated every hour
          audience: "sts.amazonaws.com"  # audience restriction

      # Source 2: ConfigMap — app configuration
      - configMap:
          name: payment-config
          items:
          - key: app.properties
            path: config/app.properties  # mounted at /var/run/secrets/config/app.properties

      # Source 3: Secret — database password
      - secret:
          name: db-credentials
          items:
          - key: password
            path: db/password            # mounted at /var/run/secrets/db/password
          - key: username
            path: db/username

      # Source 4: Pod metadata (optional)
      - downwardAPI:
          items:
          - path: labels
            fieldRef:
              fieldPath: metadata.labels  # mounted at /var/run/secrets/labels

  containers:
  - name: payment-service
    image: payment-service:v1.2.3
    volumeMounts:
    - name: combined-config
      mountPath: /var/run/secrets       # single mount point for ALL sources
      readOnly: true

    # All files accessible at:
    # /var/run/secrets/token            ← SA token
    # /var/run/secrets/config/app.properties ← ConfigMap
    # /var/run/secrets/db/password      ← Secret
    # /var/run/secrets/labels           ← DownwardAPI
```

**Why use projected volumes vs separate volumes:**
```yaml
# Without projected volumes (verbose):
volumes:
- name: sa-token
  serviceAccountToken:
    audience: "sts.amazonaws.com"
    expirationSeconds: 3600
    path: token
- name: app-config
  configMap:
    name: payment-config
- name: db-creds
  secret:
    secretName: db-credentials

volumeMounts:
- name: sa-token
  mountPath: /var/run/secrets/token
  subPath: token
- name: app-config
  mountPath: /var/run/config
- name: db-creds
  mountPath: /var/run/db

# With projected volumes: ONE volume, ONE mountPath, cleaner spec
```

> 💡 **Interview tip:** The **`serviceAccountToken` with custom audience** in projected volumes is the key enabler for **IRSA (IAM Roles for Service Accounts)** in EKS. The projected token has `sts.amazonaws.com` as the audience, which AWS STS accepts for `AssumeRoleWithWebIdentity`. Without projected volumes, you would need the old approach of mounting a long-lived static token — projected tokens are short-lived and automatically rotated, which is significantly more secure.

---

### Q212 — GitHub Actions | Scenario-Based | Advanced

> Your team wants to implement **dependency review** and **security scanning** in GitHub Actions:
> - Block PRs that introduce **critical CVE vulnerabilities** in dependencies
> - Run **SAST (Static Application Security Testing)** on every PR
> - Scan Docker images for vulnerabilities before pushing to ECR
> - Generate a **SBOM (Software Bill of Materials)** on every release
> - Store scan results in **GitHub Security tab**
>
> Write the complete workflow implementing all security checks.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions` — security scanning, dependency review, SARIF sections

#### Key Points to Cover in Your Answer:

```yaml
name: Security Scanning

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
    tags: ['v*']

permissions:
  contents: read
  security-events: write    # required for uploading SARIF to Security tab
  pull-requests: write      # required for dependency review comments

jobs:
  # ─────────────────────────────────────────────
  # Job 1: Dependency Review (PR only)
  # Blocks PRs introducing vulnerable dependencies
  # ─────────────────────────────────────────────
  dependency-review:
    name: Dependency Review
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: critical    # block PRs with CRITICAL CVEs
          deny-licenses: GPL-3.0, AGPL-3.0  # also block certain licenses

  # ─────────────────────────────────────────────
  # Job 2: SAST with CodeQL
  # ─────────────────────────────────────────────
  codeql-analysis:
    name: SAST — CodeQL
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript, python    # adjust for your languages
          queries: security-extended       # more thorough than default

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:javascript"
          upload: true    # uploads SARIF to GitHub Security tab

  # ─────────────────────────────────────────────
  # Job 3: Docker Image Scanning with Trivy
  # ─────────────────────────────────────────────
  image-scan:
    name: Docker Image Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t my-app:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: my-app:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          exit-code: 1                    # fail if vulnerabilities found
          severity: CRITICAL,HIGH
          ignore-unfixed: true            # ignore vulnerabilities with no fix yet

      - name: Upload Trivy results to GitHub Security tab
        if: always()                      # upload even if scan failed
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
          category: container-scanning

  # ─────────────────────────────────────────────
  # Job 4: SBOM Generation (on release tags only)
  # ─────────────────────────────────────────────
  generate-sbom:
    name: Generate SBOM
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
      - uses: actions/checkout@v4

      - name: Generate SBOM with Syft
        uses: anchore/sbom-action@v0
        with:
          image: my-app:${{ github.sha }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Upload SBOM as release asset
        uses: softprops/action-gh-release@v2
        with:
          files: sbom.spdx.json

      - name: Attest SBOM (sign with Sigstore)
        uses: actions/attest-sbom@v1
        with:
          subject-name: my-app
          subject-digest: ${{ steps.build.outputs.digest }}
          sbom-path: sbom.spdx.json
          push-to-registry: true
```

**What each tool provides:**

| Tool | Purpose | Output |
|---|---|---|
| `dependency-review-action` | Block PRs with known CVEs in dependencies | PR comment + fail check |
| CodeQL | Find security bugs in source code (SQLi, XSS, etc.) | SARIF → GitHub Security tab |
| Trivy | Scan Docker images for OS + library CVEs | SARIF → GitHub Security tab |
| Syft + attest-sbom | List all software components + sign with Sigstore | SBOM attached to release |

> 💡 **Interview tip:** The **SARIF format** (Static Analysis Results Interchange Format) is the key to getting scan results into the GitHub Security tab — any tool that outputs SARIF can integrate with GitHub Advanced Security. Mention that storing results in the Security tab creates a **persistent audit trail** of what vulnerabilities existed when each release was made, which is important for SOC2/compliance. Also mention `ignore-unfixed: true` in Trivy — without this, you will get failures for vulnerabilities that have no available fix, which is noise that frustrates developers.

---

### Q213 — Terraform | Conceptual | Advanced

> Explain **Terraform `locals`** vs **`variables`** vs **`outputs`**. When would you use each?
>
> What is the difference between **input variables** with `default` values and **local values**? Write a real-world example where using `locals` to compute derived values makes the configuration significantly cleaner than repeating expressions.

📁 **Reference:** `nawab312/Terraform` — locals, variables, outputs, expressions sections

#### Key Points to Cover in Your Answer:

**Variables (`variable`) — external inputs:**
```hcl
# Defined by: operator at runtime (CLI, tfvars, environment)
# Purpose: parameters that CHANGE between environments
# Can be overridden by user

variable "environment" {
  type        = string
  description = "Deployment environment"
  default     = "dev"                    # default value (optional)
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod"
  }
}

variable "instance_count" {
  type    = number
  default = 2
}
```

**Locals (`locals`) — computed internal values:**
```hcl
# Defined by: configuration author
# Purpose: computed values, reduce repetition, complex expressions
# CANNOT be overridden by user

locals {
  # Derived from variables — computed once, reused everywhere
  name_prefix = "${var.project}-${var.environment}"
  is_prod     = var.environment == "prod"

  # Conditional sizing based on environment
  instance_type = local.is_prod ? "t3.xlarge" : "t3.micro"
  min_capacity  = local.is_prod ? 3 : 1

  # Common tags applied to ALL resources
  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "Terraform"
    CreatedAt   = formatdate("YYYY-MM-DD", timestamp())
  }
}
```

**Outputs (`output`) — exported values:**
```hcl
# Purpose: expose values for use by other modules or humans
# Available after apply via: terraform output

output "vpc_id" {
  value       = aws_vpc.main.id
  description = "ID of the created VPC"
}

output "db_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = true    # hide from CLI output, still accessible programmatically
}
```

**Real-world example — locals prevent repetition:**
```hcl
# WITHOUT locals — repeating same expression everywhere (BAD)
resource "aws_instance" "web" {
  tags = {
    Name        = "${var.project}-${var.environment}-web"
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "Terraform"
  }
}

resource "aws_db_instance" "main" {
  tags = {
    Name        = "${var.project}-${var.environment}-db"  # duplicated pattern
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "Terraform"
  }
}

# WITH locals — DRY, consistent, easy to update (GOOD)
locals {
  name_prefix  = "${var.project}-${var.environment}"
  common_tags  = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "Terraform"
  }
}

resource "aws_instance" "web" {
  tags = merge(local.common_tags, {Name = "${local.name_prefix}-web"})
}

resource "aws_db_instance" "main" {
  tags = merge(local.common_tags, {Name = "${local.name_prefix}-db"})
}
# Change common_tags once → applies everywhere
```

**Key difference: `variable` with default vs `local`:**
```hcl
# variable with default = CAN be overridden by user
variable "instance_type" {
  default = "t3.micro"    # user can change via tfvars or CLI
}

# local = CANNOT be overridden, computed from other values
locals {
  instance_type = var.environment == "prod" ? "t3.xlarge" : "t3.micro"
  # Logic lives here, user cannot override it
}
```

> 💡 **Interview tip:** The most important distinction is: **variables are for things that change between runs** (environment name, region, team name), while **locals are for derived logic** that should always be computed consistently from those variables. Putting business logic (like "prod = large instances") in locals rather than variables ensures that logic cannot be accidentally bypassed by passing a different value at runtime.

---

### Q214 — Prometheus | Troubleshooting | Advanced

> Your Prometheus is showing `context deadline exceeded` errors when scraping some targets, and those targets show as `DOWN` in the targets page. The services are actually running and healthy.
>
> What causes scrape timeouts in Prometheus? Walk me through diagnosing whether the issue is in Prometheus configuration, network, or the target itself, and how to fix it.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus` — scrape configuration, timeouts, target health sections

#### Key Points to Cover in Your Answer:

**What `context deadline exceeded` means:**
- Prometheus tried to scrape the target
- The target did not respond within `scrape_timeout` (default: 10 seconds)
- Prometheus marked target as DOWN
- Target may actually be running but its `/metrics` endpoint is too slow

**Step 1 — Verify target is actually healthy:**
```bash
# Test the /metrics endpoint directly (from Prometheus pod or same network)
kubectl exec -it prometheus-pod -- \
  curl -v --max-time 15 http://payment-service:8080/metrics

# Check response time
time curl http://payment-service:8080/metrics | wc -l
# If takes > 10 seconds → metrics endpoint is too slow
```

**Step 2 — Check metric volume:**
```bash
# Count number of metrics the target exposes
curl -s http://payment-service:8080/metrics | wc -l
# If > 100,000 lines → cardinality explosion
# Prometheus takes too long to parse → timeout

# Check for high cardinality metrics
curl -s http://payment-service:8080/metrics | \
  grep -v "^#" | cut -d'{' -f1 | sort | uniq -c | sort -rn | head -20
```

**Step 3 — Check Prometheus scrape config:**
```yaml
# prometheus.yml
scrape_configs:
  - job_name: payment-service
    scrape_interval: 15s
    scrape_timeout: 10s     # MUST be less than scrape_interval

    # If target is slow, increase timeout
    scrape_timeout: 30s     # increase if metrics endpoint is legitimately slow
    scrape_interval: 60s    # must be larger than scrape_timeout
```

**Common causes and fixes:**

| Cause | Evidence | Fix |
|---|---|---|
| Metrics endpoint too slow | curl takes > 10s | Increase `scrape_timeout` |
| High cardinality explosion | Millions of metric lines | Fix application metric labels |
| Network latency | curl slow, target healthy | Check network path, DNS |
| Target CPU throttled | High CPU on target pod | Increase CPU limits |
| TLS handshake slow | HTTPS metrics endpoint | Pre-warm connection or check cert |
| Missing NetworkPolicy egress | Works from inside pod, not from Prometheus | Fix NetworkPolicy |

**Fix high cardinality (root cause in application):**
```python
# BAD — user_id label creates millions of unique series
request_count = Counter('requests_total', 'Total requests',
                        ['endpoint', 'user_id'])  # user_id = high cardinality!

# GOOD — only use low-cardinality labels
request_count = Counter('requests_total', 'Total requests',
                        ['endpoint', 'status_code'])  # bounded cardinality
```

**Fix in Prometheus config (drop high cardinality at scrape time):**
```yaml
# Drop the high-cardinality label before storing
metric_relabel_configs:
- source_labels: [__name__]
  regex: 'requests_total_with_user_id'
  action: drop                # drop the entire metric
# OR
- regex: 'user_id'
  action: labeldrop           # drop just the label
```

> 💡 **Interview tip:** **Cardinality explosion** causing scrape timeouts is one of the most common Prometheus production problems. A single metric with a `user_id` or `request_id` label can create millions of time series, causing Prometheus to use gigabytes of memory and timeout on scrapes. The fix must happen in the application — never use high-cardinality values as metric labels. The rule of thumb: a label should have fewer than 100 unique values.

---

### Q215 — AWS | Conceptual | Advanced

> Explain **AWS Spot Instances** in depth. What is the **Spot interruption** process — how much notice do you get and what happens to the instance?
>
> How would you design an application on Kubernetes (EKS) to **handle Spot interruptions gracefully**? What is the role of **interruption handlers** like `aws-node-termination-handler`?

📁 **Reference:** `nawab312/AWS` and `nawab312/Kubernetes` — Spot Instances, interruption handling, cost optimization sections

#### Key Points to Cover in Your Answer:

**Spot Instance basics:**
- AWS sells **spare EC2 capacity** at 70-90% discount vs On-Demand
- AWS can **reclaim** the instance with **2-minute notice** when capacity is needed
- Price fluctuates based on supply/demand (Spot price)
- Best for: fault-tolerant, stateless, interruptible workloads

**Spot interruption process:**
```
1. AWS decides to reclaim the instance
2. Instance Metadata Service (IMDS) gets the interruption notice
   → GET http://169.254.169.254/latest/meta-data/spot/interruption-action
3. EventBridge event fired: EC2 Spot Instance Interruption Warning
4. ⏱️ 2 MINUTES until instance is stopped/terminated
5. Instance terminates (or hibernates if configured)
```

**Detecting interruption in application:**
```bash
# Poll IMDS every 5 seconds in application startup
while true; do
  ACTION=$(curl -s --max-time 1 \
    http://169.254.169.254/latest/meta-data/spot/interruption-action)
  if [ ! -z "$ACTION" ]; then
    echo "Spot interruption notice received! Action: $ACTION"
    # Trigger graceful shutdown
    kill -SIGTERM $(pgrep myapp)
    break
  fi
  sleep 5
done
```

**`aws-node-termination-handler` (NTH) for EKS:**
```yaml
# NTH DaemonSet runs on every node
# Watches for Spot interruption notices via IMDS OR EventBridge
# When interruption detected:
# 1. Cordons the node (no new pods scheduled)
# 2. Drains the node (evicts pods gracefully with 2-min timeout)
# 3. Pods rescheduled on healthy nodes
# 4. Node terminates cleanly

# Install via Helm
helm install aws-node-termination-handler \
  eks/aws-node-termination-handler \
  --namespace kube-system \
  --set enableSpotInterruptionDraining=true \
  --set enableScheduledEventDraining=true \
  --set enableRebalanceMonitoring=true
```

**Designing for Spot resilience:**

```yaml
# 1. Use Karpenter with multiple instance families
# → More Spot capacity pools = lower interruption probability
spec:
  requirements:
  - key: karpenter.k8s.aws/instance-family
    operator: In
    values: ["m5", "m5a", "m6i", "m7i", "c5", "c6i"]  # 6 families = much better availability

# 2. Set proper terminationGracePeriodSeconds
spec:
  terminationGracePeriodSeconds: 90  # less than 2 min notice
  containers:
  - name: myapp
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 30"]  # drain connections

# 3. PodDisruptionBudget to ensure minimum availability
spec:
  minAvailable: 2   # always keep 2 pods running during drain

# 4. Use On-Demand for critical components
# Karpenter: label critical pods with on-demand preference
nodeSelector:
  karpenter.sh/capacity-type: on-demand
```

> 💡 **Interview tip:** The **2-minute notice** is often misunderstood — mention that it is actually the MINIMUM notice. In practice, AWS often gives more time because they try to reclaim capacity gracefully. However, always design for exactly 2 minutes. Also mention **Spot Rebalance Recommendations** (a newer feature) — AWS sends a rebalance signal BEFORE the interruption signal when it detects capacity risk, giving you more time to proactively migrate workloads.

---

### Q216 — Kubernetes | Scenario-Based | Advanced

> You need to implement **Kubernetes `NetworkPolicy`** for a 3-tier application:
> - **Frontend** (namespace: `web`) — only accessible from internet via Ingress
> - **Backend API** (namespace: `api`) — only accessible from frontend
> - **Database** (namespace: `db`) — only accessible from backend API
> - **Monitoring** (namespace: `monitoring`) — needs to scrape metrics from all tiers
>
> Write all required NetworkPolicies for complete isolation with the minimum required access.

📁 **Reference:** `nawab312/Kubernetes` → `03_NETWORKING.md` and `07_SECURITY.md` — NetworkPolicy, multi-tier isolation sections

#### Key Points to Cover in Your Answer:

**First — label all namespaces (required for namespaceSelector):**
```bash
kubectl label namespace web name=web
kubectl label namespace api name=api
kubectl label namespace db name=db
kubectl label namespace monitoring name=monitoring
kubectl label namespace ingress-nginx name=ingress-nginx
```

**Frontend NetworkPolicy (namespace: web):**
```yaml
# Allow ingress from Ingress Controller only
# Allow egress to backend API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: web
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx    # only Ingress Controller can reach frontend
    ports:
    - port: 3000
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: api
      podSelector:
        matchLabels:
          tier: backend
    ports:
    - port: 8080
  - to:    # allow DNS
    - namespaceSelector: {}
    ports:
    - port: 53
      protocol: UDP
```

**Backend API NetworkPolicy (namespace: api):**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: api
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes: [Ingress, Egress]
  ingress:
  # From frontend
  - from:
    - namespaceSelector:
        matchLabels:
          name: web
      podSelector:
        matchLabels:
          tier: frontend
    ports:
    - port: 8080
  # From monitoring (metrics scraping)
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - port: 9090    # metrics port
  egress:
  # To database
  - to:
    - namespaceSelector:
        matchLabels:
          name: db
      podSelector:
        matchLabels:
          tier: database
    ports:
    - port: 5432
  - to:    # DNS
    - namespaceSelector: {}
    ports:
    - port: 53
      protocol: UDP
```

**Database NetworkPolicy (namespace: db):**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: db
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes: [Ingress, Egress]
  ingress:
  # Only backend API can reach database
  - from:
    - namespaceSelector:
        matchLabels:
          name: api
      podSelector:
        matchLabels:
          tier: backend
    ports:
    - port: 5432
  # Monitoring metrics
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - port: 9187   # postgres-exporter port
  egress: []        # database should not initiate connections
```

**Monitoring NetworkPolicy (namespace: monitoring):**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: monitoring-egress
  namespace: monitoring
spec:
  podSelector:
    matchLabels:
      app: prometheus
  policyTypes: [Egress]
  egress:
  # Scrape all namespaces on metrics ports
  - to:
    - namespaceSelector: {}    # all namespaces
    ports:
    - port: 9090    # generic metrics
    - port: 9187    # postgres-exporter
    - port: 8080    # app metrics
  - to:
    - namespaceSelector: {}
    ports:
    - port: 53
      protocol: UDP
```

> 💡 **Interview tip:** The most common mistake is using separate `from` list items (which creates OR logic) instead of combined selectors under the same item (which creates AND logic). For example, `namespaceSelector` and `podSelector` under the SAME `-from` item means "pods with label X in namespace Y" — this is what you want for security. Separate items would mean "any pod in namespace Y OR any pod with label X anywhere" — which is too permissive. Always double-check the YAML indentation when writing NetworkPolicies.

---

### Q217 — ArgoCD | Troubleshooting | Advanced

> ArgoCD is syncing an application but the sync keeps **failing with `ComparisonError`**. The error message says:
>
> `rpc error: code = Unknown desc = Manifest generation error (cached): failed to generate manifest`
>
> What are all the possible causes of manifest generation errors in ArgoCD, and how would you diagnose and fix each one?

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — manifest generation, Helm, Kustomize errors sections

#### Key Points to Cover in Your Answer:

**What manifest generation does:**
ArgoCD renders final Kubernetes manifests from source (raw YAML, Helm charts, Kustomize) before comparing with live cluster state. If rendering fails → `ComparisonError`

**Diagnosis — force refresh to clear cached error:**
```bash
# Clear the cached error and force re-generation
argocd app get <app-name> --refresh

# Or use hard refresh
argocd app get <app-name> --hard-refresh

# Check application events
kubectl describe application <app-name> -n argocd
```

**Cause 1 — Helm chart values error:**
```bash
# Test Helm rendering locally
helm template my-app ./charts/my-app \
  -f values/production.yaml \
  --debug

# Common errors:
# "required value not provided" → missing required value in values.yaml
# "template: ... :XX:XX: executing ... error calling ..." → template logic error
# "Error: found in Chart.yaml, but missing in charts/ directory" → missing subchart
```

**Cause 2 — Helm dependency not downloaded:**
```bash
# ArgoCD needs chart dependencies
helm dependency update ./charts/my-app
# This creates/updates Chart.lock and downloads dependencies to charts/

# In ArgoCD: enable "Use Helm Dependencies" or commit Chart.lock
```

**Cause 3 — Kustomize error:**
```bash
# Test Kustomize rendering locally
kubectl kustomize ./overlays/production

# Common errors:
# "no such file or directory" → missing base file referenced in kustomization.yaml
# "JSON6902 patch target not found" → patch targeting non-existent resource
# "vars are not allowed" → using old vars syntax removed in newer kustomize
```

**Cause 4 — Invalid YAML in manifests:**
```bash
# Validate YAML syntax
kubectl apply --dry-run=client -f ./manifests/

# Check specific file
python3 -c "import yaml; yaml.safe_load(open('deployment.yaml'))"
```

**Cause 5 — ArgoCD repo-server cannot access Git:**
```bash
# Check repo-server logs
kubectl logs -n argocd deployment/argocd-repo-server --tail=50

# Check if SSH key / token for private repo is still valid
argocd repo list
argocd repo get https://github.com/myorg/myapp
```

**Cause 6 — Plugin/tool version mismatch:**
```bash
# ArgoCD uses specific versions of Helm/Kustomize
kubectl exec -n argocd deploy/argocd-repo-server -- helm version
kubectl exec -n argocd deploy/argocd-repo-server -- kustomize version

# If your chart requires Helm 3.14+ but ArgoCD has 3.12 → error
# Fix: upgrade ArgoCD or use Helm plugin override
```

**Cause 7 — Config Management Plugin (CMP) failure:**
```bash
# If using custom CMP (sidecar plugin)
kubectl logs -n argocd pod/argocd-repo-server -c <plugin-container>
# Check plugin script output
```

**Fix the cached error:**
```bash
# Sometimes ArgoCD caches the error even after fix
# Force hard refresh:
argocd app get <app-name> --hard-refresh

# Or via API:
curl -X POST https://argocd.example.com/api/v1/applications/<app>/refresh \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"force": true}'
```

> 💡 **Interview tip:** The **"(cached)"** in the error message is important — it means ArgoCD is showing a cached error from a previous attempt. After fixing the underlying issue, always do a **hard refresh** (`--hard-refresh`) to clear the cache. Regular refresh just reads the cache; hard refresh forces re-generation from source. Many engineers spend time trying to fix a problem that is already fixed but ArgoCD is still showing the cached error.

---

### Q218 — Linux / Bash | Conceptual | Advanced

> Explain **Linux `cgroups v1` vs `cgroups v2`**. What changed in v2 and why does it matter for Kubernetes?
>
> What **subsystems** does cgroups control? How does Kubernetes use cgroups to enforce **CPU and memory limits** on containers? What is the difference between CPU **`limits`** using CFS (Completely Fair Scheduler) throttling vs CPU **`requests`** using shares?

📁 **Reference:** `nawab312/DSA` → `Linux` and `nawab312/Kubernetes` → `13_DOCKER_CONTAINER_FUNDAMENTALS.md` — cgroups, CPU throttling sections

#### Key Points to Cover in Your Answer:

**cgroups (Control Groups) overview:**
- Kernel mechanism to **limit, account, and isolate** resource usage of process groups
- Used by Docker and Kubernetes to enforce container resource limits

**cgroups v1 subsystems:**
```
cpu       → CPU time allocation (shares and CFS quota)
cpuset    → Pin processes to specific CPU cores
memory    → Memory limits and OOM control
blkio     → Block I/O throttling
net_cls   → Network traffic classification
pids      → Process count limit
```

**cgroups v1 problems:**
- Multiple hierarchy trees — each subsystem has its own tree
- Inconsistent semantics between subsystems
- Complex interaction between multiple controllers
- Race conditions in memory accounting

**cgroups v2 improvements:**
```
Unified hierarchy — single tree for ALL subsystems
Better memory accounting — includes page cache in limits
cpu.max replaces cpu.cfs_quota_us + cpu.cfs_period_us
memory.oom_group — kill all processes in group when OOM
io.max replaces blkio limits
Better delegation to non-root users (rootless containers)
```

**Why it matters for Kubernetes:**
- Kubernetes 1.25+ enables cgroups v2 by default (if kernel supports it)
- cgroups v2 enables **MemoryQoS** — more precise memory limits
- Fixes issues with page cache not being counted in memory limits (v1 problem)
- Required for `memory.oom_group` (kill entire pod on OOM, not just one container)

**CPU limits vs requests (CFS-based):**
```
CPU REQUESTS (shares-based):
  → Uses cpu.shares in cgroups v1 (cpu.weight in v2)
  → Proportional sharing when CPU is CONTENDED
  → 500m request + 1000m request = 1:2 ratio gets CPU
  → If CPU is idle: container can use MORE than requested (burstable)
  → Used by scheduler to PLACE pods on nodes

CPU LIMITS (CFS quota-based):
  → Uses cpu.cfs_quota_us + cpu.cfs_period_us in v1 (cpu.max in v2)
  → Hard cap — container is THROTTLED when it exceeds the limit
  → Even if node CPU is idle: container cannot exceed its limit
  → Example: 500m limit = container gets max 50ms out of every 100ms period
```

**Real-world implications:**
```bash
# Container with limits: cpu: "500m"
# Under the hood in cgroups v1:
cat /sys/fs/cgroup/cpu/kubepods/.../cpu.cfs_quota_us   # 50000 (50ms)
cat /sys/fs/cgroup/cpu/kubepods/.../cpu.cfs_period_us  # 100000 (100ms)
# = 50000/100000 = 0.5 cores = 500m

# In cgroups v2:
cat /sys/fs/cgroup/kubepods/.../cpu.max    # 50000 100000

# Check if CPU throttling is happening:
cat /sys/fs/cgroup/cpu/kubepods/.../cpu.stat | grep throttled_time
# If throttled_time is growing → app is hitting CPU limit
```

> 💡 **Interview tip:** **CPU throttling is the most common invisible performance problem** in Kubernetes. An application can appear healthy (low CPU usage in `kubectl top`) but still be throttled because it bursts above its limit for short periods. The key metric is `container_cpu_cfs_throttled_seconds_total` in Prometheus — if this is non-zero and growing, your application is being CPU throttled even though average usage appears fine. The fix is either increasing CPU limits or optimizing the application to avoid bursty CPU usage.

---

### Q219 — AWS | Scenario-Based | Advanced

> You need to implement **blue/green deployment** on AWS ECS using **CodeDeploy**. The deployment must:
> - Shift traffic **gradually**: 10% → 30% → 100% over 30 minutes
> - Automatically **rollback** if CloudWatch alarms trigger
> - Have a **bake time** of 10 minutes at each traffic shift
> - Send **SNS notifications** at each deployment lifecycle event
>
> Walk me through the complete setup.

📁 **Reference:** `nawab312/AWS` — ECS, CodeDeploy, blue/green, traffic shifting sections

#### Key Points to Cover in Your Answer:

**How ECS Blue/Green with CodeDeploy works:**
```
Blue environment:  current task set (receiving 100% traffic)
Green environment: new task set (new version, receives 0% traffic initially)

ALB has two target groups:
- Blue target group → current tasks
- Green target group → new tasks

CodeDeploy:
1. Creates new (green) task set with new image
2. Shifts traffic incrementally from blue → green
3. After validation period: terminates blue tasks
```

**Step 1 — Prerequisites:**
```bash
# Create two ALB target groups
aws elbv2 create-target-group --name payment-blue --protocol HTTP --port 8080 ...
aws elbv2 create-target-group --name payment-green --protocol HTTP --port 8080 ...

# ECS Service must use CODE_DEPLOY deployment controller
aws ecs create-service \
  --service-name payment-service \
  --deployment-controller type=CODE_DEPLOY \
  --load-balancers targetGroupArn=<blue-tg-arn>,containerName=payment,containerPort=8080
```

**Step 2 — CodeDeploy Application and Deployment Group:**
```bash
# Create CodeDeploy application
aws deploy create-application \
  --application-name payment-service \
  --compute-platform ECS

# Create deployment group with traffic shifting config
aws deploy create-deployment-group \
  --application-name payment-service \
  --deployment-group-name production \
  --service-role-arn <codedeploy-role-arn> \
  --deployment-config-name CodeDeployDefault.ECSLinear10PercentEvery3Minutes \
  --ecs-services clusterName=production,serviceName=payment-service \
  --load-balancer-info targetGroupPairInfoList=[{
    targetGroups=[{name=payment-blue},{name=payment-green}],
    prodTrafficRoute={listenerArns=[<listener-arn>]}
  }] \
  --auto-rollback-configuration enabled=true,events=DEPLOYMENT_FAILURE,DEPLOYMENT_STOP_ON_ALARM \
  --alarm-configuration enabled=true,alarms=[{name=payment-error-rate-alarm}]
```

**Step 3 — Custom traffic shifting (10→30→100 in 30 min):**
```bash
# Create custom deployment configuration
aws deploy create-deployment-config \
  --deployment-config-name payment-custom-traffic-shift \
  --compute-platform ECS \
  --traffic-routing-config type=TimeBasedLinear,timeBasedLinear={
    linearPercentage=10,    # start at 10%
    linearInterval=10       # shift 10% every 10 minutes → 10%→20%→30%...100%
  }
```

**Step 4 — AppSpec file (defines the deployment):**
```yaml
# appspec.yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:us-east-1:123456:task-definition/payment:NEW_VERSION"
        LoadBalancerInfo:
          ContainerName: "payment"
          ContainerPort: 8080
        PlatformVersion: "LATEST"

Hooks:
  - BeforeInstall: "LambdaFunctionToValidateBeforeInstall"
  - AfterInstall: "LambdaFunctionToValidateAfterInstall"
  - AfterAllowTestTraffic: "LambdaFunctionRunIntegrationTests"
  - BeforeAllowTraffic: "LambdaFunctionFinalValidation"
  - AfterAllowTraffic: "LambdaFunctionPostDeploymentChecks"
```

**Step 5 — CloudWatch alarms for auto-rollback:**
```bash
# Create alarm on error rate
aws cloudwatch put-metric-alarm \
  --alarm-name payment-error-rate-alarm \
  --metric-name HTTPCode_Target_5XX_Count \
  --namespace AWS/ApplicationELB \
  --dimensions Name=LoadBalancer,Value=<alb-name> \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions <sns-topic-arn>

# CodeDeploy will automatically rollback if this alarm fires during deployment
```

**Step 6 — SNS notifications:**
```bash
# CodeDeploy sends to SNS on lifecycle events
aws deploy update-deployment-group \
  --application-name payment-service \
  --current-deployment-group-name production \
  --trigger-configurations \
    triggerName=deployment-notification,\
    triggerTargetArn=<sns-topic-arn>,\
    triggerEvents=DeploymentStart,DeploymentSuccess,DeploymentFailure,DeploymentRollback
```

> 💡 **Interview tip:** The most important safety feature to mention is the **automatic rollback on CloudWatch alarm** — CodeDeploy monitors your alarms throughout the deployment and rolls back immediately if an alarm fires. This means you don't need humans watching dashboards during deployment. Also mention the **bake time** concept — even after 100% traffic is shifted to green, CodeDeploy waits for the configured bake time before terminating the blue tasks, giving you time to detect post-deployment issues.

---

### Q220 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `CRD` (Custom Resource Definition)** versioning. What is **conversion webhooks** and why are they needed?
>
> If you have a CRD with `v1alpha1` in production and want to introduce `v1beta1` with schema changes, how do you handle the migration without breaking existing resources? What is the **`served`** and **`storage`** version concept?

📁 **Reference:** `nawab312/Kubernetes` → `10_CRDs_EXTENSIBILITY.md` — CRD versioning, conversion webhooks sections

#### Key Points to Cover in Your Answer:

**Why CRD versioning is needed:**
- As custom resources evolve, their schema changes
- Existing resources (stored in etcd) use old schema
- New clients use new schema
- Need to support both simultaneously during migration

**`served` vs `storage` versions:**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.com
spec:
  versions:
  - name: v1alpha1
    served: true        # API accepts requests using this version
    storage: false      # NOT the version stored in etcd
    schema:
      openAPIV3Schema:
        properties:
          spec:
            properties:
              replicas: {type: integer}

  - name: v1beta1
    served: true        # also accessible
    storage: true       # THIS version is stored in etcd
    schema:
      openAPIV3Schema:
        properties:
          spec:
            properties:
              replicas: {type: integer}
              backupEnabled: {type: boolean}   # new field in v1beta1
```

**`served: true, storage: false` → API accessible but converts to storage version**
**`served: false, storage: false` → deprecated, not accessible**
**`served: true, storage: true` → the canonical stored version**

**Conversion Webhook:**
```yaml
# Without conversion webhook:
# Client reads v1alpha1 resource → etcd has v1beta1 → no automatic conversion → error

# With conversion webhook:
# Client reads v1alpha1 → Kubernetes calls webhook → webhook converts v1beta1 → v1alpha1 response → client happy

spec:
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: ["v1"]
      clientConfig:
        service:
          namespace: default
          name: crd-conversion-webhook
          path: /convert
```

**Conversion webhook logic:**
```go
// Webhook receives: object in source version
// Must return: object in target version
func convertDatabase(src *v1alpha1.Database) *v1beta1.Database {
    return &v1beta1.Database{
        Spec: v1beta1.DatabaseSpec{
            Replicas:      src.Spec.Replicas,
            BackupEnabled: false,  // default for existing resources
        },
    }
}
```

**Migration process (v1alpha1 → v1beta1):**
```bash
# Step 1: Add v1beta1 to CRD with served=true, storage=false
#         Keep v1alpha1 with served=true, storage=true

# Step 2: Deploy conversion webhook

# Step 3: Migrate storage to v1beta1:
# storage: true → v1beta1, storage: false → v1alpha1

# Step 4: Existing resources in etcd (v1alpha1) are re-encoded as v1beta1
#         via: kubectl get databases --all-namespaces -o yaml | kubectl apply -f -
# OR via Kubernetes storage migration controller

# Step 5: After all resources migrated, set v1alpha1 served=false (deprecate)
```

> 💡 **Interview tip:** The **storage version** concept is the most important — there can only be ONE storage version at a time, but multiple served versions. All reads/writes to etcd use the storage version; conversion webhooks handle translating between API versions and storage version. The migration process requires: (1) add new version with conversion webhook, (2) migrate storage, (3) deprecate old version — all while maintaining zero downtime for existing users of the old API version.

---

### Q221 — Grafana | Scenario-Based | Advanced

> You are asked to implement **Grafana as Code** — all dashboards and datasources should be version controlled in Git and automatically provisioned.
>
> Walk me through:
> 1. Setting up **Grafana provisioning** via config files
> 2. Managing dashboards as **JSON in Git**
> 3. Using **Grafonnet** (Jsonnet library) to generate dashboards programmatically
> 4. Automating dashboard deployment via **CI/CD pipeline**

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana` — provisioning, Grafana as code sections

#### Key Points to Cover in Your Answer:

**Step 1 — Grafana provisioning directory structure:**
```
grafana/
├── provisioning/
│   ├── datasources/
│   │   └── prometheus.yaml      # auto-loaded datasources
│   ├── dashboards/
│   │   └── dashboards.yaml      # dashboard provider config
│   └── alerting/
│       └── policies.yaml        # alert routing
└── dashboards/
    ├── team-a/
    │   ├── service-overview.json
    │   └── deployment-history.json
    └── team-b/
        └── cost-dashboard.json
```

**Step 2 — Dashboard provider config:**
```yaml
# provisioning/dashboards/dashboards.yaml
apiVersion: 1
providers:
  - name: team-a
    folder: Team A
    folderUid: team-a-folder
    type: file
    disableDeletion: true
    updateIntervalSeconds: 30
    allowUiUpdates: false      # prevent UI edits (code is source of truth)
    options:
      path: /var/lib/grafana/dashboards/team-a

  - name: team-b
    folder: Team B
    folderUid: team-b-folder
    type: file
    disableDeletion: true
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards/team-b
```

**Step 3 — Grafonnet for programmatic dashboard generation:**
```jsonnet
// dashboards/service-overview.jsonnet
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local graphPanel = grafana.graphPanel;
local prometheus = grafana.prometheus;

// Define reusable panel templates
local requestRatePanel(service) =
  graphPanel.new(
    title='Request Rate — ' + service,
    datasource='Prometheus',
  )
  .addTarget(
    prometheus.target(
      'sum(rate(http_requests_total{service="' + service + '"}[5m]))',
      legendFormat='RPS'
    )
  );

// Generate dashboards for multiple services
local services = ['payment', 'auth', 'user'];

// Create one dashboard per service
{
  ['dashboards/' + service + '-overview.json']:
    dashboard.new(service + ' Service Overview')
    .addPanel(requestRatePanel(service), gridPos={h:8, w:12, x:0, y:0})
    .addPanel(errorRatePanel(service), gridPos={h:8, w:12, x:12, y:0})
  for service in services
}
```

**Build Grafonnet dashboards:**
```bash
# Install jsonnet and grafonnet
go install github.com/google/go-jsonnet/cmd/jsonnet@latest

# Generate JSON dashboards from Jsonnet
jsonnet -J grafonnet-lib dashboards/service-overview.jsonnet \
  -m dashboards/generated/

# Output: dashboards/generated/payment-overview.json
#         dashboards/generated/auth-overview.json
#         dashboards/generated/user-overview.json
```

**Step 4 — CI/CD pipeline for dashboard deployment:**
```yaml
# .github/workflows/grafana-deploy.yml
name: Deploy Grafana Dashboards

on:
  push:
    branches: [main]
    paths: ['grafana/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install jsonnet
        run: go install github.com/google/go-jsonnet/cmd/jsonnet@latest

      - name: Generate dashboards from Jsonnet
        run: |
          for file in grafana/dashboards/*.jsonnet; do
            jsonnet -J grafonnet-lib "$file" -m grafana/dashboards/generated/
          done

      - name: Validate dashboard JSON
        run: |
          for file in grafana/dashboards/**/*.json; do
            python3 -c "import json; json.load(open('$file'))" || exit 1
          done

      - name: Deploy to Grafana via API
        run: |
          for file in grafana/dashboards/**/*.json; do
            curl -X POST \
              -H "Authorization: Bearer ${{ secrets.GRAFANA_API_KEY }}" \
              -H "Content-Type: application/json" \
              -d @"$file" \
              ${{ vars.GRAFANA_URL }}/api/dashboards/db
          done

      - name: Verify deployment
        run: |
          # Check all dashboards are accessible
          curl -s -H "Authorization: Bearer ${{ secrets.GRAFANA_API_KEY }}" \
            ${{ vars.GRAFANA_URL }}/api/search | jq '.[] | .title'
```

**Kubernetes deployment with ConfigMap:**
```yaml
# For Kubernetes-deployed Grafana:
# Mount dashboards as ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  labels:
    grafana_dashboard: "1"   # Grafana sidecar watches for this label
data:
  payment-overview.json: |
    { ... dashboard JSON ... }
```

> 💡 **Interview tip:** The **`allowUiUpdates: false`** setting in the provisioning config is critical for true GitOps — without it, someone can edit a dashboard in the UI and the file-based config will overwrite their changes on the next sync. Setting it to `false` forces all changes to go through Git, maintaining code as the single source of truth. Also mention **Grafonnet** vs raw JSON — Grafonnet lets you create dashboard templates with variables and loops, which is much more maintainable than duplicating 500-line JSON files for each service.

---

### Q222 — Terraform | Scenario-Based | Advanced

> You are building a **Terraform module** for deploying a containerized microservice on ECS Fargate that needs to be **production-ready**. The module must support:
> - Auto Scaling based on CPU and custom metrics
> - Service discovery via **AWS Cloud Map**
> - **Secret injection** from Secrets Manager at runtime
> - **Structured logging** to CloudWatch with log retention
> - **ALB integration** with health checks
>
> Design the module interface (variables and outputs) and key resource blocks.

📁 **Reference:** `nawab312/Terraform` and `nawab312/AWS` — ECS module design, Cloud Map, Secrets Manager sections

#### Key Points to Cover in Your Answer:

**Module interface (variables.tf):**
```hcl
# Required inputs
variable "service_name" {
  type        = string
  description = "Name of the microservice"
}

variable "container_image" {
  type        = string
  description = "Docker image URI (e.g. 123456.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0)"
}

variable "container_port" {
  type        = number
  description = "Port the container listens on"
}

# Optional with sensible defaults
variable "cpu" {
  type        = number
  default     = 256
  description = "CPU units (256 = 0.25 vCPU)"
}

variable "memory" {
  type        = number
  default     = 512
  description = "Memory in MiB"
}

variable "desired_count" {
  type    = number
  default = 2
}

variable "secrets" {
  type = list(object({
    name      = string   # env var name in container
    valueFrom = string   # Secrets Manager ARN or SSM path
  }))
  default     = []
  description = "Secrets to inject from Secrets Manager"
}

variable "autoscaling" {
  type = object({
    min_capacity        = number
    max_capacity        = number
    cpu_target          = number
    scale_in_cooldown   = number
    scale_out_cooldown  = number
  })
  default = {
    min_capacity        = 2
    max_capacity        = 10
    cpu_target          = 60
    scale_in_cooldown   = 300
    scale_out_cooldown  = 60
  }
}

variable "health_check" {
  type = object({
    path                = string
    healthy_threshold   = number
    unhealthy_threshold = number
    interval            = number
  })
  default = {
    path                = "/healthz"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
  }
}

variable "log_retention_days" {
  type    = number
  default = 30
}
```

**Key resource blocks (main.tf):**
```hcl
locals {
  full_name = "${var.environment}-${var.service_name}"
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "service" {
  name              = "/ecs/${local.full_name}"
  retention_in_days = var.log_retention_days
  tags              = local.common_tags
}

# ECS Task Definition with secret injection
resource "aws_ecs_task_definition" "service" {
  family                   = local.full_name
  cpu                      = var.cpu
  memory                   = var.memory
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  execution_role_arn       = aws_iam_role.execution.arn
  task_role_arn            = aws_iam_role.task.arn

  container_definitions = jsonencode([{
    name      = var.service_name
    image     = var.container_image
    essential = true

    portMappings = [{
      containerPort = var.container_port
      protocol      = "tcp"
    }]

    # Secrets injected at runtime from Secrets Manager
    secrets = var.secrets

    # Structured logging to CloudWatch
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = aws_cloudwatch_log_group.service.name
        awslogs-region        = data.aws_region.current.name
        awslogs-stream-prefix = "ecs"
      }
    }
  }])
}

# ECS Service with Cloud Map service discovery
resource "aws_ecs_service" "service" {
  name            = local.full_name
  cluster         = var.cluster_id
  task_definition = aws_ecs_task_definition.service.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"

  load_balancer {
    target_group_arn = aws_lb_target_group.service.arn
    container_name   = var.service_name
    container_port   = var.container_port
  }

  # Cloud Map service discovery
  service_registries {
    registry_arn = aws_service_discovery_service.service.arn
  }

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.service.id]
    assign_public_ip = false
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true   # auto-rollback on deployment failure
  }

  lifecycle {
    ignore_changes = [desired_count]  # managed by autoscaling
  }
}

# Auto Scaling
resource "aws_appautoscaling_target" "service" {
  service_namespace  = "ecs"
  resource_id        = "service/${var.cluster_name}/${local.full_name}"
  scalable_dimension = "ecs:service:DesiredCount"
  min_capacity       = var.autoscaling.min_capacity
  max_capacity       = var.autoscaling.max_capacity
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "${local.full_name}-cpu-scaling"
  service_namespace  = "ecs"
  resource_id        = aws_appautoscaling_target.service.resource_id
  scalable_dimension = "ecs:service:DesiredCount"
  policy_type        = "TargetTrackingScaling"

  target_tracking_scaling_policy_configuration {
    target_value       = var.autoscaling.cpu_target
    scale_in_cooldown  = var.autoscaling.scale_in_cooldown
    scale_out_cooldown = var.autoscaling.scale_out_cooldown

    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
  }
}
```

**Outputs (outputs.tf):**
```hcl
output "service_name" {
  value = aws_ecs_service.service.name
}

output "alb_dns_name" {
  value = aws_lb.service.dns_name
}

output "cloudwatch_log_group" {
  value = aws_cloudwatch_log_group.service.name
}

output "service_discovery_arn" {
  value = aws_service_discovery_service.service.arn
}

output "task_role_arn" {
  value       = aws_iam_role.task.arn
  description = "Attach additional policies to this role for AWS service access"
}
```

> 💡 **Interview tip:** The `deployment_circuit_breaker { rollback = true }` block is critical for production-ready ECS deployments — without it, a bad deployment keeps replacing healthy tasks with broken ones until the service is completely down. With circuit breaker enabled, ECS detects that tasks are consistently failing health checks and automatically rolls back to the previous task definition. Always include this in production ECS service definitions.

---

### Q223 — Git | Scenario-Based | Advanced

> Your team is adopting **trunk-based development** moving away from GitFlow. You have 20 developers who are used to long-lived feature branches.
>
> What changes to **Git workflow**, **CI/CD pipeline**, and **deployment strategy** are needed to make trunk-based development work safely? What is **feature flags** role in this transition and how do you implement them?

📁 **Reference:** `nawab312/CI_CD` → `Git` — trunk-based development, feature flags, CI/CD integration sections

#### Key Points to Cover in Your Answer:

**GitFlow vs Trunk-Based:**
```
GitFlow:
main ← develop ← feature/xxx (can live for months)
PRs merged infrequently → large diffs → painful merges

Trunk-Based:
main ← short-lived branches (hours to days, never weeks)
Small, frequent commits to main
Feature flags hide incomplete features
```

**Git workflow changes:**
```bash
# Rule 1: Branches live for < 2 days maximum
git checkout -b feature/add-payment-timeout
# ... small focused change ...
git push origin feature/add-payment-timeout
# Create PR → review → merge → delete branch
# All in the same day

# Rule 2: Rebase on main before merging (keep history linear)
git fetch origin
git rebase origin/main
git push --force-with-lease  # safe force push (only if no one else pushed)

# Rule 3: Squash to single commit per feature (clean history)
git rebase -i origin/main  # squash multiple WIP commits
```

**CI/CD pipeline changes:**
```yaml
# Every commit to main triggers deployment to production
# BUT: incomplete features are hidden by feature flags

on:
  push:
    branches: [main]   # ALL pushes to main → deploy to prod

jobs:
  test:      ...
  build:     needs: test
  deploy:    needs: build   # no manual approval needed (feature flags protect users)
```

**Feature flags implementation:**
```python
# Feature flag service (LaunchDarkly, AWS AppConfig, or simple DB)
import boto3, json

class FeatureFlags:
    def __init__(self):
        self.client = boto3.client('appconfig')

    def is_enabled(self, flag_name: str, user_id: str = None) -> bool:
        response = self.client.get_configuration(
            Application='myapp',
            Environment='production',
            Configuration='feature-flags',
            ClientId=user_id or 'anonymous'
        )
        flags = json.loads(response['Content'].read())
        return flags.get(flag_name, False)

ff = FeatureFlags()

# In application code:
@app.route('/checkout')
def checkout():
    if ff.is_enabled('new_payment_flow', user_id=current_user.id):
        return new_payment_flow()   # new incomplete feature (deployed but hidden)
    else:
        return old_payment_flow()   # current stable code
```

**Feature flags strategies:**
```
Percentage rollout:  enable for 1% → 10% → 50% → 100% of users
User targeting:      enable only for internal users / beta testers first
Kill switch:         disable immediately if issues found (no deployment needed)
Environment flags:   enabled in staging, disabled in prod
```

**Team practices needed:**
```
1. Short-lived branches → merge within 1 day
2. Every PR reviewed and merged same day
3. All new features wrapped in feature flags
4. Comprehensive automated tests (trunk-based relies on CI confidence)
5. Fast CI pipeline (< 10 minutes) — slow CI kills trunk-based
6. Feature flag cleanup → remove flags after 100% rollout
```

> 💡 **Interview tip:** The most important enabler of trunk-based development is **feature flags** — without them, you cannot deploy incomplete features to production. The key cultural shift is: "deployment ≠ release". You can deploy to production (feature hidden by flag) and release to users (turn on the flag) as separate events. This decoupling is what makes trunk-based development safe and is increasingly the approach used by high-performing engineering teams like Google, Facebook, and Netflix.

---

### Q224 — AWS + Kubernetes | Scenario-Based | Advanced

> Your EKS cluster is running in a **private VPC** with no public internet access. You need to:
> - Pull container images from **public ECR/Docker Hub**
> - Allow pods to call **external APIs** (payment gateway, email service)
> - Access **AWS services** (S3, Secrets Manager) without going through internet
> - Developers need to run **`kubectl` commands** from their laptops
>
> Design the complete **network architecture** to satisfy all requirements.

📁 **Reference:** `nawab312/AWS` and `nawab312/Kubernetes` — private EKS, VPC endpoints, NAT Gateway, bastion sections

#### Key Points to Cover in Your Answer:

**Architecture Overview:**
```
Internet
    ↓
Internet Gateway
    ↓
Public Subnet:
  - NAT Gateway (for pods → internet)
  - Bastion Host OR VPN endpoint (for developer kubectl access)

Private Subnet (EKS nodes):
  - Worker nodes
  - Pods

VPC Endpoints (private connectivity to AWS services):
  - ECR endpoint (image pulls stay within AWS network)
  - S3 endpoint (no internet, no NAT needed)
  - Secrets Manager endpoint
  - EKS endpoint
```

**Requirement 1: Pull images from public ECR / Docker Hub:**
```
Option A (NAT Gateway — simplest):
Pod → NAT Gateway → Internet → ECR/Docker Hub

Option B (ECR mirror — better for cost + control):
1. Create private ECR repository
2. Use ECR Pull Through Cache:
   aws ecr create-pull-through-cache-rule \
     --ecr-repository-prefix dockerhub \
     --upstream-registry-url registry-1.docker.io
3. Pods reference: <account>.dkr.ecr.us-east-1.amazonaws.com/dockerhub/nginx:latest
4. ECR pulls from Docker Hub on first request, caches for subsequent pulls
5. No NAT needed for cached images
```

**Requirement 2: Pods calling external APIs:**
```
NAT Gateway setup:
- Public subnet: NAT Gateway with Elastic IP
- Private subnet route table: 0.0.0.0/0 → NAT Gateway
- Pods get internet access via NAT (outbound only, not reachable from internet)

Security Group on pods:
- Outbound: allow TCP 443 to 0.0.0.0/0 (HTTPS to payment gateway)
- NO inbound from internet
```

**Requirement 3: Access AWS services without internet (VPC Endpoints):**
```bash
# S3 Gateway Endpoint (FREE)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-private-1 rtb-private-2

# ECR Interface Endpoints (saves NAT costs for ECR)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.ecr.api \
  --subnet-ids subnet-private-1 subnet-private-2 \
  --security-group-ids sg-vpce

aws ec2 create-vpc-endpoint \
  --service-name com.amazonaws.us-east-1.ecr.dkr  # for Docker layer pulls

# Secrets Manager Interface Endpoint
aws ec2 create-vpc-endpoint \
  --service-name com.amazonaws.us-east-1.secretsmanager

# EKS (so kubectl works from within VPC)
aws ec2 create-vpc-endpoint \
  --service-name com.amazonaws.us-east-1.eks

# Also needed for EKS cluster networking:
# EC2 endpoint, STS endpoint (for IRSA), CloudWatch Logs endpoint
```

**Requirement 4: Developer kubectl access from laptops:**
```
Option A — Bastion Host (simplest):
  Developer → SSH → Bastion in public subnet → kubectl to EKS API endpoint
  EKS API: privateEndpoint=true, publicEndpoint=false (or restricted CIDR)

Option B — AWS Systems Manager Session Manager (no SSH, no bastion):
  Developer → AWS CLI → SSM → private EC2 in VPC → kubectl
  No SSH keys, no open port 22, full audit trail

Option C — AWS Client VPN (best for teams):
  Developer laptop → Client VPN endpoint (in public subnet) → VPN tunnel → VPC
  Works like being inside the VPC
  kubectl commands go directly to private EKS API endpoint
  Requires: VPN client software + certificate authentication

# EKS API endpoint configuration:
aws eks update-cluster-config \
  --name my-cluster \
  --resources-vpc-config endpointPrivateAccess=true,endpointPublicAccess=false
  # or restrict public to office IP:
  # endpointPublicAccess=true,publicAccessCidrs=["203.0.113.5/32"]
```

**Complete security group design:**
```
EKS Node SG:
  Inbound: all from EKS Node SG (nodes talk to each other)
  Inbound: 443, 10250 from EKS Control Plane SG
  Outbound: all

VPC Endpoint SG:
  Inbound: 443 from EKS Node SG
  Outbound: all

Bastion SG:
  Inbound: 22 from developer IPs only
  Outbound: 443 to EKS API endpoint
```

> 💡 **Interview tip:** The **ECR Pull Through Cache** is an often-missed optimization — instead of routing Docker Hub pulls through NAT Gateway (which costs money per GB and is slow), you mirror images to ECR and pods pull from ECR via the ECR VPC endpoint (free, fast, within AWS network). Also mention that for EKS clusters, you need VPC endpoints for at least **ECR API, ECR DKR, S3, STS, and EC2** to function without internet access — missing any of these will break cluster functionality in unexpected ways.

---

### Q225 — All Topics | System Design | Advanced

> You are designing a **GitOps-based platform** for a large enterprise with:
> - **200 microservices** across 5 teams
> - **4 environments**: dev, staging, UAT, production
> - **Compliance requirement**: every production change must have an audit trail and require 2 approvals
> - **Security requirement**: no human has direct `kubectl` access to production
> - **Reliability requirement**: platform itself must be self-healing
>
> Design the complete GitOps platform covering: Git repository structure, ArgoCD setup, CI pipeline, secret management, access control, audit logging, and disaster recovery of the platform itself.

📁 **Reference:** All repositories — comprehensive enterprise GitOps platform design

#### Key Points to Cover in Your Answer:

**Git Repository Structure:**
```
repos:
1. app-repos/ (one per microservice — 200 repos)
   payment-service/
   ├── src/                  application code
   ├── Dockerfile
   └── .github/workflows/   CI pipeline (test, build, push image)

2. platform-config/ (single GitOps repo — source of truth)
   ├── apps/                 ArgoCD Application definitions
   │   ├── team-a/
   │   │   ├── payment-service.yaml
   │   │   └── auth-service.yaml
   │   └── team-b/
   ├── environments/
   │   ├── dev/              Helm values per environment
   │   │   └── payment-service/values.yaml
   │   ├── staging/
   │   ├── uat/
   │   └── production/       production values (requires 2 approvals to merge)
   ├── infrastructure/       cluster-level resources (namespaces, RBAC, network policies)
   └── platform/             ArgoCD, cert-manager, Prometheus operator
```

**ArgoCD Multi-Environment Setup:**
```yaml
# ApplicationSet — creates one app per environment × service combination
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: all-services
spec:
  generators:
  - matrix:
      generators:
      # All services from Git
      - git:
          repoURL: https://github.com/company/platform-config
          revision: main
          directories:
          - path: apps/*
      # All environments
      - list:
          elements:
          - env: dev
            cluster: https://dev-eks.company.com
          - env: staging
            cluster: https://staging-eks.company.com
          - env: uat
            cluster: https://uat-eks.company.com
          - env: production
            cluster: https://prod-eks.company.com

  template:
    metadata:
      name: '{{path.basename}}-{{env}}'
    spec:
      project: '{{env}}-project'    # ArgoCD project restricts which cluster/namespace
      source:
        repoURL: https://github.com/company/platform-config
        targetRevision: main
        path: 'apps/{{path.basename}}'
        helm:
          valueFiles:
          - environments/{{env}}/{{path.basename}}/values.yaml
      destination:
        server: '{{cluster}}'
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true    # self-healing for dev/staging/uat
        # production: no automated sync → manual sync required
```

**CI Pipeline (app-repo → platform-config):**
```yaml
# In each app-repo, GitHub Actions:
1. Run tests
2. Build Docker image
3. Push to ECR with SHA tag
4. Update platform-config/environments/dev/<service>/values.yaml
   → image.tag: abc1234
5. Create PR to platform-config (auto-merged for dev)
   → For staging: requires 1 approval
   → For production: requires 2 approvals + compliance check
```

**2-Approval Production Requirement:**
```yaml
# GitHub Branch Protection for platform-config main:
# Required reviews: 2
# Required status checks: compliance-check workflow
# Dismiss stale reviews: true
# Restrict who can push: only CI bot

# Compliance check workflow validates:
# - Change is in environments/production/ path
# - Changelog entry exists
# - Security scan passed
# - Load test results attached
```

**No kubectl Access to Production:**
```yaml
# Developers have NO kubeconfig for production cluster
# ArgoCD is the ONLY way to change production

# ArgoCD RBAC:
# Teams can sync their own apps in dev/staging
# Teams can REQUEST sync in production (creates ArgoCD sync request)
# Only senior engineers + ArgoCD bot can approve production syncs

# ArgoCD Projects enforce:
p, role:team-a, applications, sync, team-a-dev/*, allow
p, role:team-a, applications, sync, team-a-staging/*, allow
p, role:team-a, applications, sync, team-a-production/*, deny  # cannot sync prod
p, role:senior-devops, applications, sync, */*, allow          # can sync anything
```

**Secret Management:**
```yaml
# External Secrets Operator syncs from Secrets Manager to Kubernetes
# Developers commit secret REFERENCES (not values) to Git

# platform-config/apps/payment-service/external-secret.yaml:
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-db-secret
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager
  target:
    name: payment-db-creds
  data:
  - secretKey: password
    remoteRef:
      key: production/payment-service/db
      property: password
```

**Audit Logging:**
```
Every production change creates audit trail via:
1. Git commit history (who merged what, when) — platform-config repo
2. ArgoCD sync history (what was synced to which cluster, when, by whom)
3. AWS CloudTrail (API calls made during deployment)
4. Kubernetes audit logs (every kubectl/API action on cluster)

All forwarded to:
- S3 with Object Lock (7-year retention for compliance)
- Elasticsearch (searchable, 90-day hot storage)
```

**Platform Self-Healing:**
```yaml
# ArgoCD manages itself via App of Apps
# If ArgoCD crashes → Kubernetes restarts it (Deployment)
# If Kubernetes crashes → kubeadm/EKS recovers control plane
# If etcd has issues → automatic compaction, regular backups to S3

# Platform components managed by ArgoCD:
/platform/
  argocd-app.yaml           # ArgoCD manages its own config
  prometheus-stack-app.yaml
  cert-manager-app.yaml
  external-secrets-app.yaml

# Backup strategy:
# etcd: CronJob → snapshot every 6 hours → S3 (30-day retention)
# platform-config repo: GitHub backup to S3 daily
# ECR images: replicated to secondary region
```

**Disaster Recovery:**
```bash
# RTO: 2 hours (rebuild entire platform from scratch)
# RPO: 6 hours (last etcd backup)

# DR procedure:
1. Create new EKS cluster in DR region
2. Restore etcd snapshot (or start fresh + resync from Git)
3. Bootstrap ArgoCD: kubectl apply -f argocd-bootstrap.yaml
4. ArgoCD syncs platform-config → recreates all 200 services
5. Point Route53 to DR cluster
```

> 💡 **Interview tip:** The most impressive part of this design is the **"Git is source of truth" principle taken to its logical conclusion** — not just for applications, but for the platform itself (ArgoCD manages ArgoCD via App of Apps). This means disaster recovery is simply: create a cluster, install ArgoCD, point it at the Git repo, and everything self-heals. Also emphasize the **CI → platform-config PR flow** — this is what makes compliance possible, because every production change is a Git PR with 2 approvals and a full audit trail, regardless of which team or service is being deployed.

---

## Key Topics Coverage (Q181–Q225)

| Topic | Questions |
|---|---|
| Kubernetes | Q181, Q186, Q191, Q197, Q201, Q206, Q211, Q216, Q220 |
| AWS | Q182, Q188, Q195, Q200, Q205, Q210, Q215, Q219, Q224 |
| Terraform | Q184, Q193, Q203, Q213, Q222 |
| Prometheus | Q185, Q196, Q214 |
| Grafana | Q202, Q221 |
| ELK Stack | Q190, Q207 |
| Linux / Bash | Q183, Q192, Q199, Q209, Q218 |
| ArgoCD | Q198, Q217 |
| Jenkins | Q189, Q208 |
| GitHub Actions | Q194, Q212 |
| Git | Q187, Q223 |
| Python | Q204 |
| System Design | Q225 |

---

*Part 5 Enhanced Edition — DevOps/SRE Interview Questions with Key Points & Interview Tips*
*nawab312 GitHub repositories*
