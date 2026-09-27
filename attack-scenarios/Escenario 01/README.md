# Escenario 01 — Escaneo de puertos (Reconnaissance)

## Contexto dentro del ciclo de ataque

Primer paso del ciclo documentado en `attack-scenarios/mitre-mapping.md`: antes de atacar cualquier servicio específico, hay que identificar qué está expuesto en `rhel-blueteam-lab`. Se ejecuta desde Linux Mint, contra la IP del servidor en la red LAN aislada del lab (sin salida a internet, según lo definido en `00-introduccion.md`). 

A partir de lo que revela este escaneo se decide el siguiente vector de ataque del ciclo: SSH (→ escenario 02) y el puerto 8080 de Mutillidae II (→ escenario 03 en adelante).

## Técnica MITRE ATT&CK asociada

- **Táctica:** Reconnaissance
- **Técnica:** Active Scanning: Vulnerability Scanning / Scanning IP Blocks — [T1595](https://attack.mitre.org/techniques/T1595/)

## Qué vas a encontrar en este escenario

- **`ataque.md`** — punto de vista ofensivo: herramienta (Nmap), los tres comandos ejecutados (SYN scan completo, detección de versión/servicios sobre 22 y 8080, escaneo UDP acotado) y el resultado observado (puertos abiertos y servicios detrás de cada uno).
- **`respuesta.md`** — punto de vista defensivo: qué detectó Suricata (reglas `ET SCAN` de Emerging Threats) y cómo llega esa alerta al dashboard de Wazuh, qué información se recopiló sobre el origen del escaneo, las limitaciones de visibilidad (por qué un SYN scan no deja rastro en logs de aplicación) y el estado de las mitigaciones relacionadas (hardening de superficie, firewalld, rate limiting pendiente).

## Resumen rápido

|Puerto|Estado|Servicio|Detección esperada|
|---|---|---|---|
|22/tcp|open|ssh (OpenSSH)|Suricata (`ET SCAN Potential SSH Scan`)|
|8080/tcp|open|http (Apache/httpd)|Suricata (umbral de conexiones SYN por IP)|
|443/tcp|open|https (Wazuh dashboard)|Fuera de alcance de ataque — infraestructura de detección|

## Capturas

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/videos/01/Wazuh.png" width="600"> </p>