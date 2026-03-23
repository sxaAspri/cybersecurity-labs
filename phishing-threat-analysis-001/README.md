# 🎣 Phishing Threat Analysis #001

![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat-square)
![Type](https://img.shields.io/badge/Type-Blue%20Team%20Analysis-blue?style=flat-square)
![Environment](https://img.shields.io/badge/Environment-Kali%20Linux-557C94?style=flat-square&logo=kalilinux&logoColor=white)
![Method](https://img.shields.io/badge/Analysis-Static%20%7C%20Passive-orange?style=flat-square)
![Framework](https://img.shields.io/badge/Framework-MITRE%20ATT%26CK-red?style=flat-square)

> Static analysis of a phishing campaign targeting institutional email accounts at Politécnico Grancolombiano. The campaign abused Microsoft Azure infrastructure and Discord webhooks for real-time credential exfiltration.

---

## 📋 Case Overview

| Field | Details |
|---|---|
| **Date** | March 2, 2026 |
| **Target** | Institutional email domain — Poligran |
| **Attack Type** | Credential Harvesting Phishing |
| **Infrastructure** | Webcindario (redirect) → Microsoft Azure (phishing page) |
| **Exfiltration Channel** | Discord Webhook (HTTP POST) |
| **Analysis Environment** | Kali Linux (isolated, controlled) |
| **Approach** | Passive — no active interaction with attacker infrastructure |

---

## 🎯 Objectives

- Identify attacker infrastructure and redirection chain
- Analyze the JavaScript-based credential harvesting mechanism
- Determine the data exfiltration method
- Extract Indicators of Compromise (IOCs)
- Map techniques to the MITRE ATT&CK framework
- Preserve digital evidence with cryptographic integrity

---

## 🔗 Attack Chain Summary

```
Phishing Email
      ↓
ingresoinmediato[.]webcindario[.]com        ← Redirect layer
      ↓  (meta http-equiv="Refresh")
actualizardatosss[.]z13[.]web[.]core[.]windows[.]net   ← Azure-hosted phishing page
      ↓  (excedata.js)
Credential Harvesting (email, password, phone, SMS, PIN)
      ↓  (HTTP POST)
Discord Webhook                             ← Real-time exfiltration
```

---

## 📁 Repository Structure

```
phishing-threat-analysis-001/
│
├── README.md                       ← You are here
├── attack-flow.md                  ← Full attack chain breakdown (6 stages)
├── iocs.md                         ← Indicators of Compromise (domains, hashes)
├── mitre-mapping.md                ← MITRE ATT&CK technique mapping
└── defensive-recommendations.md   ← Defensive controls and mitigations
```

---

## 🧠 Key Technical Findings

**1. Two-layer infrastructure**
The attacker used a free hosting service (Webcindario) purely as a redirect, concealing the real phishing page hosted on Microsoft Azure — a legitimate cloud provider trusted by most security filters.

**2. No server-side backend**
The phishing kit required zero infrastructure maintenance. All credential processing happened client-side via JavaScript (`excedata.js`), with data sent directly to a Discord webhook.

**3. Multi-factor bypass intent**
Beyond email and password, the form captured SMS codes, PINs, and additional verification codes — indicating a full Account Takeover (ATO) objective, not just basic credential theft.

**4. Victim fingerprinting**
The script queried `api.ipify.org` and `ipapi.co` to attach the victim's public IP and geolocation to every stolen credential submission.

---

## 🗂️ MITRE ATT&CK Coverage

| Technique ID | Name | Tactic |
|---|---|---|
| T1566 | Phishing | Initial Access |
| T1056.003 | Credential Harvesting via Web Forms | Credential Access |
| T1102.002 | Web Service — Exfiltration | Exfiltration |
| T1041 | Exfiltration Over Web Protocol | Exfiltration |
| T1583.006 | Acquire Infrastructure — Cloud Services | Resource Development |

→ Full mapping in [`mitre-mapping.md`](./mitre-mapping.md)

---

## ⚠️ Disclaimer

This repository is for **educational and defensive cybersecurity research purposes only**.  
All sensitive information including webhook URLs, active domains, and tokens has been redacted.  
No active interaction was performed against attacker-controlled infrastructure.
