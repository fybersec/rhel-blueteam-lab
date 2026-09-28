## Enfoque general

El servicio SSH es el único punto de entrada administrativo al servidor (`00-introduccion.md`), y queda expuesto en **dos zonas distintas** de firewalld (`hardening/firewalld/zonas-reglas.md`): la zona `internal` (LAN del lab, donde también está Kali) y la zona `trusted-vpn`. Este documento no toca el puerto ni las zonas ya definidas — endurece la configuración del propio demonio `sshd`, que aplica por igual a ambas rutas de acceso.

**Punto de partida (`01-preparacion-servidor.md`):** usuario `soc-admin`, grupo `wheel`, autenticación por password **o** clave. El hardening de este documento cierra esa ambigüedad: En este caso se debería mantener acceso por clave únicamente, pero con fin de realizar fuerza bruta exitoso, no aplicare esa configuración. 

|Opción|Ventaja|Desventaja|
|---|---|---|
|**Solo clave pública (elegido)**|Inmune a fuerza bruta y a diccionarios; el riesgo se traslada a proteger la clave privada, no a recordar/rotar una contraseña.|Si se pierde la clave privada sin backup, no hay forma de entrar por SSH (mitigado con el snapshot de la VM y acceso por consola de VMware como último recurso).|
|Password + clave (estado inicial)|Más simple de recuperar si se pierde la clave.|El servidor SSH queda expuesto a intentos de fuerza bruta desde la propia LAN — y Kali, que vive en esa misma LAN por diseño (`01-preparacion-servidor.md`), es justamente una máquina pensada para atacar. Mantener password activo en ese contexto es la superficie más floja del propio hardening.|
|Solo password con `fail2ban`|No depende de gestionar claves.|Sigue siendo una superficie de ataque activa; `fail2ban` mitiga pero no elimina el vector, y agrega un servicio más para mantener.|

## Configuración aplicada

```bash
sudo vi /etc/ssh/sshd_config
```

```sshconfig
# Autenticación
PermitRootLogin no
PasswordAuthentication yes #Con el fin de en /attacks-scenarios poder hacer fuerza bruta al ssh(22)
PubkeyAuthentication yes
PermitEmptyPasswords no
AuthenticationMethods publickey

# Superficie de ataque
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowTcpForwarding no

# Alcance de usuarios
AllowUsers soc-admin

# Sesiones inactivas
ClientAliveInterval 300
ClientAliveCountMax 2
```

| Directiva                                     | Valor       | Justificación                                                                                                                                                                                                                |
| --------------------------------------------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `PermitRootLogin`                             | `no`        | El usuario administrativo es `soc-admin` vía `wheel` (`01-preparacion-servidor.md`); no hay razón operativa para loguear como `root` directamente.                                                                           |
| `AuthenticationMethods`                       | `publickey` | Refuerza explícitamente que ningún otro método (keyboard-interactive, etc.) sirve de puerta trasera.                                                                                                                         |
| `MaxAuthTries`                                | `3`         | Limita intentos por conexión antes de cortar — reduce ventana de prueba automatizada aun con clave como único método.                                                                                                        |
| `AllowUsers soc-admin`                        | —           | Acota SSH al único usuario administrativo real del lab; cualquier otra cuenta del sistema queda excluida de este vector aunque exista.                                                                                       |
| `X11Forwarding` / `AllowTcpForwarding`        | `no`        | No hay caso de uso en este lab para forwarding gráfico ni túneles arbitrarios sobre SSH; se cierra por defecto.                                                                                                              |
| `ClientAliveInterval` / `ClientAliveCountMax` | `300` / `2` | Cierra sesiones colgadas tras ~10 minutos de inactividad — relevante porque el acceso llega tanto desde la LAN como desde Tailscale, y una sesión olvidada abierta en cualquiera de las dos rutas es superficie innecesaria. |

## Aplicar los cambios

```bash
sudo sshd -t                      # valida sintaxis antes de reiniciar
sudo systemctl restart sshd
```

> **Práctica de seguridad recomendada:** antes de cerrar la sesión actual, abrir una **segunda** conexión SSH nueva para confirmar que el acceso por clave funciona con la configuración ya aplicada. Si algo falla, la sesión original sigue abierta para revertir sin necesidad de recurrir a la consola de VMware. Esta es la misma lógica de "no romper lo que ya funciona" detrás del snapshot recomendado en `01-preparacion-servidor.md`.

## Verificación post-hardening

- [ ] `sudo sshd -t` no reporta errores de sintaxis.
- [ ] Login por clave funciona desde la LAN (Kali/máquina de administración).
- [ ] Login por clave funciona desde Tailscale (colaborador).
- [ ] Un intento de login con password es rechazado (`Permission denied (publickey)`).
- [ ] Un intento de login como `root` es rechazado antes de pedir autenticación.
- [ ] Una sesión SSH inactiva se cierra sola después del tiempo configurado (`ClientAliveInterval` × `ClientAliveCountMax`).
- [ ] Ningún usuario fuera de `soc-admin` puede autenticarse por SSH.