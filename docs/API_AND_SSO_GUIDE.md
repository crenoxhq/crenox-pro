# Crenox Pro — REST Management API and SSO Integration Guide

This guide provides technical specifications, JSON schemas, authentication mechanisms, and integration patterns for the Crenox Pro Master Controller API and Enterprise Single Sign-On (SSO) subsystem.

---

## 1. REST API Architecture & Standards

The `crenox-master` controller exposes an authenticated HTTP/HTTPS JSON REST API designed for integration with enterprise security orchestration platforms (SOAR), Identity Providers (IdP), and automated workstation provisioning scripts.

### 1.1 Global Technical Specifications
- **Base Endpoint**: All management endpoints are rooted under `/api/v1` (with unauthenticated health checks at `/healthz` and distribution endpoints at the root).
- **Transport Security**: TLS 1.2 or TLS 1.3 is enforced when TLS is active.
- **Request Limits**: Maximum request body size is capped at 10 MB for standard JSON endpoints, 64 MB for multipart binary package uploads (`/api/v1/ota/upgrade`), and 1 MB for workstation registration payloads (`http.MaxBytesReader`).
- **Header Protections**: Every HTTP response carries standard enterprise security headers:
  - `X-Content-Type-Options: nosniff`
  - `X-Frame-Options: DENY`
  - `X-XSS-Protection: 1; mode=block`
  - `Content-Security-Policy: default-src 'self'; ...`
  - `Referrer-Policy: strict-origin-when-cross-origin`
- **Rate Limiting**: Authentication endpoints (`/api/v1/auth/login`) enforce automated progressive lockouts per client IP and username to mitigate credential brute-force attacks.

### 1.2 Authentication Methods

The API recognizes three distinct authentication mechanisms:

1. **Admin Bearer Token / Session Cookie**
   - Supplied via the `Authorization: Bearer <session_token>` header or via the `crenox_session` HTTP-only cookie.
   - Required for administrative fleet management, policy mutation, license management, and audit export.
2. **Agent Cluster Key (`X-Agent-Key`)**
   - Configured during server deployment via `-agent-key` or `CRENOX_AGENT_KEY`.
   - Supplied via the `X-Agent-Key: <key>` header or query parameter `?agent_key=<key>`.
   - Used by workstation agents for initial fleet registration and cluster-level operations.
3. **Workstation Token (`X-Device-ID` + `X-Device-Token`)**
   - Issued to enrolled developer workstations upon successful completion of the enrollment sequence.
   - Supplied via headers:
     ```http
     X-Device-ID: dev_a1b2c3d4e5f6
     X-Device-Token: <64-character-hex-token>
     ```
   - Required for periodic heartbeats, audit event batch ingestion, policy synchronization, and JIT bypass requests.
4. **SSO Identity Delegation**
   - Evaluated when operating behind an enterprise reverse proxy (such as NGINX, HAProxy, or OAuth2-Proxy) terminating SAML 2.0 or OIDC.

---

## 2. API Endpoint Directory

The table below lists all endpoints exposed by `crenox-master`:

| Method | Route | Authorization | Description |
| :--- | :--- | :--- | :--- |
| **System & Health** | | | |
| `GET` | `/healthz`, `/health`, `/livez` | Public | Liveness probe returning service operational status |
| `GET` | `/api/v1/health` | Public | Alias for liveness verification |
| **Authentication & SSO** | | | |
| `GET` | `/api/v1/auth/status` | Public | Check whether the admin account is configured and SSO is enabled |
| `POST` | `/api/v1/auth/setup` | Public (Initial) | Configure the master administrator account on first boot |
| `POST` | `/api/v1/auth/login` | Public | Authenticate administrator credentials; returns a Bearer session token |
| `POST` | `/api/v1/auth/logout` | Session / Bearer | Invalidate the active administrative session |
| `GET` | `/api/v1/auth/sso/config` | Admin Bearer | Retrieve active enterprise SSO configuration (OIDC/Header) |
| `PUT` | `/api/v1/auth/sso/config` | Admin Bearer | Update enterprise SSO parameters |
| `GET` | `/api/v1/auth/sso/login` | Public | Initiate OIDC authorization code redirect flow |
| `GET` | `/api/v1/auth/sso/callback` | Public | Handle OIDC identity provider authorization callback |
| `POST` | `/api/v1/auth/sso/test` | Admin Bearer | Test reachability and metadata validation for configured IdP |
| **Fleet Management & Enrollment** | | | |
| `GET` | `/api/v1/fleet/stats` | Admin Bearer | Retrieve fleet capacity, active seat count, and license quotas |
| `GET` | `/api/v1/fleet/devices` | Admin Bearer | List enrolled developer workstations with filtering |
| `POST` | `/api/v1/fleet/register` | Enrollment Token | Enroll a new workstation agent and bind hardware fingerprint |
| `POST` | `/api/v1/fleet/heartbeat` | Device Token | Ingest agent heartbeat and return active approved bypasses |
| `POST` | `/api/v1/fleet/devices/{id}/revoke` | Admin Bearer | Revoke workstation authorization |
| `POST` | `/api/v1/fleet/devices/{id}/reactivate` | Admin Bearer | Reactivate a previously revoked workstation |
| `POST` | `/api/v1/fleet/devices/{id}/seat` | Admin Bearer | Update workstation seat role, department, or administrative notes |
| `GET` | `/api/v1/fleet/devices/{id}/events` | Admin Bearer | Retrieve incident events and bypass history for a specific workstation |
| `DELETE`, `POST` | `/api/v1/fleet/devices/{id}` | Admin Bearer | Permanently delete a workstation record (also accepts `/delete` suffix) |
| `GET` | `/api/v1/fleet/tokens` | Admin Bearer | List pre-provisioned enrollment tokens |
| `POST` | `/api/v1/fleet/tokens` | Admin Bearer | Generate single-use or multi-use workstation enrollment token |
| `DELETE` | `/api/v1/fleet/tokens/{id}` | Admin Bearer | Revoke and delete an active enrollment token |
| `GET` | `/api/v1/fleet/analytics` | Admin Bearer | Aggregate security metrics across departments and branches |
| `GET`, `POST` | `/api/v1/fleet/policy` | Device Token | Return cryptographically signed enterprise policy envelope |
| `GET` | `/api/v1/fleet/stream` | Device Token / AgentKey | Persistent Server-Sent Events (SSE) stream for real-time fleet push synchronization |
| **Audit Ledger & Compliance** | | | |
| `GET` | `/api/v1/audit/events` | Admin Bearer | List paginated security violation audit records |
| `POST` | `/api/v1/audit/events` | Device Token | Ingest batch security incidents and watchdog alerts from workstations |
| `GET` | `/api/v1/audit/export` | Admin Bearer | Export immutable audit ledger in CSV format |
| **Just-In-Time (JIT) Exemption Workflow** | | | |
| `GET` | `/api/v1/bypass/requests` | Admin Bearer | List pending and historical developer bypass requests |
| `POST` | `/api/v1/bypass/request` | Device Token | Submit a new developer exemption request |
| `GET` | `/api/v1/bypass/active` | Device Token | Fetch active approved bypasses for current workstation |
| `POST` | `/api/v1/bypass/consume` | Device Token | Mark a single-commit bypass as consumed upon commit |
| `POST` | `/api/v1/bypass/approve` | Admin Bearer | Approve bypass and generate signed cryptographic token |
| `POST` | `/api/v1/bypass/reject` | Admin Bearer | Reject a pending developer bypass request |
| **Policy Governance & Engine Tuning** | | | |
| `GET` | `/api/v1/config` | Admin Bearer | Retrieve active central enterprise security configuration (masked secrets) |
| `PUT` | `/api/v1/config` | Admin Bearer | Update enterprise configuration and sign updated policy |
| `POST` | `/api/v1/config/test` | Admin Bearer | Test custom regex and evaluate token entropy in sandbox |
| `GET` | `/api/v1/config/catalog` | Admin Bearer | List full catalog of built-in and Pro secret detection rules |
| `GET` | `/api/v1/config/export` | Admin Bearer | Export central ruleset as annotated YAML (`crenox-pro.yaml`) |
| `POST` | `/api/v1/config/import` | Admin Bearer | Upload and validate external YAML ruleset |
| `POST`, `GET` | `/api/v1/config/allowlist/reveal` | Admin Bearer | Reveal plaintext allowlist secret with immutable audit logging |
| **Executive Reporting & Diagnostics** | | | |
| `GET` | `/api/v1/report/weekly` | Admin Bearer | Generate executive HTML compliance report |
| `GET` | `/api/v1/system/logs` | Admin Bearer | Retrieve system execution and security log events |
| `GET` | `/api/v1/diag/bundle` | Admin Bearer | Download redacted ZIP diagnostic bundle with integrity manifest |
| **Historical Scanning & Purge** | | | |
| `POST` | `/api/v1/scan/history` | Admin Bearer | Execute server-side historical Git repository inspection |
| `GET` | `/api/v1/scan/suggest-paths` | Admin Bearer | Auto-complete candidate repository paths with source metadata |
| `POST` | `/api/v1/purge/run` | Admin Bearer | Execute Git history token redaction |
| `POST` | `/api/v1/purge/restore` | Admin Bearer | Restore repository from pre-purge safety snapshot |
| `GET` | `/api/v1/purge/backups` | Admin Bearer | List available pre-purge repository backups |
| **Commercial License** | | | |
| `GET` | `/api/v1/license` | Admin Bearer | Inspect active commercial license parameters and features |
| `PUT`, `POST` | `/api/v1/license` | Admin Bearer | Upload and activate new license key (`.lic`) |
| `POST` | `/api/v1/license/renew` | Admin Bearer | Alias to upload and activate new license key |
| **Automated Distribution & OTA** | | | |
| `GET` | `/install.sh` | Public | Shell installer script for Linux and macOS |
| `GET` | `/install.ps1` | Public | PowerShell installer script for Windows |
| `POST`, `GET` | `/api/v1/download/ticket` | Token / Admin / AgentKey | Generate single-use temporary download ticket for agent binary |
| `GET` | `/api/v1/download/agent` | Ticket / Token / Admin / AgentKey | Download statically compiled agent binary (anonymous rejected with 401) |
| `GET` | `/api/v1/ota/check` | Admin Bearer | Check upstream repository for controller software updates (`?channel=stable\|beta`) |
| `POST` | `/api/v1/ota/upgrade` | Admin Bearer | Apply controller update via atomic binary swap |
| `GET` | `/api/v1/ota/agent/check` | Device Token | Check whether an updated agent binary is available (`?channel=stable\|beta`) |
| `GET` | `/api/v1/ota/agent/download`| Device Token | Download updated agent binary (returns 404 if no binary is hosted in distribution path) |

---

## 3. Detailed Request & Response Schemas

### 3.0 First-Time Admin Setup: `POST /api/v1/auth/setup`

Configures the primary administrator account during initial startup when the controller is started without `-admin-pass`. Remote requests (non-loopback) must include `setup_token` from `initial-setup.token` in the database directory.

**Request:**
```bash
curl -X POST https://crenox-master:8443/api/v1/auth/setup \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "<admin-secure-password>",
    "setup_token": "<token-from-initial-setup.token>"
  }'
```

**Success Response (`200 OK`):**
```json
{
  "expires_in": 86400,
  "status": "ok",
  "token": "<session-token>",
  "username": "admin"
}
```

**Error Responses (`403 Forbidden`):**
```json
{
  "code": "SETUP_TOKEN_REQUIRED",
  "error": "valid setup_token is required for remote first-run initialization (see initial-setup.token in the database directory)"
}
```
```json
{
  "code": "ALREADY_CONFIGURED",
  "error": "administrator account is already configured"
}
```

---

### 3.1 Authentication: `POST /api/v1/auth/login`

Authenticates an administrative user and issues an active session token.

**Request:**
```bash
curl -X POST https://crenox-master:8443/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "<admin-secure-password>"
  }'
```

**Success Response (`200 OK`):**
```json
{
  "status": "ok",
  "token": "<session-token>",
  "expires_in": 86400
}
```

**Error Response (`401 Unauthorized`):**
```json
{
  "error": "invalid admin credentials"
}
```

---

### 3.2 Provisioning: `POST /api/v1/fleet/tokens`

Creates an enrollment token to authorize one or more workstation installations.

**Request:**
```bash
curl -X POST https://crenox-master:8443/api/v1/fleet/tokens \
  -H "Authorization: Bearer <admin-session-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "role": "Developer",
    "department": "Core Banking Engine",
    "notes": "Provisioned for Q4 Engineering cohort",
    "duration_days": 14,
    "max_uses": 25
  }'
```

**Success Response (`200 OK`):**
```json
{
  "status": "ok",
  "token": {
    "id": "tok_9918273645",
    "token": "<enrollment-token>",
    "role": "Developer",
    "department": "Core Banking Engine",
    "notes": "Provisioned for Q4 Engineering cohort",
    "created_at": "2026-09-30T12:00:00Z",
    "expires_at": "2026-10-14T12:00:00Z",
    "used": false,
    "max_uses": 25,
    "uses_count": 0,
    "remaining_uses": 25
  },
  "agent_key": "<enterprise-cluster-key>",
  "is_self_signed": false,
  "master_url": "https://crenox-master:8443",
  "detected_ips": ["10.0.4.15"],
  "install_command": "curl -sSL https://crenox-master:8443/install.sh | bash -s -- --token <enrollment-token>",
  "install_command_unix": "curl -sSL https://crenox-master:8443/install.sh | bash -s -- --token <enrollment-token>",
  "install_command_win": "& ([scriptblock]::Create((iwr -useb https://crenox-master:8443/install.ps1).Content)) -Token <enrollment-token>",
  "install_command_direct": "crenox enroll --master https://crenox-master:8443 --token <enrollment-token>",
  "config_yaml": "master:\n  url: \"https://crenox-master:8443\"\n  enrollment_token: \"<enrollment-token>\"\nseat:\n  role: \"Developer\"\n  department: \"Core Banking Engine\"\n"
}
```

---

### 3.3 Workstation Registration: `POST /api/v1/fleet/register`

Executes automated workstation enrollment, binding the physical hardware fingerprint to the seat.

**Request Headers:**
- `X-Agent-Key: <cluster-agent-key>` (if `-agent-key` is configured on the master)
- `X-Hardware-Fingerprint: CNX-4A9F-B3C1-90E2-55D4`

**Request Body:**
```json
{
  "enrollment_token": "<enrollment-token>",
  "id": "dev_77a8b9c0d1",
  "hostname": "workstation-dev-01",
  "username": "jdoe",
  "os": "linux/amd64",
  "ip": "10.0.4.15",
  "agent_version": "1.0.0",
  "hardware_fingerprint": "CNX-4A9F-B3C1-90E2-55D4"
}
```

**Success Response (`200 OK`):**
```json
{
  "status": "ok",
  "device": {
    "id": "dev_77a8b9c0d1",
    "hostname": "workstation-dev-01",
    "username": "jdoe",
    "os": "linux/amd64",
    "ip": "10.0.4.15",
    "agent_version": "1.0.0",
    "registered_at": "2026-09-30T12:05:00Z",
    "last_seen_at": "2026-09-30T12:05:00Z",
    "status": "active",
    "seat_role": "Developer",
    "department": "Core Banking Engine"
  },
  "device_token": "3f90a1b2c3d4e5f60718293a4b5c6d7e8f90123456789abcdef0123456789abc",
  "master_public_key": "f83f8e2b16d5f5103bf5908760a794f2f4aaed15187099e4ba0868d867902f1b",
  "enroll_timestamp": 1790779500,
  "signature": "a1b2c3d4...<64-byte-ed25519-hex-signature>",
  "licensed_features": ["purge", "historical-scan", "custom-rules", "dashboard", "api", "reports", "sso", "audit-log"],
  "license_customer": "Acme Global Bank",
  "license_expires_at": "2027-09-30T00:00:00Z",
  "entitlements_sig": "e5f60718...<64-byte-ed25519-hex-signature>",
  "tls_cert_pem": "-----BEGIN CERTIFICATE-----\n...",
  "is_self_signed": false
}
```

**Seat Capacity Exceeded Response (`402 Payment Required`):**
```json
{
  "code": "SEAT_LIMIT_EXCEEDED",
  "error": "fleet: license seat limit exceeded; revoke unused workstations or upgrade license"
}
```

---

### 3.4 Workstation Heartbeat: `POST /api/v1/fleet/heartbeat`

Periodic heartbeat dispatched by `crenox daemon` or triggered during Git pre-commit. Returns active, approved bypass authorizations.

**Request:**
```bash
curl -X POST https://crenox-master:8443/api/v1/fleet/heartbeat \
  -H "X-Device-ID: dev_77a8b9c0d1" \
  -H "X-Device-Token: 3f90a1b2c3d4e5f60718293a4b5c6d7e8f90123456789abcdef0123456789abc" \
  -H "X-Hardware-Fingerprint: CNX-4A9F-B3C1-90E2-55D4" \
  -H "Content-Type: application/json" \
  -d '{
    "device_id": "dev_77a8b9c0d1",
    "ip": "10.0.4.15",
    "agent_version": "1.0.0",
    "username": "jdoe",
    "hostname": "workstation-dev-01",
    "applied_policy_hash": "ef9bc6bac921fc23722233cccf743d64b10d6e8dcabb860b40e00712c6331482",
    "applied_policy_signed_at": 1792609975
  }'
```

**Success Response (`200 OK`):**
```json
{
  "status": "ok",
  "approved_bypasses": [
    {
      "id": "byp_5521",
      "file_path": "services/auth/mock_test.go",
      "secret_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
      "reason": "Test fixture dummy token for unit test runner",
      "status": "approved",
      "approved_by": "secops-admin@enterprise.internal",
      "scope": "single_commit",
      "max_uses": 1,
      "used_count": 0,
      "expires_at": "2026-10-07T12:00:00Z",
      "signature": "3a4b5c6d...<64-byte-ed25519-hex-signature>"
    }
  ],
  "master_public_key": "f83f8e2b16d5f5103bf5908760a794f2f4aaed15187099e4ba0868d867902f1b",
  "seat_role": "Developer",
  "department": "Core Banking Engine"
}
```

---

### 3.5 JIT Bypass Approval: `POST /api/v1/bypass/approve`

Administrative authorization of an exemption request. Signs the request using the master authority key.

**Request:**
```bash
curl -X POST https://crenox-master:8443/api/v1/bypass/approve \
  -H "Authorization: Bearer <admin-session-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "byp_5521",
    "duration_days": 7,
    "approver": "SecOps On-Call Lead",
    "scope": "single_commit"
  }'
```

**Success Response (`200 OK`):**
```json
{
  "status": "ok",
  "message": "bypass approved and cryptographically signed",
  "bypass": {
    "id": "byp_5521",
    "status": "approved",
    "approved_by": "SecOps On-Call Lead",
    "scope": "single_commit",
    "max_uses": 1,
    "signature": "a8f102...<128-hex-chars>"
  }
}
```

---

### 3.6 Audit Event Ingestion: `POST /api/v1/audit/events`

Dispatches a batch of security violations or watchdog detection events from a workstation to the master ledger.

**Request:**
```bash
curl -X POST https://crenox-master:8443/api/v1/audit/events \
  -H "X-Device-ID: dev_77a8b9c0d1" \
  -H "X-Device-Token: 3f90a1b2c3d4e5f60718293a4b5c6d7e8f90123456789abcdef0123456789abc" \
  -H "Content-Type: application/json" \
  -d '{
    "events": [
      {
        "id": "evt_109283",
        "timestamp": "2026-09-30T12:15:30Z",
        "event_type": "SCAN_STAGED_BLOCKED",
        "device_id": "dev_77a8b9c0d1",
        "developer": "jdoe",
        "repository": "payment-gateway",
        "branch": "feature/direct-debit",
        "findings_count": 1,
        "details": "Stripe Live Secret Key detected in services/payment/client.go:14",
        "code_snippet": "const StripeKey = \"<sk_live_redacted>\"",
        "line_number": 14,
        "entropy": 4.82,
        "rule_id": "stripe-live-secret",
        "severity": "CRITICAL",
        "detection_tier": "trie",
        "sequence": 104,
        "prev_hash": "a1b2c3d4...",
        "record_hash": "e5f6a7b8..."
      }
    ]
  }'
```

**Success Response (`200 OK`):**
```json
{
  "status": "ok",
  "ingested": 1
}
```

---

### 3.7 Authenticated Agent Distribution & Ephemeral Tickets

To eliminate unauthorized extraction of proprietary binaries, `/api/v1/download/agent` enforces strict authentication. Anonymous requests are rejected with `HTTP 401 Unauthorized`. Workstations download binaries using either an enrollment token, cluster agent key, admin session, or a single-use ephemeral download ticket.

#### A. Issue Download Ticket: `POST /api/v1/download/ticket`
Generates a short-lived (5-minute TTL), single-use cryptographic ticket for non-interactive installer scripts (`install.sh`, `install.ps1`).

**Request (Using Enrollment Token):**
```bash
curl -X POST https://crenox-master:8443/api/v1/download/ticket \
  -H "Authorization: Bearer <enrollment-token>"
```
*(Also supports `?token=crx_tok_...` query parameter or `X-Agent-Key` header).*

**Success Response (`200 OK`):**
```json
{
  "status": "ok",
  "ticket": "crx_dl_8f90a1b2c3d4e5f67890abcdef123456",
  "expires_in": 300,
  "expires_in_seconds": 300,
  "expires_at": "2026-10-01T04:45:00Z"
}
```

#### B. Download Agent Binary: `GET /api/v1/download/agent`
Streams the statically compiled client executable. Requires one of:
1. `ticket` query parameter (single-use, burned immediately upon redemption).
2. `token` query parameter or `X-Enroll-Token` header with valid enrollment token.
3. `X-Agent-Key` header or `agent_key` query parameter with cluster secret key.
4. Active administrative session cookie (`crenox_session`) or Bearer token.
5. `X-Device-Token` (+ optional `X-Device-ID`) header from an enrolled device.

**Request:**
```bash
curl -sSL "https://crenox-master:8443/api/v1/download/agent?ticket=crx_dl_8f90a1b2c3d4e5f67890abcdef123456&os=linux&arch=amd64" \
  -o /usr/local/bin/crenox
```
*(Every downloaded binary is verified by the server to match the target OS executable header (`\x7fELF`, Mach-O, or `MZ`) before streaming and includes an `X-Crenox-SHA256` response header).*

**Query Parameters:**
| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `os` | string | Optional | Target operating system: `linux`, `darwin`, `windows`, `android`. Directory traversal characters (`/`, `\`, `..`) are rejected immediately. |
| `arch` | string | Optional | Target architecture: `amd64`, `arm64`, `386`, `arm` (aliases: `x86_64`, `aarch64`, `armv7l`, etc.). |
| `ticket` | string | Optional | Ephemeral download ticket issued by `/api/v1/download/ticket`. |
| `token` | string | Optional | Fleet enrollment token. |

**Unauthorized Response (`401 Unauthorized`):**
```json
{
  "code": "UNAUTHORIZED",
  "error": "unauthorized: valid download ticket, enrollment token, or authentication required"
}
```

**Invalid Parameter / Traversal Response (`400 Bad Request`):**
```json
{
  "error": "invalid path characters in os or arch parameter",
  "code": "INVALID_PARAMETERS"
}
```

---


### 3.8 Policy Governance: Secret Masking & Step-Up Allowlist Reveal

To satisfy zero-trust compliance (SOC2 / ISO 27001) and prevent administrative credential harvesting, allowlist secrets are never returned in plaintext by standard read APIs.

#### A. Masked Configuration Retrieval: `GET /api/v1/config`
Tokens stored in the central allowlist are masked with 8 fixed asterisks preserving only the first 4 and last 4 characters.

**Request:**
```bash
curl -H "Authorization: Bearer $ADMIN_TOKEN" https://crenox-master:8443/api/v1/config
```

**Response Snippet (`200 OK`):**
```json
{
  "allowlist": {
    "tokens": [
      "QAWL********efgh"
    ],
    "paths": ["test/fixtures/"],
    "patterns": ["test_token_approved_prefix"]
  }
}
```

#### B. Safe Policy Updates: `PUT /api/v1/config`
When updating configuration via API or console, masked tokens (`********`) are automatically preserved as-is. This prevents inadvertent modification or clearing of active allowlist entries.

#### C. Audited Secret Reveal: `POST /api/v1/config/allowlist/reveal` or `GET /api/v1/config/allowlist/reveal`
Administrative users requiring full plaintext visibility (such as key rotation audits) can reveal masked allowlist secrets by zero-based index or token query. Every reveal operation is permanently recorded in the immutable audit ledger.

**Request (JSON Body):**
```bash
curl -X POST https://crenox-master:8443/api/v1/config/allowlist/reveal \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "index": 0
  }'
```

**Alternative Request (Query Parameter):**
```bash
curl -H "Authorization: Bearer $ADMIN_TOKEN" \
  "https://crenox-master:8443/api/v1/config/allowlist/reveal?index=0"
```

**Success Response (`200 OK`):**
```json
{
  "status": "ok",
  "index": 0,
  "token": "<revealed-plaintext-token>"
}
```

#### D. Plaintext Policy Export: `GET /api/v1/config/export`

Exports the active policy configuration in YAML format.

| Parameter | Type | In | Description |
| :--- | :--- | :--- | :--- |
| `unmasked` | boolean | query | When set to `true`, allowlist tokens are exported in plaintext. Default is `false` (tokens are masked). |

**Request:**
```bash
curl -H "Authorization: Bearer $ADMIN_TOKEN" \
  "https://crenox-master:8443/api/v1/config/export?unmasked=true" \
  -o crenox-pro-backup.yaml
```

#### E. Configuration Validation Rules: `PUT /api/v1/config`

The configuration API enforces the following validation constraints:
- **Allowlist Tokens & Patterns**: Must not contain wildcard characters (`*`, `?`, `[`, `]`, `{`, `}`). Violations return HTTP 400 (`INVALID_ALLOWLIST_TOKEN` or `INVALID_ALLOWLIST_PATTERN`).
- **Custom Signatures**: Regular expressions must be syntactically valid. Invalid expressions return HTTP 400 (`INVALID_SIGNATURE_REGEX`).
- **Fleet Synchronization**: Successfully updating policy configuration broadcasts an update notification across active agent SSE connections (`/api/v1/fleet/stream`).

---

### 3.9 Filesystem Path Auto-Completion: `GET /api/v1/scan/suggest-paths`

Provides verified candidate repository paths on the controller host for server-side historical scans. Every suggested path includes clear origin, scope, and description metadata.

**Request:**
```bash
curl -H "Authorization: Bearer $ADMIN_TOKEN" https://crenox-master:8443/api/v1/scan/suggest-paths
```

**Success Response (`200 OK`):**
```json
{
  "suggestions": [
    {
      "path": "/data/repositories/core-banking",
      "source": "git_repository",
      "scope": "workspace",
      "description": "Active Git repository detected in controller workspace"
    },
    {
      "path": "/var/repos/fintech-api",
      "source": "configured_directory",
      "scope": "system",
      "description": "Explicitly configured enterprise repository directory"
    }
  ]
}
```

---

### 3.10 Sandbox Signature Testing: `POST /api/v1/config/test`

Tests candidate detection rules against sample input to verify matching behavior and entropy calculation.

> [!NOTE]
> Testing custom signatures requires the `custom_rules` entitlement in the active commercial license (`crenox.lic`).

**Request:**
```bash
curl -X POST https://crenox-master:8443/api/v1/config/test \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "prefix": "wkpl",
    "regex": "(zqxv|wkpl)[0-9]{6}",
    "sample": "wkpl654321"
  }'
```

**Success Response (`200 OK`):**
```json
{
  "matched": true,
  "token": "<matched-sample-token>",
  "entropy": 2.58,
  "execution_time": "14.2µs",
  "execution_nanos": 14200
}
```

---

### 3.11 Commercial License Posture: `GET /api/v1/license`

Inspects active commercial license parameters, authorized seat quotas, licensed feature flags, and verifies that the embedded `host_fingerprint` strictly matches the controller's runtime `current_host_fingerprint`.

**Request:**
```bash
curl -H "Authorization: Bearer $ADMIN_TOKEN" https://crenox-master:8443/api/v1/license
```

**Success Response (`200 OK`):**
```json
{
  "id": "lic_99a8b7c6d5",
  "customer": "Acme Global Bank",
  "contract_id": "CNT-2026-001",
  "tier": "Enterprise Fleet",
  "is_trial": false,
  "seats": 200,
  "issued_at": "2026-10-01T00:00:00Z",
  "expires_at": "2027-10-01T00:00:00Z",
  "features": [
    "dashboard",
    "custom-rules",
    "audit-log",
    "historical-scan",
    "reports",
    "api",
    "purge",
    "sso"
  ],
  "host_fingerprint": "CNX-4A9F-B3C1-90E2-55D4",
  "current_host_fingerprint": "CNX-4A9F-B3C1-90E2-55D4",
  "node_binding_status": "verified",
  "migration_grace_days_remaining": 0,
  "migration_warning": "",
  "status": "verified",
  "expired": false,
  "detected_ips": ["10.0.1.15"]
}
```

> [!NOTE]
> **Single-Master Enforcement & 14-Day Hardware Migration Grace**: When `host_fingerprint` matches `current_host_fingerprint`, `status` and `node_binding_status` report `"verified"`. If a server is migrated to new hardware (`host_fingerprint` differs from `current_host_fingerprint`), `status` reports `"migration_grace"` with `migration_grace_days_remaining` counting down from `14` so the administrator can request a free replacement license without downtime. After 14 days without a matching license, `status` transitions to `"node_mismatch_locked"` (pausing new enrollments and administrative mutations while keeping existing enrolled workstations' pre-commit scanning active).

---

### 3.12 License Renewal & Activation: `POST /api/v1/license/renew`

Uploads and activates a new commercial license file (`.lic`), persists `.active_license.lic` in the database directory, and automatically re-keys the encrypted master database (`crenox-master.db`) to the new license without downtime or data loss.

**Request:**
```bash
curl -X POST https://crenox-master:8443/api/v1/license/renew \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "license_key": "--- BEGIN CRENOX PRO LICENSE ---\n...\n--- END CRENOX PRO LICENSE ---"
  }'
```

**Success Response (`200 OK`):**
```json
{
  "status": "ok",
  "message": "License verified and activated successfully",
  "id": "ca8e8b7f-fa0f-4989-91aa-9ca026722db4",
  "customer": "Acme Global Bank",
  "seats": 200,
  "tier": "Enterprise Fleet",
  "is_trial": false,
  "expires_at": "2027-10-01T23:59:59Z"
}
```

---

### 3.13 Historical Repository Scan: `POST /api/v1/scan/history`

Executes a historical commit scan on a repository path on the master controller.

**Request:**
```bash
curl -X POST https://crenox-master:8443/api/v1/scan/history \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "repo_path": "/var/repos/payment-service",
    "branch": "main",
    "max_commits": 200
  }'
```

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `repo_path` | string | Optional | Filesystem path to target Git repository (aliases: `path`, `repository`, `repo`). Default: `.` |
| `branch` | string | Optional | Specific branch reference to inspect. |
| `max_commits` | integer | Optional | Maximum number of commits to scan. Default: `200` |

**Success Response (`200 OK`):**
```json
{
  "total_commits": 200,
  "total_blobs_seen": 450,
  "unique_blobs_scanned": 310,
  "blobs_deduplicated": 140,
  "duration": 48291040,
  "findings": [],
  "scan_errors": null,
  "warnings": [
    "git history: skipped large file or binary diff: assets/model.bin"
  ]
}
```

> [!NOTE]
> `warnings` lists files skipped by Git as binary or exceeding the 1GB threshold (`core.bigFileThreshold=1g`).

---

### 3.14 History Token Purge: `POST /api/v1/purge/run`

Rewrites Git history on a repository path on the master controller to replace target secret tokens.

> [!NOTE]
> History purging requires the `purge` entitlement in your commercial license (`crenox.lic`).

**Request:**
```bash
curl -X POST https://crenox-master:8443/api/v1/purge/run \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "repo_path": "/var/repos/payment-service",
    "target_tokens": ["<leaked-secret-token>"],
    "replacement": "[REDACTED_BY_CRENOX]",
    "auto_backup": true,
    "dry_run": false
  }'
```

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `repo_path` | string | Optional | Target Git repository path (aliases: `path`, `repository`, `repo`). Default: `.` |
| `target_tokens` | string array | Required | List of secret token strings to redact (alias: `tokens`). |
| `replacement` | string | Optional | Replacement string. Default: `[REDACTED_BY_CRENOX]` |
| `auto_backup` | boolean | Optional | Create safety snapshot reference under `refs/original/`. Default: `true` |
| `force` | boolean | Optional | Bypass dirty working tree check. Default: `false` |
| `dry_run` | boolean | Optional | Preview changes without modifying repository references. Default: `false` |

**Success Response (`200 OK`):**
```json
{
  "success": true,
  "backup_prefix": "refs/original/crenox-backup-20261005-142210-60a3929a",
  "total_commits_inspected": 200,
  "commits_rewritten": 1,
  "blobs_redacted": 1,
  "branches_updated": ["main"],
  "duration": 95123040,
  "dry_run": false
}
```

**Error Response (`409 Conflict`):**
```json
{
  "error": "working tree has uncommitted or untracked changes: commit, stash, or pass force=true to override",
  "code": "DIRTY_WORKING_TREE"
}
```

> [!NOTE]
> Returns `409 Conflict` (`DIRTY_WORKING_TREE`) if the repository has uncommitted or untracked changes. Set `"force": true` to override. `--dry-run` is permitted on dirty repositories.

---

### 3.15 Audit Ledger Export: `GET /api/v1/audit/export`

Exports stored audit events in CSV format or JSON structure, with optional filtering by event type.

**Query Parameters:**
| Parameter | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `format` | string | `json` | Set to `csv` to receive RFC 4180 CSV attachment; otherwise returns JSON |
| `event_type` | string | (all) | Filter records by security event type (e.g. `SECRET_BLOCKED`, `TAMPER_DETECTED`, `NO_VERIFY_VIOLATION`, `POLICY_SYNC`) |

**Request:**
```bash
# Export all events as CSV
curl -H "Authorization: Bearer $ADMIN_TOKEN" \
  "https://crenox-master:8443/api/v1/audit/export?format=csv" \
  -o crenox-audit-export.csv

# Export only policy tampering security incidents
curl -H "Authorization: Bearer $ADMIN_TOKEN" \
  "https://crenox-master:8443/api/v1/audit/export?format=csv&event_type=TAMPER_DETECTED" \
  -o crenox-tamper-events.csv
```

**Response Headers (CSV):**
```http
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="crenox-audit-log.csv"
```

---

### 3.16 Fleet Event Push Stream: `GET /api/v1/fleet/stream`

Provides a persistent Server-Sent Events (SSE) push stream delivering policy updates, exemption authorizations, and fleet administrative signals to active workstations, replacing periodic polling with event-driven delivery.

**Connection Protocol & Authentication:**
- **Protocol**: HTTP/1.1 or HTTP/2 Server-Sent Events (`text/event-stream`).
- **Headers**:
  - `Accept: text/event-stream`
  - `X-Device-ID: dev_77a8b9c0d1`
  - `X-Device-Token: <64-character-hex-token>`
  - `X-Hardware-Fingerprint: CNX-4A9F-B3C1-90E2-55D4`
  - `Last-Event-ID: <sequence>` *(Optional; supplied upon reconnection for monotonic catch-up)*

**Monotonic Sequence Catch-Up:**
- Every fleet event carries a globally increasing 64-bit monotonic sequence number (`id: <sequence>`).
- When an agent reconnects after network interruption, it supplies its last processed sequence via `Last-Event-ID` header or `?since_sequence=<sequence>` query parameter.
- The controller replays missed backlog events in sequence order. If the workstation was offline longer than the backlog journal retention window, the controller dispatches a full authoritative snapshot (`SYNC_ALL`).

**Stream Event Types:**
- `SYNC_ALL`: Full state envelope dispatched on initial connection or journal overflow, containing complete signed policy and active exemptions.
- `POLICY_UPDATED`: Push notification triggered upon administrative policy updates in the Master console.
- `BYPASS_UPDATED` — Push notification containing cryptographically signed exemption details dispatched to the target workstation upon approval.
- `BYPASS_REVOKED` — Revocation signal purging an exemption from local workstation caches.
- `DEVICE_REVOKED`: Revocation signal commanding the workstation to purge cached exemptions and terminate the session.
- `PING`: Periodic keepalive frame maintaining persistent network connections.

**Sample Stream Frame:**
```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

id: 1042
event: POLICY_UPDATED
data: {"policy":{"disabled_rules":["test-mock"],"scan_engine":{"entropy_threshold":4.5}},"signed_at":1773400000,"signature":"..."}

: keepalive

```

---

### 3.17 OTA Software Updates & Dual-Channel Selection: `GET /api/v1/ota/check` & `POST /api/v1/ota/upgrade`

Checks for signed software updates across two supported release channels (`stable` and `beta`) and supports air-gapped binary package deployment.

#### A. Check Update Channel: `GET /api/v1/ota/check`
When `?channel=stable` or `?channel=beta` is supplied, the preferred channel is persisted in `crenox-master.db`.

**Request:**
```bash
curl -H "Authorization: Bearer $ADMIN_TOKEN" \
  "https://crenox-master:8443/api/v1/ota/check?channel=beta"
```

**Success Response (`200 OK`):**
```json
{
  "status": "ok",
  "current_version": "v0.0.1-beta",
  "latest_version": "0.0.1-beta",
  "up_to_date": true,
  "channel": "beta",
  "update_channel": "beta",
  "available_channels": ["stable", "beta"],
  "platform": "linux/amd64",
  "last_checked": "2026-10-24T15:30:00Z",
  "verifier_active": true,
  "root_public_key": "f83f8e2b16d5f5103bf5908760a794f2f4aaed15187099e4ba0868d867902f1b",
  "repository_url": "https://github.com/crenoxhq/crenox-pro.git",
  "releases_url": "https://github.com/crenoxhq/crenox-pro/releases"
}
```

#### B. Air-Gapped Binary Package Upload: `POST /api/v1/ota/upgrade`
Uploads a sealed release binary (`crenox-master` or `crenox-agent`). The server verifies the executable format (`\x7fELF`, Mach-O, or `MZ`), validates either the embedded 119-byte `CRENOX_SEAL_V1` Ed25519 integrity trailer or a detached `signature` field against `CrenoxHQReleasePublicKey`, and performs an atomic binary hot-swap.

**Request:**
```bash
curl -X POST https://crenox-master:8443/api/v1/ota/upgrade \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -F "binary=@crenox-master-linux-amd64" \
  -F "component=crenox-master"
```

---

## 4. Enterprise Single Sign-On (SSO) Integration

The Single Sign-On module provides federated identity verification for the central web console and administrative REST API.

> [!NOTE]
> SSO capabilities require the `sso` feature flag in your commercial license (`crenox.lic`). Systems without this entitlement respond with `HTTP 403 Forbidden` (`{"error": "feature_locked", "feature": "sso", ...}`).

The system supports two distinct integration architectures:
1. **Direct OpenID Connect (OIDC)**: Native OAuth 2.0 / OIDC authorization code exchange with enterprise IdPs (Okta, Keycloak, Ping Identity, Microsoft Entra ID).
2. **Reverse Proxy Header Gateway (SAML 2.0 / Perimeter)**: Identity assertion parsed from upstream reverse proxies or enterprise ingress controllers.

---

### 4.1 OIDC Integration Architecture

```
┌──────────────┐          ┌───────────────────┐          ┌────────────────┐
│   Browser    │          │   Crenox Master   │          │  Enterprise    │
│  (Admin UI)  │          │   (:8443)         │          │  Identity IdP  │
└──────┬───────┘          └─────────┬─────────┘          └───────┬────────┘
       │                            │                            │
       │  1. GET /auth/sso/login    │                            │
       ├───────────────────────────>│                            │
       │  2. 302 Redirect to IdP    │                            │
       │<───────────────────────────┤                            │
       │                            │                            │
       │  3. Authenticate & Grant   │                            │
       ├────────────────────────────────────────────────────────>│
       │  4. 302 Callback with Code                              │
       │<────────────────────────────────────────────────────────┤
       │                            │                            │
       │  5. GET /auth/sso/callback?code=XYZ                     │
       ├───────────────────────────>│                            │
       │                            │  6. Exchange Code for JWT  │
       │                            ├───────────────────────────>│
       │                            │  7. Return Claims & Groups │
       │                            │<───────────────────────────┤
       │  8. Set Session Cookie     │                            │
       │<───────────────────────────┤                            │
```

#### OIDC Configuration Schema: `PUT /api/v1/auth/sso/config`

```json
{
  "enabled": true,
  "provider": "oidc",
  "issuer_url": "https://identity.enterprise.com/oauth2/v1",
  "client_id": "<oidc-client-id>",
  "client_secret": "<oidc-client-secret>",
  "redirect_url": "https://crenox-master.internal:8443/api/v1/auth/sso/callback",
  "scopes": ["openid", "profile", "email", "groups"]
}
```

#### Validation & Role-Based Access Control (RBAC)
Upon authorization code exchange, the controller validates:
1. **Cryptographic State / Nonce**: Prevents CSRF; nonces expire after 10 minutes.
2. **Identity Claims**: Extracts `email`, `sub`, and `groups` (or `roles`).
3. **Group Authorization**: Only users belonging to `CrenoxAdmins`, `SecOps`, or `SecurityAdministrators` are granted administrative console sessions.

---

### 4.2 Reverse Proxy Ingress Header Gateway (SAML 2.0)

In high-security, air-gapped enclaves where an external gateway (e.g., F5 BIG-IP, NGINX Plus, or AWS ALB) terminates SAML 2.0, identity claims are forwarded via downstream HTTP headers.

```nginx
# NGINX Edge Gateway Configuration
location / {
    proxy_pass https://crenox-master-upstream:8443;
    
    # Cryptographic proof of origin shared secret
    proxy_set_header X-Crenox-SSO-Secret "<proxy-shared-secret>";
    
    # Identity assertions extracted from SAML 2.0 token
    proxy_set_header X-Forwarded-User   $saml_username;
    proxy_set_header X-Forwarded-Email  $saml_email;
    proxy_set_header X-Forwarded-Groups $saml_groups;
}
```

#### Header Configuration Schema: `PUT /api/v1/auth/sso/config`

```json
{
  "enabled": true,
  "provider": "header",
  "header_user": "X-Forwarded-User",
  "header_email": "X-Forwarded-Email",
  "header_groups": "X-Forwarded-Groups",
  "proxy_shared_key": "<proxy-shared-secret>",
  "trusted_proxies": ["10.0.1.5", "10.0.1.6", "127.0.0.1"]
}
```

#### Security Validation Checks
1. **Source IP Whitelist**: Requests MUST originate from an IP address listed in `trusted_proxies`.
2. **Shared Secret Proof**: `X-Crenox-SSO-Secret` header MUST match `proxy_shared_key`.
3. **Tamper Rejection**: Requests presenting spoofed headers from untrusted network ranges are rejected with `HTTP 401 Unauthorized`.


---

[← Documentation Index](../README.md) · [Commercial EULA](../COMMERCIAL_EULA.md) · [Enterprise Specification & Pre-Order](https://crenoxhq.github.io/crenox/pro.html)
