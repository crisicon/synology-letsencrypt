# synology-letsencrypt
How to setup Synology to use Letsencrypt certificates without opening ports on the router


Let's Encrypt Automation on Synology NAS (Docker acme.sh)
=========================================================

System Summary
--------------
Certificate Renewal Engine : acme.sh running in Docker container
Container Name             : acme.sh1
Volume Mapping             : /volume1/docker/acme.sh -> /acme.sh
DNS Challenge Provider     : Cloudflare (via API Token)
Renewal Interval           : Every 90 days (Let's Encrypt default)
Scheduled Task             : Runs renewal, copies certs, restarts nginx

IMPORTANT NOTE: Use the original neilpang/acme.sh:latest

How to Connect to the NAS via SSH
---------------------------------
1. Enable SSH access in DSM:
   - Go to Control Panel > Terminal & SNMP > Terminal.
   - Enable SSH service on port 22.

2. Use a terminal or SSH client (e.g., PuTTY, macOS Terminal) to connect:
   ssh admin_username@your_nas_ip

3. You must have admin rights to manage Docker and system files.

Cloudflare Setup (API Token)
----------------------------
1. Log in to Cloudflare Dashboard.
2. Go to My Profile > API Tokens > Create Token.
3. Use the "Edit zone DNS" template.
4. Scope:
   - Include Zone Resources: All Zones (or specify your domain).
5. Copy the API Token securely.

You will use this token inside the Docker container as an environment variable: CF_Token

Container Setup Details (DSM Docker UI)
---------------------------------------
- Create container from Image: neilpang/acme.sh:latest
- Container Name: acme.sh1
- Advanced Settings:
  - Enable auto-restart
  - Volume Mapping:
    - Host Path: /volume1/docker/acme.sh
    - Container Path: /acme.sh
  - Environment Variables:
    - LE_CONFIG_HOME = /acme.sh
    - CF_Token = (Your Cloudflare API Token)
  - Execution Command:
    - /entry.sh sleep infinity
    (Container stays alive without executing any extra commands immediately.)
- Networking: Bridge mode (default)
- No exposed ports needed (uses DNS-01 validation only)

Important Setup Notes
----------------------
- Volume must map to `/acme.sh` exactly (no dot, not `/.acme.sh`).
- Do NOT configure daemon mode. Sleep infinity is sufficient.
- acme.sh is manually triggered when needed.
- Synology DSM binds services to certificates stored under `_archive/CERT_ID/`.

How to Discover DSM Active Certificate ID
------------------------------------------
1. SSH into NAS.
2. Run:
   cat /usr/syno/etc/certificate/_archive/INFO

3. Example output:
   {
     "default": "CERT_ID",
     "other_certificates": [...]
   }

The value under "default" is your current system-bound certificate ID.

Renewal Script (Scheduled Task content)
---------------------------------------
#!/bin/bash

# Force renew all certs inside Docker container
docker exec acme.sh1 acme.sh --renew-all --force

# Overwrite DSM live certs (replace CERT_ID with actual ID)
cp /volume1/docker/acme.sh/yourdomain.com_ecc/fullchain.cer /usr/syno/etc/certificate/_archive/CERT_ID/fullchain.pem
cp /volume1/docker/acme.sh/yourdomain.com_ecc/fullchain.cer /usr/syno/etc/certificate/_archive/CERT_ID/cert.pem
cp /volume1/docker/acme.sh/yourdomain.com_ecc/yourdomain.com.key /usr/syno/etc/certificate/_archive/CERT_ID/privkey.pem

# Restart nginx to apply new certificates
synosystemctl restart nginx

IMPOTANT NOTE: The scheduled task MUST be set to run as root

Future 90-Day Check Plan
-------------------------
1. SSH into NAS.
2. Check cert file validity:
   openssl x509 -in /usr/syno/etc/certificate/_archive/CERT_ID/cert.pem -noout -dates

3. Verify nginx serving the cert:
   openssl s_client -connect localhost:5001 -servername yourdomain.com </dev/null 2>/dev/null | openssl x509 -noout -dates

4. If needed, run Scheduled Task manually or trigger renewal via:
   docker exec acme.sh1 acme.sh --renew-all --force

Special Situations
-------------------
- If DSM default cert changes: Update Scheduled Task copy paths to new CERT_ID folder.
- If Docker/Surveillance Station error: "Repair" via DSM.
- If nginx stuck: synosystemctl restart nginx
- If SSL fully broken: Restore old certs from backup folders.

Key Commands Quick Reference
-----------------------------
- synosystemctl get-active-status nginx
- synosystemctl restart nginx
- cat /usr/syno/etc/certificate/_archive/INFO
- ls -lah /usr/syno/etc/certificate/_archive/<CERT_ID>/

Glossary
--------
- **SSL/TLS**: Protocol securing communication between server and client.
- **acme.sh**: Lightweight, pure shell script for managing Let's Encrypt certificates.
- **Cloudflare API Token**: A secure way to allow acme.sh to update DNS records automatically for DNS-01 challenges.
- **DSM nginx**: The built-in Synology webserver handling DSM UI and web services.
- **Certificate Archive**: Synology's internal system for storing and managing active certificates.

Mission Status: COMPLETE
-------------------------
- Fully automated Let's Encrypt renewal via Docker ✅
- Cert visible in DSM GUI and nginx ✅
- Documented step-by-step ✅
- Recovery and failover procedures validated ✅

