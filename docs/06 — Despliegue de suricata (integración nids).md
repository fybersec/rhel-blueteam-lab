## Enfoque general

Suricata se despliega como motor NIDS (Network Intrusion Detection System) **nativo** sobre el mismo servidor RHEL donde corre el agente de Wazuh, siguiendo la misma lógica de arquitectura documentada en `04-despliegue-wazuh-docker.md`: Suricata necesita leer/escribir directamente sobre el sistema de archivos del host (interfaz de red física, log `eve.json`) y el agente de Wazuh necesita leer ese mismo archivo en disco, por lo que ninguno de los dos vive en un contenedor Docker.

La integración sigue el flujo oficial documentado por Wazuh (Suricata → `eve.json` → agente Wazuh vía `localfile` → manager → indexer → dashboard), pero la guía oficial está escrita para Ubuntu. Este documento recoge las adaptaciones necesarias para RHEL 8.10 y las decisiones tomadas durante el despliegue real en este lab.

## Por qué Suricata (y no otro NIDS)

| Opción                 | Ventaja                                                                                                                                                                                       | Desventaja                                                                                                                                        |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Suricata (elegido)** | Integración oficial y documentada con Wazuh, multi-hilo, soporta reglas de Emerging Threats out-of-the-box, salida nativa en JSON (`eve.json`) que Wazuh parsea sin transformación adicional. | Build de EPEL para RHEL 8 no compila soporte para protocolos industriales (Modbus, DNP3, EtherNet/IP) — no relevante para el alcance de este lab. |
| Snort                  | Más liviano, ampliamente documentado.                                                                                                                                                         | Integración con Wazuh menos directa; formato de log requiere parseo adicional.                                                                    |
| Zeek                   | Análisis de protocolo más profundo, mejor para forense de red.                                                                                                                                | Curva de aprendizaje mayor, no es el foco de este lab (detección en tiempo real, no análisis forense post-mortem).                                |

## Diferencias clave frente a la guía oficial (escrita para Ubuntu)

|Aspecto|Ubuntu (guía oficial)|RHEL 8.10 (este lab)|
|---|---|---|
|Gestor de paquetes|`apt` + PPA de Suricata|`dnf` + repositorio EPEL|
|Repositorio de Suricata|`ppa:oisf/suricata-stable`|No existe equivalente nativo; se usa el paquete de EPEL (requiere también CodeReady Builder habilitado)|
|Nomenclatura de interfaz|`eth0` / `enp0s3`|`ens160` (nomenclatura predecible de systemd, propia de RHEL/VMware)|
|Control de la interfaz de escucha|Solo `af-packet` en `suricata.yaml`|`af-packet` en `suricata.yaml` **y** `OPTIONS` en `/etc/sysconfig/suricata` — ambos deben coincidir, o el servicio arranca en la interfaz por defecto (`eth0`) sin importar el `.yaml`|
|Soporte de protocolos SCADA|Sin problema reportado|El build de EPEL no soporta Modbus/DNP3/EtherNet-IP; cargar reglas de esos protocolos (`emerging-scada.rules`) provoca un **crash (SEGV)** del proceso, no solo un error de parseo|
|Control de acceso obligatorio|AppArmor (perfil laxo por defecto)|SELinux en `enforcing` — requiere verificación de denials antes de descartar problemas de permisos|

## Infraestructura

| Endpoint                      | Descripción                                                                                                                                                                                                   |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| rhel-blueteam-lab (RHEL 8.10) | Mismo servidor donde corre el agente de Wazuh nativo (`05-configuracion-agente-wazuh.md`). Suricata monitorea el tráfico de la interfaz `ens160`, la misma que usa el agente para comunicarse con el manager. |

## Requisitos previos

- [ ] Agente de Wazuh ya instalado y en estado `active (running)` (ver `05-configuracion-agente-wazuh.md`).
- [ ] Interfaz de red y IP del servidor identificadas (`ip a`).
- [ ] Snapshot de la VM tomado antes de empezar (recomendado — ver `01-preparacion-servidor.md`).

## Instalación de Suricata

RHEL no incluye Suricata en sus repositorios base. Se obtiene desde EPEL, que a su vez depende del repositorio CodeReady Builder (CRB):

```bash
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo subscription-manager repos --enable codeready-builder-for-rhel-8-x86_64-rpms
sudo dnf install suricata -y
```

> **Decisión:** se evitó correr `dnf update -y` como paso previo. En este entorno generó conflictos de dependencias entre `docker-ce`/`containerd.io` (instalados para el stack de `04-despliegue-wazuh-docker.md`) y el `runc` nativo de RHEL — un conflicto preexistente sin relación con Suricata. Actualizar el sistema completo queda fuera del alcance de esta fase.

**Versión instalada:** Suricata 7.0.16-1.el8 (EPEL)

```bash
suricata --version
```

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/06-Suricata/1.png" width="600"> </p>

## Configuración de reglas (Emerging Threats)

```bash
cd /tmp/
curl -LO https://rules.emergingthreats.net/open/suricata-7.0.3/emerging.rules.tar.gz
sudo tar -xvzf emerging.rules.tar.gz
sudo mkdir -p /etc/suricata/rules
sudo mv rules/*.rules /etc/suricata/rules/
sudo chown -R suricata:suricata /etc/suricata/rules
```

### Exclusión de reglas SCADA (decisión de compatibilidad)

El build de Suricata de EPEL para RHEL 8 no compila soporte para protocolos industriales. Cargar `emerging-scada.rules` (que contiene firmas Modbus/DNP3) no solo genera errores de parseo — **provoca un segmentation fault (SEGV) que tumba el proceso completo**, dejando el NIDS caído sin aviso claro en `systemctl status`.

Dado que estos protocolos no forman parte del alcance de este lab (`00-introduccion.md`), se optó por excluir el archivo del directorio activo de reglas en vez de intentar parchear el binario:

```bash
sudo mkdir -p /etc/suricata/rules/disabled
sudo mv /etc/suricata/rules/emerging-scada.rules /etc/suricata/rules/disabled/
```


## Configuración de `/etc/suricata/suricata.yaml`

```yaml
HOME_NET: "<IP_DEL_SERVIDOR>"
EXTERNAL_NET: "any"

default-rule-path: /etc/suricata/rules
rule-files:
  - "*.rules"

stats:
  enabled: yes

af-packet:
  - interface: ens160
```

> **Decisión de laboratorio:** se define `HOME_NET` como la IP propia del servidor (no el rango `/24` completo), replicando el criterio de la guía original. Esto hace que **cualquier origen distinto a la propia IP** dispare una alerta al recibir tráfico — útil para un entorno de práctica donde se quiere ver detección inmediata, pero no representa el uso recomendado en un entorno de producción (donde `HOME_NET` normalmente define la red interna completa).

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/06-Suricata/2.png" width="600"> </p>

## Configuración del arranque del servicio (`/etc/sysconfig/suricata`)

A diferencia de Ubuntu, en RHEL el nombre de la interfaz con la que arranca el servicio systemd **no se toma únicamente del `af-packet` del `.yaml`** — el archivo `/etc/sysconfig/suricata` define el flag `-i` que se le pasa al binario en el arranque, y tiene prioridad. Si no se ajusta, el servicio sigue escuchando en `eth0` (interfaz que no existe en RHEL) sin importar lo configurado en el `.yaml`.

```bash
sudo vi /etc/sysconfig/suricata
```

```bash
OPTIONS="-i <interface>"
```

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/06-Suricata/3.png" width="600"> </p>

## Habilitar y arrancar el servicio

```bash
sudo systemctl enable suricata
sudo systemctl restart suricata
sudo systemctl status suricata
```

Verificación esperada en la salida:

- Estado `active (running)`.
- El proceso listado muestra `-i ens160` (no `eth0`).
- Pueden aparecer avisos `E: detect-parse:` de protocolos no soportados si queda alguna firma suelta — no deben provocar un `core-dump`.

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/06-Suricata/4.png" width="600"> </p>

## Integración con el agente de Wazuh (`ossec.conf`)

```bash
sudo vi /var/ossec/etc/ossec.conf
```

Se agrega un bloque **`<ossec_config>` independiente**, con apertura y cierre propios:

```xml
<ossec_config>
  <localfile>
    <log_format>json</log_format>
    <location>/var/log/suricata/eve.json</location>
  </localfile>
</ossec_config>
```

> **Nota de troubleshooting documentada:** el archivo `ossec.conf` de este agente ya contiene múltiples bloques `<ossec_config>` separados (uno para `<client>`, otro para logs de sistema). El bloque de `<localfile>` para Suricata debe llevar su **propio** envoltorio `<ossec_config>` — agregarlo suelto al final del archivo (fuera de cualquier bloque) provoca el error `Invalid element in the configuration: 'localfile'` y el agente no arranca (`No client configured. Exiting`).

```bash
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/06-Suricata/5.png" width="600"> </p>

> **Nota de red:** el `HOME_NET` apunta a la IP del servidor sobre la interfaz `ens160` (red LAN/bridged del lab). El tráfico de prueba debe originarse desde un host que efectivamente llegue por esa interfaz — el tráfico interno de las redes Docker del propio host (`172.17.0.0/16`, `172.18.0.0/16`, usadas por el stack de `04-despliegue-wazuh-docker.md`) no es representativo de un ataque externo real.

## Verificación post-instalación

- [ ] `systemctl status suricata` en `active (running)`, sin `core-dump`.
- [ ] El proceso de Suricata escucha en la interfaz correcta (`ens160`), confirmado en la salida de `systemctl status`.
- [ ] `systemctl status wazuh-agent` en `active (running)`, sin errores de configuración.
- [ ] El archivo `/var/log/suricata/eve.json` recibe eventos nuevos tras el ping/nmap de prueba.
- [ ] Las alertas aparecen en el dashboard de Wazuh, en **Threat Hunting → Events**, filtrando por `rule.groups:suricata`.

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/06-Suricata/5.png" width="600"> </p>
