# Indicators of Compromise (IOCs)

> All indicators extracted through passive static analysis. No active probing was performed against attacker infrastructure.

---

## 🌐 Malicious Domains

| Indicator | Type | Role | Status |
|---|---|---|---|
| `ingresoinmediato[.]webcindario[.]com` | Domain | Redirect layer | Defanged |
| `actualizardatosss[.]z13[.]web[.]core[.]windows[.]net` | Domain | Phishing page (Azure) | Defanged |

> **Note:** Brackets `[.]` are used to defang indicators and prevent accidental resolution.

---

## 🏗️ Infrastructure

| Layer | Provider | Purpose |
|---|---|---|
| Layer 1 — Redirect | Webcindario (free hosting) | Hide final phishing infrastructure |
| Layer 2 — Phishing page | Microsoft Azure Static Web Apps | Host credential harvesting page |
| Exfiltration | Discord Webhook API | Receive stolen credentials in real time |

---

## 📄 Malicious File

| Field | Value |
|---|---|
| **Filename** | `excedata.js` |
| **Type** | JavaScript |
| **Role** | Credential harvesting + exfiltration logic |
| **Hash (SHA256)** | `b641ac4ee70eda78f867a685b28491b1a97dbf69d017d08f5009cd2ce4c4711c` |

The SHA256 hash was generated from a locally downloaded copy of the file to preserve evidence integrity.

---

## 📡 External Services Abused

| Service | URL | Purpose |
|---|---|---|
| IPify | `api.ipify.org` | Retrieve victim's public IP address |
| ipapi | `ipapi.co` | Geolocate victim by IP |
| Discord Webhook | Redacted | Exfiltrate captured credentials |

---

## 🎯 Data Targeted

| Data Type | Captured |
|---|---|
| Email address | ✅ |
| Password | ✅ |
| Phone number | ✅ |
| SMS verification code | ✅ |
| PIN | ✅ |
| Additional verification codes | ✅ |
| Victim IP address | ✅ |
| Victim geolocation | ✅ |

---

> The Discord webhook URL has been **redacted** from this repository for security reasons.
