# RHEL Blue Team Lab
Laboratorio personal de detección de intrusos construido sobre **Red Hat Enterprise Linux 8.10**. El objetivo es levantar un entorno controlado donde practicar hardening manual, desplegar un stack de detección (Wazuh + Suricata) y ejecutar escenarios de ataque simulados contra un servicio web vulnerable para verificar qué se detecta y qué no.

No es una guía genérica de "cómo instalar Wazuh": es la documentación de decisiones específicas tomadas en este entorno, incluyendo los errores encontrados y cómo se resolvieron. Tampoco es un entorno expuesto a internet ni pensado para producción.

<br clear="left"/>
## Arquitectura

| Rol                  | SO / Firmware | RAM         | Almacenamiento |
| -------------------- | ------------- | ----------- | -------------- |
| Servidor             | Red Hat 8.10  | 4 GB        | 437.4 GiB      |
| Administrador remoto | Windows 11    | 16 GB       | 238 GiB        |
| Pentesting           | Linux Mint    | 4 GB        | 60 GiB         |
| Router               | GL-SFT1200    | 128 MB DDR3 | 128 MB         |

El laboratorio corre sobre una única VM RHEL 8.10. Dentro de ella conviven el stack de detección (Wazuh + Suricata) y el servicio web objetivo, todos aislados en una red LAN privada sin salida directa a producción. El acceso remoto para administración se hace vía Tailscale, sin exponer el servidor ni depender de reenvío de puertos.


<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/00-introduccion/Topologia.png" width="600"> </p>

## Stack utilizado

- **Sistema operativo:** Red Hat Enterprise Linux 8.10
- **Detección:** Wazuh (Docker, single-node) + Suricata (nativo, NIDS)
- **Acceso remoto:** Tailscale (WireGuard)
- **Objetivo de enumeración/ataque:** OWASP Mutillidae II (nativo sobre httpd)
- **Contenerización de servicios auxiliares:** Docker / Docker Compose
## Capacidades de detección

- Hardening manual documentado: reducción de superficie de ataque, SSH, SELinux (enforcing) y firewalld con zonas/reglas explícitas.
- Ingesta de logs de Suricata (`eve.json`) y de Apache (`access_log`) vía agente nativo de Wazuh.
- Cobertura de escenarios de ataque mapeados a MITRE ATT&CK contra Mutillidae II (SQLi, XSS, IDOR, LFI, Command Injection, entre otros).
- Informes de respuesta a incidentes estructurados por escenario.

## Alcance

**Dentro de alcance:** un solo servidor RHEL, hardening y despliegue 100% manual (sin automatización, salvo Kickstart para la imagen reproducible), ataques simulados localmente sin tráfico a terceros, escenarios acotados a los mapeados en `attack-scenarios/mitre-mapping.md`.

**Fuera de alcance:** entornos multi-nodo o distribuidos, cobertura completa de MITRE ATT&CK, hardening o compliance certificado (no reemplaza un benchmark formal como CIS).

## Cómo navegar el repo

- **Replicar el lab desde cero:** empezar por `docs/es/01-preparacion-servidor.md` y seguir el orden numérico en `docs/es/`.
- **Decisiones de hardening específicas:** ver `hardening/` (SSH, SELinux, firewalld).
- **Despliegue del stack de detección:** ver `docs/es/04` a `docs/es/07`.
- **Escenarios de ataque y detección:** ver `attack-scenarios/`, con su mapeo a MITRE ATT&CK.

## Índice de documentación

|Doc|Contenido|
|---|---|
|00 — Introducción|Qué es el proyecto, objetivos, arquitectura, alcance|
|01 — Preparación del servidor|Especificaciones de la VM, instalación de RHEL|
|02 — Acceso remoto |Acceso seguro para colaboración sin exponer el servidor|
|03 — Hardening|Reducción de superficie de ataque, enlaces a SSH/SELinux/firewalld|
|04 — Despliegue de Wazuh (Docker)|Manager/indexer/dashboard en Docker single-node|
|05 — Agente de Wazuh (RHEL nativo)|Instalación del agente vía wizard|
|06 — Despliegue de Suricata (NIDS)|Integración con Wazuh, adaptaciones para RHEL|
|07 — Servicio web objetivo (webapp)|OWASP Mutillidae II como objetivo de ataque|

