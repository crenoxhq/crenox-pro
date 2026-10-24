# Crenox Pro — Enterprise Security Policy & Vulnerability Disclosure

This document details the security posture, vulnerability reporting protocol, coordinated disclosure timelines, and diagnostic triage procedures for Crenox Pro (`crenox-master` and `crenox-agent`).

---

## 1. Supported Software Versions

Security updates and vulnerability patches are actively delivered for supported enterprise releases:

| Release Channel | Supported | Status | Servicing Tier |
| :--- | :---: | :--- | :--- |
| `0.x` (Beta) | Yes | Active Beta Channel | Critical patches, security updates, signature catalog updates |
| Prior Releases | No | End of Life (EOL) | Unsupported |

---

## 2. Platform Security Architecture

Crenox Pro is designed for deployment within isolated, customer-controlled environments:

- **Local Execution Boundary**: Secret scanning, Git buffer evaluation, and policy enforcement execute entirely in memory on customer-controlled workstations and internal servers. Source code, credentials, and commit metadata do not leave the local perimeter.
- **Fail-Closed Pre-Commit Enforcement**: If an unhandled I/O error, corrupt buffer, or execution fault occurs, Git commits are halted to prevent uninspected code from passing.
- **Workstation Anti-Tampering & Self-Healing**: Workstations enforce active integrity defenses to prevent unauthorized modification or disabling of security policies and hooks. Policy configurations are cryptographically verified on every execution; any detected tampering is logged to the central console and automatically remediated to maintain continuous protection.
- **Zero-Trust Endpoint Isolation**: Workstation agents operate under the principle of least privilege and data minimization. Administrative cluster credentials are not retained on developer machines, minimizing endpoint attack surfaces.
- **Memory & Process Protection**: The runtime disables core dumps, cleans sensitive credential buffers from process memory, and enforces process execution boundaries.
- **Tamper-Evident Audit Ledger**: Security events, blocked commits, and authorized exemptions are recorded to an append-only, cryptographically validated audit ledger.

---

## 3. Reporting a Vulnerability

If you discover a security vulnerability or bypass mechanism in Crenox Pro, disclose it privately through designated channels. Do not create public GitHub issues or discussions for security reports.

### Primary Reporting Channels
1. **GitHub Security Advisories**:
   Submit an advisory directly via [Crenox Pro Security Advisories](https://github.com/crenoxhq/crenox-pro/security/advisories/new).
2. **Direct Security Email**:
   Transmit details to: `crenoxhq@proton.me`
   - Subject Format: `[SECURITY REPORT] <Component> - <Vulnerability Class>`

### Report Specifications
To ensure rapid triage and remediation, include:
1. **Component & Build**: Binary name (`crenox` or `crenox-master`) and version output (`crenox version` or `crenox-master version`).
2. **Vulnerability Classification**: (e.g., Pre-Commit Bypass, Privilege Escalation, Integrity Flaw, Denial of Service).
3. **Proof of Concept (PoC)**: Minimal steps or sample repository reproducing the behavior.
4. **Diagnostic Bundle**: Sanitized archive from `crenox-master bug-report` (if applicable).

---

## 4. Response & Remediation Service Level Targets (SLAs)

| Severity Level | Initial Triage & Acknowledgment | Remediation Hotfix Target |
| :--- | :---: | :---: |
| **Critical** (Pre-commit bypass, remote code execution, integrity forgery) | < 24 business hours | 72 hours |
| **High** (Authentication bypass, audit log tampering) | < 48 business hours | 7 calendar days |
| **Medium** (Denial of service, local permission bypass) | < 72 business hours | 14 calendar days |
| **Low** (Informational, verbose diagnostic output) | < 5 business days | Next release cycle |

---

## 5. Sanitized Diagnostic Telemetry

When submitting technical reports, utilize the built-in diagnostic facilities:

### Central Controller (`crenox-master`)
Generate a sanitized diagnostic bundle:
```bash
crenox-master bug-report \
  -db /var/lib/crenox/crenox-master.db \
  -license /etc/crenox/crenox.lic \
  -out /tmp/crenox-diagnostic-bundle.zip
```
*Note: The `bug-report` facility automatically sanitizes credentials, passwords, cryptographic keys, and internal network identifiers prior to archive creation.*

### Workstation Client (`crenox`)
Inspect local audit logs and verify hook integrity:
```bash
# Query the local audit ledger
crenox audit --limit 50 --json > /tmp/crenox-audit.json

# Validate active Git hook registration and state
crenox hook verify
```

---

## 6. Coordinated Disclosure Policy

CrenoxHQ adheres to coordinated vulnerability disclosure. We collaborate with security researchers and enterprise customers to validate, patch, and release updates prior to publishing public advisories.
