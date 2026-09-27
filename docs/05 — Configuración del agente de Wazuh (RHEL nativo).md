## Enfoque general

El agente de Wazuh se instala **nativo** (fuera de Docker) sobre el mismo servidor RHEL donde corre el manager en contenedores. La justificación de esta arquitectura está documentada en `docs/es/04-despliegue-wazuh-docker.md`.

Este documento describe el método de instalación **vía el wizard "Deploy new agent"** del dashboard de Wazuh, que resultó ser el camino más limpio: evita el registro manual con `manage_agents` y el riesgo de auto-registro duplicado que puede darse con la instalación por línea de comandos "a mano".

## Por qué el wizard y no la instalación manual

El wizard genera un único comando de instalación con las variables de entorno correctas ya inyectadas (`WAZUH_MANAGER` y `WAZUH_AGENT_NAME`), en vez de dejarlas para configurar después en `ossec.conf`. Esto evita dos problemas típicos:

- Que el agente tome el hostname del sistema como nombre (en este entorno, contaminado por la configuración de Tailscale documentada.
- Que una IP de manager mal escrita quede grabada en `ossec.conf` y el agente arranque sin poder conectarse (`Never connected` en `agent_control -l`).

## 1. Acceso al wizard

Desde el dashboard de Wazuh (`Overview` → `Deploy new agent`, o directamente desde `Endpoints` → `Deploy new agent`):

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/05-Wazuh-agente/1.png" width="600"> </p>

## 2. Selección de paquete y dirección del servidor

- Sistema: **Linux → RPM amd64**
- **Server address**: `127.0.0.1` (manager en Docker sobre el mismo host)

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/05-Wazuh-agente/2.png" width="600"> </p>

## 3. Configuración opcional: nombre del agente

- **Assign an agent name**: `rhel-blueteam-lab`
- **Grupo**: `default`

> El nombre del agente es único y no se puede cambiar una vez enrolado. Fijarlo acá evita que el agente herede el hostname del sistema.

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/05-Wazuh-agente/3.png" width="600"> </p>

## 4. Comando de instalación generado

El wizard arma el siguiente comando, con el manager y el nombre del agente ya parametrizados:

```bash
curl -o wazuh-agent-4.14.7-1.x86_64.rpm https://packages.wazuh.com/4.x/yum/wazuh-agent-4.14.7-1.x86_64.rpm && sudo WAZUH_MANAGER='127.0.0.1' WAZUH_AGENT_NAME='rhel-blueteam-lab' rpm -ihv wazuh-agent-4.14.7-1.x86_64.rpm
```

Ejecutado en el servidor RHEL:

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/05-Wazuh-agente/4.png" width="600"> </p>

> Durante la instalación aparece una advertencia (`NOKEY`) porque no se importó la clave GPG del repositorio antes de instalar el paquete directamente por `curl`. No afecta la instalación; si se quiere evitar la advertencia, importar la clave con `rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH` antes de este paso.

## 5. Habilitar e iniciar el servicio

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/05-Wazuh-agente/5.png" width="600"> </p>
<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/05-Wazuh-agente/6.png" width="600"> </p>

## 6. Verificación

Desde el dashboard, en `Endpoints`:

|ID|Nombre|IP|Grupo|SO|Versión|Estado|
|---|---|---|---|---|---|---|
|001|rhel-blueteam-lab|127.0.0.1|default|Red Hat Enterprise Linux 8.10|v4.14.7|active|

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/05-Wazuh-agente/7.png" width="600"> </p>

Un solo registro, activo desde el primer intento, sin necesidad de limpiar registros huérfanos.

## Comparación con el método manual (referencia / troubleshooting)

|Aspecto|Instalación manual (repo + `manage_agents`)|Wizard "Deploy new agent"|
|---|---|---|
|Config de repo|Manual (`rpm --import` + `.repo`)|No aplica (descarga directa del `.rpm`)|
|IP del manager|Riesgo de error tipeado, corrección posterior en `ossec.conf`|Fijada correctamente desde el campo del wizard|
|Nombre del agente|Heredado del hostname si no se fija explícitamente|Fijado en el propio comando de instalación|
|Registro|Manual (`manage_agents` + clave) o auto-registro posterior|Automático, sin pasos intermedios|
|Riesgo de duplicados|Sí, si se combinan ambos mecanismos|No|

## Requisitos de red (coordinación con firewalld)

Los mismos puertos definidos en `docs/es/04-despliegue-wazuh.md` aplican acá:

|Puerto|Protocolo|Uso|
|---|---|---|
|1514|TCP|Comunicación agente → manager (datos)|
|1515|TCP|Comunicación agente → manager (registro/enrollment)|

## Verificación post-instalación

- [ ] El servicio `wazuh-agent` está `active (running)` (`systemctl status wazuh-agent`).
- [ ] El agente aparece en `Endpoints` del dashboard con estado `active`.
- [ ] El nombre y la IP del agente coinciden con los definidos en el wizard.

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/05-Wazuh-agente/Prueba_1-agente.jpeg" width="600"> </p>
<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/05-Wazuh-agente/Prueba_2-agente.png" width="600"> </p>