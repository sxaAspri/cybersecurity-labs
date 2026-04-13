# Silentium — Mitigations & Blue Team Reference

![Blue Team](https://img.shields.io/badge/Perspective-Blue%20Team-blue?style=for-the-badge)
![MITRE ATT&CK](https://img.shields.io/badge/Framework-MITRE%20ATT%26CK-red?style=for-the-badge)
![OWASP](https://img.shields.io/badge/Framework-OWASP-orange?style=for-the-badge)

> This document is intended as a genuine Blue Team reference. It maps each vulnerability in the Silentium attack chain to its root cause, real-world parallels, detection opportunities, and layered defense recommendations. Remediation code is intentionally excluded — the focus is on architectural and operational security posture.

---

## Table of Contents

- [Vulnerability 1 — Password Reset Token Disclosure](#vulnerability-1--password-reset-token-disclosure)
- [Vulnerability 2 — CVE-2025-59528: Flowise CustomMCP Code Injection (RCE)](#vulnerability-2--cve-2025-59528-flowise-custommcp-code-injection-rce)
- [Vulnerability 3 — Credential Exposure via Container Environment Variables](#vulnerability-3--credential-exposure-via-container-environment-variables)
- [Vulnerability 4 — CVE-2025-8110: Gogs Symlink Path Traversal (Privilege Escalation)](#vulnerability-4--cve-2025-8110-gogs-symlink-path-traversal-privilege-escalation)
- [Defense-in-Depth Kill Chain Mapping](#defense-in-depth-kill-chain-mapping)

---

## Vulnerability 1 — Password Reset Token Disclosure

### Description

The Flowise password reset API returned the generated `tempToken` directly in the HTTP response body. This token is intended to be delivered exclusively via email to the account owner. Any unauthenticated attacker with knowledge of a valid email address could request a password reset and immediately use the token from the response — without access to the target's inbox.

### Classification

| Framework | Reference |
|-----------|-----------|
| CWE | CWE-200: Exposure of Sensitive Information to an Unauthorized Actor |
| CWE | CWE-640: Weak Password Recovery Mechanism for Forgotten Password |
| OWASP Top 10 | A07:2021 — Identification and Authentication Failures |
| MITRE ATT&CK | T1078 — Valid Accounts |

### Root Cause

The API response was designed to return the full user object after a password reset request — presumably for debugging or internal tooling purposes. The `tempToken` field was included in this object without being filtered out before serialization to JSON. The `x-request-from: internal` header was intended to gate access to this endpoint, but this is a client-controlled header with no cryptographic validation, making it trivially bypassable by any attacker who inspects browser traffic.

### Real-World Parallel

In 2020, a similar vulnerability was disclosed in multiple SaaS platforms where password reset tokens were inadvertently included in API responses, analytics beacons, or HTTP referrer headers. The OWASP API Security Top 10 (API3:2023 — Broken Object Property Level Authorization) specifically addresses scenarios where APIs return more fields than intended to the caller.

### Detection Opportunities

- **Anomalous reset volume:** Monitor for a high number of password reset requests originating from a single IP or within a short time window. Legitimate users rarely trigger resets repeatedly.
- **Token usage without email delivery:** If email delivery is logged, correlate reset token generation events with email send events. A token used without a corresponding email send is a strong indicator of exploitation.
- **Header anomaly detection:** Log and alert on requests containing `x-request-from: internal` originating from external IP addresses. This header should only be sent by the frontend server itself — never by external clients.

### Defense Recommendations

The token should never appear in the HTTP response body under any circumstances. The only communication channel for the reset token should be the registered email address. Additionally, `x-request-from: internal` must not be treated as a trust signal — if internal routing requires differentiation, it should be enforced at the network layer (e.g., internal-only routes on a private interface) rather than through a client-supplied header. Rate limiting on password reset endpoints is a baseline control that should be present regardless.

---

## Vulnerability 2 — CVE-2025-59528: Flowise CustomMCP Code Injection (RCE)

### Description

The `CustomMCP` node in Flowise 3.0.5 allows users to define configuration for external Model Context Protocol (MCP) servers. The configuration string is processed by a `convertToValidJSONString` function that passes user-supplied input directly to JavaScript's `Function()` constructor. This is functionally equivalent to `eval()` — any JavaScript expression in the input is executed with full Node.js runtime privileges. Attackers with a valid API key can use this to execute arbitrary OS commands on the server.

### Classification

| Framework | Reference |
|-----------|-----------|
| CWE | CWE-94: Improper Control of Generation of Code (Code Injection) |
| CWE | CWE-78: Improper Neutralization of Special Elements in OS Commands |
| OWASP Top 10 | A03:2021 — Injection |
| OWASP API Security | API8:2023 — Security Misconfiguration |
| MITRE ATT&CK | T1190 — Exploit Public-Facing Application |
| MITRE ATT&CK | T1059.007 — JavaScript (Command and Scripting Interpreter) |
| CVSS v3.1 | 10.0 (Critical) — AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H |

### Root Cause

The `Function()` constructor in JavaScript evaluates a string as code in the global scope. Using it to parse user-supplied configuration data — instead of a safe parser like `JSON.parse()` or `JSON5.parse()` — is a fundamental design error. The assumption that configuration input would always be valid JSON is incorrect in adversarial contexts. The patch (version 3.0.6) replaced the `Function()` call with `JSON5.parse()`, treating the input as data rather than executable code.

### Real-World Parallel

This class of vulnerability has precedent in several high-profile incidents. The 2021 Log4Shell vulnerability (CVE-2021-44228) involved similar logic: a library designed to parse and format log messages evaluated user-controlled strings as JNDI lookups, leading to RCE. The pattern — treating user data as code — recurs across ecosystems and is the root cause of template injection, SSTI, and eval-based injection vulnerabilities.

The active exploitation of CVE-2025-59528 in the wild (observed beginning April 2026 by VulnCheck) against over 12,000 exposed instances underscores how rapidly AI platform vulnerabilities are weaponized once disclosed.

### Detection Opportunities

- **Process spawning from Node.js:** Monitor for `node` or `flowise` processes spawning unexpected child processes — particularly shells (`sh`, `bash`) or network utilities (`nc`, `curl`, `wget`). EDR solutions with process tree analysis are effective here.
- **Outbound network connections from the Flowise process:** Flowise should not initiate outbound TCP connections outside of its configured integrations. Connections to arbitrary external IPs (especially on ports like 4444, 1337, 9001) are strong indicators of reverse shell activity.
- **API request content anomalies:** WAF rules inspecting the `mcpServerConfig` parameter for JavaScript execution patterns (`Function(`, `require(`, `process.mainModule`, `child_process`) can detect and block exploitation attempts at the perimeter.
- **File system writes in unexpected locations:** Any file write by the Flowise process outside its data directory (`/root/.flowise` or equivalent) should trigger an alert.

### Defense Recommendations

Upgrade to Flowise 3.0.6 or later is the primary remediation. Beyond patching, Flowise instances should not be publicly accessible — they should sit behind a reverse proxy with authentication requirements enforced at the network boundary. In containerized deployments, Flowise should run as a non-root user: the default behavior of running as root amplifies the impact of any RCE. Network segmentation between the Flowise container and other internal services (databases, Git servers, SSH) would limit lateral movement even after a container compromise.

---

## Vulnerability 3 — Credential Exposure via Container Environment Variables

### Description

The Flowise Docker container was configured with sensitive credentials passed as environment variables: `FLOWISE_PASSWORD`, `SMTP_PASSWORD`, and JWT secrets. These were accessible to any process running inside the container — including an attacker who obtained a shell via CVE-2025-59528. The `SMTP_PASSWORD` (`r04D!!_R4ge`) was reused as the system account password for the `ben` user on the host, enabling SSH access and lateral movement from the container to the host.

### Classification

| Framework | Reference |
|-----------|-----------|
| CWE | CWE-522: Insufficiently Protected Credentials |
| CWE | CWE-256: Plaintext Storage of a Password |
| OWASP Top 10 | A02:2021 — Cryptographic Failures |
| OWASP API Security | API8:2023 — Security Misconfiguration |
| MITRE ATT&CK | T1552.007 — Container API (Credentials from Container) |
| MITRE ATT&CK | T1078.003 — Local Accounts (Valid Accounts) |

### Root Cause

Two compounding issues exist here. First, sensitive credentials were stored in plaintext environment variables — a practice that is convenient but increases exposure surface since environment variables are readable by all processes in the container and may be logged by orchestration platforms. Second, the SMTP password was reused as an OS-level credential — violating the principle of credential uniqueness across services.

### Real-World Parallel

Credential exposure via environment variables is among the most common findings in cloud and container security assessments. The 2019 Capital One breach involved an SSRF vulnerability used to query EC2 instance metadata, which returned IAM credentials stored in the environment. Container environments — Docker, Kubernetes, ECS — frequently expose secrets this way when teams prioritize deployment simplicity over security posture. The 2022 Codecov supply chain attack also involved extraction of environment variables containing CI/CD secrets from compromised build scripts.

### Detection Opportunities

- **Unusual `env` or `/proc/self/environ` access:** Monitor for processes reading environment files outside of initialization. While `env` execution is common, execution from a shell spawned by a web service is anomalous.
- **SSH login from unexpected sources:** Alert on SSH authentication for `ben` (or any service account) from the container subnet (`172.18.0.0/16`). Service accounts should not authenticate interactively from container IP ranges.
- **Credential reuse detection:** Implement monitoring for the same credential being used across multiple services. Enterprise IAM and SIEM solutions can correlate authentication events by credential hash or username.

### Defense Recommendations

Secrets management solutions (HashiCorp Vault, AWS Secrets Manager, Kubernetes Secrets with external backends) should be used instead of plaintext environment variables. In Docker environments, Docker Secrets (Swarm) or mounted secret volumes provide better isolation. Each service should have a unique credential — SMTP, application, and OS credentials must never share passwords. Container network policies should prevent the Flowise container from initiating outbound connections to the host SSH port (`172.18.0.1:22`), which would have broken the lateral movement path even with valid credentials.

---

## Vulnerability 4 — CVE-2025-8110: Gogs Symlink Path Traversal (Privilege Escalation)

### Description

Gogs versions ≤ 0.13.3 allow symbolic links to be committed to Git repositories. The `PutContents` API endpoint, which writes file content to repository paths, does not validate whether the target path resolves through a symlink to a location outside the repository directory. An authenticated attacker can commit a symlink pointing to any system file (e.g., `/root/.ssh/authorized_keys`), then use the API to write attacker-controlled content to that file. Since Gogs ran as root in this deployment, this enabled overwriting the root user's SSH authorized keys and gaining full system access.

### Classification

| Framework | Reference |
|-----------|-----------|
| CWE | CWE-22: Improper Limitation of a Pathname to a Restricted Directory (Path Traversal) |
| CWE | CWE-61: UNIX Symbolic Link Following |
| CWE | CWE-732: Incorrect Permission Assignment for Critical Resource |
| OWASP Top 10 | A01:2021 — Broken Access Control |
| MITRE ATT&CK | T1574 — Hijack Execution Flow |
| MITRE ATT&CK | T1098.004 — Account Manipulation: SSH Authorized Keys |
| MITRE ATT&CK | T1548 — Abuse Elevation Control Mechanism |
| CVSS v4.0 | 8.7 (High) |

### Root Cause

This vulnerability is a bypass of the fix for CVE-2024-55947, which addressed path traversal via `../` sequences in file paths. The maintainers added validation to reject paths containing directory traversal patterns but did not account for symbolic links — which are a valid Git feature and follow a completely different code path. The `IsFile()` function used `os.Stat()` to check whether a path referred to a regular file, but `os.Stat()` follows symlinks, so a symlink appears as a regular file. The fix (version 0.13.4) replaced this with `os.Lstat()`, which does not follow symlinks, and added an explicit check rejecting any path that resolves through a symlink.

This is an instance of a broader class of vulnerabilities where security fixes address the symptom (specific attack string) rather than the root cause (insufficient trust boundary enforcement), leaving alternative bypass paths open.

### Real-World Parallel

Symlink attacks have a long history in Unix security. The `/tmp` race condition family of attacks (TOCTOU — Time of Check to Time of Use) has been exploited in privilege escalation contexts for decades. More recently, the Zip Slip vulnerability (2018) affected multiple archive extraction libraries that failed to validate symlinks, enabling path traversal during extraction. CVE-2025-8110 fits this same pattern — a component that processes file content without validating the full resolution path of the target.

The active exploitation of this vulnerability in the wild (discovered by Wiz Research in July 2025) against over 700 public Gogs instances demonstrates that self-hosted Git services are increasingly targeted as a lateral movement and persistence vector, particularly in organizations where they run with elevated privileges.

### Detection Opportunities

- **Symlink commits in repositories:** Monitor Git repository activity for commits containing symbolic links (`git log --diff-filter=A -- '*'` combined with `git cat-file` to check blob types). Symlinks in repositories are uncommon in legitimate workflows and should trigger review.
- **PutContents API calls on symlink paths:** Log all API calls to the `PutContents` endpoint. Calls targeting filenames that are symbolic links to paths outside the repository root are strong exploitation indicators.
- **Unexpected writes to sensitive system files:** File integrity monitoring (FIM) on critical files — `/root/.ssh/authorized_keys`, `/etc/passwd`, `/etc/crontab`, `/etc/sudoers` — should alert immediately on unexpected modifications. These files change rarely in production and any write outside of a change management window is suspicious.
- **SSH authentication with unexpected keys:** Monitor for new SSH public keys appearing in `authorized_keys` files. Key-based SSH authentication events using keys not provisioned through your IAM or configuration management tooling should alert the security team.

### Defense Recommendations

Upgrade to Gogs 0.13.4 or later. Beyond patching, Gogs — and any self-hosted Git service — should never run as root. A dedicated low-privilege service account (e.g., `git`) should own the process and data directories. If root-level access to repository storage is required, it should be achieved through a dedicated privileged helper with a narrow scope, not by running the entire application as root. Open registration should be disabled in internal deployments — `DISABLE_REGISTRATION = true` in `app.ini`. Network access to Gogs should be restricted to authorized internal IP ranges; it should never be accessible from container subnets or external networks without VPN. Repository creation permissions should also be restricted to trusted accounts.

---

## Defense-in-Depth Kill Chain Mapping

The table below maps each step of the Silentium attack chain to the defensive controls that would have interrupted it.

| Attack Step | Technique | Defensive Control | Kill Chain Impact |
|-------------|-----------|-------------------|-------------------|
| Vhost discovery of `staging.silentium.htb` | Subdomain enumeration | Restrict DNS zone transfers; remove staging from public DNS; use separate infrastructure for staging | Delays reconnaissance — attacker must guess or brute force |
| Password reset token disclosed in API response | Info Disclosure (CWE-200) | Never include `tempToken` in API response; enforce email-only delivery; validate `x-request-from` at network layer | **Breaks the chain** — attacker cannot authenticate |
| API key obtained from Flowise UI | Credential exposure | Scope API keys to specific operations; implement key rotation policies; alert on first-time key use from new IPs | Limits blast radius of credential exposure |
| CVE-2025-59528 — CustomMCP RCE | Code injection (CWE-94) | Patch to Flowise 3.0.6; WAF rules blocking `Function(`, `require(`, `child_process` in API params; non-root container | **Breaks the chain** — no code execution; if bypassed, root impact reduced |
| Reverse shell via nc | Command execution (T1059) | Block outbound connections from container except to configured integrations; alert on unexpected child process spawning | **Breaks the chain** — shell cannot connect back to attacker |
| Container env var enumeration | Credential harvesting (T1552.007) | Use secrets management (Vault, AWS Secrets Manager); unique passwords per service; no credentials in env vars | Attacker cannot harvest reusable credentials |
| SSH from container to host | Lateral movement (T1021.004) | Network policy blocking container → host SSH; firewall rule `172.18.0.0/16 → 172.18.0.1:22 DENY` | **Breaks the chain** — lateral movement blocked |
| Gogs open registration | Unauthorized account creation | Disable open registration (`DISABLE_REGISTRATION = true`); require admin approval | Attacker cannot create account needed for CVE-2025-8110 |
| CVE-2025-8110 — symlink traversal | Path traversal (CWE-22) | Patch to Gogs 0.13.4; FIM on `authorized_keys`; run Gogs as non-root service account | **Breaks the chain** — symlink write blocked; if bypassed, root impact reduced |
| SSH as root via injected authorized key | Privilege escalation (T1098.004) | Alert on unauthorized key additions; restrict root SSH login (`PermitRootLogin no`); monitor for new key-based auth | Last line of defense — root SSH blocked even with key injection |

---

*Documented by: Aspri*
*Date: April 2026*
*Frameworks: OWASP Top 10 (2021), OWASP API Security Top 10 (2023), MITRE ATT&CK v14, CWE*
