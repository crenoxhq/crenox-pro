# Crenox Pro — Developer & Workstation Operations Guide

This guide provides software engineers, DevOps specialists, and developers with practical instructions for installing, configuring, and operating the Crenox Pro workstation client (`crenox-agent`, aliased as `crenox`).

---

## 1. Workstation Architecture & Prerequisites

Crenox Pro runs locally on developer workstations to inspect staged code buffers in memory prior to commit creation, preventing sensitive credentials from being committed to version control.

### System Requirements
- **Supported Platforms**: Linux (`x86_64`, `arm64`), macOS (Intel & Apple Silicon), Windows 10/11 (`x86_64`).
- **Workstation Overhead**: Lightweight runtime footprint with minimal resource consumption.
- **Dependencies**: Zero external dependencies; Git 2.20 or later must be installed in your system `PATH`.
- **Network**: Periodic TCP connectivity to your organization's internal `crenox-master` server.

---

## 2. Installation & Workstation Enrollment

### Option A: Automated One-Liner (Recommended)

Run the bootstrap command provided by your security administrator. This command downloads the binary, enrolls the workstation seat, and sets up your environment:

**Linux / macOS (POSIX Shell):**
```bash
curl -k -sSL https://crenox-master.internal:8443/install.sh | bash -s -- \
  --token crx_seat_6d3233bd1930002121d6ae9fb0e5b8d9bb6267d24c60e79a
```

> [!NOTE]
> - When connecting to a master server running with `-auto-tls` (self-signed TLS certificate), include `-k` in `curl` (the Web Console **Copy** button automatically includes `-k` for HTTPS/self-signed servers).
> - Individual Seat Enrollment Tokens (`crx_seat_...`) authorize workstation registration directly without requiring the cluster `-agent-key`. If enrolling a headless CI/CD runner using a shared cluster key instead of a seat token, pass `--agent-key <cluster-key>`.

**Windows (PowerShell 5.1+ / Core):**
```powershell
& ([scriptblock]::Create((iwr -useb https://crenox-master.internal:8443/install.ps1).Content)) -Token crx_seat_6d3233bd1930002121d6ae9fb0e5b8d9bb6267d24c60e79a
```
*(If the master uses self-signed TLS certificates, prefix the command with `[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}; `)*

---

### Option B: Manual Installation

1. Download the statically compiled `crenox` binary from the master controller (authentication required):
   ```bash
   # Download agent binary authenticated via enrollment token
   curl -sSL "https://crenox-master.internal:8443/api/v1/download/agent?os=linux&arch=amd64&token=<ENROLLMENT_TOKEN>" \
     -o /usr/local/bin/crenox
   chmod +x /usr/local/bin/crenox
   ```
   *(Note: Authentication is required. You can authenticate via the `token` parameter, the `X-Enroll-Token: <ENROLLMENT_TOKEN>` header, or an ephemeral single-use ticket via `POST /api/v1/download/ticket?token=<ENROLLMENT_TOKEN>`. Anonymous requests return `HTTP 401 Unauthorized`).*

2. Enroll your workstation with the central controller:
   ```bash
   crenox enroll \
     --master https://crenox-master.internal:8443 \
     --token crx_seat_6d3233bd1930002121d6ae9fb0e5b8d9bb6267d24c60e79a \
     --fingerprint SHA256:f83f8e2b16d5f5103bf5908760a794f2f4aaed15187099e4ba0868d867902f1b \
     --role "Developer" \
     --department "Core Engineering"
   ```
   *(Note: When connecting to master instances configured with `-auto-tls`, add the `--insecure` flag or set `CRENOX_CA_CERT`).*

Upon enrollment:
- The workstation is registered and cryptographically bound to its hardware identity.
- Single-use enrollment credentials are automatically discarded upon registration.
- Workstation security profiles are initialized with restricted access controls.

---

## 3. Git Hook Configuration

To activate pre-commit interception in your code repositories:

### Per-Repository Installation
Navigate to your Git repository and run:
```bash
cd /path/to/project
crenox hook install
```

### Global Installation (All Repositories)
To enforce secret scanning across every Git repository on your workstation automatically:
```bash
crenox hook install --global
```
This configures Git's global `core.hooksPath` setting, directing Git to execute Crenox hooks for all existing and newly cloned repositories.

### Verifying Hook Status
Confirm that security hooks are active and healthy:
```bash
crenox hook verify
```
Expected output (in a repository with global hooks configured):
```
crenox: global hooks active (core.hooksPath: /home/developer/.crenox/hooks)
crenox: hooks active (hooks/pre-commit, hooks/commit-msg, hooks/post-commit, hooks/pre-merge-commit, hooks/post-merge)
```
*(If local per-repository hooks are configured without global hooks, the first line is omitted).*

---

## 4. Daily Developer Workflow

### Understanding Staged vs. Unstaged Changes
Crenox pre-commit enforcement and default `crenox scan` inspect the Git staging area (Git Index / `git diff --cached`):
- **Staged Files**: When you stage files using `git add <file>`, Crenox evaluates the staged diff buffers directly in memory prior to commit finalization.
- **Unstaged Files**: If files are modified in the working tree without being staged (`git add`), they are not included in the commit. Running `crenox scan` or `git commit` when nothing is staged will report `clean (0 files scanned)` because the Git staging index is empty.
- **Unstaged Notice**: If `crenox scan` detects unstaged changes in the working directory while the staging index is empty, it displays an informational notice:
  `crenox: notice: Working tree has unstaged modifications. Stage changes with 'git add <file>' to include them in pre-commit scan.`
- Always stage modified files with `git add <file>` before committing or scanning staged changes.

---

### Scenario A: Clean Commit (Normal Flow)
When you commit clean code containing no credentials:

```bash
git add services/auth/login.go
git commit -m "feat: enhance session refresh rate"
```

The hook evaluates your staged changes in-memory. The commit completes normally without delay.

---

### Scenario B: Blocked Commit (Credential Detected)
If you accidentally commit a file containing a sensitive token or private key:

```bash
git add services/payment/stripe.go
git commit -m "feat: add stripe integration"
```

Crenox intercepts the commit, halts the operation, and displays structured diagnostic feedback:

```
crenox: Pre-Commit Security Guard
crenox: secrets detected in staged changes

  [blocked] CRITICAL: Stripe Live Secret Key
    file:    services/payment/stripe.go:18
    rule:    stripe-live-secret
    token:   <sk_live_redacted_890>
    context: const StripeApiKey = "<sk_live_redacted_890>"

Remediation:
  1. Remove secret from code or use environment variables
  2. Unstage changes: git reset HEAD <file>
  3. Request exemption: crenox request-bypass --file <file> --reason "..." --token "<secret>"
```

---

### Scenario C: Remediating the Violation
To resolve the block and complete your commit:

1. **Remove the credential** from `services/payment/stripe.go` and replace it with an environment variable lookup:
   ```go
   var StripeApiKey = os.Getenv("STRIPE_API_KEY")
   ```
2. **Re-stage the file**:
   ```bash
   git add services/payment/stripe.go
   ```
3. **Commit cleanly**:
   ```bash
   git commit -m "feat: add stripe integration"
   ```

---

### Scenario D: Secret Committed as a Filename
If a file or directory is staged whose path or name contains a secret credential (e.g. `ghp_SAMPLE_TOKEN_PLACEHOLDER.env` or `backup_AKIA_SAMPLE_TOKEN.tar.gz`), Crenox intercepts the commit before Git can write the tree:

```bash
touch ghp_SAMPLE_TOKEN_PLACEHOLDER.env
git add ghp_SAMPLE_TOKEN_PLACEHOLDER.env
git commit -m "chore: add env template"
```

Crenox evaluates both staged content modifications and staged file paths, blocking the commit immediately:

```
crenox: Pre-Commit Security Guard
crenox: secrets detected in staged changes

  [blocked] CRITICAL: GitHub Personal Access Token (classic) (detected in staged filename)
    file:    ghp_SAMPLE_TOKEN_PLACEHOLDER.env:1
    rule:    github-pat-classic
    token:   <ghp_redacted_000>
    context: [FILENAME] <ghp_redacted_000>.env

Remediation:
  1. Remove secret from code or use environment variables
  2. Unstage changes: git reset HEAD <file>
  3. Request exemption: crenox request-bypass --file <file> --reason "..." --token "<secret>"
```

---

## 5. Exemption & JIT Bypass Lifecycle

When you are intentionally working with authorized mock fixtures, synthetic test data, or encrypted credentials that trigger a finding, you can request an exemption through the Just-In-Time (JIT) bypass workflow.

### Step 1: Submit Exemption Request
Submit a bypass request from your terminal (note: by default `skip_test_files: true` automatically skips `*_test.go` and `test/` paths, so bypasses are only needed for scanned files or when `skip_test_files` is `false`):

```bash
crenox request-bypass \
  --file services/payment/client.go \
  --reason "Mock key for local integration harness; not a live production credential" \
  --token "<secret-token>"
```

The agent hashes the token locally using a secure one-way hash (the plaintext secret is never transmitted across the network) and submits the request to the central controller.

### Step 2: Administrative Review
Your security administrator or SecOps team reviews the request in the Crenox Master Web Console and approves it for either:
- **Single Commit**: Automatically expires as soon as the commit is created.
- **Time-Limited**: Valid for a designated window (e.g., 7 days) on that specific repository branch.

### Step 3: Commit Execution
Once approved, the central controller distributes the signed exemption authorization to your workstation over the event stream. During Git commits, the hook evaluates staged buffers in local memory without blocking on network requests, verifies the cryptographic signature from local storage, and permits the commit to complete.

---

## 6. Historical Codebase Scanning

You can inspect existing commits, feature branches, and full repository histories for historical leaks:

```bash
# Scan full repository history across all branches
crenox scan --history --all-refs --workers 8

# Scan commits on current branch created in the last 14 days
crenox scan --history --since 2026-09-15
```

> [!NOTE]
> Historical scans configure `core.bigFileThreshold=1g` (files up to 1GB). Skipped binary or oversized files emit a warning: `[WARN] git history: skipped large file or binary diff: <path>`.

---

## 7. Complex Git Operations & Fail-Closed Guarantees

### Git Merges & Conflict Resolution
During merge conflict resolutions (`git merge`), Crenox automatically disables `DiffOnly` mode and evaluates entire staged files. This prevents conflicts that accidentally re-introduce deleted secrets from slipping through.

### Anti-Bypass Watchdog (`--no-verify`)
If a developer attempts to bypass local pre-commit checks using `git commit --no-verify`:
1. The `post-commit` watchdog hook inspects the commit immediately after creation.
2. It detects the absence of a valid pre-commit verification receipt.
3. If the commit is clean, it logs a `NO_VERIFY_VIOLATION` event to the local tamper-evident audit ledger and synchronizes the violation to the Master security console.
4. If the unverified commit contains leaked secrets, it automatically rolls back the commit from branch history (while preserving the working tree) and logs a `SCAN_LEAKED_COMMIT_ROLLED_BACK` event.

### Concurrent Commit Integrity (Race Protection)
In shared repositories or parallel build environments, Crenox binds pre-commit scan receipts to the executing Git process ID (PID). The `commit-msg` hook re-verifies staged tree digest integrity prior to commit object creation, aborting execution (exit code `1`) if concurrent index mutations occur.

### `core.hooksPath` Override Tamper Protection & Auto-Healing
If a developer or script attempts to bypass scanning by overriding Git's hook directory (e.g. `git config core.hooksPath <empty-dir>` or pointing to an unmanaged location):
1. **Verification Detection**: `crenox hook verify` detects the non-standard `core.hooksPath` setting and warns of the integrity breach.
2. **Auto-Healing**: The background daemon (`crenox daemon`), `crenox sync`, and hook execution automatically detect configuration drift, repair the repository hooks, and re-anchor security enforcement.
3. **Audit Telemetry**: The tampering attempt is logged as an integrity alert in the local and central audit ledgers.

### Local Audit Log Integrity
The local audit log includes offline tamper detection. If any local audit records are modified or removed offline, Crenox emits an integrity warning during execution and alerts the central console.

### Policy Verification and Auto-Healing
Local policy files (`policy.json`) are cryptographically verified prior to scan execution. If local modification is detected:
1. The agent logs a `TAMPER_DETECTED` event to the central controller.
2. The agent retrieves and restores the authoritative signed policy envelope from the master server.
3. Pre-commit evaluation resumes using the verified enterprise policy.

```text
crenox: [SECURITY WARNING] Local policy '/path/to/.git/crenox/policy.json' hash mismatch. File content was altered.
crenox: notice: Local changes to enterprise policies are prohibited. Restoring verified policy from Master...
crenox: [OK] Verified enterprise policy restored successfully.
```

If the master controller is unreachable to verify or restore policy, the operation terminates with exit code `3` (`INTEGRITY`).

### Fail-Closed Policy Enforcement
In environments configured for strict compliance, if the Master controller is unreachable and policy enforcement requires verification:
```
crenox: [SECURITY ERROR] Master server is unreachable and Fail-Closed offline policy enforcement is active. Commit rejected.
```
Commits are safely blocked until policy compliance can be verified, ensuring no uninspected code bypasses enforcement.

### Working Tree Protection During History Purge (`crenox purge`)
`crenox purge` verifies the working tree is clean (`git status --porcelain`) before rewriting history. If uncommitted edits or untracked files exist, it aborts (exit code `1`) to prevent data loss. Stash changes (`git stash -u`) before purging, or pass `--force` (`-f`) to override. `--dry-run` is permitted on dirty trees.

### Background Event Stream Daemon
To maintain current enterprise policies and exemptions without network dependencies during commit execution, run the background daemon:

```bash
crenox daemon
```

The daemon establishes a persistent Server-Sent Events (SSE) push stream with `crenox-master` (`/api/v1/fleet/stream`). When administrators update policies or approve exemptions in the central console, updates are delivered to the workstation. If network connectivity is interrupted, the daemon reconnects with backoff and retrieves missed events using monotonic sequence tracking.
*(On managed systems, this is automatically configured as a user-level background service).*


---

[← Documentation Index](../README.md) · [Commercial EULA](../COMMERCIAL_EULA.md) · [Enterprise Specification & Pre-Order](https://crenoxhq.github.io/crenox/pro.html)
