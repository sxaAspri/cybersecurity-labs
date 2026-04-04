# 🛡️ Mitigations & Defense Strategy — HTB Facts

This document analyzes the four vulnerabilities exploited during the Facts machine from a **Blue Team perspective**: what went wrong, why it was exploitable, how it could be detected, and how to fix it.

The goal is not just to list countermeasures, but to understand the *root design failure* behind each vulnerability so the same class of error is not repeated.

---

## Table of Contents

1. [CVE-2025-2304 – Mass Assignment (Privilege Escalation)](#vulnerability-1-cve-2025-2304--mass-assignment-privilege-escalation)
2. [CVE-2024-46987 – Path Traversal (File Disclosure)](#vulnerability-2-cve-2024-46987--path-traversal-file-disclosure)
3. [Weak SSH Key Passphrase](#vulnerability-3-weak-ssh-key-passphrase)
4. [Sudo Misconfiguration – Facter RCE](#vulnerability-4-sudo-misconfiguration--facter-rce)
5. [Defense-in-Depth Summary](#defense-in-depth-summary)

---

## Vulnerability #1: CVE-2025-2304 – Mass Assignment (Privilege Escalation)

### 🔍 What Happened on This Machine

After creating a low-privilege account (role: `client`), the endpoint `/admin/users/{id}/updated_ajax` was designed to allow users to update their own profile — password changes, email, etc. However, the Rails controller passed user-supplied parameters directly to the model update method without filtering which attributes could actually be modified.

By sending `password[role]=admin` in the request body, the server accepted the parameter, treated it as a valid attribute update, and silently elevated the account to administrator. There was no secondary authorization check verifying that a `client` user should never be able to modify their own `role`. The result: full CMS admin panel access from a freshly registered account.

This unlocked the S3/MinIO credential panel, which became the pivot point for the rest of the attack chain.

### Root Cause

- `permit!` (or equivalent unrestricted mass assignment) used in the Rails controller
- No attribute-level authorization — the endpoint checked *authentication* but not *what* the authenticated user was allowed to change
- Sensitive model fields (`role`, `is_admin`) were not excluded from user-controlled input

### Security Classification

| Standard | Reference |
|----------|-----------|
| CWE | CWE-915 – Improperly Controlled Modification of Object Attributes |
| OWASP | A01:2021 – Broken Access Control |

### 🛡️ Mitigations

**Fix the parameter filtering:**
```ruby
# WRONG — never do this
params.permit!

# CORRECT — whitelist only what users can change themselves
params.require(:user).permit(:email, :password, :password_confirmation)
```

**Separate endpoints by privilege level:**  
A user updating their password and an admin changing someone's role are fundamentally different operations. They should never share the same endpoint or controller action.

**Implement attribute-level authorization:**  
Use gems like `pundit` or `cancancan` with policies that explicitly define which attributes each role can modify.

**Never expose privileged fields in user-facing forms:**  
Fields like `role`, `is_admin`, `confirmed`, `locked` should never appear in any HTML form or be accepted as user input, regardless of validation.

### 🔎 Detection Opportunities

- **Log all role/privilege changes** with before/after values and the session that triggered them. A user modifying their own role should generate an immediate alert.
- **Alert on parameter anomalies:** Requests to profile update endpoints containing `role`, `admin`, `privilege`, or `permission` keys should be flagged.
- **Monitor for rapid privilege escalation:** A newly registered account becoming admin within minutes of creation is a strong indicator of exploitation.

---

## Vulnerability #2: CVE-2024-46987 – Path Traversal (File Disclosure)

### 🔍 What Happened on This Machine

The endpoint `/admin/download_private_file?file=` was intended to serve files stored in the application's S3/MinIO bucket to administrators. The `file` parameter was taken directly from the request and concatenated with a base path to construct the file location to read.

By injecting `../` sequences (e.g., `?file=../../etc/passwd`), the constructed path escaped the intended directory and resolved to arbitrary locations on the server's filesystem. This allowed reading `/etc/passwd` — revealing valid system users — and then directly extracting the SSH private key at `/home/trivia/.ssh/id_ed25519`.

Without this vulnerability, the S3 credentials extracted from step one would have been a dead end. The chaining of both CVEs is what made the SSH compromise possible.

### Root Cause

- Direct string concatenation of user input into file paths with no sanitization
- No path normalization (`realpath()`) to detect directory traversal
- No validation that the resolved path remained within the intended directory
- Application process had read access to sensitive system files (`/etc/passwd`, SSH keys)

### Security Classification

| Standard | Reference |
|----------|-----------|
| CWE | CWE-22 – Improper Limitation of a Pathname to a Restricted Directory |
| OWASP | A01:2021 – Broken Access Control |

### 🛡️ Mitigations

**Always resolve and validate the final path:**
```python
import os

BASE_DIR = "/app/private_files"
requested = request.args.get("file")
resolved = os.path.realpath(os.path.join(BASE_DIR, requested))

if not resolved.startswith(BASE_DIR):
    abort(403)  # Attempted traversal — reject
```

**Prefer indirect file references:**  
Instead of accepting file paths from users, store files in a database with UUIDs. The user submits `?file_id=abc123` and the server resolves the actual path internally. User input never touches the filesystem.

**Block common traversal patterns at the WAF/input layer:**  
Reject requests containing `../`, `%2e%2e`, `%252e`, null bytes (`%00`), or absolute path indicators.

**Apply least privilege to the application process:**  
The web app should never have read access to `/home/`, `/etc/shadow`, or SSH key directories. A properly isolated process can't leak what it can't read.

### 🔎 Detection Opportunities

- **Monitor for traversal patterns in logs:** Requests to file download endpoints containing `..`, `%2e`, or absolute paths (`/etc/`, `/home/`) should trigger immediate alerts.
- **File integrity monitoring:** Tools like AIDE or Wazuh can alert if sensitive files (`/etc/passwd`, SSH keys) are accessed by unusual processes.
- **Anomalous file access from web server process:** If `nginx` or `ruby` reads `/home/trivia/.ssh/id_ed25519`, that's a strong indicator of exploitation in progress.

---

## Vulnerability #3: Weak SSH Key Passphrase

### 🔍 What Happened on This Machine

The SSH private key for user `trivia` was protected with the passphrase `dragonballz`. Once the key was extracted via Path Traversal, it was converted to a crackable hash using `ssh2john` and then subjected to dictionary attack with `rockyou.txt`. The passphrase was found in approximately 2–3 minutes.

The private key itself was encrypted — which is correct practice — but the passphrase chosen was a single common word from pop culture, making the encryption effectively decorative. The attacker went from having an encrypted key to an active SSH session in under five minutes.

### Root Cause

- Passphrase selected from a common dictionary word (`dragonballz` appears in rockyou.txt)
- No organizational policy enforcing passphrase entropy for SSH keys
- No detection or alerting for offline cracking attempts (by definition undetectable once the key file is stolen)
- No additional authentication layer (e.g., MFA) required for SSH access

### Security Classification

| Standard | Reference |
|----------|-----------|
| CWE | CWE-521 – Weak Password Requirements |
| OWASP | A02:2021 – Cryptographic Failures |

### 🛡️ Mitigations

**Enforce strong passphrases:**  
A passphrase should be either a random string of ≥16 characters or a diceware phrase of ≥5 words. Single dictionary words — regardless of length — are crackable in seconds with modern GPU attacks.

**Use certificate-based SSH with short-lived certificates:**  
Tools like HashiCorp Vault SSH Secrets Engine or AWS EC2 Instance Connect issue signed certificates valid for minutes or hours. Even if a key is stolen, it's useless once the certificate expires.

**Implement SSH MFA:**  
Require a TOTP second factor for SSH logins via `pam_google_authenticator` or similar. A stolen key alone is not enough.

**Rotate SSH keys regularly:**  
Keys should have defined lifespans. A key that was created and never rotated will eventually be compromised — it's a matter of when.

**Restrict SSH access by IP or network:**  
If SSH should only be accessible from specific IPs (VPN, jump host), enforce this at the firewall level. Limits exposure even if credentials are compromised.

### 🔎 Detection Opportunities

- **Failed SSH login attempts:** Multiple failed attempts before a success may indicate credential stuffing or key testing, especially from unfamiliar IPs.
- **Geolocation / time anomalies:** A login from an unexpected country or at an unusual hour warrants investigation.
- **Key file access events:** As noted above — if the key file was read by the web server process, that precedes the SSH compromise and is the detection opportunity.

---

## Vulnerability #4: Sudo Misconfiguration – Facter RCE

### 🔍 What Happened on This Machine

The user `trivia` had the following sudo permission:

```
(ALL) NOPASSWD: /usr/bin/facter
```

Facter is a Puppet tool that collects system "facts" (hardware info, OS details, etc.). It supports a `--custom-dir` flag that loads additional facts from a user-specified directory. Custom facts are written in Ruby and executed at runtime.

By creating a file `/tmp/custom/shell.rb` with a single Ruby instruction (`exec('/bin/sh')`), and then running `sudo /usr/bin/facter --custom-dir=/tmp/custom`, the Ruby code executed as root — spawning a root shell. The entire privilege escalation took under two minutes from discovering the sudo entry.

The problem was not that `facter` is inherently dangerous, but that the sudo permission allowed passing arbitrary arguments (`--custom-dir`) pointing to attacker-controlled content.

### Root Cause

- Sudo permission granted on the binary without restricting arguments
- `facter` supports dynamic code execution via `--custom-dir` — this was not considered when granting the permission
- No audit of what granted binaries are actually capable of
- The principle of least privilege was not applied: `trivia` didn't need to run `facter` as root for any legitimate purpose

### Security Classification

| Standard | Reference |
|----------|-----------|
| CWE | CWE-250 – Execution with Unnecessary Privileges |
| OWASP | A01:2021 – Broken Access Control |
| MITRE ATT&CK | T1548.003 – Abuse Elevation Control Mechanism: Sudo |

### 🛡️ Mitigations

**Restrict arguments in sudoers, not just the binary:**
```bash
# WRONG — allows any argument, including --custom-dir
trivia ALL=(ALL) NOPASSWD: /usr/bin/facter

# BETTER — restrict to no arguments or specific safe flags only
# (facter with no custom-dir is still limited — review what flags are safe)
trivia ALL=(ALL) NOPASSWD: /usr/bin/facter ""
```

**Audit every sudo entry against the binary's full capability:**  
Before granting sudo access to any tool, read its man page completely. If it accepts paths, scripts, interpreters, or plugins as arguments — it can likely be abused. GTFOBins (https://gtfobins.github.io/) is an essential reference.

**Ask: does this user actually need this?**  
In this case, `trivia` was a web application user. There is no legitimate reason for a web app user to run Puppet fact collection as root. The permission should not have existed at all.

**Avoid granting sudo to any binary that executes dynamic code:**  
Python, Ruby, Perl, Node, bash, awk, find, vim, less, env — all can be used to spawn shells or execute arbitrary code if run as root. Prefer purpose-built, argument-restricted wrappers written in compiled languages if elevated execution is truly needed.

**Use `NOEXEC` where applicable:**  
The `NOEXEC` sudoers option prevents the binary from executing other programs via shell escapes, limiting some (but not all) escalation paths.

### 🔎 Detection Opportunities

- **Log all sudo executions with full argument strings:** `sudo -l` shows permissions; actual executions should be captured in `/var/log/auth.log` or via auditd.
- **Alert on `--custom-dir` or similar dynamic-path flags** in monitored binaries.
- **File creation in `/tmp` followed immediately by a sudo execution** is a strong behavioral indicator of this attack pattern.
- **Monitor for new shell processes spawned by root** when the parent process is `facter` or similar unexpected tools.

---

## Defense-in-Depth Summary

This machine demonstrates that **no single vulnerability caused the breach**. Each step only became possible because the previous one succeeded. The real lesson is about layered failure:

```
Broken access control in CMS
  → exposed cloud credentials
    → enabled path traversal to read private keys
      → weak passphrase allowed offline cracking
        → sudo misconfiguration completed root escalation
```

Break any single link and the chain fails. That's the point of defense-in-depth.

### Controls That Would Have Stopped This Chain

| Layer | Control | Breaks Chain At |
|-------|---------|----------------|
| Application | Strict parameter whitelisting | Step 1 — no admin escalation |
| Application | Path validation with `realpath()` | Step 2 — no file disclosure |
| OS | Restrict web app process to app directory | Step 2 — even if traversal worked |
| Credential | Strong SSH passphrase (diceware) | Step 3 — key unusable after theft |
| SSH | Certificate-based auth with short TTL | Step 3 — stolen key expires immediately |
| OS | Properly scoped sudo or no sudo | Step 4 — no root escalation |
| Monitoring | Alerting on role changes + path traversal patterns | Any step — early detection |

### Key Principles Reinforced

- **Principle of Least Privilege** — applies to user roles, file permissions, process permissions, and sudo grants equally
- **Never trust user input** — validate, sanitize, and reject anything that touches the filesystem or sensitive model attributes
- **Authentication ≠ Authorization** — verifying *who* someone is does not automatically determine *what* they're allowed to do
- **Know your tools** — every binary you grant elevated permissions to must be fully audited for misuse potential
- **Detection is not optional** — prevention will eventually fail; logging, monitoring, and alerting are what limit the blast radius

---

**Author:** Aspri · **Date:** April 2026  
**Reference machine:** HTB Facts (Easy) — [Full Walkthrough](./WALKTHROUGH.md)
