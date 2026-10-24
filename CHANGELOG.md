# Changelog

All notable public updates to the **Crenox Pro** platform will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [0.0.1-beta] - 2026-10-24

### Added
- **Central Management Controller (`crenox-master`)**: REST API, administrative web console, strict hardware node-bound licensing enforcement with a 14-day hardware migration grace period, automatic database re-keying on license renewal, remote initial-setup token protection (`initial-setup.token`), and centralized policy distribution.
- **Workstation Security Agent (`crenox-agent` / `crenox`)**: Pre-commit, commit-msg, and merge hooks evaluating staged code buffers and filenames in memory prior to commit completion.
- **Detection Pipeline**: Pattern matching against built-in and custom signature rules, entropy evaluation, and contextual filtering.
- **Allowlist Controls**: Masking of sensitive tokens in management interfaces and audited step-up authorization for plaintext access.
- **Hook Integrity Verification**: Detection of configuration drift for Git hook settings (`core.hooksPath`) with automatic remediation.
- **Audit Logging**: Append-only, tamper-evident audit ledger with CSV and JSON export capabilities.
- **Enterprise Single Sign-On (SSO)**: OpenID Connect (OIDC) and reverse proxy authentication header support.
- **Exemption Lifecycle**: Supervised developer bypass requests with cryptographically signed authorizations.
- **Git History Redaction**: Historical commit scanning with worker pools, safety reference snapshots, and commit graph rebuilding.
- **Process Memory & Runtime Hardening**: Ephemeral credential ingestion, core dump suppression, and environment boundary enforcement.
- **Distribution & Updates**: Authenticated agent downloads via single-use tickets with target OS binary format verification, `X-Crenox-SHA256` headers, and dual-channel (`stable` and `beta`) signed OTA update manifests.

[0.0.1-beta]: https://github.com/crenoxhq/crenox-pro/blob/main/CHANGELOG.md
