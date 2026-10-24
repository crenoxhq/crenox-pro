# Crenox Pro — Enterprise Policy Configuration Guide

This guide provides an authoritative reference for configuring detection rules, entropy thresholds, allowlists, and suppression policies across the Crenox Pro platform.

---

## 1. Policy Evaluation Hierarchy & Precedence

Crenox Pro resolves configuration directives through a centralized policy hierarchy. Security policies are governed from the central Master Web Dashboard; unmanaged local developer overrides are prohibited by design.

```
┌────────────────────────────────────────────────────────────────────────┐
│                   Tier 1: Safety Floors                                │
│   • Hard entropy cap (max 6.0)     • Zero wildcards allowed (*)        │
│   • Fail-closed on corrupt buffer  • Anti-self-leak (.crenox/ guard)   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Overrides
┌───────────────────────────────────▼────────────────────────────────────┐
│              Tier 2: Signed Master Policy                              │
│   • Enterprise rules distributed by crenox-master                      │
│   • Governed via Central Web Console / Dashboard                       │
│   • Cryptographically verified by workstation agents                   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Overrides
┌───────────────────────────────────▼────────────────────────────────────┐
│               Tier 3: Built-in Signature Catalogue                     │
│   • Pre-compiled signature rules catalog (341 rules)                   │
└────────────────────────────────────────────────────────────────────────┘
```

### Centralized Policy Governance
- **Centralized Administration**: Workstation agents enforce policies received from the master controller. All policy definitions, custom rules, allowlists, and scan engine parameters are managed by administrators via the Master Web Dashboard.
- **Policy Verification**: Policies are cryptographically signed by the central controller. Workstations verify policy integrity on execution; unauthorized local modifications are overwritten with the authoritative policy upon synchronization.
- **Suppression Restrictions**: If the master policy sets `require_secops_approval: true`, developers cannot bypass findings locally without administrative approval through the JIT exemption workflow.

---

## 2. Universal Enterprise Ruleset: `crenox-pro.yaml`

The central master server distributes policy updates using the authoritative `Ruleset` schema. Administrators can author and tune this configuration directly inside the Master Web Dashboard, or import/export it as a declarative YAML file (`crenox-pro.yaml`) via the **Engine Settings** panel.

When a policy YAML file is imported:
1. The controller validates the ruleset schema and enforces immutable safety bounds.
2. The configuration is persisted and cryptographically signed by the Master server.
3. Updated policies propagate fleet-wide to all enrolled workstations during the next synchronization cycle.

```yaml
version: "1.0"
organization: "Acme Global Financial"
updated_at: "2026-09-30T12:00:00Z"

# 1. Global Rule Disablement (By Rule ID)
disabled_rules:
  - "stripe-test-secret"          # Allow developer test keys in staging

# 2. Rule Severity Overrides (LOW, MEDIUM, HIGH, CRITICAL)
severity_overrides:
  aws-access-key: "CRITICAL"
  slack-webhook: "MEDIUM"
  github-pat-classic: "CRITICAL"
  jwt: "LOW"

# 3. Organization-Specific Custom Signatures
# Custom rules require an identifiable prefix anchor. The system automatically derives
# prefixes from pattern definitions (including alternations). Signatures without a valid prefix
# are rejected during policy validation.
custom_signatures:
  - id: "core-banking-api-token"
    description: "Core Banking High-Privilege Transaction Token"
    prefix: "cnxbnk_"             # Anchor prefix
    regex: "^cnxbnk_[a-zA-Z0-9]{32,64}$"
    severity: "CRITICAL"
    enabled: true

  - id: "internal-jwt-signing-secret"
    description: "Internal Microservice HMAC-SHA256 Signing Secret"
    prefix: "sec_live_"
    regex: "^sec_live_[0-9a-fA-F]{64}$"
    severity: "CRITICAL"
    enabled: true

# 4. Global Allowlists (Tokens, File Paths, and Regex Patterns)
# Allowlisted secrets are masked on read (e.g., QAWL********efgh) and safely preserved on update.
# Plaintext revelation requires administrative step-up authentication.
# Note: Wildcard characters (*, ?, [], {}) are not permitted in tokens or patterns.
allowlist:
  tokens:
    - "approved_mock_token_1234567890"
    - "<aws-example-key-id>"      # Standard AWS documentation fixture
  paths:
    - "docs/samples/"
    - "test/fixtures/"
  patterns:
    - "test_token_approved_prefix"

# 5. Core Scan Engine Parameters
scan_engine:
  entropy_threshold: 4.5          # Default: 4.5 (Valid range: 2.0 to 6.0)
  min_secret_length: 12           # Minimum token length before entropy calculation (Valid range: 4 to 64)
  exclude_extensions:
    - ".lock"
    - ".sum"
    - ".min.js"
    - ".min.css"
    - ".map"
  exclude_paths:
    - "vendor/**"
    - "node_modules/**"
    - "dist/**"
    - "build/**"
  false_positive_mode: "balanced" # Options: strict | balanced | permissive
  skip_test_files: true           # Automatically skip test directories

# 6. Custom Alert Notifications & Remediation Instructions
# Note: YAML import accepts {message, contact} objects; JSON API (GET/PUT /api/v1/config) stores map[rule_id]message_string.
notifications:
  aws-access-key:
    message: "CRITICAL: Live AWS credentials detected. Rotate access keys in AWS IAM immediately."
    contact: "secops@acme.com"
  core-banking-api-token:
    message: "BANKING BREACH: Production transaction secret detected. Incident ticket opened."
    contact: "fintech-incident-response@acme.com"

# 7. Developer Suppression & Bypass Governance
suppression_policy:
  allow_inline_suppression: false # When false, inline comments (e.g., // crenox:ignore) are rejected
  require_secops_approval: true   # Requires JIT bypass request via Master Console
  audit_all_suppressions: true    # All exemptions recorded in immutable audit ledger

# 8. Security Enforcement (Offline Fail-Closed & Global Hooks)
security_enforcement:
  fail_closed_offline: false      # When true, reject commits if Master server is unreachable
  enforce_global_hooks: true      # Enforces and re-anchors Git core.hooksPath fleet-wide
```

> [!NOTE]
> By default, `GET /api/v1/config/export` exports allowlist tokens in masked format. To export tokens in plaintext for backup or migration, specify `?unmasked=true`.

---

## 3. Built-in Detection Signature Index

The engine comes equipped with 341 pre-compiled signatures organized into 9 enterprise security categories:

### 3.1 Cloud Providers & Infrastructure
| Rule ID | Description | Prefix Filter | Default Severity |
| :--- | :--- | :--- | :--- |
| `aws-access-key` | AWS Access Key ID | `AKIA` | `CRITICAL` |
| `aws-sts` | AWS Temporary STS Key | `ASIA` | `CRITICAL` |
| `aws-mfa-key` | AWS MFA Device Serial | `ABIA` | `CRITICAL` |
| `gcp-service-account`| GCP Service Account Private Key (JSON) | `"type": "service_account"` | `CRITICAL` |
| `gcp-api-key` | Google Cloud API Key | `AIza` | `HIGH` |
| `azure-storage-key` | Azure Storage Account Shared Access Key | Heuristic | `CRITICAL` |

### 3.2 Source Control & CI/CD Platforms
| Rule ID | Description | Prefix Filter | Default Severity |
| :--- | :--- | :--- | :--- |
| `github-pat-classic` | GitHub Personal Access Token (Classic) | `ghp_` | `CRITICAL` |
| `github-oauth` | GitHub OAuth Access Token | `gho_` | `CRITICAL` |
| `github-app-install`| GitHub App Installation Token | `ghs_` | `CRITICAL` |
| `github-pat-fine` | GitHub Fine-Grained Personal Access Token | `github_pat_` | `CRITICAL` |
| `gitlab-pat` | GitLab Personal Access Token | `glpat-` | `CRITICAL` |
| `gitlab-pipeline` | GitLab Pipeline Trigger Token | `glptt-` | `HIGH` |
| `gitlab-runner` | GitLab Runner Registration Token | `GR1348941` | `HIGH` |
| `docker-pat` | Docker Personal Access Token | `dckr_pat_` | `CRITICAL` |

### 3.3 SaaS, APIs & Payment Gateways
| Rule ID | Description | Prefix Filter | Default Severity |
| :--- | :--- | :--- | :--- |
| `stripe-live-secret`| Stripe Production API Key | `sk_live_` | `CRITICAL` |
| `stripe-live-restricted`| Stripe Restricted Access Key | `rk_live_` | `CRITICAL` |
| `slack-bot-token` | Slack Bot User Token | `xoxb-` | `CRITICAL` |
| `slack-user-token`| Slack User Workspace Token | `xoxp-` | `CRITICAL` |
| `openai-key` | OpenAI API Secret Key | `sk-` | `HIGH` |
| `openai-project-key`| OpenAI Scoped Project API Key | `sk-proj-` | `CRITICAL` |
| `anthropic-key` | Anthropic Claude API Key | `sk-ant-` | `CRITICAL` |
| `twilio-auth-token`| Twilio Account Authorization Token | `SK` | `HIGH` |
| `sendgrid-key` | SendGrid Mail API Key | `SG.` | `HIGH` |
| `cloudflare-api-token` | Cloudflare API Token | `CLOUDFLARE_TOKEN` | `HIGH` |
| `cloudflare-api-key` | Cloudflare API Key | `CLOUDFLARE_API_KEY` | `HIGH` |
| `1password-secret-key`| 1Password Enterprise Secret Key | Heuristic | `CRITICAL` |

### 3.4 Cryptographic Keys & Blockchain Seeds
| Rule ID | Description | Prefix Filter | Default Severity |
| :--- | :--- | :--- | :--- |
| `rsa-private-key` | RSA Private Key (PEM) | `BEGIN RSA` + `PRIVATE KEY` | `CRITICAL` |
| `ec-private-key` | Elliptic Curve Private Key (PEM) | `BEGIN EC` + `PRIVATE KEY` | `CRITICAL` |
| `openssh-private-key`| OpenSSH Format Private Key | `BEGIN OPENSSH` + `PRIVATE KEY` | `CRITICAL` |
| `pkcs8-private-key`| PKCS#8 Generic Private Key | `BEGIN` + `PRIVATE KEY` | `CRITICAL` |
| `pgp-private-key` | PGP ASCII Armored Private Key | `BEGIN PGP` + `PRIVATE KEY BLOCK` | `CRITICAL` |
| `age-secret-key` | Age File Encryption Key | `AGE-SECRET-KEY-1` | `CRITICAL` |
| `bip39-mnemonic` | Cryptocurrency Recovery Mnemonic | Algorithmic BIP-39 Dictionary Check | `CRITICAL` |

---

## 4. Token Entropy & False-Positive Tuning

The scanning engine evaluates high-randomness tokens (such as unstructured API credentials, Base64 strings, and hexadecimal values) using entropy calculation:

$$H(X) = -\sum_{i=1}^{n} P(x_i) \log_2 P(x_i)$$

### Threshold Recommendations
| Environment / Profile | Entropy Threshold | Tuning Characteristics |
| :--- | :--- | :--- |
| **Strict Compliance (Fintech / Defense)** | `3.8 - 4.2` | High sensitivity; flags short or low-randomness tokens; requires comprehensive allowlists. |
| **Standard Enterprise (Default)** | `4.5` | Recommended baseline; filters natural language while catching standard Base64/Hex credentials. |
| **Permissive / Legacy Codebases** | `5.0 - 5.5` | Minimal false positives; focuses exclusively on high-entropy cryptographic hashes and private keys. |

### False-Positive Filtering Modes
- **`strict`**: Context classifier is disabled. Any token exceeding the entropy threshold or matching a pattern is blocked immediately. Test directories are fully evaluated.
- **`balanced` (Default)**: Evaluates surrounding AST context (comments, variable names, dummy mock prefixes, UUIDs, and RFC examples). Standard test directories (`tests/`, `fixtures/`) are exempted.
- **`permissive`**: Employs aggressive contextual whitelisting. Suppresses findings located in documentation blocks, unit test fixtures, and mock files.

---

## 5. Detection Hardening & Governance

### 5.1 Custom Rule Validation & Automated Prefix Optimization
Custom detection rules require an anchor prefix to guarantee optimal performance during code evaluations.
- **Automated Prefix Extraction**: If the `prefix` field is omitted, Crenox Pro automatically extracts candidate prefixes from the provided pattern (including alternation groups such as `(zqxv|wkpl)[0-9]{6}`).
- **Integrity Validation**: Signatures with an empty prefix and non-derivable pattern shapes are rejected with `HTTP 400 Bad Request` to ensure every enabled rule actively enforces policy without performance penalties.

### 5.2 Anti-Evasion Normalization & Token Boundary Analysis
To defeat intentional obfuscation, bypass attempts, and encoding evasion techniques:
- **Unicode & BiDi De-obfuscation**: Strips zero-width characters (`\u200B`, `\u200C`, `\u200D`), byte-order marks (`\uFEFF`), word joiners (`\u2060`), and bidirectional override control characters (`\u200E`, `\u200F`, `\u202A`–`\u202E`, `\u2066`–`\u2069`, `\u061C`, `\u0085`) designed to visually conceal secrets.
- **Encoding Normalization**: Unescapes JSON unicode escapes (`\u00XX`) and decodes URL percent-encoded printable ASCII characters (`%XX`) prior to pattern evaluation.
- **Delimiter-Bounded Extraction**: Analyzes candidate credentials bounded by structural delimiters (quotes, backticks, parentheses, brackets, assignment operators `=`, `:`, `:=`, or query string separators).
- **Identifier Boundary Protection**: Enforces whole-word boundary rules on keyword signatures to eliminate false positives on benign code identifiers (e.g., distinguishing `tcf_exts_for_each_action` from Cloudflare tokens) while maintaining uncompromised detection for high-confidence anchored credentials (`ghp_`, `AKIA`, `sk_live_`).

### 5.3 Allowlist Security & Access Controls
To uphold Zero-Trust principles and prevent administrative exposure:
- **Redacted Display**: Allowlisted tokens are masked (`QAWL********efgh`) across administrative interfaces and default API responses.
- **Safe Policy Synchronization**: Updates preserve masked credentials without risk of overwriting or accidental clearance.
- **Step-Up Verification**: Plaintext inspection requires re-authentication via the administration console and is recorded in the immutable audit log.


---

[← Documentation Index](../README.md) · [Commercial EULA](../COMMERCIAL_EULA.md) · [Enterprise Specification & Pre-Order](https://crenoxhq.github.io/crenox/pro.html)
