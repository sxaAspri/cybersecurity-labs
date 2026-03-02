# Detailed Audit Report – Lynis Security Assessment

## 1. Baseline Security Assessment

An initial security audit was performed using Lynis to evaluate the system’s hardening level and identify potential vulnerabilities.

- Hardening Index (Initial): 60
- System State: Default installation with minimal hardening
- Exposure Level: Moderate to High (depending on network environment)

The assessment revealed multiple configuration weaknesses and outdated components that increase the system's attack surface.

---

## 2. Critical and High Risk Findings

### 2.1 Vulnerable Packages Detected (PKGS-7392) – Critical

**Risk Level:** High  
**Impact:**  
Presence of installed packages with known vulnerabilities (potential CVEs), which may allow:

- Remote Code Execution (RCE)
- Privilege Escalation
- Full system compromise

Unpatched systems represent one of the most common entry points for attackers.

---

### 2.2 Firewall Without Active Filtering Rules (FIRE-4512) – Critical

**Risk Level:** High  
**Impact:**  
Although the firewall module was active, no filtering rules were configured.

This resulted in:

- Acceptance of all inbound traffic
- Increased network exposure
- Elevated attack surface in networked environments

This configuration violates the principle of least privilege.

---

### 2.3 Telnet Service Installed – High

**Risk Level:** Medium-High  
**Impact:**  
Telnet transmits credentials in plaintext, making it susceptible to interception and credential harvesting attacks within untrusted networks.

Legacy protocols significantly weaken security posture.

---

### 2.4 Weak SSH Configuration – High

Identified weaknesses included:

- High `MaxAuthTries`
- X11 forwarding enabled
- TCP forwarding enabled
- Default port usage
- Low log verbosity

**Impact:**

- Increased brute-force exposure
- Potential unauthorized tunneling
- Reduced visibility of malicious login attempts

SSH is a critical service and must be hardened accordingly.

---

## 3. Medium Risk Findings

### 3.1 Weak Password Policies

Issues identified:

- No strict password aging configuration
- Weak password complexity enforcement
- Permissive default settings

**Impact:**

- Increased risk of credential compromise
- Reduced resistance against brute-force attacks

---

### 3.2 Non-Hardened Kernel Parameters

Kernel parameters such as:

- `accept_redirects`
- `send_redirects`
- `kernel.kptr_restrict`
- `kernel.sysrq`

were not optimized for secure environments.

**Impact:**

- Reduced protection against advanced exploitation techniques
- Higher exposure to network-based manipulation

---

## 4. Low Risk / Advanced Hardening Opportunities

The following items are considered security best practices but were not critical in the lab environment:

- No separate partitions for /tmp, /var, /home
- No malware scanning solution installed
- No file integrity monitoring system (AIDE)
- No GRUB bootloader password protection

These controls are more relevant in production or regulated environments.

---

## 5. Remediation Plan

The following mitigation measures were defined:

### Patch Management
- Update repositories
- Upgrade installed packages
- Enable automatic security updates (optional)

### Firewall Hardening
- Enable UFW
- Deny all incoming traffic by default
- Allow only required services (e.g., SSH)

### Service Reduction
- Remove Telnet service
- Disable unnecessary services

### SSH Hardening
- Reduce `MaxAuthTries`
- Disable X11Forwarding
- Disable AllowTcpForwarding
- Disable root login
- Increase log verbosity

### Authentication Hardening
- Configure password aging policies
- Enforce strong password complexity via PAM

### Kernel Hardening
- Apply secure sysctl parameters
- Disable unnecessary kernel capabilities

---

## 6. Expected Security Posture Improvement

After applying the proposed mitigation plan, the expected improvements include:

- Elimination of critical vulnerabilities
- Reduced network exposure
- Hardened authentication mechanisms
- Minimized attack surface
- Improved compliance with security best practices (e.g., CIS benchmarks)

Estimated Hardening Index Improvement: 75–85 range.

---

## 7. Residual Risk

Even after mitigation, some residual risks may remain:

- Zero-day vulnerabilities
- Insider threats
- Misconfiguration during future changes
- Lack of continuous monitoring

Security should be treated as an ongoing process rather than a one-time configuration effort.

---

## 8. Professional Reflection

This assessment demonstrates the importance of:

- Continuous vulnerability management
- Secure baseline configurations
- Principle of least privilege
- Defense-in-depth strategy

Default system installations are not inherently secure and require structured hardening to align with modern security standards.
