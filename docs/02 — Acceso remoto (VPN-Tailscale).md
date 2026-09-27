## Por qué esta fase existe

Antes de avanzar con el despliegue del stack de detección, se necesita una forma de acceder al servidor RHEL sin exponerlo directamente a internet ni depender de reenvío de puertos en el router doméstico (GL-SFT1200).

**Esto no reemplaza el hardening de SSH ya documentado en `hardening/ssh/`** — Tailscale resuelve _cómo llegar_ a la red del lab de forma segura; el hardening de SSH sigue siendo responsable de _qué se permite_ una vez que ya se llegó.

[Instalación de Tailscale](https://tailscale.com/docs/install/linux)

## Por qué Tailscale (y no otra alternativa)

| Opción                                     | Ventaja                                                                                                                                                                      | Desventaja                                                                                                                                             |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Tailscale (elegido)**                    | Configuración simple basada en WireGuard, red mesh sin necesidad de abrir puertos en el router, control de acceso granular por ACLs/tags, cada nodo con su propia identidad. | Depende de un servicio de coordinación de terceros (aunque el tráfico de datos es punto a punto).                                                      |
| VPN tradicional (OpenVPN/WireGuard manual) | Control total, sin dependencia de terceros.                                                                                                                                  | Requiere abrir puertos en el router, gestionar certificados/claves manualmente — más superficie de mantenimiento para un lab de dos personas.          |
| Reenvío de puertos + SSH directo           | Sin herramientas adicionales.                                                                                                                                                | Expone el servidor directamente a internet — contradice el principio de "sin salida directa a producción/exposición" definido en `00-introduccion.md`. |

## Alcance de este cambio

- **Qué cambia:** se agrega una interfaz de red virtual (Tailscale) sobre el servidor RHEL y sobre las máquinas de administración remota y del colaborador.
- **Qué NO cambia:** la red bridged/LAN local definida en `01-preparacion-servidor.md` sigue existiendo para el tráfico entre el servidor, Kali y el resto del lab durante los escenarios de ataque (que se siguen ejecutando sin salida a internet, como ya está documentado).
- **Separación de tráficos:** Tailscale se usa exclusivamente para **administración remota** (acceso SSH del colaborador, revisión de dashboards). El tráfico de los escenarios de ataque (`attack-scenarios/`) sigue corriendo de forma aislada, sin pasar por la VPN.

## Instalación


<p align="center"> <img src="[imagen de la instalacion](https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/02-Tailscale/1.png)" width="600"> </p>
<p align="center"> <img src="[imagen de la instalacion](https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/02-Tailscale/2.png)" width="600"> </p>
<p align="center"> <img src="[imagen de la instalacion](https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/02-Tailscale/3.png)" width="600"> </p>
