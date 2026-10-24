# Crenox Pro — Historical Remediation and Purge Runbook

This runbook outlines standard operating procedures for detecting, redacting, and permanently purging leaked credentials from Git repository histories using the Crenox Pro remediation engine.

---

## 1. Remediation Lifecycle Overview

When a credential or private key is inadvertently committed to a Git repository, revoking the credential at the provider (AWS, GitHub, Stripe) is the mandatory first step. However, removing the credential from the Git history is required to prevent perpetual exposure during audits, clones, and code sharing.

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│ 1. Incident     │       │ 2. Dry-Run      │       │ 3. Automated    │
│    Detection    ├──────>│    Simulation   ├──────>│    Safety       │
│ (Scan History)  │       │ (--dry-run)     │       │    Snapshot     │
└─────────────────┘       └─────────────────┘       └────────┬────────┘
                                                             │
┌─────────────────┐       ┌─────────────────┐       ┌────────▼────────┐
│ 6. Team & CI    │       │ 5. Upstream     │       │ 4. Topological  │
│    Rebase       │<──────┤    Forced Push  │<──────┤    Commit       │
│    Coordination │       │ (--force-with)  │       │    Redaction    │
└─────────────────┘       └─────────────────┘       └─────────────────┘
```

The remediation engine rewrites Git object trees topologically, substituting the targeted sensitive tokens with `[REDACTED_BY_CRENOX]` while strictly preserving original commit metadata, timestamps, author identities, and parentage graphs.

---

## 2. Step-by-Step Remediation Procedure

### Step 1: Historical Discovery & Audit
Before altering history, scan the repository to locate every commit, branch, and reference containing the exposed secret.

```bash
# Scan all branches and historical commits concurrently
crenox scan --history --all-refs --workers 8
```

Example terminal output:
```
crenox: Historical Repository Audit
crenox: 1 secret(s) found across 3 commits:

  [CRITICAL] Stripe Live Secret Key
    commit:  9a8f10b2 (2026-08-14 by developer@enterprise.com)
    file:    services/billing/stripe.go:42
    rule:    stripe-live-secret
    match:   [REDACTED_PREVIEW]

Remediation:
  Redact secrets from Git history:
    crenox purge --token="<secret>"
```

> [!NOTE]
> Historical scans configure `core.bigFileThreshold=1g` (files up to 1GB). Skipped binary or oversized files emit a warning:
> `crenox: [WARN] git history: skipped large file or binary diff: <file>`

---

### Step 2: Execute a Dry-Run Simulation
Simulate the history rewrite operation without modifying the Git object database or moving branch references.

```bash
crenox purge \
  --token "<secret-token>" \
  --dry-run
```

Example output:
```
crenox: Git History Sanitizer & Purge
  [OK] History purge simulation completed (dry-run)
  - Commits inspected:         1420
  - Commits to be rewritten:   3
  - Blobs to be sanitized:     3
  (No changes applied to Git repository; HEAD unmodified)
```

---

### Step 3: Execute Live Purge with Automated Backup
Execute the redaction. The engine automatically creates an immutable safety snapshot under `refs/original/` before rewriting any commit.

> [!IMPORTANT]
> `crenox purge` requires a clean working tree. If uncommitted or untracked changes exist, it exits 1 to prevent data loss. Stash changes (`git stash -u`) before purging, or pass `--force` to bypass. (`--dry-run` is allowed on dirty trees).

```bash
crenox purge \
  --token "<secret-token>" \
  --replacement "[REDACTED_BY_CRENOX]"
```

To purge multiple leaked tokens simultaneously:
```bash
crenox purge \
  -t "<token-1>" \
  -t "<token-2>" \
  --replacement "[REDACTED_BY_CRENOX]"
```

Example output:
```
crenox: Git History Sanitizer & Purge
  [OK] History purge completed
  - Commits inspected: 1420
  - Commits rewritten: 3
  - Blobs sanitized:   3
  - Restore point:     refs/original/crenox-backup-20260930-120000-8f9a2b1c
```

---

### Step 4: Verification of Sanitized History
Re-execute a historical scan to confirm that zero matching secrets remain in active branch commits or trees:

```bash
# Verify active branch commits
crenox scan --history
```

Expected output:
```
crenox: Historical Repository Audit
crenox: clean (1420 commits scanned)
```

> [!NOTE]
> Safety Backup References: By default, `crenox purge` creates safety snapshots under `refs/original/crenox-backup-.../` so that historical changes can be undone if needed via `crenox purge-restore`.
> If you execute `crenox scan --history --all-refs` before purging safety snapshots, the scanner will inspect the pre-purge commits inside `refs/original/`. Once verified, you can permanently remove the backup snapshots with `crenox purge --prune` or `crenox purge --delete-backup`, after which `crenox scan --history --all-refs` will report zero secrets across all Git references.

---

### Step 5: Upstream Synchronization
Because commit hashes have changed for all descendant commits, pushing the sanitized history to the remote repository requires a lease-protected force push.

> [!WARNING]
> Always use `--force-with-lease` rather than bare `--force` to prevent overwriting commits pushed by colleagues during the operation.

```bash
# Push sanitized branches to the remote upstream
git push origin --force-with-lease --all

# Push updated tags
git push origin --force-with-lease --tags
```

---

## 3. Disaster Recovery & Rollback Runbook

If a purge operation was executed with incorrect parameters or requires rollback, the repository can be restored to its pre-purge state using the safety snapshot.

### Step 1: Identify Available Restore Points
List existing safety snapshots created during previous purge executions:

```bash
git for-each-ref refs/original/
```

Output:
```
refs/original/crenox-backup-20260930-120000-8f9a2b1c/heads/main
refs/original/crenox-backup-20260930-120000-8f9a2b1c/heads/release/v1.0
```

### Step 2: Execute Rollback
Revert all branch and tag references to the selected backup snapshot:

```bash
crenox purge-restore --backup-prefix refs/original/crenox-backup-20260930-120000-8f9a2b1c
```

Example output:
```
crenox: auto-detected latest backup prefix: refs/original/crenox-backup-20260930-120000-8f9a2b1c
crenox: repository restored from refs/original/crenox-backup-20260930-120000-8f9a2b1c
```

---

## 4. Post-Remediation Team Coordination

When a repository history is rewritten, other team members with local clones must resynchronize their working trees to avoid re-introducing the old commit hashes.

### Instructions for Developers
Team members should execute the following sequence in their local clones:

```bash
# 1. Fetch updated references from upstream
git fetch origin

# 2. Rebase unpushed local work onto the sanitized upstream branch
git rebase origin/main

# Alternatively, for clean environments:
git reset --hard origin/main
```

---

## 5. Permanent Object Pruning & Reflog Expiration

By default, Git preserves unreachable historical commits in its local reflog (`.git/logs/`) for 30–90 days. For total eradication where the repository is audited by external compliance assessors:

```bash
# Clear reflogs, delete backup references, and prune unreachable loose objects
crenox purge \
  --token "<secret-token>" \
  --expire-reflog \
  --prune
```

This permanently eliminates backup snapshots under `refs/original/` and runs a garbage collection pass to purge dangling objects from the local `.git/objects/` store.


---

[← Documentation Index](../README.md) · [Commercial EULA](../COMMERCIAL_EULA.md) · [Enterprise Specification & Pre-Order](https://crenoxhq.github.io/crenox/pro.html)
