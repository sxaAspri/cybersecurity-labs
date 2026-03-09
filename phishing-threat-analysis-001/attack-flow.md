\# Attack Chain Reconstruction



\## Stage 1 – Phishing Delivery



The victim receives a phishing email containing a link that appears to direct to a legitimate service.



The link points to an intermediary domain designed to initiate the attack chain.





\## Stage 2 – Redirection



The initial webpage contains an HTML meta refresh tag that automatically redirects the user to another domain.



Example mechanism:



<meta http-equiv="refresh" content="0; URL=..." />



This redirection technique is commonly used to obscure the final phishing infrastructure.





\## Stage 3 – Phishing Infrastructure



The final phishing page was hosted on public cloud infrastructure.



Hosting Provider:

Microsoft Azure



Attackers frequently abuse legitimate cloud providers to increase credibility and evade automated blocking mechanisms.





\## Stage 4 – Credential Harvesting



A malicious JavaScript file (`excedata.js`) captures user input from the phishing form.



Captured data includes:



\- Email address

\- Password

\- Phone number

\- SMS verification code

\- PIN

\- Additional verification codes





\## Stage 5 – Victim Information Collection



The script performs external API requests to gather additional victim information.



Services used:



\- api.ipify.org (public IP detection)

\- ipapi.co (IP geolocation)





\## Stage 6 – Data Exfiltration



Captured credentials are transmitted via HTTPS to a Discord webhook controlled by the attacker.



Characteristics of this method:



\- No attacker-controlled server required

\- Real-time credential delivery

\- Abuse of legitimate communication platform





---



\## Simplified Attack Flow



Victim  

↓  

Phishing Email  

↓  

Redirect Domain  

↓  

Azure Hosted Phishing Page  

↓  

Credential Harvesting Script  

↓  

Discord Webhook

