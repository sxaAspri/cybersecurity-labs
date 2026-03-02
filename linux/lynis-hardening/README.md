# 9.1.14 Lab – Linux System Hardening (Terminal Security)
Overview
This lab focuses on reinforcing the security posture of a Linux system by performing a security audit and applying hardening measures based on identified vulnerabilities.
The activity is based on the Cisco lab:

9.1.14 – Lab: Reforzar un sistema Linux en seguridad de terminales
The audit was conducted using Lynis to evaluate the system’s baseline security configuration and identify areas for improvement.

Objective

Perform a security audit of a Linux system.
Identify vulnerabilities and weak configurations.
Classify findings by risk level.
Design and apply a mitigation plan.
Improve the overall system hardening index.

Environment
Platform: Linux (Debian-based distribution)

Audit Tool: Lynis

Audit Mode:sudo lynis audit system

Executive Summary of Findings

Critical
Vulnerable packages detected (PKGS-7392)
Firewall active but without filtering rules (FIRE-4512)

High
Telnet service installed
Weak SSH configuration

Medium
Weak password policies
Non-hardened kernel parameters

Low
No partition separation
No malware scanner
No file integrity monitoring (AIDE)
No GRUB password protection

A detailed technical breakdown is available in:
audit-summary.md

Mitigation Strategy

The mitigation plan focused on:
Updating vulnerable packages
Enforcing firewall rules using UFW
Removing insecure services (Telnet)
Hardening SSH configuration
Strengthening password policies (PAM)
Applying kernel hardening parameters (sysctl)


Key Security Takeaways
Default Linux installations require additional hardening.
Firewall configuration is critical even in lab environments.
Legacy services such as Telnet introduce unnecessary risk.
SSH misconfigurations significantly increase exposure to brute-force attacks.
Kernel-level hardening enhances resistance against advancd threats.
Security auditing should be continuous, not a one-time activity.

Conclusion
The system initially demonstrated a moderate security posture (Hardening Index 60), typical of a default installation without additional protections.

After applying the proposed mitigation plan:

Critical vulnerabilities are eliminated.
Network exposure is significantly reduced.
Core services are hardened.
Authentication mechanisms are strengthened.
Overall system resilience is improved.

This lab reinforces the importance of proactive system auditing and structured hardening processes aligned with industry best practices such as CIS benchmarks.








Implementing optional advanced security improvements
