# Cómo está organizado este repositorio

Este no es un proyecto pensado para recibir contribuciones externas activas (es documentación de un lab personal), pero está estructurado para que sea fácil de leer, replicar y adaptar. Esta guía explica esa estructura y cómo forkearlo para tu propio entorno.

## Estructura del repo

```
├── docs/es/                  # Documentación numerada, en orden de despliegue (00 → 07+)
├── hardening/                 # Detalle de SSH, SELinux y firewalld (referenciado desde docs/es/03)
│   ├── ssh/
│   └── firewalld/
├── attack-scenarios/           # Escenarios de ataque mapeados a MITRE ATT&CK
│   └── mitre-mapping.md
├── incident-reports/           # Informes de respuesta a incidentes por escenario
├── webapp/                     # (si aplica) código del servicio web propio, si se documenta
├── screenshots-videos/assets/  # Capturas referenciadas desde docs/es/
├── README.md
├── LICENSE
├── SECURITY.md
└── CONTRIBUTING.md
```

La documentación en `docs/es/` explica **decisiones y el porqué**; el detalle técnico exhaustivo de cada componente vive en su carpeta dedicada (`hardening/`) para no duplicar contenido entre ambos lugares.

## Cómo adaptarlo a tu propio entorno

Este lab fue construido para una topología específica (una VM RHEL, un router GL-SFT1200, direccionamiento propio). Para replicarlo:

1. **Forkeá el repositorio.**
2. **Ajustá direccionamiento e identidad:**
    - IPs estáticas referenciadas en `docs/es/01-preparacion-servidor.md`.
    - Hostname (`rhel-lab-blue`) y nombre de agente de Wazuh (`rhel-blueteam-lab`) en `docs/es/05-configuracion-agente-wazuh.md`.
    - Nombre de interfaz de red (`ens160` en este lab; puede variar según hipervisor) en la configuración de Suricata y en `/etc/sysconfig/suricata`.
3. **Revisá los requisitos de hardware**, en particular la RAM asignada a la VM (ver advertencia de recursos en `docs/es/04-despliegue-wazuh-docker.md` sobre el heap del indexer de Wazuh).
4. **Ajustá el acceso remoto:** la sección de Tailscale (`docs/es/02-acceso-remoto-vpn-tailscale.md`) asume un tailnet propio; reemplazá por tu propia configuración de VPN o acceso remoto si usás otra herramienta.
5. **No reutilices el servicio web objetivo tal cual en un entorno expuesto:** Mutillidae II es intencionalmente vulnerable (ver `SECURITY.md`).
