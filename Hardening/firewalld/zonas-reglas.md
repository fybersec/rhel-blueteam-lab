## Enfoque general

El criterio de este documento **no es "máxima seguridad"**, sino coherencia con el propósito del lab: este es un entorno de detección que necesita *ver* tráfico de ataque (ping, escaneos de puertos, intentos de explotación web) para poder generar alertas en Wazuh/Suricata. Un firewall que bloquee todo ese tráfico antes de que llegue a la interfaz de red estaría anulando el objetivo del proyecto completo (`00-introduccion.md`).

Por eso, `firewalld` se configura acá con un modelo de **zonas por origen de confianza**, no de "todo cerrado salvo lo mínimo". La regla general:

- Tráfico desde la **red LAN del lab** (donde están Kali y la máquina de administración) → permitido de forma amplia, porque es precisamente el tráfico que se quiere generar y observar.
- Tráfico desde **Tailscale** (administración remota, `02-acceso-remoto-vpn-tailscale.md`) → permitido solo para lo que es genuinamente administración (SSH, dashboard), no para los servicios vulnerables a propósito.
- Cualquier otro origen → zona por defecto, sin servicios expuestos.

## Por qué zonas por interfaz (y no reglas sueltas con `iptables`/`nftables` directo)

| Opción                           | Ventaja                                                                                                                                                            | Desventaja                                                                                                                                                        |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Zonas de firewalld (elegido)** | Declarativo, persistente entre reinicios, separa intención por origen de tráfico (LAN vs VPN vs desconocido) sin tener que pensar en cadenas de `iptables` a mano. | Una capa de abstracción más sobre `nftables`; hay que entender el modelo de zonas para no terminar con reglas contradictorias.                                    |
| Reglas directas en `nftables`    | Control total, sin abstracción.                                                                                                                                    | Sin persistencia automática de forma nativa, más fricción para documentar "qué se permite y por qué" de forma legible.                                            |
| Firewall desactivado             | Cero fricción para el tráfico de ataque.                                                                                                                           | Sin ningún registro ni filtrado — pierde valor formativo; un lab de blue team sin firewall configurado es una omisión notoria en cualquier revisión del proyecto. |
|                                  |                                                                                                                                                                    |                                                                                                                                                                   |

## Zonas asignadas

| Zona                                                        | Interfaz / origen                                                                            | Propósito                                                                                                                        |
| ----------------------------------------------------------- | -------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `internal`                                                  | Interfaz bridged de la LAN del lab (la que ve a Kali y a la máquina de administración local) | Zona de trabajo principal: acá es donde se generan y se quieren detectar los ataques.                                            |
| `trusted-vpn` *(zona personalizada, clonada de `internal`)* | `tailscale0`                                                                                 | Administración remota del colaborador — solo lo estrictamente administrativo, sin exponer los servicios vulnerables a propósito. |
| `public` *(por defecto)*                                    | Cualquier interfaz no asignada explícitamente                                                | Sin servicios habilitados. Cualquier cosa que llegue por acá no está contemplada en el diseño del lab.                           |

```bash
# Confirmar interfaz de la LAN y de Tailscale
ip a

# Asignar la interfaz LAN a la zona internal
sudo firewall-cmd --zone=internal --change-interface=<interfaz_LAN> --permanent

# Crear una zona propia para Tailscale, clonada de internal como base
sudo firewall-cmd --permanent --new-zone=trusted-vpn
sudo firewall-cmd --zone=trusted-vpn --change-interface=tailscale0 --permanent
```

## Servicios y puertos por zona

### Zona `internal` (LAN del lab — donde se generan los ataques)

|Puerto/Servicio|Protocolo|Uso|Justificación|
|---|---|---|---|
|22 (ssh)|TCP|Administración local|Ya cubierto por el hardening de `hardening/ssh/hardening_ssh.md`; se declara acá también para que la zona quede autocontenida.|
|1514|TCP|Agente Wazuh → manager|Requerido para que el propio agente nativo del servidor hable con el manager en Docker (`04-despliegue-wazuh-docker.md`).|
|1515|TCP|Enrollment de agentes|Mismo motivo que el anterior.|
|443|TCP|Dashboard de Wazuh|Acceso al SIEM desde la LAN.|
|8080|TCP|Mutillidae II|Objetivo de enumeración (`07-despliegue-webapp.md`) — debe ser alcanzable desde Kali para poder atacarlo.|
|ICMP (echo-request)|—|Ping de prueba para validar detección de Suricata|**No bloquear.** Es el escenario de verificación funcional documentado en `06-despliegue-suricata.md`; bloquear ICMP acá invalidaría la prueba más básica del NIDS.|

```bash
sudo firewall-cmd --zone=internal --add-service=ssh --permanent
sudo firewall-cmd --zone=internal --add-port=1514/tcp --permanent
sudo firewall-cmd --zone=internal --add-port=1515/tcp --permanent
sudo firewall-cmd --zone=internal --add-service=https --permanent
sudo firewall-cmd --zone=internal --add-port=8080/tcp --permanent
```

> Nota: la zona `internal` predefinida de firewalld ya trae ICMP permitido por defecto (no aparece bloqueado salvo que se agregue explícitamente un `icmp-block`). No es necesario ni conveniente tocar esta configuración — hacerlo rompería la prueba de ping documentada en `06-despliegue-suricata.md`.

### Zona `trusted-vpn` (Tailscale — solo administración remota)

|Puerto/Servicio|Protocolo|Uso|Justificación|
|---|---|---|---|
|22 (ssh)|TCP|Acceso remoto del colaborador|Ya cubierto por el hardening SSH; sigue aplicando igual sobre Tailscale (`02-acceso-remoto-vpn-tailscale.md`).|
|443|TCP|Dashboard de Wazuh|El colaborador necesita revisar alertas sin estar en la LAN física.|

**Deliberadamente NO se expone acá:** 8080 (Mutillidae) ni 1514/1515. El tráfico de ataque documentado en `attack-scenarios/` se ejecuta desde la LAN local, sin pasar por la VPN (así quedó definido en `02-acceso-remoto-vpn-tailscale.md`) — exponer el objetivo vulnerable también sobre Tailscale no aporta nada al lab y sí amplía innecesariamente la superficie de la VPN.

```bash
sudo firewall-cmd --zone=trusted-vpn --add-service=ssh --permanent
sudo firewall-cmd --zone=trusted-vpn --add-service=https --permanent
```

### Regla específica: API del indexer (9200)

El puerto 9200 (API del Wazuh indexer, `04-despliegue-wazuh-docker.md`) no se agrega como servicio abierto de zona — es tráfico administrativo/depuración (como el usado en las pruebas de ISM), no algo que cualquier host de la LAN deba poder alcanzar libremente. Se restringe con una *rich rule* a la IP de la máquina de administración únicamente:

```bash
sudo firewall-cmd --zone=internal --add-rich-rule='rule family="ipv4" source address="<IP_ADMIN>" port port="9200" protocol="tcp" accept' --permanent
```

Esto documenta una decisión real de higiene sin caer en bloquear el puerto por completo (que rompería el troubleshooting vía `curl` que ya se usó en este lab).

## Logging de paquetes rechazados (valor para un lab de blue team)

Por defecto, firewalld no registra los paquetes que descarta. Para un laboratorio cuyo objetivo es *observar*, tiene sentido activar el registro — da visibilidad adicional sobre qué tráfico llegó y fue descartado en la zona `public` (todo lo no contemplado):

```bash
sudo firewall-cmd --set-log-denied=all --permanent
```

Los eventos quedan en el log del kernel (`journalctl -k` / `/var/log/messages`), y pueden sumarse como una fuente más para el agente de Wazuh si más adelante se quiere correlacionarlos con las alertas de Suricata.

## Aplicar los cambios

```bash
sudo firewall-cmd --reload
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=internal --list-all
sudo firewall-cmd --zone=trusted-vpn --list-all
```

## Nota de coordinación con Docker

Como ya se documentó en `04-despliegue-wazuh-docker.md`, Docker gestiona sus propias reglas de `iptables`/`nftables` para publicar los puertos de los contenedores de Wazuh. Si `firewalld` se reinicia después de que Docker ya inició, esas reglas pueden quedar inconsistentes. Para evitarlo de forma persistente (en vez de reiniciar Docker manualmente cada vez que ocurra), se puede forzar el orden de arranque a nivel systemd:

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
echo -e "[Unit]\nAfter=firewalld.service" | sudo tee /etc/systemd/system/docker.service.d/after-firewalld.conf
sudo systemctl daemon-reload
```

## Verificación post-configuración

- [ ] Desde Kali (LAN), el `ping` y el escaneo Nmap documentados en `06-despliegue-suricata.md` siguen llegando y generando alertas.
- [ ] Desde Kali (LAN), Mutillidae (8080) sigue siendo accesible.
- [ ] Desde una sesión por Tailscale, el dashboard (443) y SSH funcionan; el puerto 8080 **no** responde.
- [ ] El puerto 9200 solo responde desde la IP de administración definida en la rich rule.
