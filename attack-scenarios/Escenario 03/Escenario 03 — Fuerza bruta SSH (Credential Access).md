## Objetivo de la técnica

Obtener credenciales válidas del servicio SSH expuesto en el puerto 22/tcp del servidor, identificado en el escenario 01 como único punto de entrada administrativo. Un acceso exitoso acá representaría el vector de acceso inicial administrativo del ciclo de ataque.

## Herramienta

- **Hydra** (incluido en Kali por defecto).

## Preparación

- **Lista de usuarios:** archivo acotado con nombres de usuario plausibles para el entorno del lab (ej. usuario administrativo documentado en `01-instalacion-base.md` + variantes comunes como `admin`, `root`).
- **Lista de contraseñas:** diccionario de referencia (`rockyou.txt` o un subconjunto reducido para acotar el tiempo de ejecución del ejercicio).

> Nota: al ser un entorno de lab controlado, se conoce de antemano si la contraseña real está o no incluida en el diccionario usado, lo cual condiciona el resultado esperado (ver `respuesta.md` para la lectura de ese resultado desde el lado defensivo).

## Comando ejecutado

```bash
hydra -L usuarios.txt -P contraseñas.txt ssh://<IP_DEL_SERVIDOR> -t 4 -V
```

- `-L usuarios.txt`: lista de usuarios a probar.
- `-P contraseñas.txt`: diccionario de contraseñas a probar contra cada usuario.
- `-t 4`: número de conexiones en paralelo — acotado deliberadamente para no saturar el servicio ni generar un volumen de intentos que distorsione la lectura de la detección.
- `-V`: modo verboso, muestra cada intento usuario/contraseña en tiempo real.

## Resultado observado

- Volumen de intentos: cientos de combinaciones usuario/contraseña contra el puerto 22 en la ventana de ejecución.
- Resultado de autenticación: **completar según el resultado real del despliegue** — si el usuario/contraseña administrativo está dentro del diccionario probado, Hydra reporta la combinación válida; si no, el ataque agota el diccionario sin éxito.
- Independientemente del resultado final, el valor del escenario para el ciclo de blue team es el mismo: generar el patrón de autenticación fallida repetida que debe disparar la detección documentada en `respuesta.md`.

## Técnica MITRE ATT&CK asociada

- **Táctica:** Credential Access
- **Técnica:** Brute Force: Password Guessing — [T1110.001](https://attack.mitre.org/techniques/T1110/001/)
