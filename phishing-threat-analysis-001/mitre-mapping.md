\# Clasificación del Incidente según MITRE ATT\&CK



Se utilizó el framework \*\*MITRE ATT\&CK\*\* para mapear las técnicas observadas durante el análisis del incidente de phishing.



---



\## A. T1566 – Phishing (Initial Access)



\*\*Descripción\*\*



La técnica de phishing se utiliza para engañar a los usuarios y hacer que interactúen con contenido malicioso, generalmente a través de enlaces o páginas web fraudulentas.



\*\*Evidencia observada\*\*



\- Dominio engañoso utilizado para imitar un servicio legítimo.

\- Redirección automática hacia una página falsa de autenticación.

\- Solicitud directa de credenciales al usuario.



---



\## B. T1056.003 – Credential Harvesting (Web Forms)



\*\*Descripción\*\*



Los atacantes capturan credenciales mediante formularios web falsos que imitan páginas de autenticación legítimas.



\*\*Evidencia observada\*\*



\- Funciones de captura implementadas en el archivo `excedata.js`.

\- Procesamiento local de la información ingresada por el usuario.

\- Envío posterior de las credenciales a un webhook controlado por el atacante.



---



\## C. T1102.002 – Web Service Exfiltration



\*\*Descripción\*\*



Uso de servicios web legítimos para exfiltrar información robada.



\*\*Evidencia observada\*\*



\- Uso de un \*\*webhook de Discord\*\* como canal de exfiltración.

\- Envío de información mediante solicitudes \*\*HTTP POST\*\*.



---



\## D. T1041 – Exfiltration Over Web Protocol



\*\*Descripción\*\*



Los datos robados se envían fuera del sistema comprometido mediante protocolos web estándar.



\*\*Evidencia observada\*\*



\- Uso de protocolo \*\*HTTPS\*\* para transmitir la información.

\- Serialización de los datos en formato \*\*JSON\*\* antes del envío.



---



\## E. T1583.006 – Acquire Infrastructure (Cloud Services)



\*\*Descripción\*\*



Los atacantes adquieren infraestructura en la nube para alojar contenido malicioso o infraestructura de ataque.



\*\*Evidencia observada\*\*



\- Hosting del contenido malicioso en \*\*Microsoft Azure\*\*.



---



\## Conclusión



El análisis del incidente muestra un flujo de ataque típico de \*\*phishing con recolección de credenciales\*\*, seguido de \*\*exfiltración mediante servicios web legítimos\*\*.



El uso del framework \*\*MITRE ATT\&CK\*\* permite comprender mejor las tácticas y técnicas empleadas por el atacante, facilitando su detección y mitigación dentro de entornos de seguridad defensiva.

