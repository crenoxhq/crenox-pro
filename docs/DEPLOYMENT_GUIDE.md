# Crenox Pro — Enterprise Deployment & Operations Guide

This guide provides infrastructure engineers, systems administrators, and SecOps teams with architectural specifications and operational procedures for deploying, hardening, and maintaining the Crenox Pro central controller (`crenox-master`).

---

## 1. System Architecture & Sizing

The Crenox Pro server operates as a single, statically compiled executable (`CGO_ENABLED=0`). It requires zero external database daemons (e.g., PostgreSQL, MySQL), zero container runtimes, and zero interpreted runtimes (Node.js, Python).

### Architectural Model
- **Master Controller (`crenox-master`)**: Serves the REST API, coordinates workstation fleet registrations, manages cryptographic policy distribution, hosts the embedded administrative console, and maintains the persistent audit ledger.
- **Persistent Storage**: Utilizes an embedded, crash-safe, pure Go persistent storage engine with atomic writes and asynchronous disk flushing.
- **Embedded Web Console**: All HTML5, CSS, SVG icons, and vanilla JavaScript assets are embedded directly into the binary at compilation time (`go:embed`). No external CDN or internet access is ever required.

### Hardware Resource Allocations
| Deployment Tier | Active Enrolled Seats | vCPU | RAM | Disk Storage |
| :--- | :--- | :---: | :---: | :---: |
| **Small / Pilot** | Up to 500 workstations | 2 | 2 GB | 20 GB SSD |
| **Medium Enterprise** | Up to 5,000 workstations | 4 | 4 GB | 50 GB SSD |
| **Global Enterprise** | 25,000+ workstations | 8 | 8 GB | 150 GB SSD |

---

## 2. Server Installation & Execution

### Binary Acquisition
Place the `crenox-master` binary in `/usr/local/bin` and set standard execution permissions:

```bash
chmod +x /usr/local/bin/crenox-master
```

### Initial Configuration Parameters
The controller is started using the `serve` command:

```bash
crenox-master serve \
  -port :8443 \
  -db /var/lib/crenox/crenox-master.db \
  -license /etc/crenox/crenox.lic \
  -admin-pass "<YOUR_SECURE_ADMIN_PASSWORD>" \
  -agent-key "<ENTERPRISE_CLUSTER_SECRET_KEY>" \
  -auto-tls
```

#### Core Flags:
- `-port` (or `-bind`): TCP address and port (default: `:8443`).
- `-db`: File path to the persistent storage database (default: `crenox-master.db`).
- `-license`: File path to the cryptographically signed `.lic` commercial license file.
- `-admin-pass` — Master password for console authentication (or set `CRENOX_ADMIN_PASSWORD`).
- `-agent-key`: Shared cluster secret for headless/CI agent registration when not using individual Seat Enrollment Tokens (or set `CRENOX_AGENT_KEY`; automatically generated and persisted in `crenox-master.db` if omitted).
- `-auto-tls`: Automatically generates and persists a self-signed TLS certificate (`auto-tls.crt` / `auto-tls.key`) in the database directory when dedicated certificates are not supplied.

### First-Time Interactive Web Setup Wizard
If `crenox-master serve` is executed without supplying `-admin-pass` (or `CRENOX_ADMIN_PASSWORD`), the controller starts in First-Time Setup Mode:
1. Navigate to the web dashboard at `https://<controller-host>:8443`.
2. When accessing from a remote network address, enter the one-time setup token generated in `<db-dir>/initial-setup.token` alongside your chosen administrator credentials (direct loopback requests on `127.0.0.1` do not require the token).
3. Once initialized, `initial-setup.token` is deleted automatically, setup mode permanently locks, and future logins authenticate via standard session tokens.

---

## 3. Production Deployment Architectures

### Option A: Systemd Service Unit (Recommended)

Running `crenox-master` under `systemd` guarantees automated recovery, lifecycle management, and operating system sandboxing.

1. Create a dedicated system user and data directory:
   ```bash
   useradd -r -s /bin/false -d /var/lib/crenox crenox
   mkdir -p /var/lib/crenox /etc/crenox
   chown -R crenox:crenox /var/lib/crenox /etc/crenox
   chmod 700 /var/lib/crenox /etc/crenox
   ```

2. Create the service definition at `/etc/systemd/system/crenox-master.service`:
   ```ini
   [Unit]
   Description=Crenox Pro Central Security Controller
   After=network.target
   Documentation=https://github.com/crenoxhq/crenox-pro

   [Service]
   Type=simple
   User=crenox
   Group=crenox
   ExecStart=/usr/local/bin/crenox-master serve \
     -port 127.0.0.1:8443 \
     -db /var/lib/crenox/crenox-master.db \
     -license /etc/crenox/crenox.lic \
     -admin-pass "<ADMIN_PASSWORD>" \
     -agent-key "<CLUSTER_SECRET_KEY>"

   Restart=always
   RestartSec=5s

   # Process Sandboxing & Hardening
   LimitNOFILE=65536
   ProtectSystem=strict
   ProtectHome=true
   NoNewPrivileges=true
   ReadWritePaths=/var/lib/crenox
   ReadOnlyPaths=/etc/crenox
   PrivateTmp=true
   CapabilityBoundingSet=

   [Install]
   WantedBy=multi-user.target
   ```

3. Enable and activate the service:
   ```bash
   systemctl daemon-reload
   systemctl enable --now crenox-master.service
   systemctl status crenox-master.service
   ```

---

### Option B: NGINX Reverse Proxy with Enterprise TLS

In production environments, placing `crenox-master` behind NGINX provides TLS 1.3 termination, rate limiting, and standard port binding (`443`).

Create `/etc/nginx/conf.d/crenox.conf`:

```nginx
upstream crenox_backend {
    server 127.0.0.1:8443;
    keepalive 32;
}

server {
    listen 80;
    server_name crenox.enterprise.internal;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name crenox.enterprise.internal;

    ssl_certificate     /etc/ssl/certs/crenox-enterprise.crt;
    ssl_certificate_key /etc/ssl/private/crenox-enterprise.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Maximum upload body size (10MB general JSON, 64MB for multipart OTA binary uploads)
    client_max_body_size 64M;

    # Event Stream (SSE)
    location /api/v1/fleet/stream {
        proxy_pass http://crenox_backend;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header Connection "";

        # Required settings for Server-Sent Events (SSE)
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding off;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    location / {
        proxy_pass http://crenox_backend;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header Connection "";

        proxy_connect_timeout 10s;
        proxy_read_timeout    60s;
        proxy_send_timeout    60s;
    }
}
```

#### Enterprise Private CA Certificate Configuration
If your enterprise deploys internal root or intermediate Certificate Authorities (CAs) rather than public commercial certificates, configure developer workstations to validate the master node by setting `CRENOX_CA_CERT`:

```bash
export CRENOX_CA_CERT="/etc/ssl/certs/enterprise-internal-ca.pem"
```

The agent loads and appends the custom certificate bundle into the host operating system's system certificate pool during all master-agent communication.

---

## 4. Host Node-Locking & Commercial Licensing

Crenox Pro enforces an offline licensing model for private environments. To verify seat limits and prevent license duplication without external network communication, each commercial license file (`crenox.lic`) is cryptographically bound to the designated production host.

### Single-Master Instance Policy
- **One License per Master**: Each commercial subscription authorizes one (1) active production instance of `crenox-master`.
- **Node-Locked Binding**: The license is cryptographically signed and bound to the hardware digest of the designated master server host.
- **Offline Verification**: Because `crenox-master` operates without external activation servers, host binding verifies authorized deployment while preserving network isolation.
- **Multiple Instances**: Organizations requiring separate master controllers for distinct environments (such as isolated network enclaves, staging environments, or disaster-recovery systems) require dedicated licenses for each master node.

### Provisioning & Issuance Workflow

```
[Target Master Host]                    [CrenoxHQ Commercial Desk]
         │                                          │
 1. Run 'crenox-master fingerprint'                 │
         │                                          │
 2. Transmit Host Fingerprint ─────────────────────>│
         │                                    3. Verify Order &
         │                                       Generate 'crenox.lic'
         │                                       (Bound to Fingerprint)
         │<──────────────────────────────────── 4. Deliver License File
 5. Install at /etc/crenox/crenox.lic               │
 6. Start service & verify via 'status'             │
```

#### Step 1: Extract Host Node Fingerprint
On the physical server or virtual machine designated to run `crenox-master`, execute:

```bash
crenox-master fingerprint
```

Standard Output:
```
Crenox Pro Master — Hardware Node Fingerprint
─────────────────────────────────────────────────────────────
  Node Host ID : CNX-4A9F-B3C1-90E2-55D4
  Hostname     : controller-prod-01.internal
  Architecture : linux/amd64
─────────────────────────────────────────────────────────────
Provide this Node Host ID to CrenoxHQ when requesting a node-locked
Enterprise Master license key.
```

#### Step 2: Transmit Fingerprint to CrenoxHQ
Transmit the generated `Node Host ID` (e.g., `CNX-4A9F-B3C1-90E2-55D4`) alongside your organization details to the Crenox Commercial Desk (`crenoxhq@proton.me`).

#### Step 3: Receive and Install License File
Upon issuance, CrenoxHQ provides a cryptographically signed `crenox.lic` file containing your authorized seat capacity, licensed feature set, validity period, and embedded node digest.

Place the file on the master server with restricted permissions:

```bash
mkdir -p /etc/crenox
cp crenox.lic /etc/crenox/crenox.lic
chmod 600 /etc/crenox/crenox.lic
chown crenox:crenox /etc/crenox/crenox.lic
```

#### Step 4: Initialize Database and Verify License Status
Initialize the database (if not yet created by `serve`) and confirm that the license matches the server hardware:

```bash
crenox-master init -db /var/lib/crenox/crenox-master.db -license /etc/crenox/crenox.lic
crenox-master status -db /var/lib/crenox/crenox-master.db -license /etc/crenox/crenox.lic
```

Output confirms the active customer name, authorized seat quota, hardware binding status (`VALID`), and expiration boundary.

### Hardware Migration & Server Replacement Policy
If a production master server must be decommissioned, upgraded, or migrated to new hardware:
1. Copy the existing database (`crenox-master.db`) and license (`crenox.lic`) to the replacement server and start `crenox-master`.
2. Upon detecting the new hardware fingerprint, `crenox-master` automatically enters a **14-day hardware migration grace period** (`migration_grace`) without interrupting existing operations or workstation enrollments.
3. Execute `crenox-master fingerprint` on the new host (or copy the current server fingerprint from the **License & Quotas** tab in the web console) and send a **License Re-Key Request** to `crenoxhq@proton.me`.
4. Upload the replacement `crenox.lic` via the web console (**License & Quotas → Input License Key**) or replace `/etc/crenox/crenox.lic`. The database automatically re-keys its encrypted records to the new license without downtime.

**Contractual Migration Terms:**
- **Annual Migration Quota**: Customers receive up to two (2) complimentary hardware re-keying transfers per 12-month subscription term.
- **14-Day Migration Grace Window**: During the 14-day grace window, full administrative and enrollment capabilities remain active. If a replacement license is not installed within 14 days, administrative features and new device enrollments pause while existing enrolled workstations continue normal pre-commit scanning.

---

## 5. Backup, Maintenance & Disaster Recovery

The embedded database engine utilizes atomic file swapping, asynchronous batched fsync, and automated disaster recovery from rolling backups (.bak).

### Automated Database Backup
Because database writes are synchronized to disk via atomic file swapping (`.tmp` to `.db`), creating a consistent backup requires copying the database file:

```bash
# Automated daily snapshot script
DATE=$(date +%Y%m%d_%H%M%S)
cp /var/lib/crenox/crenox-master.db /backup/crenox/crenox-master-$DATE.db
chmod 600 /backup/crenox/crenox-master-$DATE.db
```

### Air-Gapped Diagnostic Bundle (`bug-report`)
Before requesting technical support or during security incident analysis, generate an air-gapped, sanitized diagnostic package:

```bash
crenox-master bug-report \
  -db /var/lib/crenox/crenox-master.db \
  -license /etc/crenox/crenox.lic \
  -out /tmp/crenox-diagnostic-bundle.zip
```

The diagnostic facility:
- Sanitizes and redacts passwords, proprietary secrets, and private API keys using over 50 automated regex rules.
- Gathers hardware metrics, fleet enrollment quotas, system logs, and database integrity statistics.
- Generates an `integrity.json` manifest with cryptographic SHA-256 digests and file sizes verifying the package's contents.

### Security Audit Ledger & CSV Export
The central master controller maintains an append-only, tamper-evident audit ledger capturing blocked secrets, bypass requests, configuration drift, and policy tampering alerts.
- **Web Console Export**: SecOps administrators can export audit events directly to CSV from the **Security Events** tab using the **Export CSV** button.
- **Granular Event Filtering**: Exports can be generated for all security events or filtered by specific event types (such as `SECRET_BLOCKED`, `TAMPER_DETECTED`, `NO_VERIFY_VIOLATION`, or `POLICY_SYNC`).
- **Automated SIEM Integration**: The REST API supports programmatic exports via `GET /api/v1/audit/export?format=csv&event_type=<EVENT_TYPE>`.



---

[← Documentation Index](../README.md) · [Commercial EULA](../COMMERCIAL_EULA.md) · [Enterprise Specification & Pre-Order](https://crenoxhq.github.io/crenox/pro.html)
