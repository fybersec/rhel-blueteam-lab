# Mapeo a MITRE ATT&CK

Este documento centraliza qué técnicas de [MITRE ATT&CK](https://attack.mitre.org/) se ejercitan en cada escenario de `attack-scenarios/`, y qué fuente de detección (Wazuh, Suricata, o ambos) debería generar alerta. No cubre la matriz completa — ver limitaciones en `docs/es/00-introduccion.md`.

## Resumen de escenarios

| #   | Escenario                                          | Táctica(s)                  | Técnica(s)                                                             | ID                                                                                                        | Fuente de detección esperada                                                |
| --- | -------------------------------------------------- | --------------------------- | ---------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| 01  | [Escaneo de puertos](01-escaneo-puertos/README.md) | Reconnaissance              | Active Scanning: Vulnerability Scanning / Scanning IP Blocks           | [T1595](https://attack.mitre.org/techniques/T1595/)                                                       | Suricata (reglas de scan de Emerging Threats)                               |
| 02  | [Enumeración web](03-enumeracion-web/README.md)    | Reconnaissance              | Active Scanning: Vulnerability Scanning                                | [T1595.002](https://attack.mitre.org/techniques/T1595/002/)                                               | Wazuh (reglas de autenticación fallida SSH, `sshd`)                         |
| 03  | [Fuerza bruta SSH](02-fuerza-bruta-ssh/README.md)  | Credential Access           | Brute Force: Password Guessing                                         | [T1110.001](https://attack.mitre.org/techniques/T1110/001/)                                               | Wazuh (`access_log` de Apache) + Suricata (firmas HTTP de Emerging Threats) |
| 04  | [SQL Injection](04-sql-injection/README.md)        | Initial Access              | Exploit Public-Facing Application                                      | [T1190](https://attack.mitre.org/techniques/T1190/)                                                       | Suricata (si el payload va por GET/URL) + Wazuh (`access_log`)              |

## Criterio de selección

Los escenarios se eligieron para cubrir, con recursos acotados a este lab (una sola VM RHEL como objetivo, Kali como atacante), un ciclo representativo de un ataque real de principio a fin:

1. **Reconocimiento externo** (escaneo de puertos con Nmap) — identifica qué servicios están expuestos, incluyendo SSH y el puerto 8080 de Mutillidae II.
2. **Acceso por fuerza bruta** contra el servicio administrativo expuesto (SSH).
3. **Enumeración web** dirigida específicamente contra Mutillidae II — descubrimiento de rutas, parámetros y módulos vulnerables usando Nmap (scripts NSE), Gobuster, Nikto y WFuzz/FFuF.
4. **Explotación de la aplicación web** — una vez identificada la superficie de ataque en el paso anterior, se ejecutan los escenarios de SQL Injection  directamente sobre Mutillidae II.

## Nota sobre visibilidad de Suricata en ataques web

Un criterio transversal a los escenarios 04-07 es que las peticiones por método **POST** no suelen ser fácilmente detectables por un NIDS basado en firmas como Suricata, ya que el cuerpo de la petición no queda expuesto en la URL. Por eso, donde la vulnerabilidad de Mutillidae II lo permite, se prioriza ejecutar el vector de ataque por **GET** (parámetro visible en la URL), lo que sí genera coincidencias de firma en Suricata además de quedar registrado en `access_log` para Wazuh. Cuando el escenario solo es explotable por POST, la detección recae principalmente en Wazuh (correlación de `access_log`, integridad de archivos o procesos, según el caso) y esto se documenta explícitamente en el `README.md` de ese escenario como una limitación conocida, no como un fallo de configuración.

## Cómo leer cada escenario

Cada carpeta numerada contiene:

- **`README.md`** con el objetivo de la técnica dentro del ciclo de ataque, la herramienta y el comando exacto ejecutado desde Kali, y la técnica MITRE asociada.
- **`ataque.md`** (o el nombre de archivo equivalente dentro de la carpeta) con el detalle del payload/comando ejecutado y su contexto.
- **`respuesta.md`** con el comportamiento observado en el dashboard de Wazuh ante ese ataque específico: qué alerta se generó (o no), qué información recopiló el sistema sobre el origen (IP, user-agent, patrón de la petición) y cualquier dato que ayude a perfilar al posible atacante. Ningún escenario bloquea el tráfico — el objetivo es observación y correlación, no contención.