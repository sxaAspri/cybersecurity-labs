# HTB – Facts Machine

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=flat-square)
![OS](https://img.shields.io/badge/OS-Linux%20Ubuntu-orange?style=flat-square)
![Status](https://img.shields.io/badge/Status-Completed-success?style=flat-square)

## Overview

**Facts** is an Easy-rated Linux machine on Hack The Box (Starting Point). Despite the difficulty label, it requires chaining multiple real-world vulnerabilities across a web CMS, SSH key cracking, and a sudo misconfiguration — mimicking a realistic attack path from anonymous web access to full root compromise.

**Services exposed:** SSH (22), HTTP/Nginx (80), MinIO S3 (54321)  
**Target application:** Camaleon CMS v2.9.0

---

## Attack Chain

```
Anonymous HTTP access
  └─► Register as low-privilege user (role: client)
        └─► CVE-2025-2304: Mass Assignment → escalate to CMS Admin
              └─► Extract S3/MinIO credentials from admin settings
                    └─► CVE-2024-46987: Path Traversal → read /etc/passwd + SSH private key
                          └─► Crack SSH key passphrase with John the Ripper (rockyou.txt)
                                └─► SSH as user "trivia" → capture user flag
                                      └─► Sudo misconfiguration on /usr/bin/facter
                                            └─► Malicious Ruby custom fact → RCE as root
                                                  └─► Capture root flag ✅
```

---

## CVEs & Techniques

| # | Vulnerability | Type | Impact |
|---|--------------|------|--------|
| 1 | CVE-2025-2304 | Mass Assignment (Rails) | Client → CMS Admin |
| 2 | CVE-2024-46987 | Path Traversal | Read arbitrary server files |
| 3 | Weak SSH passphrase | Credential weakness | SSH key decrypted via dictionary |
| 4 | Sudo misconfiguration | Facter `--custom-dir` abuse | Local privilege escalation to root |

---

## Files in This Folder

| File | Description |
|------|-------------|
| `WALKTHROUGH.md` | Full technical writeup with commands, outputs and analysis |
| `MITIGATIONS.md` | Blue Team perspective — root causes, detection and remediation |

---

## Key Takeaways

- A single weak CMS endpoint exposed an entire privilege chain
- Path Traversal became critical only *after* gaining admin access — vulnerability chaining matters
- Dictionary selection is decisive in credential cracking (rockyou.txt vs. /usr/share/dict/words)
- Sudo permissions on tools that execute dynamic code (Ruby, Python) are high-risk regardless of the binary itself

---

**Author:** Aspri · **Date:** April 2026 · **Platform:** Hack The Box – Starting Point
