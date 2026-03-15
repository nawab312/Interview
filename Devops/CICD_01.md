# CI/CD — Complete Gaps Reference
## Every Missing Topic from Q1–Q675 | Key Points + Interview Tips

> **What this file covers:** Every CI/CD topic missing or only mentioned in
> passing across all previous question files. After this file, CI/CD is 100% covered.

---

## CONFIRMED GAPS BEING FILLED

| # | Topic | Previously |
|---|---|---|
| 1 | **Shift Left Security / DevSecOps** | ❌ Zero coverage |
| 2 | **Secret Scanning** (gitleaks, trufflehog) | ❌ Zero coverage |
| 3 | **SAST** (Semgrep, CodeQL, Bandit) | ❌ Only mentioned |
| 4 | **SCA** (Snyk, OWASP DC, Dependabot, Renovate) | ❌ Only mentioned |
| 5 | **Container Image Scanning** (Trivy deep dive) | ❌ Only mentioned |
| 6 | **DAST** (OWASP ZAP in pipeline) | ❌ Zero coverage |
| 7 | **SBOM** (Syft, CycloneDX) | ❌ Only mentioned |
| 8 | **SLSA + Cosign** (supply chain deep dive) | ❌ Only mentioned |
| 9 | **License compliance** (FOSSA) | ❌ Zero coverage |
| 10 | **Complete secure pipeline** (all stages) | ❌ Zero coverage |
| 11 | **Jenkins webhook triggers** | ❌ Zero coverage |
| 12 | **GitHub Environments + approvals** | ❌ Only mentioned |
| 13 | **GitHub Actions timeouts** | ❌ Only mentioned |
| 14 | **Unit / Integration / E2E testing in pipeline** | ❌ Zero coverage |
| 15 | **Smoke tests post-deploy** | ❌ Zero coverage |
| 16 | **Performance testing in pipeline** (k6, Gatling) | ❌ Zero coverage |
| 17 | **DB migrations in CI/CD** (Flyway, Liquibase) | ❌ Zero coverage |
| 18 | **Recreate / Shadow / A/B deployment strategies** | ❌ Zero coverage |
| 19 | **Slack notifications in pipeline** | ❌ Zero coverage |
| 20 | **Build/test reports** (JUnit XML, PR comments) | ❌ Zero coverage |
| 21 | **Kaniko/Buildah** (rootless Docker builds) | ❌ Only mentioned |
| 22 | **Image tagging strategy** (SHA, semver, latest) | ❌ Only mentioned |
| 23 | **Jenkins JCasC** (Configuration as Code) | ❌ Only mentioned |
| 24 | **OWASP Top 10** | ❌ Zero coverage |

---

## SECTION 1 — SHIFT LEFT SECURITY

### Q-CICD-01 — CI/CD Security | Conceptual | Intermediate

> Explain **Shift Left Security** and **DevSecOps** — what are they,
> why do they matter, and what is the cost model that justifies them?
>
> What is the **complete security pipeline** — what runs at each stage from
> developer's IDE all the way to production?

#### Key Points to Cover:

**What Shift Left means:**
```
Traditional (Shift RIGHT — security at the end):
  Code written → merged → built → deployed → security team reviews
  → Vulnerabilities found late = expensive, slow, embarrassing

Shift Left (security at every stage — move checks earlier):
  IDE        → real-time feedback (linters, IDE plugins)
  pre-commit → secret scan, basic lint (seconds, local)
  PR/CI      → SAST, SCA, IaC scan, secret scan (minutes)
  Build      → container image scan, SBOM, image signing
  Staging    → DAST, integration tests, compliance checks
  Production → runtime security monitoring

Cost to fix a vulnerability (IBM research):
  Developer IDE:  $1
  Pre-commit:     $5
  CI/CD (PR):     $100
  QA/Staging:     $1,000
  Production:     $10,000+
  After breach:   $1,000,000+

→ Finding in PR = 100x cheaper than finding in production
→ This ROI justifies every second of CI security scan time
```

**DevSecOps — security is everyone's responsibility:**
```
DevOps:    Dev + Ops collaboration
DevSecOps: Dev + Sec + Ops — security built INTO every step

Old model:  Security team = separate gatekeeper → slows releases
DevSecOps:  Security = automated checks in pipeline → fast releases

5 Principles:
1. Security as code — policies, rules, scans in version control
2. Automate security — no manual reviews for known vulnerability patterns
3. Fail fast — block at PR, not at production deploy
4. Shared responsibility — developers own security, not just security team
5. Continuous feedback — developer sees security issues immediately
```

**Complete security stages:**
```
Stage 1: Developer IDE (real-time, zero cost):
  → IDE plugins: SonarLint, AWS Toolkit, Snyk plugin
  → Shows vulnerabilities as you type (before even committing)

Stage 2: Pre-commit hooks (seconds, local):
  → gitleaks: detect secrets before they reach Git
  → terraform fmt / validate
  → Basic linting

Stage 3: PR / CI Pipeline (minutes, automated gate):
  → Secret scanning: trufflehog / gitleaks
  → SAST: Semgrep / CodeQL / SonarQube
  → SCA: Snyk / OWASP Dependency-Check
  → IaC scan: Checkov / tfsec
  → License scan: FOSSA
  → Container scan: Trivy (on built image)
  → SBOM generation: Syft
  → Block merge on CRITICAL findings

Stage 4: Build/Registry:
  → Image signing: Cosign
  → SBOM attestation attached to image
  → Image pushed only if signed

Stage 5: Deploy:
  → Admission webhook: verify image is signed before deploy
  → Policy enforcement: OPA / Kyverno

Stage 6: Runtime:
  → Runtime security: Falco (detect suspicious behavior)
  → Continuous CVE monitoring: ECR Enhanced Scanning
  → Periodic re-scan even without new deployments
```

> 💡 **Interview tip:** The best answer structure: start with the **cost model** (why it matters financially), then describe the **pipeline stages** (what runs where), then give a **specific example** from your experience. Interviewers are impressed when you mention that shifting left is not just about security — it's about **developer experience**. A developer who finds a CVE in their PR immediately (with a link to the fix) resolves it in 5 minutes. The same developer finding it a week later in a production incident response spends hours. Shift left = better developer experience AND better security.

---

## SECTION 2 — SECRET SCANNING

### Q-CICD-02 — CI/CD Security | Conceptual | Intermediate

> Explain **secret scanning** in CI/CD — what tools exist, how do you set up
> **pre-commit** and **CI-level** scanning? What do you do when a secret
> is already committed to git history?

#### Key Points to Cover:

**Why secret scanning is critical:**
```
Most commonly leaked secrets:
  → AWS Access Keys (AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY)
  → GitHub / GitLab tokens
  → Database passwords in connection strings
  → API keys (Stripe, Twilio, SendGrid)
  → Private SSH keys
  → JWT secrets / signing keys
  → Docker registry credentials
  → Slack webhook URLs

Why it happens:
  → "Temporary" hardcoding for testing — forgotten
  → .env files accidentally committed
  → Copy-paste from docs without cleanup
  → Debug code left in

Risk: Bots scan GitHub within SECONDS of a public push.
AWS keys leaked → charges thousands of dollars in minutes.
```

**gitleaks (most popular):**
```bash
# Install:
brew install gitleaks

# Scan current repo (uncommitted changes):
gitleaks detect --source .

# Scan entire git history (CRITICAL — finds past leaks):
gitleaks detect --source . --log-opts="--all"

# Pre-commit hook (.pre-commit-config.yaml):
repos:
- repo: https://github.com/gitleaks/gitleaks
  rev: v8.18.0
  hooks:
  - id: gitleaks

# GitHub Actions:
- name: Gitleaks Secret Scan
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
```

**trufflehog (deep history + verification):**
```bash
# Speciality: verifies if found secrets are ACTUALLY ACTIVE
# (not just pattern matches — actually calls the API)

brew install trufflesecurity/trufflehog/trufflehog

# Scan git history:
trufflehog git https://github.com/org/repo

# Only report verified (active) secrets:
trufflehog git file://. --only-verified

# GitHub Actions:
- name: TruffleHog Scan
  uses: trufflesecurity/trufflehog@main
  with:
    path: ./
    base: ${{ github.event.repository.default_branch }}
    head: HEAD
    extra_args: --only-verified
```

**When secret IS already committed:**
```bash
# Step 1: ROTATE THE SECRET IMMEDIATELY (before removing from history)
# Bots may have already scraped it. Rotation is more urgent than removal.
# AWS: delete + create new access key
# GitHub: Settings → Developer settings → Tokens → Delete

# Step 2: Remove from git history (BFG — fastest):
brew install bfg
# Create file with secrets to redact:
echo "AKIAIOSFODNN7EXAMPLE" > secrets.txt
bfg --replace-text secrets.txt myrepo.git
git push --force --all

# OR git filter-repo (modern):
pip install git-filter-repo
git filter-repo --replace-text expressions.txt

# Step 3: Force push ALL branches and tags
git push --force --all
git push --force --tags

# Step 4: All team members MUST re-clone (history rewritten)

# Step 5: Add to .gitignore:
echo ".env" >> .gitignore
echo "*.pem" >> .gitignore
echo "credentials.json" >> .gitignore
```

> 💡 **Interview tip:** The most important point about leaked secrets: **rotating the credential is more urgent than removing it from git history**. History removal takes coordination and time. The credential could be scraped by automated bots within minutes of a public push. Rotate first, clean second. Also: `--only-verified` in trufflehog is valuable — it only reports secrets that are actually active (makes an API call to verify), drastically reducing false positives vs pattern-based tools.

---

## SECTION 3 — SAST (STATIC APPLICATION SECURITY TESTING)

### Q-CICD-03 — CI/CD Security | Conceptual | Intermediate

> Explain **SAST** — what is it, what does it find, what are the main tools,
> and how do you integrate it into CI/CD without alert fatigue?
>
> What is the difference between **SAST**, **DAST**, and **SCA**?

#### Key Points to Cover:

**SAST vs DAST vs SCA:**
```
SAST (Static Application Security Testing):
  → Analyzes SOURCE CODE without executing it (white-box)
  → Fast — runs at PR time, no deployment needed
  → Finds: SQL injection, XSS, hardcoded creds, insecure crypto
  → High false positive rate (needs tuning)
  → Run: every PR

DAST (Dynamic Application Security Testing):
  → Tests RUNNING APPLICATION (black-box)
  → Requires deployed application
  → Finds: runtime auth bypass, actual injection exploits
  → Slow — needs live environment
  → Run: staging environment after deployment

SCA (Software Composition Analysis):
  → Scans THIRD-PARTY DEPENDENCIES for known CVEs
  → Fast, low false positives (CVE database lookup)
  → Finds: vulnerable libraries you use (not your code)
  → Run: every PR + continuously (new CVEs discovered daily)
```

**Semgrep (fast, rules-based, multi-language):**
```yaml
# GitHub Actions:
- name: Semgrep SAST
  uses: semgrep/semgrep-action@v1
  with:
    config: >-
      p/security-audit
      p/owasp-top-ten
      p/secrets
      p/python          # language-specific rules
  env:
    SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

# CLI:
semgrep --config=p/owasp-top-ten .
semgrep --config=p/security-audit --severity ERROR .  # only errors
```

**CodeQL (GitHub native, deep analysis):**
```yaml
# .github/workflows/codeql.yml
name: CodeQL
on: [push, pull_request]
jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
    - uses: actions/checkout@v4
    - uses: github/codeql-action/init@v3
      with:
        languages: python, javascript, java
    - uses: github/codeql-action/autobuild@v3
    - uses: github/codeql-action/analyze@v3
    # Results appear in GitHub Security tab
    # Can block PR merge via branch protection rules
```

**Bandit (Python specific):**
```bash
pip install bandit
bandit -r ./src -ll          # medium + high severity only
bandit -r . -f json -o bandit_report.json

# In GitHub Actions:
- name: Bandit Python Security
  run: |
    pip install bandit
    bandit -r src/ -ll --exit-zero    # --exit-zero = warn, don't fail
    bandit -r src/ -lll               # -lll = only HIGH → fail pipeline
```

**SonarQube (covered in Q547 — recap):**
```yaml
# Quality Gate blocks merge if:
# New bugs > 0, New vulnerabilities > 0, Coverage < 80%
- name: SonarQube Scan
  uses: SonarSource/sonarqube-scan-action@master
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
  with:
    args: -Dsonar.qualitygate.wait=true
```

**Avoiding alert fatigue:**
```
Start with HIGH/CRITICAL only:
  → Don't fail pipeline on LOW severity from day 1
  → Fix critical issues, then lower the threshold over time

Use baseline mode:
  → Scan existing code → save as baseline
  → Only FAIL on NEW issues introduced in this PR
  → Prevents legacy issues blocking all new development

Separate blocking from reporting:
  → CRITICAL: block PR merge (hard fail)
  → HIGH: warning comment on PR (soft fail)
  → MEDIUM/LOW: report to dashboard only

Exit codes:
  semgrep --error    # exit 1 on any finding (strict)
  semgrep --strict   # exit 1 on errors/warnings
  bandit -lll        # only HIGH = fail
  bandit -ll         # MEDIUM + HIGH = fail
```

> 💡 **Interview tip:** The most practical SAST insight: **start with `--severity CRITICAL` only** when first introducing SAST to a team. A fresh codebase scan with all severities produces hundreds of findings — developers ignore everything. Start strict (only CRITICAL), fix those, then add HIGH, then MEDIUM. This "ratchet" approach builds security culture progressively without creating immediate backlash. Also: **SAST is bad at context** — it sees code patterns, not business logic. A hardcoded value that looks like a password might be a test fixture. This is why false positive management is critical.

---

## SECTION 4 — SCA (SOFTWARE COMPOSITION ANALYSIS)

### Q-CICD-04 — CI/CD Security | Conceptual | Intermediate

> Explain **SCA** — what is it, why is it critical, and what are the main tools?
> How does **Dependabot** work? What is your vulnerability management workflow?

#### Key Points to Cover:

**Why SCA is critical:**
```
Modern apps are 80%+ third-party code:
  Node.js app: 20 direct deps → 500+ transitive deps
  Java app:    50 direct deps → 200+ Maven transitive deps

Any of these can have a known CVE.
Log4Shell (Log4j CVE-2021-44228):
  → Millions of apps "didn't use Log4j"
  → But used it TRANSITIVELY through Elasticsearch, etc.
  → SCA would have found it immediately

SCA scans the ENTIRE dependency tree, not just direct deps.
```

**Snyk (most popular, commercial):**
```bash
# Install:
npm install -g snyk

# Test:
snyk test                               # uses package.json/pom.xml etc.
snyk test --severity-threshold=high    # only HIGH/CRITICAL

# Fix (auto-upgrade):
snyk fix

# GitHub Actions:
- name: Snyk SCA
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
    # Fails on HIGH or CRITICAL CVEs

# Snyk also: container scanning, IaC, SAST (all-in-one option)
```

**OWASP Dependency-Check (open source):**
```yaml
- name: OWASP Dependency Check
  uses: dependency-check/Dependency-Check_Action@main
  with:
    project: 'my-app'
    path: '.'
    format: 'HTML'
    args: --failOnCVSS 7     # fail if CVSS score >= 7 (HIGH)
```

**Dependabot (GitHub native — automated PRs):**
```yaml
# .github/dependabot.yml — place in repo root
version: 2
updates:
  # npm:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"        # daily | weekly | monthly
    open-pull-requests-limit: 10
    labels: ["dependencies", "security"]
    ignore:
      - dependency-name: "lodash"
        versions: ["4.x"]       # ignore specific versions

  # Docker base images:
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"

  # GitHub Actions versions:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"

  # Python pip:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"

# How Dependabot works:
# → Scans your dependencies weekly
# → Opens PR: "Bump lodash from 4.17.20 to 4.17.21"
# → PR runs full CI pipeline (your tests)
# → If tests pass → auto-merge (if configured)
# → If security alert → opened immediately (not waiting for schedule)
```

**Renovate Bot (more configurable alternative):**
```json
// renovate.json
{
  "extends": ["config:base"],
  "schedule": ["before 6am on Monday"],
  "automerge": true,
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true         // auto-merge patch updates
    },
    {
      "matchUpdateTypes": ["major"],
      "automerge": false,       // require review for major versions
      "labels": ["major-update"]
    }
  ]
}
```

**Vulnerability management workflow:**
```
1. Scan on every PR → block CRITICAL
2. Monitor continuously → new CVEs trigger immediate alerts
3. Triage:
   → Is this CVE in a code path we actually use?
   → Some CVEs are in features we never call
   → CVSS score + exploitability + your usage context
4. Fix: upgrade dependency to patched version
5. If no fix available:
   → Suppress with expiry date + justification
   → Monitor for patch release
6. Track SLA: CRITICAL ≤ 24h, HIGH ≤ 7 days, MEDIUM ≤ 30 days
```

> 💡 **Interview tip:** The key SCA concept: **transitive dependencies**. Your direct dependencies are 20 packages. Each of those has 25 dependencies. You have 500+ packages in total. Log4Shell affected Java applications that never heard of Log4j because it was a transitive dependency of Elasticsearch, which they DID use. SCA tools scan the full dependency tree. For Dependabot: **grouping security updates** with `groups` in dependabot.yml reduces PR noise — instead of 20 separate PRs for 20 minor updates, one grouped PR with all of them.

---

## SECTION 5 — CONTAINER IMAGE SCANNING

### Q-CICD-05 — CI/CD Security | Conceptual | Intermediate

> Explain **container image scanning** with **Trivy** in depth. How do you
> integrate it into CI/CD? What is your severity policy for blocking deploys?

#### Key Points to Cover:

**How image scanning works:**
```
Container image = base OS layer + app layers

Scanning checks:
  1. OS packages: apt/yum packages in base image
  2. Language packages: npm/pip/maven in app layers
  3. Misconfigurations: running as root, world-writable dirs
  4. Secrets: accidentally included credentials

CVE databases used:
  → NVD (National Vulnerability Database)
  → OS-specific: Ubuntu USN, RHEL RHSA, Debian DSA
  → GitHub Advisory Database
  → OSV (Open Source Vulnerabilities)
```

**Trivy (most popular, open source):**
```bash
# Scan image:
trivy image nginx:latest
trivy image myapp:v1.2.3

# Specific severity only:
trivy image --severity CRITICAL,HIGH myapp:latest

# Fail CI on CRITICAL:
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# Output formats:
trivy image --format sarif --output results.sarif myapp:latest  # GitHub Security tab
trivy image --format json --output results.json myapp:latest
trivy image --format table myapp:latest                         # human readable

# Scan filesystem (before building image):
trivy fs --severity CRITICAL,HIGH .

# Scan IaC files (Dockerfile, K8s YAML, Terraform):
trivy config .

# Scan git repo:
trivy repo https://github.com/myorg/myrepo
```

**Trivy in GitHub Actions:**
```yaml
jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Build Docker image
      run: docker build -t myapp:${{ github.sha }} .

    - name: Trivy vulnerability scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: myapp:${{ github.sha }}
        format: sarif
        output: trivy-results.sarif
        severity: CRITICAL,HIGH
        exit-code: '1'           # fail pipeline on CRITICAL or HIGH

    - name: Upload to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v3
      if: always()               # upload even if scan failed
      with:
        sarif_file: trivy-results.sarif
```

**Severity policy:**
```
CRITICAL → Always block — do not deploy, fix immediately
HIGH     → Block production deploys, warn in dev/staging
MEDIUM   → Warn only, fix within 30 days
LOW      → Track, fix in next release cycle

Handling "no fix available":
  Some CVEs have no upstream fix yet.
  → Add to .trivyignore file with reason + expiry:
  CVE-2021-12345  # No fix available, affects unused code path. Review 2024-06-01

  # .trivyignore:
  CVE-2021-12345
  CVE-2022-67890

  # Run with ignore file:
  trivy image --ignorefile .trivyignore myapp:latest
```

**Minimal base images (reduce CVE surface):**
```dockerfile
# Many CVEs = bloated base image with unnecessary packages

# BAD: ubuntu:22.04 (350MB+, hundreds of packages)
FROM ubuntu:22.04

# BETTER: alpine (5MB, minimal packages)
FROM alpine:3.19

# BEST: distroless (no shell, no package manager)
FROM gcr.io/distroless/python3:nonroot
# Contains ONLY the runtime — 90%+ fewer CVEs
# No shell = attacker can't exec into container if compromised

# Multi-stage to get distroless final image:
FROM python:3.12 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt
COPY src/ .

FROM gcr.io/distroless/python3:nonroot
COPY --from=builder /root/.local /root/.local
COPY --from=builder /app /app
CMD ["/app/main.py"]
```

> 💡 **Interview tip:** The most impactful image scanning advice: **scan the base image choice first**. `FROM ubuntu:20.04` might have 50 CRITICAL CVEs before you add one line of your application code. `FROM alpine:3.19` has near zero. `FROM gcr.io/distroless/python3` has essentially zero (no packages at all — just the runtime). Switching base image is often faster than patching individual CVEs. Also mention: **Trivy is not just for images** — `trivy config .` scans Dockerfiles, K8s manifests, and Terraform files for misconfigurations, making it an all-in-one security scanner.

---

## SECTION 6 — DAST (DYNAMIC APPLICATION SECURITY TESTING)

### Q-CICD-06 — CI/CD Security | Conceptual | Intermediate

> Explain **DAST** — what is it, how does it differ from SAST, and how do
> you integrate **OWASP ZAP** into a CI/CD pipeline?

#### Key Points to Cover:

**What DAST does:**
```
DAST = tests your RUNNING application from outside (black-box)
     = sends actual HTTP requests to find vulnerabilities

What DAST finds that SAST cannot:
  → Authentication bypass (needs running auth system)
  → Session management flaws (needs running sessions)
  → Server configuration issues (exposed headers, etc.)
  → Actual injection vulnerabilities (tests real responses)
  → Business logic flaws visible in HTTP traffic

DAST requirements:
  → Application must be deployed and running
  → Test/staging environment (NEVER run against production)
  → Authentication credentials for logged-in scans

Typical CI/CD placement:
  Deploy to staging → run DAST → check results → approve/block prod deploy
```

**OWASP ZAP (Zed Attack Proxy) — free, most popular:**
```yaml
# GitHub Actions — ZAP baseline scan:
- name: Deploy to staging
  run: ./deploy-staging.sh

- name: OWASP ZAP Baseline Scan
  uses: zaproxy/action-baseline@v0.12.0
  with:
    target: 'https://staging.myapp.com'
    rules_file_name: '.zap/rules.tsv'   # custom rule config
    cmd_options: '-a'                    # -a = adjust for ajax apps

# ZAP scan types:
# Baseline: passive scan only (no active attacks) — fast, safe
# Full scan: active attacks (slower, more thorough)
# API scan: for REST/GraphQL APIs (uses OpenAPI spec)

- name: OWASP ZAP API Scan
  uses: zaproxy/action-api-scan@v0.7.0
  with:
    target: 'https://staging.myapp.com/api/openapi.json'
    format: openapi

# ZAP Full Scan (more thorough):
- name: OWASP ZAP Full Scan
  uses: zaproxy/action-full-scan@v0.10.0
  with:
    target: 'https://staging.myapp.com'
    fail_action: true    # fail job if alerts found above threshold
```

**ZAP results handling:**
```yaml
- name: ZAP Scan
  uses: zaproxy/action-baseline@v0.12.0
  with:
    target: 'https://staging.myapp.com'
  continue-on-error: true    # don't fail immediately

- name: Upload ZAP report
  uses: actions/upload-artifact@v4
  with:
    name: zap-report
    path: report_html.html

# ZAP creates:
# report_html.html  = human-readable report
# report_json.json  = machine-readable
# report_md.md      = markdown (for PR comments)
```

> 💡 **Interview tip:** DAST is the most commonly skipped security stage because it requires a running environment. The pragmatic approach: **ZAP baseline scan** (passive, no active attacks) runs in 5-10 minutes against staging after every deployment. This catches many common issues (missing security headers, information disclosure) without the risk of active attack mode against your test data. Run **ZAP full scan** only in dedicated security testing sprints, not on every pipeline run — it's too slow and may interfere with test data.

---

## SECTION 7 — SBOM (SOFTWARE BILL OF MATERIALS)

### Q-CICD-07 — CI/CD Security | Conceptual | Intermediate

> Explain **SBOM** — what is it, why is it required (especially post-Executive
> Order 14028), and how do you generate and use one with **Syft**?

#### Key Points to Cover:

**What SBOM is:**
```
SBOM = complete inventory of all software components in your application
     = "ingredients list" for your software

Contains:
  → Every library and dependency (direct + transitive)
  → Versions of each component
  → License of each component
  → Where it came from (source, hash)
  → Known vulnerabilities (CVEs) in each component

Why it matters:
  → US Executive Order 14028 (2021): federal software must have SBOM
  → Enables: rapid response to new CVEs
    "Is Log4j anywhere in our estate?" → query SBOM → answer in seconds
    Without SBOM: scan every system manually = days

SBOM formats:
  CycloneDX: XML/JSON, OWASP standard, most tooling support
  SPDX:      Linux Foundation standard, used by GitHub
  SWID:      older standard
```

**Syft — generate SBOM:**
```bash
# Install:
brew install syft

# Generate SBOM for container image:
syft myapp:v1.2.3                                    # table output
syft myapp:v1.2.3 -o cyclonedx-json > sbom.json     # CycloneDX JSON
syft myapp:v1.2.3 -o spdx-json > sbom.spdx.json     # SPDX JSON
syft myapp:v1.2.3 -o syft-json > sbom.syft.json      # Syft native format

# Generate for filesystem:
syft dir:./src -o cyclonedx-json > sbom.json

# GitHub Actions:
- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    image: myapp:${{ github.sha }}
    format: cyclonedx-json
    output-file: sbom.cyclonedx.json

- name: Upload SBOM as artifact
  uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: sbom.cyclonedx.json
```

**Grype — scan SBOM for CVEs:**
```bash
# Grype can scan an SBOM (not just images):
grype sbom:./sbom.cyclonedx.json           # scan from SBOM file
grype sbom:./sbom.cyclonedx.json --fail-on critical

# Workflow: generate SBOM → scan SBOM for CVEs:
syft myapp:v1.2.3 -o cyclonedx-json > sbom.json
grype sbom:./sbom.json --fail-on critical
```

**Attaching SBOM to image (with Cosign):**
```bash
# Sign the SBOM and attach to image registry:
cosign attach sbom --sbom sbom.cyclonedx.json myapp:v1.2.3

# Later: retrieve SBOM for audit:
cosign download sbom myapp:v1.2.3
```

> 💡 **Interview tip:** The most compelling SBOM use case for interviews: **zero-day response time**. When Log4Shell was disclosed, organizations WITH SBOMs answered "are we affected?" in hours. Organizations WITHOUT SBOMs took days or weeks of manual scanning. An SBOM is essentially an asset registry for your software — query it like a database: `"find all containers using log4j version < 2.16"`. This is why the US government mandated SBOMs: not to prevent vulnerabilities, but to enable rapid response when they're discovered.

---

## SECTION 8 — COSIGN + SLSA (SUPPLY CHAIN SECURITY)

### Q-CICD-08 — CI/CD Security | Conceptual | Advanced

> Explain **image signing with Cosign** — how does it work and why does it matter?
> What is the **SLSA framework** (Supply Chain Levels for Software Artifacts)?

#### Key Points to Cover:

**Cosign — container image signing:**
```bash
# Problem: anyone can push to your registry
# "myapp:v1.2.3" pulled by K8s — was it really built by your CI/CD?
# Supply chain attack: attacker compromises registry or CI/CD, pushes malicious image

# Solution: sign images with private key, verify before deploy

# Install:
brew install cosign

# Generate key pair:
cosign generate-key-pair
# Creates: cosign.key (private) + cosign.pub (public)

# Sign image (after push to ECR):
cosign sign --key cosign.key myregistry.com/myapp:v1.2.3

# Verify before deploy:
cosign verify --key cosign.pub myregistry.com/myapp:v1.2.3

# Keyless signing (Sigstore — uses OIDC, no keys to manage):
# GitHub Actions with keyless:
- name: Sign image with Cosign (keyless)
  uses: sigstore/cosign-installer@v3

- name: Sign the Docker image
  run: |
    cosign sign --yes myregistry.com/myapp:${{ github.sha }}
  env:
    COSIGN_EXPERIMENTAL: 1    # keyless mode — uses GitHub OIDC token
```

**Enforce signed images in Kubernetes:**
```yaml
# Admission controller (Kyverno policy):
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  rules:
  - name: check-image-signature
    match:
      resources:
        kinds: [Pod]
    verifyImages:
    - image: "myregistry.com/*"
      key: |-
        -----BEGIN PUBLIC KEY-----
        MFkwEwYHKoZIzj0CAQY...
        -----END PUBLIC KEY-----
# Any pod using unsigned image from your registry → REJECTED
```

**SLSA Framework (Supply Chain Levels for Software Artifacts):**
```
SLSA = framework for software supply chain security
       Defines 4 levels of security guarantees

Level 1: Build provenance exists
  → Build process generates provenance (metadata about how artifact was built)
  → Who built it, when, from what source

Level 2: Hosted build, signed provenance
  → Build runs on hosted service (GitHub Actions, not developer laptop)
  → Provenance is signed by build system
  → Can verify: this artifact was built by GitHub Actions

Level 3: Isolated build, non-forgeable provenance
  → Build is isolated (ephemeral environment)
  → Provenance cannot be forged even by maintainers
  → Hardened build platform

Level 4: Two-person review, parameterless build
  → All code changes reviewed by 2+ people
  → Build is fully reproducible

In practice for most teams: target SLSA Level 2
  → Use GitHub Actions (hosted build) ✅
  → Generate provenance → sign with cosign ✅
  → Store provenance alongside image ✅

Generate provenance in GitHub Actions:
- name: Generate provenance
  uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v1
  with:
    image: myregistry.com/myapp
    digest: ${{ steps.build.outputs.digest }}
```

> 💡 **Interview tip:** The simplest way to explain Cosign to an interviewer: **"It's GPG signing for container images."** You sign the image hash with your private key after building it. Your Kubernetes admission controller verifies the signature with your public key before allowing deployment. If an attacker pushes a malicious image to your registry, it won't have your signature → Kyverno rejects it → the pod never starts. The keyless signing mode (Sigstore) is the modern approach — it uses GitHub's OIDC token so there are no long-lived private keys to manage or leak.

---

## SECTION 9 — LICENSE COMPLIANCE

### Q-CICD-09 — CI/CD Security | Conceptual | Intermediate

> Explain **license compliance scanning** — why does it matter and how do
> you integrate **FOSSA** or similar tools into CI/CD?

#### Key Points to Cover:

**Why licenses matter:**
```
Every open-source dependency has a license.
Using a library incorrectly can create legal liability.

License types (restrictiveness):
  MIT / Apache 2.0 / BSD:
    → Permissive — use freely, include in commercial software
    → Just include attribution/copyright notice

  LGPL (Lesser GPL):
    → Use in commercial software OK
    → But if you MODIFY the library, release modifications as LGPL

  GPL v2/v3:
    → "Copyleft" — if you include GPL code in your product
    → Your ENTIRE product may need to be GPL (open source)
    → Risk: including GPL library in commercial SaaS

  AGPL:
    → Like GPL but triggered by NETWORK USE
    → If users access your AGPL software over network → your code must be open
    → MongoDB, Elastic use AGPL to prevent cloud providers from using freely

  Commercial / Proprietary:
    → Cannot use without purchasing license
    → Must track and pay for usage

Risk examples:
  → Using a GPL library in your commercial product → legal obligation to open source
  → Using an AGPL library in your SaaS → must release your SaaS code
  → These are accidental — developers don't check licenses when doing npm install
```

**FOSSA (most popular license scanner):**
```bash
# Install:
brew install fossa-cli

# Scan project:
fossa analyze
fossa test    # fails if unapproved licenses found

# GitHub Actions:
- name: FOSSA License Scan
  uses: fossas/fossa-action@main
  with:
    api-key: ${{ secrets.FOSSA_API_KEY }}

# Configure allowed licenses (.fossa.yml):
version: 1
analyze:
  modules:
    - name: my-app
      type: npm
      target: .
project:
  policy: "Company Policy"    # policy defined in FOSSA UI
# Policy defines: allowed licenses, forbidden licenses, needs-review
```

**License policy in CI/CD:**
```yaml
# Fail pipeline on forbidden licenses:
- name: Check licenses
  run: |
    pip install licensecheck
    licensecheck --zero    # exit 0 only if all licenses are compatible
    # OR
    license-checker --production --failOn "GPL;AGPL"
    # Fail if any production dependency uses GPL or AGPL

# npm license-checker:
npx license-checker --production --failOn 'GPL'
npx license-checker --summary    # quick overview of all license types
```

> 💡 **Interview tip:** License compliance is the **most underrated CI/CD security topic** — companies get caught by it in M&A due diligence. When a company gets acquired, the acquirer's legal team scans all code for license issues. Finding GPL code in a commercial product right before a $100M acquisition is a serious problem. Automating license checks in CI/CD catches these issues when a developer accidentally adds a GPL library — immediately, with a clear fix (use a different library with a permissive license).

---

## SECTION 10 — COMPLETE SECURE CI/CD PIPELINE

### Q-CICD-10 — CI/CD Security | System Design | Advanced

> Design a **complete secure CI/CD pipeline** for a containerized microservice
> — from code commit to production deployment. Include every security gate,
> explain where each tool runs, and what triggers a pipeline failure.

#### Key Points to Cover:

```yaml
# Complete secure pipeline — GitHub Actions
name: Secure CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # ─────────────────────────────────────────────────────
  # STAGE 1: CODE QUALITY & SECURITY (runs on every PR)
  # ─────────────────────────────────────────────────────
  security-scan:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0    # full history for secret scanning

    # 1. Secret scanning (gitleaks):
    - name: Secret Scan
      uses: gitleaks/gitleaks-action@v2
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
      # BLOCKS: any hardcoded secrets in code

    # 2. SAST (Semgrep):
    - name: SAST - Semgrep
      uses: semgrep/semgrep-action@v1
      with:
        config: p/security-audit p/owasp-top-ten
      # BLOCKS: critical code vulnerabilities

    # 3. SCA (Snyk dependency scan):
    - name: SCA - Snyk
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=critical
      # BLOCKS: critical CVEs in dependencies

    # 4. IaC security (Checkov):
    - name: IaC Scan - Checkov
      uses: bridgecrewio/checkov-action@master
      with:
        directory: terraform/
        framework: terraform
        soft_fail: false
        check: CKV_AWS_20,CKV_AWS_57    # specific checks or use all
      # BLOCKS: critical Terraform misconfigurations

    # 5. License compliance (FOSSA):
    - name: License Compliance
      uses: fossas/fossa-action@main
      with:
        api-key: ${{ secrets.FOSSA_API_KEY }}
      # BLOCKS: GPL/AGPL license violations

  # ─────────────────────────────────────────────────────
  # STAGE 2: BUILD + IMAGE SECURITY
  # ─────────────────────────────────────────────────────
  build-and-scan:
    runs-on: ubuntu-latest
    needs: security-scan
    permissions:
      id-token: write
      contents: read
      security-events: write
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
    - uses: actions/checkout@v4

    # AWS OIDC auth (no stored keys):
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
        aws-region: us-east-1

    - name: Login to ECR
      uses: aws-actions/amazon-ecr-login@v2

    # Build with layer caching:
    - name: Build Docker image
      id: build
      run: |
        IMAGE="${{ secrets.ECR_REGISTRY }}/myapp:${{ github.sha }}"
        docker build \
          --cache-from type=gha \
          --cache-to type=gha,mode=max \
          -t "$IMAGE" .
        echo "digest=$(docker inspect --format='{{index .RepoDigests 0}}' $IMAGE)" >> $GITHUB_OUTPUT

    # 6. Container image scan (Trivy):
    - name: Trivy Image Scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ secrets.ECR_REGISTRY }}/myapp:${{ github.sha }}
        format: sarif
        output: trivy-results.sarif
        severity: CRITICAL
        exit-code: '1'
      # BLOCKS: critical CVEs in container image

    - name: Upload scan results
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: trivy-results.sarif

    # 7. SBOM generation:
    - name: Generate SBOM
      uses: anchore/sbom-action@v0
      with:
        image: ${{ secrets.ECR_REGISTRY }}/myapp:${{ github.sha }}
        format: cyclonedx-json
        output-file: sbom.json

    # Push image:
    - name: Push to ECR
      run: docker push ${{ secrets.ECR_REGISTRY }}/myapp:${{ github.sha }}

    # 8. Sign image (Cosign keyless):
    - name: Install Cosign
      uses: sigstore/cosign-installer@v3

    - name: Sign image
      run: cosign sign --yes ${{ secrets.ECR_REGISTRY }}/myapp:${{ github.sha }}
      env:
        COSIGN_EXPERIMENTAL: 1
      # IMAGE IS NOW: scanned + signed + has SBOM attached

  # ─────────────────────────────────────────────────────
  # STAGE 3: DEPLOY TO STAGING
  # ─────────────────────────────────────────────────────
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build-and-scan
    if: github.event_name == 'push'
    environment: staging
    steps:
    - name: Deploy to staging
      run: |
        kubectl set image deployment/myapp \
          myapp=${{ secrets.ECR_REGISTRY }}/myapp:${{ github.sha }}
        kubectl rollout status deployment/myapp --timeout=5m

    # 9. DAST (OWASP ZAP baseline):
    - name: DAST - OWASP ZAP
      uses: zaproxy/action-baseline@v0.12.0
      with:
        target: 'https://staging.myapp.com'
      # BLOCKS: critical runtime vulnerabilities found

    # 10. Smoke tests post-deploy:
    - name: Smoke tests
      run: |
        curl -f https://staging.myapp.com/health
        curl -f https://staging.myapp.com/api/status

  # ─────────────────────────────────────────────────────
  # STAGE 4: PRODUCTION (manual approval required)
  # ─────────────────────────────────────────────────────
  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production    # requires manual approval in GitHub UI
    steps:
    - name: Deploy to production
      run: |
        kubectl set image deployment/myapp \
          myapp=${{ secrets.ECR_REGISTRY }}/myapp:${{ github.sha }}
        kubectl rollout status deployment/myapp --timeout=10m

    - name: Notify success
      uses: slackapi/slack-github-action@v1
      with:
        payload: '{"text":"✅ ${{ github.sha }} deployed to production"}'
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

**Summary of security gates:**
```
Gate 1: Secret scan     → no secrets in code
Gate 2: SAST            → no critical code vulnerabilities
Gate 3: SCA             → no critical CVE in dependencies
Gate 4: IaC scan        → no critical infra misconfigurations
Gate 5: License check   → no GPL/AGPL violations
Gate 6: Image scan      → no critical CVE in container
Gate 7: DAST            → no critical runtime vulnerabilities
Gate 8: Image signed    → only signed images deployed (K8s admission)
Gate 9: Smoke tests     → app works after deploy
Gate 10: Manual approval → human reviewed before production
```

> 💡 **Interview tip:** The **sequence of gates matters** — put fastest checks first (secret scan = seconds), slowest last (DAST = minutes). Developers get fastest feedback on most common failures. Also: **never run DAST against production** — always staging only. A ZAP full scan actively attacks your application and can corrupt data, create test accounts, trigger rate limiting. Run ZAP baseline (passive) on every deploy to staging, full scan only in dedicated security sprints.

---

## SECTION 11 — JENKINS SPECIFICS

### Q-CICD-11 — Jenkins | Conceptual | Intermediate

> Explain **Jenkins webhook triggers** — how do you configure Jenkins to
> automatically trigger builds when code is pushed to GitHub?
>
> Also explain **Jenkins JCasC (Configuration as Code)** — what problem
> does it solve and how does it work?

#### Key Points to Cover:

**Jenkins webhook triggers:**
```
Without webhook: Jenkins polls GitHub every X minutes (wasteful, slow)
With webhook:    GitHub pushes event to Jenkins immediately on push (fast, efficient)

Setup steps:
1. Install GitHub plugin in Jenkins
2. In Jenkins job/pipeline:
   Build Triggers → "GitHub hook trigger for GITScm polling"
   OR in Jenkinsfile:
   triggers {
     githubPush()    // trigger on any push
   }

3. In GitHub repository:
   Settings → Webhooks → Add webhook
   Payload URL: http://jenkins.company.com/github-webhook/
   Content type: application/json
   Events: Push, Pull Request (select what triggers builds)
   Secret: use a secret token for verification

4. Jenkins verifies webhook signature:
   Configure → GitHub Server → Manage hooks (auto-creates webhook)

Webhook security:
  → Jenkins validates HMAC-SHA256 signature of payload
  → Prevents fake webhook events from unauthorized sources
  → Configure secret token both in GitHub and Jenkins

Multibranch pipeline triggers:
  triggers {
    githubPush()           // any branch push
  }
  // OR branch-specific filter:
  when { branch 'main' }
```

**Jenkins JCasC (Jenkins Configuration as Code):**
```yaml
# Problem: Jenkins config lives in XML files + UI clicks
#          → Not version controlled
#          → Can't reproduce after failure
#          → Drift between instances
#          → Onboarding = hours of UI clicking

# JCasC solution: define entire Jenkins config as YAML
# Version controlled, reproducible, reviewable

# jenkins.yaml (JCasC config file):
jenkins:
  systemMessage: "Jenkins managed by JCasC"
  numExecutors: 0     # controller has no executors (agents only)
  mode: EXCLUSIVE

  securityRealm:
    ldap:
      configurations:
        - server: ldap://company-ldap.com
          rootDN: "dc=company,dc=com"

  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: admin
            permissions:
              - Overall/Administer
            assignments:
              - platform-team

  clouds:
    - kubernetes:
        name: kubernetes
        serverUrl: https://kubernetes.default
        namespace: jenkins
        templates:
          - name: jnlp-agent
            containers:
              - name: jnlp
                image: jenkins/inbound-agent:latest

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              id: "github-credentials"
              username: "jenkins-bot"
              password: "${GITHUB_TOKEN}"   # from env var

tool:
  git:
    installations:
      - name: Default
        home: git

unclassified:
  location:
    url: https://jenkins.company.com/
    adminAddress: devops@company.com
  slackNotifier:
    teamDomain: company
    tokenCredentialId: slack-token
```

```bash
# Apply JCasC config:
# Jenkins reads jenkins.yaml from CASC_JENKINS_CONFIG env var
# Or: Manage Jenkins → Configuration as Code → Apply new configuration

# Docker run with JCasC:
docker run -p 8080:8080 \
  -v $(pwd)/jenkins.yaml:/var/jenkins_home/casc_configs/jenkins.yaml \
  -e CASC_JENKINS_CONFIG=/var/jenkins_home/casc_configs/ \
  jenkins/jenkins:lts
```

> 💡 **Interview tip:** JCasC + Jenkins in Kubernetes (using Helm) is the **modern production Jenkins setup**. The Helm chart deploys Jenkins with JCasC config from a ConfigMap. If Jenkins pod dies, Kubernetes recreates it and JCasC re-applies configuration automatically — no manual reconfiguration. This changes Jenkins from a "pet" (manually maintained, never recreate) to "cattle" (destroy and recreate anytime). Pair this with the Kubernetes plugin for ephemeral agents — Jenkins has zero state that isn't in Git or JCasC config.

---

## SECTION 12 — GITHUB ACTIONS REMAINING GAPS

### Q-CICD-12 — GitHub Actions | Conceptual | Intermediate

> Explain **GitHub Environments with protection rules** — what are they and
> how do you use them for production deployment approval?
>
> Also explain **step/job outputs**, **timeouts**, and **Slack notifications**
> in GitHub Actions workflows.

#### Key Points to Cover:

**GitHub Environments + protection rules:**
```yaml
# GitHub Environments = named deployment targets (staging, production)
# Can configure: required reviewers, wait timers, branch restrictions

# Setup in GitHub UI:
# Settings → Environments → New environment → "production"
# → Required reviewers: [platform-team, tech-lead]
# → Wait timer: 10 minutes (cooling off period)
# → Deployment branches: main only

# Use in workflow:
jobs:
  deploy-production:
    environment: production    # triggers approval requirement
    # Pipeline PAUSES here until required reviewers approve
    steps:
    - name: Deploy
      run: ./deploy-prod.sh

# Environment secrets (different from repo secrets):
# production environment has: PROD_DB_URL, PROD_API_KEY
# staging environment has:    STAGING_DB_URL, STAGING_API_KEY
# Same workflow can use different secrets per environment
```

**Step outputs and job outputs:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.set-tag.outputs.tag }}    # expose step output as job output
      version: ${{ steps.get-version.outputs.version }}

    steps:
    - name: Set image tag
      id: set-tag
      run: |
        TAG="sha-${GITHUB_SHA::8}"
        echo "tag=$TAG" >> $GITHUB_OUTPUT    # set step output

    - name: Get app version
      id: get-version
      run: |
        VERSION=$(cat version.txt)
        echo "version=$VERSION" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - name: Use build outputs
      run: |
        echo "Deploying image tag: ${{ needs.build.outputs.image-tag }}"
        echo "App version: ${{ needs.build.outputs.version }}"
        kubectl set image deployment/app \
          app=myregistry/myapp:${{ needs.build.outputs.image-tag }}
```

**Timeouts:**
```yaml
# Job level timeout:
jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 30      # kill job if running > 30 minutes

    steps:
    # Step level timeout:
    - name: Long running step
      timeout-minutes: 10    # kill this specific step after 10 min
      run: ./long-script.sh

    # Without timeout: hung step runs forever, wastes GitHub Actions minutes
    # Always set timeout on: deploy steps, test runs, curl commands
```

**Slack notifications:**
```yaml
# Method 1: Slack official action:
- name: Notify Slack on success
  if: success()
  uses: slackapi/slack-github-action@v1.26.0
  with:
    payload: |
      {
        "text": "✅ Deployment successful!",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "✅ *${{ github.repository }}* deployed `${{ github.sha }}` to production\nTriggered by: ${{ github.actor }}"
            }
          }
        ]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK

# Method 2: curl directly:
- name: Notify Slack on failure
  if: failure()
  run: |
    curl -X POST \
      -H 'Content-type: application/json' \
      --data "{\"text\":\"❌ Pipeline FAILED: ${{ github.repository }} - ${{ github.workflow }} - ${{ github.run_url }}\"}" \
      ${{ secrets.SLACK_WEBHOOK_URL }}

# Common conditions:
if: success()          # only on success
if: failure()          # only on failure
if: always()           # always (even if cancelled)
if: cancelled()        # only if cancelled
```

**Build/test reports as PR comments:**
```yaml
# Publish test results as PR comment:
- name: Run tests
  run: npm test -- --reporter=junit --reporter-options="output=test-results.xml"

- name: Publish test results
  uses: EnricoMi/publish-unit-test-result-action@v2
  if: always()
  with:
    junit_files: "test-results.xml"
    # Creates PR comment with test summary:
    # ✅ 150 tests passed, ❌ 2 failed

# Code coverage comment:
- name: Code coverage comment
  uses: irongut/CodeCoverageSummary@v1.3.0
  with:
    filename: coverage.xml
    badge: true
    format: markdown
    output: both
    fail_below_min: true
    thresholds: '60 80'   # warn below 60%, fail below 80%

- name: Add coverage comment to PR
  uses: marocchino/sticky-pull-request-comment@v2
  with:
    recreate: true
    path: code-coverage-results.md
```

> 💡 **Interview tip:** GitHub Environments with **required reviewers** is the production-safe deployment pattern — it creates a mandatory approval gate before any production deployment. Without it, a merge to main can automatically deploy to production with zero human approval. The environment protection rules also support **branch restrictions** (only `main` can deploy to production) and **wait timers** (mandatory 10-minute delay between staging and production deploys, giving time to observe staging behavior).

---

## SECTION 13 — TESTING IN CI/CD

### Q-CICD-13 — CI/CD | Conceptual | Intermediate

> Explain the **testing pyramid in CI/CD** — what types of tests run at each
> pipeline stage? What are **smoke tests**, **contract tests**, and
> **performance tests** in a pipeline context?

#### Key Points to Cover:

**Testing pyramid in CI/CD:**
```
        /\
       /E2E\           ← few, slow, expensive (staging)
      /------\
     / Integ  \        ← some, medium speed (CI)
    /----------\
   / Unit Tests \      ← many, fast, cheap (every commit)
  /______________\

Unit tests (every commit, < 1 minute):
  → Test individual functions/classes in isolation
  → Mock external dependencies
  → Fast: hundreds in seconds
  → Block merge if any fail
  → Example: test calculateDiscount() logic

Integration tests (every PR, 2-5 minutes):
  → Test multiple components together
  → Often use real database (Testcontainers)
  → Test: API → service → database interaction
  → Slower: tens of tests in minutes

E2E tests (on staging, 10-30 minutes):
  → Test complete user flows (Selenium, Playwright, Cypress)
  → Real browser, real backend
  → Slow, brittle, expensive
  → Run AFTER deploy to staging, not in PR
```

**Unit + Integration in GitHub Actions:**
```yaml
jobs:
  unit-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Run unit tests
      run: |
        npm ci
        npm test -- --coverage
    - name: Upload coverage
      uses: codecov/codecov-action@v4
      with:
        fail_ci_if_error: true
        threshold: 80    # fail if coverage drops below 80%

  integration-test:
    runs-on: ubuntu-latest
    services:
      postgres:             # Testcontainers alternative: run service in runner
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: --health-cmd="pg_isready" --health-interval=10s
      redis:
        image: redis:7
        ports:
          - 6379:6379
    steps:
    - uses: actions/checkout@v4
    - name: Run integration tests
      run: npm run test:integration
      env:
        DATABASE_URL: postgresql://postgres:test@localhost:5432/testdb
        REDIS_URL: redis://localhost:6379
```

**Smoke tests (post-deploy validation):**
```bash
# Smoke tests = minimal checks that app is UP and responding after deploy
# Run IMMEDIATELY after every deploy (staging and production)
# Fast: < 60 seconds
# Purpose: catch "the deploy broke everything" scenario immediately

#!/usr/bin/env bash
BASE_URL="${1:-https://staging.myapp.com}"
TIMEOUT=30

check() {
    local endpoint="$1"
    local expected="$2"
    local status
    status=$(curl -s -o /dev/null -w "%{http_code}" \
        --max-time "$TIMEOUT" "${BASE_URL}${endpoint}")
    if [ "$status" = "$expected" ]; then
        echo "✅ ${endpoint} returned ${status}"
    else
        echo "❌ ${endpoint} returned ${status} (expected ${expected})"
        exit 1
    fi
}

check "/health" "200"
check "/api/version" "200"
check "/api/products" "200"
check "/api/nonexistent" "404"

# In GitHub Actions after deploy:
- name: Smoke tests
  run: bash scripts/smoke-tests.sh https://staging.myapp.com
```

**Performance tests in pipeline (k6):**
```yaml
# k6 — load testing as code, fast, JavaScript-based
# Run in CI: quick load test to catch obvious regressions

- name: k6 performance test
  uses: grafana/k6-action@v0.3.1
  with:
    filename: tests/performance/load-test.js
  env:
    BASE_URL: https://staging.myapp.com

# tests/performance/load-test.js:
# import http from 'k6/http';
# import { check, sleep } from 'k6';
#
# export const options = {
#   stages: [
#     { duration: '30s', target: 50 },   // ramp to 50 users
#     { duration: '1m',  target: 50 },   // stay at 50
#     { duration: '10s', target: 0 },    // ramp down
#   ],
#   thresholds: {
#     http_req_duration: ['p(95)<500'],  // 95% of requests < 500ms
#     http_req_failed: ['rate<0.01'],    // < 1% failure rate
#   },
# };
#
# export default function() {
#   const res = http.get(`${__ENV.BASE_URL}/api/products`);
#   check(res, { 'status 200': (r) => r.status === 200 });
#   sleep(1);
# }
```

**Contract tests (Pact — covered in Q229, brief recap):**
```
Contract testing: API consumer and provider agree on contract
  Consumer writes test: "I expect GET /api/user/123 to return {id, name, email}"
  Provider verifies:    "I fulfil this contract"
  CI blocks deploy if contract is broken
```

> 💡 **Interview tip:** The **biggest CI/CD testing mistake** is running E2E tests in the PR pipeline. E2E tests are slow (30+ min), brittle (fail on network issues, timing), and expensive (need full environment). They should run AFTER deploy to staging, not before merge. The correct pipeline: unit tests + integration tests → merge → deploy staging → E2E + smoke tests → approve → deploy production. Also: `services:` in GitHub Actions (postgres, redis) is often more practical than Testcontainers for simple integration tests — zero code change needed, just YAML configuration.

---

## SECTION 14 — DB MIGRATIONS IN CI/CD

### Q-CICD-14 — CI/CD | Conceptual | Intermediate

> How do you handle **database schema migrations** in a CI/CD pipeline?
> What are **Flyway** and **Liquibase**? What is the **expand-contract** pattern
> for zero-downtime migrations?

#### Key Points to Cover:

**Why DB migrations are hard in CI/CD:**
```
Problem: deploying a new app version that requires DB schema changes
  → Can't change schema and deploy app simultaneously (race condition)
  → Old app still running during deploy — can't break it with new schema
  → Rollback: what if new schema change is hard to revert?

Solutions:
  1. Run migrations AS PART of app startup (risky)
  2. Run migrations as a separate CI/CD step BEFORE deploy (better)
  3. Expand-contract pattern (best for zero-downtime)
```

**Flyway (Java-centric, SQL migrations):**
```sql
-- Migrations in: src/main/resources/db/migration/
-- Naming: V{version}__{description}.sql

-- V1__Create_users_table.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- V2__Add_name_to_users.sql
ALTER TABLE users ADD COLUMN name VARCHAR(255);

-- V3__Add_index_email.sql
CREATE INDEX idx_users_email ON users(email);
```

```yaml
# In CI/CD pipeline (before deploy):
- name: Run Flyway migrations
  run: |
    flyway \
      -url=jdbc:postgresql://${{ secrets.DB_HOST }}/mydb \
      -user=${{ secrets.DB_USER }} \
      -password=${{ secrets.DB_PASSWORD }} \
      migrate

# Flyway tracks: which migrations ran (flyway_schema_history table)
# On re-run: skips already-applied migrations (idempotent)
# On failure: marks migration as failed, blocks further deploys until fixed
```

**Liquibase (XML/YAML/JSON/SQL, database-agnostic):**
```yaml
# db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: siddharth
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: email
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
                    unique: true

# Run in CI/CD:
- name: Liquibase migration
  run: liquibase --url=jdbc:postgresql://$DB_HOST/mydb update
```

**Expand-Contract pattern (zero-downtime migrations):**
```
For breaking schema changes, use 3-step deploy:

Step 1: EXPAND (add new, keep old)
  → Add new column/table (nullable or with default)
  → Old app still works (ignores new column)
  → New app writes to BOTH old and new column
  Deploy: new schema → deploy new app version
  ✅ Zero downtime

Step 2: MIGRATE DATA
  → Backfill new column with data from old column
  → Can run as background job, no downtime

Step 3: CONTRACT (remove old)
  → Remove old column after all app instances use new
  → Requires: new app version fully deployed + old app version gone
  Deploy: migration removing old column → done
  ✅ Zero downtime

Example: rename 'username' to 'display_name':
  WRONG (causes downtime):
    ALTER TABLE users RENAME COLUMN username TO display_name;
    Deploy new app. Old app crashes (column not found)!

  CORRECT (expand-contract):
    Step 1: ALTER TABLE users ADD COLUMN display_name VARCHAR(255);
    Step 1: Deploy app that reads display_name (falls back to username)
    Step 2: UPDATE users SET display_name = username WHERE display_name IS NULL;
    Step 3: Deploy app that ONLY uses display_name
    Step 3: ALTER TABLE users DROP COLUMN username;
```

> 💡 **Interview tip:** The expand-contract pattern is the **senior-level answer** to "how do you do zero-downtime DB migrations?" Junior answer: "use a maintenance window." Senior answer: "design migrations to be backward-compatible, use expand-contract for breaking changes." Also mention: **always test migrations against a production-sized database** in staging — a migration that takes 2 seconds on 1,000 rows takes 20 minutes on 10 million rows (table lock during that time = production outage). Use `CONCURRENTLY` for index creation in PostgreSQL to avoid table locks.

---

## SECTION 15 — REMAINING DEPLOYMENT STRATEGIES

### Q-CICD-15 — CI/CD | Conceptual | Intermediate

> Explain the **less common deployment strategies** — Recreate, Shadow/Dark
> launch, and A/B testing deployment. When would you use each?

#### Key Points to Cover:

```
Complete deployment strategies (all 6):

1. Recreate:
   → Stop ALL old pods → start ALL new pods
   → Downtime: YES (gap between stop and start)
   → Use: dev/staging environments, stateful apps that can't run two versions
   → Simple, fast, zero complexity
   → K8s: strategy: type: Recreate

2. Rolling Update (default K8s):
   → Gradually replace old with new pods
   → Downtime: no (if configured correctly)
   → Risk: brief period with mixed versions
   → K8s: strategy: type: RollingUpdate

3. Blue/Green:
   → Two identical environments, switch traffic
   → Downtime: no (instantaneous switch)
   → Rollback: instant (switch back)
   → Cost: 2x infrastructure during deployment
   → Use: critical services needing instant rollback

4. Canary:
   → Route small % of traffic to new version
   → Gradually increase if metrics are healthy
   → Downtime: no
   → Risk: only % of users affected if bug
   → Use: large-scale services, risk-sensitive deployments

5. Shadow (Dark Launch / Traffic Mirroring):
   → Copy real traffic to new version WITHOUT users seeing responses
   → New version processes requests but responses are DISCARDED
   → Users only see responses from old version
   → Use: validate new version behavior against real traffic before launch
   → Test with production traffic, zero user impact
   → K8s Istio: traffic mirroring
   → AWS: ALB target group traffic mirroring

   Example: new ML model — mirror 10% of production requests to new model
   Compare: does new model give same answers as old?
   If yes: switch traffic. If no: fix without any user impact.

6. A/B Testing (different from canary):
   Canary: gradually roll out new VERSION to all users
   A/B:    permanently serve different VARIANTS to different user GROUPS
           for experimentation/optimization

   A/B: 50% users see button color "red", 50% see "blue"
        Measure: which converts better?
        Not a deployment strategy — a product experiment

   Implementation:
   → Feature flags (LaunchDarkly, split.io)
   → Nginx: split traffic by cookie/header
   → Route 53: weighted routing
   → CloudFront: Lambda@Edge for user segmentation
```

> 💡 **Interview tip:** The **Shadow deployment** is the most impressive strategy to mention — most candidates know blue/green and canary but few know shadow. It answers the question "how do I test a new version with real production traffic but with zero risk to users?" Perfect for ML model validation, major algorithm changes, or new service versions where canary testing would still affect real users. The trade-off: you need infrastructure to handle 2x traffic (both versions process every mirrored request).

---

## SECTION 16 — KANIKO & ROOTLESS IMAGE BUILDS

### Q-CICD-16 — CI/CD | Conceptual | Intermediate

> Explain **Kaniko** and **Buildah** — why are they needed for building
> Docker images in Kubernetes without privileged mode?

#### Key Points to Cover:

**The problem:**
```
Building Docker images traditionally requires Docker daemon
Docker daemon requires privileged/root access

In Kubernetes:
  → Running privileged containers = security risk
  → Docker daemon inside pod (DinD) = very dangerous
  → If attacker escapes container → full host access

Solution: rootless image builders that don't need Docker daemon
```

**Kaniko:**
```yaml
# Kaniko runs as a regular (non-privileged) Kubernetes pod
# Reads Dockerfile, builds image, pushes to registry
# No Docker daemon needed

# Kubernetes Job to build with Kaniko:
apiVersion: batch/v1
kind: Job
metadata:
  name: build-myapp
spec:
  template:
    spec:
      containers:
      - name: kaniko
        image: gcr.io/kaniko-project/executor:latest
        args:
          - "--context=git://github.com/myorg/myapp"
          - "--dockerfile=Dockerfile"
          - "--destination=myregistry.com/myapp:v1.0"
          - "--cache=true"                    # layer caching
          - "--cache-repo=myregistry.com/cache"
        volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
      volumes:
      - name: docker-config
        secret:
          secretName: docker-registry-credentials
      restartPolicy: Never

# Jenkins pipeline with Kaniko agent:
pipeline {
  agent {
    kubernetes {
      yaml '''
        spec:
          containers:
          - name: kaniko
            image: gcr.io/kaniko-project/executor:debug
            command: ["/busybox/cat"]
            tty: true
      '''
    }
  }
  stages {
    stage('Build') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor \
              --context . \
              --dockerfile Dockerfile \
              --destination myregistry.com/myapp:${GIT_COMMIT}
          '''
        }
      }
    }
  }
}
```

**Buildah:**
```bash
# Buildah: OCI image builder, runs rootless
# More flexible than Kaniko (not limited to Dockerfile)

# Build from Dockerfile:
buildah bud -t myapp:latest .

# Build programmatically (no Dockerfile):
container=$(buildah from alpine:3.19)
buildah run $container -- apk add --no-cache python3
buildah copy $container ./app /app
buildah config --cmd "/app/main.py" $container
buildah commit $container myapp:latest

# Push:
buildah push myapp:latest docker://myregistry.com/myapp:latest
```

**Docker BuildKit caching in CI:**
```yaml
# GitHub Actions with BuildKit cache:
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myregistry.com/myapp:${{ github.sha }}
    cache-from: type=gha         # GitHub Actions cache
    cache-to: type=gha,mode=max
    # mode=max: cache ALL layers (not just final stage)
    # Without caching: 5 min builds
    # With caching: 30 second builds (only changed layers rebuilt)
```

> 💡 **Interview tip:** Kaniko is the **standard choice for building images in Kubernetes CI/CD** (Jenkins on K8s, Tekton, etc.). The key advantage: zero privileged access required, making it suitable for shared multi-tenant Kubernetes clusters. BuildKit with GitHub Actions cache (`type=gha`) is the fastest approach for GitHub Actions — it stores Docker layer cache in GitHub's cache storage and restores it on subsequent runs, turning 5-minute image builds into 30-second builds after the first run.

---

## SECTION 17 — IMAGE TAGGING STRATEGY

### Q-CICD-17 — CI/CD | Conceptual | Intermediate

> Explain **container image tagging strategies** — what are the options,
> what are the risks of using `latest`, and what is best practice?

#### Key Points to Cover:

```
Image tagging strategies:

1. latest tag (AVOID in production):
   myapp:latest
   → Mutable — can point to different images over time
   → "I deployed v1.2 but latest now points to v1.3" — not the same image
   → No rollback history (what was "latest" yesterday?)
   → Kubernetes caching: if imagePullPolicy=IfNotPresent, doesn't re-pull
   → NEVER use latest in Kubernetes manifests for production

2. Git commit SHA (RECOMMENDED):
   myapp:abc1234f                  # short SHA (7-8 chars)
   myapp:sha-abc1234f              # with prefix
   → Immutable — SHA ties to exact commit
   → Full audit trail (which commit = which image)
   → Can reproduce any historical deployment
   → K8s: imagePullPolicy: IfNotPresent (cache works correctly)

3. Semantic version (for released versions):
   myapp:1.2.3                     # exact version
   myapp:v1.2.3                    # with v prefix
   → Clear versioning, good for external consumers
   → Combine with SHA for internal tracking

4. Branch + SHA (for dev branches):
   myapp:main-abc1234f             # branch + SHA
   myapp:feature-auth-abc1234f     # feature branch builds

5. Date-based (avoid — no semantics):
   myapp:20240115                  # date only — no ordering guarantee

Best practice (combine approaches):
  myapp:sha-abc1234f               # primary (immutable, always use this in K8s)
  myapp:v1.2.3                     # also tag on release
  myapp:latest                     # also tag latest (for discovery, NOT for deploy)

In Kubernetes manifests:
  image: myregistry.com/myapp:sha-abc1234f    # ✅ immutable
  image: myregistry.com/myapp:latest          # ❌ mutable, don't use

ECR image immutability (block tag overwriting):
  aws ecr put-image-tag-mutability \
    --repository-name myapp \
    --image-tag-mutability IMMUTABLE
  # Now: pushing same tag twice = error (protects your immutable SHAs)
```

> 💡 **Interview tip:** The `latest` tag anti-pattern causes a specific production incident scenario: developer pushes a broken image → CI/CD tags it as `latest` → a pod is rescheduled on a different node → new node pulls `latest` → broken image → pod crashes. The healthy pods (on other nodes with cached old image) work fine. Half your pods crash while the other half work — bizarre intermittent failures that are hard to diagnose. **Immutable SHA tags + ECR immutability** completely eliminate this class of problem.

---

## SECTION 18 — OWASP TOP 10

### Q-CICD-18 — Security | Conceptual | Intermediate

> Explain the **OWASP Top 10** — what are the most critical web application
> security vulnerabilities? How does each relate to what you check in CI/CD?

#### Key Points to Cover:

```
OWASP Top 10 (2021 edition) — most critical web app vulnerabilities:

A01 — Broken Access Control (most common):
  → Users can access data/functions they shouldn't
  → Missing authorization checks
  → CI/CD: SAST checks for missing @Auth annotations, exposed endpoints
  → Example: /api/admin accessible without admin role

A02 — Cryptographic Failures:
  → Sensitive data transmitted/stored without encryption
  → Weak algorithms: MD5, SHA1, DES, ECB mode
  → CI/CD: SAST flags use of MD5/SHA1/weak crypto
  → Example: passwords stored as MD5 hash

A03 — Injection (SQL, NoSQL, OS, LDAP):
  → Untrusted data sent to interpreter as command
  → SQL injection: ' OR '1'='1 in login form
  → CI/CD: SAST detects string concatenation in SQL queries
  → Fix: parameterized queries, prepared statements

A04 — Insecure Design:
  → Missing security controls by design
  → No rate limiting, no account lockout
  → CI/CD: hard to automate — needs security architecture review

A05 — Security Misconfiguration:
  → Default credentials, unnecessary features enabled
  → S3 buckets public, debug mode in production
  → CI/CD: IaC scanning (Checkov/tfsec), DAST security headers check

A06 — Vulnerable and Outdated Components:
  → Third-party libraries with known CVEs
  → CI/CD: SCA (Snyk, OWASP DC, Dependabot) — directly addresses this

A07 — Identification and Authentication Failures:
  → Weak passwords, brute force possible
  → Session tokens predictable
  → CI/CD: SAST checks for hardcoded credentials, weak JWT secrets

A08 — Software and Data Integrity Failures:
  → Untrusted plugins/libraries/updates
  → Supply chain attacks
  → CI/CD: image signing (Cosign), SLSA, dependency hash verification

A09 — Security Logging and Monitoring Failures:
  → Missing logs for security events
  → No alerting on suspicious activity
  → CI/CD: can check log statements in code, runtime: AWS CloudTrail, GuardDuty

A10 — Server-Side Request Forgery (SSRF):
  → Server makes request to internal resource based on user input
  → Attacker: fetch http://169.254.169.254/ (AWS metadata service)
  → CI/CD: SAST detects unchecked URL inputs used in HTTP requests

CI/CD coverage of OWASP Top 10:
  A01, A02, A03, A07, A10 → SAST (code analysis)
  A06                       → SCA (dependency scanning)
  A05, A08                  → IaC scanning + image signing
  A05 (runtime)             → DAST (security headers, config)
  A09                       → monitoring and logging standards
  A04                       → manual security design review
```

> 💡 **Interview tip:** Knowing OWASP Top 10 is the **baseline security knowledge** every DevOps/SRE engineer needs. The key interview insight: **CI/CD pipeline can automatically catch A01/A02/A03/A06/A07 through SAST and SCA**. This is exactly the value of Shift Left — these 5 categories (which are the most common vulnerabilities) can be automatically detected before they reach production. A03 (injection) is the classic — SAST tools find SQL string concatenation immediately: `"SELECT * FROM users WHERE id=" + userId` is flagged as injection risk in every SAST tool.

---

## QUICK REFERENCE — All Security Tools

```
SECRET SCANNING:
  gitleaks     → pre-commit + CI, pattern-based
  trufflehog   → git history deep scan, verifies active secrets

SAST (code):
  Semgrep      → fast, rules-based, multi-language
  CodeQL       → GitHub native, deep analysis
  Bandit       → Python specific
  SonarQube    → code quality + security, quality gates

SCA (dependencies):
  Snyk         → commercial, comprehensive, fix suggestions
  OWASP DC     → open source, Java/Maven focus
  Dependabot   → GitHub native, automatic upgrade PRs
  Renovate     → configurable, grouping, auto-merge

IMAGE SCANNING:
  Trivy        → fast, comprehensive, free
  Grype        → Anchore, also scans SBOM
  ECR Scanning → AWS native (Inspector)

IaC SECURITY:
  Checkov      → multi-framework (TF, K8s, Dockerfile)
  tfsec        → Terraform specific
  terrascan    → multi-cloud IaC

DAST:
  OWASP ZAP    → free, baseline + full scan

SBOM:
  Syft         → generate SBOM from image/filesystem
  CycloneDX    → SBOM format standard
  SPDX         → SBOM format standard (GitHub)

SUPPLY CHAIN:
  Cosign       → sign + verify images
  Sigstore     → keyless signing infrastructure
  SLSA         → framework for supply chain levels

LICENSE:
  FOSSA        → commercial, comprehensive
  license-checker → npm/Node.js specific

POLICY:
  OPA          → policy as code (general)
  Kyverno      → K8s native admission policies
  Checkov      → IaC policy enforcement
```

---

## SECTION 19 — ARTIFACT PROMOTION

### Q-CICD-19 — CI/CD | Conceptual | Intermediate

> Explain **artifact promotion** — what is it, why is it important, and what
> is the **Build Once, Deploy Many** principle?
>
> Walk through the complete artifact promotion pipeline from build to production
> covering: Docker images, Helm charts, Maven JARs, and npm packages.

#### Key Points to Cover:

**What artifact promotion is:**
```
Artifact promotion = moving the SAME artifact through pipeline stages
                     without rebuilding it

The wrong approach (rebuild at each stage):
  dev branch  → build image A → test → pass
  staging     → build image B → test → pass
  production  → build image C → deploy

  Problem: image A ≠ image B ≠ image C
  → Different build times = different dependency resolutions
  → "Works in staging" but prod has different library version
  → You never actually tested what you deployed
  → Non-deterministic builds = trust problem

The correct approach (build once, promote same artifact):
  CI builds ONE artifact (image A with SHA abc123)
  dev        → test image A → pass
  staging    → test image A → pass  ← SAME image
  production → deploy image A       ← SAME image

  Guarantee: exactly what you tested is what you deployed
```

**Build Once, Deploy Many principle:**
```
Core idea:
  → Build artifact ONCE (in CI on PR merge)
  → Promote THE SAME artifact through environments
  → Environment-specific config via ConfigMaps/env vars (NOT baked into image)
  → Never rebuild for a different environment

What changes per environment:
  ✅ Environment variables (DATABASE_URL, LOG_LEVEL)
  ✅ ConfigMaps / Secrets
  ✅ Replica counts, resource limits
  ✅ Ingress hostnames

What does NOT change per environment:
  ❌ The container image (same SHA in dev, staging, prod)
  ❌ The Helm chart version
  ❌ The JAR/WAR file

Benefits:
  → Deterministic: prod runs exactly what was tested
  → Faster: no rebuild time per environment
  → Auditable: SHA ties back to exact Git commit
  → Safer: no "surprise" dependency change between test and deploy
```

---

### Q-CICD-20 — CI/CD | Conceptual | Intermediate

> Explain **Docker image promotion** — how do you promote a container image
> from dev → staging → production without rebuilding?
>
> What tools exist for copying images between registries? What is the
> **GitOps-based image promotion** pattern using ArgoCD Image Updater?

#### Key Points to Cover:

**Docker image promotion strategies:**

**Strategy 1 — Single registry, promote by retagging:**
```bash
# Build once, tag with SHA:
docker build -t myregistry.com/myapp:sha-abc1234 .
docker push myregistry.com/myapp:sha-abc1234

# Promote to staging (add staging tag to same image):
docker pull myregistry.com/myapp:sha-abc1234
docker tag  myregistry.com/myapp:sha-abc1234 \
            myregistry.com/myapp:staging
docker push myregistry.com/myapp:staging

# Promote to production (add prod tag):
docker tag  myregistry.com/myapp:sha-abc1234 \
            myregistry.com/myapp:production
docker push myregistry.com/myapp:production
docker tag  myregistry.com/myapp:sha-abc1234 \
            myregistry.com/myapp:v1.2.3
docker push myregistry.com/myapp:v1.2.3

# Result: same image layers, multiple tags
# sha-abc1234 = staging = production = v1.2.3 → all point to SAME image
```

**Strategy 2 — skopeo (copy without pulling/pushing full image):**
```bash
# skopeo copies images DIRECTLY between registries (no docker daemon needed)
# Much faster: only transfers changed layers

# Install:
brew install skopeo

# Copy within same registry (retag):
skopeo copy \
  docker://myregistry.com/myapp:sha-abc1234 \
  docker://myregistry.com/myapp:staging

# Copy between different registries:
skopeo copy \
  docker://dev-registry.com/myapp:sha-abc1234 \
  docker://prod-registry.com/myapp:sha-abc1234

# Copy from ECR dev account to ECR prod account:
skopeo copy \
  --src-creds AWS:$(aws ecr get-login-password --region us-east-1) \
  --dest-creds AWS:$(aws ecr get-login-password --region us-east-1 \
    --profile prod-account) \
  docker://111111111111.dkr.ecr.us-east-1.amazonaws.com/myapp:sha-abc1234 \
  docker://222222222222.dkr.ecr.us-east-1.amazonaws.com/myapp:sha-abc1234

# Key advantage: skopeo never downloads the image layers locally
# Pure registry-to-registry copy → much faster than docker pull + push
```

**Strategy 3 — crane (Google's image tool):**
```bash
# crane: lightweight image manipulation tool
brew install crane

# Copy image between registries:
crane copy \
  dev-registry.com/myapp:sha-abc1234 \
  prod-registry.com/myapp:sha-abc1234

# Retag without pulling:
crane tag myregistry.com/myapp:sha-abc1234 staging
crane tag myregistry.com/myapp:sha-abc1234 v1.2.3

# Get image digest:
crane digest myregistry.com/myapp:sha-abc1234
# sha256:abc123def456...

# List all tags for an image:
crane ls myregistry.com/myapp
```

**Strategy 4 — ECR cross-account replication (AWS native):**
```bash
# Auto-replicate images to prod account on push to dev account
aws ecr put-replication-configuration \
  --replication-configuration '{
    "rules": [{
      "destinations": [{
        "region": "us-east-1",
        "registryId": "PROD_ACCOUNT_ID"
      }],
      "repositoryFilters": [{
        "filter": "myapp",
        "filterType": "PREFIX_MATCH"
      }]
    }]
  }'
# Any push to dev ECR myapp/* → auto-replicated to prod ECR
# No manual copy step needed
```

**GitOps image promotion (ArgoCD Image Updater):**
```yaml
# ArgoCD Image Updater watches registry for new image tags
# Automatically updates image tag in Git (GitOps compliant)

# Install:
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

# Annotate ArgoCD Application for auto-update:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  namespace: argocd
  annotations:
    # Watch this image in registry:
    argocd-image-updater.argoproj.io/image-list: myapp=myregistry.com/myapp
    # Update strategy (semver, latest, digest, name):
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
    # Which tag constraint:
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    # Write back to Git:
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
spec:
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    path: apps/myapp/staging
    targetRevision: main

# Flow:
# CI pushes myapp:v1.2.3 to registry
# ArgoCD Image Updater detects new tag
# Updates k8s-manifests/apps/myapp/staging/deployment.yaml
#   image: myregistry.com/myapp:v1.2.3  ← auto-committed to Git
# ArgoCD syncs → staging deploys v1.2.3
# Manual/automated gate → update production manifest
```

**GitHub Actions promotion pipeline:**
```yaml
name: Artifact Promotion Pipeline

on:
  push:
    branches: [main]

jobs:
  # Stage 1: Build ONCE
  build:
    runs-on: ubuntu-latest
    outputs:
      image-sha: ${{ steps.build.outputs.sha }}
    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
        aws-region: us-east-1

    - name: Login to ECR
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build and push (ONCE)
      id: build
      run: |
        SHA="sha-${GITHUB_SHA::8}"
        IMAGE="${{ secrets.ECR_REGISTRY }}/myapp:${SHA}"
        docker build -t "$IMAGE" .
        docker push "$IMAGE"
        echo "sha=$SHA" >> $GITHUB_OUTPUT
        echo "✅ Built: $IMAGE"

  # Stage 2: Deploy to dev (auto)
  deploy-dev:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - name: Promote to dev
      run: |
        # Retag: sha → dev (no rebuild)
        skopeo copy \
          docker://${{ secrets.ECR_REGISTRY }}/myapp:${{ needs.build.outputs.image-sha }} \
          docker://${{ secrets.ECR_REGISTRY }}/myapp:dev
        # Deploy
        kubectl set image deployment/myapp \
          myapp=${{ secrets.ECR_REGISTRY }}/myapp:${{ needs.build.outputs.image-sha }}

  # Stage 3: Run tests against dev
  test-dev:
    needs: deploy-dev
    runs-on: ubuntu-latest
    steps:
    - name: Integration tests
      run: npm run test:integration -- --baseUrl=https://dev.myapp.com
    - name: Smoke tests
      run: bash scripts/smoke-tests.sh https://dev.myapp.com

  # Stage 4: Promote to staging (auto, same artifact)
  promote-staging:
    needs: test-dev
    runs-on: ubuntu-latest
    steps:
    - name: Promote SHA to staging tag
      run: |
        # Same SHA image → now tagged as staging
        skopeo copy \
          docker://${{ secrets.ECR_REGISTRY }}/myapp:${{ needs.build.outputs.image-sha }} \
          docker://${{ secrets.ECR_REGISTRY }}/myapp:staging
        kubectl set image deployment/myapp \
          myapp=${{ secrets.ECR_REGISTRY }}/myapp:${{ needs.build.outputs.image-sha }} \
          -n staging
        echo "✅ Promoted ${{ needs.build.outputs.image-sha }} → staging"

  # Stage 5: Test staging
  test-staging:
    needs: promote-staging
    runs-on: ubuntu-latest
    steps:
    - name: E2E tests on staging
      run: npm run test:e2e -- --baseUrl=https://staging.myapp.com

  # Stage 6: Promote to production (manual approval)
  promote-production:
    needs: test-staging
    runs-on: ubuntu-latest
    environment: production    # requires manual approval
    steps:
    - name: Promote SHA to production
      run: |
        # SAME SHA from build step → production
        # This is exactly what was tested in dev and staging
        skopeo copy \
          docker://${{ secrets.ECR_REGISTRY }}/myapp:${{ needs.build.outputs.image-sha }} \
          docker://${{ secrets.ECR_REGISTRY }}/myapp:production
        skopeo copy \
          docker://${{ secrets.ECR_REGISTRY }}/myapp:${{ needs.build.outputs.image-sha }} \
          docker://${{ secrets.ECR_REGISTRY }}/myapp:v1.2.3
        kubectl set image deployment/myapp \
          myapp=${{ secrets.ECR_REGISTRY }}/myapp:${{ needs.build.outputs.image-sha }} \
          -n production
        echo "✅ PROMOTED ${{ needs.build.outputs.image-sha }} → production"
```

> 💡 **Interview tip:** The most powerful argument for artifact promotion: **you can prove to auditors exactly what ran in production**. `sha-abc1234` maps to Git commit `abc1234` which maps to a code review, a PR, tests that passed, security scans that passed. Without promotion (rebuild each env), you cannot prove the prod binary is the same as the staging binary. This is a **SOC2 / ISO 27001 compliance requirement** — most enterprises require it. The `skopeo copy` command is the tool most candidates don't know — it copies images directly registry-to-registry without downloading them locally, making promotions fast even for large images.

---

### Q-CICD-21 — CI/CD | Conceptual | Intermediate

> Explain **artifact promotion in Nexus/Artifactory** for non-container
> artifacts (Maven JARs, npm packages, Python wheels).
>
> What is the difference between **snapshot** and **release** repositories?
> What is **staging repository promotion** in Nexus?

#### Key Points to Cover:

**Nexus/Artifactory promotion concept:**
```
Artifact repositories have MULTIPLE repositories per artifact type:

Maven example:
  maven-snapshots  → development builds (mutable, can overwrite)
  maven-staging    → candidate releases (tested, pending approval)
  maven-releases   → production releases (immutable, never overwrite)

Promotion flow:
  Dev builds         → maven-snapshots (myapp-1.2.3-SNAPSHOT.jar)
  CI tests pass      → promote to maven-staging (myapp-1.2.3.jar)
  QA approves        → promote to maven-releases (myapp-1.2.3.jar)
  Never rebuilt — same JAR bytes move through repositories
```

**Snapshot vs Release:**
```
SNAPSHOT (dev builds):
  → Version: 1.2.3-SNAPSHOT
  → Mutable: each build overwrites previous
  → Timestamped internally: 1.2.3-20240115.142305-3
  → Downloaded fresh every build (Maven checks for updates)
  → Use: active development, frequent builds

RELEASE (stable builds):
  → Version: 1.2.3 (no SNAPSHOT suffix)
  → Immutable: once published, NEVER overwrite
  → Cached forever by Maven/Gradle (no re-download)
  → Use: deployable artifacts, production

Promotion = convert SNAPSHOT → RELEASE
  Same bytes, different repository, different version string
  1.2.3-SNAPSHOT in snapshots repo
  → (tests pass, approve)
  → 1.2.3 in releases repo
```

**Nexus Staging Plugin (Maven):**
```xml
<!-- pom.xml: configure staging plugin -->
<plugin>
  <groupId>org.sonatype.plugins</groupId>
  <artifactId>nexus-staging-maven-plugin</artifactId>
  <version>1.6.13</version>
  <extensions>true</extensions>
  <configuration>
    <serverId>nexus</serverId>
    <nexusUrl>https://nexus.company.com/</nexusUrl>
    <autoReleaseAfterClose>false</autoReleaseAfterClose>
    <!-- false = manual promotion step required -->
  </configuration>
</plugin>
```

```bash
# CI pipeline:
# Step 1: Deploy to staging repository:
mvn deploy -P release \
  -DskipTests=false \
  -Dnexus.staging.deploy=true

# Step 2: Run tests against staged artifact
# ...automated tests...

# Step 3: Promote staging → release (if tests pass):
mvn nexus-staging:release   # promotes staged artifact to release repo

# Step 4: If tests fail, drop staging:
mvn nexus-staging:drop      # removes staged artifact (no release created)
```

**Artifactory promotion (REST API + JFrog CLI):**
```bash
# JFrog CLI promotion:
jfrog rt build-promote \
  my-build \              # build name
  42 \                    # build number
  libs-release-local \    # target repository
  --status="Released" \
  --comment="Promoted after QA approval" \
  --copy=true             # copy (not move) artifacts

# REST API promotion:
curl -X POST \
  -H "Content-Type: application/json" \
  -u admin:password \
  "https://artifactory.company.com/api/build/promote/my-build/42" \
  -d '{
    "status": "Released",
    "comment": "Approved for production",
    "ciUser": "jenkins",
    "timestamp": "2024-01-15T14:00:00.000Z",
    "sourceRepo": "libs-staging-local",
    "targetRepo": "libs-release-local",
    "copy": true,
    "artifacts": true,
    "dependencies": false
  }'

# What promotion records:
# → Which build promoted this artifact
# → Who approved it (audit trail)
# → When it was promoted
# → From which repo to which repo
# → Build properties (Git commit, branch, PR number)
```

**npm package promotion:**
```bash
# npm registry promotion (Nexus/Artifactory):
# dev publishes to npm-snapshots registry:
npm publish --registry https://nexus.company.com/repository/npm-snapshots/

# After tests pass, promote to npm-releases:
# (copy via Nexus API or Artifactory CLI)
jfrog rt copy \
  npm-snapshots-local/mypackage/-/mypackage-1.2.3.tgz \
  npm-releases-local/mypackage/-/mypackage-1.2.3.tgz

# Python wheel promotion:
# Build:
python setup.py bdist_wheel
# Publish to staging:
twine upload \
  --repository-url https://nexus.company.com/repository/pypi-snapshots/ \
  dist/mypackage-1.2.3-py3-none-any.whl
# Promote staging → releases:
jfrog rt copy \
  pypi-snapshots-local/mypackage/1.2.3/ \
  pypi-releases-local/mypackage/1.2.3/
```

> 💡 **Interview tip:** The **Maven SNAPSHOT anti-pattern** in production is one of the most common Java CI/CD mistakes — deploying a `-SNAPSHOT` artifact to production. SNAPSHOT versions are MUTABLE — someone can accidentally overwrite them in Nexus, and a new deploy pulls a different binary than what was tested. Production MUST only deploy RELEASE artifacts (immutable). The promotion process enforces this: SNAPSHOT → tests pass → promote to RELEASE → deploy RELEASE to production. The promotion step is the gate that ensures immutability.

---

### Q-CICD-22 — CI/CD | Conceptual | Intermediate

> Explain **Helm chart promotion** — how do you promote a Helm chart through
> environments? What is **ChartMuseum** and how does it compare to using
> an OCI registry for charts?
>
> How do you handle **chart versioning** and what is the promotion workflow?

#### Key Points to Cover:

**Helm chart as a promotable artifact:**
```
Helm chart = versioned package of K8s manifests + templates

Like container images, Helm charts should be:
  → Built ONCE (packaged, versioned)
  → Tested in dev/staging with that exact chart version
  → Same chart version deployed to production

Chart versioning (Chart.yaml):
  version: 1.2.3         # chart version (the package)
  appVersion: "2.5.0"    # application version (optional)

  Bump chart version when:
    → K8s manifest changes
    → Template logic changes
    → Default values changes
  (NOT necessarily when app version changes)
```

**ChartMuseum (traditional Helm chart repository):**
```bash
# ChartMuseum = HTTP server that serves Helm charts
# Install (Docker):
docker run -p 8080:8080 \
  -e STORAGE=local \
  -e STORAGE_LOCAL_ROOTDIR=/charts \
  -v /charts:/charts \
  ghcr.io/helm/chartmuseum:latest

# Push chart to ChartMuseum:
helm package ./charts/myapp        # creates myapp-1.2.3.tgz
helm cm-push myapp-1.2.3.tgz my-chartmuseum

# Add repo and install:
helm repo add myrepo http://chartmuseum.company.com
helm repo update
helm install myapp myrepo/myapp --version 1.2.3
```

**OCI registry for Helm charts (modern approach):**
```bash
# Helm 3.8+: push/pull charts to OCI registry (ECR, Docker Hub, etc.)
# No ChartMuseum needed — use existing container registry

# Package chart:
helm package ./charts/myapp
# Creates: myapp-1.2.3.tgz

# Push to ECR OCI:
aws ecr create-repository --repository-name helm-charts/myapp
helm push myapp-1.2.3.tgz \
  oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm-charts

# Pull and install:
helm install myapp \
  oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm-charts/myapp \
  --version 1.2.3

# Benefits of OCI over ChartMuseum:
# → No separate infrastructure (use existing ECR)
# → Same auth mechanism (IAM roles)
# → Immutability enforced by ECR (same as images)
# → Content-addressable (digest-based)
```

**Helm chart promotion pipeline:**
```yaml
# GitHub Actions chart promotion:
jobs:
  build-chart:
    runs-on: ubuntu-latest
    outputs:
      chart-version: ${{ steps.version.outputs.version }}
    steps:
    - uses: actions/checkout@v4

    - name: Get chart version
      id: version
      run: |
        VERSION=$(grep '^version:' charts/myapp/Chart.yaml | awk '{print $2}')
        echo "version=$VERSION" >> $GITHUB_OUTPUT

    - name: Package chart (ONCE)
      run: helm package charts/myapp

    - name: Push to ECR (dev stage)
      run: |
        aws ecr get-login-password | helm registry login \
          --username AWS \
          --password-stdin \
          123456789012.dkr.ecr.us-east-1.amazonaws.com
        helm push myapp-${{ steps.version.outputs.chart-version }}.tgz \
          oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm-charts

  deploy-staging:
    needs: build-chart
    runs-on: ubuntu-latest
    steps:
    - name: Deploy with exact chart version
      run: |
        helm upgrade --install myapp \
          oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm-charts/myapp \
          --version ${{ needs.build-chart.outputs.chart-version }} \
          --values values/staging.yaml \
          --namespace staging

  promote-production:
    needs: [build-chart, deploy-staging]
    runs-on: ubuntu-latest
    environment: production
    steps:
    - name: Deploy SAME chart version to production
      run: |
        helm upgrade --install myapp \
          oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm-charts/myapp \
          --version ${{ needs.build-chart.outputs.chart-version }} \
          --values values/production.yaml \   # only values differ
          --namespace production
        # SAME chart version (1.2.3) in staging AND production
        # Only values file differs per environment
```

> 💡 **Interview tip:** The most common Helm promotion mistake: **different `values.yaml` in different environments** is correct and expected, but **different chart versions in different environments** means you never actually tested what you deployed to production. The rule: chart version must be identical across all environments. Only the values files differ. With OCI registries for Helm charts, promotion is simple — the chart is already in ECR, staging and production both pull the same version (`--version 1.2.3`) with different values. No copying needed, just reference the same OCI artifact.

---

### Q-CICD-23 — CI/CD | Conceptual | Intermediate

> Explain **promotion gates** — what checks must pass before an artifact
> is promoted to the next environment?
>
> How do you implement an **automated promotion decision** vs a
> **manual promotion gate**? What metrics/signals drive automated promotion?

#### Key Points to Cover:

**What promotion gates are:**
```
Promotion gate = set of conditions that must ALL pass
                 before artifact moves to next environment

Without gates: any broken build could reach production
With gates:    only healthy, validated artifacts are promoted

Gate types:
  Quality gates:   tests pass, coverage > threshold, no CRITICAL CVEs
  Performance gates: p99 latency within SLO, error rate < 1%
  Security gates:  no CRITICAL vulnerabilities, image signed, SBOM generated
  Manual gates:    human approval (compliance, high-risk changes)
  Time gates:      no deploys Friday 4pm–Monday 9am (change freeze)
```

**Automated promotion (pipeline as gate):**
```yaml
# Automated promotion decision based on signals:

promote-to-staging:
  needs: [unit-tests, integration-tests, security-scan, sca-scan]
  # All jobs in needs must succeed (status: success)
  # Any failure → promotion blocked automatically
  if: |
    needs.unit-tests.result == 'success' &&
    needs.integration-tests.result == 'success' &&
    needs.security-scan.result == 'success' &&
    needs.sca-scan.result == 'success'
  steps:
  - name: Auto-promote to staging
    run: ./scripts/promote.sh staging ${{ needs.build.outputs.image-sha }}

# Performance-based automated gate:
promote-to-production:
  needs: [deploy-staging, performance-test]
  steps:
  - name: Check performance gate
    run: |
      # Read k6 results:
      P99=$(cat k6-results.json | jq '.metrics.http_req_duration.values["p(99)"]')
      ERROR_RATE=$(cat k6-results.json | jq '.metrics.http_req_failed.values.rate')

      # Gate: p99 < 500ms AND error rate < 1%
      if (( $(echo "$P99 > 500" | bc -l) )); then
        echo "❌ GATE FAILED: p99 latency ${P99}ms > 500ms threshold"
        exit 1
      fi
      if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
        echo "❌ GATE FAILED: error rate ${ERROR_RATE} > 1% threshold"
        exit 1
      fi
      echo "✅ Performance gate passed: p99=${P99}ms, errors=${ERROR_RATE}"
```

**Manual promotion gate (GitHub Environments):**
```yaml
# Human approval required before production promotion:
deploy-production:
  environment: production     # configured in GitHub Settings:
                              # → Required reviewers: [tech-lead, platform-team]
                              # → Wait timer: 10 minutes
                              # → Allowed branches: main only
  needs: [test-staging]
  steps:
  - name: Deploy to production
    run: ./scripts/promote.sh production $IMAGE_SHA
```

**Canary-based promotion gate (Argo Rollouts):**
```yaml
# Automated promotion based on live traffic metrics:
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10          # 10% traffic to new version
      - pause: {duration: 5m}  # observe for 5 minutes
      - analysis:              # automated analysis gate
          templates:
          - templateName: error-rate-analysis
      - setWeight: 50          # promote to 50% if analysis passed
      - pause: {duration: 5m}
      - setWeight: 100         # full promotion

  # AnalysisTemplate: automated go/no-go based on Prometheus metrics:
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-analysis
spec:
  metrics:
  - name: error-rate
    interval: 1m
    failureLimit: 1          # abort rollout if fails once
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status=~"5..",app="myapp"}[5m]))
          / sum(rate(http_requests_total{app="myapp"}[5m]))
    successCondition: result[0] < 0.01  # < 1% error rate = promote
    failureCondition: result[0] >= 0.05 # >= 5% error rate = abort rollback
```

**Promotion audit trail:**
```bash
# Every promotion should be recorded:
# Who promoted, when, from what, to what, which artifact

# Git tag as audit trail:
git tag -a "promoted/staging/sha-abc1234" \
  -m "Promoted to staging by $GITHUB_ACTOR at $(date -u)"
git push origin "promoted/staging/sha-abc1234"

# ECR image labels as audit:
docker build \
  --label "promoted.by=$GITHUB_ACTOR" \
  --label "promoted.at=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --label "promoted.from=dev" \
  --label "promoted.to=staging" \
  --label "git.sha=$GITHUB_SHA" \
  -t myimage:staging .

# Query promotion history:
docker inspect myimage:staging | jq '.[0].Config.Labels'
```

> 💡 **Interview tip:** The most mature promotion gate pattern is **progressive delivery with Argo Rollouts** — instead of a binary "promote or don't promote", you route a small % of real traffic to the new version, measure error rate and latency against Prometheus, and automatically promote or rollback based on real user traffic. This is what companies like Netflix, Uber, and Google do. The key metric to use in the analysis template: **error rate change relative to baseline** (not absolute), so normal traffic variance doesn't trigger false rollbacks. A 0.5% error rate on both versions = no change = promote. A 0.5% → 2% increase = rollback.

---

## ARTIFACT PROMOTION QUICK REFERENCE

```
CONCEPT:
  Build Once, Deploy Many     → same artifact from dev to prod
  Promotion                   → move artifact, never rebuild
  Gate                        → conditions that must pass to promote

DOCKER IMAGE PROMOTION TOOLS:
  docker tag + push           → simple, requires docker daemon
  skopeo copy                 → fast, no docker daemon, registry-to-registry
  crane tag                   → lightweight, Google tool
  ECR replication             → AWS native, automatic on push

MAVEN/JAVA:
  SNAPSHOT                    → mutable, dev builds
  RELEASE                     → immutable, production
  Nexus staging plugin        → formal promote/drop workflow
  Artifactory CLI             → jfrog rt build-promote

HELM CHARTS:
  ChartMuseum                 → traditional HTTP chart server
  OCI registry (ECR)          → modern, same infra as images
  helm push / helm pull       → OCI commands
  --version pin               → always use exact chart version

PROMOTION GATES:
  Pipeline success            → all jobs passed (automated)
  Performance metrics         → p99, error rate thresholds
  Security gates              → no CRITICAL CVEs
  Manual approval             → GitHub Environments + reviewers
  Canary analysis             → Argo Rollouts + Prometheus metrics

AUDIT TRAIL:
  Git tags                    → promoted/staging/sha-abc1234
  Docker labels               → who, when, what was promoted
  Artifactory build info      → full promotion history
  ArgoCD history              → deployment audit log
```

---

*Artifact Promotion — Section 19–23 to be appended to CICD_Complete_Gaps_Reference.md*
*DevOps / SRE Interview Preparation — nawab312 GitHub repositories*

*CI/CD Complete Gaps Reference*
*Covers ALL CI/CD security, testing, and pipeline topics missing from Q1–Q675*
*DevOps / SRE Interview Preparation — nawab312 GitHub repositories*
