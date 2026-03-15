# Linux & Bash — Complete Gaps Reference
## All Missing Topics from Q1–Q675 | With Key Points + Interview Tips

> **What this file covers:**
> Every Linux/Bash topic missing or only mentioned in passing across all 675 questions.
> Includes: File Transfer, User Management, Bash Scripting Fundamentals,
> Text Processing, find command, Networking gaps, System Admin gaps, and more.

---

## COMPLETE TOPIC INDEX

| Section | Topics |
|---|---|
| **1. File Transfer** | scp vs rsync, tar+SSH, netcat, bbcp, pv, S3 |
| **2. User Management + Root Access** | useradd, usermod, wheel/sudo, NOPASSWD, visudo |
| **3. Bash — Variables** | declare, local, export, scope, types |
| **4. Bash — Loops** | for, while, until, break, continue |
| **5. Bash — Arrays** | indexed arrays, associative arrays |
| **6. Bash — Functions** | define, return values, local scope |
| **7. Bash — String Manipulation** | ${var#}, ${var%}, ${var/}, ${#var}, substrings |
| **8. Bash — Parameter Expansion** | ${var:-default}, ${var:?}, ${var:+}, ${var:=} |
| **9. Bash — trap & Signals** | EXIT, ERR, SIGINT, SIGTERM, cleanup |
| **10. Bash — getopts** | argument parsing, flags, options |
| **11. Bash — Regex =~** | pattern matching, capture groups |
| **12. Bash — IFS** | parsing files, CSV, custom delimiters |
| **13. Bash — Subshell vs Current Shell** | ( ) vs { } vs source |
| **14. Bash — Parallel Execution** | &, wait, job control, background |
| **15. Bash — Process/Command Substitution** | $(), <(), >() |
| **16. Bash — curl + jq in Scripts** | API calls, JSON parsing |
| **17. Bash — Lockfile/flock** | prevent duplicate runs |
| **18. find command** | -name, -type, -mtime, -size, -exec |
| **19. Text Processing** | sort, tr, tee, diff, regex BRE vs ERE |
| **20. Networking Gaps** | TCP vs UDP, firewalld/ufw, bonding |
| **21. ACLs** | setfacl, getfacl, extended permissions |
| **22. System Admin Gaps** | .bashrc vs .profile, env vars, swap, iperf |
| **23. Job Control** | jobs, fg, bg, nohup, disown, sar |

---

## SECTION 1 — FILE TRANSFER

### Q-LX-01 — Linux | Scenario-Based | Intermediate

> You need to transfer a **50GB database dump** from Server A to Server B.
> A junior engineer says: *"I'll use `scp`."*
>
> **Why is `scp` not the best tool for large files? What are the better
> alternatives and when would you use each?**

#### Key Points to Cover:

**Why `scp` is NOT ideal for large files:**
```
scp problems:
1. No resume — network drops at 49GB → restart from 0%
2. Single TCP stream — cannot saturate 10Gbps link
3. No progress detail — no ETA, no MB/s history
4. No delta sync — re-transfers entire file even if 99% identical
5. No compression tuning or bandwidth throttling
```

**`rsync` — BEST for large files (resumable):**
```bash
# Basic with progress and resume:
rsync -avhP /data/backup.sql.gz user@serverB:/backup/
# -a = archive (preserve permissions, timestamps)
# -v = verbose
# -h = human readable sizes
# -P = --partial --progress (resume + progress bar)

# With compression (great for SQL/text, huge speedup):
rsync -avhPz /data/database.sql user@serverB:/data/

# Bandwidth throttle (don't saturate network):
rsync -avhP --bwlimit=50000 /data/file user@serverB:/data/
# 50000 = 50MB/s limit

# Dry run first (see what WILL transfer):
rsync -avhPn /data/ user@serverB:/data/

# WHY rsync beats scp: --partial saves partial file on failure
# Next run = resumes from checkpoint, not from 0%
```

**`tar` piped over SSH — streaming, no temp file:**
```bash
# Streams directly: no intermediate file created
tar czf - /data/large-dir/ | ssh user@serverB "tar xzf - -C /backup/"

# With progress (using pv):
tar czf - /data/large-dir/ \
  | pv -s $(du -sb /data/large-dir/ | awk '{print $1}') \
  | ssh user@serverB "tar xzf - -C /backup/"
# Shows: [=====>  ] 23GiB 45:32 [87MiB/s] ETA 0:12:45

# Faster: parallel gzip (all CPU cores):
tar cf - /data/ | pigz | pv | ssh user@serverB "pigz -d | tar xf - -C /backup/"
```

**`netcat` — fastest raw transfer (no encryption):**
```bash
# Receiver first:
nc -l -p 9999 | tar xzf - -C /backup/

# Sender:
tar czf - /data/ | nc serverB 9999

# Fastest possible — no SSH overhead
# ONLY use on trusted internal networks (no encryption)
```

**`bbcp` — parallel streams for wire-speed:**
```bash
# Multiple TCP streams — saturates 10Gbps links
bbcp -P 16 /data/50gb.sql user@serverB:/backup/
# -P 16 = 16 parallel TCP streams
# Single stream: ~1-2 Gbps on 10Gbps link
# 16 streams: 8-9 Gbps on 10Gbps link
```

**`pv` — progress bar on any command:**
```bash
pv /file | ssh dest "cat > /dest/file"
# Shows real-time speed, ETA for any transfer

# Check achievable throughput BEFORE starting:
# Dest: nc -l 9999 > /dev/null
# Src:  dd if=/dev/zero bs=1M count=100 | pv | nc dest 9999
```

**S3 as transfer medium (cross-cloud/no direct path):**
```bash
# Source:
aws s3 cp /data/50gb.sql s3://transfer-bucket/dump.sql

# Destination:
aws s3 cp s3://transfer-bucket/dump.sql /backup/dump.sql

# Cleanup:
aws s3 rm s3://transfer-bucket/dump.sql
```

**Decision matrix:**
```
Same DC, trusted network     → netcat        (fastest, no encryption)
Same DC, needs encryption    → rsync         (fast + resumable + secure)
Unreliable network           → rsync -P      (resumes on failure)
Text/SQL (compressible)      → rsync -z      (compression = less data)
Near wire-speed 10Gbps       → bbcp -P 16    (parallel streams)
Cross-cloud / no direct path → S3 staging    (works anywhere)
Streaming, no temp file      → tar | ssh     (direct pipe)
Directory sync (ongoing)     → rsync         (delta sync)
```

> 💡 **Interview tip:** The reason `scp` is NOT the best answer: **no resume capability**. Drop at 49GB → restart from 0%. `rsync -P` saves partial file and resumes from checkpoint. This is the #1 practical difference. The interviewer likely wanted `rsync`. For bonus: mention `pv` for monitoring, `pigz` for parallel compression, `bbcp` for near-wire-speed. Always test throughput first with `netcat + dd` before committing to a large transfer estimate.

---

## SECTION 2 — USER MANAGEMENT + ROOT ACCESS

### Q-LX-02 — Linux | Conceptual | Beginner-Intermediate

> How do you **create a new user** and **give them root access** in Linux?
>
> Explain 3 ways to give root-level privileges. What is the difference between
> `sudo` group and `wheel` group? What is `NOPASSWD` and when is it appropriate?

#### Key Points to Cover:

**Step 1 — Create user:**
```bash
# Full creation with home dir and bash shell:
useradd -m -s /bin/bash siddharth
# -m = create home directory /home/siddharth
# -s = set login shell

# Set password (required for login):
passwd siddharth

# Alternative (Ubuntu/Debian — interactive, friendlier):
adduser siddharth

# Verify:
id siddharth
# uid=1001(siddharth) gid=1001(siddharth) groups=1001(siddharth)
```

**Step 2 — Give root access (3 methods):**

**Method 1 — sudo group (Ubuntu/Debian):**
```bash
usermod -aG sudo siddharth
# -a = APPEND (CRITICAL — without -a, replaces ALL groups)
# -G = supplementary groups

# /etc/sudoers already has: %sudo ALL=(ALL:ALL) ALL
# User can now: sudo apt install nginx
```

**Method 2 — wheel group (RHEL/CentOS/Amazon Linux):**
```bash
usermod -aG wheel siddharth
# /etc/sudoers already has: %wheel ALL=(ALL) ALL
# Same concept, different group name per distro
```

**Method 3 — visudo (fine-grained control):**
```bash
visudo    # ALWAYS use visudo — validates syntax before saving!
          # Direct edit of /etc/sudoers = risk of lockout on typo

# Add line:
siddharth ALL=(ALL:ALL) ALL

# Or specific commands only (more secure):
siddharth ALL=(ALL) /usr/bin/systemctl restart nginx

# Best practice — separate file per user:
echo "siddharth ALL=(ALL:ALL) ALL" > /etc/sudoers.d/siddharth
chmod 440 /etc/sudoers.d/siddharth
```

**sudo group vs wheel group:**
```
sudo group:  Ubuntu, Debian, Mint
wheel group: RHEL, CentOS, Fedora, Amazon Linux, Rocky Linux

Same purpose, different name per distro.
Always check: grep sudo /etc/sudoers  AND  grep wheel /etc/sudoers
```

**NOPASSWD — passwordless sudo:**
```bash
# In /etc/sudoers or /etc/sudoers.d/:
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart app

# When to use:
# ✅ CI/CD deploy accounts (automated, can't type password)
# ✅ Service accounts needing specific system commands
# ❌ Regular human accounts (removes security layer)
# ❌ NOPASSWD: ALL for humans (full root, no friction = dangerous)
```

**Verify, revoke, lock:**
```bash
# Verify access:
su - siddharth
sudo whoami    # prints: root

# Revoke sudo:
gpasswd -d siddharth sudo

# Lock user (cannot login):
usermod -L siddharth

# Delete user + home:
userdel -r siddharth

# Who has sudo access:
getent group sudo
cat /etc/sudoers | grep -v "^#"
ls /etc/sudoers.d/
```

> 💡 **Interview tip:** The most important detail: `usermod -aG` vs `usermod -G`. Without `-a` (append), you **replace ALL existing groups** — user loses docker access, other groups, everything. Always use `-aG`. Also: syntax error in `/etc/sudoers` = ALL sudo access locked out. `visudo` validates before saving. Never edit sudoers directly with vim/nano.

---

## SECTION 3 — BASH VARIABLES

### Q-LX-03 — Bash | Conceptual | Intermediate

> Explain **Bash variables** — what are the different types, what is the difference
> between `local`, `export`, and `declare`? What is variable **scope** in Bash?

#### Key Points to Cover:

**Variable basics:**
```bash
# Assignment (no spaces around =):
name="siddharth"       # string
count=42               # number (but still stored as string in bash)
pi=3.14

# Access with $:
echo $name
echo ${name}           # preferred in complex expressions
echo "Hello ${name}!"

# Unset variable = empty string (no error by default)
echo $undefined        # prints nothing

# set -u catches undefined variables:
set -u
echo $undefined        # Error: unbound variable
```

**Variable scope:**
```bash
# Global (default) — accessible everywhere in script:
GLOBAL_VAR="I am global"

function show() {
    echo $GLOBAL_VAR    # works — global is accessible inside function
    local LOCAL_VAR="only inside this function"
    echo $LOCAL_VAR     # works inside function
}

show
echo $LOCAL_VAR         # empty — local var not accessible outside function!

# local — restrict variable to function only:
function calculate() {
    local result=$((5 + 3))    # only exists inside calculate()
    echo $result
}
```

**`export` — environment variables:**
```bash
# export makes variable available to CHILD PROCESSES
MY_VAR="hello"
export MY_VAR          # now child processes can see it

# Or set and export in one line:
export DATABASE_URL="postgresql://localhost:5432/mydb"

# Verify exported variables:
env | grep DATABASE
printenv DATABASE_URL

# Remove export:
unset DATABASE_URL

# Common exported variables:
export PATH="$PATH:/usr/local/bin"    # add to PATH
export JAVA_HOME="/usr/lib/jvm/java-11"
```

**`declare` — explicit type declaration:**
```bash
declare -i count=0          # integer (arithmetic auto-applied)
declare -r CONSTANT="fixed" # readonly (cannot change)
declare -l lower_var        # auto-lowercase value
declare -u upper_var        # auto-uppercase value
declare -x exported_var     # same as export
declare -a my_array         # indexed array
declare -A my_map           # associative array

# Readonly without declare:
readonly CONFIG_FILE="/etc/app.conf"
CONFIG_FILE="other"         # Error: readonly variable
```

**Special variables:**
```bash
$0    # script name
$1    # first argument
$2    # second argument
$@    # all arguments as separate words
$*    # all arguments as single string
$#    # number of arguments
$?    # exit code of last command
$$    # PID of current script
$!    # PID of last background command
```

> 💡 **Interview tip:** The most common bash variable bug: **forgetting `local` in functions**. Without `local`, all function variables are global and pollute the script's namespace. Production scripts always use `local` for every function variable. Also: variable names — use UPPERCASE for environment variables (exported), lowercase for local script variables. This convention makes it obvious what's an env var vs local var.

---

## SECTION 4 — BASH LOOPS

### Q-LX-04 — Bash | Conceptual | Intermediate

> Explain the **3 types of loops** in Bash — `for`, `while`, `until`.
> Give practical examples of each including `break`, `continue`, and
> looping over files, arrays, and command output.

#### Key Points to Cover:

**`for` loop — iterate over list:**
```bash
# List of values:
for color in red green blue; do
    echo "Color: $color"
done

# Range of numbers:
for i in {1..10}; do
    echo "Number: $i"
done

# Range with step:
for i in {0..100..10}; do    # 0, 10, 20, ... 100
    echo $i
done

# C-style for loop:
for ((i=0; i<5; i++)); do
    echo "i = $i"
done

# Loop over files:
for file in /var/log/*.log; do
    echo "Processing: $file"
    gzip "$file"
done

# Loop over array:
servers=("web01" "web02" "db01")
for server in "${servers[@]}"; do
    echo "Pinging $server"
    ping -c 1 "$server"
done

# Loop over command output:
for pid in $(pgrep nginx); do
    echo "nginx PID: $pid"
done

# Loop over lines in file:
while IFS= read -r line; do    # safer than for loop for files
    echo "Line: $line"
done < /etc/hosts
```

**`while` loop — run while condition is true:**
```bash
# Counter:
count=0
while [ $count -lt 5 ]; do
    echo "Count: $count"
    ((count++))
done

# Read file line by line (correct way):
while IFS= read -r line; do
    echo "$line"
done < input.txt

# Wait for service to be ready:
while ! curl -s http://localhost:8080/health > /dev/null; do
    echo "Waiting for service..."
    sleep 2
done
echo "Service is up!"

# Infinite loop with break:
while true; do
    read -p "Enter 'quit' to exit: " input
    if [ "$input" = "quit" ]; then
        break
    fi
    echo "You entered: $input"
done
```

**`until` loop — run until condition is true:**
```bash
# Opposite of while (loop until condition becomes TRUE)
count=0
until [ $count -ge 5 ]; do
    echo "Count: $count"
    ((count++))
done

# Wait for file to exist:
until [ -f /tmp/ready.flag ]; do
    echo "Waiting for flag file..."
    sleep 1
done
echo "Flag file found!"
```

**`break` and `continue`:**
```bash
# break — exit loop immediately:
for i in {1..10}; do
    if [ $i -eq 5 ]; then
        break       # stop loop when i=5
    fi
    echo $i         # prints 1 2 3 4
done

# continue — skip current iteration:
for i in {1..10}; do
    if [ $((i % 2)) -eq 0 ]; then
        continue    # skip even numbers
    fi
    echo $i         # prints 1 3 5 7 9
done

# break from nested loops (with level number):
for i in {1..3}; do
    for j in {1..3}; do
        if [ $j -eq 2 ]; then
            break 2    # break out of BOTH loops
        fi
        echo "$i $j"
    done
done
```

> 💡 **Interview tip:** The most important loop pattern for DevOps: **`while IFS= read -r line`** for processing files line by line. The `IFS=` prevents stripping leading/trailing whitespace, and `-r` prevents backslash interpretation. Using `for line in $(cat file)` is WRONG — it breaks on spaces within lines. Always use `while read` for file processing. This pattern comes up constantly in log processing scripts.

---

## SECTION 5 — BASH ARRAYS

### Q-LX-05 — Bash | Conceptual | Intermediate

> Explain **indexed arrays** and **associative arrays** in Bash.
> Show common operations: create, add, access, loop, delete, length.

#### Key Points to Cover:

**Indexed arrays (ordered, integer index):**
```bash
# Create:
fruits=("apple" "banana" "cherry")
# OR:
declare -a fruits
fruits[0]="apple"
fruits[1]="banana"

# Access:
echo ${fruits[0]}          # apple
echo ${fruits[1]}          # banana
echo ${fruits[-1]}         # cherry (last element)

# All elements:
echo ${fruits[@]}          # apple banana cherry
echo ${fruits[*]}          # same but as single string (less safe)

# Number of elements:
echo ${#fruits[@]}         # 3

# Length of specific element:
echo ${#fruits[0]}         # 5 (length of "apple")

# All indices:
echo ${!fruits[@]}         # 0 1 2

# Loop:
for fruit in "${fruits[@]}"; do
    echo "Fruit: $fruit"
done

# Add element:
fruits+=("date")           # append
fruits[10]="elderberry"    # sparse index

# Delete element:
unset fruits[1]            # removes banana (index stays!)

# Slice (elements 1 and 2):
echo ${fruits[@]:1:2}      # banana cherry

# Common DevOps use:
servers=("web01" "web02" "web03" "db01")
failed=()
for server in "${servers[@]}"; do
    if ! ping -c 1 "$server" &>/dev/null; then
        failed+=("$server")
    fi
done
echo "Failed servers: ${failed[@]}"
```

**Associative arrays (key-value, like dictionary):**
```bash
# Must declare explicitly:
declare -A config

# Set values:
config["host"]="localhost"
config["port"]="5432"
config["user"]="admin"
config["db"]="production"

# Or create all at once:
declare -A config=(
    [host]="localhost"
    [port]="5432"
    [user]="admin"
)

# Access:
echo ${config["host"]}     # localhost
echo ${config[port]}       # 5432 (quotes optional for simple keys)

# All keys:
echo ${!config[@]}         # host port user db

# All values:
echo ${config[@]}          # localhost 5432 admin production

# Loop over key-value pairs:
for key in "${!config[@]}"; do
    echo "$key = ${config[$key]}"
done

# Check if key exists:
if [[ -v config["host"] ]]; then
    echo "host is set"
fi

# Delete entry:
unset config["user"]

# Common DevOps use:
declare -A service_ports=(
    [nginx]=80
    [postgres]=5432
    [redis]=6379
    [prometheus]=9090
)

for service in "${!service_ports[@]}"; do
    port=${service_ports[$service]}
    if nc -z localhost $port 2>/dev/null; then
        echo "✅ $service (port $port) is UP"
    else
        echo "❌ $service (port $port) is DOWN"
    fi
done
```

> 💡 **Interview tip:** Always use `"${array[@]}"` (with double quotes) when iterating — not `${array[@]}` (without quotes). Without quotes, array elements with spaces split incorrectly. `"${array[@]}"` preserves each element as a separate word regardless of spaces. The difference: `files=("file one.txt" "file two.txt")` — `for f in ${files[@]}` gives 4 iterations (splits on space). `for f in "${files[@]}"` gives 2 iterations (correct).

---

## SECTION 6 — BASH FUNCTIONS

### Q-LX-06 — Bash | Conceptual | Intermediate

> Explain **functions** in Bash — how to define them, pass arguments,
> return values, and manage scope. What is the difference between
> `return` and `echo` for returning values?

#### Key Points to Cover:

**Define and call functions:**
```bash
# Syntax 1 (preferred):
function greet() {
    echo "Hello, $1!"
}

# Syntax 2 (POSIX compatible):
greet() {
    echo "Hello, $1!"
}

# Call:
greet "Siddharth"    # Hello, Siddharth!

# Arguments inside function:
# $1, $2, $3... = positional arguments
# $@ = all arguments
# $# = number of arguments
# $0 = script name (NOT function name)
```

**Return values — two methods:**
```bash
# Method 1: return (only returns EXIT CODE 0-255)
is_running() {
    pgrep "$1" > /dev/null
    return $?    # 0 = found, 1 = not found
}

if is_running "nginx"; then
    echo "nginx is running"
fi

# Method 2: echo (return actual data — most common)
get_ip() {
    hostname -I | awk '{print $1}'
}

my_ip=$(get_ip)     # capture with command substitution
echo "My IP: $my_ip"

# WRONG — this does NOT return a value, sets global:
get_value() {
    result="some value"    # modifies global variable!
}
```

**Local scope in functions:**
```bash
counter=0

increment() {
    local counter=10    # local — does NOT affect global counter
    echo "Inside function: $counter"   # 10
}

increment
echo "Outside function: $counter"     # 0 (unchanged)

# Without local:
increment_bad() {
    counter=10    # modifies GLOBAL counter
}
increment_bad
echo "After bad increment: $counter"  # 10 (global changed!)
```

**Production function template:**
```bash
#!/usr/bin/env bash
set -euo pipefail

# Logging function used throughout script:
log() {
    local level="$1"
    local message="$2"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "${timestamp} [${level}] ${message}" | tee -a /var/log/deploy.log
    [[ "$level" == "ERROR" ]] && echo "${timestamp} [${level}] ${message}" >&2
}

# Function with validation:
create_backup() {
    local source_dir="$1"
    local dest_dir="$2"
    local timestamp
    timestamp=$(date +%Y%m%d_%H%M%S)

    # Validate arguments:
    if [[ $# -ne 2 ]]; then
        log "ERROR" "Usage: create_backup <source> <dest>"
        return 1
    fi

    if [[ ! -d "$source_dir" ]]; then
        log "ERROR" "Source directory does not exist: $source_dir"
        return 1
    fi

    local backup_file="${dest_dir}/backup_${timestamp}.tar.gz"
    tar czf "$backup_file" "$source_dir" 2>/dev/null
    log "INFO" "Backup created: $backup_file"
    echo "$backup_file"    # return the backup path
}

# Usage:
backup_path=$(create_backup /var/app /backups)
log "INFO" "Backup saved to: $backup_path"
```

> 💡 **Interview tip:** The `echo` vs `return` confusion is the #1 function mistake in Bash interviews. `return` can ONLY return a number (0-255) — it's an exit status, not a value. To return actual data (strings, numbers > 255), use `echo` inside the function and capture with `$(function_name)`. Best practice: use `return 0` for success, `return 1` for failure (make functions testable with `if function; then`), and `echo` for actual output values.

---

## SECTION 7 — STRING MANIPULATION

### Q-LX-07 — Bash | Conceptual | Intermediate

> Explain **Bash string manipulation** — how do you extract, replace, trim,
> and split strings using built-in parameter expansion (no external tools)?

#### Key Points to Cover:

**String length:**
```bash
str="hello world"
echo ${#str}           # 11
```

**Substring extraction:**
```bash
str="Hello World DevOps"
echo ${str:6}          # "World DevOps" (from index 6)
echo ${str:6:5}        # "World" (5 chars from index 6)
echo ${str: -6}        # "DevOps" (last 6 chars)
```

**Remove prefix (from start):**
```bash
filename="backup_20240115.tar.gz"

# Shortest match from start (#):
echo ${filename#backup_}       # "20240115.tar.gz"

# Longest match from start (##):
path="/usr/local/bin/nginx"
echo ${path##*/}               # "nginx" (removes longest path prefix)
echo ${path#*/}                # "usr/local/bin/nginx" (removes shortest)
```

**Remove suffix (from end):**
```bash
filename="backup_20240115.tar.gz"

# Shortest match from end (%):
echo ${filename%.gz}           # "backup_20240115.tar"

# Longest match from end (%%)
echo ${filename%%.*}           # "backup_20240115" (removes all extensions)

# Practical:
for file in *.tar.gz; do
    base="${file%.tar.gz}"     # strip .tar.gz from each filename
    mkdir -p "$base"
    tar xzf "$file" -C "$base"
done
```

**String replacement:**
```bash
str="hello world world"

# Replace first occurrence:
echo ${str/world/Linux}        # "hello Linux world"

# Replace ALL occurrences:
echo ${str//world/Linux}       # "hello Linux Linux"

# Replace only at start (/#):
str="world hello world"
echo ${str/#world/Linux}       # "Linux hello world"

# Replace only at end (/%):
echo ${str/%world/Linux}       # "world hello Linux"

# Remove substring (replace with nothing):
echo ${str//world/}            # "hello  "
```

**Case conversion:**
```bash
str="Hello World"
echo ${str,,}    # "hello world" (all lowercase)
echo ${str^^}    # "HELLO WORLD" (all uppercase)
echo ${str,}     # "hello World" (first char lowercase)
echo ${str^}     # "Hello World" (first char uppercase)
```

**String splitting:**
```bash
# Split by delimiter using IFS:
csv="one,two,three,four"
IFS=',' read -ra parts <<< "$csv"
for part in "${parts[@]}"; do
    echo "$part"
done
# one
# two
# three
# four
```

**Check if string contains substring:**
```bash
str="Hello World"

# Method 1 (case statement):
case "$str" in
    *"World"*) echo "Contains World" ;;
    *) echo "Does not contain" ;;
esac

# Method 2 (double bracket):
if [[ "$str" == *"World"* ]]; then
    echo "Contains World"
fi

# Check if empty:
if [[ -z "$str" ]]; then echo "empty"; fi

# Check if not empty:
if [[ -n "$str" ]]; then echo "not empty"; fi
```

> 💡 **Interview tip:** Bash string manipulation with `${var#}`, `${var%}`, `${var/}` is much faster than calling `sed`, `awk`, or `cut` for simple operations — no subprocess spawned. In a loop processing 10,000 files, using `${file%.gz}` (built-in) vs `$(echo $file | sed 's/.gz//')` (subprocess) makes a HUGE performance difference. The rule: use built-in parameter expansion for simple string ops, use `awk`/`sed` only when you need regex or multi-field processing.

---

## SECTION 8 — PARAMETER EXPANSION

### Q-LX-08 — Bash | Conceptual | Intermediate

> Explain **Bash parameter expansion** for default values, error handling,
> and variable assignment. What are `${var:-}`, `${var:=}`, `${var:?}`, `${var:+}`?

#### Key Points to Cover:

```bash
# ${var:-default}  →  use default if var is unset or empty
name=""
echo ${name:-"Anonymous"}      # "Anonymous" (var is empty)
name="Siddharth"
echo ${name:-"Anonymous"}      # "Siddharth" (var is set)
# NOTE: does NOT set the variable, just uses default for this expression

# ${var:=default}  →  use default AND assign to var if unset/empty
echo ${name:="Default Name"}   # assigns if empty
echo $name                     # variable is now set

# ${var:?error_message}  →  print error and exit if unset/empty
DATABASE_URL=${DATABASE_URL:?"ERROR: DATABASE_URL must be set"}
# If DATABASE_URL not set: "ERROR: DATABASE_URL must be set" → script exits

# ${var:+alternative}  →  use alternative if var IS set (opposite of :-)
debug=${DEBUG:+"--verbose"}    # "--verbose" only if DEBUG is set, otherwise empty
./app $debug

# Practical examples:
#!/usr/bin/env bash
set -euo pipefail

# Fail fast if required variables not set:
: "${AWS_REGION:?AWS_REGION must be set}"
: "${CLUSTER_NAME:?CLUSTER_NAME must be set}"
: "${IMAGE_TAG:?IMAGE_TAG must be set}"

# Defaults for optional variables:
LOG_LEVEL="${LOG_LEVEL:-info}"
TIMEOUT="${TIMEOUT:-30}"
ENVIRONMENT="${ENVIRONMENT:-development}"

echo "Deploying to ${CLUSTER_NAME} in ${AWS_REGION}"
echo "Log level: ${LOG_LEVEL}, Timeout: ${TIMEOUT}s"
```

**Indirect expansion:**
```bash
# Access variable whose name is stored in another variable:
var_name="DATABASE_URL"
echo ${!var_name}              # prints value of DATABASE_URL
```

> 💡 **Interview tip:** `${var:?message}` with the `: ` command (colon) is the **production standard for required variable validation** at the top of scripts. The `: ` command does nothing but evaluate its arguments — so `": ${VAR:?error}"` fails the script with a clear message if VAR is unset. This is cleaner than writing `if [ -z "$VAR" ]; then echo "error"; exit 1; fi` for every required variable. Put all required variable checks at the top of your script — fail fast with a clear error rather than failing mysteriously 30 steps later.

---

## SECTION 9 — TRAP & SIGNALS

### Q-LX-09 — Bash | Conceptual | Intermediate

> Explain **`trap`** in Bash — what is it, what signals can you trap,
> and how do you use it for cleanup and error handling?
>
> Write a production script that uses trap for cleanup on EXIT, ERR, and SIGINT.

#### Key Points to Cover:

**What trap does:**
```bash
# trap executes a command when a signal or event occurs
# Syntax: trap 'command' SIGNAL

# Common signals:
# EXIT    - script exits (any reason)
# ERR     - any command returns non-zero (with set -e)
# SIGINT  - Ctrl+C (interrupt)
# SIGTERM - kill signal
# SIGHUP  - terminal closed
# DEBUG   - before every command

# Simple cleanup on exit:
trap 'rm -f /tmp/lockfile.$$' EXIT
# This runs ALWAYS when script exits (success or failure)
```

**Production cleanup pattern:**
```bash
#!/usr/bin/env bash
set -euo pipefail

LOCKFILE="/tmp/deploy.lock"
TMPDIR=$(mktemp -d)
BACKUP_FILE=""

# Cleanup function:
cleanup() {
    local exit_code=$?
    echo "Cleaning up..."
    rm -rf "$TMPDIR"
    rm -f "$LOCKFILE"
    if [ -n "$BACKUP_FILE" ] && [ -f "$BACKUP_FILE" ]; then
        rm -f "$BACKUP_FILE"
    fi
    if [ $exit_code -ne 0 ]; then
        echo "Script FAILED with exit code: $exit_code" >&2
    fi
    exit $exit_code
}

# Error handler:
error_handler() {
    echo "ERROR: Script failed at line $LINENO: $BASH_COMMAND" >&2
}

# Summary on SIGINT (Ctrl+C):
sigint_handler() {
    echo ""
    echo "=== Interrupted by user ==="
    echo "Processed: $processed_count files"
    cleanup
}

# Register traps:
trap cleanup EXIT             # always run cleanup
trap error_handler ERR        # run on any error
trap sigint_handler SIGINT    # handle Ctrl+C gracefully
trap sigint_handler SIGTERM   # handle kill signal

# Script body:
echo $$ > "$LOCKFILE"    # write PID to lockfile
processed_count=0

for file in /data/*.sql; do
    BACKUP_FILE="${file}.bak"
    cp "$file" "$BACKUP_FILE"
    process_file "$file"
    rm -f "$BACKUP_FILE"
    processed_count=$((processed_count + 1))
done

echo "Done! Processed $processed_count files"
# cleanup() runs automatically on EXIT
```

**Trap for temporary files:**
```bash
# Most common production pattern:
TMPFILE=$(mktemp)
trap "rm -f $TMPFILE" EXIT   # ALWAYS delete temp file when done

# Download something, process, cleanup:
curl -s https://api.example.com/data > "$TMPFILE"
process "$TMPFILE"
# $TMPFILE deleted automatically on exit
```

> 💡 **Interview tip:** `trap cleanup EXIT` is the single most important trap to know — it runs on EVERY exit path (normal completion, error, `exit` command, killed by signal). Without it, temporary files and lockfiles accumulate on disk. The `$LINENO` and `$BASH_COMMAND` in the ERR trap are powerful debugging tools — they tell you exactly which line and which command failed. In production scripts, always use `trap 'error_handler' ERR` alongside `set -e` for automatic error reporting.

---

## SECTION 10 — GETOPTS (ARGUMENT PARSING)

### Q-LX-10 — Bash | Conceptual | Intermediate

> Explain **`getopts`** — how do you parse command-line flags and arguments
> in a Bash script? What is the difference between `getopts` and `getopt`?

#### Key Points to Cover:

```bash
#!/usr/bin/env bash
# Script: deploy.sh -e production -i v1.2.3 -v

usage() {
    echo "Usage: $0 -e <environment> -i <image-tag> [-v] [-h]"
    echo "  -e  Environment (dev|staging|prod)"
    echo "  -i  Image tag to deploy"
    echo "  -v  Verbose mode"
    echo "  -h  Show this help"
    exit 1
}

# Defaults:
ENVIRONMENT=""
IMAGE_TAG=""
VERBOSE=false

# getopts loop:
while getopts ":e:i:vh" opt; do
    # : before options = silent error mode
    # : after option letter = option requires an argument
    case $opt in
        e)
            ENVIRONMENT="$OPTARG"    # OPTARG = argument value
            ;;
        i)
            IMAGE_TAG="$OPTARG"
            ;;
        v)
            VERBOSE=true
            ;;
        h)
            usage
            ;;
        :)
            echo "ERROR: Option -$OPTARG requires an argument" >&2
            usage
            ;;
        \?)
            echo "ERROR: Unknown option -$OPTARG" >&2
            usage
            ;;
    esac
done

# Shift past processed options (remaining args in $@):
shift $((OPTIND - 1))

# Validate required args:
[[ -z "$ENVIRONMENT" ]] && { echo "ERROR: -e required"; usage; }
[[ -z "$IMAGE_TAG" ]]   && { echo "ERROR: -i required"; usage; }

$VERBOSE && echo "Verbose mode enabled"
echo "Deploying $IMAGE_TAG to $ENVIRONMENT"

# Usage:
# ./deploy.sh -e production -i v1.2.3 -v
# ./deploy.sh -e dev -i sha-abc123
```

**`getopts` vs `getopt`:**
```
getopts (built-in, recommended):
  → Built into bash/sh (no install needed)
  → Portable — works on all Unix systems
  → Only supports single-character flags (-e, -v, -h)
  → Does NOT support long options (--environment, --verbose)

getopt (external command):
  → Supports long options (--environment prod)
  → NOT portable (different behavior on macOS vs Linux)
  → Requires careful quoting to handle spaces in arguments
  → Use only when long options are required and portability doesn't matter

For most DevOps scripts: use getopts (simpler, portable)
```

> 💡 **Interview tip:** The leading colon in `getopts ":e:i:vh"` enables **silent error mode** — without it, getopts prints its own error messages to stderr which you can't control. With the leading colon, invalid options set `opt` to `?` and missing arguments set `opt` to `:`, giving you full control over error messages. Always use the leading colon in production scripts. Also mention `$OPTIND` — `shift $((OPTIND - 1))` after the getopts loop shifts away the processed flags so `$@` contains only non-option arguments.

---

## SECTION 11 — REGEX IN BASH (=~)

### Q-LX-11 — Bash | Conceptual | Intermediate

> Explain **regex pattern matching** in Bash using the `=~` operator.
> What is the difference between glob patterns and regex in Bash?
> How do you capture groups?

#### Key Points to Cover:

```bash
# =~ operator inside [[ ]] — regex matching
string="Error: connection timeout after 30 seconds"

# Basic match:
if [[ "$string" =~ timeout ]]; then
    echo "Found timeout"
fi

# Match at start:
if [[ "$string" =~ ^Error ]]; then
    echo "Starts with Error"
fi

# Match digits:
if [[ "$string" =~ [0-9]+ ]]; then
    echo "Contains numbers"
fi

# Capture groups (BASH_REMATCH array):
date_str="2024-01-15"
if [[ "$date_str" =~ ^([0-9]{4})-([0-9]{2})-([0-9]{2})$ ]]; then
    echo "Full match: ${BASH_REMATCH[0]}"   # 2024-01-15
    echo "Year:       ${BASH_REMATCH[1]}"   # 2024
    echo "Month:      ${BASH_REMATCH[2]}"   # 01
    echo "Day:        ${BASH_REMATCH[3]}"   # 15
fi

# Validate IP address:
ip="192.168.1.100"
if [[ "$ip" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]]; then
    echo "Valid IP format"
fi

# Extract version number from string:
version_str="nginx/1.25.3"
if [[ "$version_str" =~ ([0-9]+\.[0-9]+\.[0-9]+) ]]; then
    echo "Version: ${BASH_REMATCH[1]}"    # 1.25.3
fi

# Practical: validate environment argument:
validate_env() {
    local env="$1"
    if [[ ! "$env" =~ ^(dev|staging|prod)$ ]]; then
        echo "ERROR: Invalid environment '$env'. Must be dev, staging, or prod"
        return 1
    fi
}
```

**Glob vs Regex in Bash:**
```bash
# Glob (used with ==, file matching):
[[ "file.txt" == *.txt ]]      # glob: * = any chars
[[ "hello" == h?llo ]]         # glob: ? = single char
[[ "hello" == h[ae]llo ]]      # glob: [ae] = character class

# Regex (used with =~):
[[ "file.txt" =~ \.txt$ ]]     # regex: must escape dots, use anchors
[[ "hello" =~ ^h.llo$ ]]       # regex: . = any single char

# Glob: simpler, used for pattern matching
# Regex: more powerful, used for validation and extraction
```

> 💡 **Interview tip:** The `BASH_REMATCH` array is the most useful and least known bash feature. After `[[ string =~ pattern ]]`, `BASH_REMATCH[0]` contains the full match, and `BASH_REMATCH[1]`, `[2]`, `[3]` contain capture groups. This avoids calling `sed` or `awk` just to extract parts of a string. Critical: **store regex patterns in variables** to avoid quoting issues: `pattern='^[0-9]+$'; [[ "$str" =~ $pattern ]]` — putting the pattern directly in the `=~` expression can cause quoting problems with special characters.

---

## SECTION 12 — IFS (INTERNAL FIELD SEPARATOR)

### Q-LX-12 — Bash | Conceptual | Intermediate

> Explain **IFS** — what is it, what is its default value, and why is it
> critical for parsing strings and files correctly in Bash?

#### Key Points to Cover:

```bash
# IFS = Internal Field Separator
# Default: space, tab, newline
# Used by: read, for loops, word splitting

# Default behavior (splits on space):
line="one two three"
for word in $line; do
    echo "$word"    # one / two / three
done

# Change IFS to parse CSV:
csv="apple,banana,cherry"
IFS=',' read -ra fruits <<< "$csv"
for fruit in "${fruits[@]}"; do
    echo "$fruit"    # apple / banana / cherry
done

# Parse /etc/passwd (colon-separated):
IFS=':' read -r username _ uid gid _ home shell <<< "root:x:0:0:root:/root:/bin/bash"
echo "User: $username, UID: $uid, Shell: $shell"
# User: root, UID: 0, Shell: /bin/bash

# Restore IFS after changing:
OLD_IFS="$IFS"
IFS=','
# ... do CSV processing ...
IFS="$OLD_IFS"

# Or use local scope in function:
parse_csv() {
    local IFS=','    # local IFS — restored on function exit
    read -ra fields <<< "$1"
    echo "${fields[@]}"
}

# Why IFS= read -r is important for files:
while IFS= read -r line; do    # IFS= = no word splitting on read
    echo "$line"               # preserves leading/trailing spaces
done < file.txt
# Without IFS= : leading spaces stripped from each line
```

**Common IFS patterns:**
```bash
# Split PATH into array:
IFS=':' read -ra path_dirs <<< "$PATH"
for dir in "${path_dirs[@]}"; do
    echo "PATH dir: $dir"
done

# Join array with delimiter:
arr=("one" "two" "three")
IFS=','
joined="${arr[*]}"    # $* uses IFS as separator: "one,two,three"
echo "$joined"
IFS=$' \t\n'          # restore default

# Process CSV file:
while IFS=',' read -r name age city; do
    echo "Name: $name, Age: $age, City: $city"
done < users.csv
```

> 💡 **Interview tip:** The most common IFS bug: **forgetting `IFS=` before `read`** when reading files. Without it, `read` strips leading/trailing whitespace from each line. If your file has indented lines or lines with meaningful leading spaces, `while IFS= read -r line` is the ONLY correct way to read them. The empty `IFS=` means "don't split on anything" — combined with `-r` (no backslash interpretation), it reads each line exactly as-is.

---

## SECTION 13 — SUBSHELL vs CURRENT SHELL

### Q-LX-13 — Bash | Conceptual | Intermediate

> Explain the difference between **subshell** `( )`, **current shell** `{ }`,
> and **`source`** (dot command). When does each matter?

#### Key Points to Cover:

```bash
# SUBSHELL ( ) — runs in a child process
# Changes inside do NOT affect parent:
(
    cd /tmp
    MY_VAR="set in subshell"
    echo "Inside subshell: $PWD"    # /tmp
)
echo "Outside subshell: $PWD"       # original directory (unchanged!)
echo $MY_VAR                        # empty (subshell changes lost)

# Use subshell for: isolated operations, cd without affecting parent
(cd /tmp && tar czf backup.tar.gz /data)   # cd only lasts in subshell

# CURRENT SHELL { } — runs in current shell
# Changes AFFECT current shell:
{
    cd /tmp
    MY_VAR="set in current shell"
    echo "Inside block: $PWD"        # /tmp
}
echo "After block: $PWD"            # /tmp (changed!)
echo $MY_VAR                        # "set in current shell" (visible!)
# Note: { requires space after { and ; or newline before }

# source (.) — run script in CURRENT shell
# All changes affect current shell:
# script.sh:  export DB_URL="postgresql://localhost/mydb"
source script.sh        # or: . script.sh
echo $DB_URL            # postgresql://localhost/mydb (set in current shell!)

# vs running script directly:
./script.sh             # runs in subshell
echo $DB_URL            # empty (subshell changes lost)

# Practical importance:
# .bashrc, .bash_profile are SOURCED (not executed) by bash
# That's why changes to PATH in .bashrc affect your current shell
# If bash EXECUTED .bashrc, PATH changes would be in a subshell and lost

# Common use of source:
source ~/.bashrc                # reload shell config
source /etc/profile.d/java.sh   # load Java environment
. venv/bin/activate             # activate Python virtualenv (must be sourced!)
```

**Pipeline subshell trap:**
```bash
# Each element of a pipeline runs in a subshell!
count=0
cat file.txt | while read line; do
    count=$((count + 1))    # modifies count in SUBSHELL
done
echo $count    # 0 (unchanged! the while loop was in a subshell)

# Fix with process substitution:
count=0
while read line; do
    count=$((count + 1))
done < <(cat file.txt)    # process substitution — while runs in CURRENT shell
echo $count    # correct count
```

> 💡 **Interview tip:** The **pipeline subshell trap** catches many experienced developers. `cat file | while read line; do count=$((count+1)); done; echo $count` prints 0, not the file line count. The `while` loop runs in a subshell (piped from cat) so `count` changes are lost. This is one of the most common subtle bash bugs. The fix is `while read line; do ...; done < <(command)` — process substitution puts the command in a subshell but the while loop runs in the current shell.

---

## SECTION 14 — PARALLEL EXECUTION

### Q-LX-14 — Bash | Conceptual | Intermediate

> Explain **parallel execution** in Bash — how do you run tasks in the
> background, wait for them, and handle failures from background jobs?

#### Key Points to Cover:

```bash
# & = run command in background (creates subshell)
long_task() {
    sleep 5
    echo "Task $1 done"
}

# Sequential (slow):
long_task 1    # wait 5s
long_task 2    # wait 5s
long_task 3    # wait 5s
# Total: 15 seconds

# Parallel (fast):
long_task 1 &    # background PID saved in $!
long_task 2 &
long_task 3 &
wait             # wait for ALL background jobs to finish
# Total: ~5 seconds

# Wait for specific PID:
long_task 1 &
pid1=$!
long_task 2 &
pid2=$!
wait $pid1    # wait only for task 1
echo "Task 1 done, task 2 still running"
wait $pid2

# Check exit status of background jobs:
long_task 1 &
pid=$!
wait $pid
echo "Exit code: $?"

# Parallel with failure detection:
pids=()
exit_codes=()

for i in {1..5}; do
    process_item "$i" &
    pids+=($!)
done

# Wait and collect exit codes:
for pid in "${pids[@]}"; do
    wait "$pid"
    exit_codes+=($?)
done

# Check for failures:
failed=0
for code in "${exit_codes[@]}"; do
    if [ "$code" -ne 0 ]; then
        failed=$((failed + 1))
    fi
done

if [ $failed -gt 0 ]; then
    echo "ERROR: $failed tasks failed"
    exit 1
fi

# Limit concurrency (max N jobs at once):
MAX_JOBS=4
running=0

for server in "${servers[@]}"; do
    # Wait if at max concurrency:
    while [ $running -ge $MAX_JOBS ]; do
        wait -n 2>/dev/null    # wait for ANY job (-n = any, bash 4.3+)
        running=$((running - 1))
    done

    deploy_to_server "$server" &
    running=$((running + 1))
done

wait    # wait for remaining jobs
```

> 💡 **Interview tip:** The most important parallel bash pattern for DevOps: **deploy to multiple servers simultaneously**. Without parallelism, deploying to 20 servers × 30 seconds each = 10 minutes. With `&` + `wait`, it's ~30 seconds (all run at once). The critical safety rule: **always use `wait` after background jobs** — without it, the script exits and kills all background jobs immediately. Also mention `xargs -P` as an alternative: `printf '%s\n' "${servers[@]}" | xargs -P 4 -I{} deploy_to_server {}` runs deploy in parallel with max 4 at once.

---

## SECTION 15 — PROCESS & COMMAND SUBSTITUTION

### Q-LX-15 — Bash | Conceptual | Intermediate

> Explain **command substitution** `$()` and **process substitution** `<()` `>()`.
> What is the difference and when would you use each?

#### Key Points to Cover:

**Command substitution `$()`:**
```bash
# Captures stdout of a command as a string

# Basic:
current_date=$(date +%Y-%m-%d)
echo "Today is: $current_date"

# In string:
echo "Running as: $(whoami) on $(hostname)"

# Assign to variable:
files_count=$(ls /var/log/*.log | wc -l)
echo "Found $files_count log files"

# Nested:
newest_file=$(ls -t $(find /data -name "*.sql") | head -1)

# Old style (backticks) — avoid:
result=`command`    # harder to read, can't nest easily
result=$(command)   # preferred — nestable and readable

# Multi-line output becomes single string:
content=$(cat /etc/hosts)
echo "$content"   # prints with newlines preserved
```

**Process substitution `<()`:**
```bash
# Creates a named pipe (FIFO) and passes command output as a "file"
# Use when a command expects a FILE argument, not stdin

# Compare two command outputs:
diff <(sort file1.txt) <(sort file2.txt)
# Without <(): must create temp files:
# sort file1.txt > /tmp/f1 && sort file2.txt > /tmp/f2 && diff /tmp/f1 /tmp/f2

# diff between running configs:
diff <(ssh server1 "cat /etc/nginx/nginx.conf") \
     <(ssh server2 "cat /etc/nginx/nginx.conf")

# Loop over command output (while in current shell — not subshell):
while read line; do
    echo "Processing: $line"
done < <(find /data -name "*.log" -mtime +7)
# vs: find ... | while read line — this runs while in SUBSHELL!

# Join two sorted files (comm, join require sorted file args):
comm <(sort list1.txt) <(sort list2.txt)
```

**Output process substitution `>()`:**
```bash
# Send output of current pipeline to a command that expects a file

# Tee to two different processing pipelines:
cat big_log.txt | tee >(grep ERROR | wc -l > error_count.txt) \
                      >(grep INFO  | wc -l > info_count.txt) \
                      > /dev/null

# Log and process simultaneously:
./app 2>&1 | tee >(logger -t myapp) > /var/log/app.log
# Sends output to: syslog (via logger) AND log file simultaneously
```

> 💡 **Interview tip:** Process substitution `<()` solves the **pipeline subshell problem**. When you need `while read` to modify variables in the current shell, use `while read line; do ...; done < <(command)` instead of `command | while read line; do ...; done`. The `< <(command)` runs the command in a subshell (normal) but keeps the `while` loop in the current shell. This is used constantly in real DevOps scripts for file/output processing.

---

## SECTION 16 — CURL + JQ IN SCRIPTS

### Q-LX-16 — Bash | Practical | Intermediate

> Explain how to use **`curl`** for API calls and **`jq`** for JSON parsing
> in Bash scripts. Write a script that queries an API and processes the response.

#### Key Points to Cover:

**`curl` for API calls:**
```bash
# Basic GET:
curl https://api.example.com/users

# With headers (auth token):
curl -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     https://api.example.com/users

# POST with JSON body:
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"name": "siddharth", "role": "admin"}' \
     https://api.example.com/users

# Useful flags:
# -s        = silent (no progress bar)
# -S        = show errors even with -s
# -f        = fail on HTTP errors (exit code 22 for 4xx/5xx)
# -L        = follow redirects
# -o file   = output to file
# -w "%{http_code}" = print HTTP status code
# --max-time 10     = timeout after 10 seconds
# --retry 3         = retry up to 3 times

# Check HTTP status code:
http_code=$(curl -s -o /dev/null -w "%{http_code}" https://api.example.com/health)
if [ "$http_code" -ne 200 ]; then
    echo "API unhealthy: HTTP $http_code"
    exit 1
fi

# Download large file with progress and resume:
curl -L -C - -o /data/file.tar.gz https://example.com/large-file.tar.gz
# -C - = resume from where it stopped
```

**`jq` for JSON parsing:**
```bash
# Install: apt install jq / yum install jq

json='{"name":"siddharth","age":28,"skills":["kubernetes","terraform","aws"]}'

# Get field:
echo "$json" | jq '.name'              # "siddharth"
echo "$json" | jq -r '.name'           # siddharth (raw, no quotes)

# Nested field:
echo "$json" | jq '.address.city'

# Array element:
echo "$json" | jq '.skills[0]'         # "kubernetes"
echo "$json" | jq '.skills[]'          # all elements
echo "$json" | jq -r '.skills[]'       # all elements without quotes

# Array length:
echo "$json" | jq '.skills | length'   # 3

# Filter array:
echo "$json" | jq '[.skills[] | select(. == "terraform")]'

# Multiple fields:
echo "$json" | jq '{name: .name, first_skill: .skills[0]}'

# From file:
jq '.users[] | .name' users.json

# Loop over JSON array in bash:
echo "$json" | jq -r '.skills[]' | while read skill; do
    echo "Skill: $skill"
done
```

**Real DevOps script — Kubernetes deployment check:**
```bash
#!/usr/bin/env bash
set -euo pipefail

K8S_API="https://kubernetes.default.svc"
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
NAMESPACE="${NAMESPACE:-production}"

# Query deployment status:
response=$(curl -s -f \
    -H "Authorization: Bearer $TOKEN" \
    --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    "${K8S_API}/apis/apps/v1/namespaces/${NAMESPACE}/deployments")

# Parse with jq:
total=$(echo "$response" | jq '.items | length')
echo "Total deployments: $total"

# Check each deployment's ready replicas:
echo "$response" | jq -r '
    .items[] | 
    "\(.metadata.name) \(.status.readyReplicas // 0)/\(.status.replicas // 0)"
' | while read name ready total; do
    if [ "$ready" != "$total" ]; then
        echo "❌ DEGRADED: $name ($ready/$total ready)"
    else
        echo "✅ HEALTHY:  $name ($ready/$total ready)"
    fi
done
```

> 💡 **Interview tip:** `jq -r` (raw output) is critical for using jq output in bash variables — without it, strings are quoted: `name=$(echo $json | jq '.name')` gives `"siddharth"` (with quotes). With `-r`: `name=$(echo $json | jq -r '.name')` gives `siddharth` (without quotes). Always use `-r` when assigning jq output to bash variables. Also: `jq '// 0'` (alternative operator) handles null values — `jq '.missingField // 0'` returns 0 instead of null when the field doesn't exist.

---

## SECTION 17 — LOCKFILE / FLOCK

### Q-LX-17 — Bash | Practical | Intermediate

> How do you prevent a Bash script from running **multiple instances simultaneously**?
> Explain `flock` and the manual lockfile pattern.

#### Key Points to Cover:

```bash
# Problem: cron runs every minute, but script takes 3 minutes
# Without lock: 3 instances running simultaneously → data corruption

# Method 1: flock (kernel-level file locking — RECOMMENDED)
#!/usr/bin/env bash
LOCKFILE="/var/lock/my-script.lock"

# Exclusive lock (-x), non-blocking (-n), fail with exit 1 if locked:
exec 9>"$LOCKFILE"
if ! flock -n 9; then
    echo "ERROR: Another instance is running. Exiting."
    exit 1
fi
# Lock held for entire script (fd 9 is open)

echo "Running script..."
sleep 60
echo "Done"
# Lock released when script exits (fd 9 closed)

# Method 2: flock as command wrapper:
flock -n /var/lock/my-script.lock -c "bash /opt/scripts/backup.sh"
# Simpler — flock wraps the command directly

# Method 3: Manual PID lockfile:
LOCKFILE="/tmp/my-script.lock"

cleanup() {
    rm -f "$LOCKFILE"
}
trap cleanup EXIT

# Check if lock exists and process is still running:
if [ -f "$LOCKFILE" ]; then
    pid=$(cat "$LOCKFILE")
    if kill -0 "$pid" 2>/dev/null; then
        echo "ERROR: Already running (PID $pid)"
        exit 1
    else
        echo "Stale lockfile found (PID $pid dead), removing"
        rm -f "$LOCKFILE"
    fi
fi

# Create lockfile with our PID:
echo $$ > "$LOCKFILE"
echo "Running as PID $$"
sleep 60

# Method 4: mkdir as atomic lock (portable):
LOCKDIR="/tmp/my-script.lock"

if mkdir "$LOCKDIR" 2>/dev/null; then
    trap "rmdir '$LOCKDIR'" EXIT
    echo "Lock acquired, running..."
else
    echo "ERROR: Another instance is running"
    exit 1
fi
# mkdir is ATOMIC on most filesystems — safe on NFS too
```

> 💡 **Interview tip:** `flock` is superior to manual PID lockfiles because it handles **crash recovery automatically** — if the script dies unexpectedly, the OS releases the file lock immediately (no stale lockfile left behind). With manual PID lockfiles, a crashed script leaves the lockfile behind and you need logic to detect stale locks. Use `flock` for cron jobs on local filesystems, use `mkdir` for scripts that need to work on NFS (where flock may not work reliably across network filesystems).

---

## SECTION 18 — find COMMAND

### Q-LX-18 — Linux | Conceptual | Intermediate

> Explain the **`find`** command in depth — what are the most important flags
> and how do you use `-exec` for bulk operations?

#### Key Points to Cover:

```bash
# Basic syntax: find <path> <conditions> <action>

# Find by name:
find /var/log -name "*.log"           # files ending in .log
find /var/log -name "app*.log"        # wildcard in name
find /var/log -iname "*.LOG"          # case-insensitive

# Find by type:
find /home -type f    # files only
find /home -type d    # directories only
find /home -type l    # symbolic links only
find /tmp  -type s    # sockets only

# Find by size:
find /var -size +100M             # larger than 100MB
find /tmp -size +1G               # larger than 1GB
find /home -size -1k              # smaller than 1KB
find /data -size +500M -size -1G  # between 500MB and 1GB

# Find by time (modification time):
find /var/log -mtime +7           # modified MORE than 7 days ago
find /tmp -mtime -1               # modified LESS than 1 day ago
find /data -mtime 0               # modified TODAY
find /home -newer /tmp/reference  # newer than reference file

# Find by permissions:
find /etc -perm 644               # exactly 644
find /tmp -perm -o+w              # world-writable (security risk!)
find / -perm -4000                # SUID files (security audit)
find / -perm -2000                # SGID files

# Find by owner:
find /home -user siddharth        # owned by user
find /var -group www-data         # owned by group
find /tmp -nouser                 # no valid owner (orphaned files)

# Find by depth:
find /etc -maxdepth 1 -name "*.conf"   # only in /etc (not subdirs)
find /etc -mindepth 2 -name "*.conf"   # only in subdirs

# Combining conditions:
find /var/log -name "*.log" -size +100M -mtime +30    # AND (default)
find /tmp -name "*.tmp" -o -name "*.temp"              # OR
find /home ! -name "*.txt"                              # NOT

# -exec: run command on each found file:
# {} = placeholder for found file
# \; = end of command

# Delete found files:
find /tmp -name "*.tmp" -mtime +7 -exec rm {} \;

# Safer delete (confirm first with ls):
find /var/log -name "*.gz" -mtime +30 -exec ls -lh {} \;
find /var/log -name "*.gz" -mtime +30 -delete    # -delete is faster than -exec rm

# Change permissions on found files:
find /var/www -type f -exec chmod 644 {} \;
find /var/www -type d -exec chmod 755 {} \;

# -exec with + (batches files — faster than \;):
find /tmp -name "*.log" -exec gzip {} +    # passes multiple files at once to gzip

# Count found files:
find /var/log -name "*.log" | wc -l

# Find and move:
find /data -name "*.csv" -exec mv {} /archive/ \;

# Find large files for cleanup:
find / -type f -size +500M 2>/dev/null | sort -k5 -rn

# Find files changed in last hour and backup:
find /etc -mmin -60 -type f -exec cp {} /tmp/recent-changes/ \;

# Security audit — SUID/SGID files:
find / -perm -4000 -type f 2>/dev/null    # SUID files
find / -perm -2000 -type f 2>/dev/null    # SGID files
find /tmp -perm -o+w -type f 2>/dev/null  # world-writable files
```

> 💡 **Interview tip:** `-exec cmd {} +` (plus sign) is faster than `-exec cmd {} \;` (semicolon). The semicolon runs the command once per file. The plus sign batches files together and runs the command once with all files as arguments (like xargs). For `rm`, `gzip`, `chmod` — use `+`. For commands that only work on ONE file at a time — use `\;`. Also: always use `2>/dev/null` when searching from `/` to suppress "Permission denied" errors for system directories.

---

## SECTION 19 — TEXT PROCESSING

### Q-LX-19 — Linux | Conceptual | Intermediate

> Explain **`sort`**, **`tr`**, **`tee`**, **`diff`** commands and
> **regex (BRE vs ERE)** in Linux. Give practical DevOps examples for each.

#### Key Points to Cover:

**`sort` — sort lines:**
```bash
# Basic sort (alphabetical):
sort file.txt

# Numeric sort:
sort -n numbers.txt           # 1, 2, 10, 100 (not 1, 10, 100, 2)

# Reverse:
sort -r file.txt
sort -rn file.txt             # reverse numeric

# Sort by field (-k column):
# -t = delimiter, -k = field number
sort -t',' -k2 data.csv       # sort by 2nd CSV field
sort -t':' -k3 -n /etc/passwd # sort /etc/passwd by UID (field 3) numerically

# Sort and remove duplicates:
sort -u file.txt              # unique lines only
sort file.txt | uniq          # same result

# Sort by file size (from du output):
du -sh /var/log/* | sort -rh  # -h = human-readable sizes
# 2.1G /var/log/journal
# 450M /var/log/messages

# Sort multiple keys:
sort -t',' -k2 -k3n data.csv  # primary sort by field 2, secondary by field 3 numeric

# Common DevOps use:
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
# Top 10 IPs by request count
```

**`tr` — translate/delete characters:**
```bash
# Translate characters:
echo "Hello World" | tr 'a-z' 'A-Z'   # HELLO WORLD
echo "Hello World" | tr 'A-Z' 'a-z'   # hello world

# Delete characters (-d):
echo "hello123world" | tr -d '0-9'    # helloworld
echo "Hello, World!" | tr -d '[:punct:]'  # Hello World

# Squeeze repeated characters (-s):
echo "hello   world" | tr -s ' '      # "hello world" (multiple spaces → one)
echo "aaabbbccc" | tr -s 'a-z'        # "abc"

# Replace newlines (common use):
echo -e "one\ntwo\nthree" | tr '\n' ','  # one,two,three

# Common DevOps use:
# Remove Windows carriage returns from file:
tr -d '\r' < windows_file.txt > unix_file.txt

# Convert colons to newlines (parse PATH):
echo $PATH | tr ':' '\n'
```

**`tee` — split output to file AND stdout:**
```bash
# Without tee: choose file OR screen
command > file.txt      # only to file (can't see it)
command                 # only to screen (can't save it)

# With tee: BOTH simultaneously
command | tee output.txt              # file + screen
command | tee -a output.txt           # -a = append (don't overwrite)
command | tee file1.txt file2.txt     # write to multiple files

# Common DevOps use:
terraform plan 2>&1 | tee plan.txt    # save plan output + see it live
./deploy.sh 2>&1 | tee deploy_$(date +%Y%m%d).log
ansible-playbook site.yml | tee /var/log/ansible.log

# Tee in the middle of pipeline:
cat access.log | tee raw.log | grep ERROR | tee errors.log | wc -l
# raw.log = full log
# errors.log = only errors
# wc -l = count of error lines
```

**`diff` — compare files:**
```bash
# Basic diff:
diff file1.txt file2.txt
# < = line only in file1
# > = line only in file2
# --- = file1 marker
# +++ = file2 marker

# Unified format (like git diff):
diff -u file1.txt file2.txt
# -3/+3 context lines shown
diff -u --context=5 file1.txt file2.txt    # 5 lines context

# Recursive directory diff:
diff -r dir1/ dir2/

# Side-by-side:
diff -y --width=200 file1.txt file2.txt

# Common DevOps use:
# Compare configs between servers:
diff <(ssh server1 "cat /etc/nginx/nginx.conf") \
     <(ssh server2 "cat /etc/nginx/nginx.conf")

# Compare current K8s resource with local file:
kubectl get deployment myapp -o yaml > current.yaml
diff current.yaml desired.yaml
```

**BRE vs ERE (Basic vs Extended Regex):**
```bash
# BRE (Basic Regular Expressions) — grep default:
# Special chars: . * ^ $ [] need NO escaping
# + ? { } | ( ) need ESCAPING: \+ \? \{ \| \(

grep "error\+" file.txt       # BRE: one or more 'r's
grep "colou\?r" file.txt      # BRE: optional 'u'
grep "\(foo\|bar\)" file.txt  # BRE: foo or bar

# ERE (Extended Regular Expressions) — grep -E, egrep, awk:
# Special chars: + ? { } | ( ) need NO escaping
# MORE readable and intuitive

grep -E "error+" file.txt     # ERE: one or more 'r's
grep -E "colou?r" file.txt    # ERE: optional 'u'
grep -E "(foo|bar)" file.txt  # ERE: foo or bar

# Practical comparison:
grep "^[0-9]\{3\}-[0-9]\{3\}-[0-9]\{4\}$" phones.txt    # BRE: 3-digit patterns
grep -E "^[0-9]{3}-[0-9]{3}-[0-9]{4}$" phones.txt        # ERE: cleaner

# sed uses BRE by default, sed -E for ERE:
sed 's/[0-9]\+/NUM/g' file.txt      # BRE sed
sed -E 's/[0-9]+/NUM/g' file.txt    # ERE sed (cleaner)

# Character classes (same in BRE and ERE):
[0-9]    = any digit
[a-z]    = lowercase letter
[A-Z]    = uppercase letter
[a-zA-Z0-9]  = alphanumeric
[:digit:]    = POSIX digit class
[:alpha:]    = POSIX alpha class
[:space:]    = whitespace
\d           = digit (only in PCRE/perl, NOT in basic grep/sed)
```

> 💡 **Interview tip:** The most practically useful `sort` flag in DevOps: `sort -rn` for sorting numbers in reverse (largest first). `du -sh /* | sort -rh` is how you find what's consuming disk space. The BRE vs ERE distinction: always use `grep -E` (ERE) in scripts — it's more readable and consistent with other tools. `grep` without `-E` using `+` means "literal plus sign", not "one or more" — this trips up many engineers who write patterns expecting ERE behavior.

---

## SECTION 20 — NETWORKING GAPS

### Q-LX-20 — Linux/Networking | Conceptual | Intermediate

> Explain **TCP vs UDP** — fundamental differences and when to use each.
> Also explain **`firewalld`/`ufw`** for practical firewall management.

#### Key Points to Cover:

**TCP vs UDP:**
```
TCP (Transmission Control Protocol):
  → Connection-oriented (3-way handshake before data)
  → Reliable: guarantees delivery, ordering, error detection
  → Flow control and congestion control
  → Higher overhead (headers, ACKs, state)
  → Slower due to reliability mechanisms

  Use TCP when:
  ✅ Data must arrive intact and in order
  ✅ Web (HTTP/HTTPS), email (SMTP), SSH, FTP
  ✅ Database connections (MySQL, PostgreSQL)
  ✅ File transfers (any loss = corrupt file)

UDP (User Datagram Protocol):
  → Connectionless (no handshake, no state)
  → Unreliable: no delivery guarantee, no ordering
  → No flow control
  → Low overhead (8-byte header vs TCP's 20-byte)
  → Fast — "fire and forget"

  Use UDP when:
  ✅ Speed > reliability
  ✅ Loss is acceptable (video streaming, gaming)
  ✅ Application handles its own reliability (QUIC/HTTP3)
  ✅ DNS queries (fast, single packet usually)
  ✅ Syslog, SNMP, NTP, DHCP
  ✅ VoIP (delay is worse than dropped packet)

Key comparison:
  Feature         TCP              UDP
  Connection      Required         None
  Reliability     Guaranteed       Best-effort
  Ordering        In-order         Out-of-order possible
  Speed           Slower           Faster
  Overhead        High             Low
  Use case        HTTP, SSH, DB    DNS, VoIP, video, gaming
```

**`ufw` (Ubuntu/Debian firewall):**
```bash
# Enable/disable:
ufw enable
ufw disable
ufw status verbose

# Allow/deny ports:
ufw allow 22              # allow SSH
ufw allow 80/tcp          # allow HTTP
ufw allow 443             # allow HTTPS
ufw deny 23               # deny Telnet

# Allow from specific IP:
ufw allow from 10.0.0.0/8 to any port 22
ufw allow from 192.168.1.100

# Allow service by name:
ufw allow ssh
ufw allow http
ufw allow https

# Delete rule:
ufw delete allow 80
ufw status numbered
ufw delete 3              # delete rule #3

# Default policies:
ufw default deny incoming
ufw default allow outgoing
```

**`firewalld` (RHEL/CentOS/Amazon Linux):**
```bash
# Status:
firewall-cmd --state
firewall-cmd --list-all

# Add service (permanent + runtime):
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload    # apply permanent changes

# Add specific port:
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --add-port=9090/tcp
firewall-cmd --reload

# Remove:
firewall-cmd --permanent --remove-port=8080/tcp

# Zones (firewalld concept — trust levels):
firewall-cmd --get-active-zones
firewall-cmd --zone=public --list-all
firewall-cmd --permanent --zone=public --add-service=http
```

> 💡 **Interview tip:** DNS is a great example of why UDP makes sense — a DNS query is tiny (fits in one packet), and if it fails, the client just retries. Using TCP for DNS would add 3-way handshake overhead for every single query — unacceptable for a service queried thousands of times per second. However, DNS DOES use TCP for: zone transfers (large data) and responses > 512 bytes (DNSSEC). The rule: **UDP for latency-sensitive, loss-tolerant; TCP for reliability-critical**.

---

## SECTION 21 — ACLs

### Q-LX-21 — Linux | Conceptual | Intermediate

> Explain **Linux ACLs (Access Control Lists)** — what problem do they solve
> that standard `chmod` cannot? How do you use `setfacl` and `getfacl`?

#### Key Points to Cover:

```
Problem with standard permissions:
  Standard chmod only allows ONE owner, ONE group, ONE "other"
  chmod 750 /project = owner(rwx), group(r-x), others(---)
  But what if: alice (owner) + bob's team need rwx, carol needs r-x only?
  Standard permissions cannot express this — only one group!

ACL solution:
  Add per-user or per-group permissions ON TOP of standard permissions
  Owner, group, AND multiple individual users/groups
```

```bash
# View ACLs:
getfacl /data/project/
# File: /data/project/
# Owner: alice
# Group: devteam
# user::rwx          (owner)
# user:bob:rwx       (ACL entry for bob)
# user:carol:r-x     (ACL entry for carol)
# group::r-x         (owning group)
# group:ops:rw-      (ACL entry for ops group)
# mask::rwx
# other::---

# Set ACL for specific user:
setfacl -m u:bob:rwx /data/project/          # give bob rwx
setfacl -m u:carol:rx /data/project/         # give carol read+execute
setfacl -m u:dave:--- /data/project/         # explicitly deny dave

# Set ACL for specific group:
setfacl -m g:developers:rwx /data/project/

# Recursive (apply to all files/dirs):
setfacl -R -m u:bob:rwx /data/project/

# Set default ACL (new files inherit these):
setfacl -d -m u:bob:rwx /data/project/     # -d = default
# Any new file/dir created inside inherits bob's ACL

# Remove ACL entry:
setfacl -x u:bob /data/project/            # remove bob's ACL
setfacl -x g:developers /data/project/     # remove group ACL
setfacl -b /data/project/                  # remove ALL ACL entries

# Copy ACL from one file to another:
getfacl file1.txt | setfacl --set-file=- file2.txt

# Check if filesystem supports ACLs:
tune2fs -l /dev/sda1 | grep "Default mount options"
# Must show: acl

# Mount with ACL support (if not default):
mount -o remount,acl /data
# /etc/fstab: /dev/sda2  /data  ext4  defaults,acl  0 2

# Practical DevOps use:
# Web app files owned by root, nginx needs read, app user needs write:
setfacl -m u:nginx:rx /var/www/html/
setfacl -m u:appuser:rwx /var/www/html/uploads/
setfacl -m g:webteam:rx /var/www/html/
```

> 💡 **Interview tip:** The `mask` in ACL output is important — it defines the **maximum permissions** any ACL entry (user or group) can have. `mask::r-x` means no ACL user/group can have write permission, even if setfacl grants it. The mask is automatically updated when you add ACL entries, but manually setting it can restrict all ACL entries at once. ACLs are widely used in **shared development servers** where multiple teams need different access levels to project directories without creating dozens of custom groups.

---

## SECTION 22 — SYSTEM ADMIN GAPS

### Q-LX-22 — Linux | Conceptual | Beginner-Intermediate

> Explain `.bashrc` vs `.bash_profile` vs `.profile` — when is each loaded?
> Also explain **environment variables**, **swap management**, and **sar**.

#### Key Points to Cover:

**Shell config files — when each loads:**
```
Login shell (SSH login, su -):
  1. /etc/profile        (system-wide)
  2. /etc/profile.d/*.sh (system-wide, modular)
  3. ~/.bash_profile     (user, if exists)  ← OR
     ~/.bash_login       (user, if .bash_profile missing)  ← OR
     ~/.profile          (user, if above missing)

Interactive non-login shell (new terminal window, bash in bash):
  1. /etc/bash.bashrc    (system-wide)
  2. ~/.bashrc           (user)

Non-interactive shell (script execution):
  → None of the above loaded!
  → Only inherits parent's exported variables

Summary:
  ~/.bash_profile  = login shell only (SSH sessions)
  ~/.bashrc        = interactive non-login shell
  Best practice:   put everything in ~/.bashrc
                   source ~/.bashrc from ~/.bash_profile:
                   # ~/.bash_profile:
                   [[ -f ~/.bashrc ]] && source ~/.bashrc
```

**Environment variables:**
```bash
# View all environment variables:
env
printenv
printenv PATH          # specific variable

# Set variable (current shell only):
MY_VAR="hello"

# Export (available to child processes):
export MY_VAR="hello"
export PATH="$PATH:/opt/custom/bin"

# Set and export in one:
export DATABASE_URL="postgresql://localhost:5432/mydb"

# Remove variable:
unset MY_VAR

# Permanent environment variables:
# User-level: add to ~/.bashrc or ~/.bash_profile
echo 'export JAVA_HOME="/usr/lib/jvm/java-11"' >> ~/.bashrc
source ~/.bashrc

# System-level: /etc/environment (simple key=value, no export needed)
# DATABASE_URL="postgresql://prod-db:5432/mydb"

# Or: /etc/profile.d/myapp.sh
echo 'export APP_HOME="/opt/myapp"' > /etc/profile.d/myapp.sh

# Pass env var to single command:
DATABASE_URL="postgresql://localhost/test" ./run_tests.sh

# Check child process inheritance:
export MY_VAR="inherited"
bash -c 'echo $MY_VAR'    # "inherited" (exported)
MY_UNEXP="not inherited"
bash -c 'echo $MY_UNEXP'  # "" (not exported)
```

**Swap space management:**
```bash
# Check current swap:
free -h
swapon --show
cat /proc/swaps

# Create swap file:
dd if=/dev/zero of=/swapfile bs=1G count=4   # 4GB swap file
# OR faster:
fallocate -l 4G /swapfile

chmod 600 /swapfile          # secure permissions
mkswap /swapfile             # format as swap
swapon /swapfile             # enable

# Make permanent (add to /etc/fstab):
echo "/swapfile none swap sw 0 0" >> /etc/fstab

# Tune swappiness (how aggressively to use swap):
cat /proc/sys/vm/swappiness  # default: 60
sysctl vm.swappiness=10      # prefer RAM over swap (good for servers)
echo "vm.swappiness=10" >> /etc/sysctl.conf  # permanent

# Disable swap (Kubernetes requirement!):
swapoff -a
# Comment out swap line in /etc/fstab to persist
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

**`sar` — system activity reporter:**
```bash
# sar = historical system performance data
# Requires: sysstat package (apt install sysstat / yum install sysstat)

# Enable data collection:
# /etc/default/sysstat: ENABLED="true"
# /etc/cron.d/sysstat: runs every 10 minutes

# View CPU usage (historical):
sar -u                      # today's CPU usage
sar -u 1 5                  # live: every 1s, 5 times
sar -u -f /var/log/sysstat/sa15   # specific day (15th)

# Memory:
sar -r                      # memory utilization history

# Disk I/O:
sar -d                      # disk device statistics
sar -b                      # I/O transfer rate

# Network:
sar -n DEV                  # network interface statistics
sar -n TCP                  # TCP statistics

# Load average:
sar -q                      # run queue and load

# Why sar is valuable:
# "Server was slow at 3am but now it's fine"
# sar -u -f /var/log/sysstat/sa15 shows exactly what happened at 3am
# vs top/htop: only shows CURRENT state
```

> 💡 **Interview tip:** The `.bashrc` vs `.bash_profile` confusion causes real production issues. When you add `export PATH=...` to `~/.bashrc` but connect via SSH (login shell), it doesn't load. When you add it to `~/.bash_profile` but open a terminal emulator (non-login shell), it doesn't load. The **correct solution**: put all exports in `~/.bashrc`, then add `source ~/.bashrc` to `~/.bash_profile`. This way both login and non-login shells get all your variables. Also critical for Kubernetes: **swap must be disabled** (`swapoff -a`) — kubelet refuses to start on nodes with swap enabled.

---

## SECTION 23 — JOB CONTROL

### Q-LX-23 — Linux | Conceptual | Intermediate

> Explain **job control** in Linux — what are `jobs`, `fg`, `bg`, `nohup`,
> `disown`, and `Ctrl+Z`? Also explain `pstree` and `sar`.

#### Key Points to Cover:

**Job control basics:**
```bash
# Start a long-running command:
sleep 300

# Ctrl+Z = SUSPEND the job (pauses it, does NOT kill it)
# Shows: [1]+ Stopped    sleep 300

# See all jobs in current shell:
jobs
# [1]+ Stopped    sleep 300
# [2]- Running    python server.py &
# [N] = job number, + = current job, - = previous job

# Bring suspended job to FOREGROUND:
fg              # brings most recent job
fg %1           # brings job #1
fg %sleep       # brings job named "sleep"

# Send suspended job to BACKGROUND (resume running):
bg              # most recent suspended job
bg %1           # specific job

# Start command directly in background:
python server.py &
# Returns: [1] 12345  (job number and PID)

# Wait for background job to finish:
wait            # wait for all background jobs
wait %1         # wait for job #1
wait 12345      # wait for specific PID
```

**`nohup` and `disown` — survive terminal close:**
```bash
# nohup = run command immune to SIGHUP (terminal close)
nohup python server.py &
# Output goes to nohup.out (if no redirection)
nohup python server.py > server.log 2>&1 &

# disown = remove job from shell's job table (after already started)
python server.py &
disown %1       # now shell closing won't kill it
disown -a       # disown all background jobs

# Check if process survived:
ps aux | grep "python server.py"

# nohup vs disown:
# nohup: set BEFORE starting command (handles SIGHUP from start)
# disown: use AFTER starting command (forgot to use nohup)

# Background + nohup + redirect (complete pattern):
nohup ./long-backup.sh > /var/log/backup.log 2>&1 &
echo "Backup started as PID $!, logs: /var/log/backup.log"
```

**`pstree` — process tree:**
```bash
# Show processes as tree (parent-child relationships):
pstree
pstree -p          # show PIDs
pstree -u          # show usernames
pstree 1234        # show tree rooted at PID 1234
pstree siddharth   # show tree for user's processes

# Example output:
# systemd─┬─sshd───sshd───sshd───bash───pstree
#          ├─nginx─┬─nginx
#          │       └─nginx
#          ├─postgres─┬─postgres
#          │          └─postgres
#          └─...

# Useful for:
# - Understanding which process spawned which
# - Debugging "orphan" processes
# - Seeing all nginx worker processes
# - Finding zombie process parents
```

> 💡 **Interview tip:** `Ctrl+Z` + `bg` is how you recover from accidentally starting a long process in the foreground without `&`. The process is suspended → `bg` resumes it in background → you get your terminal back without killing the process. The `nohup` + `disown` combination is the production pattern for running long scripts when you need to disconnect from SSH — always redirect output too, since `nohup.out` fills up disk if you forget.

---

## QUICK REFERENCE — Everything at a Glance

### File Transfer:
```
rsync -avhP       → resumable, encrypted, large files
tar czf - | ssh   → streaming, no temp file
nc (netcat)       → fastest, trusted networks only
bbcp -P 16        → parallel streams, wire-speed
pv                → progress bar on any transfer
S3                → cross-cloud, no direct path
```

### User Management:
```
useradd -m -s /bin/bash user  → create user
passwd user                    → set password
usermod -aG sudo user          → sudo (Ubuntu)
usermod -aG wheel user         → sudo (RHEL/Amazon)
visudo                         → edit sudoers safely
gpasswd -d user sudo           → revoke access
usermod -L user                → lock account
userdel -r user                → delete user+home
```

### Bash Scripting:
```
${var:-default}    → default if unset
${var:?error}      → exit if unset
${var#prefix}      → remove prefix
${var%suffix}      → remove suffix
${var/old/new}     → replace first
${var//old/new}    → replace all
${#var}            → string length
trap cleanup EXIT  → always cleanup
while IFS= read -r line; done < file  → read file
[[ "$s" =~ pattern ]]  → regex match
${BASH_REMATCH[1]}     → capture group
```

### Text Processing:
```
sort -rn          → numeric reverse sort
sort -t',' -k2    → sort by CSV field 2
tr 'a-z' 'A-Z'   → uppercase
tr -d '\r'        → remove carriage returns
tee file.txt      → screen + file simultaneously
diff -u f1 f2     → unified diff
grep -E           → extended regex (preferred)
find -exec {} +   → batch execute (faster)
```

---

*Linux & Bash — Complete Gaps Reference*
*Covers everything missing from Q1–Q675*
*DevOps / SRE Interview Preparation — nawab312 GitHub repositories*
