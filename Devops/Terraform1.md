# Terraform — Complete Commands & Concepts Deep Dive
## All Missing Topics | Every Command | Every Concept | With Key Points + Interview Tips

> **Why this file exists:** Previous question bank covered advanced Terraform scenarios
> but missed ALL the foundational commands and many core concepts that are asked
> in EVERY Terraform interview — `fmt`, `validate`, `console`, `graph`, `show`,
> version constraints, variable types, provider aliases, built-in functions, and more.
> This file fills every gap.

---

## Topic Coverage

| Section | Topics |
|---|---|
| **Core Commands** | fmt, validate, show, graph, console, force-unlock, get, init flags |
| **Variables & Outputs** | types, validation, sensitive, tfvars precedence, outputs |
| **State Management** | show, force-unlock, remote state data source, -target |
| **Built-in Functions** | lookup, merge, flatten, toset, cidrsubnet, templatefile |
| **Provider Configuration** | alias, version constraints, required_providers |
| **Meta-Arguments** | count, for_each, depends_on, lifecycle, provider together |
| **Provisioners** | local-exec, remote-exec, null_resource, terraform_data |
| **Testing & Quality** | fmt, validate, checkov, tfsec, infracost, terraform test |
| **New Features** | TF 1.5 import blocks, TF 1.7 ephemeral resources, TF 1.8+ |

---

## SECTION 1 — CORE COMMANDS

---

### TF-CMD-01 — Terraform | Conceptual | Beginner-Intermediate

> Explain **`terraform fmt`** and **`terraform validate`** — what does each command do, when do you run them, and how do you integrate them into a CI/CD pipeline?
>
> What is the difference between a **formatting error** and a **validation error**? Can your Terraform code be validly formatted but still be invalid?

#### Key Points to Cover:

**`terraform fmt`:**
```bash
# Formats Terraform code to canonical style (like gofmt for Go)
# Indentation, spacing, argument alignment — all standardized

terraform fmt              # format all .tf files in current directory
terraform fmt -recursive   # format all .tf files in subdirectories too
terraform fmt -check       # check only — exit code 1 if changes needed (CI use)
terraform fmt -diff        # show diff without writing (preview changes)
terraform fmt main.tf      # format specific file only

# What it fixes:
# Before:                    After:
resource "aws_instance" "web" {   resource "aws_instance" "web" {
instance_type="t3.medium"           instance_type = "t3.medium"
  ami           =    "ami-123"       ami           = "ami-123"
}                                  }
```

**`terraform validate`:**
```bash
# Validates Terraform configuration syntax and internal consistency
# Does NOT connect to AWS — purely offline check

terraform validate         # validate current directory
terraform validate -json   # machine-readable JSON output (for CI parsing)

# What it checks:
# ✅ HCL syntax is valid (no missing braces, typos in keywords)
# ✅ Required arguments present (resource type needs all mandatory fields)
# ✅ Variable references exist (var.name is defined)
# ✅ Module source paths exist
# ✅ Resource type names are valid

# What it does NOT check:
# ❌ AWS credentials (no AWS API calls)
# ❌ Whether resources actually exist in AWS
# ❌ Whether IAM permissions are correct
# ❌ Whether AMI IDs are valid

# Can be formatted but invalid?
# YES:
resource "aws_instance" "web" {     # perfectly formatted
  ami           = "ami-123"
  instance_type = "t3.medium"
  subnet_id     = var.subnet_id     # ERROR: var.subnet_id not declared
}                                    # validate catches this, fmt doesn't care
```

**CI/CD Integration:**
```yaml
# GitHub Actions Terraform CI pipeline
- name: Terraform Format Check
  run: terraform fmt -check -recursive
  # Fails if any .tf file is not properly formatted
  # Forces team to format before committing

- name: Terraform Validate
  run: |
    terraform init -backend=false   # init without backend for validation
    terraform validate
  # -backend=false: skip backend init (faster, no credentials needed)

# Pre-commit hook (local enforcement):
# .pre-commit-config.yaml
repos:
- repo: https://github.com/antonbabenko/pre-commit-terraform
  hooks:
  - id: terraform_fmt
  - id: terraform_validate
```

> 💡 **Interview tip:** `terraform fmt` and `terraform validate` serve **completely different purposes** — fmt is cosmetic (style), validate is functional (correctness). Always run them in this order in CI: `fmt -check` first (fast, no AWS needed), then `validate` (requires `init` but no AWS credentials), then `plan` (requires AWS). Mention that `terraform fmt -check` exits with code 1 if formatting is wrong — this fails the CI pipeline and forces the developer to run `terraform fmt` locally before pushing. This enforces consistent code style across the team automatically.

---

### TF-CMD-02 — Terraform | Conceptual | Intermediate

> Explain **`terraform show`**, **`terraform graph`**, and **`terraform console`** — what does each do and give a real-world use case for each.

#### Key Points to Cover:

**`terraform show`:**
```bash
# Shows human-readable output of state file OR saved plan

# Show current state (what Terraform knows is deployed):
terraform show

# Show saved plan (what WILL be done on next apply):
terraform plan -out=tfplan
terraform show tfplan          # review plan in human-readable format
terraform show -json tfplan    # machine-readable for CI/CD processing

# Real use case:
# Before a production apply — share the plan output with a reviewer
# terraform plan -out=prod.tfplan
# terraform show prod.tfplan > plan_review.txt
# → send to team lead for approval before applying
```

**`terraform graph`:**
```bash
# Outputs DOT format dependency graph of all resources

terraform graph                      # show full graph
terraform graph | dot -Tpng > tf.png # visualize as image (requires graphviz)
terraform graph -type=plan           # graph for planned changes only

# What it shows:
# → Which resources depend on which other resources
# → Order Terraform will create/destroy resources
# → Helps debug dependency issues

# Real use case:
# "Why is Terraform creating resource X before Y?"
# → terraform graph → visualize → understand dependency chain
# "We have 500 resources — show what depends on this VPC"
# → terraform graph | grep "aws_vpc" → see all dependencies

# Example output (DOT format):
# digraph {
#   "aws_instance.web" -> "aws_security_group.web"
#   "aws_security_group.web" -> "aws_vpc.main"
# }
```

**`terraform console`:**
```bash
# Interactive REPL for evaluating Terraform expressions
# Access to: variables, locals, functions, resource attributes from state

terraform console

# Inside console — examples:
> var.environment
"production"

> local.common_tags
{
  "Environment" = "production"
  "Team" = "platform"
}

> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"

> length(["a", "b", "c"])
3

> upper("hello world")
"HELLO WORLD"

> jsonencode({"key" = "value"})
"{\"key\":\"value\"}"

# Real use case:
# Testing cidrsubnet() calculations before writing code
# Verifying what a complex local value evaluates to
# Checking what a data source returns
# Debugging: "what does this expression actually produce?"
```

> 💡 **Interview tip:** `terraform console` is the **most underused Terraform command** and knowing it shows real-world experience. It is your Terraform REPL — you can test any expression, function, or reference against your actual state and variables before putting it in code. Especially useful for `cidrsubnet()` calculations (easy to get the math wrong) and `for` expressions (complex transformations). Mention it in interviews when discussing debugging Terraform — it shows you know how to test without running a full plan/apply.

---

### TF-CMD-03 — Terraform | Conceptual | Intermediate

> Explain **`terraform force-unlock`** — what is a state lock, when does a lock get stuck, and how do you safely force-unlock it?
>
> Also explain **`terraform init`** flags — `-upgrade`, `-backend-config`, `-reconfigure`, and `-migrate-state`. When do you need each?

#### Key Points to Cover:

**`terraform force-unlock`:**
```bash
# State lock: DynamoDB item created when apply/plan starts
# Prevents concurrent applies from corrupting state
# Normally auto-released when apply completes or fails

# When lock gets STUCK:
# → terraform apply process killed (Ctrl+C, server crash, CI timeout)
# → DynamoDB item remains → all subsequent plan/apply fail with:
#   "Error: Error acquiring the state lock"
#   "Lock Info: ID: abc123, Operation: apply, Who: user@machine"

# Check who holds lock:
terraform plan    # error message shows lock ID and holder

# Force unlock (use with caution):
terraform force-unlock <LOCK_ID>

# Safety checklist before force-unlock:
# 1. Confirm the original apply process is definitely dead (not just slow)
# 2. Check DynamoDB for the lock: aws dynamodb scan --table-name terraform-locks
# 3. Verify NO other apply is running (check CI/CD pipelines)
# 4. Only then: terraform force-unlock <lock-id>

# WARNING: Never force-unlock while another apply is running
# → Two applies simultaneously → state corruption
```

**`terraform init` flags:**
```bash
# Basic init:
terraform init                    # download providers + modules, init backend

# -upgrade: update providers/modules to latest allowed version
terraform init -upgrade
# Use when: provider has released a new version, you want to update
# Without -upgrade: terraform uses cached versions

# -backend-config: pass backend config at init time (partial backend)
terraform init \
  -backend-config="bucket=company-tf-state" \
  -backend-config="key=prod/terraform.tfstate" \
  -backend-config="region=us-east-1"
# Use when: backend config has secrets (can't commit to Git)
# Main .tf has: terraform { backend "s3" {} }  (empty backend block)

# -reconfigure: force backend re-initialization (ignore existing .terraform/)
terraform init -reconfigure
# Use when: backend config changed and Terraform complains about mismatch

# -migrate-state: move state from old backend to new backend
terraform init -migrate-state
# Use when: switching from local backend to S3 (or between S3 buckets)
# Terraform asks: "Do you want to copy existing state to new backend?"

# -backend=false: skip backend initialization (for CI validation only)
terraform init -backend=false && terraform validate
# Use when: just need to validate syntax, don't need state
```

> 💡 **Interview tip:** `terraform force-unlock` is a **last resort** that should require a second person's approval in production. The correct process: confirm the holding process is dead → check no other apply is running → force unlock. In CI/CD, prevent stuck locks by always setting a CI timeout longer than your longest apply AND using `trap` to run `terraform force-unlock` if the CI job is killed. Also mention **partial backend configuration** (`-backend-config`) as a security best practice — it prevents S3 bucket names and DynamoDB table names from being hardcoded in committed .tf files, allowing different teams to use different state backends with the same module code.

---

## SECTION 2 — VARIABLES, OUTPUTS & TFVARS

---

### TF-VAR-01 — Terraform | Conceptual | Beginner-Intermediate

> Explain **Terraform variable types** — what are the primitive types and complex types? What is **variable validation**? What does **`sensitive = true`** do?
>
> Explain the **tfvars file precedence** — in what order does Terraform load variable values when multiple sources exist?

#### Key Points to Cover:

**Variable types:**
```hcl
# Primitive types:
variable "environment" {
  type    = string
  default = "dev"
}

variable "instance_count" {
  type    = number
  default = 3
}

variable "enable_monitoring" {
  type    = bool
  default = true
}

# Complex types:
variable "allowed_cidrs" {
  type    = list(string)          # ordered, duplicates allowed
  default = ["10.0.0.0/8", "192.168.0.0/16"]
}

variable "tags" {
  type    = map(string)           # key-value pairs, string values
  default = { Environment = "dev", Team = "platform" }
}

variable "db_config" {
  type = object({                  # structured, specific field types
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
  default = {
    instance_class = "db.t3.medium"
    storage_gb     = 100
    multi_az       = false
  }
}

variable "subnet_cidrs" {
  type = set(string)              # unordered, no duplicates
}

variable "instance_sizes" {
  type = tuple([string, number])  # ordered, different types per position
}

# any: accepts any type (avoid — loses type safety)
variable "flexible" {
  type = any
}
```

**Variable validation:**
```hcl
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_count" {
  type = number
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 100
    error_message = "Instance count must be between 1 and 100."
  }
}
```

**`sensitive = true`:**
```hcl
variable "db_password" {
  type      = string
  sensitive = true    # value NEVER shown in plan/apply output
  # Shows: (sensitive value) instead of the actual value
}

output "db_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = false   # safe to show
}

output "db_password_output" {
  value     = aws_db_instance.main.password
  sensitive = true    # hidden in terraform output
}
```

**tfvars precedence (highest to lowest):**
```
1. Command line: -var="key=value"                    ← highest priority
2. Command line: -var-file="custom.tfvars"
3. terraform.tfvars.json (auto-loaded)
4. terraform.tfvars (auto-loaded)
5. *.auto.tfvars.json (alphabetical order)
6. *.auto.tfvars (alphabetical order)
7. Environment variables: TF_VAR_name
8. Variable default in variable block                ← lowest priority

# Practical:
# terraform.tfvars → shared team defaults (committed to Git)
# prod.tfvars → production overrides (applied with -var-file=prod.tfvars)
# .auto.tfvars → auto-applied without -var-file flag
```

> 💡 **Interview tip:** `sensitive = true` **does not encrypt the value** — it only hides it from CLI output and Terraform Cloud UI. The value is still stored in plaintext in the state file. For truly sensitive values (passwords, tokens), combine `sensitive = true` with an encrypted state backend (S3 with SSE-KMS) AND external secret management (AWS Secrets Manager + data source). The `sensitive` flag is a display protection, not a security protection.

---

### TF-VAR-02 — Terraform | Conceptual | Intermediate

> Explain **`terraform_remote_state` data source** — how does it work and when would you use it vs passing values as inputs to modules?
>
> Also explain **`terraform output`** command — how do you use outputs in scripts and CI/CD pipelines?

#### Key Points to Cover:

**`terraform_remote_state`:**
```hcl
# Access outputs from ANOTHER Terraform state

# Use case: networking team manages VPC in one state
#            application team manages EKS in another state
#            EKS needs VPC IDs from networking state

# In EKS state:
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "company-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use the outputs from networking state:
resource "aws_eks_cluster" "main" {
  vpc_config {
    subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
  }
}

# The networking state must have these outputs defined:
output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

**`terraform_remote_state` vs module inputs:**
```
terraform_remote_state:
  ✅ Cross-team: team A reads team B's outputs (different state, different team)
  ✅ No coordination needed to pass values
  ❌ Tight coupling between states (shared outputs become an API contract)
  ❌ If networking state structure changes → breaks app state

Module inputs (preferred when possible):
  ✅ Explicit: you see exactly what data flows between modules
  ✅ Loosely coupled: module doesn't care where values come from
  ✅ Testable: pass mock values in tests
  ❌ Requires root module to fetch and pass values manually
  ❌ Doesn't work across separate Terraform projects/states
```

**`terraform output` command:**
```bash
# Show all outputs:
terraform output

# Show specific output:
terraform output vpc_id

# Machine-readable (for scripts):
terraform output -json                    # full JSON of all outputs
terraform output -raw vpc_id              # raw string value (no quotes)

# Use in CI/CD scripts:
VPC_ID=$(terraform output -raw vpc_id)
echo "VPC ID: $VPC_ID"
kubectl annotate service my-service vpc-id=$VPC_ID

# Use in GitHub Actions:
- name: Get Terraform outputs
  id: tf-outputs
  run: |
    echo "vpc_id=$(terraform output -raw vpc_id)" >> $GITHUB_OUTPUT
    echo "ecr_url=$(terraform output -raw ecr_url)" >> $GITHUB_OUTPUT

- name: Deploy using TF outputs
  run: |
    kubectl set image deployment/app app=${{ steps.tf-outputs.outputs.ecr_url }}:latest
```

> 💡 **Interview tip:** `terraform output -raw` vs `terraform output` (with quotes): `-raw` outputs the plain string value without JSON formatting or surrounding quotes. This is critical for shell scripts — `VPC_ID=$(terraform output vpc_id)` gives you `"vpc-12345"` (with quotes), while `VPC_ID=$(terraform output -raw vpc_id)` gives you `vpc-12345` (without quotes). Always use `-raw` when you need the value in a shell variable. Use `-json` when you need to parse structured outputs (maps, lists) in Python/scripts.

---

## SECTION 3 — BUILT-IN FUNCTIONS

---

### TF-FUNC-01 — Terraform | Conceptual | Intermediate

> Explain the key **Terraform built-in functions** — `lookup()`, `merge()`, `flatten()`, `toset()`, `tolist()`, `tomap()`. Give a real-world use case for each.
>
> Also explain `cidrsubnet()` and `cidrhost()` — why are they important for infrastructure code?

#### Key Points to Cover:

**`lookup()` — map lookup with default:**
```hcl
# Syntax: lookup(map, key, default)
variable "instance_types" {
  default = {
    dev     = "t3.medium"
    staging = "m5.large"
    prod    = "m5.xlarge"
  }
}

resource "aws_instance" "web" {
  instance_type = lookup(var.instance_types, var.environment, "t3.micro")
  # If var.environment = "prod" → "m5.xlarge"
  # If var.environment = "test" → "t3.micro" (default)
}
```

**`merge()` — combine maps:**
```hcl
# Merges multiple maps, later maps override earlier ones
locals {
  common_tags = {
    ManagedBy   = "terraform"
    Company     = "acme"
  }
  env_tags = {
    Environment = var.environment
    CostCenter  = "CC-001"
  }

  all_tags = merge(local.common_tags, local.env_tags)
  # Result: { ManagedBy="terraform", Company="acme", Environment="dev", CostCenter="CC-001" }
}

# Apply merged tags to ALL resources:
resource "aws_instance" "web" {
  tags = merge(local.all_tags, { Name = "web-server" })
}
```

**`flatten()` — flatten nested lists:**
```hcl
# Converts [[1,2], [3,4], [5]] → [1,2,3,4,5]

locals {
  # Each AZ has multiple subnets → need flat list
  az_subnets = [
    ["10.0.1.0/24", "10.0.2.0/24"],   # AZ-a subnets
    ["10.0.3.0/24", "10.0.4.0/24"],   # AZ-b subnets
  ]
  all_subnets = flatten(local.az_subnets)
  # ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24", "10.0.4.0/24"]
}
```

**`toset()`, `tolist()`, `tomap()`:**
```hcl
# toset(): convert list to set (removes duplicates, unordered)
variable "regions" {
  default = ["us-east-1", "us-west-2", "us-east-1"]  # duplicate!
}
locals {
  unique_regions = toset(var.regions)
  # {"us-east-1", "us-west-2"}  (duplicate removed)
}

# Use with for_each (for_each requires set or map):
resource "aws_s3_bucket" "regional" {
  for_each = toset(var.regions)
  bucket   = "company-data-${each.value}"
}

# tolist(): convert set to list (adds ordering)
# tomap(): convert object to map
```

**`cidrsubnet()` — calculate subnet CIDRs:**
```hcl
# Syntax: cidrsubnet(prefix, newbits, netnum)
# prefix:  base CIDR block
# newbits: how many bits to add to prefix
# netnum:  which subnet (0, 1, 2, ...)

locals {
  vpc_cidr = "10.0.0.0/16"

  # Create /24 subnets (add 8 bits to /16)
  public_subnet_cidrs = [
    cidrsubnet(local.vpc_cidr, 8, 1),  # 10.0.1.0/24
    cidrsubnet(local.vpc_cidr, 8, 2),  # 10.0.2.0/24
    cidrsubnet(local.vpc_cidr, 8, 3),  # 10.0.3.0/24
  ]
}

# cidrhost(): get specific host IP from CIDR
# cidrhost("10.0.1.0/24", 1)  →  "10.0.1.1"   (first host)
# cidrhost("10.0.1.0/24", -1) →  "10.0.1.254" (last host)
```

> 💡 **Interview tip:** `toset()` is required for `for_each` because Terraform needs stable keys to track which resource corresponds to which value. A `list` can have duplicates and the same value at different indices — which instance should Terraform update? A `set` has unique values, so `for_each = toset(var.regions)` creates one resource per unique region, keyed by the region name. The most common `toset()` usage: `for_each = toset(["us-east-1", "eu-west-1"])`. Use `terraform console` to test `cidrsubnet()` calculations before writing the code — getting the math wrong silently creates wrong CIDR blocks.

---

## SECTION 4 — PROVIDER CONFIGURATION

---

### TF-PROV-01 — Terraform | Conceptual | Intermediate

> Explain **Terraform version constraints** — what are the operators `=`, `!=`, `>`, `>=`, `<`, `<=`, `~>` and when would you use each?
>
> Also explain `required_providers` in detail — what information goes there and why is it important to pin provider versions?

#### Key Points to Cover:

**Version constraint operators:**
```hcl
terraform {
  required_version = ">= 1.5.0"     # any version >= 1.5.0

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"             # >= 5.0, < 6.0 (most common)
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.20.0, < 3.0.0" # combined constraints
    }
  }
}

# Operator meanings:
# = 5.0.0      exact version only (very restrictive, rarely used)
# != 5.0.0     any version except this one
# > 5.0.0      greater than (not including)
# >= 5.0.0     greater than or equal (minimum version)
# < 6.0.0      less than (upper bound)
# <= 5.99.0    less than or equal
# ~> 5.0       "pessimistic constraint" = >= 5.0, < 6.0
# ~> 5.0.0     "patch-level" = >= 5.0.0, < 5.1.0 (only patches)

# Most common production pattern:
# ~> 5.0  → allows 5.x minor and patch versions, blocks 6.x
# This prevents: accidental major version upgrades (breaking changes)
# This allows:   bug fixes and minor features automatically
```

**Why pin provider versions:**
```
Unpinned (dangerous):
  aws = { source = "hashicorp/aws" }
  → Installs latest (e.g., 6.0.0)
  → 6.0.0 has breaking changes
  → Team member runs terraform init → gets 6.0.0
  → You have 5.0.0 in CI → inconsistent behavior
  → "Works on my machine" problem for infrastructure

Pinned (production standard):
  aws = { source = "hashicorp/aws", version = "~> 5.0" }
  → Everyone gets 5.x
  → .terraform.lock.hcl commits exact version used
  → Upgrades are explicit and reviewed

.terraform.lock.hcl:
  → Generated by terraform init
  → Records EXACT version of every provider installed
  → Must be committed to Git (like package-lock.json)
  → Ensures all team members use identical provider versions
  → Run terraform init -upgrade to update it
```

> 💡 **Interview tip:** The `~>` (pessimistic constraint operator) is the **most important operator to understand**. `~> 5.0` means "at least 5.0, but less than 6.0" — it allows automatic minor updates (5.1, 5.2...) but prevents major version jumps that could have breaking changes. `~> 5.0.0` is even stricter — it allows only patch updates (5.0.1, 5.0.2) and blocks 5.1. Use `~> major.minor` for most cases. And always commit `.terraform.lock.hcl` — without it, `terraform init` on a new machine may install a different provider version causing subtle behavioral differences.

---

### TF-PROV-02 — Terraform | Conceptual | Intermediate

> Explain **provider aliases** in Terraform — what problem do they solve and how do you use them?
>
> Walk through a real-world scenario: deploying resources in **multiple AWS regions** using provider aliases, and deploying resources in **multiple AWS accounts** using assume_role.

#### Key Points to Cover:

**Provider aliases — multiple regions:**
```hcl
# Problem: deploy S3 bucket in us-east-1 AND eu-west-1 in same config

# Configure two AWS providers with aliases:
provider "aws" {
  region = "us-east-1"    # default provider (no alias)
}

provider "aws" {
  alias  = "eu"            # aliased provider
  region = "eu-west-1"
}

# Use default provider (us-east-1):
resource "aws_s3_bucket" "us_bucket" {
  bucket = "company-data-us"
  # no provider argument → uses default provider
}

# Use aliased provider (eu-west-1):
resource "aws_s3_bucket" "eu_bucket" {
  bucket   = "company-data-eu"
  provider = aws.eu          # explicitly reference aliased provider
}

# Passing provider to modules:
module "eu_vpc" {
  source = "./modules/vpc"
  providers = {
    aws = aws.eu             # pass eu provider to module
  }
}
```

**Multiple AWS accounts using assume_role:**
```hcl
# Deploy to different AWS accounts from same Terraform config

provider "aws" {
  region = "us-east-1"
  # Uses default credentials (CI role, env vars)
}

provider "aws" {
  alias  = "prod_account"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::PROD_ACCOUNT_ID:role/TerraformRole"
    # Cross-account role assumption
  }
}

provider "aws" {
  alias  = "security_account"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::SECURITY_ACCOUNT_ID:role/TerraformRole"
  }
}

# Create KMS key in security account:
resource "aws_kms_key" "prod_key" {
  provider    = aws.security_account
  description = "Production encryption key"
}

# Create S3 bucket in prod account, encrypted with security account key:
resource "aws_s3_bucket" "prod_data" {
  provider = aws.prod_account
  bucket   = "prod-data-encrypted"
}
```

> 💡 **Interview tip:** Provider aliases are essential for **multi-region disaster recovery** architectures — you can define primary and DR region providers with aliases, then deploy to both in one `terraform apply`. The key rule: **the default provider (no alias) is used unless you specify `provider = aws.alias`**. When passing providers to modules, use the `providers` map argument — without it, the module gets the default provider, not your aliased one. This is a common bug: forgetting `providers = { aws = aws.eu }` when calling a module and accidentally creating everything in us-east-1 instead of eu-west-1.

---

## SECTION 5 — META-ARGUMENTS (COMPLETE)

---

### TF-META-01 — Terraform | Conceptual | Intermediate

> Explain ALL Terraform **meta-arguments** in one place — `count`, `for_each`, `depends_on`, `lifecycle`, `provider`. What makes meta-arguments different from regular resource arguments?
>
> When should you use `count` vs `for_each`? What is the key limitation of `count` that makes `for_each` often the better choice?

#### Key Points to Cover:

**What meta-arguments are:**
```
Meta-arguments: special arguments available on ALL resources and modules
Regular arguments: specific to each resource type (ami, instance_type)

5 meta-arguments:
  count      → create N copies of a resource
  for_each   → create resources from a map or set
  depends_on → explicit dependency on another resource
  lifecycle  → control resource create/destroy behavior
  provider   → which provider instance to use
```

**`count` vs `for_each`:**
```hcl
# count: create N identical resources, indexed by number
resource "aws_iam_user" "ops" {
  count = 3
  name  = "ops-user-${count.index}"  # ops-user-0, ops-user-1, ops-user-2
}

# PROBLEM with count:
# count = ["alice", "bob", "charlie"]
# alice=index0, bob=index1, charlie=index2
# Remove "bob" from middle → charlie becomes index1 (was index2)
# Terraform: "destroy charlie, recreate as new index" → DATA LOSS RISK!

# for_each: create resources from a map/set, indexed by key
resource "aws_iam_user" "ops" {
  for_each = toset(["alice", "bob", "charlie"])
  name     = each.value    # alice, bob, charlie (keyed by name)
}
# Remove "bob" → only bob destroyed, alice and charlie unchanged ✅

# for_each with map:
variable "users" {
  default = {
    alice   = "admin"
    bob     = "read-only"
    charlie = "admin"
  }
}

resource "aws_iam_user" "team" {
  for_each = var.users
  name     = each.key     # alice, bob, charlie
  tags = {
    Role = each.value     # admin, read-only, admin
  }
}
```

**Complete `lifecycle` block:**
```hcl
resource "aws_db_instance" "production" {
  lifecycle {
    create_before_destroy = true
    # Create new resource first, then destroy old
    # Use: zero-downtime replacement of load balancers, RDS

    prevent_destroy = true
    # Block terraform destroy for this resource
    # Use: databases, critical data stores

    ignore_changes = [
      tags["LastUpdated"],    # AWS sets this automatically
      engine_version,         # RDS auto-upgrades minor versions
    ]
    # ignore_changes = all   # ignore ALL changes (rare, usually wrong)

    replace_triggered_by = [
      aws_launch_template.app.latest_version  # replace instance if LT changes
    ]
    # TF 1.2+: force replace when another resource changes
  }
}
```

> 💡 **Interview tip:** The `count` vs `for_each` difference is one of the most commonly asked Terraform questions at senior level. The key insight: `count` creates resources indexed by **position** (0, 1, 2...), `for_each` creates resources indexed by **identity** (name, ID...). When you remove an item from the MIDDLE of a `count` list, all subsequent resources get renumbered and Terraform proposes destroying and recreating them. With `for_each`, removing an item only affects that specific item — everything else is untouched. **Default rule: prefer `for_each` for almost everything.** Only use `count` when you need truly identical resources with no meaningful name difference.

---

## SECTION 6 — PROVISIONERS

---

### TF-PROV-03 — Terraform | Conceptual | Intermediate

> Explain **Terraform provisioners** — `local-exec`, `remote-exec`, and `file`. Why does HashiCorp recommend **avoiding provisioners** and what are the alternatives?
>
> Also explain **`null_resource`** and **`terraform_data`** (TF 1.4+) — what problem do they solve?

#### Key Points to Cover:

**`local-exec` provisioner:**
```hcl
# Runs command on the machine running Terraform (not the remote resource)
resource "aws_instance" "web" {
  # ... instance config ...

  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> /tmp/instance_ips.txt"
    # self = this resource's attributes
  }

  provisioner "local-exec" {
    when    = destroy              # run on terraform destroy
    command = "echo 'Destroying ${self.id}' | slack-notify"
  }
}
```

**`remote-exec` provisioner:**
```hcl
# Runs commands ON the remote resource (via SSH or WinRM)
resource "aws_instance" "web" {
  # ... instance config ...

  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
      "sudo systemctl start nginx",
    ]
  }
}
```

**Why HashiCorp recommends avoiding provisioners:**
```
Problems with provisioners:
1. Run ONCE at creation — Terraform cannot re-run them on config change
2. Failures leave resources in unknown state (tainted)
3. remote-exec requires SSH access (security concern)
4. Not idempotent — may fail if run twice
5. Connection issues (SSH not ready yet) cause flaky applies

Better alternatives:
  → Cloud-init / user_data: EC2 bootstrapping via metadata service
  → Custom AMIs (Packer): bake configuration into images
  → AWS Systems Manager: manage running instances without SSH
  → Ansible: run separately after Terraform creates infrastructure
  → Kubernetes: manage application deployment separately

"Use provisioners as last resort when no better alternative exists"
— HashiCorp documentation
```

**`null_resource` and `terraform_data`:**
```hcl
# null_resource: fake resource with no real infrastructure
# Use: attach provisioners, trigger re-runs via triggers

resource "null_resource" "cluster_setup" {
  triggers = {
    # Re-run provisioner whenever cluster_id changes
    cluster_id = aws_eks_cluster.main.id
    # Also re-run if this value changes:
    script_hash = filemd5("${path.module}/setup.sh")
  }

  provisioner "local-exec" {
    command = "bash setup.sh ${aws_eks_cluster.main.endpoint}"
  }

  depends_on = [aws_eks_cluster.main]
}

# terraform_data (TF 1.4+ replacement for null_resource):
resource "terraform_data" "cluster_setup" {
  triggers_replace = [
    aws_eks_cluster.main.id,
    filemd5("${path.module}/setup.sh")
  ]

  provisioner "local-exec" {
    command = "bash setup.sh ${aws_eks_cluster.main.endpoint}"
  }
}
# terraform_data is preferred over null_resource (built-in, no provider needed)
```

> 💡 **Interview tip:** The key `null_resource` (and `terraform_data`) pattern: **`triggers` map** causes re-execution whenever any trigger value changes. This is how you run scripts conditionally — if `triggers.script_hash = filemd5("setup.sh")` and you change `setup.sh`, the hash changes, Terraform sees a diff in the trigger, destroys and recreates the null_resource, and runs the provisioner again. Without triggers, the provisioner only runs ONCE at initial creation and never again. Mention `terraform_data` as the modern replacement — it was introduced in TF 1.4 specifically to replace `null_resource` without requiring the `hashicorp/null` provider.

---

## SECTION 7 — TESTING & CODE QUALITY

---

### TF-TEST-01 — Terraform | Conceptual | Intermediate

> Explain the **Terraform testing ecosystem** — what are `terraform test` (native), **Checkov**, **tfsec**, and **Infracost**? What does each tool check and where in the CI/CD pipeline does each run?

#### Key Points to Cover:

**Native `terraform test` (TF 1.6+):**
```hcl
# tests/main.tftest.hcl
run "creates_s3_bucket" {
  command = plan    # or apply

  variables {
    bucket_name = "test-bucket-${run.id}"
    environment = "test"
  }

  assert {
    condition     = aws_s3_bucket.main.bucket == "test-bucket-${run.id}"
    error_message = "Bucket name does not match expected value"
  }

  assert {
    condition     = aws_s3_bucket_versioning.main.versioning_configuration[0].status == "Enabled"
    error_message = "Versioning must be enabled"
  }
}

# Run tests:
terraform test               # runs all .tftest.hcl files
terraform test -filter=tests/main.tftest.hcl
```

**Checkov — static analysis for security:**
```bash
# Scans Terraform code for security misconfigurations
pip install checkov
checkov -d .                    # scan current directory

# What it checks:
# S3 bucket has public access blocked?
# Security groups not open to 0.0.0.0/0 on sensitive ports?
# RDS has deletion protection enabled?
# EBS volumes encrypted?
# CloudTrail enabled?
# 1000+ security policies built-in

# CI/CD integration:
- name: Checkov Security Scan
  run: |
    checkov -d . \
      --output cli \
      --output junitxml \
      --output-file-path console,results.xml \
      --soft-fail   # warn but don't fail pipeline (or remove for hard fail)
```

**tfsec — security scanner (fast, Go-based):**
```bash
# Similar to Checkov but faster (written in Go)
brew install tfsec
tfsec .

# Checks:
# Open security groups
# Unencrypted resources
# Missing tags
# Hardcoded credentials
# Specific AWS, Azure, GCP rules

# Both Checkov AND tfsec are often run together
# They catch different issues
```

**Infracost — cost estimation:**
```bash
# Shows estimated AWS cost BEFORE applying
brew install infracost
infracost auth login
infracost breakdown --path .

# Output example:
# aws_instance.web          $69.21/month
# aws_db_instance.main     $186.42/month
# aws_nat_gateway.main      $32.40/month
# TOTAL:                   $288.03/month

# In CI/CD (post PR comment with cost diff):
- name: Infracost cost estimate
  uses: infracost/actions/setup@v2
  with:
    api-key: ${{ secrets.INFRACOST_API_KEY }}

- name: Comment PR with cost
  run: |
    infracost diff --path . \
      --format json \
      --out-file /tmp/infracost.json
    infracost comment github \
      --path /tmp/infracost.json \
      --github-token ${{ github.token }} \
      --pull-request ${{ github.event.pull_request.number }}
```

**Complete CI/CD quality pipeline:**
```
PR created:
  1. terraform fmt -check     (style — fastest, no AWS needed)
  2. terraform validate        (syntax — no AWS needed)
  3. checkov -d .             (security — no AWS needed)
  4. tfsec .                  (security — no AWS needed)
  5. terraform plan           (requires AWS — full diff)
  6. infracost diff           (cost estimate — no AWS needed)

Merge to main:
  7. terraform apply          (with approval gate)
```

> 💡 **Interview tip:** The **CI/CD ordering** of these tools matters for speed. `fmt`, `validate`, `checkov`, `tfsec`, and `infracost` all run WITHOUT AWS credentials — they are pure static analysis. Put them first so PR authors get fast feedback (30 seconds) without waiting for `terraform plan` (2-3 minutes with AWS API calls). The rule: "fail fast, fail cheap." Also mention **pre-commit hooks** for local enforcement — running `terraform fmt` and `checkov` before every commit prevents most issues from ever reaching CI.

---

## SECTION 8 — COMPLETE COMMAND REFERENCE

---

### TF-REF-01 — Terraform | Reference | All Levels

> Complete reference of **every `terraform` command** with when to use each. This is the cheat sheet for interviews.

#### Key Points to Cover:

```bash
# ═══════════════════════════════════════════════════
# INITIALIZATION
# ═══════════════════════════════════════════════════
terraform init                    # download providers + modules, init backend
terraform init -upgrade           # update providers to latest allowed version
terraform init -backend=false     # init without backend (for CI validation)
terraform init -backend-config="key=prod/tfstate" # partial backend config
terraform init -reconfigure       # force re-init (backend changed)
terraform init -migrate-state     # move state to new backend

# ═══════════════════════════════════════════════════
# CODE QUALITY
# ═══════════════════════════════════════════════════
terraform fmt                     # format .tf files in current directory
terraform fmt -recursive          # format all subdirectories too
terraform fmt -check              # check only, exit 1 if changes needed (CI)
terraform fmt -diff               # show diff, don't write
terraform validate                # validate syntax and internal consistency
terraform validate -json          # JSON output for CI parsing

# ═══════════════════════════════════════════════════
# PLANNING & APPLYING
# ═══════════════════════════════════════════════════
terraform plan                    # show changes (no apply)
terraform plan -out=tfplan        # save plan to file
terraform plan -target=aws_instance.web  # plan only this resource
terraform plan -var="env=prod"    # pass variable value
terraform plan -var-file=prod.tfvars     # load variable file
terraform plan -refresh=false     # skip AWS API refresh (faster, risks missing drift)
terraform plan -destroy           # plan a destroy (what would be removed)

terraform apply                   # apply changes (asks for confirmation)
terraform apply -auto-approve     # apply without confirmation (CI/CD only)
terraform apply tfplan            # apply saved plan (exact, no surprises)
terraform apply -target=aws_instance.web  # apply only this resource
terraform apply -replace=aws_instance.web # force replacement of resource

terraform destroy                 # destroy all resources
terraform destroy -target=aws_instance.web  # destroy only this resource
terraform destroy -auto-approve   # no confirmation (DANGEROUS)

# ═══════════════════════════════════════════════════
# STATE MANAGEMENT
# ═══════════════════════════════════════════════════
terraform state list              # list all resources in state
terraform state show aws_instance.web     # show details of one resource
terraform state mv old_name new_name      # rename resource in state
terraform state rm aws_instance.old       # remove resource from state (unmanage)
terraform state pull              # download remote state to stdout
terraform state push terraform.tfstate    # upload local state to remote

terraform import aws_instance.web i-1234  # import existing resource to state
terraform refresh                  # update state from real infrastructure
terraform force-unlock <lock-id>   # release stuck state lock

# ═══════════════════════════════════════════════════
# INSPECTION
# ═══════════════════════════════════════════════════
terraform show                    # show current state in human-readable format
terraform show tfplan             # show saved plan
terraform show -json tfplan       # show saved plan as JSON
terraform output                  # show all outputs
terraform output vpc_id           # show specific output
terraform output -raw vpc_id      # raw value (no quotes, for shell scripts)
terraform output -json            # all outputs as JSON
terraform graph                   # output dependency graph (DOT format)
terraform graph | dot -Tpng > g.png  # render as image
terraform console                 # interactive REPL for expressions
terraform version                 # show Terraform version and provider versions
terraform providers               # show required providers and versions

# ═══════════════════════════════════════════════════
# TESTING
# ═══════════════════════════════════════════════════
terraform test                    # run all .tftest.hcl test files (TF 1.6+)
terraform test -filter=tests/     # run specific test files
```

> 💡 **Interview tip:** When an interviewer asks "walk me through your Terraform workflow," the senior answer covers ALL phases: **code quality** (`fmt` → `validate` → `checkov`) → **planning** (`plan -out` for production) → **review** (`show tfplan` to share with team) → **apply** (`apply tfplan` for exact execution) → **verification** (`output`, `state list`). Knowing the difference between `terraform refresh` (update state from real infra) and `terraform plan -refresh=false` (skip refresh for speed) shows operational maturity. And always mention: **`-auto-approve` should ONLY exist in CI/CD pipelines, never run manually by a human** in production.

---

## SECTION 9 — NEW TERRAFORM FEATURES

---

### TF-NEW-01 — Terraform | Conceptual | Advanced

> Explain the recent **Terraform features** introduced in versions 1.5, 1.6, 1.7, and 1.8. What are **import blocks** (1.5), the **`terraform test` framework** (1.6), **ephemeral resources** (1.10), and the **`check` block** (1.5)?

#### Key Points to Cover:

**TF 1.5 — Import blocks (declarative imports):**
```hcl
# Old way: terraform import aws_s3_bucket.logs my-bucket (CLI command)
# New way: declare import in .tf file (reviewed in PRs, version controlled)

import {
  id = "my-existing-bucket"
  to = aws_s3_bucket.logs
}

resource "aws_s3_bucket" "logs" {
  bucket = "my-existing-bucket"
  # ... rest of config ...
}

# Even better: terraform plan -generate-config-out=generated.tf
# → Terraform generates the resource block automatically!
# → Fix any issues → add to your .tf files → import block applies cleanly
```

**TF 1.5 — `check` block (continuous assertions):**
```hcl
# check blocks run assertions on EVERY plan/apply (not just creation)
# Unlike validation (runs on variables), checks run on real resource attributes

check "s3_bucket_public_access" {
  data "aws_s3_bucket" "verify" {
    bucket = aws_s3_bucket.main.id
  }

  assert {
    condition     = data.aws_s3_bucket.verify.bucket_domain_name != null
    error_message = "S3 bucket domain name should be set"
  }
}

check "website_is_healthy" {
  data "http" "website" {
    url = "https://${aws_cloudfront_distribution.main.domain_name}"
  }
  assert {
    condition     = data.http.website.status_code == 200
    error_message = "Website returned non-200 status: ${data.http.website.status_code}"
  }
}
# Failed checks: WARN but do NOT fail the apply (unlike precondition)
```

**TF 1.6 — Native test framework:**
```hcl
# .tftest.hcl files — proper unit and integration testing for modules
run "verify_s3_configuration" {
  command = plan
  assert {
    condition     = aws_s3_bucket_versioning.main.versioning_configuration[0].status == "Enabled"
    error_message = "Versioning must be enabled in production"
  }
}
```

**TF 1.10 — Ephemeral resources (write-only values):**
```hcl
# Ephemeral resources: values used during apply but NOT stored in state
# Perfect for secrets that should never persist in state file

ephemeral "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db.id
}

resource "aws_db_instance" "main" {
  password = ephemeral.aws_secretsmanager_secret_version.db_password.secret_string
  # Password used during apply but NEVER written to terraform.tfstate
}
# Security improvement: secrets in Secrets Manager → Terraform → resource
# WITHOUT the secret ever touching the state file
```

> 💡 **Interview tip:** The **import block** feature (TF 1.5) is one of the most practically impactful recent additions. Combined with `terraform plan -generate-config-out=generated.tf`, it enables fully automated IaC adoption for existing infrastructure. Previously, running `terraform import` required manually writing the entire resource block first — now Terraform generates it for you. Mention this when discussing "migrating existing infrastructure to Terraform" — it cuts the migration effort by 80%. The **ephemeral resources** feature solves the long-standing security criticism of Terraform: "passwords in state files." Ephemeral values are used during apply and immediately discarded.

---

*Terraform Complete Commands & Concepts Reference*
*Zero overlap with Q1–Q675 — fills all identified Terraform gaps*
*DevOps / SRE Interview Preparation — nawab312 GitHub repositories*
