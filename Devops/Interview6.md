# DevOps / SRE / Cloud Interview Questions (Q226–Q270)

> Each question includes: Key Points to Cover + Interview Tip
> Format mirrors Q64 reference style.

---

## Table of Contents

| # | Topic | Type | Difficulty |
|---|---|---|---|
| 226 | Kubernetes | Conceptual | Advanced |
| 227 | AWS | Scenario-Based | Advanced |
| 228 | Linux / Bash | Conceptual | Advanced |
| 229 | Terraform | Scenario-Based | Advanced |
| 230 | Prometheus | Conceptual | Advanced |
| 231 | Kubernetes | Troubleshooting | Advanced |
| 232 | Git | Scenario-Based | Advanced |
| 233 | AWS | Conceptual | Advanced |
| 234 | Jenkins | Troubleshooting | Advanced |
| 235 | ELK Stack | Conceptual | Advanced |
| 236 | Kubernetes | Scenario-Based | Advanced |
| 237 | Linux / Bash | Troubleshooting | Advanced |
| 238 | Terraform | Conceptual | Advanced |
| 239 | GitHub Actions | Conceptual | Advanced |
| 240 | AWS | Troubleshooting | Advanced |
| 241 | Prometheus | Troubleshooting | Advanced |
| 242 | Kubernetes | Conceptual | Advanced |
| 243 | ArgoCD | Scenario-Based | Advanced |
| 244 | Linux / Bash | Scenario-Based | Advanced |
| 245 | AWS | Scenario-Based | Advanced |
| 246 | Kubernetes | Troubleshooting | Advanced |
| 247 | Grafana | Troubleshooting | Advanced |
| 248 | Terraform | Troubleshooting | Advanced |
| 249 | Python | Scenario-Based | Advanced |
| 250 | AWS | Conceptual | Advanced |
| 251 | Kubernetes | Scenario-Based | Advanced |
| 252 | ELK Stack | Scenario-Based | Advanced |
| 253 | Jenkins | Conceptual | Advanced |
| 254 | Linux / Bash | Conceptual | Advanced |
| 255 | AWS | Scenario-Based | Advanced |
| 256 | Kubernetes | Conceptual | Advanced |
| 257 | GitHub Actions | Scenario-Based | Advanced |
| 258 | Terraform | Scenario-Based | Advanced |
| 259 | Prometheus | Scenario-Based | Advanced |
| 260 | AWS | Conceptual | Advanced |
| 261 | Kubernetes | Scenario-Based | Advanced |
| 262 | ArgoCD | Conceptual | Advanced |
| 263 | Linux / Bash | Scenario-Based | Advanced |
| 264 | AWS | Scenario-Based | Advanced |
| 265 | Kubernetes | Conceptual | Advanced |
| 266 | Grafana | Scenario-Based | Advanced |
| 267 | Terraform | Conceptual | Advanced |
| 268 | Git | Conceptual | Advanced |
| 269 | AWS + Kubernetes | Troubleshooting | Advanced |
| 270 | All Topics | System Design | Advanced |

---

### Q226 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `ownerReferences`** and garbage collection. Difference between **foreground** and **background** cascading deletion. When to use `--cascade=orphan`.

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`

```
ownerReferences — the parent-child relationship:
  Every Kubernetes object can have ownerReferences field
  pointing to its parent object

  Pod's ownerReferences:
    apiVersion: apps/v1
    kind: ReplicaSet
    name: my-app-7d9f
    uid: abc-123
    controller: true          # this owner controls this object
    blockOwnerDeletion: true  # block parent deletion until I'm gone

Garbage collection:
  Kubernetes GC controller watches for objects whose owner no longer exists
  Owner deleted → GC deletes all owned children automatically

  kubectl delete deployment my-app
  → Deployment deleted
  → ReplicaSet orphaned (owner gone) → GC deletes it
  → Pods orphaned (owner gone) → GC deletes them

Background cascading deletion (default):
  kubectl delete deployment my-app
  → Deployment object deleted IMMEDIATELY from API server
  → GC runs asynchronously to delete ReplicaSets and Pods
  → Brief window: Deployment gone but Pods still running
  → Use for: most cases — fast response to delete command

Foreground cascading deletion:
  kubectl delete deployment my-app --cascade=foreground
  → Deployment object stays in Terminating state
  → Controller deletes ALL owned objects (Pods, ReplicaSets) FIRST
  → Only after all children deleted → Deployment object removed
  → Foreground adds finalizer: foregroundDeletion
  → Use for: ensuring no orphaned resources, guaranteed cleanup order

Orphan cascade (--cascade=orphan):
  kubectl delete deployment my-app --cascade=orphan
  → Deployment deleted
  → ReplicaSets and Pods are NOT deleted — become orphans
  → GC does NOT collect orphans (they have no owner to check)

Real-world use case for --cascade=orphan:
  Scenario: migrate ReplicaSet to different Deployment (blue-green)
  kubectl delete deployment old-app --cascade=orphan
  → old-app Deployment gone
  → ReplicaSet still running (still serving traffic)
  → Create new Deployment that adopts the ReplicaSet
    (by matching selector labels)
  → Zero downtime: pods never terminated during migration

  Another use case:
  Accidentally created wrong Deployment name
  kubectl delete deployment wrong-name --cascade=orphan
  → Create correct Deployment, pods keep running — no restart
```

💡 **Interview tip:** The `blockOwnerDeletion: true` field is the mechanism behind foreground deletion — it prevents the garbage collector from deleting the owner until all objects with this flag are gone first. The `--cascade=orphan` use case is subtle but important: it lets you delete a controller object while keeping its pods alive — useful for zero-downtime migrations between controllers. Background deletion is fast but async; foreground is slower but guarantees complete cleanup before the parent object disappears.

---

### Q227 — AWS | Scenario-Based | Advanced

> AWS Config violation: S3 bucket missing server-side encryption. 200 buckets. Fix immediate violation, audit all buckets, auto-remediate, prevent with SCPs.

📁 **Reference:** `nawab312/AWS` — AWS Config, S3 encryption, auto-remediation

```
Step 1 — Fix immediate violation:
  aws s3api put-bucket-encryption \
    --bucket prod-data-bucket \
    --server-side-encryption-configuration '{
      "Rules": [{
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "arn:aws:kms:us-east-1:123:key/abc"
        },
        "BucketKeyEnabled": true
      }]
    }'

Step 2 — Audit all 200 buckets:
  #!/bin/bash
  echo "Bucket,Encryption,Compliant"
  for bucket in $(aws s3api list-buckets --query 'Buckets[].Name' --output text); do
    algo=$(aws s3api get-bucket-encryption --bucket "$bucket" \
      --query 'ServerSideEncryptionConfiguration.Rules[0].ApplyServerSideEncryptionByDefault.SSEAlgorithm' \
      --output text 2>/dev/null || echo "NONE")
    compliant=$( [ "$algo" = "aws:kms" ] && echo "YES" || echo "NO" )
    echo "$bucket,$algo,$compliant"
  done

  # Or AWS Config SQL query (scales to 1000s of buckets):
  aws configservice select-aggregate-resource-config \
    --expression "SELECT resourceId, configuration
      WHERE resourceType = 'AWS::S3::Bucket'
      AND configuration.serverSideEncryptionConfiguration IS NULL"

Step 3 — AWS Config auto-remediation with SSM Automation:
  # Managed Config rule (no custom code):
  aws configservice put-config-rule --config-rule '{
    "ConfigRuleName": "s3-bucket-server-side-encryption-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
    }
  }'

  # Auto-remediation using SSM Automation document:
  aws configservice put-remediation-configurations \
    --remediation-configurations '[{
      "ConfigRuleName": "s3-bucket-server-side-encryption-enabled",
      "TargetType": "SSM_DOCUMENT",
      "TargetId": "AWS-EnableS3BucketEncryption",
      "Parameters": {
        "AutomationAssumeRole": {
          "StaticValue": {"Values": ["arn:aws:iam::123:role/config-remediation"]}
        },
        "BucketName": {"ResourceValue": {"Value": "RESOURCE_ID"}},
        "SSEAlgorithm": {"StaticValue": {"Values": ["aws:kms"]}}
      },
      "Automatic": true,
      "MaximumAutomaticAttempts": 3,
      "RetryAttemptSeconds": 60
    }]'

Step 4 — Prevent non-compliant buckets with SCP:
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Deny",
      "Action": "s3:CreateBucket",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }]
  }
```

💡 **Interview tip:** The layered approach is the complete answer: fix → audit → auto-remediate → prevent. The SCP is the most powerful layer — it prevents non-compliant buckets at creation time regardless of IAM permissions. For auto-remediation, start with `Automatic: false` and manually trigger remediation until you're confident the SSM document works correctly before enabling fully automated remediation on 200 production buckets. The `select-aggregate-resource-config` SQL query is the scalable audit approach — far faster than a shell loop for hundreds of buckets across multiple accounts.

---

### Q228 — Linux / Bash | Conceptual | Advanced

> Explain **SELinux** and **AppArmor** — what they are, how they differ, SELinux contexts/types/policies, enforcement modes, Kubernetes interaction.

📁 **Reference:** `nawab312/DSA` → `Linux`, `nawab312/Kubernetes` → `07_SECURITY.md`

```
What problem they solve:
  DAC (Discretionary Access Control) = traditional Unix permissions
  User owns file → user decides who can access it
  Problem: compromised process running as root = game over

  MAC (Mandatory Access Control) = kernel-enforced policy
  System policy overrides user decisions
  Even root cannot bypass MAC policy
  Compromised process: limited to what policy allows, even if root

SELinux (Security-Enhanced Linux):
  Developed by: NSA, used in RHEL/CentOS/Amazon Linux
  Implementation: Linux Security Module (LSM) in kernel

  Security context (label on every object):
    user:role:type:level
    system_u:system_r:httpd_t:s0

  Type enforcement — the core mechanism:
    Rule: allow httpd_t httpd_sys_content_t : file { read open getattr }
    Apache (httpd_t) CAN read files labeled httpd_sys_content_t
    Apache CANNOT read shadow_t files even if Unix perms allow

  SELinux modes:
    enforcing:  violations DENIED and logged → production mode
    permissive: violations LOGGED but ALLOWED → audit/troubleshoot
    disabled:   SELinux completely off (requires reboot to change)

  Commands:
    getenforce                    # current mode
    sestatus                      # full status
    ls -Z /var/www/html           # show file contexts
    ps -eZ | grep httpd           # show process context
    ausearch -m avc -ts recent    # find denied operations in audit log
    restorecon -r /var/www/html   # restore default context
    chcon -t httpd_sys_content_t file  # change file context (temp)

AppArmor:
  Used in: Ubuntu/Debian (default on most cloud Ubuntu AMIs)
  Path-based profiles (simpler than label-based SELinux)
  Profile per application: allowed files, capabilities, network

  Modes:
    enforce:  violations denied + logged
    complain: violations logged only (discovery mode)

  Profile example (nginx):
    /usr/sbin/nginx {
      /var/www/html/** r,           # read web root
      /var/log/nginx/*.log w,       # write logs
      network inet stream,          # TCP networking
      capability net_bind_service,  # bind port < 1024
    }

SELinux vs AppArmor:
  SELinux:  label-based, more powerful, harder to configure
            used on RHEL/Amazon Linux
  AppArmor: path-based, simpler, easier to write profiles
            used on Ubuntu/Debian

Kubernetes interaction:
  Pod with SELinux context:
  spec:
    securityContext:
      seLinuxOptions:
        level: "s0:c123,c456"   # unique MCS categories per pod
    # Prevents container A from reading container B's files

  Pod with AppArmor profile:
  metadata:
    annotations:
      container.apparmor.security.beta.kubernetes.io/app: runtime/default
  # K8s 1.30+:
  spec:
    containers:
    - securityContext:
        appArmorProfile:
          type: RuntimeDefault
```

💡 **Interview tip:** The key insight: both SELinux and AppArmor provide defense-in-depth BELOW the application level — even if an attacker breaks out of the application, the kernel policy limits what they can do. In Kubernetes, SELinux MCS (Multi-Category Security) prevents one container from accessing another container's files on the same node — each container gets unique category labels and the kernel blocks cross-container file access. In production, when SELinux causes mysterious permission denied errors, use `ausearch -m avc -ts recent` to find denials in audit logs, then `audit2allow` to generate the required policy rule.

---

### Q229 — Terraform | Scenario-Based | Advanced

> Implement **Terraform testing** — validate/fmt in CI, Terratest unit tests, plan contract testing, integration testing, Infracost in CI.

📁 **Reference:** `nawab312/Terraform`, `nawab312/CI_CD`

```
Level 1 — Static validation (every PR, < 30 seconds):
  steps:
  - name: Format check
    run: terraform fmt -check -recursive
    # Fails if any .tf file needs formatting

  - name: Init (no real backend)
    run: terraform init -backend=false

  - name: Validate
    run: terraform validate
    # Checks: syntax, required variables, provider constraints

  - name: TFLint
    run: tflint --recursive
    # Catches: deprecated syntax, AWS best practices, naming conventions

Level 2 — Plan contract testing (per PR, ~2 minutes):
  - name: Terraform plan
    run: terraform plan -out=plan.tfplan

  - name: Assert no destructive changes
    run: |
      terraform show -json plan.tfplan > plan.json
      DESTROYS=$(jq '[.resource_changes[] | select(.change.actions[] == "delete")] | length' plan.json)
      [ "$DESTROYS" -gt 0 ] && echo "FAIL: $DESTROYS resources will be destroyed" && exit 1

Level 3 — Terratest unit tests (per module, ~10 minutes):
  // test/ecs_service_test.go
  func TestECSServiceModule(t *testing.T) {
    t.Parallel()
    opts := &terraform.Options{
      TerraformDir: "../examples/basic",
      Vars: map[string]interface{}{
        "service_name": "test-ecs-" + random.UniqueId(),
      },
    }
    defer terraform.Destroy(t, opts)  // always cleanup
    terraform.InitAndApply(t, opts)

    serviceArn := terraform.Output(t, opts, "service_arn")
    assert.Contains(t, serviceArn, "arn:aws:ecs")
  }

Level 4 — Integration test (full stack, ~30 minutes):
  # Deploy entire environment to test account
  # Run end-to-end tests (HTTP requests, DB connectivity)
  # Tear down after

Level 5 — Infracost cost gate in CI:
  - name: Generate cost estimate
    run: infracost breakdown --path=. --format=json > infracost.json

  - name: Post cost comment on PR
    run: infracost comment github --path=infracost.json \
      --repo=$GITHUB_REPOSITORY --pull-request=${{ github.event.number }} \
      --github-token=${{ secrets.GITHUB_TOKEN }}

  - name: Cost gate — fail if monthly cost increases > $100
    run: |
      DIFF=$(jq '.diffTotalMonthlyCost | tonumber' infracost.json)
      [ $(echo "$DIFF > 100" | bc) -eq 1 ] && \
        echo "Cost increase $$DIFF/month exceeds threshold" && exit 1
```

💡 **Interview tip:** The testing pyramid for Terraform mirrors software testing: fast static checks on every commit, plan contract tests on every PR, Terratest module tests on module changes. The cost gate with Infracost is increasingly important — it prevents infrastructure changes that accidentally add $10,000/month in cost from sneaking through code review unnoticed. Terratest's `defer terraform.Destroy` is critical — it ensures cleanup happens even when tests fail, preventing test resources from accumulating costs in the test AWS account. Always run Terratest in a dedicated sandbox account with strict IAM limits.

---

### Q230 — Prometheus | Conceptual | Advanced

> Explain **`relabel_configs`** vs **`metric_relabel_configs`** — when each is applied. Write rules: drop dev metrics, rename label, add static label, drop zero-value metrics.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`

```
When each is applied:
  scrape pipeline:
  Target discovery → relabel_configs → HTTP scrape → metric_relabel_configs → storage

  relabel_configs:
    Applied BEFORE the scrape (on the TARGET)
    Input: target labels (__address__, discovery labels)
    Can: filter which targets to scrape (drop entire targets)
         modify target URL or job label
    Performance: dropped targets never scraped (save HTTP calls)

  metric_relabel_configs:
    Applied AFTER the scrape (on each METRIC from the response)
    Input: metric name + all labels + value
    Can: filter which metrics to store, rename labels
    Performance: metrics scraped but dropped before storage

Rule 1 — Drop all metrics where env=dev:
  metric_relabel_configs:
  - source_labels: [env]
    regex: "dev"
    action: drop

Rule 2 — Rename label pod_name to pod:
  metric_relabel_configs:
  - source_labels: [pod_name]
    target_label: pod
  - regex: "pod_name"
    action: labeldrop

  # Shorter using labelmap:
  metric_relabel_configs:
  - action: labelmap
    regex: pod_name
    replacement: pod

Rule 3 — Add static label cluster="prod-us-east":
  # In relabel_configs (before scrape) OR global external_labels (better):
  global:
    external_labels:
      cluster: "prod-us-east"
      environment: "production"

  # Per-job via relabel_configs:
  relabel_configs:
  - target_label: cluster
    replacement: "prod-us-east"

Rule 4 — Drop metrics where value is 0:
  metric_relabel_configs:
  - source_labels: [__value__]   # special label: the metric's numeric value
    regex: "0"
    action: drop

Common relabeling actions:
  keep:      only keep metrics/targets matching regex
  drop:      drop metrics/targets matching regex
  replace:   replace label value using regex capture groups
  labelmap:  rename labels by regex
  labeldrop: remove labels matching regex
  labelkeep: remove ALL labels NOT matching regex

Real-world Kubernetes SD enrichment:
  relabel_configs:
  - source_labels: [__meta_kubernetes_namespace]
    target_label: namespace
  - source_labels: [__meta_kubernetes_pod_label_app]
    target_label: app
  - source_labels: [__meta_kubernetes_namespace]
    regex: "production"
    action: keep     # only scrape pods in production namespace
```

💡 **Interview tip:** The timing distinction is the key point — `relabel_configs` operates on targets before scraping (saves network calls by not scraping unwanted targets), while `metric_relabel_configs` operates on scraped data (saves storage by dropping unwanted metrics). Using `relabel_configs` to drop entire targets is more efficient than `metric_relabel_configs` to drop their metrics — the HTTP scrape call is never made. The `__value__` special label in `metric_relabel_configs` is a lesser-known feature that lets you filter metrics based on their numeric value — useful for dropping noisy zero-value metrics.

---

### Q231 — Kubernetes | Troubleshooting | Advanced

> Nodes in `MemoryPressure` — pods evicted but immediately evicted again. Diagnose `evictionHard` vs `evictionSoft`, resolve without new nodes.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`

```
Step 1 — Confirm memory pressure:
  kubectl get nodes
  # STATUS: Ready,MemoryPressure
  kubectl describe node <node> | grep -A5 "Conditions:"

Step 2 — Check eviction thresholds on the node:
  cat /var/lib/kubelet/config.yaml | grep -A10 eviction

  # Default evictionHard thresholds:
  evictionHard:
    memory.available: "100Mi"   # evict when < 100Mi available
    nodefs.available: "10%"
    nodefs.inodesFree: "5%"

  evictionSoft:
    memory.available: "200Mi"   # start evicting after grace period
  evictionSoftGracePeriod:
    memory.available: "1m30s"   # wait 90s before evicting

  Difference:
    evictionHard: immediate eviction when threshold crossed
    evictionSoft: eviction only if condition persists for grace period
    evictionHard thresholds are lower — last resort
    evictionSoft thresholds are higher — early warning

Step 3 — Why new pods immediately evicted:
  MemoryPressure condition = node has taints:
  node.kubernetes.io/memory-pressure:NoSchedule  (prevents new scheduling)
  node.kubernetes.io/memory-pressure:NoExecute   (evicts existing pods)

  If ALL nodes have MemoryPressure:
  Scheduler has nowhere else to schedule → puts pod on pressured node
  Pod starts → consumes memory → immediately hits threshold → evicted again

Step 4 — Find memory consumers:
  kubectl top pods -A --sort-by=memory | head -20

  # Find pods WITHOUT memory limits (can grow unbounded):
  kubectl get pods -A -o json | jq '
    .items[] |
    select(.spec.containers[].resources.limits.memory == null) |
    .metadata.namespace + "/" + .metadata.name'

Step 5 — Resolve without adding nodes:

  Option A: Evict low-priority pods manually:
    kubectl delete pod <low-priority-pod> -n production

  Option B: Add memory limits to unbounded pods:
    kubectl set resources deployment myapp \
      --limits=memory=512Mi --requests=memory=256Mi

  Option C: Identify and fix memory leak:
    watch kubectl top pod myapp-xxx
    # Steadily growing = memory leak → kubectl rollout restart deployment myapp

  Option D: Apply LimitRange to namespace (prevent future unbounded pods):
    apiVersion: v1
    kind: LimitRange
    spec:
      limits:
      - type: Container
        default:
          memory: 256Mi
          cpu: 200m
        defaultRequest:
          memory: 128Mi
          cpu: 100m
        max:
          memory: 2Gi
```

💡 **Interview tip:** The immediate re-eviction loop happens because if all nodes are under pressure, the scheduler has nowhere else to go and keeps scheduling onto the pressured node where pods are immediately evicted. The root fix is always finding and fixing the source of memory pressure: usually an unbounded pod without memory limits that grows until it starves the node. LimitRange as a preventative measure is the sustainable answer — it ensures every new pod gets default limits even if the developer forgot to specify them, preventing this class of problem from happening again.

---

### Q232 — Git | Scenario-Based | Advanced

> Managing a **library used by 10 repos as Git submodule** — update workflow, pin versions, update all dependents, pitfalls vs alternatives.

📁 **Reference:** `nawab312/CI_CD` → `Git`

```
Git submodule mechanics:
  Parent repo stores: path to submodule + specific commit SHA (NOT branch)
  .gitmodules:
    [submodule "libs/shared"]
      path = libs/shared
      url = https://github.com/company/shared-lib.git

Complete workflow for breaking change across 10 repos:

  Step 1 — Tag the library release:
    cd shared-lib
    git tag v2.0.0 && git push origin v2.0.0

  Step 2 — Update each dependent repository:
    cd repo-1
    git submodule update --init libs/shared
    cd libs/shared
    git checkout v2.0.0          # pin to specific tag
    cd ../..
    git add libs/shared          # stage the updated submodule pointer
    git commit -m "chore: update shared-lib to v2.0.0"
    git push

  Step 3 — Automate across all 10 repos:
    for repo in repo-{1..10}; do
      git clone https://github.com/company/$repo.git
      cd $repo
      git submodule update --init --recursive
      cd libs/shared && git checkout v2.0.0 && cd ../..
      git add libs/shared
      git commit -m "chore: update shared-lib to v2.0.0"
      git push && cd ..
    done

  Step 4 — New developers must:
    git clone --recursive https://github.com/company/repo-1.git
    # OR after plain clone:
    git submodule init && git submodule update

Submodule pitfalls:
  1. Easy to forget: git clone misses submodules without --recursive
  2. Detached HEAD: submodule is in detached HEAD state (not on a branch)
  3. Merge conflicts: two branches update submodule pointer differently
  4. No automatic updates: each repo must manually update pointer
  5. CI must: git submodule update or use checkout with submodules: true

Better alternatives:
  Option A: Private npm/pip/Maven package registry:
    Publish as versioned package to GitHub Packages/JFrog
    Update via: bump version in package.json → install
    Benefits: semantic versioning, Dependabot automation

  Option B: Monorepo:
    All repos and shared-lib in one repository
    No submodule pointer to manage
    Tools: Nx, Turborepo for build isolation

  Option C: Vendoring:
    Copy shared-lib into each repo under vendor/
    Hermetic builds, no external dependency at build time
```

💡 **Interview tip:** The interviewer is testing whether you know the submodule pitfalls — "forgetting to run `git submodule update`" breaks the build for everyone who clones without `--recursive`, which is a constant source of frustration. The stronger answer is knowing when NOT to use submodules: for actively developed shared code used by many repos, a private package registry with semantic versioning and Dependabot automation is far easier to manage. Submodules are acceptable for third-party code you want to vendor at a specific version but don't actively change.

---

### Q233 — AWS | Conceptual | Advanced

> Explain **AWS EventBridge** — event sources. Design event-driven architecture: EC2 termination → Lambda, RDS backup failure → Step Function, GuardDuty → Slack, weekly cost report.

📁 **Reference:** `nawab312/AWS` — EventBridge, event-driven architecture

```
EventBridge event sources:
  AWS services:  EC2, RDS, S3, Lambda, GuardDuty, Config, CodePipeline,
                 CloudTrail (API call events), Health events
  Custom apps:   PutEvents API from your application
  SaaS partners: Zendesk, Datadog, PagerDuty (EventBridge partner integrations)
  Scheduled:     cron or rate expressions (replaces CloudWatch Events)

Architecture 1 — EC2 unexpected termination → Lambda:
  {
    "source": ["aws.ec2"],
    "detail-type": ["EC2 Instance State-change Notification"],
    "detail": { "state": ["terminated"] }
  }
  # OR filter out ASG-managed terminations:
  {
    "source": ["aws.ec2"],
    "detail-type": ["AWS API Call via CloudTrail"],
    "detail": {
      "eventName": ["TerminateInstances"],
      "userAgent": [{"anything-but": "autoscaling.amazonaws.com"}]
    }
  }
  Target: Lambda → check if termination was expected → page on-call if not

Architecture 2 — RDS backup failure → Step Function:
  {
    "source": ["aws.rds"],
    "detail-type": ["RDS DB Snapshot Event"],
    "detail": {"EventID": ["RDS-EVENT-0091"]}  # "Automated backup failed"
  }
  Target: Step Functions:
    State 1: Wait 5 minutes
    State 2: Lambda — check if backup succeeded
    State 3 (still failed): SNS → PagerDuty + trigger manual backup

Architecture 3 — GuardDuty findings → Slack:
  {
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": {"severity": [{"numeric": [">=", 7]}]}  # HIGH and CRITICAL only
  }
  Target: Lambda → format finding → curl Slack webhook

Architecture 4 — Weekly cost report (scheduled):
  Schedule: cron(0 8 ? * MON *)   # 8 AM every Monday UTC
  Target: Lambda → Cost Explorer API → CSV → S3 → SES email
```

💡 **Interview tip:** EventBridge's key advantage over simple CloudWatch alarms is rich JSON pattern matching — you can filter on any field in the event, not just metric thresholds. The `anything-but` operator in patterns is particularly powerful for the EC2 termination use case — it lets you exclude expected terminations (ASG scale-in, manual teardown) and only page for unexpected ones. For GuardDuty, always filter by severity >= 7 (High/Critical) — without filtering, every Low severity finding creates a Slack message which leads to alert fatigue within hours.

---

### Q234 — Jenkins | Troubleshooting | Advanced

> Jenkins randomly fails with `ChannelClosedException: channel is already closed`. Happens at different stages, cannot reproduce. Diagnose and fix.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins` — agent connectivity

```
What ChannelClosedException means:
  Jenkins master ↔ agent communicate via persistent TCP channel (Remoting)
  Channel closed = TCP connection terminated mid-build
  Random stages = random timing of the network glitch

Step 1 — Check Jenkins logs:
  grep -i "channel\|remoting\|TCP\|disconnect" ~/.jenkins/logs/jenkins.log
  # Look for: java.net.SocketTimeoutException
  #           java.io.IOException: Connection reset

Step 2 — Check TCP keepalive settings:
  cat /proc/sys/net/ipv4/tcp_keepalive_time    # default: 7200s (2 hours!)
  # With default keepalive: TCP appears alive for up to 7875s after network goes down
  # Intermediate firewalls often have 300-900s idle timeout
  # → Connection terminated by firewall, Jenkins doesn't know for minutes

  Fix — reduce keepalive to detect dead connections sooner:
  sysctl -w net.ipv4.tcp_keepalive_time=60
  sysctl -w net.ipv4.tcp_keepalive_intvl=10
  sysctl -w net.ipv4.tcp_keepalive_probes=5

  # Permanent (/etc/sysctl.conf):
  net.ipv4.tcp_keepalive_time = 60
  net.ipv4.tcp_keepalive_intvl = 10
  net.ipv4.tcp_keepalive_probes = 5

  # Jenkins Remoting keepalive:
  java -jar agent.jar ... \
    -Dhudson.remoting.Launcher.pingIntervalSec=30 \
    -Dhudson.remoting.Launcher.pingTimeoutSec=60

Step 3 — Check for load balancer idle timeout:
  # If Jenkins master is behind ALB (default idle timeout: 60 seconds)
  # A 5-minute build with no HTTP requests = ALB closes connection

  Fix: increase ALB idle timeout:
  aws elbv2 modify-load-balancer-attributes \
    --load-balancer-arn <arn> \
    --attributes Key=idle_timeout.timeout_seconds,Value=3600

Step 4 — Use WebSocket connection (modern Jenkins, 2.217+):
  # WebSocket works through load balancers and firewalls that block TCP
  java -jar agent.jar -jnlpUrl https://jenkins/... -webSocket

Step 5 — Add retry logic as short-term workaround:
  retry(3) {
    sh 'long-running-test.sh'
  }
```

💡 **Interview tip:** The root cause is almost always a network device (firewall, load balancer, NAT gateway) with an idle connection timeout shorter than the build duration. The TCP keepalive fix is the most reliable solution — it keeps the connection active by sending probes every 60 seconds, preventing intermediate devices from closing the "idle" connection. WebSocket connections are even better for cloud environments because they reuse HTTP/HTTPS ports that are always allowed through firewalls and load balancers, unlike raw TCP on port 50000.

---

### Q235 — ELK Stack | Conceptual | Advanced

> Explain **Elasticsearch mapping** — dynamic vs explicit, `text` vs `keyword`, mapping explosion, write explicit mapping for app logs.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack`

```
Dynamic mapping (default):
  New field seen → automatically creates mapping
  String value → both text AND keyword (multi-field)
  Risks: mapping explosion, unexpected types, cannot change existing mapping

text field:
  Full-text analyzed: tokenized, lowercased, stemmed
  Enables: fuzzy search, relevance scoring, partial word match
  Cannot: sort, aggregate (terms aggregation), exact match
  Use for: log messages, descriptions, free-text search

keyword field:
  Exact value, not analyzed, stored as-is
  Enables: exact match, sorting, aggregation, cardinality
  Cannot: fuzzy search, partial word match
  Use for: service names, IP addresses, status codes, IDs

Default multi-field for strings:
  "message": {
    "type": "text",           # for full-text search
    "fields": {
      "keyword": {            # message.keyword for exact/aggregations
        "type": "keyword",
        "ignore_above": 256
      }
    }
  }

Mapping explosion:
  Problem: dynamic mapping creates new field for every unique key
  Real scenario: logging metadata as dynamic JSON with 1000 unique keys
  → 1000 new field mappings → node OOM, cluster instability

  Prevention:
  1. "dynamic": false   → reject unknown fields
  2. "type": "flattened" → stores nested JSON without sub-fields
  3. index.mapping.total_fields.limit: 500  (default: 1000)

Explicit mapping for application logs:
  PUT /app-logs
  {
    "mappings": {
      "dynamic": "strict",       # reject unknown fields — no explosion
      "properties": {
        "@timestamp": {"type": "date"},
        "level":      {"type": "keyword"},   # ERROR, WARN, INFO — aggregatable
        "service":    {"type": "keyword"},   # exact match
        "message": {
          "type": "text",
          "fields": {"keyword": {"type": "keyword", "ignore_above": 512}}
        },
        "trace_id":     {"type": "keyword"},
        "duration_ms":  {"type": "long"},
        "http_status":  {"type": "short"},
        "metadata": {
          "type": "flattened",    # dynamic JSON — no explosion
          "doc_values": false     # save disk
        }
      }
    }
  }
```

💡 **Interview tip:** Mapping explosion is a production incident waiting to happen — Elasticsearch clusters have gone down because an application started logging a UUID as a field key, creating millions of unique field mappings. The `"dynamic": "strict"` setting is safest for production — it rejects documents with unknown fields rather than silently creating mappings. The `flattened` field type is the modern solution for semi-structured data like metadata objects — it stores the entire nested object as a single leaf field, enabling basic queries without explosion. Always separate `text` (for search) from `keyword` (for aggregations) using the multi-field pattern.

---

### Q236 — Kubernetes | Scenario-Based | Advanced

> ML training job requiring GPU (NVIDIA A100), large EFS datasets, 48-hour runtime, resume from checkpoint on node failure, not blocking CPU nodes.

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`, `04_STORAGE.md`

```yaml
# GPU node taint (applied to GPU nodes):
# kubectl taint nodes gpu-node-1 nvidia.com/gpu=true:NoSchedule
# kubectl label nodes gpu-node-1 accelerator=nvidia-a100

# PVC for EFS dataset (ReadWriteMany — multiple pods can read):
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ml-dataset-pvc
spec:
  storageClassName: efs-sc
  accessModes: [ReadWriteMany]
  resources:
    requests:
      storage: 500Gi

---
# Separate PVC for checkpoints (survives pod restarts):
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ml-checkpoint-pvc
spec:
  storageClassName: gp3-wait
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Gi

---
apiVersion: batch/v1
kind: Job
metadata:
  name: ml-training-job
spec:
  backoffLimit: 10                  # allow 10 retries (node failures)
  activeDeadlineSeconds: 172800     # 48 hours max
  template:
    spec:
      restartPolicy: OnFailure      # retry on failure (not Never)

      # Only schedule on GPU nodes:
      tolerations:
      - key: nvidia.com/gpu
        operator: Equal
        value: "true"
        effect: NoSchedule
      nodeSelector:
        accelerator: nvidia-a100

      containers:
      - name: trainer
        image: company/ml-trainer:v1.2.0
        command:
        - python
        - train.py
        - --checkpoint-dir=/checkpoints
        - --data-dir=/datasets
        - --resume-from-checkpoint   # app: resume if checkpoint exists

        resources:
          requests:
            cpu: "4"
            memory: "32Gi"
            nvidia.com/gpu: "1"     # request 1 GPU
          limits:
            nvidia.com/gpu: "1"     # must equal request for GPU

        volumeMounts:
        - name: datasets
          mountPath: /datasets
          readOnly: true
        - name: checkpoints
          mountPath: /checkpoints   # save/restore checkpoints here

      # Long grace period: save checkpoint before eviction
      terminationGracePeriodSeconds: 300

      volumes:
      - name: datasets
        persistentVolumeClaim:
          claimName: ml-dataset-pvc
      - name: checkpoints
        persistentVolumeClaim:
          claimName: ml-checkpoint-pvc

# NVIDIA device plugin (required for GPU scheduling):
# helm install nvdp nvdp/nvidia-device-plugin -n kube-system
```

💡 **Interview tip:** The checkpoint resume pattern is the key architectural decision — the job must be idempotent: if restarted, it picks up from the last saved checkpoint rather than starting from scratch. This requires both Kubernetes configuration (`backoffLimit` for retries) AND application code that saves checkpoints regularly. The separate PVC for checkpoints (distinct from dataset PVC) ensures checkpoints survive pod deletion and are available when the pod restarts on a different GPU node. The `terminationGracePeriodSeconds: 300` gives the application 5 minutes to save a final checkpoint when it receives SIGTERM before being killed.

---

### Q237 — Linux / Bash | Troubleshooting | Advanced

> Process stuck in **`D` state** — cannot be killed with `kill -9`. What is D state, causes, why SIGKILL fails, recovery without reboot.

📁 **Reference:** `nawab312/DSA` → `Linux` — process states, D state

```
What D state (uninterruptible sleep) means:
  Process is waiting for I/O to complete
  Kernel has put the process to sleep INSIDE a system call
  Process is IN the kernel, not in user space

Why SIGKILL cannot kill D state:
  kill -9 sends SIGKILL → kernel delivers signal to PROCESS
  Process in D state is INSIDE kernel code — not receiving user signals
  Kernel is waiting for I/O completion (disk, NFS, network)
  Until I/O completes, the kernel thread cannot be interrupted
  This is by design: data integrity — partial I/O could corrupt data

Common causes:
  1. NFS server unresponsive:
     Process writes to NFS mount → NFS RPC call hangs
     Can stay in D state indefinitely if NFS server never responds

  2. Disk I/O hang:
     Failing disk: I/O requests queued, never complete
     Hung NVMe/SCSI device: kernel waiting for device response

  3. Kernel bug or deadlock:
     Rare: kernel code stuck in lock

Diagnosing the hang:
  # Find all D state processes:
  ps aux | awk '$8 == "D" {print $0}'

  # Find what the process is waiting on:
  cat /proc/<PID>/wchan
  # nfs_wait_on_request → NFS hang
  # jbd2_log_wait_commit → filesystem journal hang
  # mutex_lock → kernel lock contention

  # Check kernel messages:
  dmesg | tail -50 | grep -i "nfs\|timeout\|hung"

  # Check if NFS is culprit:
  df -h    # hangs if NFS mount is unresponsive

Recovery without reboot:
  Option 1: Fix the underlying I/O (best):
    NFS server unresponsive → fix/restart NFS server
    Once I/O completes → D state process wakes up, can be killed normally

  Option 2: Force unmount NFS (risky — data loss possible):
    umount -f /mnt/nfs           # force unmount
    umount -l /mnt/nfs           # lazy unmount
    # After unmount: D state process gets I/O error → wakes up

  Option 3: SysRq for diagnostics:
    echo w > /proc/sysrq-trigger  # show all tasks in D state
    echo t > /proc/sysrq-trigger  # show all stack traces

  When you MUST reboot:
    D state caused by kernel deadlock (wchan shows lock function)
    NFS server completely gone and lazy unmount doesn't work
    Multiple D state processes hanging the entire system
```

💡 **Interview tip:** The answer that distinguishes a senior engineer: "SIGKILL cannot kill a D state process because the process is executing kernel code, not user code — signals are only delivered when the process returns to user space, which it never does while waiting for I/O." The practical investigation starts with `cat /proc/<PID>/wchan` — the kernel function name tells you exactly what the process is waiting for. `nfs_wait_on_request` immediately identifies an NFS issue. Recovery without reboot: fix the I/O source first; if impossible, force unmount the hung filesystem so the D state process receives an I/O error and wakes up.

---

### Q238 — Terraform | Conceptual | Advanced

> Explain **`check` blocks** and **`precondition`/`postcondition`** (Terraform 1.2+). Examples: AMI age, ALB subnets, external API reachability.

📁 **Reference:** `nawab312/Terraform` — validation, check blocks, preconditions

```
precondition (on resource/data source):
  Evaluated DURING plan/apply for a specific resource
  Blocks apply if condition is false
  Part of resource lifecycle

postcondition (on resource/data source):
  Evaluated AFTER the resource is created/read
  Validates the resource meets expectations
  If fails after creation: apply errors

check block (Terraform 1.5+):
  Standalone assertion NOT tied to a specific resource
  Evaluated AFTER all resources applied
  Does NOT block apply — shows as WARNING, not error
  For: validating external state, connectivity checks

Example 1 — Precondition: AMI not older than 90 days:
  resource "aws_instance" "app" {
    ami = data.aws_ami.app.id

    lifecycle {
      precondition {
        condition = (
          timecmp(data.aws_ami.app.creation_date,
                  timeadd(timestamp(), "-2160h")) > 0
        )
        error_message = "AMI ${data.aws_ami.app.id} is older than 90 days. Build a fresh AMI."
      }
    }
  }

Example 2 — Postcondition: ALB has 2+ subnets in different AZs:
  resource "aws_lb" "main" {
    name    = "production-alb"
    subnets = var.subnet_ids

    lifecycle {
      postcondition {
        condition = (
          length(distinct([for s in data.aws_subnet.selected : s.availability_zone])) >= 2
        )
        error_message = "ALB must span at least 2 availability zones."
      }
    }
  }

Example 3 — Check block: verify external API reachable:
  check "external_api_reachable" {
    data "http" "api_health" {
      url = "https://api.external-service.com/health"
    }
    assert {
      condition     = data.http.api_health.status_code == 200
      error_message = "External API is not healthy (got ${data.http.api_health.status_code})."
    }
  }
  # If API is down: Terraform shows WARNING but continues apply

Variable validation (plan time):
  variable "environment" {
    type = string
    validation {
      condition     = contains(["dev", "staging", "prod"], var.environment)
      error_message = "environment must be: dev, staging, or prod"
    }
  }

Comparison:
  variable validation: blocks plan, validates user input
  precondition:  blocks apply, tied to resource, before creation
  postcondition: blocks apply, tied to resource, after creation
  check block:   warning only, standalone, after apply
```

💡 **Interview tip:** The key distinction between `check` blocks and `precondition`/`postcondition` is severity — preconditions and postconditions BLOCK the apply (hard failures), while check blocks emit warnings without blocking (soft assertions). This makes check blocks ideal for monitoring external dependencies you don't control — you want to know if an external API is down, but you don't want that to prevent your infrastructure from being updated. Variable `validation` is often confused with preconditions — validation runs at plan time on user input, preconditions run at apply time against computed values.

---

### Q239 — GitHub Actions | Conceptual | Advanced

> Explain **`concurrency` groups**. Cancel in-progress on new push. Queue production deployments. Parallel CI but one deployment per environment.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions`

```
Problem concurrency solves:
  Multiple workflow runs for same branch running simultaneously
  → Race condition: last to finish wins (not last to start)
  → Earlier commits can overwrite later commits' deployments

concurrency group:
  A string identifier — all workflows with same group string
  compete for a single "slot"

cancel-in-progress:
  true  = cancel any in-progress run for same group when new run starts
  false = queue new run (wait for in-progress to finish)

Scenario 1 — Cancel in-progress CI on new push:
  concurrency:
    group: ${{ github.workflow }}-${{ github.ref }}
    # unique per branch → each branch has its own concurrency slot
    cancel-in-progress: true
    # When commit 3 arrives: cancel commit 2's run, start commit 3
    # Use for: CI builds — always want latest code tested

Scenario 2 — Queue production deployments (never cancel):
  concurrency:
    group: deploy-production
    cancel-in-progress: false   # queue, don't cancel
    # Use for: production deployments — each must complete fully

Scenario 3 — Parallel CI per branch, one deployment per environment:
  jobs:
    test:
      concurrency:
        group: test-${{ github.ref }}
        cancel-in-progress: true   # cancel outdated CI per branch

    deploy-staging:
      needs: test
      concurrency:
        group: deploy-staging
        cancel-in-progress: false  # queue, one at a time

    deploy-production:
      needs: deploy-staging
      environment: production      # + manual approval gate
      concurrency:
        group: deploy-production
        cancel-in-progress: false

Concurrency with PR workflows:
  concurrency:
    group: pr-${{ github.event.pull_request.number }}
    cancel-in-progress: true
  # Each PR has its own group (different from other PRs)
  # New push to same PR cancels previous CI run for that PR only
  # Different PRs run in parallel (different group names)
```

💡 **Interview tip:** The `cancel-in-progress` decision depends on workflow purpose: for CI (tests), cancel is usually right — you always want the latest code tested. For deployments, cancel is dangerous — a half-deployed state is worse than a slight delay. Queue for all deployments. The `group` string design is critical: using `${{ github.ref }}` makes the group per-branch, so different branches don't interfere with each other. A common mistake is using `${{ github.workflow }}` alone as the group — this means ALL branches of ALL PRs compete for one slot, serializing all CI runs globally.

---

### Q240 — AWS | Troubleshooting | Advanced

> Lambda times out after 15 seconds in AWS but completes in 2 seconds locally. Function queries RDS and calls external API.

📁 **Reference:** `nawab312/AWS` — Lambda, VPC configuration, cold starts

```
Local vs Lambda differences:
  Local:   runs on your machine with direct internet/DB access
  Lambda:  may run in VPC, has cold start, different network path

Cause 1 — Lambda in VPC but NO NAT Gateway (most common):
  Lambda in private subnet → can reach RDS (private IP)
  External API call → tries to go to internet → NO ROUTE
  Private subnet has no internet access without NAT Gateway

  Check:
  aws lambda get-function-configuration --function-name my-function \
    | jq '.VpcConfig'
  # Shows: SubnetIds → find subnet → check route table
  aws ec2 describe-route-tables \
    --filters "Name=association.subnet-id,Values=<lambda-subnet>"
  # If no route: 0.0.0.0/0 → nat-xxxxx → Lambda can't reach internet

  Fix:
  # Option A: Add NAT Gateway to VPC:
  aws ec2 create-nat-gateway --subnet-id <public-subnet-id> ...
  # Add route: 0.0.0.0/0 → nat-gateway in private subnet route table

  # Option B: VPC Interface Endpoint for AWS services (free):
  # For Secrets Manager, S3, etc. — no NAT needed

  # Option C: Move Lambda OUT of VPC if it doesn't need private resources

Cause 2 — DB connection NOT reused (cold start overhead):
  # BAD: new connection every invocation:
  def handler(event, context):
    conn = psycopg2.connect(...)   # 200-500ms overhead every call

  # GOOD: initialize outside handler (reused between warm invocations):
  conn = psycopg2.connect(...)     # runs once per container lifetime
  def handler(event, context):
    result = conn.execute(query)   # reuses existing connection

  Long-term: RDS Proxy for connection pooling at AWS level

Cause 3 — Lambda timeout set too low:
  aws lambda get-function-configuration --function-name my-function \
    | jq '.Timeout'
  # Default: 3 seconds! Cold start (1s) + RDS (2s) + external API (3s) = 6s needed

  Fix:
  aws lambda update-function-configuration \
    --function-name my-function --timeout 30

Cause 4 — Security group blocking RDS:
  Lambda SG must have outbound rule for port 5432
  RDS SG must have inbound rule from Lambda SG

  Quick test from Lambda:
  import socket
  socket.setdefaulttimeout(5)
  s = socket.socket()
  s.connect(('rds-endpoint.rds.amazonaws.com', 5432))
  # If times out → SG or routing issue

Cause 5 — Lambda cold start with VPC (pre-2019 issue):
  Before 2019: VPC Lambda cold start = 10-15 seconds (ENI attachment)
  After AWS optimization: < 1 second
  Check: CloudWatch Lambda InitDuration metric
  If still high: recreate function with current VPC configuration
```

💡 **Interview tip:** The most common cause is Lambda-in-VPC + no NAT Gateway — developers put Lambda in a VPC to access RDS (private), but forget that private subnets need a NAT Gateway to reach the internet for external API calls. The diagnostic is: check the Lambda's VPC config, find its subnets, check those subnets' route tables for a NAT gateway route. The second most common is DB connection not being reused — creating a new DB connection inside the handler on every invocation adds 200-500ms per call, which compounds with actual query time to exceed the default 3-second timeout.

---

### Q241 — Prometheus | Troubleshooting | Advanced

> Prometheus shows `ALERTS` firing but **Alertmanager is NOT sending notifications**. Alert firing in Prometheus UI but nothing in Alertmanager UI.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`

```
Pipeline: Prometheus → Alertmanager → notification

Step 1 — Confirm alert is actually firing (not PENDING):
  # Prometheus UI → Alerts tab:
  # PENDING: condition true but `for` duration not elapsed
  # FIRING: condition true AND for duration elapsed → sent to Alertmanager
  # Alertmanager only receives FIRING alerts (not PENDING)

  curl http://prometheus:9090/api/v1/alerts | jq '.data.alerts'

Step 2 — Check Prometheus → Alertmanager connectivity:
  # Prometheus UI → Status → Runtime & Build Information → Alertmanagers
  # Should show: alertmanager:9093 (active)

  # Check Prometheus config:
  curl http://prometheus:9090/api/v1/status/config | jq '.data.yaml' \
    | grep -A5 alerting
  # alerting: alertmanagers: [] ← EMPTY = alerts never sent!

  # Test connectivity from Prometheus pod:
  kubectl exec prometheus-0 -- curl http://alertmanager:9093/-/healthy
  # Connection refused → wrong service name or port

Step 3 — Check if alerts are reaching Alertmanager:
  curl http://alertmanager:9093/api/v2/alerts | jq '.'
  # Empty: alerts not reaching Alertmanager (connectivity or config issue)
  # Populated: Alertmanager has alerts but routing is broken

  # Prometheus logs:
  kubectl logs prometheus-0 | grep -i "alertmanager\|alert"
  # "Error sending alerts to alertmanager"

Step 4 — Check for active silence:
  curl http://alertmanager:9093/api/v2/silences | \
    jq '.[] | select(.status.state == "active")'
  # Active silence = alerts suppressed! Check if maintenance window forgotten

Step 5 — Check inhibition rules:
  # Another alert may be inhibiting this one:
  cat alertmanager.yml | grep -A10 inhibit_rules
  # source_match criteria matching a firing alert → target alerts inhibited

Step 6 — Check receiver configuration:
  kubectl logs alertmanager-0 | grep -i "error\|fail\|webhook"
  # "Error sending notification" → Slack webhook URL expired?
  # Test webhook manually:
  curl -X POST $SLACK_WEBHOOK -H 'Content-Type: application/json' \
    -d '{"text": "test"}'

Step 7 — Check group_wait buffer:
  route:
    group_wait: 30s
  # Alert fired 25 seconds ago → still in group_wait buffer
  # Will send after 30s — not a bug, just wait

Systematic diagnostic order:
  1. Is it FIRING (not PENDING) in Prometheus?
  2. Is Alertmanager configured and connected?
  3. Are alerts reaching Alertmanager API?
  4. Is there an active silence?
  5. Is there an inhibition rule suppressing it?
  6. Is the receiver (Slack/PD) webhook working?
```

💡 **Interview tip:** The most common cause is an active silence — someone silenced alerts during a maintenance window and forgot to expire it. The second most common is the `for` duration: the alert is in PENDING state and Alertmanager only receives FIRING alerts. Always check the Prometheus Alerts tab first — PENDING vs FIRING is the first diagnostic step. The systematic pipeline check (configured → connected → reaching AM → being routed → not silenced/inhibited → receiver working) covers all failure modes methodically.

---

### Q242 — Kubernetes | Conceptual | Advanced

> Explain **`ResourceQuota` for non-compute resources**. Explain **API Priority and Fairness (APF)** — FlowSchema and PriorityLevelConfiguration.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`, `01_CORE_ARCHITECTURE_FUNDAMENTALS.md`

```
ResourceQuota for non-compute resources:
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: team-quota
    namespace: production
  spec:
    hard:
      # Compute:
      requests.cpu: "10"
      requests.memory: "20Gi"

      # Object counts:
      count/pods: "50"
      count/services: "20"
      count/secrets: "50"
      count/configmaps: "30"
      count/persistentvolumeclaims: "10"
      count/deployments.apps: "20"

      # Storage:
      requests.storage: "100Gi"
      ssd.storageclass.storage.k8s.io/requests.storage: "50Gi"

      # Service types (expensive resources):
      services.loadbalancers: "2"   # only 2 cloud LBs allowed
      services.nodeports: "0"       # no NodePort services

API Priority and Fairness (APF):
  Problem: 100 CI/CD jobs run kubectl list pods every second
  → 100 constant LIST requests → API server CPU maxed
  → Critical controller manager requests starve
  → HPA can't scale, deployments can't roll out

  APF solution: each request type gets its own "lane"
  High priority lanes always get API server capacity first

  FlowSchema — classifies requests into flows:
    "Which requests belong to this category?"
    Matches by: user, group, service account, resource type, verb

  PriorityLevelConfiguration — defines capacity per level:
    "How much API server capacity does this category get?"

  Built-in priority levels (highest to lowest):
    exempt:           system:masters — never queued
    node-high:        kubelet NodeStatus updates
    system:           kube-system service accounts
    leader-election:  scheduler/controller-manager
    workload-high:    kube-controller-manager, kube-scheduler
    workload-low:     other controllers
    global-default:   everyone else (kubectl by users)

  FlowSchema — rate limit CI/CD tools:
  apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
  kind: FlowSchema
  metadata:
    name: ci-cd-limited
  spec:
    priorityLevelConfiguration:
      name: global-default     # low priority bucket
    rules:
    - subjects:
      - kind: ServiceAccount
        serviceAccount:
          name: jenkins-agent
          namespace: ci
      resourceRules:
      - verbs: ["list", "watch"]
        resources: ["pods", "deployments"]

  Monitor APF:
  kubectl get --raw /metrics | grep apiserver_flowcontrol_rejected_requests_total
  # Increasing value → requests being throttled
```

💡 **Interview tip:** The non-compute ResourceQuota options most interviewers don't know: `services.loadbalancers` limits cloud load balancers (expensive!), `count/secrets` prevents secret sprawl, and storage class-specific quotas like `ssd.storageclass.storage.k8s.io/requests.storage` give fine-grained storage budget control per class. For APF, the key production scenario is CI/CD tools doing mass list operations starving critical controllers — FlowSchema lets you put CI/CD requests in a low-priority bucket without blocking them entirely, just throttling them when the API server is busy with more important work.

---

### Q243 — ArgoCD | Scenario-Based | Advanced

> Implement **ArgoCD Image Updater** — how it works, semver tracking, Git write-back, prevent auto-update to production.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — Image Updater

```
How ArgoCD Image Updater works:
  1. Polls container registry (ECR, Docker Hub) for new image tags
  2. Compares against update strategy (semver, latest, digest)
  3. When new qualifying tag found:
     Write-back git: commits new image tag to Git repo
                     ArgoCD syncs from Git (proper GitOps)
     Write-back argocd: updates Application directly (less pure GitOps)

Install Image Updater:
  kubectl apply -n argocd \
    -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

  # IRSA for ECR access:
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: argocd-image-updater
    namespace: argocd
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::123:role/image-updater-role
  # IAM role needs: ecr:DescribeImages, ecr:ListImages

Configure Application for semver tracking:
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: payment-api-staging
    annotations:
      # Track this image:
      argocd-image-updater.argoproj.io/image-list: >
        payment=123456789.dkr.ecr.us-east-1.amazonaws.com/payment-api

      # Semver — only update on new MINOR versions:
      argocd-image-updater.argoproj.io/payment.update-strategy: semver
      argocd-image-updater.argoproj.io/payment.allow-tags: "^1\\..*"
      # Allows: 1.2.0, 1.3.0 (minor updates)
      # Blocks: 2.0.0 (new major — breaking change, human review needed)

      # Write-back to Git (proper GitOps):
      argocd-image-updater.argoproj.io/write-back-method: git
      argocd-image-updater.argoproj.io/git-branch: main
      argocd-image-updater.argoproj.io/write-back-target: "helmvalues:./helm/values-staging.yaml"

What Image Updater commits to Git:
  # Before: values-staging.yaml
  image:
    tag: 1.2.3

  # Image Updater pushes commit: "build: automatic update to 1.3.0"
  image:
    tag: 1.3.0
  # ArgoCD detects Git change → syncs → deploys 1.3.0

Prevent auto-update to production:
  # Production Application — NO image updater annotations:
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: payment-api-production
    # No argocd-image-updater.argoproj.io/* annotations = not tracked
  spec:
    source:
      helm:
        valueFiles:
        - values-prod.yaml   # manually managed

  # Promotion workflow:
  # 1. Image Updater auto-updates staging to 1.3.0
  # 2. Staging passes integration tests
  # 3. Human PR: update values-prod.yaml image.tag from 1.2.3 → 1.3.0
  # 4. PR reviewed → merged → ArgoCD syncs production
```

💡 **Interview tip:** Write-back to Git (not directly to the ArgoCD Application) is the correct GitOps approach — it maintains Git as the source of truth. The Image Updater commits a "chore: update image tag" commit, and ArgoCD picks it up through its normal sync mechanism. The semver constraint `^1\\..*` is important — it prevents major version bumps (which may have breaking changes) from being auto-deployed, requiring human review. The production safety pattern (no annotations = no auto-update) is simpler and more reliable than trying to configure Image Updater to skip production.

---

### Q244 — Linux / Bash | Scenario-Based | Advanced

> Write a **database backup validator** — restore `.sql.gz` to temp Docker container, run validation queries, cleanup on exit, exit code 0/1.

📁 **Reference:** `nawab312/DSA` → `Linux`

```bash
#!/bin/bash
set -euo pipefail

BACKUP_FILE="${1:?Usage: $0 <backup.sql.gz>}"
CONTAINER_NAME="pg-validate-$$"   # $$ = current PID, unique per run
PG_IMAGE="postgres:15"
PG_PASSWORD="validate_temp_$(date +%s)"
PASS=0; FAIL=0; ERRORS=()

log() { echo "[$(date '+%H:%M:%S')] $*"; }
fail() { log "FAIL: $*"; ERRORS+=("$*"); ((FAIL++)); }
pass() { log "PASS: $*"; ((PASS++)); }

# Cleanup: always runs on exit (success or failure)
cleanup() {
  log "Cleaning up container: $CONTAINER_NAME"
  docker rm -f "$CONTAINER_NAME" 2>/dev/null || true
}
trap cleanup EXIT

[ -f "$BACKUP_FILE" ] || { echo "ERROR: File not found"; exit 1; }

# Start PostgreSQL container
log "Starting PostgreSQL container..."
docker run -d --name "$CONTAINER_NAME" \
  -e POSTGRES_PASSWORD="$PG_PASSWORD" \
  -e POSTGRES_DB=testdb \
  "$PG_IMAGE" > /dev/null

# Wait for PostgreSQL ready
for i in $(seq 1 30); do
  docker exec "$CONTAINER_NAME" pg_isready -U postgres -q 2>/dev/null && break
  [ "$i" -eq 30 ] && { log "ERROR: PostgreSQL failed to start"; exit 1; }
  sleep 1
done

# Restore backup
log "Restoring backup..."
zcat "$BACKUP_FILE" | docker exec -i "$CONTAINER_NAME" \
  psql -U postgres -d testdb -q 2>/tmp/restore-errors.log \
  || { cat /tmp/restore-errors.log; exit 1; }

# Helper: run SQL query
run_query() {
  docker exec "$CONTAINER_NAME" psql -U postgres -d testdb -t -c "$1" 2>/dev/null | tr -d ' '
}

# Validation 1: Database accessible
run_query "SELECT 1;" | grep -q "1" && pass "Database connectivity" || fail "Database connectivity"

# Validation 2: Key tables exist
for table in users orders products payments; do
  count=$(run_query "SELECT COUNT(*) FROM information_schema.tables
    WHERE table_schema='public' AND table_name='${table}';")
  [ "$count" -eq 1 ] && pass "Table exists: $table" || fail "Table missing: $table"
done

# Validation 3: Row counts non-zero
for table in users products; do
  count=$(run_query "SELECT COUNT(*) FROM ${table};" 2>/dev/null || echo "0")
  [ "$count" -gt 0 ] && pass "Table $table has $count rows" || fail "Table $table is empty"
done

# Validation 4: Referential integrity
fk_violations=$(run_query "SELECT COUNT(*) FROM orders o
  LEFT JOIN users u ON o.user_id = u.id WHERE u.id IS NULL;")
[ "$fk_violations" -eq 0 ] \
  && pass "Referential integrity: no orphaned orders" \
  || fail "Referential integrity: $fk_violations orphaned orders"

# Validation 5: Backup not older than 25 hours
backup_age_h=$(( ($(date +%s) - $(stat -c %Y "$BACKUP_FILE")) / 3600 ))
[ "$backup_age_h" -lt 25 ] \
  && pass "Backup age: ${backup_age_h}h (within window)" \
  || fail "Backup age: ${backup_age_h}h (older than 25 hours)"

# Summary
echo "=========================================="
echo "Backup Validation: PASSED=$PASS FAILED=$FAIL"
[ ${#ERRORS[@]} -gt 0 ] && printf "  - %s\n" "${ERRORS[@]}"
[ "$FAIL" -eq 0 ] && exit 0 || exit 1
```

💡 **Interview tip:** The `trap cleanup EXIT` pattern is the critical safety mechanism — it ensures the Docker container is removed whether the script succeeds, fails, or is interrupted with Ctrl+C. Without it, a failed script leaves orphaned containers running. The `$$` (current PID) in the container name ensures uniqueness when multiple validation runs execute in parallel in CI. The referential integrity check is the most valuable validation — it catches cases where the backup is technically complete but has data consistency issues that would cause application errors in production.

---

### Q245 — AWS | Scenario-Based | Advanced

> Write **SCPs**: prevent CloudTrail disable, restrict to approved regions, prevent leaving Organizations, require CostCenter tag, restrict RDS management to specific roles.

📁 **Reference:** `nawab312/AWS` — AWS Organizations, SCPs

```json
// SCP 1 — Prevent disabling CloudTrail:
{
  "Statement": [{
    "Sid": "PreventCloudTrailDisable",
    "Effect": "Deny",
    "Action": [
      "cloudtrail:DeleteTrail",
      "cloudtrail:StopLogging",
      "cloudtrail:UpdateTrail"
    ],
    "Resource": "*",
    "Condition": {
      "ArnNotLike": {
        "aws:PrincipalArn": ["arn:aws:iam::*:role/BreakGlassAdmin"]
      }
    }
  }]
}

// SCP 2 — Restrict to approved regions only:
// MUST use NotAction (not Action) — global services (IAM, Route53)
// have no region, would be blocked if using Action: "*"
{
  "Statement": [{
    "Sid": "DenyNonApprovedRegions",
    "Effect": "Deny",
    "NotAction": [
      "iam:*", "organizations:*", "route53:*",
      "budgets:*", "cloudfront:*", "sts:*", "support:*"
    ],
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
      }
    }
  }]
}

// SCP 3 — Prevent leaving AWS Organizations:
{
  "Statement": [{
    "Effect": "Deny",
    "Action": "organizations:LeaveOrganization",
    "Resource": "*"
  }]
}

// SCP 4 — Require CostCenter tag on EC2:
{
  "Statement": [{
    "Effect": "Deny",
    "Action": "ec2:RunInstances",
    "Resource": "arn:aws:ec2:*:*:instance/*",
    "Condition": {
      "Null": {"aws:RequestTag/CostCenter": "true"}
    }
  }]
}

// SCP 5 — Restrict production RDS management to specific roles:
{
  "Statement": [{
    "Effect": "Deny",
    "Action": [
      "rds:CreateDBInstance", "rds:DeleteDBInstance",
      "rds:ModifyDBInstance", "rds:CreateDBCluster", "rds:DeleteDBCluster"
    ],
    "Resource": "*",
    "Condition": {
      "ArnNotLike": {
        "aws:PrincipalArn": [
          "arn:aws:iam::*:role/DatabaseAdmin",
          "arn:aws:iam::*:role/TerraformDeployRole",
          "arn:aws:iam::*:role/BreakGlassAdmin"
        ]
      }
    }
  }]
}
```

💡 **Interview tip:** The region restriction SCP using `NotAction` instead of `Action` is the most commonly misunderstood pattern — using `Action: "*"` with a region condition would also block IAM API calls (which are global and have no `aws:RequestedRegion`). The `NotAction` approach says "deny everything EXCEPT these global services when not in approved regions." The break-glass role pattern in the CloudTrail SCP is a production best practice — it allows a specifically named emergency role to bypass the protection while blocking all other principals. SCPs apply to ALL principals in the account including root — deny + no allow elsewhere = denied even with AdministratorAccess.

---

### Q246 — Kubernetes | Troubleshooting | Advanced

> **StatefulSet stuck in rolling update for 2 hours** — pod `my-app-1` is `Terminating` but never finishes. Diagnose and safely resolve without data loss.

📁 **Reference:** `nawab312/Kubernetes` → `02_WORKLOADS.md`, `14_TROUBLESHOOTING.md`

```
Why StatefulSet rolling update is ordered:
  Updates in REVERSE order: N → N-1 → ... → 0
  Each pod must fully terminate before the next is updated
  If my-app-1 stuck Terminating → entire update blocked

Step 1 — Check for finalizers (most common cause):
  kubectl get pod my-app-1 -n production -o json | jq '.metadata.finalizers'
  # If output: ["foregroundDeletion"] or ["custom.finalizer/protection"]
  # Finalizer is blocking deletion

  # Is the finalizer controller running?
  kubectl get pods -n production | grep finalizer-controller
  # If not found → controller crashed → finalizer never removed

Step 2 — Check for stuck volume unmount:
  kubectl describe pod my-app-1 | grep -A5 "Events:"
  # "Unable to unmount volumes for pod" → EBS volume stuck

  # Check VolumeAttachment objects:
  kubectl get volumeattachment
  # If still "attached" to a gone node → stuck unmount

  # Force detach EBS in AWS:
  aws ec2 detach-volume --volume-id vol-xxx --force

Step 3 — Check for preStop hook hanging:
  kubectl describe pod my-app-1 | grep -i prestop
  # If preStop hook never finishes → pod waits forever
  # Check terminationGracePeriodSeconds (default 30s won't help if hook hangs)

Step 4 — Safe resolution (remove finalizer manually):
  # WARNING: only after confirming data is safe (not mid-transaction)

  kubectl patch pod my-app-1 -n production \
    -p '{"metadata":{"finalizers":[]}}' \
    --type=merge
  # Pod should now complete termination

  # OR surgical JSON patch:
  kubectl patch pod my-app-1 -n production \
    --type json \
    -p '[{"op":"remove","path":"/metadata/finalizers"}]'

Step 5 — Check for stuck PVC:
  kubectl get pvc -n production | grep my-app-1
  # If PVC status: Terminating → remove PVC finalizer too:
  kubectl patch pvc my-app-1-data -n production \
    --type json \
    -p '[{"op":"remove","path":"/metadata/finalizers"}]'

Step 6 — Verify StatefulSet resumes:
  kubectl get pods -n production -w
  kubectl rollout status statefulset/my-app -n production
```

💡 **Interview tip:** The finalizer is almost always the culprit for stuck Terminating pods — a finalizer controller that has crashed leaves pods in permanent Terminating state. The safe resolution is manually removing the finalizer via `kubectl patch`, but ONLY after confirming the application has actually shut down cleanly (check logs, verify no active transactions). Removing a finalizer without verifying application state can cause data corruption if the application was mid-write. For StatefulSets, the ordered termination means a single stuck pod blocks the entire rolling update with no way to skip it.

---

### Q247 — Grafana | Troubleshooting | Advanced

> Grafana shows data 5.5 hours behind Prometheus — time zone issue. Explain `$__timeFilter`, `$__interval`, `$__range` template variables.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana`

```
Why 5.5 hours offset:
  5.5 hours = UTC +5:30 = India Standard Time (IST)
  Cause: Prometheus stores timestamps in UTC
         Grafana configured for a different timezone
         OR browser's local timezone applied incorrectly

Diagnosis and fixes:
  Fix 1 — Check Grafana profile timezone:
    Grafana UI → User menu → Profile → Timezone → set to "UTC"

  Fix 2 — Check dashboard timezone override:
    Dashboard → Settings → Time options → Timezone
    Set to "UTC" or "Browser"

  Fix 3 — Check Prometheus server time:
    kubectl exec prometheus-0 -- date   # should match UTC
    # If Prometheus is in IST:
    # Add to pod spec: env: [{name: TZ, value: UTC}]

  Fix 4 — Check data source configuration:
    Grafana → Data Sources → Prometheus → "Custom query parameters"
    Verify no timezone parameter passed incorrectly

Template variables:

  $__timeFilter (for SQL/log datasources):
    Used in: Elasticsearch, MySQL, PostgreSQL, Loki queries
    Expands to dashboard time range as a filter condition
    Example in MySQL:
      SELECT * FROM metrics WHERE $__timeFilter(timestamp)
    Expands to:
      WHERE timestamp BETWEEN '2024-01-15 10:00:00' AND '2024-01-15 14:00:00'
    NOT used for: Prometheus (Prometheus handles time range via URL params)

  $__interval (auto-calculated step for current view):
    Calculated as: (time range width) / (graph pixels) ≈ right step size
    Example: 1-hour range, 500px → $__interval = ~7s
             24-hour range, 500px → $__interval = ~172s
    Use in PromQL:
      rate(http_requests_total[$__interval])   # GOOD: adapts to zoom level
      rate(http_requests_total[5m])            # BAD: too coarse when zoomed in
    $__interval auto-adapts → always appropriate granularity

  $__range (total duration of current time range):
    Example: dashboard shows "Last 6 hours" → $__range = "6h"
    Use when you need the full range as a duration:
      avg_over_time(cpu_usage[$__range])   # average over entire visible period

  Choosing between them:
    $__interval:   rate calculations, per-step aggregations (most common)
    $__range:      full-range aggregations (min/max/avg over visible period)
    $__timeFilter: SQL/log datasources WHERE clauses (not Prometheus)
```

💡 **Interview tip:** The 5.5-hour offset is a very specific clue pointing to IST (India Standard Time) — a common production issue when Prometheus runs in UTC but the developer's browser or Grafana instance is in a different timezone. Fix order: check Grafana profile timezone first (user-level), then dashboard settings (dashboard-level), then Prometheus server time. For template variables, `$__interval` vs hardcoded duration is the important best practice — using `[5m]` in PromQL looks reasonable but produces too coarse data when zoomed into 5 minutes and too fine data when viewing 7 days. `$__interval` always gives the right granularity for the current zoom level.

---

### Q248 — Terraform | Troubleshooting | Advanced

> Terraform destroy fails with "resource still referencing this" and "BucketNotEmpty". Handle each safely.

📁 **Reference:** `nawab312/Terraform` — destroy dependencies, S3 force destroy

```
Error 1 — Dependency preventing destroy:
  Error: Instance cannot be destroyed.
  A resource is still referencing this resource.
  OR:
  Error: deleting Security Group (sg-xxx): DependencyViolation

Diagnose:
  terraform graph | grep "→.*my-resource"
  # Shows what depends on the resource you're trying to destroy

  # For AWS SG dependency — find what still references it:
  aws ec2 describe-network-interfaces \
    --filters "Name=group-id,Values=sg-xxx"
  # Shows: Lambda functions, RDS, ELB, EC2 still using this SG

Fixes:
  Option A — Target destroy in correct order:
    terraform destroy -target=aws_instance.web
    terraform destroy -target=aws_security_group.web
    terraform destroy -target=aws_vpc.main

  Option B — Remove prevent_destroy temporarily:
    lifecycle {
      # prevent_destroy = true   # comment out for destroy
    }
    terraform apply -refresh-only
    terraform destroy

  Option C — Manual cleanup + state rm:
    # Manually delete the dependent resource in AWS console
    terraform state rm aws_security_group.web
    terraform destroy   # now SG is gone, VPC can destroy

Error 2 — S3 BucketNotEmpty:
  Error: deleting S3 Bucket: BucketNotEmpty

  Fix A — Add force_destroy to resource:
    resource "aws_s3_bucket" "this" {
      bucket        = "my-bucket"
      force_destroy = true   # delete all objects before deleting bucket
    }
    terraform apply -target=aws_s3_bucket.this  # update force_destroy
    terraform destroy -target=aws_s3_bucket.this

  Fix B — Empty bucket manually:
    aws s3 rm s3://my-bucket --recursive
    # For versioned buckets (must delete versions too):
    aws s3api delete-objects \
      --bucket my-bucket \
      --delete "$(aws s3api list-object-versions \
        --bucket my-bucket \
        --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}')"
    terraform destroy -target=aws_s3_bucket.this

  Fix C — AWS CLI delete + state rm:
    aws s3 rb s3://my-bucket --force
    terraform state rm aws_s3_bucket.this

Destroy safety checklist:
  1. terraform plan -destroy    # preview what will be destroyed
  2. Check for unexpected destroys
  3. Back up data (RDS snapshots, S3 versioning)
  4. Use -target for surgical destroy (NOT terraform destroy on everything)
```

💡 **Interview tip:** `force_destroy = true` on S3 is a dangerous default to leave on in production — it means `terraform destroy` will silently delete all bucket objects with no confirmation. Only add it when you specifically need to destroy a bucket that will have objects (like a logging bucket in a test environment). For the dependency error, `terraform graph` is the right debugging tool — it shows the exact dependency chain. In production, always use `-target` destroy in the correct order rather than a full destroy, which prevents accidentally destroying unrelated resources.

---

### Q249 — Python | Scenario-Based | Advanced

> Write a **Python AWS resource tagger** — scan EC2/RDS/S3 across multiple regions, find missing required tags, CSV report, `--fix` flag, concurrent scanning.

📁 **Reference:** `nawab312/DSA` → `Python` — boto3, concurrent.futures

```python
#!/usr/bin/env python3

import argparse
import boto3
import csv
import sys
from concurrent.futures import ThreadPoolExecutor, as_completed
from datetime import datetime

REQUIRED_TAGS = {"Environment", "Team", "CostCenter"}
REGIONS = ["us-east-1", "us-west-2", "eu-west-1", "ap-southeast-1"]
DEFAULT_TAGS = {"Environment": "unknown", "Team": "unassigned", "CostCenter": "0000"}


def get_missing_tags(resource_tags: list) -> list:
    existing = {t["Key"] for t in resource_tags}
    return sorted(REQUIRED_TAGS - existing)


def scan_ec2(region: str, fix: bool, account_id: str) -> list:
    violations = []
    ec2 = boto3.client("ec2", region_name=region)
    try:
        paginator = ec2.get_paginator("describe_instances")
        for page in paginator.paginate():
            for reservation in page["Reservations"]:
                for instance in reservation["Instances"]:
                    if instance["State"]["Name"] == "terminated":
                        continue
                    tags = instance.get("Tags", [])
                    missing = get_missing_tags(tags)
                    if missing:
                        rid = instance["InstanceId"]
                        violations.append({
                            "AccountId": account_id, "Region": region,
                            "ResourceType": "EC2Instance", "ResourceId": rid,
                            "MissingTags": ", ".join(missing),
                        })
                        if fix:
                            ec2.create_tags(
                                Resources=[rid],
                                Tags=[{"Key": k, "Value": DEFAULT_TAGS[k]} for k in missing]
                            )
    except Exception as e:
        print(f"  ERROR EC2 {region}: {e}", file=sys.stderr)
    return violations


def scan_rds(region: str, fix: bool, account_id: str) -> list:
    violations = []
    rds = boto3.client("rds", region_name=region)
    try:
        paginator = rds.get_paginator("describe_db_instances")
        for page in paginator.paginate():
            for db in page["DBInstances"]:
                arn = db["DBInstanceArn"]
                tags = rds.list_tags_for_resource(ResourceName=arn).get("TagList", [])
                missing = get_missing_tags(tags)
                if missing:
                    violations.append({
                        "AccountId": account_id, "Region": region,
                        "ResourceType": "RDSInstance",
                        "ResourceId": db["DBInstanceIdentifier"],
                        "MissingTags": ", ".join(missing),
                    })
                    if fix:
                        rds.add_tags_to_resource(
                            ResourceName=arn,
                            Tags=[{"Key": k, "Value": DEFAULT_TAGS[k]} for k in missing]
                        )
    except Exception as e:
        print(f"  ERROR RDS {region}: {e}", file=sys.stderr)
    return violations


def scan_s3(fix: bool, account_id: str) -> list:
    """S3 is global — scan once."""
    violations = []
    s3 = boto3.client("s3", region_name="us-east-1")
    for bucket in s3.list_buckets().get("Buckets", []):
        name = bucket["Name"]
        try:
            tags = s3.get_bucket_tagging(Bucket=name).get("TagSet", [])
        except Exception:
            tags = []
        missing = get_missing_tags(tags)
        if missing:
            violations.append({
                "AccountId": account_id, "Region": "global",
                "ResourceType": "S3Bucket", "ResourceId": name,
                "MissingTags": ", ".join(missing),
            })
            if fix:
                existing = [t for t in tags if t["Key"] not in REQUIRED_TAGS]
                existing.extend([{"Key": k, "Value": DEFAULT_TAGS[k]} for k in missing])
                s3.put_bucket_tagging(Bucket=name, Tagging={"TagSet": existing})
    return violations


def scan_region(region: str, fix: bool, account_id: str) -> list:
    print(f"Scanning region: {region}...")
    results = []
    results.extend(scan_ec2(region, fix, account_id))
    results.extend(scan_rds(region, fix, account_id))
    return results


def main():
    parser = argparse.ArgumentParser(description="AWS Resource Tag Compliance Scanner")
    parser.add_argument("--fix", action="store_true", help="Apply default tags")
    parser.add_argument("--output", default=f"tag-report-{datetime.now():%Y%m%d-%H%M}.csv")
    parser.add_argument("--regions", nargs="+", default=REGIONS)
    args = parser.parse_args()

    account_id = boto3.client("sts").get_caller_identity()["Account"]
    print(f"Account: {account_id} | Mode: {'FIX' if args.fix else 'CHECK'}")

    all_violations = []

    # Parallel region scanning (max 5 concurrent):
    with ThreadPoolExecutor(max_workers=5) as executor:
        futures = {
            executor.submit(scan_region, r, args.fix, account_id): r
            for r in args.regions
        }
        for future in as_completed(futures):
            try:
                all_violations.extend(future.result())
            except Exception as e:
                print(f"ERROR: {e}", file=sys.stderr)

    all_violations.extend(scan_s3(args.fix, account_id))

    if all_violations:
        with open(args.output, "w", newline="") as f:
            writer = csv.DictWriter(f, fieldnames=list(all_violations[0].keys()))
            writer.writeheader()
            writer.writerows(all_violations)
        print(f"\nFound {len(all_violations)} non-compliant resources → {args.output}")
    else:
        print("\nAll resources are compliant!")

    sys.exit(1 if all_violations and not args.fix else 0)


if __name__ == "__main__":
    main()
```

💡 **Interview tip:** `ThreadPoolExecutor` with `as_completed` is the right pattern for parallel I/O-bound work (AWS API calls) — it starts all region scans simultaneously and processes results as they complete. S3 is global (not per-region) — scanning it in every region loop would create duplicate results. The `sys.exit(1 if all_violations and not args.fix else 0)` exit code logic is important for CI: exit 1 when non-compliant resources are found (fails the CI check), but exit 0 when `--fix` was used and tags were applied (compliant after fix).

---

### Q250 — AWS | Conceptual | Advanced

> Explain **Direct Connect vs AWS VPN** — bandwidth, latency, reliability, cost. **Direct Connect Gateway** for multi-VPC/region.

📁 **Reference:** `nawab312/AWS` — Direct Connect, VPN, hybrid connectivity

```
AWS VPN (Site-to-Site):
  Technology:   IPsec tunnel over public internet
  Setup time:   minutes (automated)
  Bandwidth:    1.25 Gbps max per tunnel
  Latency:      variable (depends on internet conditions)
  Reliability:  depends on ISP, internet routing
  Cost:         $0.05/hr per connection + data transfer
  Encryption:   built-in (IPsec)

AWS Direct Connect:
  Technology:   dedicated physical network connection
  Setup time:   weeks to months (physical provisioning)
  Bandwidth:    1/10/100 Gbps dedicated
  Latency:      consistent, low (dedicated fiber, no internet routing)
  Reliability:  SLA-backed, not dependent on internet
  Cost:         $0.30/hr (1Gbps port) + data transfer
  Encryption:   NOT built-in (add MACsec or IPsec over DX for encryption)

When to use each:
  VPN:    smaller bandwidth, quick setup, dev/staging, backup connectivity
  DX:     large data transfer (>1TB/month), consistent latency required,
          production workloads, compliance requiring private circuits
  Best:   DX + VPN backup (failover to VPN if DX circuit fails)

Direct Connect Gateway:
  Problem without DX Gateway:
    One DX connection connects to ONE VPC only
    10 VPCs in 3 regions = 10 Virtual Interfaces — very expensive

  DX Gateway solution:
    Single DX connection → DX Gateway → up to 10 VPCs in ANY region

    On-premises
        │ Direct Connect (10Gbps)
        ▼
    Virtual Interface
        ▼
    Direct Connect Gateway (global)
        ├──► VPC us-east-1
        ├──► VPC eu-west-1
        └──► VPC ap-southeast-1
    (can be different regions!)

  DX Gateway + Transit Gateway (most scalable):
    DX → DX Gateway → Transit Gateway → all VPCs in region
    TGW handles inter-VPC routing, DX Gateway handles on-prem connectivity

  Use cases:
    Financial services: on-prem → AWS in multiple regions
    Enterprise: global VPCs with consistent low-latency to HQ
    SAP HANA: high-bandwidth from on-prem ERP systems

DX encryption note:
  DX is a dedicated circuit but NOT encrypted by default
  For PCI/HIPAA: must add MACsec (Layer 2) or run IPsec VPN over DX
  → Encrypted + dedicated low-latency = most secure hybrid connection
```

💡 **Interview tip:** The most important operational note: Direct Connect is NOT encrypted by default, which surprises many engineers who assume a dedicated connection means it's secure. For regulated industries (PCI, HIPAA), you must add MACsec or run IPsec VPN over the DX connection. The DX + VPN backup pattern is the production best practice — DX for normal operations (low latency, high bandwidth), VPN as automatic failover. Without a VPN backup, a DX circuit outage (hardware failure at the colocation facility) completely disconnects your on-premises from AWS.

---

### Q251 — Kubernetes | Scenario-Based | Advanced

> Complete **IRSA setup** for 3 pods: Pod A needs S3 GetObject, Pod B needs DynamoDB PutItem, Pod C needs Secrets Manager GetSecretValue. Least privilege.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md`, `nawab312/AWS` — IRSA, OIDC

```
Step 1 — Get EKS OIDC provider URL:
  OIDC_URL=$(aws eks describe-cluster --name my-cluster \
    --query "cluster.identity.oidc.issuer" \
    --output text | sed 's/https:\/\///')

Step 2 — Create IAM roles (Terraform example for Pod A):
  resource "aws_iam_role" "pod_a" {
    name = "pod-a-s3-role"
    assume_role_policy = jsonencode({
      Statement = [{
        Effect = "Allow"
        Principal = {
          Federated = "arn:aws:iam::${var.account_id}:oidc-provider/${local.oidc_provider}"
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${local.oidc_provider}:sub" = "system:serviceaccount:production:pod-a-sa"
            "${local.oidc_provider}:aud" = "sts.amazonaws.com"
          }
        }
      }]
    })
  }

  resource "aws_iam_role_policy" "pod_a" {
    role = aws_iam_role.pod_a.id
    policy = jsonencode({
      Statement = [{
        Effect   = "Allow"
        Action   = ["s3:GetObject"]
        Resource = "arn:aws:s3:::company-data-bucket/*"  # specific bucket only
      }]
    })
  }

  # Pod B — DynamoDB specific table only:
  Action   = ["dynamodb:PutItem"]
  Resource = "arn:aws:dynamodb:us-east-1:123:table/orders"

  # Pod C — Secrets Manager specific secrets only:
  Action   = ["secretsmanager:GetSecretValue"]
  Resource = ["arn:aws:secretsmanager:us-east-1:123:secret:prod/service-c/*"]

Step 3 — Create annotated ServiceAccounts:
  kubectl create serviceaccount pod-a-sa -n production
  kubectl annotate serviceaccount pod-a-sa -n production \
    eks.amazonaws.com/role-arn=arn:aws:iam::123:role/pod-a-s3-role

  kubectl create serviceaccount pod-b-sa -n production
  kubectl annotate serviceaccount pod-b-sa -n production \
    eks.amazonaws.com/role-arn=arn:aws:iam::123:role/pod-b-dynamodb-role

Step 4 — Reference ServiceAccount in Deployment:
  spec:
    serviceAccountName: pod-a-sa   # references the annotated SA
    containers:
    - name: app
      image: company/pod-a:v1
      # Webhook auto-injects:
      # AWS_ROLE_ARN = arn:aws:iam::123:role/pod-a-s3-role
      # AWS_WEB_IDENTITY_TOKEN_FILE = /var/run/secrets/eks.amazonaws.com/.../token
      # AWS SDK reads these automatically — no code changes needed

Step 5 — Verify IRSA:
  kubectl exec pod-a-xxx -- aws sts get-caller-identity
  # Should show: assumed-role/pod-a-s3-role/...

  # Pod A can access S3:
  kubectl exec pod-a-xxx -- aws s3 cp s3://company-data-bucket/test.txt -

  # Pod A CANNOT access DynamoDB:
  kubectl exec pod-a-xxx -- aws dynamodb list-tables
  # Error: AccessDeniedException — correct! least privilege working
```

💡 **Interview tip:** The `StringEquals` condition with both `sub` (ServiceAccount) and `aud` (audience) in the trust policy is essential for security — the `sub` condition ensures only the specific ServiceAccount in the specific namespace can assume this role, not any pod in the cluster. Without it, any pod with an OIDC token could assume the role. The resource ARN in the IAM policy should always be specific to the exact bucket/table/secret — never use `*` for resource, even in test. `aws sts get-caller-identity` from inside the pod is the quick verification that IRSA is working correctly.

---

### Q252 — ELK Stack | Scenario-Based | Advanced

> Elasticsearch cluster `RED` — unassigned primary shards. Complete diagnosis based on root cause (disk watermark, node failure, replica config, allocation settings).

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `ELK_Stack`

```
Step 1 — Assess cluster health:
  curl http://localhost:9200/_cluster/health?pretty
  # status: "red", unassigned_shards: 15

Step 2 — Find unassigned shards and WHY:
  curl "http://localhost:9200/_cat/shards?v&h=index,shard,prirep,state,unassigned.reason,node"
  # logs-2024.01  0  p  UNASSIGNED  NODE_LEFT
  # logs-2024.01  2  p  UNASSIGNED  ALLOCATION_FAILED

Step 3 — Get allocation explanation (most useful command):
  curl -X GET "localhost:9200/_cluster/allocation/explain?pretty" \
    -H 'Content-Type: application/json' \
    -d '{"index":"logs-2024.01","shard":0,"primary":true}'
  # This tells you in plain English WHY the shard cannot be allocated

Common explanations and fixes:

  Explanation A — Disk watermark exceeded:
    "node is above the high watermark [85%]"
    Fix: free disk space OR increase watermark:
    curl -X PUT "localhost:9200/_cluster/settings" -d '{
      "persistent": {
        "cluster.routing.allocation.disk.watermark.high": "95%",
        "cluster.routing.allocation.disk.watermark.flood_stage": "98%"
      }
    }'
    # Trigger reallocation after fixing:
    curl -X POST "localhost:9200/_cluster/reroute?retry_failed=true"

  Explanation B — No valid shard copy (node failure):
    "no valid shard copy found for shard id"
    → Primary AND all replicas were on dead nodes
    Fix: restore from snapshot (preferred, no data loss):
      curl -X POST "localhost:9200/_snapshot/my-repo/snapshot-1/_restore" \
        -d '{"indices": "logs-2024.01", "include_global_state": false}'
    Last resort (DATA LOSS):
      curl -X POST "localhost:9200/_cluster/reroute" \
        -d '{"commands":[{"allocate_stale_primary":{
          "index":"logs-2024.01","shard":0,"node":"data-node-2",
          "accept_data_loss":true}}]}'

  Explanation C — Allocation filter pointing to dead node:
    "node does not match index.routing.allocation.include._name"
    Fix: clear the routing allocation filter:
    curl -X PUT "localhost:9200/logs-2024.01/_settings" \
      -d '{"index.routing.allocation.require._name": null}'

  Explanation D — Single node with replica count > 0:
    "cluster has 1 node and replica is set to 1"
    Fix for single-node dev/test:
    curl -X PUT "localhost:9200/logs-2024.01/_settings" \
      -d '{"number_of_replicas": 0}'

Step 4 — Trigger reallocation after fixing root cause:
  curl -X POST "localhost:9200/_cluster/reroute?retry_failed=true"
  watch 'curl -s localhost:9200/_cluster/health | python3 -m json.tool'
```

💡 **Interview tip:** `_cluster/allocation/explain` is the most important Elasticsearch diagnostic command — it tells you in plain English exactly why a shard cannot be allocated. The disk watermark issue is the most common production cause: default high watermark is 85%, and when any node reaches 85% disk, no new shards are allocated to it. If all nodes are above 85%, the cluster goes RED. The `allocate_stale_primary` with `accept_data_loss:true` is truly a last resort — it can recover a RED cluster but may lose writes from after the last successful flush.

---

### Q253 — Jenkins | Conceptual | Advanced

> Explain **`stash`/`unstash`** vs **`archiveArtifacts`** vs **fingerprinting**. Jenkinsfile demonstrating all three.

📁 **Reference:** `nawab312/CI_CD` → `Jenkins`

```
stash/unstash — temporary inter-stage transfer:
  Store files → retrieve in another stage in same pipeline run
  Stored on Jenkins master temporarily
  Automatically deleted at end of pipeline run
  Use for: build artifacts needed by downstream stages
  NOT for: long-term storage, cross-pipeline sharing

archiveArtifacts — persistent long-term storage:
  Store files permanently in Jenkins (until retention policy)
  Accessible from Jenkins UI for every build (users can download)
  Use for: release artifacts, test reports users need to access
  NOT for: inter-stage data transfer (use stash — faster)

fingerprinting — tracks which build produced which artifact:
  MD5 hash stored with build metadata
  Jenkins answers: "which build produced this JAR?"
  Cross-pipeline: "which downstream build used this artifact?"
  Use for: traceability, compliance, debugging what's in production

pipeline {
  agent any
  stages {
    stage('Build') {
      agent { label 'build-agent' }
      steps {
        sh 'mvn package -DskipTests'

        // Stash: transfer compiled JAR to test agent
        stash name: 'app-jar',
              includes: 'target/*.jar'

        // Archive: keep for user download (with fingerprint)
        archiveArtifacts artifacts: 'target/*.jar',
                         fingerprint: true     // fingerprint while archiving
      }
    }

    stage('Test') {
      agent { label 'test-agent' }   // different agent!
      steps {
        // Unstash: retrieve JAR from build agent (via Jenkins master)
        unstash 'app-jar'
        sh 'java -jar target/app.jar --run-tests'
        publishTestResults testResultsPattern: 'test-results/*.xml'
      }
    }

    stage('Integration Test') {
      agent { label 'integration-agent' }
      steps {
        unstash 'app-jar'    // same stash, reused in another stage
        sh 'run-integration-tests.sh'
        archiveArtifacts 'integration-results/**', fingerprint: true
      }
    }

    stage('Deploy') {
      steps {
        unstash 'app-jar'
        // Fingerprint the deployed artifact:
        // Creates cross-pipeline tracking for traceability
        fingerprint 'target/app.jar'
        sh 'deploy-to-staging.sh target/app.jar'
      }
    }
  }
  options {
    buildDiscarder(logRotator(
      numToKeepStr: '20',
      artifactNumToKeepStr: '5'    // keep artifacts for last 5 builds only
    ))
  }
}

When to use each:
  Building in agent A, testing in agent B:
    stash in build → unstash in test (different workspace, same pipeline)
  Build artifact users need to download:
    archiveArtifacts in the publish stage
  Trace which JAR is deployed in production:
    fingerprint in both build and deploy pipelines
```

💡 **Interview tip:** The key practical distinction: `stash` is fast temporary transfer between agents in the same pipeline run (data stored in temp on the master), while `archiveArtifacts` stores files permanently accessible via the UI. Never use `archiveArtifacts` just for inter-stage transfer — it's slower and fills Jenkins disk. Fingerprinting is most valuable in complex pipelines with multiple downstream jobs: if a deployment fails, you can trace through fingerprint records to find exactly which build produced the artifact and then find the commit that caused the issue.

---

### Q254 — Linux / Bash | Conceptual | Advanced

> Explain **`epoll`** vs **`select`** vs **`poll`** — why epoll scales better. C10K problem. How Nginx and Node.js use event loops.

📁 **Reference:** `nawab312/DSA` → `Linux` — I/O multiplexing, epoll

```
The problem: handle thousands of concurrent connections
  Naive approach: 1 thread per connection
  10,000 connections → 10,000 threads → 10-80GB RAM just for threads
  Context switching between 10K threads: massive CPU overhead
  C10K problem (1999): "why can't a server handle 10,000 clients?"

select (oldest):
  Algorithm:
    Build bit-array of ALL FDs → kernel scans ALL FDs for readiness
    select() returns → YOUR CODE scans ALL FDs to find which are ready
  Problems:
    Max FDs: 1024 (FD_SETSIZE hardcoded)
    O(n) scan: kernel AND you scan all n FDs every iteration
    Pass full FD set to kernel every call (copy overhead)

poll (improved select):
  No 1024 FD limit — uses array of pollfd structs
  STILL O(n): kernel scans ALL FDs on every call
  Still rebuild/pass full array every call

epoll (Linux 2.5.44+):
  Three system calls:
    epoll_create1() → create epoll file descriptor (once)
    epoll_ctl()     → register ONE fd at a time (add/modify/delete)
    epoll_wait()    → block until events, returns ONLY ready FDs

  Algorithm:
    Register each connection once via epoll_ctl
    epoll_wait() → kernel returns ONLY FDs with events
    Handle only returned FDs (no scanning all 10,000)

  Why O(1) per event:
    Kernel uses red-black tree for registered FDs (fast insert/delete)
    Kernel uses linked list of ready FDs → only ready returned
    10,000 connections, 5 with data → epoll returns only 5
    select/poll would scan all 10,000 to find those 5

  Modes:
    Level-triggered: returns FD as long as data available (default)
    Edge-triggered (EPOLLET): returns FD only once on state change
                               more efficient, must drain completely

How Nginx uses epoll:
  Master process spawns worker processes (one per CPU)
  Worker: ONE thread, event loop with epoll
    while (true):
      events = epoll_wait(...)    # block until events
      for event in events:
        if new_connection:  accept() + epoll_ctl(ADD, new_fd)
        if data_ready:      read() → process → write()

  10,000 connections → 1 thread → epoll handles all efficiently
  No context switching between threads

How Node.js uses event loop (libuv):
  Single-threaded JavaScript + libuv async I/O library
  libuv uses: epoll (Linux), kqueue (macOS), IOCP (Windows)

  const server = require('http').createServer((req, res) => {
    fs.readFile('data.json', (err, data) => {
      res.end(data)  // callback runs when disk I/O completes
    })
    // Thread is not blocked — returns to event loop immediately
    // handles other connections in the meantime
  })
```

💡 **Interview tip:** The key insight is algorithmic complexity: `select`/`poll` are O(n) per iteration (scan all connections even if only 1 has data), while `epoll` is effectively O(1) per event (only ready connections returned). At 10,000 connections, `select` does 10,000 operations per iteration even if only 1 connection has data — `epoll` does 1 operation. This is the fundamental reason Nginx handles 10,000+ concurrent connections with a single thread while Apache pre-fork (1 process per connection) collapses under the same load. Node.js's "non-blocking I/O" is just epoll under the hood via libuv.

---

### Q255 — AWS | Scenario-Based | Advanced

> API Gateway returning `429 Too Many Requests` to legitimate users. Explain all throttling limits. Diagnose which is being hit. Fix.

📁 **Reference:** `nawab312/AWS` — API Gateway, throttling, usage plans

```
API Gateway throttling layers (evaluated in order):

Layer 1 — Account-level (hard AWS limit):
  Default: 10,000 RPS steady state, 5,000 burst
  Applies to: ALL APIs in the account and region combined
  Check: CloudWatch → ApiGateway → Count metric at account level
  Fix: aws service-quotas request-service-quota-increase \
         --service-code apigateway --quota-code L-A4A3C9E4 --desired-value 20000

Layer 2 — Stage-level throttling:
  Per API, per stage (prod/staging/dev)
  Default: inherits account level (no additional limit)
  aws apigateway update-stage --rest-api-id abc123 --stage-name prod \
    --patch-operations \
      '[{"op":"replace","path":"/*/*/throttling/rateLimit","value":"1000"}]'

Layer 3 — Method-level throttling:
  Per HTTP method/resource — more granular
  POST /orders can have lower limit than GET /products

Layer 4 — Usage Plans (per API key):
  throttle: 100 req/sec per key
  quota:    10,000 requests per day per key
  One abusive API key throttled → other keys unaffected

Diagnosis — which limit is being hit:

  Step 1: Check CloudWatch 4XXError metric by Stage:
    aws cloudwatch get-metric-statistics \
      --namespace AWS/ApiGateway \
      --metric-name 4XXError \
      --dimensions Name=ApiName,Value=my-api Name=Stage,Value=prod \
      --period 60 --statistics Sum

  Step 2: Check if 429s from specific API key:
    Enable access logging: $context.identity.apiKey in log format
    → All 429s from same key → usage plan throttled

  Step 3: Check account-level limit:
    If ALL APIs in region throttled → account limit hit
    aws service-quotas get-service-quota \
      --service-code apigateway --quota-code L-A4A3C9E4

  Step 4: Check Lambda concurrency (often missed!):
    API Gateway itself not throttling → Lambda is
    CloudWatch: Lambda → Throttles metric
    Fix: aws lambda put-function-concurrency \
           --function-name my-api-handler --reserved-concurrent-executions 500

Fixes summary:
  Account limit: request quota increase via Service Quotas
  Stage limit: increase in API Gateway console or CLI
  Usage plan: update throttle settings per API key tier
  Lambda throttling: increase reserved concurrency
```

💡 **Interview tip:** The most commonly missed layer is Lambda concurrency throttling — API Gateway itself isn't rate-limiting, but the Lambda integration returns a throttle error, which API Gateway converts to a 429. Always check Lambda `Throttles` metric when debugging API Gateway 429s. Usage plans are the right answer for multi-tenant APIs — each API key gets its own rate limit so one abusive customer doesn't affect others. The diagnostic must be systematic: check each layer from account level down to method level before concluding which limit is being hit.

---

### Q256 — Kubernetes | Conceptual | Advanced

> Explain **CSI drivers** — what problem vs in-tree plugins. `CSIDriver`, `CSINode`, `VolumeAttachment`. Walk through EBS volume mounting from PVC to container.

📁 **Reference:** `nawab312/Kubernetes` → `04_STORAGE.md`

```
In-tree volume plugins (old):
  Volume drivers for EBS, GCE PD, Azure Disk compiled INTO Kubernetes
  Problems:
  - New features required Kubernetes release cycle (slow)
  - Storage bugs could crash kube-controller-manager
  - Not extensible — only supported types that made it into K8s core

CSI (Container Storage Interface):
  Standard interface between Kubernetes and external storage
  Storage vendor writes CSI driver (independent deployment)
  No more in-tree plugins needed

CSI Objects:
  CSIDriver — capabilities declaration:
    apiVersion: storage.k8s.io/v1
    kind: CSIDriver
    metadata:
      name: ebs.csi.aws.com
    spec:
      attachRequired: true      # volume requires attach/detach
      fsGroupPolicy: File

  CSINode — per-node driver information (auto-created):
    apiVersion: storage.k8s.io/v1
    kind: CSINode
    metadata:
      name: ip-10-0-1-5.ec2.internal
    spec:
      drivers:
      - name: ebs.csi.aws.com
        nodeID: i-0abc123456    # EC2 instance ID (for EBS attach)
        topologyKeys: ["topology.ebs.csi.aws.com/zone"]

  VolumeAttachment — tracks EBS attachment to node:
    Created when pod is scheduled with a bound PVC
    apiVersion: storage.k8s.io/v1
    kind: VolumeAttachment
    spec:
      attacher: ebs.csi.aws.com
      source: {persistentVolumeName: pvc-abc123}
      nodeName: ip-10-0-1-5.ec2.internal
    status:
      attached: true
      attachmentMetadata:
        devicePath: /dev/xvdb

EBS volume lifecycle (PVC → container):
  1. PVC created (StorageClass: ebs-gp3, 20Gi, RWO)

  2. Dynamic provisioning:
     CSI external-provisioner watches for unbound PVCs
     Calls CreateVolume gRPC on EBS CSI driver
     AWS creates EBS volume: vol-0abc123456
     CSI driver creates PV object bound to PVC

  3. Pod scheduled to node:
     Scheduler picks node in us-east-1a (same AZ as EBS)
     [WaitForFirstConsumer StorageClass ensures AZ alignment]

  4. Volume attachment:
     Attach/detach controller creates VolumeAttachment
     external-attacher calls ControllerPublishVolume gRPC
     AWS attaches EBS → /dev/xvdb appears on node

  5. Mount on node:
     kubelet calls NodeStageVolume → CSI formats /dev/xvdb (mkfs.ext4)
     kubelet calls NodePublishVolume → CSI bind-mounts to pod path

  6. Container starts:
     Container sees: /data → EBS volume
     Reading/writing /data = reading/writing EBS volume

Debugging VolumeAttachment:
  kubectl describe volumeattachment
  # If attached: false for long time → check AWS CloudWatch for EBS attach errors
```

💡 **Interview tip:** `WaitForFirstConsumer` binding mode in StorageClass is enabled by `topologyKeys` in CSINode — the scheduler knows which AZ the node is in, and the provisioner creates the EBS volume in the same AZ. The VolumeAttachment object is a valuable debugging tool when pods are stuck waiting for volumes — `kubectl describe volumeattachment` shows whether the EBS attach API call succeeded and what device path was assigned. If it shows `attached: false` for a long time, check AWS CloudWatch for EBS attach API errors.

---

### Q257 — GitHub Actions | Scenario-Based | Advanced

> Implement **infrastructure drift detection** workflow — scheduled terraform plan, create GitHub Issue with plan output, label by severity (P1/P2/P3), comment on existing issue, Slack for P1.

📁 **Reference:** `nawab312/CI_CD` → `GithubActions`, `nawab312/Terraform`

```yaml
name: Infrastructure Drift Detection

on:
  schedule:
  - cron: '0 */6 * * *'    # every 6 hours
  workflow_dispatch:         # manual trigger

permissions:
  contents: read
  issues: write

jobs:
  detect-drift:
    runs-on: ubuntu-latest
    env:
      TF_DIR: terraform/production

    steps:
    - uses: actions/checkout@v4
    - uses: hashicorp/setup-terraform@v3

    - name: Configure AWS credentials (read-only role)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/terraform-readonly
        aws-region: us-east-1

    - name: Terraform init and plan
      id: plan
      run: |
        terraform init
        terraform plan -detailed-exitcode -out=drift.tfplan 2>&1 | tee plan.txt
        echo "exit_code=${PIPESTATUS[0]}" >> $GITHUB_OUTPUT
      working-directory: ${{ env.TF_DIR }}
      continue-on-error: true  # don't fail job if drift detected

    - name: Classify severity
      id: severity
      if: steps.plan.outputs.exit_code == '2'   # 2 = changes detected
      run: |
        DESTROYS=$(grep -c "will be destroyed" ${{ env.TF_DIR }}/plan.txt || echo 0)
        MODIFIES=$(grep -c "will be updated in-place" ${{ env.TF_DIR }}/plan.txt || echo 0)
        ADDS=$(grep -c "will be created" ${{ env.TF_DIR }}/plan.txt || echo 0)
        if [ "$DESTROYS" -gt 0 ]; then
          echo "severity=P1" >> $GITHUB_OUTPUT
          echo "label=severity:critical" >> $GITHUB_OUTPUT
        elif [ "$MODIFIES" -gt 0 ]; then
          echo "severity=P2" >> $GITHUB_OUTPUT
          echo "label=severity:high" >> $GITHUB_OUTPUT
        else
          echo "severity=P3" >> $GITHUB_OUTPUT
          echo "label=severity:low" >> $GITHUB_OUTPUT
        fi
        echo "destroys=$DESTROYS" >> $GITHUB_OUTPUT
        echo "modifies=$MODIFIES" >> $GITHUB_OUTPUT
        echo "adds=$ADDS" >> $GITHUB_OUTPUT
        head -100 ${{ env.TF_DIR }}/plan.txt > plan-summary.txt

    - name: Check for existing open drift issue
      id: existing
      if: steps.plan.outputs.exit_code == '2'
      uses: actions/github-script@v7
      with:
        result-encoding: string
        script: |
          const issues = await github.rest.issues.listForRepo({
            owner: context.repo.owner, repo: context.repo.repo,
            labels: 'terraform-drift', state: 'open'
          });
          return issues.data.length > 0 ? String(issues.data[0].number) : '';

    - name: Comment on existing issue
      if: steps.plan.outputs.exit_code == '2' && steps.existing.outputs.result != ''
      uses: actions/github-script@v7
      with:
        script: |
          const plan = require('fs').readFileSync('plan-summary.txt', 'utf8');
          await github.rest.issues.createComment({
            owner: context.repo.owner, repo: context.repo.repo,
            issue_number: parseInt('${{ steps.existing.outputs.result }}'),
            body: `## Drift still present — ${new Date().toISOString()}\n\n` +
                  `Destroys: ${{ steps.severity.outputs.destroys }} | ` +
                  `Modifies: ${{ steps.severity.outputs.modifies }} | ` +
                  `Adds: ${{ steps.severity.outputs.adds }}\n\n` +
                  `\`\`\`\n${plan}\n\`\`\``
          });

    - name: Create new drift issue
      if: steps.plan.outputs.exit_code == '2' && steps.existing.outputs.result == ''
      uses: actions/github-script@v7
      with:
        script: |
          const plan = require('fs').readFileSync('plan-summary.txt', 'utf8');
          await github.rest.issues.create({
            owner: context.repo.owner, repo: context.repo.repo,
            title: `[${{ steps.severity.outputs.severity }}] Infrastructure Drift — ${new Date().toDateString()}`,
            body: `**Severity**: ${{ steps.severity.outputs.severity }}\n\n` +
                  `\`\`\`\n${plan}\n\`\`\``,
            labels: ['terraform-drift', '${{ steps.severity.outputs.label }}']
          });

    - name: Slack alert for P1 (destructive drift only)
      if: steps.severity.outputs.severity == 'P1'
      run: |
        curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
          -H 'Content-Type: application/json' \
          -d '{"text":":rotating_light: *P1 Infrastructure Drift Detected* — destructive changes found in production Terraform. Investigate immediately."}'
```

💡 **Interview tip:** The `detailed-exitcode` flag on `terraform plan` is the key — exit code 0 means no changes, exit code 2 means changes detected, exit code 1 means error. This lets the workflow distinguish "plan succeeded, no drift" from "plan succeeded, drift found" without parsing plan output. The "comment vs create" logic prevents issue spam — only one drift issue exists at a time, and new drift detections append comments. P1 only for destructive changes keeps Slack signal-to-noise high — adds and modifications don't warrant waking someone up.

---

### Q258 — Terraform | Scenario-Based | Advanced

> **Terraform Kubernetes provider vs Helm provider vs ArgoCD GitOps** — trade-offs and recommendation.

📁 **Reference:** `nawab312/Terraform`, `nawab312/CI_CD` → `ArgoCD`

```
Option A — Terraform Kubernetes Provider:
  resource "kubernetes_deployment" "app" {
    metadata { name = "my-app" }
    spec { ... }
  }

  Pros:
  + Single tool for AWS + K8s resources
  + Dependencies between AWS and K8s in same state
    (create RDS first, then K8s ConfigMap with RDS endpoint)
  + Familiar Terraform plan/apply workflow

  Cons:
  - K8s objects in Terraform state = slow plan (API calls per resource)
  - No rollback: Terraform doesn't understand K8s rolling updates
  - No health check: doesn't wait for pods to be Ready
  - Complex HCL for K8s YAML (verbose, awkward nesting)
  - Mixing concerns: IaC for infra + app deployment in same state

Option B — Terraform Helm Provider:
  resource "helm_release" "my-app" {
    name  = "my-app"
    chart = "./charts/my-app"
    set { name = "image.tag", value = "v1.2.3" }
  }

  Pros:
  + Uses Helm charts (reusable, versioned)
  + Good for infrastructure Helm charts (Prometheus, cert-manager)
  + Dependencies: install cert-manager before ingress (depends_on)

  Cons:
  - Still Terraform state for K8s resources
  - Not GitOps: Helm releases managed by Terraform, not Git-watching controller
  - No automatic drift detection for in-cluster changes

Option C — ArgoCD GitOps (separate concerns):
  Terraform: AWS infrastructure (VPC, EKS, RDS, IAM)
  ArgoCD:    Kubernetes resources (Deployments, Services, HPA)

  Pros:
  + True GitOps: Git is source of truth for K8s state
  + ArgoCD continuously reconciles: drift auto-detected and corrected
  + Native K8s tooling: kubectl, Helm, Kustomize all work naturally
  + Developer self-service: push to Git → auto-deployed
  + Rollback: one ArgoCD click to revert to previous Git commit
  + Fast feedback: ArgoCD shows sync status per app

  Cons:
  - Two tools to maintain
  - Bootstrapping: ArgoCD must be installed first
    (use Terraform Helm provider for ArgoCD bootstrap only)

Recommendation:

  Use Terraform for:
    AWS infrastructure: VPC, EKS, RDS, S3, IAM, ECR
    Cluster setup: ArgoCD bootstrap, storage classes, namespace creation

  Use ArgoCD for:
    Application deployments: Deployments, Services, HPA, Ingress
    Everything app-level — each team's services in Git repos

  Use Terraform Helm Provider only for:
    Infrastructure Helm charts (Prometheus, cert-manager) that:
    - Are deployed once, rarely changed
    - Have dependencies on Terraform-managed AWS resources
    NOT for: application Helm charts (those belong in ArgoCD)

  Avoid Terraform Kubernetes Provider for:
    Application Deployments — slow plan, no K8s lifecycle awareness
```

💡 **Interview tip:** The recommendation that impresses interviewers: use Terraform for infrastructure (where state tracking and plan/apply makes sense) and ArgoCD for application deployments (where GitOps continuous reconciliation makes sense). Mixing them creates problems — Terraform plans become slow when including K8s resources, and Terraform doesn't understand K8s-specific concepts like rolling updates or readiness. The bootstrapping exception is worth mentioning: use `helm_release` in Terraform to install ArgoCD itself, then use ArgoCD to manage everything else — this is the "Terraform installs ArgoCD, ArgoCD manages applications" pattern.

---

### Q259 — Prometheus | Scenario-Based | Advanced

> Design **multi-cluster monitoring** for 5 clusters (dev/staging/prod-us/prod-eu/prod-ap) with single Grafana. Choose between federation, Thanos, or Grafana Agent remote_write.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Prometheus`, `Grafana`

```
Three approaches:

Option A — Prometheus federation:
  Global Prometheus scrapes /federate from each cluster Prometheus
  Only pulls aggregated/reduced metrics to global
  Limitations:
  - Global Prometheus single point of failure
  - Bounded by one Prometheus's disk
  - Only federated (reduced) metrics — lose granularity
  Works for: 2-3 clusters, mostly aggregated dashboards only

Option B — Thanos (recommended):
  Sidecar on each Prometheus → uploads blocks to S3 every 2 hours
  Thanos Querier fans out queries to all sidecars + Store Gateways

  Architecture:
  prod-us cluster:    Prometheus + Thanos Sidecar → S3-us
  prod-eu cluster:    Prometheus + Thanos Sidecar → S3-eu
  prod-ap cluster:    Prometheus + Thanos Sidecar → S3-ap
  staging cluster:    Prometheus + Thanos Sidecar → S3-staging
  dev cluster:        Prometheus + Thanos Sidecar → S3-dev
                                │
                     Thanos Querier (central)
                     ├── Sidecar gRPC: each cluster
                     ├── Store Gateway (reads S3 historical data)
                     └── Ruler (global alerting rules)
                                │
                          Grafana (one datasource: Thanos Querier)

  Querier setup:
  args:
  - query
  - --store=prometheus-thanos-sidecar.prod-us.svc:10901
  - --store=prometheus-thanos-sidecar.prod-eu.svc:10901
  - --store=dnssrv+_grpc._tcp.thanos-store.thanos.svc
  - --query.replica-label=prometheus_replica  # deduplication for HA pairs

Option C — Grafana Agent remote_write:
  Grafana Agent on each cluster → remote_write to central Mimir/Cortex
  Benefits: push-based (works across private networks), simpler than Thanos
  Limitations: requires operating Mimir/Cortex (complex)

Why Thanos over federation:
  True multi-cluster unified PromQL queries
  Unlimited S3 retention
  Production-grade HA + deduplication

Why Thanos over Agent + Mimir:
  Don't need to operate Mimir (complex distributed system)
  Each cluster keeps Prometheus for local queries (no central dependency)
  Simpler architecture for this scale (5 clusters)

Helm install:
  helm install prometheus bitnami/kube-prometheus \
    --set prometheus.thanos.create=true \
    --set prometheus.thanos.objectStorageConfig.secretName=thanos-objstore-secret
```

💡 **Interview tip:** Thanos is the standard production answer for multi-cluster Prometheus. The key configuration detail is `--query.replica-label=prometheus_replica` on the Querier — this enables deduplication when you run HA Prometheus pairs (two instances scraping same targets). Without it, you get double-counted metrics. The `dnssrv+` prefix for Store Gateway discovery uses DNS SRV records to automatically find all Store Gateway instances — more resilient than listing individual endpoints. For the exam/interview, know the component roles: Sidecar (uploads to S3 + exposes gRPC), Store Gateway (reads from S3), Querier (fans out queries), Compactor (merges/downsamples blocks in S3).

---

### Q260 — AWS | Conceptual | Advanced

> Explain **Fargate `awsvpc` network mode** vs `bridge` vs `host`. How tasks get IPs. Task-to-task and internet communication. Networking limitations vs EC2.

📁 **Reference:** `nawab312/AWS` — Fargate, awsvpc network mode

```
Network modes:
  bridge mode (legacy, EC2 only):
    Container shares host EC2 network namespace
    Docker bridge: 172.17.0.0/16
    Port mapping required: host_port:container_port
    Security group at EC2 instance level (all containers share)
    Problems: port conflicts, coarse-grained security

  host mode (EC2 only):
    Container uses host EC2 network directly (no isolation)
    Container port = host port (no mapping)
    Use for: maximum network performance
    Problems: no port isolation between containers

  awsvpc mode (required for Fargate, optional for EC2):
    Each task gets its own ENI
    Each task gets its own private VPC IP
    Security groups at TASK level (not instance level)
    Each task behaves like a mini EC2 from networking perspective
    Required for: Fargate (no other choice)

How Fargate tasks get IP addresses:
  1. Fargate task launched in VPC subnet
  2. AWS creates ENI in the subnet's IP range
  3. ENI assigned private IP from subnet CIDR (e.g., 10.0.1.45)
  4. Security groups from task definition attached to ENI
  5. Task's network = this ENI (no underlying instance IP)

Task-to-task communication:
  Same VPC: direct IP via VPC routing
    task-A (10.0.1.45) → task-B (10.0.2.33): direct, no NAT
  Via Service Discovery: AWS Cloud Map / ECS Service Connect
    payment.service.local → DNS → 10.0.2.33
  Via ALB: load balancer distributes to all task ENIs

Internet access:
  Public subnet + assign_public_ip: ENABLED → direct internet via IGW
  Private subnet (recommended):
    No public IP → needs NAT Gateway in public subnet
    Route: 0.0.0.0/0 → NAT Gateway

  Pulling ECR images from private subnet:
    Option A: NAT Gateway (costs $0.045/hr + data)
    Option B: VPC Interface Endpoints for ECR (no internet needed):
      aws ec2 create-vpc-endpoint \
        --service-name com.amazonaws.us-east-1.ecr.api ...

Networking limitations vs EC2:
  1. One ENI per Fargate task:
     EC2: multiple ENIs, more IPs per instance
     Fargate: exactly one ENI per task (simpler but less flexible)

  2. Higher IP consumption:
     100 Fargate tasks = 100 ENIs = 100 VPC IPs consumed
     Small subnets (/26 = 64 IPs) can be exhausted quickly

  3. No host networking:
     host mode not available for Fargate

  4. No placement control:
     Cannot pin to specific instance types or bare-metal
```

💡 **Interview tip:** The key point about Fargate networking is that `awsvpc` means each Fargate task behaves like a tiny EC2 instance from the VPC's perspective — its own ENI, its own private IP, its own security groups. This is why Fargate tasks can talk directly to RDS by IP without any proxy or port mapping. The IP exhaustion issue is real in production — 500 Fargate tasks in a /26 subnet (64 IPs) simply won't fit. Always plan subnet sizing for Fargate: a production environment running 100+ tasks needs a /22 or larger subnet per AZ.

---

### Q261 — Kubernetes | Scenario-Based | Advanced

> Implement **PodSecurityAdmission** — kube-system=privileged, monitoring=baseline, production=restricted with deny on violation.

📁 **Reference:** `nawab312/Kubernetes` → `07_SECURITY.md`

```
Three profiles:
  privileged: no restrictions — for system DaemonSets (CNI, CSI)
  baseline:   blocks known privilege escalations — for most workloads
  restricted: hardened — requires runAsNonRoot, seccomp, drop all capabilities

Three modes:
  enforce: violations REJECTED (pod not created)
  warn:    violations ALLOWED but client sees warning
  audit:   violations ALLOWED but logged in audit log

Namespace configuration:

  # kube-system: privileged (CNI, CSI drivers need root, hostNetwork):
  kubectl label namespace kube-system \
    pod-security.kubernetes.io/enforce=privileged \
    pod-security.kubernetes.io/enforce-version=v1.28

  # monitoring: baseline (node-exporter needs hostPath, hostPID):
  kubectl label namespace monitoring \
    pod-security.kubernetes.io/enforce=baseline \
    pod-security.kubernetes.io/enforce-version=v1.28 \
    pod-security.kubernetes.io/warn=restricted \
    pod-security.kubernetes.io/warn-version=v1.28
  # enforce=baseline: pods violating baseline REJECTED
  # warn=restricted: developers see what restricted would need

  # production: restricted (full hardening, violations denied + logged):
  kubectl label namespace production \
    pod-security.kubernetes.io/enforce=restricted \
    pod-security.kubernetes.io/enforce-version=v1.28 \
    pod-security.kubernetes.io/audit=restricted \
    pod-security.kubernetes.io/audit-version=v1.28

Compliant pod for production (restricted profile):
  spec:
    securityContext:
      runAsNonRoot: true                # required
      runAsUser: 1000
      seccompProfile:
        type: RuntimeDefault            # required
    containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false # required
        capabilities:
          drop: ["ALL"]                 # required
        readOnlyRootFilesystem: true    # best practice

What restricted REJECTS:
  ✗ running as root (runAsUser: 0 or no runAsNonRoot: true)
  ✗ any capability.add
  ✗ allowPrivilegeEscalation: true
  ✗ no seccompProfile defined
  ✗ hostNetwork: true / hostPID: true / privileged: true

Migration path (don't jump straight to enforce):
  # Phase 1: audit (discover violations, nothing breaks):
  kubectl label namespace production pod-security.kubernetes.io/audit=restricted

  # Check violations:
  kubectl get events -n production | grep PodSecurity

  # Phase 2: warn (developers see warnings, nothing breaks):
  kubectl label namespace production pod-security.kubernetes.io/warn=restricted --overwrite

  # Phase 3: enforce (after developers fixed manifests):
  kubectl label namespace production pod-security.kubernetes.io/enforce=restricted --overwrite

Important: enforce does NOT evict running pods
  Existing violations: KEEP RUNNING (PSA only checks at admission time)
  To enforce on existing pods: kubectl rollout restart deployment -n production
```

💡 **Interview tip:** The phased migration (audit → warn → enforce) is the production best practice — jumping straight to `enforce=restricted` on a namespace with existing workloads will immediately break pods that don't meet restricted requirements. The audit and warn phases let you discover violations without impact. A common gotcha: `enforce=restricted` blocks new pods but does NOT evict existing running pods that violate the policy — you must restart them explicitly. The `version` label (`enforce-version: v1.28`) pins to a specific version of the policy, preventing automatic policy changes from breaking workloads when Kubernetes is upgraded.

---

### Q262 — ArgoCD | Conceptual | Advanced

> Explain **ArgoCD handling Helm** vs raw manifests vs Kustomize. What ArgoCD does NOT support in Helm. How to pass Helm values per environment.

📁 **Reference:** `nawab312/CI_CD` → `ArgoCD` — Helm integration

```
How ArgoCD detects tool type:
  Chart.yaml exists → Helm
  kustomization.yaml exists → Kustomize
  Otherwise → raw manifests (kubectl apply)

ArgoCD + raw manifests: kubectl apply -f <path>
ArgoCD + Kustomize: kustomize build <path> | kubectl apply -f -
  Base + overlays = environment customization

ArgoCD + Helm — what ArgoCD does:
  1. Renders templates: helm template (NOT helm install)
  2. Applies rendered manifests with kubectl
  3. Tracks resources in ArgoCD's own state
  4. Helm release history (helm ls, helm rollback) → NOT maintained

What ArgoCD does NOT support in Helm:
  ❌ helm rollback — use ArgoCD app rollback instead
  ❌ helm history — ArgoCD has its own sync history
  ❌ helm test — ArgoCD doesn't run helm test
  ❌ helm secrets plugin — use ArgoCD Vault Plugin instead
  ❌ Direct helm CLI with ArgoCD-managed releases
     (helm commands may conflict with ArgoCD state)
  ❌ Helm release state in cluster (no tiller/helm release CRD)

Passing Helm values per environment:

  Method 1 — values files in Application spec:
    spec:
      source:
        repoURL: https://github.com/company/helm-charts.git
        chart: payment-api
        targetRevision: v1.2.3
        helm:
          valueFiles:
          - values.yaml              # base values
          - values-production.yaml   # environment overrides

  Method 2 — inline values:
    helm:
      values: |
        replicaCount: 5
        image:
          tag: v2.1.0

  Method 3 — ApplicationSet with generator substitution:
    generators:
    - list:
        elements:
        - env: staging
          replicas: "2"
        - env: production
          replicas: "5"
    template:
      spec:
        source:
          helm:
            parameters:
            - name: replicaCount
              value: "{{replicas}}"

  Best practice: separate values files per environment in Git:
    helm/
    ├── Chart.yaml
    ├── values.yaml           # base defaults
    ├── values-staging.yaml   # staging overrides
    └── values-prod.yaml      # prod overrides
```

💡 **Interview tip:** The most important point: ArgoCD uses `helm template` (rendering only), not `helm install` — there is NO Helm release tracked in the cluster. Running `helm ls` shows nothing, and `helm rollback` won't work. ArgoCD manages lifecycle through its own sync mechanism. This surprises teams migrating from `helm install` to ArgoCD. The ArgoCD Application rollback (revert to previous Git commit) is the equivalent of `helm rollback` — it works because ArgoCD tracks every sync as a revision in its own history.

---

### Q263 — Linux / Bash | Scenario-Based | Advanced

> Write a **GitOps-style config drift detector** — read from Git, check packages/services/files/sysctl, report drift, `--check`/`--fix` modes, JSON report, Slack alert.

📁 **Reference:** `nawab312/DSA` → `Linux`

```bash
#!/bin/bash
set -euo pipefail

MODE="${1:---check}"
GIT_REPO="${GIT_REPO:-https://github.com/company/server-configs.git}"
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"
REPORT_FILE="/tmp/drift-report-$(date +%Y%m%d-%H%M%S).json"
WORK_DIR=$(mktemp -d)
HOSTNAME=$(hostname -s)
DRIFT_COUNT=0

cleanup() { rm -rf "$WORK_DIR"; }
trap cleanup EXIT

log() { echo "[$(date '+%H:%M:%S')] $*" >&2; }

# Clone config repo
log "Fetching expected configuration from Git..."
git clone --depth=1 "$GIT_REPO" "$WORK_DIR/config" 2>/dev/null
CONFIG_FILE="$WORK_DIR/config/hosts/${HOSTNAME}.json"
[ -f "$CONFIG_FILE" ] || CONFIG_FILE="$WORK_DIR/config/common.json"
[ -f "$CONFIG_FILE" ] || { log "No config found for $HOSTNAME"; exit 1; }

DRIFT_ITEMS="[]"

add_drift() {
  local category="$1" item="$2" expected="$3" actual="$4"
  DRIFT_ITEMS=$(echo "$DRIFT_ITEMS" | jq ". += [{
    \"category\": \"$category\",
    \"item\": \"$item\",
    \"expected\": \"$expected\",
    \"actual\": \"$actual\",
    \"hostname\": \"$HOSTNAME\",
    \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"
  }]")
  ((DRIFT_COUNT++))
  log "DRIFT [$category] $item: expected='$expected' actual='$actual'"
}

# Check 1: installed packages
log "Checking packages..."
while IFS= read -r pkg; do
  if ! dpkg -l "$pkg" 2>/dev/null | grep -q "^ii"; then
    add_drift "package" "$pkg" "installed" "missing"
    if [ "$MODE" = "--fix" ]; then
      apt-get install -y "$pkg" && log "FIXED: installed $pkg"
    fi
  fi
done < <(jq -r '.packages[]' "$CONFIG_FILE" 2>/dev/null)

# Check 2: running services
log "Checking services..."
while IFS= read -r svc; do
  actual=$(systemctl is-active "$svc" 2>/dev/null || echo "inactive")
  if [ "$actual" != "active" ]; then
    add_drift "service" "$svc" "active" "$actual"
    if [ "$MODE" = "--fix" ]; then
      systemctl start "$svc" && log "FIXED: started $svc"
    fi
  fi
done < <(jq -r '.services[]' "$CONFIG_FILE" 2>/dev/null)

# Check 3: file contents
log "Checking files..."
while IFS= read -r filepath; do
  expected_content=$(jq -r ".files[\"$filepath\"]" "$CONFIG_FILE" 2>/dev/null)
  if [ -f "$filepath" ]; then
    actual_content=$(cat "$filepath")
    if [ "$actual_content" != "$expected_content" ]; then
      add_drift "file" "$filepath" "matches expected" "content differs"
      if [ "$MODE" = "--fix" ]; then
        echo "$expected_content" > "$filepath" && log "FIXED: updated $filepath"
      fi
    fi
  else
    add_drift "file" "$filepath" "exists" "missing"
    if [ "$MODE" = "--fix" ]; then
      mkdir -p "$(dirname "$filepath")"
      echo "$expected_content" > "$filepath" && log "FIXED: created $filepath"
    fi
  fi
done < <(jq -r '.files | keys[]' "$CONFIG_FILE" 2>/dev/null)

# Check 4: sysctl values
log "Checking sysctl..."
while IFS="=" read -r key expected_val; do
  actual_val=$(sysctl -n "$key" 2>/dev/null || echo "not found")
  if [ "$actual_val" != "$expected_val" ]; then
    add_drift "sysctl" "$key" "$expected_val" "$actual_val"
    if [ "$MODE" = "--fix" ]; then
      sysctl -w "${key}=${expected_val}" && log "FIXED: sysctl $key=$expected_val"
    fi
  fi
done < <(jq -r '.sysctl | to_entries[] | "\(.key)=\(.value)"' "$CONFIG_FILE" 2>/dev/null)

# Write JSON report
echo "$DRIFT_ITEMS" | jq "{hostname: \"$HOSTNAME\", mode: \"$MODE\",
  drift_count: $DRIFT_COUNT, items: .}" > "$REPORT_FILE"
log "Report written to: $REPORT_FILE"

# Slack alert if drift found in --check mode
if [ "$DRIFT_COUNT" -gt 0 ] && [ "$MODE" = "--check" ] && [ -n "$SLACK_WEBHOOK" ]; then
  curl -s -X POST "$SLACK_WEBHOOK" \
    -H 'Content-Type: application/json' \
    -d "{\"text\": \":warning: *Config Drift Detected* on \`$HOSTNAME\`: $DRIFT_COUNT items drifted. Check the report for details.\"}"
fi

# Exit code
[ "$DRIFT_COUNT" -eq 0 ] && { log "No drift detected"; exit 0; } \
  || { log "$DRIFT_COUNT drift items found"; [ "$MODE" = "--fix" ] && exit 0 || exit 1; }
```

💡 **Interview tip:** The `trap cleanup EXIT` ensures the temp clone directory is removed even if the script fails mid-run. The JSON report format (using `jq`) makes the output machine-parseable — it can be ingested by a central monitoring system to track drift trends across a fleet. The separate `--check` vs `--fix` modes follow the standard configuration management pattern (audit first, fix separately) — running `--fix` automatically is safe for idempotent changes (sysctl, file creation) but should require human approval for package installation in production.

---

### Q264 — AWS | Scenario-Based | Advanced

> Design **automated SaaS tenant provisioning** — VPC/RDS/S3 per customer, under 5 minutes, IaC, deprovisioning, billing tracking.

📁 **Reference:** `nawab312/AWS`, `nawab312/Terraform` — multi-tenant, automated provisioning

```
Architecture overview:
  Customer signs up → API call → Step Functions workflow →
  Terraform via Lambda OR AWS Service Catalog →
  Isolated environment per tenant

Tenant isolation strategy:
  Option A: Per-tenant AWS account (strongest isolation):
    AWS Organizations: create account per customer via API
    Pros: complete blast radius isolation, separate billing
    Cons: 5-15 minute account creation, overhead for 500+ tenants

  Option B: Per-tenant VPC in shared account (practical):
    VPC + subnets per tenant (10.TENANT_ID.0.0/16)
    Separate RDS instance per tenant
    Separate S3 bucket per tenant
    IAM roles scoped to tenant resources
    Pros: under 5 minutes, manageable at 500 tenants
    Cons: account-level blast radius

Provisioning workflow (Step Functions + Terraform):

  State 1: ValidateSignup
    Lambda: validate customer data, generate tenant_id
    DynamoDB: create tenant record (status: provisioning)

  State 2: CreateInfrastructure
    Lambda: invoke terraform apply via CodeBuild
    OR: Service Catalog: provision approved Terraform product

    Terraform module inputs:
      tenant_id    = "cust-abc123"
      environment  = "prod"
      tier         = "standard"  # S3/RDS sizing

  State 3: WaitForProvisioning
    Poll: check CodeBuild status
    Timeout: 5 minutes → fail and cleanup

  State 4: UpdateDNS
    Lambda: create Route53 record: cust-abc123.app.company.com

  State 5: SendWelcomeEmail
    SES: send credentials and portal URL

  State 6: UpdateStatus
    DynamoDB: tenant record status = active

Terraform module (per-tenant resources):
  module "tenant" {
    source    = "./modules/tenant"
    tenant_id = var.tenant_id

    # VPC (10.{hash(tenant_id) % 250}.0.0/16 for unique CIDR):
    vpc_cidr = "10.${var.tenant_cidr_octet}.0.0/16"

    # RDS (size based on tier):
    db_instance_class = var.tier == "premium" ? "db.r5.large" : "db.t3.medium"

    # S3 bucket with tenant isolation:
    bucket_name = "company-${var.tenant_id}-data"
  }

  # Resource tagging for billing:
  default_tags = {
    TenantId    = var.tenant_id
    Environment = "production"
    BillingCode = var.tenant_id    # AWS Cost Explorer groups by TenantId tag
  }

Deprovisioning (customer churns):
  Step Functions workflow:
    State 1: BackupData
      S3 cross-region replication snapshot + RDS final snapshot
      Store 90 days for legal/compliance

    State 2: DeleteResources
      terraform destroy -target=module.tenant["cust-abc123"]
      Delete VPC, RDS, S3 (after backup confirmed)

    State 3: UpdateBilling
      Mark tenant inactive in DynamoDB
      Final invoice generation

Billing tracking:
  All tenant resources tagged: TenantId = "cust-abc123"
  AWS Cost Explorer: group by tag TenantId
  → Per-tenant cost visible for margin analysis

  API: aws ce get-cost-and-usage \
    --group-by Type=TAG,Key=TenantId \
    --filter ...
```

💡 **Interview tip:** The 5-minute SLA is the key constraint — AWS account creation takes 5-15 minutes (too slow), so per-tenant VPC in a shared account is the practical answer for rapid provisioning. Terraform modules parameterized by `tenant_id` make provisioning idempotent — running the same module twice for the same tenant is safe (Terraform's `terraform apply` is idempotent). The billing tracking via resource tags is the production answer — AWS Cost Explorer's tag-based grouping gives you per-tenant cost breakdown without building custom billing infrastructure.

---

### Q265 — Kubernetes | Conceptual | Advanced

> Explain **Kubernetes `Finalizers`** — how they work, what problems they solve. What happens on `kubectl delete`. Stuck finalizer removal.

📁 **Reference:** `nawab312/Kubernetes` → `10_CRDs_EXTENSIBILITY.md`

```
What finalizers are:
  Strings in metadata.finalizers list
  Prevent Kubernetes from deleting an object until ALL finalizers are removed
  Act like "prerequisites before deletion"

How they work:
  1. Object has finalizers: ["storage.k8s.io/pvc-protection"]

  2. kubectl delete pvc my-pvc
     → Kubernetes does NOT delete immediately
     → Sets DeletionTimestamp on the object
     → Object enters "Terminating" state

  3. Controller watching this finalizer sees DeletionTimestamp set
     → Controller does its pre-deletion work:
       (ensure volumes are safely unmounted, data archived, etc.)
     → Controller removes its finalizer from the list

  4. Once ALL finalizers removed → Kubernetes actually deletes the object

Real-world examples:
  kubernetes.io/pvc-protection:
    Prevents PVC deletion while pods are using it
    Mount the volume, delete PVC → PVC stays until no pods use it
    Prevents data loss from accidental PVC deletion mid-use

  foregroundDeletion:
    Added by kubectl delete --cascade=foreground
    Prevents parent deletion until all children are gone

  Custom finalizers (CRD operators):
    Database operator creates DB instance
    Adds finalizer: "db.example.com/cleanup"
    On deletion: operator backs up data, deletes cloud DB, removes finalizer
    Without finalizer: Kubernetes would delete CRD object but cloud DB remains (orphaned)

What happens on kubectl delete with finalizer:
  kubectl delete myrdb my-database
  → DeletionTimestamp set, status: Terminating
  → DB operator sees DeletionTimestamp
  → Takes final backup
  → Deletes AWS RDS instance
  → Removes finalizer: kubectl patch myrdb my-database --type=json \
      -p '[{"op":"remove","path":"/metadata/finalizers/0"}]'
  → Kubernetes deletes the CRD object

Stuck finalizer — when controller crashes:
  Controller responsible for removing finalizer is down
  Object stays in Terminating forever

  Safe removal process:
  1. Confirm the application is actually shut down (check logs, no active connections)
  2. Remove finalizer manually:
     kubectl patch <resource> <name> -n <ns> \
       -p '{"metadata":{"finalizers":[]}}' --type=merge

  3. OR more surgical:
     kubectl patch <resource> <name> -n <ns> \
       --type json \
       -p '[{"op":"remove","path":"/metadata/finalizers"}]'

  WARNING: Never remove finalizers without confirming the pre-deletion
  work is complete — doing so can cause:
  - Orphaned cloud resources (RDS instances without K8s objects)
  - Data loss (volumes unmounted before backup)
  - Billing surprises (cloud resources running but untracked)
```

💡 **Interview tip:** Finalizers are the Kubernetes mechanism that ensures "things are done right before deletion." The real-world importance: without the `kubernetes.io/pvc-protection` finalizer, a developer could accidentally delete a PVC while pods are still using it — causing immediate data loss. For custom operators, finalizers are what prevent a CRD object's deletion from leaving orphaned cloud resources. The stuck finalizer scenario is a common production incident — always verify the pre-deletion work was completed before manually removing a finalizer. The two-command approach (check application state, then remove finalizer) is the safe production procedure.

---

### Q266 — Grafana | Scenario-Based | Advanced

> Implement **Grafana annotations** for deployment correlation — add from CI/CD via API, show on all dashboards, query from Elasticsearch, annotation alerts for firing periods.

📁 **Reference:** `nawab312/Monitoring-and-Observability` → `Grafana`

```
What annotations are:
  Vertical markers on Grafana time series panels
  Show events at specific timestamps (deployments, incidents, alerts)
  Visual correlation: "latency spike happened right after this deploy"

Method 1 — Add deployment annotation from CI/CD via Grafana API:

  # In GitHub Actions / Jenkins deploy step:
  curl -s -X POST \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $GRAFANA_API_TOKEN" \
    "$GRAFANA_URL/api/annotations" \
    -d "{
      \"time\": $(date +%s%3N),
      \"tags\": [\"deployment\", \"$SERVICE_NAME\"],
      \"text\": \"Deployed $SERVICE_NAME:$IMAGE_TAG by $GITHUB_ACTOR\"
    }"

  # For ranged annotation (deployment start → end):
  curl -X POST "$GRAFANA_URL/api/annotations" \
    -d "{
      \"time\": $START_MS,
      \"timeEnd\": $END_MS,
      \"tags\": [\"deployment\", \"$SERVICE_NAME\"],
      \"text\": \"Deploy $SERVICE_NAME in progress\"
    }"

Method 2 — Show annotations on all dashboards:
  # Global annotation query (shows on every dashboard):
  # Grafana → Dashboards → Annotations → New Annotation Query
  Name: Deployments
  Data source: -- Grafana -- (built-in)
  Query: tags = "deployment"

  # OR per-dashboard annotation:
  # Dashboard settings → Annotations → Add annotation
  Filter by: tags → service=payment-api
  # Only shows relevant service's deployments on that service's dashboard

Method 3 — Query from Elasticsearch (if deployments logged there):
  # Annotation query using ES datasource:
  Name: Deployments from ES
  Data source: Elasticsearch
  Query: {"query": {"match": {"event.type": "deployment"}}}
  Time field: @timestamp
  Text field: message
  Tags field: service

Method 4 — Alert-state annotation (mark when alert was firing):
  # In Grafana alerting:
  # Alert rule → Annotations → automatically adds annotation on state change

  # Manual via API for historical:
  curl -X POST "$GRAFANA_URL/api/annotations" \
    -d "{
      \"time\": $ALERT_START_MS,
      \"timeEnd\": $ALERT_END_MS,
      \"tags\": [\"alert\", \"high-error-rate\"],
      \"text\": \"Alert: HighErrorRate was firing\"
    }"

Dashboard correlation example:
  Latency panel shows spike at 14:32
  Annotation marker appears at 14:31: "Deployed payment-api:v2.1.0"
  Correlation: new deployment caused latency spike
  Action: rollback deployment, investigate diff between v2.0.9 and v2.1.0

  Without annotations: would need to cross-reference deploy logs manually
  With annotations: visual correlation in seconds
```

💡 **Interview tip:** Grafana annotations are one of the most underutilised observability features — they transform dashboards from "show me metrics" to "show me metrics AND what changed." The CI/CD pipeline integration (posting a deployment annotation via the Grafana API at deploy time) takes 2 lines of curl and provides immediate visual correlation between deployments and metric changes. The `tags` field is key for filtering — use `["deployment", "service-name"]` so dashboards can filter to show only relevant service deployments. Global annotations (tag = "deployment") on all dashboards give every oncall engineer instant deployment context without any dashboard-specific configuration.

---

### Q267 — Terraform | Conceptual | Advanced

> Explain **`dynamic` blocks** — problem they solve, when to use vs not. Write AWS Security Group with dynamic ingress/egress rules and dynamic tags.

📁 **Reference:** `nawab312/Terraform` — dynamic blocks, meta-arguments

```
Problem dynamic blocks solve:
  Without dynamic: must hardcode repeated blocks
  resource "aws_security_group" "web" {
    ingress {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
    # What if you need 10 ingress rules? Repeat 10 times?
  }

Dynamic block: generate repeated blocks from a variable:

variable "ingress_rules" {
  default = [
    { port = 80,  cidr = "0.0.0.0/0",      description = "HTTP" },
    { port = 443, cidr = "0.0.0.0/0",      description = "HTTPS" },
    { port = 8080, cidr = "10.0.0.0/8",   description = "Internal API" },
  ]
}

variable "enable_egress" {
  default = true
}

variable "extra_tags" {
  default = {
    CostCenter  = "engineering"
    Owner       = "platform-team"
  }
}

resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Web security group"
  vpc_id      = var.vpc_id

  # Dynamic ingress rules from variable list:
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
      description = ingress.value.description
    }
  }

  # Conditional egress: only if enable_egress = true:
  dynamic "egress" {
    for_each = var.enable_egress ? [1] : []   # 1 item = include, empty = skip
    content {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
      description = "All outbound"
    }
  }

  # Dynamic tags from map variable:
  tags = merge(
    {
      Name = "web-sg"
      Environment = var.environment
    },
    var.extra_tags    # additional tags from variable
  )
}

When to use dynamic blocks:
  ✓ Variable number of repeated sub-blocks (ingress rules, lifecycle rules)
  ✓ Conditional blocks (include/exclude based on boolean)
  ✓ Generating multiple similar configurations from a list

When NOT to use dynamic blocks:
  ✗ Simple cases where hardcoding is clearer (2 static rules)
  ✗ When for_each on the resource itself is better
    (creating 5 security groups is better than one SG with dynamic rules)
  ✗ Deeply nested dynamic blocks (hard to read, consider refactoring)

Important: for_each = [] completely removes the block:
  dynamic "egress" {
    for_each = var.enable_egress ? [1] : []
    # [1] = one item = one egress block generated
    # []  = no items = no egress block at all
  }
  This is the idiomatic conditional block pattern in Terraform
```

💡 **Interview tip:** The conditional block pattern `for_each = var.condition ? [1] : []` is the most important dynamic block technique to know — it's the standard way to conditionally include an entire block based on a boolean variable. Without dynamic blocks, conditional blocks require duplicating entire resource configurations. The merge() function for tags is the standard pattern for combining required tags (Name, Environment) with optional team-specific tags from a variable — it's cleaner than a dynamic block for maps.

---

### Q268 — Git | Conceptual | Advanced

> Explain **`git worktree`** — what problem it solves, when to use. Explain **`git sparse-checkout`** for large monorepos in CI/CD.

📁 **Reference:** `nawab312/CI_CD` → `Git`

```
git worktree — work on multiple branches simultaneously:

  Problem without worktree:
    You are in the middle of a feature (modified files, can't commit)
    Urgent hotfix needed on main branch
    Options:
    1. stash → checkout main → fix → stash pop (context switching pain)
    2. clone repo again (takes minutes, wastes disk)
    3. git worktree (proper solution)

  How worktree works:
    One .git directory, multiple working directories
    Each working directory = a different branch
    No network clone needed — instant

  Commands:
    # Add a new worktree for the hotfix:
    git worktree add ../hotfix-branch main
    # Creates a new directory ../hotfix-branch checked out to main branch

    cd ../hotfix-branch
    # Make and commit the fix

    cd ../main-feature
    # Continue where you left off — no stash needed, no interrupted work

    # When done: remove the worktree:
    git worktree remove ../hotfix-branch

  Real-world DevOps use cases:
    Release manager: keep release/v2.1 and develop checked out simultaneously
                     compare behavior without switching
    Code review: check out reviewer's branch in separate directory
                 run both branches side-by-side

git sparse-checkout — only check out subset of files:

  Problem with large monorepo in CI/CD:
    15GB monorepo, 20 services
    CI builds service-A: needs services/service-a/ (100MB)
    But git clone downloads all 15GB first → wastes 10+ minutes

  How sparse-checkout works:
    Clone only downloads commits/tree objects (metadata)
    Working tree only materializes files matching patterns
    Result: small clone even from large repo

  Setup in CI/CD pipeline:
    # Clone without checking out any files (cone mode):
    git clone --filter=blob:none --no-checkout --depth=1 \
      https://github.com/company/monorepo.git
    cd monorepo

    # Enable sparse-checkout with cone mode (faster pattern matching):
    git sparse-checkout init --cone

    # Specify which directories to materialize:
    git sparse-checkout set services/service-a shared/libs

    # Checkout:
    git checkout main
    # Only services/service-a/ and shared/libs/ exist locally
    # All other directories not on disk

  GitHub Actions integration:
    - uses: actions/checkout@v4
      with:
        sparse-checkout: |
          services/service-a
          shared/libs
        sparse-checkout-cone-mode: true

  Performance impact on 15GB monorepo:
    Full clone:           10-15 minutes + 15GB disk
    Sparse-checkout:      30-60 seconds + 200MB disk
    Speedup:              10-20× faster CI startup

  Cone mode vs non-cone:
    Non-cone: arbitrary glob patterns (*.tf, src/**/*.py) — flexible but slower
    Cone mode: directory-based patterns — faster pattern matching, recommended
```

💡 **Interview tip:** `git worktree` is a lesser-known feature that solves the "I need to switch branches but I have uncommitted work" problem elegantly — it's much cleaner than stashing because your feature work remains exactly as you left it. For CI/CD, `git sparse-checkout` is a significant performance win for large monorepos — the checkout time directly impacts pipeline startup time and developer feedback loop speed. The `--filter=blob:none` (partial clone) combined with `--sparse` is the modern combination: only download the tree structure metadata on clone, then only materialize the specific files you need.

---

### Q269 — AWS + Kubernetes | Troubleshooting | Advanced

> EKS pods getting `OOMKilled` but `kubectl top pods` shows memory well below the limit. Explain container limits vs JVM heap vs off-heap vs kernel memory. How to size JVM properly.

📁 **Reference:** `nawab312/Kubernetes` → `06_SCHEDULING_RESOURCE_MANAGEMENT.md`, `nawab312/AWS`

```
Why OOMKill despite appearing within limits:

  kubectl top pods shows: 400Mi (out of 512Mi limit)
  Pod still gets OOMKilled

  kubectl top pods shows container heap usage — NOT total process memory
  The container's cgroup limit enforces TOTAL process memory including:
  - JVM heap (what kubectl top shows via JMX)
  - JVM non-heap (class loading, JIT compiled code, thread stacks)
  - Off-heap memory (direct ByteBuffers, native libraries)
  - Metaspace (class metadata)
  - Kernel memory (page cache for the process)

JVM memory anatomy:
  Total JVM memory = heap + non-heap + off-heap

  Heap:
    -Xmx=400m → JVM limits heap to 400MB
    This is what kubectl top / JMX metrics show
    Young gen (Eden + Survivor) + Old gen

  Non-heap (not counted in -Xmx):
    Metaspace:      class definitions (grows with loaded classes)
                    default: unlimited! can grow to 500MB+
    Code Cache:     JIT compiled native code (~256MB default)
    Thread stacks:  each thread uses 512KB-1MB of stack
                    100 threads = 50-100MB stack space
    Compressed class space: ~1GB max

  Off-heap (deliberately outside heap):
    Direct ByteBuffers: network I/O libraries use these extensively
    Java NIO: Netty, gRPC, Kafka clients allocate GB of off-heap
    Native libs: TensorFlow, OpenSSL, etc.

  Example: Spring Boot application with -Xmx=400m
    Heap:          400MB (limited by -Xmx)
    Metaspace:     200MB (all loaded classes)
    Code Cache:    256MB (JIT compiled code)
    Thread stacks: 50MB  (100 threads × 512KB)
    Netty off-heap: 200MB (direct buffers for HTTP connections)
    Total:         ~1.1GB — OOMKilled with 512Mi limit!
    kubectl top shows: 380MB (only heap)

Proper JVM memory sizing for Kubernetes:

  Formula:
    container_limit = Xmx + non-heap overhead + off-heap + buffer

  Typical overhead:
    -Xmx=400m → container_limit should be at least 700-900Mi

  JVM flags for Kubernetes:
    # Limit Metaspace (prevent unbounded growth):
    -XX:MaxMetaspaceSize=256m

    # Limit Code Cache:
    -XX:ReservedCodeCacheSize=256m

    # Enable container awareness (K8s 1.10+ with Java 8u191+, Java 11+):
    -XX:+UseContainerSupport

    # If UseContainerSupport enabled, JVM reads cgroup limits automatically:
    # Defaults: -Xmx = 25% of container limit (conservative)
    # Override:
    -XX:MaxRAMPercentage=75.0   # use 75% of container memory for heap

  Example for 1Gi container:
    resources:
      limits:
        memory: 1Gi
    JVM flags:
      -XX:+UseContainerSupport
      -XX:MaxRAMPercentage=60.0    # 60% of 1Gi = 614Mi for heap
      -XX:MaxMetaspaceSize=256m
      -XX:ReservedCodeCacheSize=128m
    # Remaining ~150Mi for thread stacks, off-heap, kernel

Debugging actual memory breakdown:
  kubectl exec pod-xxx -- jcmd 1 VM.native_memory summary
  # Shows: heap, metaspace, code, threads, GC, class breakdown
  # Most accurate total memory accounting

  kubectl exec pod-xxx -- cat /sys/fs/cgroup/memory/memory.usage_in_bytes
  # Shows actual cgroup total memory (what container limit enforces)
  # This is what triggers OOMKill when it reaches the limit
```

💡 **Interview tip:** `kubectl top pods` is misleading for JVM applications — it shows JVM heap usage via metrics, but the cgroup enforces total process memory including non-heap, Metaspace, JIT code cache, thread stacks, and off-heap. A rule of thumb: for Spring Boot with `-Xmx=400m`, the container limit should be at least 700-800Mi. The `-XX:+UseContainerSupport` flag (default ON in Java 11+) makes the JVM container-aware — it reads the cgroup memory limit and sizes the heap accordingly. Use `-XX:MaxRAMPercentage=60` rather than explicit `-Xmx` in containers — it automatically adjusts when you resize the container limit.

---

### Q270 — All Topics | System Design | Advanced

> Design **PCI DSS compliant infrastructure** for payments on Kubernetes/AWS — network segmentation, encryption, access control, vulnerability management, incident response, KMS, tamper-proof logs.

📁 **Reference:** All repositories — PCI DSS compliance, security architecture

```
PCI DSS key requirements:
  Protect cardholder data (CHD) — card numbers, CVV, expiry
  Cardholder Data Environment (CDE): systems that touch CHD
  Goal: isolate CDE, encrypt CHD, audit all access

Network Segmentation (Requirement 1):

  AWS account structure:
    Production account (CDE) — isolated from all other accounts
    Dev/Staging account — NEVER touches real card data

  VPC segmentation:
    CDE VPC: 10.1.0.0/16
      CDE subnet: 10.1.1.0/24 (payment processing pods)
    Non-CDE VPC: 10.2.0.0/16 (frontend, logging)
    Transit Gateway: controlled routing — non-CDE CANNOT route to CDE

  Kubernetes NetworkPolicy (CDE namespace isolation):
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: cde-isolation
      namespace: payment-cde
    spec:
      podSelector: {}     # all pods in namespace
      policyTypes: [Ingress, Egress]
      ingress:
      - from:
        - namespaceSelector:
            matchLabels:
              pci: "cde"  # only from other CDE namespaces
      egress:
      - to:
        - namespaceSelector:
            matchLabels:
              pci: "cde"
      - ports:            # allow egress to KMS, Secrets Manager
        - port: 443

  WAF on ALB: block non-HTTPS, rate limit, SQLi/XSS protection

Encryption (Requirements 3 & 4):

  In transit:
    TLS 1.2+ enforced everywhere (ALB security policy: ELBSecurityPolicy-TLS13-1-2)
    mTLS between CDE microservices (Istio/App Mesh)
    ALB listener: HTTP → HTTPS redirect (port 80 redirects to 443)

  At rest:
    EBS volumes: KMS CMK encryption (aws:kms)
    RDS: KMS CMK for storage encryption
    S3 (CHD): KMS CMK, S3 Object Lock (WORM) for compliance retention
    Secrets Manager: KMS-encrypted for all credentials
    etcd: EncryptionConfiguration with KMS provider
      → K8s Secrets encrypted at rest in etcd using KMS

  KMS key policies (least privilege):
    CDE encryption key: only accessible to payment-service IAM role
    Log encryption key: accessible to logging service + audit role
    Annual key rotation: enabled on all CMKs

Access Control (Requirements 7 & 8):

  No shared credentials anywhere:
    IRSA for pod-level AWS access (no static keys)
    AWS SSO for human access (temporary credentials, MFA required)
    IAM roles: named per service, scoped to required resources only

  Kubernetes RBAC:
    Namespace scoped: payment team can only manage payment-cde namespace
    ArgoCD AppProject: scoped to specific repos and namespaces
    OPA/Gatekeeper: enforce no-root, no-privileged, read-only root filesystem

  Network access:
    VPN or AWS Client VPN required for all SSH/kubectl access
    No direct public access to K8s API server
    Private EKS endpoint: kubectl via VPN only

Vulnerability Management (Requirement 6):

  Container image scanning (CI/CD):
    ECR scanning on push: auto-scan all images
    Block critical/high CVEs in CI gate:
      aws ecr describe-image-scan-findings --image-id imageTag=latest
      → CRITICAL findings → pipeline fails, image not deployed

  Kubernetes hardening:
    PodSecurityAdmission: restricted profile on CDE namespace
    No containers run as root (enforced by PSA)
    Read-only root filesystem on all CDE pods
    Regular kube-bench scans: CIS Kubernetes Benchmark

  Dependency scanning:
    Snyk or Dependabot: automated PR for CVE patches
    Weekly: scan production images for newly discovered CVEs

Incident Response (Requirements 10 & 12):

  Detection:
    GuardDuty: anomaly detection (unusual API calls, crypto mining)
    AWS Security Hub: aggregates findings from GuardDuty, Config, Inspector
    CloudWatch alarms: access to CDE outside business hours
    Prometheus alert: payment_error_rate > threshold

  72-hour notification requirement:
    PCI DSS 4.0: notify card brands within 72 hours of suspected breach
    EventBridge rule: GuardDuty HIGH severity → Step Functions IR workflow
    IR workflow:
      T+0: Alert triggered
      T+15: Automated containment (deny IAM policy on compromised principal)
      T+30: IR team notified via PagerDuty
      T+60: Initial assessment complete
      T+72: Card brand notification if breach confirmed

  Containment automation:
    Lambda: on GuardDuty finding → attach deny policy to compromised role
    Lambda: on suspicious pod activity → add NoSchedule taint to node
    Lambda: on RDS anomaly → force failover, enable enhanced monitoring

Tamper-proof Audit Logs (Requirement 10):

  CloudTrail: all API calls logged, immutable S3 storage
    S3 Object Lock: WORM (Write Once Read Many), 7-year retention
    aws cloudtrail create-trail \
      --name pci-audit-trail \
      --s3-bucket-name pci-cloudtrail-logs \
      --is-multi-region-trail \
      --enable-log-file-validation   # SHA-256 hash for integrity check

  Log file validation:
    aws cloudtrail validate-logs --trail-arn ... --start-time ...
    Detects: modified, deleted, or forged log files

  Kubernetes audit logs:
    EKS audit logs → CloudWatch Logs → S3 (immutable)
    Captures: who did what to which K8s object when

  Separation: log storage account different from production account
    Even if production account is compromised, logs remain intact
```

💡 **Interview tip:** PCI DSS interviews want to see depth across the 12 requirements, not just one or two. The most distinctive answers: KMS CMK for etcd encryption (most forget etcd contains all K8s Secrets in plaintext otherwise), S3 Object Lock for CloudTrail logs (WORM ensures tamper-proof audit trail), and the 72-hour breach notification automation (shows you understand compliance timelines, not just technical controls). For Kubernetes specifically, the CDE namespace with strict NetworkPolicy and PSA restricted profile shows you understand that compliance must extend into the container layer, not just the AWS infrastructure layer.

---

*DevOps/SRE Interview Questions Q226–Q270 — nawab312 GitHub repositories*
*Total question bank: Q1–Q270*
