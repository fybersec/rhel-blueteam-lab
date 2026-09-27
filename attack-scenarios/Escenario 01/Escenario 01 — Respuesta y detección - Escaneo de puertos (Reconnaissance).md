## Fuente de detección esperada

Suricata detectó el escaneo por el patrón de múltiples intentos de conexión a distintos puertos en una ventana de tiempo corta, proveniente de una única IP origen (la de Kali). Esto coincidió con firmas de la categoría `ET SCAN` cargadas desde Emerging Threats (ver `06-despliegue-suricata.md`).

**Alerta representativa generada:**

```
[**] [1:2010935:2] ET SCAN Potential SSH Scan [**]
[Classification: Detection of a Network Scan] [Priority: 3]
{TCP} <Linux Mint>:XXXXX -> <Red hat>:22
```

Al tratarse de un `-p-` (65535 puertos), también se dispararon firmas genéricas de escaneo de puertos (`ET SCAN Possible Nmap User-Agent Observed` no aplica acá por no ser HTTP, pero sí firmas basadas en umbral de conexiones SYN por IP en poco tiempo).

Estas alertas llegan al dashboard de Wazuh vía la integración `eve.json` → agente → manager documentada en `06-despliegue-suricata.md`, visibles en **Threat Hunting → Events**, filtrando por `rule.groups:suricata`.

## Información recopilada sobre el posible atacante

- **IP origen:** la de la VM de Linux Mint (única IP en la red aislada del lab, no hay margen de ambigüedad en este entorno de una sola máquina atacante).
- **Patrón de la petición:** múltiples SYN a puertos secuenciales/aleatorios en corto tiempo — comportamiento típico de escaneo automatizado, no de tráfico legítimo de usuario.
- **Puertos objetivo:** todo el rango 1-65535, lo que por sí solo ya es un indicador de reconocimiento y no de uso normal del servicio.

## Limitaciones observadas

- El **SYN scan (`-sS`)** no completa el handshake TCP en los puertos donde no hay intención de conectarse realmente, por lo que **no aparece en absoluto en logs de aplicación** (ni `sshd`, ni `access_log`). La detección depende exclusivamente de Suricata a nivel de red.
- El escaneo UDP (`-sU`) generó bastante menos señal: UDP no tiene handshake, así que las firmas de Emerging Threats para scan UDP dependen de umbrales de tasa, que con un `--top-ports 20` acotado no siempre se dispararon de forma consistente.

## Mitigación / hardening relacionado

| Medida                                                                                 | Ya aplicada en este lab                    | Notas                                                                                                                                                                                                                                       |
| -------------------------------------------------------------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Reducción de superficie de ataque                                                      | Sí (`03-hardening.md`)                     | CUPS y Rpcbind eliminados — menos puertos que un escaneo puede revelar como abiertos.                                                                                                                                                       |
| firewalld con zonas explícitas                                                         | Sí (`hardening/firewalld/zonas-reglas.md`) | Solo los puertos necesarios (22, 8080, 443, 1514/1515, 9200) están permitidos; el resto responde con RST/drop en vez de quedar en un estado ambiguo.                                                                                        |
| Rate limiting de conexiones nuevas (ej. `firewalld` con `rich rules` de límite por IP) | No aplicada                                | Pendiente para una futura iteración — permitiría frenar la velocidad de un escaneo `-T4` sin bloquear tráfico legítimo.                                                                                                                     |
| Port knocking / ocultar SSH detrás de Tailscale exclusivamente                         | Parcial                                    | Tailscale ya existe para administración (`02-acceso-remoto-vpn.md`), pero el puerto 22 sigue expuesto también en la LAN del lab para no romper el acceso desde Kali durante los escenarios de ataque — decisión consciente, no un descuido. |
| Alertas correlacionadas por umbral en Wazuh (no solo Suricata)                         | No aplicada                                | Se podría sumar una regla de Wazuh que correlacione múltiples eventos de Suricata de la misma IP en poco tiempo y eleve el nivel de alerta automáticamente. Queda como mejora futura.                                                       |

## Checklist de verificación

- [ ] La alerta de Suricata aparece en el dashboard de Wazuh dentro de los segundos posteriores al escaneo.
- [ ] La IP origen registrada coincide con la de Kali.
- [ ] Los puertos reportados como "abiertos" por Nmap coinciden con los servicios realmente expuestos según `firewalld --list-ports`.
- [ ] No se generaron falsos positivos por tráfico de Docker (`172.17.0.0/16`, `172.18.0.0/16`) durante la ventana del escaneo.

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/videos/01/Wazuh.png" width="600"> </p>
