# Crenox Pro

<!-- SEO: enterprise git secret scanner, on-premises secret detection, air-gapped secret scanner, pre-commit secret enforcement, gitguardian alternative, trufflehog enterprise alternative, gitleaks enterprise alternative, detect-secrets alternative, git-secrets, devsecops fleet governance, git history redaction -->

**Enterprise On-Premises Secret Detection and Pre-Commit Enforcement Platform**

[![Specification & Pre-Order](https://img.shields.io/badge/Specification%20%26%20Pre--Order-crenoxhq.github.io-0F172A?style=flat-square)](https://crenoxhq.github.io/crenox/pro.html)
[![License](https://img.shields.io/badge/License-Commercial%20EULA-B45309?style=flat-square)](COMMERCIAL_EULA.md)
[![Platform](https://img.shields.io/badge/Platform-Linux%20%7C%20macOS%20%7C%20Windows-334155?style=flat-square)](docs/DEPLOYMENT_GUIDE.md)

---

## Executive Summary

Crenox Pro is an on-premises secret detection and pre-commit security platform designed to prevent credentials, private keys, database connection strings, and sensitive API tokens from entering version control systems. It inspects staged Git buffers in workstation memory prior to commit finalization.

The system operates entirely within customer-managed infrastructure without external network dependencies. Secret detection, policy distribution, and audit verification execute on local systems.

---

## Core Capabilities

- **In-Memory Pre-Commit Inspection**: Evaluates staged Git buffers and index entries directly in workstation memory prior to commit completion. Source code, file contents, and staged diffs remain on the local machine.
- **Fail-Closed Policy Enforcement**: Halts commits immediately upon detecting sensitive tokens or policy violations. Detection combines signature rules, randomness evaluation, and contextual filtering to reduce false positives.
- **Hook and Process Verification**: Monitors Git hook configurations and detects commit bypass attempts (such as `--no-verify`), recording integrity alerts to the audit log and rolling back unverified commits that contain secrets.
- **Centralized Policy Governance**: Administrators define, test, and distribute rules, allowlists, and severity thresholds across workstation fleets from a central management console.
- **Supervised Exemption Lifecycle**: Provides a structured workflow for developers to request commit-scoped or time-bounded exemptions subject to administrative review and cryptographic authorization.
- **Repository History Redaction**: Includes a dedicated history-rewriting utility to identify and redact exposed credentials across past commits and branches while maintaining repository graph integrity.
- **Audit Logging and Compliance**: Records blocked commits, scan findings, policy synchronization, and exemption events to an append-only, tamper-evident audit ledger for compliance verification.

### Architectural Comparison

| Dimension | Crenox Pro (On-Premises Fleet) | Standalone CLI (Gitleaks, TruffleHog, detect-secrets, git-secrets) | Cloud SaaS (GitGuardian, Remote Post-Push) |
| :--- | :--- | :--- | :--- |
| **Execution Boundary** | 100% Customer-Managed / Air-Gapped | Local Unmanaged CLI | External Third-Party Cloud |
| **Interception Stage** | Pre-Commit (In-Memory) | Manual Hook / CI | Post-Push (After Secret Enters Git) |
| **Policy & Rule Sync** | Centralized Signed Distribution | Per-Repository Local Config | Cloud Dashboard |
| **Bypass & Exemption Control** | Hook Integrity Guard + Supervised JIT Exemptions | Bypassed via `git commit --no-verify` | Post-Incident Alert Only |
| **Runtime Footprint** | 2 Statically Compiled Go Binaries | Go / Python / Perl Runtimes | Cloud Webhooks & Agents |

---

## System Architecture

Crenox Pro operates as an integrated system consisting of two statically compiled binaries:

```
┌────────────────────────────────────────────────────────────────────────┐
│                   Customer-Controlled Infrastructure                   │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │               Central Controller (crenox-master)               │   │
│   │                                                                │   │
│   │  • Central Policy Authority       • Fleet & Seat Management    │   │
│   │  • Administrative Console         • OIDC/SAML 2.0 Auth         │   │
│   │  • Persistent Audit Storage       • Signed Agent Updates       │   │
│   │  • Real-Time Push Stream (SSE)    • Monotonic Event Journal    │   │
│   └──────────────────────┬──────────────────▲──────────────────────┘   │
│                          │                  │                          │
│        Real-Time Push    │                  │  Offline Catch-Up,       │
│        Events & Policies │                  │  Audit & Telemetry Logs  │
│        (/fleet/stream)   ▼                  │                          │
│   ┌─────────────────────────────────────────┴──────────────────────┐   │
│   │             Developer Workstations (crenox-agent)              │   │
│   │                                                                │   │
│   │  • Git Hook Interception          • In-Memory Buffer Analysis  │   │
│   │  • Fail-Closed Execution          • History Redaction Engine   │   │
│   │  • Post-Commit Watchdog           • Local Pre-Commit Hooks     │   │
│   │  • Real-Time Event Daemon         • Monotonic Sequence Sync    │   │
│   └────────────────────────────────────────────────────────────────┘   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 1. Central Management Controller (`crenox-master`)
- Serves as the central policy distribution authority and audit repository.
- Provides an administrative web console and REST API without third-party CDN or asset dependencies.
- Broadcasts policy updates, approved exemptions, and seat revocations over persistent Server-Sent Events (SSE) streams.
- Maintains a monotonic event journal enabling disconnected or offline workstations to catch up upon reconnecting.
- Supports enterprise authentication via OpenID Connect (OIDC) and perimeter reverse proxy headers.
- Distributes cryptographically signed policies, exemption tokens, and binary updates.
- Binds strictly to configured private internal network interfaces.

### 2. Workstation Enforcement Agent (`crenox-agent` / `crenox`)
- Evaluates staged Git buffers, paths, and filenames locally in memory during commit operations without blocking on network requests.
- Enforces pre-commit, commit-msg, post-commit, pre-merge-commit, and post-merge hooks locally.
- Runs a background push listener (`crenox daemon`) receiving policy updates and approved exemptions.
- Monitors configured hook paths to detect unauthorized configuration changes.
- Normalizes character encodings and Unicode variations prior to rule evaluation.
- Restricts uninstallation and configuration modifications to authorized administrative roles.

---

## Supported Environments & Prerequisites

### Server Requirements (`crenox-master`)
| Specification | Minimum Requirement (Up to 500 Seats) | Recommended (Up to 5,000 Seats) |
| :--- | :--- | :--- |
| **Operating System** | Linux, macOS, Windows | Linux (Ubuntu, RHEL, Rocky Linux) |
| **CPU / RAM** | 2 vCPU / 2 GB RAM | 4 vCPU / 4 GB RAM |
| **Storage** | 20 GB persistent storage | 50 GB persistent storage |
| **Network** | TCP port (default: `:8443`) | Reverse proxy with TLS termination |
| **Dependencies** | None (statically linked binary) | None |

### Workstation Requirements (`crenox-agent`)
| Specification | Requirement |
| :--- | :--- |
| **Operating System** | Linux, macOS, Windows |
| **Version Control** | Git 2.20 or later |
| **Memory / CPU** | Standard workstation hardware |
| **Network** | TCP access to internal `crenox-master` URL |

---

## Quick Start Guide

### 1. Deploying the Master Controller

1. **Extract Host Node Fingerprint**: Each commercial license is bound to a designated master server host. On the target machine, run:
   ```bash
   ./crenox-master fingerprint
   ```
2. **Obtain License File**: Transmit the generated fingerprint string to the Crenox Commercial Desk (`crenoxhq@proton.me`) to receive your signed commercial license file (`crenox.lic`).
3. **Start the Controller Service**:
   ```bash
   ./crenox-master serve \
     -port :8443 \
     -db /var/lib/crenox/crenox-master.db \
     -license /etc/crenox/crenox.lic \
     -admin-pass "<YOUR_SECURE_ADMIN_PASSWORD>" \
     -agent-key "<ENTERPRISE_CLUSTER_SECRET_KEY>" \
     -auto-tls
   ```
4. Access the administrative console at `https://localhost:8443` (or your configured internal domain).

### 2. Provisioning Developer Workstations

1. Generate an enrollment token from the administrative console or REST API.
2. Install the workstation agent from the internal master controller:

   **Linux / macOS:**
   ```bash
   curl -k -sSL https://crenox-master.internal:8443/install.sh | bash -s -- \
     --token crx_seat_<48_HEX_TOKEN> \
     --role "Developer"
   ```
   *(Note: Include `-k` when connecting to a master controller running with `-auto-tls` self-signed certificates).*

   **Windows (PowerShell):**
   ```powershell
   & ([scriptblock]::Create((iwr -useb https://crenox-master.internal:8443/install.ps1).Content)) `
     -Token crx_seat_<48_HEX_TOKEN> `
     -Role "Developer"
   ```

3. Activate security enforcement in your local Git repository:
   ```bash
   cd /path/to/repository
   crenox hook install
   ```

---

## Normal Developer Workflow

Once installed, Crenox Pro operates within standard Git workflows:

```bash
# Normal commit workflow
git add .
git commit -m "feat: implement payment gateway integration"
```

- If **no secrets are detected**, the commit completes normally without delay.
- If **a secret is detected**, the commit is immediately blocked with diagnostic details:
  ```
  crenox: Pre-Commit Security Guard
  crenox: secrets detected in staged changes

    [blocked] CRITICAL: Stripe Live Secret Key
      file:    services/payment/client.go:14
      rule:    stripe-live-secret
      token:   <sk_live_redacted_890>
      context: const StripeKey = "<sk_live_redacted_890>"

  Remediation:
    1. Remove secret from code or use environment variables
    2. Unstage changes: git reset HEAD <file>
    3. Request exemption: crenox request-bypass --file <file> --reason "..." --token "<secret>"
  ```

---

## Documentation Directory

Comprehensive technical guides and reference manuals are located in the [`docs/`](docs/) directory:

- [**Deployment & Operations Guide**](docs/DEPLOYMENT_GUIDE.md): Architecture, systemd service units, TLS proxy configuration, and persistent database maintenance.
- [**Developer & Workstation Guide**](docs/DEVELOPER_GUIDE.md): Workstation enrollment, global Git hook lifecycle via `core.hooksPath`, exemption workflows, and history scanning.
- [**Policy & Signature Governance**](docs/POLICY_CONFIGURATION.md): Declarative policy schema, custom signature syntax, hierarchical policy merging, and entropy threshold tuning.
- [**Remediation & History Purge Guide**](docs/REMEDIATION_AND_PURGE.md): Step-by-step procedures for token redaction across Git history, safety backups, and reflog cleanup.
- [**Command-Line Reference**](docs/CLI_REFERENCE.md): Parameter, flag, and environment variable reference for `crenox-master` and `crenox-agent`.
- [**REST API & Enterprise SSO Guide**](docs/API_AND_SSO_GUIDE.md): REST API endpoints and Single Sign-On configuration (OIDC/SAML).
- [**Security Policy & Vulnerability Reporting**](SECURITY.md): Security architecture boundaries and coordinated vulnerability disclosure procedures.
- [**Release Changelog**](CHANGELOG.md): Version history and release notes.

---

## Commercial Licensing & Support

Crenox Pro is proprietary commercial software licensed per authorized workstation seat under the [Commercial End User License Agreement (EULA)](COMMERCIAL_EULA.md).

For licensing inquiries, seat expansion, or technical support:
- **Enterprise Specification & Pre-Order Desk**: [https://crenoxhq.github.io/crenox/pro.html](https://crenoxhq.github.io/crenox/pro.html)
- **Open-Source Core Repository**: [https://github.com/crenoxhq/crenox](https://github.com/crenoxhq/crenox)
- **Commercial Inquiries & Support**: `crenoxhq@proton.me`
- **Diagnostic Bundles**: Use `crenox-master bug-report` to generate a sanitized, redacted diagnostic bundle for transmission to technical support.
