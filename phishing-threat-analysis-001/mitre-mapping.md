# MITRE ATT&CK Mapping

> Techniques observed during the phishing campaign mapped to the [MITRE ATT&CK Enterprise framework](https://attack.mitre.org/).

---

## Technique Summary

| ID | Name | Tactic | Observed |
|---|---|---|---|
| [T1566](https://attack.mitre.org/techniques/T1566/) | Phishing | Initial Access | ✅ |
| [T1056.003](https://attack.mitre.org/techniques/T1056/003/) | Credential Harvesting via Web Forms | Credential Access | ✅ |
| [T1102.002](https://attack.mitre.org/techniques/T1102/002/) | Web Service — Bidirectional Communication | Command and Control | ✅ |
| [T1041](https://attack.mitre.org/techniques/T1041/) | Exfiltration Over C2 Channel | Exfiltration | ✅ |
| [T1583.006](https://attack.mitre.org/techniques/T1583/006/) | Acquire Infrastructure — Web Services | Resource Development | ✅ |

---

## T1566 — Phishing
**Tactic:** Initial Access

Adversaries send phishing messages to trick users into interacting with malicious content — typically a link leading to a fraudulent page.

**Evidence observed:**
- Deceptive domain mimicking a legitimate service
- Automatic redirection to a fake authentication page
- Direct credential request presented to the victim

---

## T1056.003 — Credential Harvesting via Web Forms
**Tactic:** Credential Access

Adversaries use web forms on phishing pages to capture credentials entered by the victim. Processing is handled client-side to avoid exposing backend infrastructure.

**Evidence observed:**
- Credential capture logic implemented in `excedata.js`
- Form with no `action` attribute — all processing done in JavaScript
- Credentials processed locally before exfiltration

---

## T1102.002 — Web Service Abuse
**Tactic:** Command and Control

Adversaries leverage legitimate external web services to avoid building and maintaining their own infrastructure.

**Evidence observed:**
- Discord webhook used as the exfiltration endpoint
- Data transmitted via HTTP POST to a trusted platform
- No attacker-controlled backend server required

---

## T1041 — Exfiltration Over Web Protocol
**Tactic:** Exfiltration

Stolen data is transmitted out of the victim environment using standard web protocols, making traffic harder to distinguish from legitimate activity.

**Evidence observed:**
- HTTPS used for all exfiltration traffic
- Data serialized as JSON before transmission
- Exfiltration blends with normal web traffic patterns

---

## T1583.006 — Acquire Infrastructure (Cloud Services)
**Tactic:** Resource Development

Adversaries acquire cloud infrastructure to host malicious content, taking advantage of the reputation and availability of legitimate providers.

**Evidence observed:**
- Phishing page hosted on Microsoft Azure Static Web Apps
- Azure's trusted domain reputation reduces filter effectiveness
- No self-managed server required

---

## Attack Tactic Chain

```
Resource Development → Initial Access → Credential Access → Exfiltration
       T1583.006           T1566          T1056.003        T1041 / T1102.002
```

This incident represents a complete, low-infrastructure phishing chain combining social engineering, client-side credential harvesting, and exfiltration via legitimate web services — a pattern increasingly common in targeted credential theft campaigns.
