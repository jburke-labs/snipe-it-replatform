# Snipe-IT Replatform , Proxmox LXC to Dedicated Ubuntu Docker Host

**Project Cerberus Homelab · Internal Asset Management Platform Migration + Security Hardening**

---

## Overview

This project covers the full replatform of a production Snipe-IT asset management deployment from a Proxmox community-script LXC into a dedicated Ubuntu 24.04 host running Docker Compose, with a Caddy reverse proxy providing internal HTTPS and a complete data migration preserving all users, assets, devices, history and uploads.

The goal was not simply to get Snipe-IT running on different hardware. The objective was to build something defensible , controlled access, no internet exposure, a repeatable and documented build, and full data integrity proven end to end. The design was also built with corporate deployment in mind from the start, with VLAN segmentation, ACL enforcement, and an ISO 27001-aligned evidence approach built into the planning rather than applied retrospectively.

---

## Environment

| Component | Detail |
|---|---|
| Platform | Dell OptiPlex 7070 SFF (i7 / 16GB RAM) |
| OS | Ubuntu 24.04 LTS + XFCE |
| App Stack | Docker Compose , Snipe-IT + MariaDB + Caddy |
| TLS | Internal CA via Caddy |
| Internal DNS | snipeit.local |
| Source Environment | Proxmox LXC via community helper script |
| Migration Method | MariaDB dump + upload archive + APP_KEY transfer |

---

## Final Architecture

```
IT Admin Device
  → HTTPS 443
    → Caddy Reverse Proxy (0.0.0.0:443)
      → Snipe-IT App Container (127.0.0.1:8080 , localhost only)
        → MariaDB Container (Docker internal network , no LAN exposure)
```

Key design decisions:

- Snipe-IT app container bound to `127.0.0.1` only , no direct LAN access to the app port
- Caddy is the only externally reachable component, handling TLS termination
- MariaDB has no exposed ports outside the Docker network
- All access must come through the reverse proxy , no port-knocking the app directly

---

## Why This Was Done This Way

The source deployment was installed via the Proxmox community helper script. It worked, but it was not a repeatable build, it had no internal TLS, and it was not structured in a way that could be cleanly handed to an auditor or deployed into a segmented corporate network without significant rework.

Moving to Docker Compose solved several problems at once. The build became reproducible from a `docker-compose.yml` and a `.env` file. The reverse proxy and TLS handling became explicit and controllable. The data layer was isolated properly. And the whole thing could be documented cleanly as a known state rather than "whatever the script did."

The OptiPlex 7070 SFF was chosen because it fits the profile of an internal service appliance, stable, power efficient, physically small, and suitable for rack placement with a dedicated VLAN and ACL enforcement in front of it.

---

## Build Phases and Key Issues

### Phase A , OS and Clean Slate

The OptiPlex previously had BitLocker enabled with no recovery key available. A fresh Ubuntu 24.04 LTS install from a live USB gave a clean baseline with no inherited state or unknown configuration to work around.

---

### Phase B , Network and DNS Not Working After Install

**Symptoms:**
- `ping 1.1.1.1` failed with network is unreachable
- Hostname resolution failing

**Root causes:**
- Missing network tooling in the minimal install
- Interface link state down with no default route
- DNS resolvers not set or not persisting across reboots

**Actions taken:**
- Verified interface and link state, checked routing table
- Temporarily set IP and route manually to prove L2 and L3 independently
- Corrected netplan YAML , invalid formatting and file permissions too open
- Set DNS resolvers and confirmed persistence after reboot

**Outcome:** Stable network and working DNS.

---

### Phase C , XFCE GUI Session Failure

**Problem:** After XFCE install, login screen returned "Failed to start session". Username mismatch between GUI and CLI added confusion.

**Root cause:** Incomplete session dependencies and display manager session mismatch.

**Fix:** Installed missing XFCE session components and selected the correct session at the login screen. Confirmed stable after reboot.

---

### Phase D , Docker Compose YAML Validation Issues

**Problems:**
- Schema errors , `services.networks` additional properties not allowed
- Indentation mistakes and incorrect placement of `networks:` and `volumes:` blocks
- Caddy service definition confusion around how config files are mounted

**Fixes:**
- Standardised the Compose structure , `services:` blocks at the top level, `networks:` and `volumes:` correctly defined at root
- Used `docker compose config` to validate output before starting any services
- Created the Caddy config as a real file (not a directory) and mounted it correctly

**Outcome:** Docker stack started cleanly.

---

### Phase E , Containers Up but Site Not Loading

**Symptom:** Containers showing as running but browser could not reach the site.

**Root cause:** The app container was bound to localhost only. Network clients must go through Caddy , there is no direct path to the app port from the LAN.

**Fix:**
```bash
ss -tulpen       # confirmed app only on 127.0.0.1:8080
docker compose ps  # confirmed Caddy on 0.0.0.0:443
```

Confirmed UFW was not blocking port 443. Site became reachable over internal HTTPS.

---

### Phase F , APP_URL Double-HTTPS Redirect Loop

**Symptom:** Root endpoint returned HTTP 302 with `Location: https://https://snipeit.local/setup`

**Root causes:**
- Bad `APP_URL` value in `.env` , double-scheme caused by incorrect value
- Cached Laravel config persisting after the `.env` change

**Fix:**
```bash
# Corrected .env
APP_URL=https://snipeit.local

# Cleared Laravel cache stack
php artisan config:clear
php artisan cache:clear
php artisan route:clear
php artisan view:clear
```

Verified with `curl -kI https://snipeit.local` , clean redirect to `/login`.

---

### Phase G , TLS Behaviour and SNI Mismatch

**Symptoms:** TLS handshake internal errors when accessing via IP address or `localhost` instead of the configured hostname.

**Root cause:** Internal TLS via Caddy requires hostname-based access. SNI (Server Name Indication) uses the hostname in the request to select the correct certificate. Accessing by IP bypasses this and causes a mismatch.

**Fix:** Hostname-based access only , `https://snipeit.local`. Verified with:

```bash
curl -vkI --resolve snipeit.local:443:127.0.0.1 https://snipeit.local/login
```

**Outcome:** HTTPS stable via hostname. Expected certificate warnings on clients until trust is deployed via GPO or Intune.

---

## Migration , Proxmox LXC to Docker

The source Snipe-IT was not a Docker deployment. It was installed via the Proxmox community helper script, which means the migration method was export and import rather than a container lift. The underlying paths and structure were different, so attempting to move the container itself would have been more complicated and less reliable than a clean data export.

### Data Extracted from the Source LXC

```bash
# Confirm source database
mysql -u root -e "SHOW DATABASES;"

# Export database
mysqldump -u root snipeit_db > snipeit_db.sql

# Export uploads
tar -czf snipeit_uploads.tgz /var/www/snipeit/public/uploads

# Note APP_KEY from source .env , critical step
grep APP_KEY /var/www/snipeit/.env
```

**Critical finding at this stage:** The `APP_KEY` on the source and target environments was different. Laravel uses this key to encrypt and decrypt values stored in the database. Importing data without aligning the key first would have caused silent decryption failures , not obvious errors, just corrupt-looking data and broken behaviour.

### Import into Docker Host

```bash
# Take a break-glass backup before overwriting anything
docker exec snipeit-db mysqldump -u snipeit -p snipeit > optiplex_preimport_backup.sql

# Stop app and proxy during import
docker compose stop snipeit caddy

# Drop and recreate target database, then import
docker exec -i snipeit-db mysql -u snipeit -p snipeit < snipeit_db.sql

# Validate record counts post-import
docker exec snipeit-db mysql -u snipeit -p snipeit \
  -e "SELECT COUNT(*) FROM users; SELECT COUNT(*) FROM assets;"
```

**Post-import record counts:**
- Users: 82 ✅
- Assets: 148 ✅

### Upload Path Mismatch

**Issue discovered:** The Snipe-IT Docker image uses an internal symlink:

```
/var/www/html/public/uploads → /var/lib/snipeit/data/uploads
```

Uploads restored into `./data/snipeit/public/uploads` on the host were not visible to the app because the container resolves uploads through the symlink path, which maps to `./data/snipeit/data/uploads` on the host.

**Fix:** Synced uploads into the correct host path and verified inside the container:

```bash
docker exec snipeit ls /var/lib/snipeit/data/uploads
# logos  avatars  barcodes  manufacturers  , all present
```

### APP_KEY Alignment

Updated the OptiPlex `.env` `APP_KEY` to match the source LXC value, then cleared Laravel caches. After this change UI behaviour was stable, logins worked correctly, and all assets and users were visible with no decryption errors.

---

## Final Validation

| Check | Result |
|---|---|
| HTTPS endpoint `/login` | ✅ HTTP/2 200 |
| User login | ✅ Working |
| Users visible (82) | ✅ Confirmed |
| Assets visible (148) | ✅ Confirmed |
| Uploads rendering | ✅ Logos, avatars, barcodes present |
| Caddy on 443 externally | ✅ Confirmed |
| App on localhost only | ✅ Confirmed |
| DB internal only | ✅ Confirmed |

---

## Corporate Deployment Design

The build was designed from the start with a corporate deployment in mind. The following controls were planned and documented as part of the project rather than as an afterthought.

### Network Segmentation

- Dedicated VLAN for the Snipe-IT host
- FortiGate ACL allowlist , IT admin subnets only inbound on TCP 443
- Default deny all other VLANs
- No internet inbound , no VIP or NAT for this service
- Outbound restricted to OS update sources only, or routed via internal proxy or mirror

### TLS Trust Deployment

Two options depending on environment:

1. **Preferred:** Corporate PKI certificate issued for the internal hostname , no client-side trust steps required
2. **Alternative:** Export Caddy root CA and deploy to IT admin endpoints via GPO or Intune Trusted Root store

### SSH Hardening

```bash
# /etc/ssh/sshd_config
PasswordAuthentication no
PermitRootLogin no
# Restrict AllowUsers to named admin accounts only
# Restrict access by source subnet at firewall level
```

### Docker Hardening , Planned Next Steps

- Pin image versions , no `:latest` in production
- Document update cadence and patch window
- Implement scheduled backup jobs with restore testing for audit evidence

---

## ISO 27001:2022 Control Mapping

| Control Area | Implementation |
|---|---|
| Asset Management | Snipe-IT is the asset register , this project is the tool that supports the control |
| Access Control | VLAN segmentation, IP allowlisting, admin-only access design |
| Secure Configuration | Docker hardening, SSH hardening, UFW default deny |
| Cryptography | Internal TLS via Caddy, APP_KEY integrity preserved through migration |
| Network Security | No internet inbound, FortiGate ACL enforcement, isolated host design |
| Logging and Monitoring | Fail2ban, UFW logging, Wazuh agent onboarding planned |
| Backup and Recovery | Break-glass backup taken before import, restore testing planned |
| Change Management | Full migration documented, each step validated before proceeding |

---

## Key Technical Takeaways

**The migration method matters as much as the migration itself.** Moving a script-deployed application into a Docker Compose stack is not a simple lift. Understanding what the source environment actually looked like , how it was installed, where it stored things, how it handled encryption , made the difference between a clean migration and a broken one.

**APP_KEY is not optional.** Laravel's encryption is tied to the key used when the data was first written. Getting this wrong produces silent failures that are harder to diagnose than an obvious error message. Checking and aligning the key before import is not a minor step , it is a required one.

**Container symlinks will catch you out if you assume the tar structure maps directly.** Validating the actual path inside the running container before deciding where to restore data is the right approach. Assumptions about paths are where migrations can & will go wrong.

**Building for auditability from the start costs very little extra.** Documenting design decisions, keeping an evidence pack in mind during the build, and planning ACL structure/controls upfront takes no more time than doing it reactively , and produces a significantly cleaner result.

---

## Skills Demonstrated

- Linux server provisioning and troubleshooting on Ubuntu 24.04 LTS
- Network debugging , link state, routing, DNS, netplan YAML validation
- Docker Compose deployment and YAML schema debugging
- Reverse proxy and internal TLS configuration using Caddy
- Web application debugging , redirects, environment config, Laravel cache behaviour
- Database migration , MariaDB dump and import with post-migration integrity validation
- Application-specific migration considerations , upload paths, container symlinks, encryption key alignment
- Security-first design , segmentation, least privilege, controlled egress, non-internet-facing posture
- ISO 27001-aligned thinking applied throughout rather than retrospectively

---

## Full Documentation

The complete project writeup including full context, extended build notes, corporate deployment checklist, and ISO 27001 control mapping is available in the [Project Cerberus Notion workspace](https://dent-trampoline-c53.notion.site/Project-Cerberus-Public-Portfolio-351fe4fa7fd3816384c5d06cabbc1a9d?source=copy_link).

---

## Related Projects

- [Project Cerberus , Infrastructure Overview](https://jburke-labs.github.io/project-cerberus)
- [NGINX Proxy Manager + AdGuard + Wildcard Certificate](https://github.com/jburke-labs/nginx-proxy-wildcard-cert)
- [WireGuard VPN , Double NAT Deployment](https://github.com/jburke-labs/wireguard-vpn-deployment)
- [Wazuh SOC Automation Pipeline](https://github.com/jburke-labs/wazuh-soc-automation)
