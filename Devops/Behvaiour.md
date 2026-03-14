# Behavioral Interview — STAR Stories
### Siddharth Singh | Senior DevOps / SRE Engineer

> These are your 6 polished STAR stories for behavioral interview questions.
> Each story is based on your real experience from your resume.
> Practice saying them out loud — aim for 2-3 minutes per story.

---

## What is STAR Format?

| Letter | Meaning | What to cover |
|---|---|---|
| **S** | Situation | Context — what was happening, what was at stake |
| **T** | Task | Your specific responsibility in that situation |
| **A** | Action | Exactly what YOU did — step by step |
| **R** | Result | Measurable outcome — numbers, business impact |

> ⚠️ **Interview tip:** Always say "I" not "we" when describing actions. Interviewers want to know YOUR contribution specifically.

---

---

## Story 1 — Production Incident You Handled

### Question Triggers:
- *"Tell me about a time you handled a critical production incident"*
- *"Describe a situation where you had to troubleshoot under pressure"*
- *"Tell me about your biggest on-call experience"*

---

### STAR Answer:

**Situation:**
At Infosys, I was responsible for the infrastructure supporting PNB's mobile banking application — a mission-critical system used by millions of customers. We had a production incident where the application started throwing 500 errors intermittently. This was during peak banking hours and the business impact was significant — customers could not complete transactions. The on-call alert fired at an inconvenient time and the root cause was not immediately obvious.

**Task:**
As the Senior DevOps engineer on-call, my responsibility was to identify the root cause, restore service as quickly as possible, and prevent recurrence. I had to coordinate across the development team, DBA team, and business stakeholders simultaneously while actively debugging.

**Action:**
I immediately checked our **Grafana dashboards** and noticed memory usage on the Kubernetes pods was spiking to the limit just before each wave of 500 errors. I ran `kubectl top pods` and `kubectl describe pod` and confirmed pods were being OOMKilled and restarting in a loop — CrashLoopBackOff. I then checked the **ELK Stack** logs and traced the spike to a specific API endpoint that was making unoptimized database queries, returning massive payloads. The issue had been introduced in the last deployment. I took three immediate actions: first, I scaled up the deployment replicas to absorb traffic while we fixed the root cause; second, I increased the memory limits temporarily as a stopgap; third, I rolled back the specific service to the previous version using `kubectl rollout undo` while I notified the development team about the problematic query. I also set up a specific **CloudWatch alarm** and Prometheus alert on memory utilization percentage so this pattern would be caught earlier in future.

**Result:**
Service was restored within **22 minutes** of the alert firing. The rollback eliminated the OOMKill loop immediately. Post-incident, I worked with the dev team to fix the query and added memory-based HPA scaling so the application auto-scales before hitting OOM thresholds. This contributed to the **40% reduction in incident resolution time** we achieved that year and helped us maintain **99.9% uptime** for the RDS database and banking application infrastructure.

---

---

## Story 2 — System You Built From Scratch

### Question Triggers:
- *"Tell me about the most impactful system you built"*
- *"Describe a project you are most proud of"*
- *"Walk me through a CI/CD pipeline you designed end-to-end"*

---

### STAR Answer:

**Situation:**
When I joined Infosys as a System Engineer working on the PNB Mobile Banking application, the deployment process was entirely manual. Developers would hand off builds to the ops team, who would manually SSH into servers, copy artifacts, restart services, and verify deployments. This was taking 4-6 hours per deployment cycle, deployments happened only once a week, and there were frequent rollback situations because there was no automated testing gate. The team was spending more time on deployments than on actual engineering work.

**Task:**
I was tasked with designing and implementing a complete end-to-end CI/CD pipeline from scratch for a Java-based microservices application deployed on Kubernetes (EKS). The pipeline had to cover build, test, security scanning, containerization, and deployment — all automated.

**Action:**
I designed a multi-stage **Jenkins pipeline** integrated with **ArgoCD** for GitOps-based continuous delivery. The pipeline worked as follows: on every pull request, Jenkins triggered unit tests, code quality checks via SonarQube, and security scans. On merge to main, Jenkins built the Docker image, pushed it to **JFrog Artifactory**, and updated the image tag in the Helm chart values file in the GitOps repository. **ArgoCD** detected the change and automatically deployed to the EKS cluster using the updated Helm chart. I implemented **rolling update** and **blue-green deployment** strategies in the Helm charts so zero-downtime deployments were guaranteed. I also set up automated health checks post-deployment — if the new pods did not pass readiness probes within 5 minutes, ArgoCD automatically rolled back. I documented the entire pipeline and trained the team on the GitOps workflow.

**Result:**
Deployment time dropped from **4-6 hours to under 30 minutes** — a **75% reduction**. The team went from weekly deployments to deploying multiple times per week with confidence. Rollback incidents dropped significantly because automated testing caught regressions before they reached production. This was the system I am most proud of because it fundamentally changed how the team operated — from reactive firefighting during deployments to confident, automated releases.

---

---

## Story 3 — Failure / Deployment Gone Wrong

### Question Triggers:
- *"Tell me about a time you made a mistake and what you learned"*
- *"Describe a deployment that did not go as planned"*
- *"Tell me about a time something you built caused a problem"*

---

### STAR Answer:

**Situation:**
At Infosys, during the early stages of the Terraform infrastructure automation initiative, I was migrating manually provisioned AWS infrastructure to Terraform-managed infrastructure. This was for banking application servers supporting multiple Regional Rural Banks (RRBs). I had been working on writing Terraform modules and was eager to show progress. I ran `terraform apply` on what I believed was the staging environment configuration. The command executed successfully — but I realized moments later that I had applied it against the **production environment state** because I had forgotten to switch workspace. This caused several EC2 instances and their associated security groups to be recreated, causing a **15-minute outage** for two RRB banking applications.

**Task:**
My immediate task was to restore service as fast as possible and minimize customer impact. My longer-term task was to do a proper post-incident analysis and put safeguards in place so this could never happen again.

**Action:**
I immediately checked the AWS console, identified the recreated resources, and verified that the RDS databases were untouched (I had `prevent_destroy = true` on the database resources, which saved us from a much worse situation). I restored service by re-applying the correct configuration and verifying security group rules were correct. Applications came back online within 15 minutes. For the post-incident fix, I implemented three safeguards: first, I restructured the Terraform codebase into **separate directories per environment** (`environments/prod`, `environments/staging`) instead of using workspaces — making it impossible to accidentally target the wrong environment. Second, I added `prevent_destroy = true` lifecycle rules to ALL production resources. Third, I enforced that all production `terraform apply` commands could only run through the **CI/CD pipeline** with a manual approval gate — no direct local applies allowed for production. I also wrote a runbook for the team covering safe Terraform practices.

**Result:**
The outage was contained to 15 minutes. More importantly, in the **18 months** following this incident, we had **zero infrastructure-related outages** caused by Terraform misapplication. The separate directory structure and CI/CD-enforced applies became the team standard. This was a painful lesson but it made our infrastructure automation significantly more reliable. I also learned that `prevent_destroy` on critical resources is non-negotiable.

---

---

## Story 4 — Process You Improved / Automation

### Question Triggers:
- *"Tell me about a time you improved a process significantly"*
- *"Describe a situation where you identified inefficiency and fixed it"*
- *"Tell me about your biggest infrastructure automation achievement"*

---

### STAR Answer:

**Situation:**
At Infosys, the infrastructure provisioning process for banking application servers across multiple RRBs and PNB servers was completely manual. When a new server was needed or configuration needed to be updated, an engineer would SSH into each server individually, run commands manually, and document what was done in a spreadsheet. With over 100 servers across multiple banks, this was taking **3-4 days per provisioning request** and was highly error-prone — servers were often in inconsistent states because manual steps were missed or performed differently by different engineers. Compliance audits were also painful because there was no single source of truth for server configuration.

**Task:**
I was given ownership of the configuration management initiative. My task was to standardize server provisioning and application deployment across all RRB and PNB servers and make it repeatable, consistent, and fast.

**Action:**
I implemented **Ansible** as the configuration management tool and **Terraform** for infrastructure provisioning. I started by auditing the existing server configurations across all environments and documenting the desired state. I then wrote Ansible playbooks covering: OS hardening, package installation, application deployment, SSL/TLS certificate configuration, iptables firewall rules, SSH hardening, cron job setup for log rotation and backups, and health check scripts. I organized the playbooks in a role-based structure so each component was reusable. I integrated these Ansible playbooks into the **Jenkins CI/CD pipeline** so that every infrastructure change was version controlled in Git, reviewed via pull request, and applied automatically. I also set up **Ansible facts** collection to maintain a live inventory of all server configurations, which replaced the outdated spreadsheet tracking. For new server provisioning, I combined Terraform (to create the EC2 instance) with Ansible (to configure it), making the entire workflow automated end-to-end.

**Result:**
Server provisioning time dropped from **3-4 days to 2-4 hours** — a **70-80% reduction** in manual workload. Configuration drift across servers was eliminated because everything was defined as code. Compliance audits became straightforward because we could show exactly what configuration was applied to every server and when. The team repurposed the time saved on manual provisioning toward higher-value engineering work. This was also the initiative that gave me confidence in Terraform, which I subsequently applied to automate the broader AWS infrastructure — leading to the **40-50% reduction in manual infrastructure processes** mentioned in my performance review.

---

---

## Story 5 — Disagreement / Conflict with Teammate or Manager

### Question Triggers:
- *"Tell me about a time you disagreed with a technical decision"*
- *"Describe a conflict with a colleague and how you resolved it"*
- *"Tell me about a time you had to push back on your manager"*

---

### STAR Answer:

**Situation:**
At Coffeebeans, during the database migration project — migrating Java and Python applications from IBM DB2 to PostgreSQL — there was a disagreement between me and the development lead about the deployment strategy for the migrated applications on AWS ECS. The development lead wanted to do a **big-bang cutover** — migrate everything at once over a single weekend, switch all traffic to the new PostgreSQL-backed application, and decommission DB2 immediately. I had serious concerns about this approach for mission-critical banking data pipelines. The stakes were high because any data discrepancy or application compatibility issue would affect live banking data.

**Task:**
My responsibility was both the infrastructure readiness and the observability setup for the migration. I felt strongly that the big-bang approach was too risky and I needed to make my case without creating team conflict, since the development lead had significantly more tenure at the company.

**Action:**
Rather than simply opposing the plan in a team meeting (which would have put people on the defensive), I prepared a structured technical risk analysis. I documented specifically: what could go wrong with a big-bang cutover (untested edge cases in production data, PostgreSQL query plan differences causing timeouts, batch job failures), what the rollback process would look like under each approach, and the estimated time to detect and recover from each failure mode. I proposed an alternative — a **parallel run strategy** where both DB2 and PostgreSQL backends ran simultaneously, with our Java applications writing to both and the results compared via **CloudWatch alarms** I would set up to detect discrepancies. I presented this to the development lead privately first, not in the full team meeting, which gave him the chance to consider it without feeling publicly challenged. He asked good questions and raised a valid concern — the parallel run would extend the migration timeline by 3 weeks. I acknowledged this trade-off but showed the risk of a failed big-bang cutover could cost far more time in incident recovery. He agreed to take it to the project manager together.

**Result:**
The project manager approved the parallel run approach. We ran DB2 and PostgreSQL in parallel for 3 weeks, and during that period we caught **4 data discrepancy issues** in edge-case transactions that would have caused silent data corruption under the big-bang approach. We fixed them before the final cutover. The migration completed successfully with zero data loss and zero downtime. The development lead later told me the parallel run was the right call. More importantly, I learned that technical disagreements are best resolved with data and private conversation before escalating — and that being right matters less than being aligned.

---

---

## Story 6 — Most Proud Achievement / Cost Optimization

### Question Triggers:
- *"What is your most significant achievement as a DevOps engineer?"*
- *"Tell me about a time you had significant business impact"*
- *"Describe a project where you showed ownership and initiative"*

---

### STAR Answer:

**Situation:**
At Coffeebeans, after completing the ECS deployment and infrastructure automation work, I noticed during routine monitoring that our AWS costs had been growing month-over-month with no clear visibility into which teams or services were driving the growth. We were running an EKS cluster with multiple namespaces and services, but there was no way to tell which namespace was consuming the most resources or whether those resources were actually being used efficiently. Engineering teams were not aware of their infrastructure costs and had no incentive to optimize. The monthly AWS bill was becoming a concern for the business.

**Task:**
I took ownership of implementing a cost visibility and optimization solution. This was not explicitly assigned to me — I identified the problem and proposed the solution to my manager, who gave me the green light to implement it alongside my regular work.

**Action:**
I deployed **Kubecost** on the EKS cluster — an open-source cost monitoring tool that provides namespace-level and service-level cost attribution. I configured it to pull actual AWS pricing data and allocate costs based on real resource consumption (CPU, memory, storage, network). I then built a **custom Grafana dashboard** that showed: current month cost per namespace, cost trend over the last 30 days per team, CPU and memory utilization vs what was requested (to identify over-provisioned workloads), and projected monthly cost. I integrated the Grafana dashboard into the team's weekly engineering review so every team could see their own costs. I also used the **VPA (Vertical Pod Autoscaler) in Off mode** — using Goldilocks to generate right-sizing recommendations for every deployment. I identified several deployments that had CPU requests of 2 cores but were using less than 200m on average. I presented these findings to the respective teams with specific recommendations. Additionally, I identified that several batch jobs on AWS Batch were using On-Demand instances for short-duration jobs that could safely use Spot instances, and made that configuration change.

**Result:**
Within **6 weeks** of implementing Kubecost and acting on the right-sizing recommendations, we reduced the EKS-related AWS costs by approximately **25-30%** without any degradation in performance or reliability. Teams became cost-aware for the first time — engineers started checking the Grafana cost dashboard in weekly reviews and making more efficient resource requests in their deployments. This was the initiative I am most proud of because I identified it independently, built the complete solution, and created a lasting culture change around cost ownership. It also directly contributed to the cost analytics work listed in my resume and demonstrated that observability applies not just to reliability but also to financial efficiency.

---

---

## Quick Reference — Which Story to Use When

| Interview Question | Best Story |
|---|---|
| "Tell me about a production incident" | Story 1 — OOMKill CrashLoopBackOff incident |
| "Most impactful thing you built" | Story 2 — CI/CD pipeline (75% deployment reduction) |
| "Tell me about a failure / mistake" | Story 3 — Wrong Terraform workspace (15-min outage) |
| "Biggest process improvement" | Story 4 — Ansible automation (70-80% time saving) |
| "Handled a disagreement" | Story 5 — DB2 to PostgreSQL migration strategy conflict |
| "Most proud achievement" | Story 6 — Kubecost cost visibility (25-30% AWS cost reduction) |
| "Tell me about ownership / initiative" | Story 6 — Kubecost (self-initiated) |
| "How do you handle pressure" | Story 1 — Production incident under pressure |
| "Example of collaboration" | Story 5 — Disagreement resolved privately with data |
| "Tell me about a time you showed leadership" | Story 4 — Ansible standardization across 100+ servers |

---

## Key Numbers to Remember

> These are your **proof points** — memorize them. Interviewers love specifics.

| Achievement | Number |
|---|---|
| Deployment time reduction (CI/CD pipeline) | **75%** |
| Manual workload reduction (Ansible automation) | **70-80%** |
| Terraform manual process reduction | **40-50%** |
| Incident resolution time improvement | **40%** |
| System uptime achieved | **99.9%** |
| ELK Stack — RCA time reduction | **40%** |
| Kubernetes downtime reduction | **30-50%** |
| AWS cost reduction (Kubecost initiative) | **25-30%** |
| Servers managed via Ansible | **100+** |
| Production outage duration (Terraform incident) | **15 minutes** (then zero for 18 months) |

---

## Tips for Delivering These Stories

**DO:**
- ✅ Keep each story to **2-3 minutes** when spoken aloud
- ✅ Emphasize **"I"** actions — what YOU specifically did
- ✅ Always end with a **number or business outcome**
- ✅ Mention the **tools** you used — shows technical depth
- ✅ For Story 3 (failure) — spend 60% of the time on what you LEARNED and FIXED, not on the mistake itself
- ✅ Practice out loud at least 3 times per story

**DON'T:**
- ❌ Don't say "we did everything together" — own your contributions
- ❌ Don't make the failure story too catastrophic — keep it proportionate
- ❌ Don't memorize word-for-word — know the key points and speak naturally
- ❌ Don't skip the Result — it's the most important part

---

## Bonus — Common Follow-up Questions Per Story

### After Story 1 (Incident):
- *"What would you have done differently?"* → Set up memory-based HPA from day one, not after the incident
- *"How did you communicate with stakeholders during the incident?"* → Immediate Slack update to the manager with estimated resolution time, 5-minute status updates after that

### After Story 2 (CI/CD Pipeline):
- *"How did you handle resistance to the new process?"* → Ran the old and new pipeline in parallel for 2 weeks so the team could see the difference before committing
- *"What was the hardest part?"* → Getting ArgoCD and Jenkins to work together cleanly — the handoff from Jenkins building the image to ArgoCD detecting the Helm values change

### After Story 3 (Failure):
- *"How did your manager react?"* → He was understandably concerned but focused on the fix — the safeguards I put in place rebuilt trust quickly
- *"Would you do anything differently?"* → Yes — I should have validated the workspace in every apply with a pre-apply script check before touching any production work

### After Story 4 (Automation):
- *"How did you get buy-in?"* → Ran a proof of concept on 5 servers first, showed the time comparison, then proposed rolling it out to all servers
- *"What was the biggest challenge?"* → Legacy server configurations were inconsistent — I had to write idempotent Ansible tasks that handled both old and new configurations gracefully

### After Story 5 (Conflict):
- *"What if the development lead had still disagreed?"* → I would have escalated to the project manager with both positions clearly documented, letting them make the call
- *"What did you learn about working with others?"* → Data wins arguments, but timing and approach matter as much as the data itself

### After Story 6 (Cost Optimization):
- *"How did you get engineering teams to care about costs?"* → Made it visible in their weekly review — people act on what they can see
- *"What would you do next to optimize further?"* → Implement Spot instances for more workloads and use KEDA to scale to zero during off-hours

---

*Prepared based on Siddharth Singh's resume — Coffeebeans + Infosys experience*
*Target role: Senior DevOps / SRE Engineer*
