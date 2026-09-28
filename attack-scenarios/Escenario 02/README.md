# Escenario 02 — Enumeración web (Reconnaissance)

## Contexto dentro del ciclo de ataque

Segundo paso del ciclo documentado en `attack-scenarios/mitre-mapping.md`. El escaneo de puertos del escenario 01 identificó el puerto 8080/tcp como Mutillidae II corriendo sobre Apache. Este escenario enumera esa aplicación web en detalle — rutas, parámetros, módulos y tecnología detrás — para identificar la superficie de ataque concreta que se explota a partir del escenario 04 (SQL Injection).

Se ejecuta desde Kali, contra el puerto 8080 del servidor, en la red LAN aislada del lab.

## Técnica MITRE ATT&CK asociada

- **Táctica:** Reconnaissance
- **Técnica:** Active Scanning: Vulnerability Scanning — [T1595.002](https://attack.mitre.org/techniques/T1595/002/)

## Qué vas a encontrar en este escenario

- **`ataque.md`** — punto de vista ofensivo: herramientas utilizadas (Nmap con scripts NSE, Gobuster, Nikto, WFuzz/FFuF), los comandos ejecutados contra el puerto 8080 y el resultado: rutas, módulos de Mutillidae II y puntos de entrada identificados como candidatos a explotación.
- **`respuesta.md`** — punto de vista defensivo: qué queda registrado en `access_log` de Apache y correlacionado por Wazuh, qué firmas HTTP de Emerging Threats dispara Suricata ante este tipo de tráfico, la información recopilada sobre el origen (IP, user-agent, patrón de peticiones), las limitaciones (ruido esperado de un fuzzing agresivo vs. señal real) y el estado de las mitigaciones relacionadas.

## Resumen rápido

| Aspecto            | Detalle                                                                     |
| ------------------ | --------------------------------------------------------------------------- |
| Objetivo           | Mutillidae II, puerto 8080/tcp                                              |
| Herramientas       | Nmap (NSE), Gobuster, Nikto, WFuzz/FFuF                                     |
| Detección esperada | Wazuh (`access_log` de Apache) + Suricata (firmas HTTP de Emerging Threats) |
| Alimenta a         | Escenario 04 (SQLi)                                                         |

## Capturas

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/videos/02/02-scenario_enum_web.png" width="600"> </p>
