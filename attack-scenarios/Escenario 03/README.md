# Escenario 03 — Fuerza bruta SSH (Credential Access)

## Contexto dentro del ciclo de ataque

Segundo paso del ciclo documentado en `attack-scenarios/mitre-mapping.md`. El escaneo de puertos del escenario 01 identificó el puerto 22/tcp (SSH) como el único punto de entrada administrativo expuesto en `rhel-blueteam-lab`. Este escenario intenta obtener credenciales válidas contra ese servicio mediante fuerza bruta, ejecutado desde Kali contra la IP del servidor en la red LAN aislada del lab.

De confirmarse acceso, este sería el vector de acceso inicial administrativo del ciclo; en caso contrario, el foco de explotación pasa a la superficie web (Mutillidae II), cubierta a partir del escenario 03.

## Técnica MITRE ATT&CK asociada

- **Táctica:** Credential Access
- **Técnica:** Brute Force: Password Guessing — [T1110.001](https://attack.mitre.org/techniques/T1110/001/)

## Qué vas a encontrar en este escenario

- **[`ataque.md`](Escenario%2003%20—%20Fuerza%20bruta%20SSH%20(Credential%20Access).md)** — punto de vista ofensivo: herramienta (Hydra), diccionario de usuarios/contraseñas utilizado, el comando ejecutado contra el puerto 22 y el resultado obtenido (credenciales válidas o no, y por qué).
- **[`respuesta.md`](Escenario%2003%20—%20Fuerza%20bruta%20SSH%20respuesta%20(Credential%20Access).md)** — punto de vista defensivo: qué detectó Wazuh a partir de los logs de `sshd` (intentos de autenticación fallida), la alerta representativa generada, la información recopilada sobre el origen del ataque, las limitaciones (ej. dependencia total de logs de aplicación, sin señal de Suricata por no ser tráfico con firma de red) y el estado de las mitigaciones relacionadas (fail2ban, límite de intentos, deshabilitar auth por password).

## Resumen rápido

| Aspecto              | Detalle                                                        |
| --------------------- | ---------------------------------------------------------------- |
| Puerto objetivo       | 22/tcp (SSH)                                                     |
| Herramienta            | Hydra                                                             |
| Detección esperada     | Wazuh (reglas de autenticación fallida SSH sobre `sshd`)         |
| Fuera de alcance       | Suricata — un intento de login SSH no deja firma de red distintiva |

## Capturas

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/videos/03/wazuh.png" width="600"> </p>
