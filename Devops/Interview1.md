# DevOps / SRE / Platform Engineer — Interview Question Bank
> 200+ questions across 10 topic areas. Moderate to difficult. No answers.

---

## 1. AWS

### Core Infrastructure & Compute
1. An EC2 instance is showing high CPU steal time. What does steal time mean, how do you diagnose it, and what are your options to fix it?
2. You launch an EC2 instance and it passes both status checks but your application is unreachable. Walk me through every possible reason.
3. What is the difference between an instance store and an EBS volume? When would you choose one over the other, and what happens to each when the instance stops/terminates?
4. Explain the difference between On-Demand, Reserved, Spot, and Savings Plans. Design a cost strategy for a mixed production + batch workload.
5. An Auto Scaling Group is not scaling out despite CPU being at 90%. What could prevent it from scaling?
6. What is EC2 placement groups? Compare cluster, partition, and spread placement groups and give a use case for each.
7. You need zero-downtime deployment on EC2 with an ALB. Walk me through the exact steps using ASG lifecycle hooks.

### Networking / VPC
8. Walk me through exactly what happens at the network level when an EC2 instance in a private subnet makes a request to the internet via a NAT Gateway.
9. What is the difference between a Security Group and a NACL? Give a scenario where a NACL is required but a Security Group alone is insufficient.
10. You have a VPC with CIDR 10.0.0.0/16. You need to add a new subnet but all IPs are allocated. What are your options?
11. Explain VPC Flow Logs — what they capture, what they don't capture, and how you'd use them to diagnose a connectivity issue.
12. What is AWS PrivateLink and when would you use it instead of VPC Peering?
13. A route table has two entries: 10.0.0.0/16 → local, and 0.0.0.0/0 → IGW. A new entry 10.0.1.0/24 → NAT is added. Explain routing precedence.
14. Your application in us-east-1 needs to communicate with a service in eu-west-1 privately without traversing the internet. What are your options?

### Storage
15. Compare S3 storage classes (Standard, IA, Glacier, Glacier Deep Archive). Design a lifecycle policy for logs that are frequently accessed for 30 days, occasionally for 90 days, then archived.
16. An S3 bucket policy and IAM policy conflict — one allows, one denies. What's the outcome? How does AWS evaluate these?
17. What is S3 Object Lock and how does it differ from versioning? When would you need both?
18. You're seeing high S3 latency for small object reads. What could cause this and how do you fix it?
19. EBS volume performance is degraded. Walk me through diagnosing whether it's an IOPS, throughput, or latency issue.

### Containers / Serverless
20. ECS task fails to pull an image from ECR. Walk me through every possible reason.
21. What is the difference between ECS task role and ECS execution role? What breaks if you confuse them?
22. A Lambda function works fine locally but times out in AWS. What are the possible causes?
23. Explain Lambda cold starts — what causes them, how to measure them, and every option to mitigate them.
24. What is the difference between Lambda provisioned concurrency and reserved concurrency?
25. You need a Lambda function to access an RDS database. What is the recommended pattern and why not just use credentials in environment variables?

### Databases
26. Your RDS instance has high read latency. Walk through your diagnosis — what metrics, what logs, what queries?
27. What is the difference between RDS Multi-AZ and Read Replicas? Can you use a Read Replica for failover?
28. Explain Aurora's storage architecture and why it's fundamentally different from standard RDS MySQL.
29. Your DynamoDB table has hot partitions. How do you detect this and what are your options to fix it?
30. Compare DynamoDB provisioned vs on-demand capacity. When does on-demand become more expensive?

### Observability
31. What is the difference between CloudWatch Metrics, CloudWatch Logs, and CloudWatch Logs Insights? When would you use each?
32. Explain CloudWatch Contributor Insights and give a real use case where it would catch something standard metrics wouldn't.
33. You need to correlate logs across multiple AWS services for a single user request. What's your architecture?
34. What is X-Ray sampling and why is 100% tracing often not the right choice in production?

### Security
35. What is the difference between an IAM role, IAM user, and IAM group? Why should applications never use IAM users?
36. Explain AWS STS AssumeRole. Walk me through the exact token flow when an ECS task assumes a role.
37. What is AWS Secrets Manager vs SSM Parameter Store? When would you choose one over the other?
38. Your AWS account shows unrecognized API calls in CloudTrail. Walk me through your incident response steps.
39. What is SCPs (Service Control Policies) in AWS Organizations? How do they interact with IAM policies?
40. Explain IMDSv2 and why IMDSv1 is a security risk. What attack does IMDSv2 prevent?

### Messaging
41. Compare SQS Standard vs FIFO queues. When does ordering matter and what are the FIFO throughput limits?
42. What is the difference between SQS visibility timeout and message retention period? What happens when a consumer crashes mid-processing?
43. Explain the fan-out pattern with SNS + SQS. Why not just use SNS directly to Lambda?
44. Your SQS queue depth is growing unbounded. Walk through your diagnosis and remediation options.

### Infrastructure Automation & Cost
45. What is AWS CDK vs CloudFormation vs Terraform? When would you choose CDK over Terraform?
46. Explain CloudFormation drift detection and its limitations.
47. Your AWS bill jumped 40% this month. Walk me through how you'd identify the cause using native AWS tools.
48. What are AWS Compute Optimizer recommendations based on, and what data does it need before recommendations are reliable?

---

## 2. CI/CD & Jenkins

### Architecture & Fundamentals
49. Explain the Jenkins master-agent architecture. What happens to running jobs if the master goes down?
50. What is the difference between Jenkins Freestyle jobs and Pipeline jobs? Why would you never use Freestyle in a modern setup?
51. How does Jenkins handle agent allocation for parallel stages? What happens if no agent is available?
52. What is the Jenkins build executor and how does setting executors to 0 on the master affect things?

### Pipelines
53. What is the difference between `node`, `agent`, and `label` in a Jenkins pipeline?
54. Explain `stash` and `unstash` in Jenkins pipelines. When is this necessary and what are the performance implications?
55. Your Jenkins pipeline passes on the first run but fails intermittently on subsequent runs. What are the common causes?
56. What is the difference between `parallel` stages in Jenkins and what happens when one parallel branch fails?
57. Explain Jenkins shared libraries — structure, loading mechanisms, and why they're critical for large teams.
58. What is the `Jenkinsfile` checkout scm behavior and when does it cause problems in multi-branch pipelines?

### Advanced Patterns
59. How do you implement blue-green deployments in a Jenkins pipeline?
60. What is Jenkins Configuration as Code (JCasC) and what problem does it solve?
61. Explain the Jenkins Kubernetes plugin — how does dynamic agent provisioning work and what are its failure modes?
62. How would you implement pipeline-level mutex/locking to prevent concurrent deploys to the same environment?
63. Your Jenkins agents are ephemeral containers. A build requires Docker-in-Docker. What are the security implications and alternatives?

### Security & Scaling
64. What is Jenkins credential binding and why should you never use `withCredentials` and then print environment variables?
65. How do you prevent a malicious `Jenkinsfile` from exfiltrating secrets in a multi-tenant Jenkins setup?
66. Jenkins master is the single point of failure. How would you architect Jenkins for HA?
67. What is the Jenkins job DSL plugin and how does it differ from pipelines?

---

## 3. Terraform

### Core Fundamentals
68. Explain the difference between `terraform plan` and `terraform apply`. What can cause them to produce different results when run back to back?
69. What is the Terraform dependency graph? How does Terraform determine the order of resource creation?
70. What is the difference between implicit and explicit dependencies in Terraform? Give a case where implicit dependency fails.
71. Explain `terraform refresh` and why it's dangerous in production.

### State Management
72. Your Terraform state file is corrupt. Walk me through recovery options.
73. What is `terraform state mv` and when would you use it? What happens if you move state without also moving the config?
74. Two engineers accidentally ran `terraform apply` simultaneously and now state is inconsistent. How do you diagnose and fix this?
75. What is a Terraform workspace and when should you NOT use workspaces for environment separation?
76. Explain the implications of storing Terraform state in S3 without versioning enabled.

### Modules & Patterns
77. What is the difference between a root module and a child module? How does variable passing work between them?
78. You have a module that creates an S3 bucket. You need to add a lifecycle rule but the module doesn't support it. What are your options without forking the module?
79. Explain `terraform registry` module versioning. Why should you pin module versions in production?
80. What is the `moved` block in Terraform and when was it introduced? What problem does it solve over `terraform state mv`?

### Advanced
81. What is `dynamic` block in Terraform? Give a real use case and explain why overusing it is a code smell.
82. Explain `for_each` vs `count` tradeoffs in detail — when does each cause problems?
83. What is a `null_resource` and `terraform_data`? Give a legitimate use case.
84. Explain Terraform's `lifecycle { prevent_destroy }` — what does it protect against and what doesn't it protect against?
85. What is Terragrunt and what problems does it solve that vanilla Terraform doesn't?
86. How do you test Terraform modules? Compare Terratest, `terraform validate`, `tflint`, and Checkov.

---

## 4. GitHub Actions

### Architecture & Fundamentals
87. Explain the GitHub Actions execution model — what is a runner, what is a job, and how does context propagate between steps?
88. What is the difference between `on: push` and `on: workflow_dispatch`? How do you restrict `workflow_dispatch` to specific branches?
89. Explain GitHub Actions expressions syntax — `${{ }}`. What is the difference between `github`, `env`, `secrets`, and `vars` contexts?
90. What is the difference between `jobs.<job>.env` and `steps.<step>.env`? Which takes precedence?

### Security
91. A GitHub Actions workflow runs on `pull_request_target`. Why is this dangerous and when is it required?
92. Explain GITHUB_TOKEN permissions — what it can and cannot do by default, and how to scope it down.
93. How do you prevent secret exfiltration from GitHub Actions when using third-party actions?
94. What is OpenID Connect (OIDC) in GitHub Actions and how does it eliminate the need for stored cloud credentials?

### Advanced Patterns
95. How do you implement a matrix build that skips certain combinations? Show the `exclude` syntax.
96. What is the `concurrency` key in GitHub Actions and what's the difference between `cancel-in-progress: true` vs `false`?
97. How do you pass data between jobs in GitHub Actions? What are the size limits?
98. Explain GitHub Actions cache — how does cache key resolution work and what happens on a cache miss?
99. Your GitHub Actions workflow needs to deploy to 5 environments sequentially, each requiring manual approval. How do you design this?

### Self-Hosted Runners & Monorepos
100. What are the security risks of self-hosted runners on public repositories?
101. How do you implement path-based filtering in a monorepo to only run workflows for changed services?
102. Explain GitHub Actions `needs` dependency and how it handles failures in dependent jobs.
103. What is a composite action vs a reusable workflow? When would you choose each?

---

## 5. Kubernetes

### Architecture
104. Explain what happens step by step when you run `kubectl apply -f deployment.yaml` — from API server to pod running.
105. What is the role of the controller manager? How does the ReplicaSet controller work?
106. Explain etcd's role in Kubernetes. What happens if etcd loses quorum?
107. What is the difference between the API server's watch mechanism and polling? How does this affect controller performance?
108. Explain the Kubernetes admission controller pipeline — what is the difference between validating and mutating webhooks?

### Workloads
109. What is the difference between a Deployment and a StatefulSet? When must you use a StatefulSet?
110. Explain the difference between `RollingUpdate` and `Recreate` deployment strategies. When is `Recreate` the right choice?
111. What is a PodDisruptionBudget and why can it prevent node draining?
112. Explain Kubernetes Job vs CronJob failure handling — `backoffLimit`, `activeDeadlineSeconds`, and `concurrencyPolicy`.
113. What is an init container and how does it differ from a sidecar? Give a production use case for each.
114. Your Deployment has 3 replicas. You update the image and all 3 pods crash. Explain what `maxUnavailable` and `maxSurge` control and how to prevent complete outage.

### Networking
115. Explain kube-proxy modes — iptables vs ipvs. What are the performance differences at scale?
116. What is a Headless Service and when would you use it instead of a ClusterIP service?
117. Explain how DNS resolution works in Kubernetes — from pod making a DNS query to receiving an answer.
118. What is an Ingress controller? How does it differ from a LoadBalancer service?
119. Explain NetworkPolicy — what is default behavior without any NetworkPolicy, and how do you implement default-deny?
120. A service has endpoints but traffic is being dropped. What layer would you check next after confirming endpoints exist?

### Storage
121. Explain the PV/PVC/StorageClass relationship. What is dynamic provisioning?
122. What is the `ReadWriteMany` access mode and why do most cloud block storage providers not support it?
123. Explain CSI (Container Storage Interface) and why it replaced in-tree volume plugins.
124. A pod with a PVC is stuck in `Terminating` state for 30 minutes. What is causing this and how do you fix it?

### Scheduling & Resources
125. Explain QoS classes in Kubernetes — Guaranteed, Burstable, BestEffort — and how they affect OOMKill priority.
126. What is the difference between resource requests and limits? What happens when a node is overcommitted?
127. Explain pod affinity vs pod anti-affinity. Give a production scenario where anti-affinity is critical.
128. What is the Cluster Autoscaler and what conditions must be true before it scales down a node?
129. Explain taints and tolerations with a real multi-tenant cluster use case.

### Security
130. What is a ServiceAccount token and how does the projected volume automount work in newer Kubernetes versions?
131. Explain RBAC in Kubernetes — Role vs ClusterRole, RoleBinding vs ClusterRoleBinding.
132. What is the principle of least privilege applied to a pod — list every security context field you should set.
133. What is Falco and what class of threats does it detect that RBAC and PSA cannot?

### Operations & Troubleshooting
134. A node is in `NotReady` state. Walk through your complete diagnosis.
135. `kubectl drain` is hanging. What are the possible reasons and how do you force it?
136. Explain what happens during a graceful pod termination — `SIGTERM`, `preStop` hooks, `terminationGracePeriodSeconds`.
137. You need to do a cluster upgrade from 1.27 to 1.29. What is the correct procedure and what are the risks?
138. Explain etcd backup and restore. How often should you back up etcd in production?

### Istio / Service Mesh
139. What problem does a service mesh solve that Kubernetes networking alone doesn't?
140. Explain Istio's sidecar injection mechanism. What happens if the sidecar fails to start?
141. What is an Istio VirtualService vs DestinationRule? Give a canary deployment example.
142. How does mutual TLS (mTLS) work in Istio and what does it protect against?

---

## 6. Prometheus & Grafana

### Core Concepts & Architecture
143. Explain the Prometheus pull model. What are its advantages and disadvantages vs a push model?
144. What is a Prometheus target and how does the scrape lifecycle work — what happens on a failed scrape?
145. Explain the difference between a Counter, Gauge, Histogram, and Summary. When would you use a Histogram over a Summary?
146. What is cardinality in Prometheus and why does high cardinality kill performance?
147. Explain Prometheus TSDB — how does it store data on disk, what are blocks and the WAL?

### PromQL
148. What is the difference between `rate()` and `irate()`? When does `irate()` give misleading results?
149. Explain the difference between `sum by` and `sum without`. When would you use each?
150. What is a range vector vs an instant vector in PromQL?
151. Write a query to find the 95th percentile request latency from a Histogram metric.
152. What is subquery in PromQL and when is it necessary?
153. Explain `offset` modifier and give a use case for week-over-week comparison.

### Alerting & Alertmanager
154. What is the difference between `for:` duration and `keep_firing_for:` in Prometheus alerts?
155. Explain Alertmanager routing tree — how does it match alerts to receivers?
156. What is alert inhibition and when would you use it to prevent alert storms?
157. Explain the difference between `group_wait`, `group_interval`, and `repeat_interval` in Alertmanager.
158. Your Alertmanager is sending duplicate alerts. What are the possible causes?

### Service Discovery & Exporters
159. Explain Kubernetes service discovery in Prometheus — how does `kubernetes_sd_configs` work?
160. What is a relabeling rule in Prometheus and what is the difference between `relabel_configs` and `metric_relabel_configs`?
161. What is the Node Exporter and what can't it collect that requires a custom exporter?
162. Explain the Blackbox exporter — what types of probes does it support and what is it used for?

### HA & Scaling
163. How do you run Prometheus in HA mode? What is the problem with simple active-active?
164. What is Thanos and what components does it add to a Prometheus stack?
165. What is the difference between Thanos and Cortex/Mimir for long-term storage?

### Grafana & LGTM Stack
166. What is the difference between Grafana data sources and panels? How does templating work?
167. Explain Grafana Loki's architecture — how does it differ from Elasticsearch for log storage?
168. What is Grafana Tempo and how does it integrate with Prometheus and Loki for correlation?
169. Your Grafana dashboard is slow to load. What are the possible causes and fixes?

---

## 7. Linux & Bash

### Process Management
170. What is the difference between a process and a thread in Linux? How does the kernel schedule them?
171. Explain zombie processes — what causes them, how do you find them, and are they harmful?
172. What is `nice` vs `ionice`? When would you adjust process priority in production?
173. Explain Linux OOM killer — how does it choose which process to kill?
174. What is `cgroups` and how does Kubernetes use it to enforce resource limits?

### Bash Scripting
175. What is the difference between `$()` and backticks in Bash? Are there cases where they behave differently?
176. Explain `set -e`, `set -u`, `set -o pipefail` — what does each do and why should all three be in production scripts?
177. What is the difference between `[[ ]]` and `[ ]` in Bash conditionals?
178. How do you handle signals in a Bash script? Write a trap that cleans up temp files on EXIT.
179. Explain process substitution `<()` and give a use case where it's necessary.

### Networking
180. What does `ss -tlnp` show that `netstat` doesn't? What do the flags mean?
181. Walk me through using `tcpdump` to capture HTTP traffic on port 8080 and write it to a file.
182. What is `conntrack` and why do you need to understand it when debugging iptables rules?
183. Explain the difference between `ping`, `traceroute`, and `mtr`. What does each tell you?

### Storage & I/O
184. What is `inode` exhaustion and how do you diagnose it? Can a disk be full with free space showing?
185. Explain `iostat` output — what is `await`, `svctm`, and `%util`? What does 100% util actually mean?
186. What is the Linux page cache? How does it affect memory usage reporting?
187. Explain `lsof` and give three production use cases where it's essential.

### Performance & Observability
188. Explain the USE method for performance analysis. What does Utilization, Saturation, and Errors mean for CPU?
189. What is `perf` and what types of performance problems can it diagnose that `top` cannot?
190. Explain `strace` vs `ltrace`. When would you use each and what are the performance implications?
191. What is `dmesg` and what types of issues appear there that won't show in application logs?

---

## 8. ArgoCD

### Core Concepts & Architecture
192. Explain ArgoCD's reconciliation loop — how does it detect drift and what triggers a sync?
193. What is the difference between ArgoCD Application `source` and `destination`?
194. Explain ArgoCD's `health` status vs `sync` status — can an app be `Synced` but `Degraded`?
195. What is the ArgoCD repo server and what happens if it goes down?

### Sync & Deployment Patterns
196. What is the difference between `prune` and `selfHeal` in ArgoCD sync policy?
197. Explain ArgoCD sync waves and sync hooks — how do you control order of resource deployment?
198. What is an ArgoCD `PreSync`, `Sync`, and `PostSync` hook? Give a database migration use case.
199. How do you implement a canary deployment pattern using ArgoCD and Argo Rollouts?
200. What is the App of Apps pattern? What are its limitations at scale?

### ApplicationSets & Multi-Cluster
201. What is an ApplicationSet and how does the `git` generator differ from the `cluster` generator?
202. How do you manage secrets in ArgoCD — what are the options and tradeoffs (Sealed Secrets, ESO, Vault)?
203. Explain ArgoCD multi-cluster deployment — how does ArgoCD authenticate to remote clusters?
204. What happens during ArgoCD disaster recovery — what do you need to back up?

### RBAC & Security
205. Explain ArgoCD RBAC — how does it integrate with SSO (Dex) and what is a project policy?
206. What is an ArgoCD AppProject and how does it restrict what an application can deploy?

---

## 9. Python Scripting

### Automation & AWS
207. Write a Python script that finds all EC2 instances tagged with `Env=prod` and stops any that have been running for more than 7 days without recent CPU activity.
208. How do you handle AWS API pagination in Boto3? What happens if you don't handle it?
209. Explain Boto3 session vs client vs resource. When would you use each?
210. Write a Python function that retries an AWS API call with exponential backoff on `ThrottlingException`.
211. How do you assume a cross-account IAM role in Boto3 and use it to list S3 buckets?

### Error Handling & Patterns
212. What is the difference between `Exception` and `BaseException` in Python? When should you catch `BaseException`?
213. Explain Python context managers — implement one using `__enter__`/`__exit__` for a temp directory.
214. What is the difference between `subprocess.run`, `subprocess.Popen`, and `os.system`? Why is `os.system` dangerous?
215. How do you parse and manipulate large JSON/CSV files in Python without loading them into memory?

### CLI & Testing
216. Compare `argparse`, `click`, and `typer` for building CLI tools. When would you choose each?
217. How do you mock AWS services in Python unit tests without making real API calls?
218. What is the difference between `unittest.mock.patch` and `unittest.mock.MagicMock`?
219. Explain Python's `asyncio` — when would async patterns help in a DevOps automation script?

---

## 10. ELK Stack

### Elasticsearch Architecture
220. Explain Elasticsearch sharding — what is a primary shard vs replica shard and what happens during a node failure?
221. What is the difference between an Elasticsearch index and a data stream?
222. Explain the Elasticsearch write path — from document indexing to it being searchable.
223. What is a split-brain scenario in Elasticsearch and how does `minimum_master_nodes` (now `cluster.initial_master_nodes`) prevent it?
224. Compare Elasticsearch to OpenSearch — what diverged after the fork?

### Performance & Operations
225. Your Elasticsearch cluster is red. Walk me through your diagnosis from first principles.
226. What is heap pressure in Elasticsearch and what are the symptoms of too-small heap?
227. Explain ILM phases (hot/warm/cold/frozen/delete) and the difference between `rollover` and `shrink` actions.
228. What is force merge in Elasticsearch and why is it dangerous to run on active indices?
229. Explain fielddata vs doc values. Why does enabling fielddata on text fields cause memory issues?

### Querying & Mappings
230. What is dynamic mapping in Elasticsearch and why should you disable it in production?
231. Explain the difference between `term` query and `match` query. When does `term` fail on text fields?
232. What is an Elasticsearch analyzer and what are the three components of an analysis chain?
233. Write an Elasticsearch query that finds all logs with level=ERROR in the last hour, aggregated by service name.

### Logstash & Beats
234. Explain the Logstash pipeline — input, filter, output. What happens when the output is unavailable?
235. What is the difference between Filebeat and Logstash? When do you need both?
236. What is the Logstash persistent queue and why is it important for production?
237. Explain Beats autodiscover for Kubernetes — how does it handle pod restarts and log rotation?

### Security & Scaling
238. Explain Elasticsearch X-Pack security — index-level vs document-level security.
239. How do you scale Elasticsearch for write-heavy vs read-heavy workloads differently?
240. What is a hot-warm-cold architecture in Elasticsearch and what hardware characteristics should each tier have?

---

## Coding / Scripting Questions

### Bash
- Write a one-liner that finds all files modified in the last 24 hours under `/var/log` and counts lines containing `ERROR`.
- Write a Bash script that takes a list of servers and checks if port 443 is open on each, with a 2-second timeout.
- Write a script that rotates logs — compresses files older than 7 days, deletes files older than 30 days.
- Write a Bash function that retries a command up to 5 times with exponential backoff.

### Python
- Write a Python function using Boto3 that lists all unattached EBS volumes across all regions.
- Write a Python script that reads a CSV of EC2 instance IDs and tags them with `Owner` from a second column.
- Write a Python function that polls an SQS queue, processes messages, and handles visibility timeout extension for long-running tasks.
- Write a Python CLI tool using `argparse` that takes a Kubernetes namespace and lists all pods with their restart counts.

### Terraform
- Write a Terraform module that creates an S3 bucket with versioning, server-side encryption, and a lifecycle policy.
- Write a `for_each` resource block that creates IAM users from a map of username → email.
- Write a data source block that looks up an existing VPC by tag and uses its ID in a subnet resource.

### Kubernetes YAML
- Write a Deployment manifest with resource requests/limits, liveness/readiness probes, and a non-root security context.
- Write a NetworkPolicy that allows ingress only from pods with label `app=frontend` in the same namespace.
- Write a CronJob manifest that runs every 6 hours, keeps 3 successful jobs, deletes failed jobs immediately.
- Write a HorizontalPodAutoscaler that scales between 2 and 10 replicas based on 70% CPU utilization.

### GitHub Actions YAML
- Write a workflow that builds a Docker image, pushes to ECR, and triggers only on changes to the `src/` directory.
- Write a matrix workflow that runs tests on Python 3.9, 3.10, 3.11 in parallel.
- Write a workflow that uses `concurrency` to cancel in-progress runs on the same branch but never cancel a deploy to prod.

### PromQL
- Write a query that shows the top 5 pods by memory usage in a namespace.
- Write an alert rule that fires when a pod has restarted more than 5 times in the last hour.
- Write a query that calculates the error rate percentage for a service broken down by endpoint.

### ArgoCD YAML
- Write an ArgoCD Application manifest that syncs from a Helm chart in a private Git repo with auto-sync and self-heal enabled.
- Write an ApplicationSet using the `git` directory generator to create one Application per directory under `apps/`.

---

*Total: 240+ questions across 10 domains*
*Coding questions: 30+ covering Bash, Python, Terraform, K8s YAML, GitHub Actions, PromQL, ArgoCD*
