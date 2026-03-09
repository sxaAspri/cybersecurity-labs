\# Phishing Threat Analysis #001



\## Overview



This case study documents the identification and analysis of a phishing email received in an institutional environment.



The objective of this project was to analyze the phishing infrastructure, identify the credential harvesting mechanism, and document the attack from a Blue Team defensive perspective.



This investigation was conducted in a controlled environment using Kali Linux and focuses strictly on passive analysis.



---



\## Objectives



\- Identify attacker infrastructure

\- Analyze credential harvesting mechanisms

\- Identify exfiltration techniques

\- Extract Indicators of Compromise (IOCs)

\- Map the attack to MITRE ATT\&CK techniques



---



\## Analysis Scope



The investigation included:



\- Domain analysis

\- Redirection identification

\- Static analysis of HTML and JavaScript code

\- Identification of data exfiltration mechanisms

\- Infrastructure attribution

\- IOC extraction



No active interaction with the attacker infrastructure was performed.



---



\## Attack Summary



The phishing campaign followed a multi-stage attack chain:



1\. Initial phishing domain

2\. Automatic redirection

3\. Phishing page hosted on cloud infrastructure

4\. Credential harvesting via JavaScript

5\. Real-time exfiltration through a Discord webhook



More technical details can be found in the following files:



\- attack-flow.md

\- iocs.md

\- mitre-mapping.md

\- defensive-recommendations.md



---



\## Disclaimer



This repository is for educational and defensive cybersecurity research purposes only.



All sensitive information, tokens, and active URLs have been redacted.

