## Fuente de detección esperada

Según `attack-scenarios/mitre-mapping.md`: **Wazuh**, mediante las reglas de autenticación fallida SSH sobre los logs de `sshd` (`/var/log/secure` en RHEL). Suricata no interviene directamente acá porque un intento de login SSH es tráfico cifrado y sin patrón de firma de red distintivo (a diferencia del escaneo de puertos del escenario 01) — la señal solo existe a nivel de aplicación, en el propio servicio SSH.

## Comportamiento observado en Wazuh

Wazuh recolecta los eventos de `sshd` vía el agente instalado en el servidor (ver `01-instalacion-base.md` / despliegue del agente) y los correlaciona contra las reglas por defecto del decoder `sshd`, que incrementan el nivel de alerta ante múltiples fallos de autenticación desde el mismo origen en una ventana de tiempo corta.

**Alerta representativa generada:**

```
** Alert <timestamp>: - syslog,sshd,authentication_failures,
Rule: 5720 (level 10) -> 'sshd: brute force trying to get access to the system. Authentication failed.'
Src IP: <IP_KALI>
User: <usuario_probado>
```

Estas alertas son visibles en el dashboard de Wazuh en **Threat Hunting → Events**, filtrando por `rule.groups:authentication_failed` o por el ID de regla asociado a fuerza bruta SSH.

## Información recopilada sobre el posible atacante

- **IP origen:** la de la VM de Kali (única IP en la red aislada del lab).
- **Usuarios probados:** lista de nombres de usuario extraída de los eventos de `sshd`, útil para inferir si el atacante conoce o está adivinando nombres de cuenta válidos.
- **Frecuencia de intentos:** volumen y ritmo de los fallos de autenticación en la ventana del ataque — un patrón sostenido de intentos en segundos es indicador claro de automatización, no de un error de tipeo legítimo.
- **Resultado del ataque:** si alguna combinación usuario/contraseña tuvo éxito, Wazuh también genera un evento de autenticación exitosa (`sshd: authentication success`) inmediatamente después de una racha de fallos desde la misma IP — patrón que por sí solo amerita una alerta de mayor prioridad, más allá del volumen de fallos.

## Limitaciones observadas

- La detección depende **exclusivamente de los logs de `sshd`** recolectados por el agente de Wazuh; si el agente estuviera caído o el log rotara antes de la ingesta, se perdería visibilidad completa sobre el ataque.
- Suricata no aporta señal en este escenario: al ser tráfico SSH cifrado, no hay payload visible para una firma de red, y el volumen de paquetes de un intento de login no es por sí solo un patrón de escaneo como el del escenario 01.
- Si el diccionario de contraseñas no contiene la credencial real, el ataque no compromete la cuenta, pero de todas formas genera el mismo volumen de alertas de fuerza bruta — la detección no depende del éxito del ataque.

## Mitigación / hardening relacionado

| Medida                                                  | Ya aplicada en este lab                    | Notas                                                                                                                                                                       |
| ------------------------------------------------------- | ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Autenticación por clave pública en vez de password      | No aplicada                                | Rompería el ejercicio de fuerza bruta; se mantiene auth por password deliberadamente para este escenario del lab.                                                           |
| firewalld con zonas explícitas                          | Sí (`hardening/firewalld/zonas-reglas.md`) | El puerto 22 está permitido explícitamente para no romper el acceso desde Kali durante los escenarios de ataque — decisión consciente, no un descuido.                      |
| Acceso administrativo exclusivamente vía Tailscale      | Parcial                                    | Tailscale ya existe para administración (`02-acceso-remoto-vpn.md`), pero el puerto 22 sigue expuesto también en la LAN del lab por el mismo motivo que en el escenario 01. |
| Alerta correlacionada de umbral (Wazuh activo-response) | No aplicada                                | Se podría sumar una regla de active response que bloquee la IP origen automáticamente tras superar un umbral de fallos. Queda como mejora futura.                           |

> Nota general: en este escenario específico **no se aplicó ninguna mitigación activa durante la prueba** (ni bloqueo, ni throttling) — consistente con el criterio documentado en `mitre-mapping.md`: el objetivo del lab es observación y correlación, no contención.

## Checklist de verificación

- [ ] Las alertas de fuerza bruta SSH aparecen en el dashboard de Wazuh dentro de los segundos posteriores a los intentos.
- [ ] La IP origen registrada coincide con la de Kali.
- [ ] Los usuarios probados según Wazuh coinciden con la lista usada en `ataque.md`.
- [ ] Si hubo autenticación exitosa, se generó un evento diferenciado de éxito posterior a la racha de fallos.
- [ ] No se generaron falsos positivos por accesos legítimos (ej. administración vía Tailscale) durante la ventana del ataque.

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/videos/03/wazuh.png" width="600"> </p>
