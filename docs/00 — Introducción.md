## ¿Qué es este proyecto?

**RHEL Blue Team Lab** es un laboratorio personal de detección de intrusos construido sobre Red Hat Enterprise Linux. El objetivo es levantar un entorno controlado donde se pueda practicar hardening manual, desplegar un stack de detección (Wazuh + Suricata) y ejecutar escenarios de ataque simulados para verificar qué se detecta y qué no.

Este proyecto nació con un foco en entender el ciclo: preparar el sistema, endurecerlo, instrumentarlo, atacarlo de forma controlada, y documentar la respuesta.

**Esto no es:**

- Un producto listo para producción.
- Una guía genérica de "cómo instalar Wazuh" — es la documentación de _mis_ decisiones específicas en _este_ entorno.
- Un entorno expuesto a internet o accesible fuera del lab aislado.

## Objetivos del laboratorio

- Aplicar hardening manual sobre RHEL (SSH, firewalld) documentando cada decisión y su justificación.
- Desplegar y configurar Wazuh + Suricata como núcleo de detección.
- Levantar un servicio web vulnerable propio (nativo) como objetivo de enumeración, para practicar escaneo (Gobuster, Nikto) desde el lado ofensivo y verificar detección desde el lado defensivo.
- Ejecutar escenarios de ataque mapeados a MITRE ATT&CK y documentar qué alertas dispara cada uno.
- Practicar respuesta a incidentes con informes estructurados.

## Arquitectura general

| Rol                  | SO / Firmware   | RAM         | Almacenamiento |
| -------------------- | --------------- | ----------- | -------------- |
| Servidor             | Red Hat 8.10    | 8 GB        | 80 GiB         |
| Administrador Remoto | Windows 11/10   | 16 GB       | 238 GiB        |
| Attacks              | Linux Mint 22.3 | 3.8 GB      | 62.5 GiB       |
| Router               | GL-SFT1200      | 128 MB DDR3 | 128 MB         |

El laboratorio corre sobre una única máquina virtual RHEL 8.10. Dentro de ella conviven el stack de detección (Wazuh + Suricata) y el servicio web objetivo, todos aislados en una red LAN privada dentro del router GL-SFT1200, sin salida directa a producción. La red sí tiene acceso a internet durante la instalación de servicios (paquetes, imágenes Docker, etc.), pero los escenarios de ataque para generar alertas se ejecutan sin conexión a internet, por seguridad.


<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/00-introduccion/Topologia.png" width="600"> </p>



## Alcance y limitaciones

**Dentro de alcance:**

- Un solo servidor RHEL.
- Hardening y despliegue 100% manual, sin Ansible ni otras herramientas de automatización.
- Ataques simulados localmente, sin tráfico real hacia terceros.
- Escenarios acotados a los mapeados.

**Fuera de alcance:**

- Entornos multi-nodo o distribuidos.
- Cobertura completa de MITRE ATT&CK (se documentan técnicas puntuales, no la matriz entera).
- Hardening o compliance certificado.

## Cómo navegar el repo

- Si querés replicar el lab desde cero, empezá por `docs/es/01-preparacion-servidor.md` y seguí el orden numérico.
- Si solo te interesan los escenarios de ataque y detección, andá directo a `attack-scenarios/`.
- Si buscás las decisiones de hardening específicas, revisá `hardening/`.
- Los informes de incidentes generados durante las pruebas están en `incident-reports/`.

## Stack utilizado

- **Sistema operativo:** Red Hat Enterprise Linux 8.10
- **Detección:** Wazuh, Suricata
- **Objetivo de enumeración:** servicio web propio (ver `webapp/`)
- **Contenerización de servicios auxiliares:** Docker / Docker Compose
