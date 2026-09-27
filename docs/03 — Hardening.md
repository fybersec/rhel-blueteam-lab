## Enfoque general

El hardening de este laboratorio se organiza en tres frentes, cada uno con un objetivo distinto dentro de la estrategia de defensa en profundidad:

1. **Reducción de superficie de ataque** — eliminar servicios que vienen activos por defecto pero que este proyecto no necesita.
2. **Acceso remoto (SSH)** — endurecer el único punto de entrada administrativo al servidor. Detalle completo en `hardening/ssh/`.
3. **Filtrado de tráfico (firewalld)** — definir zonas y reglas explícitas en vez de dejar el firewall en su configuración por defecto. Detalle completo en`hardening/firewalld/`.

Este documento explica las decisiones de nivel general y el primer frente (reducción de superficie de ataque) en su totalidad. Los otros dos frentes se documentan en profundidad en sus archivos dedicados dentro de `hardening/`, para no duplicar contenido entre ambos lugares.

## Actualización inicial del sistema

Antes de tocar cualquier servicio, se actualizó el sistema completo (`dnf update`) para partir de una base sin vulnerabilidades conocidas ya corregidas por Red Hat.

- **Cuándo se hizo:** antes de eliminar servicios, como primer paso del hardening.

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/03-hardening/1.png" width="600"> </p>

## Reducción de superficie de ataque

Servicios que vienen habilitados por defecto en RHEL pero que no forman parte del alcance de este proyecto, y por lo tanto representan superficie de ataque innecesaria (procesos escuchando en puertos sin una razón de negocio/lab que lo justifique).

|Servicio|Puerto|Función original|Motivo de eliminación|Estado final|
|---|---|---|---|---|
|CUPS|631|Gestión de impresoras|El lab no imprime ni gestiona impresoras; un puerto abierto sin uso es riesgo innecesario.|Eliminado|
|Rpcbind|111|Mapeo de programas RPC (usado por NFS)|No se comparten archivos por NFS en este entorno.|Eliminado|
<p align="center"> <img src="[puertos_Antes](https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/03-hardening/2.png)" width="600"> </p>

1. Eliminar el servicio CUPS (Puerto 631)
```
sudo systemctl stop cups
sudo systemctl disable cups
sudo dnf remove cups -y
```
2. Eliminar el servicio Rpcbind (Puerto 111)
```
sudo systemctl stop rpcbind rpcbind.socket
sudo systemctl disable rpcbind rpcbind.socket
sudo dnf remove rpcbind -y
```

**Criterio usado para decidir qué eliminar:** se partió de los servicios que RHEL activa por defecto tras una instalación estándar (sin selección de "servidor mínimo") y se comparó contra la lista de servicios que el proyecto sí necesita (SSH, los que usan Wazuh/Suricata, y el servicio web objetivo). Todo lo que no aparecía en esa lista de necesarios se marcó como candidato a eliminar.

 <p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/03-hardening/3.png" width="600"> </p>

> Nota: se optó por **eliminar** el paquete (`dnf remove`) en vez de solo detener y deshabilitar el servicio, para que no quede binario instalado que pueda reactivarse por error o por una actualización futura.
