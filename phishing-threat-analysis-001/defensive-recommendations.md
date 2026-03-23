# Defensive Recommendations

> Mitigations derived directly from the techniques observed in this phishing campaign. Each recommendation maps to a specific attack behavior identified during analysis.

---

## Control Matrix

| Control | Addresses | Priority |
|---|---|---|
| Email filtering + URL sandboxing | T1566 — Phishing delivery | 🔴 High |
| MFA hardening | T1056.003 — Credential harvesting | 🔴 High |
| Webhook abuse monitoring | T1102.002 — Discord exfiltration | 🔴 High |
| DNS filtering | T1583.006 — Azure-hosted page | 🟠 Medium |
| Security awareness training | T1566 — Social engineering | 🟠 Medium |
| Domain monitoring | T1583.006 — Infrastructure acquisition | 🟡 Low |
| SIEM detection rules | T1041 — Exfiltration over HTTPS | 🟡 Low |

---

## 1. Email Filtering and URL Sandboxing
**Addresses:** T1566

This campaign relied on a phishing email as the initial delivery vector. The malicious link pointed to a free hosting domain (`webcindario`) before redirecting to Azure infrastructure.

**Recommended controls:**
- Deploy anti-phishing email gateways with link rewriting and detonation
- Enforce DMARC, DKIM, and SPF on institutional email domains
- Enable URL sandboxing to evaluate redirect chains before delivery
- Flag emails containing links to free hosting services (Webcindario, etc.)

---

## 2. MFA Hardening Against Real-Time Phishing
**Addresses:** T1056.003

The phishing kit captured not only passwords but also SMS codes, PINs, and additional verification codes — a clear attempt to bypass standard MFA in real time.

**Recommended controls:**
- Replace SMS-based MFA with phishing-resistant methods (FIDO2 / hardware security keys)
- Implement number matching or additional context in authenticator apps
- Configure conditional access policies to detect impossible travel or unusual sign-in locations
- Monitor for authentication attempts immediately following email link clicks

> Standard TOTP and SMS codes are vulnerable to real-time adversary-in-the-middle (AiTM) relay — this campaign's design suggests exactly that intent.

---

## 3. Block and Monitor Webhook Abuse
**Addresses:** T1102.002, T1041

All stolen data was exfiltrated via a Discord webhook — no attacker-controlled server was required. Outbound HTTPS traffic to `discord.com/api/webhooks/` from corporate networks is rarely legitimate.

**Recommended controls:**
- Block outbound connections to `discord.com/api/webhooks/` at the network perimeter
- Monitor and alert on HTTP POST requests to known webhook endpoints (Discord, Slack, Telegram)
- Apply egress filtering to restrict outbound HTTPS to approved destinations
- Add threat intelligence feeds that track webhook-based exfiltration patterns

---

## 4. DNS Filtering and Cloud Domain Inspection
**Addresses:** T1583.006

The phishing page was hosted on `*.z13.web.core.windows.net` — a legitimate Azure Static Web Apps domain. Standard reputation-based filtering may not flag this as malicious.

**Recommended controls:**
- Deploy DNS filtering solutions capable of evaluating subdomain patterns, not just root domains
- Flag newly registered or anomalous subdomains on cloud providers (Azure, AWS, GCP)
- Consider blocking access to Azure Static Web Apps subdomains that are not business-relevant
- Integrate threat intelligence feeds with recent cloud-hosted phishing IOCs

---

## 5. Security Awareness Training
**Addresses:** T1566

This attack succeeded at the delivery stage through social engineering. The email prompted users to act quickly and input institutional credentials.

**Key training topics based on this campaign:**
- Recognizing redirect chains — the first link is rarely the final destination
- Verifying the legitimacy of login pages before entering credentials
- Understanding that cloud-hosted pages (Azure, AWS) are not inherently safe
- Reporting suspicious emails through official institutional channels

---

## 6. SIEM Detection Opportunities

The following behavioral patterns could be used to build detection rules relevant to this type of attack:

| Event | Detection Opportunity |
|---|---|
| Login attempt shortly after clicking an email link | Correlate email gateway + IdP logs |
| Authentication from unexpected geolocation | Alert on sign-in location anomaly |
| Multiple failed MFA attempts | Possible AiTM relay attempt |
| Outbound POST to `discord.com/api/webhooks` | Potential webhook-based exfiltration |
| Access to `api.ipify.org` or `ipapi.co` from browser | Victim fingerprinting behavior |

---

## Summary

This campaign illustrates how a low-cost, low-infrastructure phishing kit can effectively target institutional credentials using entirely legitimate services (Azure, Discord, ipify). Defensive controls must account for the abuse of trusted platforms — traditional reputation-based filtering is insufficient on its own.

A layered approach combining **email security, phishing-resistant MFA, network egress controls, and user awareness** is the minimum viable defense against this attack pattern.
