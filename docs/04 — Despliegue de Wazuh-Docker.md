## Enfoque general

Wazuh se despliega usando el repositorio oficial wazuh, en su variante **single-node** (un manager, un indexer, un dashboard). El detalle técnico completo, este documento explica las decisiones y el porqué.

## Por qué Docker (y no instalación nativa)

- Facilidad para destruir y recrear el stack completo si algo se rompe durante las pruebas de ataque, sin arriesgar el sistema base ya endurecido en `03-hardening.md`.
- El manager y el indexer no necesitan acceso directo al sistema de archivos del host (a diferencia de Suricata, ver `06-despliegue-suricata.md`), por lo que el aislamiento de red que impone Docker no genera fricción.

## Por qué single-node (y no multi-node)

El modo multi-node de Wazuh (2 managers, 3 indexers) está pensado para alta disponibilidad en entornos de producción. Para un lab de una VM, no aporta nada y consume recursos que este proyecto no tiene disponibles. Single-node es la opción correcta acá.

## Advertencia de recursos (importante)

El Wazuh indexer (basado en OpenSearch) suele requerir entre 2 y 4 GB de heap de memoria por sí solo. 

- **Requisito técnico obligatorio:** `vm.max_map_count` debe estar en al menos `262144` en el host, o el indexer falla al arrancar. Esto se configura antes de levantar el stack.

## Arquitectura de componentes

|Componente|Dónde corre|Por qué|
|---|---|---|
|Wazuh manager|Contenedor Docker|Recibe datos por red desde los agentes; no necesita tocar el sistema de archivos del host.|
|Wazuh indexer|Contenedor Docker|Almacenamiento e indexado de eventos; aislarlo en contenedor facilita gestión de volúmenes y recursos.|
|Wazuh dashboard|Contenedor Docker|Interfaz web; sin necesidad de acceso al host.|
|Wazuh agente (en el propio RHEL)|**Nativo**, fuera de Docker|Necesita leer archivos locales del sistema (logs de Suricata, integridad de archivos, etc.) — un contenedor no vería esos archivos sin montar volúmenes adicionales, complejidad innecesaria para este caso.|

## Instalación de docker 
[Instalación paso a paso de docker en Ret Hat](https://help.hcl-software.com/bigfix/11.0/mcm/MCM/Install/install_docker_ce_docker_compose_on_rhel_8.html)
<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/Wazuh-docker/docker1.png" width="600"> </p>
<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/Wazuh-docker/docekr2.png" width="600"> </p>

## Instalación y despliegue de wazuh-docker 
[Pasos de instalación](https://documentation.wazuh.com/current/deployment-options/docker/wazuh-container.html)
<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/Wazuh-docker/wazuh1.png" width="600"> </p>
<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/Wazuh-docker/wazuh2.png" width="600"> </p>
<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/Wazuh-docker/wazuh3.png" width="600"> </p>



## Versión utilizada

- **Rama/tag de `wazuh-docker` clonado:** v4.14.7

> Nota de compatibilidad: los agentes de Wazuh solo son compatibles con managers de versión igual o más nueva que ellos. Si más adelante actualizás el manager, hay que revisar que el agente nativo siga siendo compatible.

## Requisitos de red (coordinación con firewalld)

Para que el agente nativo y el dashboard puedan comunicarse con los contenedores, los siguientes puertos deben estar permitidos en la zona de firewalld correspondiente (definida en `hardening/firewalld/zonas-reglas.md`):

|Puerto|Protocolo|Uso|
|---|---|---|
|1514|TCP|Comunicación agente → manager (datos)|
|1515|TCP|Comunicación agente → manager (registro/enrollment)|
|443|TCP|Acceso al dashboard vía HTTPS|
|9200|TCP|API del indexer (normalmente solo accedido internamente, evaluar si necesita exponerse fuera del host)|

> Nota técnica: Docker gestiona sus propias reglas de `iptables`/`nftables` para publicar estos puertos. Si firewalld se reinicia después de que Docker ya inició, las reglas de Docker pueden quedar inconsistentes — si esto ocurre, reiniciar el servicio de Docker suele resolverlo. Documentar acá si te pasó.

## Verificación post-despliegue

- [ ] Los tres contenedores (manager, indexer, dashboard) están en estado `Up` (`docker compose ps`).
- [ ] El dashboard es accesible vía HTTPS desde la máquina de administración remota.
- [ ] No hubo errores de comunicación entre dashboard y API del manager (el típico error de API mal configurado que se buscó evitar usando Docker).
