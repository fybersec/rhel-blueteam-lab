# Escenario 04 — SQL Injection (Initial Access)

## Contexto dentro del ciclo de ataque

Cuarto paso del ciclo documentado en `attack-scenarios/mitre-mapping.md`. La enumeración web del escenario 03 identificó el módulo de SQL Injection de Mutillidae II como accesible por **GET**, lo que se priorizó deliberadamente para mantener visibilidad tanto en `access_log` (Wazuh) como en el tráfico de red (Suricata) — según el criterio transversal documentado en `mitre-mapping.md` sobre POST vs. GET.

Este escenario explota esa inyección para extraer datos de la base de datos subyacente a Mutillidae II, ejecutado desde Kali contra el puerto 8080 del servidor.

## Técnica MITRE ATT&CK asociada

- **Táctica:** Initial Access
- **Técnica:** Exploit Public-Facing Application — [T1190](https://attack.mitre.org/techniques/T1190/)

## Qué vas a encontrar en este escenario

- **`ataque.md`** — punto de vista ofensivo: módulo de Mutillidae II explotado, el payload utilizado (manual y/o con SQLmap), el parámetro vulnerable y los datos extraídos de la base.
- **`respuesta.md`** — punto de vista defensivo: qué firma de Suricata dispara el payload al viajar por URL, qué queda registrado en `access_log` y correlacionado por Wazuh, la información recopilada sobre el origen, las limitaciones (dependencia de que el vector sea GET) y el estado de las mitigaciones relacionadas (sanitización de entradas, WAF, principio de mínimo privilegio en la cuenta de base de datos).

## Resumen rápido

| Aspecto              | Detalle                                                              |
| --------------------- | ------------------------------------------------------------------------ |
| Objetivo               | Módulo de SQL Injection de Mutillidae II, puerto 8080/tcp                |
| Vector                 | GET (parámetro visible en la URL, priorizado según `mitre-mapping.md`)   |
| Herramientas           | Payloads manuales + SQLmap                                               |
| Detección esperada     | Suricata (payload en la URL) + Wazuh (`access_log`)                      |

## Capturas

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/videos/04/wazuh.png" width="600"> </p>
