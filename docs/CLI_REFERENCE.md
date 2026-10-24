# Crenox Pro — Command-Line Interface (CLI) Reference

This document provides the reference for all commands, parameters, environment variables, and exit codes for `crenox-master` and `crenox-agent` (aliased as `crenox`).

---

## 1. Global Conventions & Standards

- **Static Binary Architecture**: Both executables are compiled with pure Go (`CGO_ENABLED=0`) and have zero external runtime shared-library dependencies (`libc`, `glibc`, `musl`).
- **Standard Formatting**: Informational, success, and warning messages adhere to enterprise terminal conventions: `[OK]`, `[ERROR]`, `[WARN]`, `[INFO]`, `[BLOCKED]`.
- **Fail-Closed Security Posture**: If an unhandled I/O error, corrupt buffer, or security boundary violation occurs during pre-commit evaluation, the process terminates with a non-zero exit code (`1`), immediately halting the Git commit.

---

## 2. Central Controller Reference: `crenox-master`

The central controller executable manages fleet state, cryptographic policy distribution, audit persistence, and the embedded administration console. It uses standard Go flag syntax (`-flag` or `--flag`).

```bash
Usage:
  crenox-master <command> [options]
```

### Commands

### `crenox-master serve`
Launches the HTTP/HTTPS REST API and the embedded administration web console.

```bash
crenox-master serve [flags]
```

| Flag | Environment Variable | Default | Description |
| :--- | :--- | :--- | :--- |
| `-port`, `-bind` | (none) | `:8443` | TCP address or port to listen on (e.g. `:8443`, `127.0.0.1:8080`) |
| `-db` | (none) | `crenox-master.db` | Path to the persistent database file |
| `-license` | (none) | (none) | Path to signed enterprise `.lic` commercial license file (bound to host node fingerprint) |
| `-admin-pass` | `CRENOX_ADMIN_PASSWORD` | (none) | Master administrator console password |
| `-agent-key` | `CRENOX_AGENT_KEY` | (auto-generated) | Shared cluster API key for headless/CI registration (auto-generated and stored in DB if omitted) |
| `-tls` | (none) | `false` | Enable TLS/HTTPS listener |
| `-tls-cert` | (none) | (none) | Path to TLS certificate PEM file |
| `-tls-key` | (none) | (none) | Path to TLS private key PEM file |
| `-auto-tls` | (none) | `false` | Automatically generate and persist a self-signed TLS certificate (`auto-tls.crt` / `auto-tls.key`) if none is provided |

> [!NOTE]
> **First-Time Setup vs. Credential Recovery**:
> - **Initial Startup**: If `-admin-pass` (or `CRENOX_ADMIN_PASSWORD`) is omitted during initial database creation, `crenox-master` enters Interactive Setup Mode (`/setup`) and writes a one-time setup token to `<db-dir>/initial-setup.token` (required for remote non-loopback setup requests, and automatically deleted once setup completes). If `-admin-pass` is provided, it initializes the administrator credential immediately.
> - **Credential Recovery & Rotation**: Supplying `-admin-pass` (or `CRENOX_ADMIN_PASSWORD`) on subsequent startups against an existing database serves as an emergency credential recovery mechanism. It rotates the stored salted hash, emits a warning log, and invalidates all active web sessions.

**Example:**
```bash
crenox-master serve \
  -port :8443 \
  -db /var/lib/crenox/crenox-master.db \
  -license /etc/crenox/crenox.lic \
  -admin-pass "<ADMIN_PASSWORD>" \
  -agent-key "<CLUSTER_SECRET_KEY>" \
  -auto-tls
```

---

### `crenox-master init`
Initializes a new database file with default tables and factory policies without opening network listeners.

```bash
crenox-master init [-db <path>] [-license <path>]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `-db` | `crenox-master.db` | Target path where the database file will be initialized |
| `-license` | (none) | Path to cryptographic `.lic` license file (optional; auto-detects from database or local files) |

---

### `crenox-master status`
Inspects local database metrics, enrolled workstation counts, active seats, and license posture.

> [!NOTE]
> The database file must already be initialized (via `crenox-master init` or `crenox-master serve`) before running `crenox-master status`.

```bash
crenox-master status [-db <path>] [-license <path>]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `-db` | `crenox-master.db` | Path to the database file |
| `-license` | (none) | Path to license key file (optional; auto-detects from database or local files) |

---

### `crenox-master fingerprint`
Computes and outputs the host hardware fingerprint. Crenox Pro binds commercial licenses to this identifier to enforce capacity without requiring external network connectivity. Administrators transmit this output to CrenoxHQ (`crenoxhq@proton.me`) to generate or re-key a license file.

```bash
crenox-master fingerprint
```

*(No flags supported)*

**Example Output:**
```
Crenox Pro Master — Hardware Node Fingerprint
─────────────────────────────────────────────────────────────
  Node Host ID : CNX-9F4A-B1C2-88E0-44D1
  Hostname     : controller-prod-01.internal
  Architecture : linux/amd64
─────────────────────────────────────────────────────────────
Provide this Node Host ID to CrenoxHQ when requesting a node-locked
Enterprise Master license key.
```

---

### `crenox-master bug-report`
Generates a sanitized, air-gapped diagnostic archive (`.zip`) containing system telemetry, fleet statistics, and redacted error logs for technical support.

```bash
crenox-master bug-report [-db <path>] [-out <output.zip>] [-license <path>]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `-db` | `crenox-master.db` | Path to database file |
| `-license` | (none) | Path to `.lic` license file |
| `-out` | `crenox-diag-<timestamp>.zip` | Target path for the output diagnostic archive |

---

### `crenox-master version`
Displays version and compilation build information.

```bash
crenox-master version
```

---

## 3. Workstation Client Reference: `crenox-agent` / `crenox`

The developer workstation client (`crenox-agent`, aliased as `crenox`) intercepts Git operations, scans code buffers, and synchronizes state with the master controller. It uses standard POSIX flag conventions (`--flag` or `-f`).

```bash
Usage:
  crenox [command] [flags]
```

### Global Persistent Flags
| Flag | Default | Description |
| :--- | :--- | :--- |
| `--repo`, `-C` | `.` | Target Git repository working directory path |
| `--help`, `-h` | `false` | Display command help and usage parameters |
| `--version`, `-v` | `false` | Print Crenox Pro Agent version and build info |

---

### `crenox completion`
Generates shell autocompletion script for the specified shell.

```bash
crenox completion [bash|zsh|fish|powershell]
```

---

### `crenox enroll`
Enrolls the developer workstation with the central `crenox-master` server.

```bash
crenox enroll --master <URL> --token <TOKEN> [flags]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--master` | (Required) | Central controller URL (e.g. `https://crenox.internal:8443`) |
| `--token` | (Required) | Enrollment token issued by administrator |
| `--agent-key` | (none) | Master cluster agent API key (or set `CRENOX_AGENT_KEY`) |
| `--fingerprint` | (none) | Expected cryptographic fingerprint of master public key for authenticity pinning |
| `--role` | (none) | Seat role override (e.g., `Developer`, `SecOps`) |
| `--department` | (none) | Department assignment (e.g., `Engineering`, `FinTech`) |
| `--install-hooks` | `true` | Automatically install Git pre-commit hooks in current repository |
| `--insecure`, `-k` | `false` | Allow TLS connections without certificate verification (e.g. self-signed certificates) |

**Example:**
```bash
crenox enroll \
  --master https://crenox.internal:8443 \
  --token crx_seat_6d3233bd1930002121d6ae9fb0e5b8d9bb6267d24c60e79a \
  --agent-key crx_cluster_key_99182736 \
  --fingerprint SHA256:f83f8e2b16d5f5103bf5908760a794f2f4aaed15187099e4ba0868d867902f1b \
  --role "Developer" \
  --department "Billing Core"
```

> [!NOTE]
> **Ephemeral Provisioning**: Single-use enrollment tokens and temporary credentials are automatically discarded upon successful workstation registration.

---

### `crenox hook`
Manages Git security interception hooks.

#### `crenox hook install`
Installs pre-commit, commit-msg, post-commit, pre-merge-commit, and post-merge hooks.

```bash
crenox hook install [--bin <path>] [--global]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--bin` | (none) | Custom path to the `crenox-agent` executable invoked by hooks |
| `--global`, `-g` | `false` | Install hooks globally across all repositories via Git's `core.hooksPath` |

#### `crenox hook uninstall`
Removes Crenox hooks from the repository or globally.

```bash
crenox hook uninstall [--global]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--global`, `-g` | `false` | Uninstall globally (unsets `core.hooksPath`) |

> [!NOTE]
> Workstations enrolled with the `Developer` seat role are blocked by RBAC from executing `crenox hook uninstall`. Hook uninstallation requires administrative or `SecOps` privileges.

#### `crenox hook verify`
Validates that Crenox security hooks are actively installed and executable, and verifies that repository-level `core.hooksPath` configuration has not been diverted to bypass security checks.

```bash
crenox hook verify
```

#### Hook Execution Subcommands (Invoked by Git)
- `crenox hook pre-commit [--diff-only]`: Evaluates staged Git Index buffers in-memory as well as staged file paths and names. Defaults to `--diff-only=true`.
- `crenox hook commit-msg <path_to_msg_file>`: Scans commit message text, validates staged index integrity against process-bound pre-commit receipts, and self-heals tampered hooks.
- `crenox hook post-commit`: Compliance watchdog verifying HMAC scan receipt; detects `--no-verify` bypass attempts (logs `NO_VERIFY_VIOLATION` on clean commits, or automatically rolls back the commit and logs `SCAN_LEAKED_COMMIT_ROLLED_BACK` if secrets are present).
- `crenox hook pre-merge-commit`: Evaluates merged trees before merge completion (`--diff-only=false`).
- `crenox hook post-merge`: Watchdog verification following Git merge operations.

> [!NOTE]
> **Tamper Resistance**: Installed hook scripts are actively protected against unauthorized local modification or deletion.

---

### `crenox scan`
Scans repository staged changes or complete Git historical commits for exposed credentials. Inspects both file contents and staged file paths.

```bash
crenox scan [flags]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--staged` | `true` | Scans staged changes in the Git Index (including staged filenames) |
| `--diff-only` | `true` (staged) | Scans added or modified lines only rather than whole file buffers |
| `--history` | `false` | Enables historical commit scanning across revisions |
| `--branch` | (none) | Specific branch to scan during history mode (e.g. `main`) |
| `--all-refs` | `false` | Scans all reachable branches and references in history mode |
| `--since` | (none) | Scan commits starting on or after date (`YYYY-MM-DD`) |
| `--until` | (none) | Scan commits created on or before date (`YYYY-MM-DD`) |
| `--max-commits` | `0` (all) | Limit number of historical commits scanned |
| `--workers` | `GOMAXPROCS` | Number of concurrent scan worker threads |
| `--no-dedup` | `false` | Disable blob deduplication optimization during history scanning |

> [!NOTE]
> **Staged vs. Unstaged Changes**: Default scanning evaluates files staged in the Git Index (`git diff --cached`). If the staging index contains zero files but unstaged changes exist in the working directory, `crenox scan` outputs an informational notice:
> `crenox: notice: Working tree has unstaged modifications. Stage changes with 'git add <file>' to include them in pre-commit scan.`
>
> Historical scans configure Git with `core.bigFileThreshold=1g` (up to 1GB per file). Skipped binary or oversized files emit a warning: `crenox: [WARN] git history: skipped large file or binary diff: <path>`.

**Examples:**
```bash
# Scan current staged changes
crenox scan

# Scan full commit history across all branches
crenox scan --history --all-refs --workers 8

# Scan commits from the last 30 days on main
crenox scan --history --branch main --since 2026-09-01
```

---

### `crenox request-bypass`
Submits an exemption request for an authorized secret or test fixture.

```bash
crenox request-bypass --file <path> --reason <justification> --token <token> [flags]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--file`, `-f` | (Required) | Relative path to the file containing the secret |
| `--reason`, `-r` | (Required) | Justification for the bypass request |
| `--token`, `-t` | (Required) | Secret token string (bound to specific credential; hashed locally before transmission) |
| `--developer` | `$USER` | Developer username or email |
| `--branch` | (none) | Git branch name |
| `--scope` | `single_commit` | Exemption scope: `single_commit` or `temporary` |

**Example:**
```bash
crenox request-bypass \
  -f "services/billing/client.go" \
  -r "Mock Stripe fixture token for local integration harness" \
  -t "<secret-token>"
```

---

### `crenox list-bypass`
Lists all active and pending bypass requests for the current workstation.

```bash
crenox list-bypass [--json]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--json` | `false` | Formats output as structured JSON |

---

### `crenox audit`
Queries the local security audit log ledger.

```bash
crenox audit [flags]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--limit` | `50` | Maximum number of recent events to display |
| `--since` | (none) | Filter events created on or after date (`YYYY-MM-DD`) |
| `--json` | `false` | Formats output as structured JSON |

---

### `crenox purge`
Rewrites Git history to redact sensitive tokens across all commits.

```bash
crenox purge --token <TOKEN> [flags]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--token`, `-t` | (Required) | Secret tokens to redact (repeatable or comma-separated) |
| `--replacement`, `-r` | `[REDACTED_BY_CRENOX]` | Replacement string substituted in place of credentials |
| `--backup-id` | (none) | Custom identifier for safety backup snapshot |
| `--expire-reflog` | `false` | Clear Git reflog cache (`.git/logs`) |
| `--prune` | `false` | Remove safety backup references and reflogs completely |
| `--delete-backup` | `false` | Delete backup snapshot after successful purge |
| `--force`, `-f` | `false` | Bypass clean working tree check |
| `--dry-run` | `false` | Preview history rewrite operations without modifying repository |

> [!NOTE]
> `crenox purge` exits 1 if the working tree has uncommitted or untracked changes. Stash changes (`git stash -u`) or pass `--force` to override. `--dry-run` is allowed on dirty trees.

---

### `crenox purge-restore`
Restores a repository to a pre-purge safety snapshot.

```bash
crenox purge-restore [--backup-prefix <prefix>]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--backup-prefix` | (none) | Backup snapshot prefix (e.g. `refs/original/crenox-backup-...`) |

---

### `crenox daemon`
Executes a background service that maintains a persistent Server-Sent Events (SSE) connection with the central Master controller (`/api/v1/fleet/stream`), receiving and applying enterprise policy updates, approved exemptions, and security signals. The daemon reconnects with exponential backoff upon network interruptions, recovers missed events using monotonic sequence tracking, and flushes pending local audit queues.

```bash
crenox daemon [--interval <duration>]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--interval`, `-i` | `30s` | Heartbeat and audit queue flush interval |

---

### `crenox sync`
Synchronizes the local agent state with the master controller.

```bash
crenox sync [-C <repo_path>]
```

This operation performs the following tasks:
- Reports agent status and active policy hash (`applied_policy_hash`) via heartbeat.
- Flushes queued audit records to the master controller.
- Verifies the local policy cache against the master signature.
- Validates and restores local Git hook configurations.

**Output Example:**
```
crenox: synced with https://crenox-master.internal:8443 (0 events, 0 bypasses, policy verified: ef9bc6bac921)
```

> [!NOTE]
> The `policy verified: <HASH_12>` field displays the first 12 characters of the SHA-256 hash of the active signed policy envelope. If local policy modifications are detected, `crenox sync` emits a warning and restores the signed policy from the master controller.

---

### `crenox update`
Updates the Crenox Agent binary from the Master server or applies a local binary directly.

```bash
crenox update [flags]
```

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--master` | (none) | Alternate master controller URL |
| `--check-only` | `false` | Check if update is available without downloading |
| `--force` | `false` | Force update even if version is already current |
| `--binary` | (none) | Apply local compiled binary file directly |

> [!NOTE]
> Connecting to master servers serving self-signed TLS certificates requires setting `CRENOX_CA_CERT` with the trusted CA certificate path.

---

### `crenox version`
Displays compiled agent version and build information.

```bash
crenox version
```

---

## 4. Environment Variables Reference

| Variable | Target Binary | Description |
| :--- | :--- | :--- |
| `CRENOX_ADMIN_PASSWORD` | `crenox-master` | Administrator password for initial startup or emergency credential rotation |
| `CRENOX_AGENT_KEY` | `crenox-master`, `crenox-agent` | Cluster shared secret key required for agent registration |
| `CRENOX_CA_CERT` | `crenox-agent` | File path to custom enterprise root CA certificate bundle (PEM format) for TLS verification |
| `CRENOX_LICENSE` | `crenox-agent` | Path to commercial license file (`crenox.lic`) for standalone execution or CI/CD runners |
| `CRENOX_DEVICE_TOKEN` | `crenox-agent` | Workstation authentication token override for containerized or headless CI/CD runners |
| `CRENOX_MASTER_URL` | `crenox-agent` | Central controller URL for containerized or headless CI/CD runners |
| `CRENOX_CONFIG_DIR` | `crenox-agent` | Directory path override for agent configuration (default: `~/.crenox`) |

> [!NOTE]
> Credential variables are wiped from process memory upon startup to prevent environment exposure or child process inheritance.

---

## 5. Exit Codes & Error Classifications

Every command adheres strictly to standard POSIX process exit conventions:

| Exit Code | Classification | Meaning & Operational Context |
| :---: | :--- | :--- |
| **`0`** | `SUCCESS` | Operation completed cleanly. In `scan` and `hook pre-commit`, indicates zero secrets found. |
| **`1`** | `BLOCKED` | Security violation. Secrets or corrupt buffers detected during pre-commit evaluation. |
| **`2`** | `USAGE` | Invalid command syntax, unrecognized flags, or missing required arguments. |
| **`3`** | `INTEGRITY` | Cryptographic verification failure, policy alteration, binary seal mismatch, or invalid license. |
| **`4`** | `CAPACITY` | Enrolled seat quota exceeded on central controller. |


---

[← Documentation Index](../README.md) · [Commercial EULA](../COMMERCIAL_EULA.md) · [Enterprise Specification & Pre-Order](https://crenoxhq.github.io/crenox/pro.html)
