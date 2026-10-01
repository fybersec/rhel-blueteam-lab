## Enfoque general

Se despliega **OWASP Mutillidae II** como servicio web objetivo, en reemplazo del webapp propio y de DVWA usados en iteraciones previas de esta fase. Se instala **nativo** sobre `httpd` en el mismo servidor RHEL, siguiendo la misma lógica de arquitectura que Suricata (`06-despliegue-suricata.md`) y que los despliegues web anteriores: el log de acceso de Apache queda en `/var/log/httpd/access_log`, y ese mismo archivo es la fuente que el agente de Wazuh ya lee (ver `05-configuracion-agente-wazuh.md`).

Mutillidae II es un proyecto público y de código abierto que cubre más de 40 vulnerabilidades mapeadas contra las categorías del OWASP Top Ten (2007, 2010, 2013 y 2017), lo que da una superficie mucho más amplia de escenarios de ataque que el webapp propio o DVWA para los fines de `attack-scenarios/mitre-mapping.md`.

## Por qué Mutillidae II (y no el webapp propio o DVWA)

| Opción                      | Ventaja                                                                                                                                                                                         | Desventaja                                                                                                                               |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **Mutillidae II (elegido)** | Público, mantenido activamente, cubre múltiples categorías del OWASP Top Ten (SQLi, XSS, IDOR, LFI, Command Injection, entre otras) en un solo despliegue, con niveles de seguridad ajustables. | Documentación oficial de instalación (`README-INSTALLATION.md`) son videos hechos sobre Ubuntu, sin pasos escritos para la familia RHEL. |
| Webapp propio               | Control total sobre el código y las vulnerabilidades introducidas.                                                                                                                              | Solo cubre dos vulnerabilidades (SQLi y fuerza bruta); no es un recurso público citable.                                                 |
| DVWA                        | Público, ampliamente documentado, instalación más simple.                                                                                                                                       | Cobertura de vulnerabilidades más acotada que Mutillidae.                                                                                |

## Recurso público utilizado

| Recurso                              | Enlace                                      |
| ------------------------------------ | ------------------------------------------- |
| Repositorio oficial de Mutillidae II | https://github.com/webpwnized/mutillidae    |
| Link de youtube                      | https://www.youtube.com/watch?v=PoYmEQvggUU |

## Infraestructura

|Endpoint|Descripción|
|---|---|
|rhel-blueteam-lab (RHEL 8.10)|Mismo servidor donde corren el agente de Wazuh y Suricata. Mutillidae corre sobre `httpd` en el puerto **8080** (ver sección de cambio de puerto), monitoreado por ambos.|

## Requisitos previos

- [ ] Agente de Wazuh ya instalado y en estado `active (running)` (ver `05-configuracion-agente-wazuh.md`).
- [ ] Suricata ya instalado y en estado `active (running)` (ver `06-despliegue-suricata.md`).
- [ ] Snapshot de la VM tomado antes de empezar.
- [ ] Servidor accesible **solo desde la red del laboratorio**. Mutillidae es vulnerable a propósito y no debe exponerse a Internet.

## Instalación de Apache y PHP

RHEL nombra el paquete del servidor web `httpd`, no `apache2`, y no existe un paquete equivalente a `libapache2-mod-php`: en RHEL, `httpd` ejecuta PHP a través de `php-fpm`.

```bash
sudo dnf install -y httpd
``` 

Agregar o editar: 
```bash 
sudo vi /etc/httpd/conf/httpd.conf

<Directory "/var/www/html">
    AllowOverride All
    Require all granted
</Directory>
```

```bash 
sudo systemctl restart httpd
sudo systemctl enable httpd
```

```bash
sudo dnf install -y php php-mysqlnd mariadb-server php-fpm
sudo systemctl enable --now php-fpm mariadb
```

RHEL 8 trae PHP 7.2 por defecto. Se intentó habilitar un módulo más reciente, pero en este entorno el flujo `php:8.1` no estaba disponible en los repositorios habilitados:

```bash
sudo dnf module reset php -y
sudo dnf module enable php:7.2 -y
# Error: Problemas en la petición: módulos o grupos que faltan: php:8.1
```

Se continuó con **PHP 7.2.24** (versión provista por el flujo `php:7.2` por defecto de RHEL 8.10), sin bloquear la instalación.

Módulos adicionales requeridos por distintos módulos de Mutillidae:

```bash
sudo dnf install -y php-cli php-curl php-mbstring php-xml php-gd
```

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/07-Webapp/1.png" width="600"> </p>
<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/07-Webapp/2.png" width="600"> </p>

## Base de datos (MariaDB)

A diferencia de MySQL Server de Oracle (usado en la guía de referencia para Ubuntu), MariaDB en RHEL no requiere el ajuste de `plugin='mysql_native_password'` para que la autenticación por contraseña funcione, aunque tampoco genera error si se ejecuta.

```bash
sudo mysql -u root
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'mutillidae';
FLUSH PRIVILEGES;
EXIT;
```

> **Decisión de laboratorio:** se usó `root` como usuario de conexión de la aplicación, en vez de un usuario dedicado. Aceptable en un lab aislado; en un entorno con mayores exigencias de higiene se recomienda un usuario `mutillidae` con privilegios acotados a su propia base, siguiendo el mismo criterio aplicado en el despliegue del webapp propio (`portal_user` en la fase anterior).

## Descarga de Mutillidae II

```bash
cd /var/www/html
sudo git clone https://github.com/webpwnized/mutillidae.git
```

El código fuente del proyecto vive en el subdirectorio `src/` del repositorio, no en la raíz clonada. Se movió su contenido para que la aplicación quede servida directamente en la raíz web:

```bash
sudo mv mutillidae/src/* .
sudo chown -R apache:apache /var/www/html/mutillidae
```

Tras validar el sitio funcionando dentro de `/mutillidae/`, se decidió aplanar la estructura para servir la aplicación directamente en la raíz (`/var/www/html/`), evitando el prefijo `/mutillidae/` en cada URL:

```bash
sudo mv mutillidae/* .
```

> **Nota de troubleshooting:** el primer intento (`mv mutillidae/* .` sin `sudo`) falló con "Permiso denegado" en todos los archivos, porque quedaron propiedad de `apache:apache` tras el `chown` anterior. Repetir el comando con `sudo` lo resolvió. Los archivos ocultos (`.git`, `.github`, `.gitignore`) no se movieron con el glob `*` y quedaron en `mutillidae/`; como no son necesarios para servir la aplicación, el directorio remanente se eliminó con `sudo rm -rf mutillidae/`.

## SELinux y firewall

```bash
sudo restorecon -Rv /var/www/html
sudo setsebool -P httpd_can_network_connect_db 1
sudo systemctl restart httpd php-fpm
```

## Configuración de la base de datos de la aplicación

Al acceder por primera vez, Mutillidae reporta que la base de datos está fuera de línea. El diagnóstico entregado por la propia aplicación (`database-offline.php`) fue clave para resolverlo en dos pasos:

**1. Credenciales incorrectas.** El primer error indicaba `Access denied for user 'root'@'localhost'`, porque las credenciales por defecto de `includes/database-config.inc` no coincidían con la contraseña de `root` fijada en MariaDB. Se corrigió alineando la contraseña de `root` en MariaDB con la que la aplicación espera (`mutillidae`, ver sección de base de datos arriba).

**2. Base de datos inexistente.** Resuelto el paso anterior, el error cambió a `Unable to select default database mutillidae` — la conexión ya era válida, pero la base `mutillidae` no existía:

```bash
sudo mysql -u root -p
Enter password: mutillidae
```

```sql
CREATE DATABASE mutillidae;
GRANT ALL PRIVILEGES ON mutillidae.* TO 'root'@'localhost';
FLUSH PRIVILEGES;
```

**3. Tablas inexistentes.** Con la base ya creada, el error cambió una vez más a `Table 'mutillidae.blogs_table' doesn't exist`. La base existía, pero vacía. Se resolvió otorgando privilegios globales al usuario de conexión (necesarios para que el propio script de setup de la aplicación pueda crear las tablas) y ejecutando el asistente de configuración de la aplicación:

```sql
GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

Luego, desde el navegador, se accedió al enlace de configuración que la propia página de error ofrece (**setup/reset the DB**), lo que creó el esquema completo de tablas dentro de `mutillidae`.

## Cambio de puerto (8080)

El servidor ya aloja el dashboard de Wazuh en el puerto 443. Al acceder a Mutillidae desde otra máquina por `http://<IP>/` (puerto 80), el navegador redirigía al 443 de Wazuh. Para evitar el conflicto, se movió Apache al puerto **8080**.

```bash
sudo vi /etc/httpd/conf/httpd.conf
```

```apache
Listen 8080
```

SELinix no permite por defecto que `httpd` escuche en puertos fuera de su lista autorizada (`http_port_t`). Este paso es específico de RHEL y no tiene equivalente en la guía de Ubuntu:

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
sudo systemctl restart httpd
```

Verificación:

```bash
sudo systemctl status httpd --no-pager
sudo ss -tlnp | grep 8080
```

## Acceso final

```
http://<IP_DEL_SERVIDOR>:8080/
```
Click en **Click here**

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/07-Webapp/6.png" width="600"> </p>

## Integración con el agente de Wazuh (`ossec.conf`)

Se reutiliza el mismo bloque `<localfile>` sobre `/var/log/httpd/access_log` ya definido para los despliegues web anteriores (webapp propio, DVWA). Como el archivo de log no cambia con el cambio de puerto de Apache, no fue necesario modificar `ossec.conf`:

```bash
sudo vi /var/ossec/etc/ossec.conf
```

```xml
<ossec_config>
  <localfile>
    <log_format>apache</log_format>
    <location>/var/log/httpd/access_log</location>
  </localfile>
</ossec_config>
```

Si no existiera (por ejemplo, en una instalación limpia que arranca directo con Mutillidae), se agrega siguiendo el mismo criterio documentado en `06-despliegue-suricata.md` — en su propio bloque `<ossec_config>` independiente:

```bash
sudo systemctl restart wazuh-agent
```

## Verificación post-instalación

- [ ] `systemctl status httpd`, `php-fpm` y `mariadb` en `active (running)`.
- [ ] `ss -tlnp | grep 8080` muestra a `httpd` escuchando en el puerto 8080.
- [ ] El puerto 8080 aparece permitido en `firewall-cmd --list-services` / `--list-ports`.
- [ ] `http://<IP_DEL_SERVIDOR>:8080/` carga la página de inicio de Mutillidae sin el mensaje "Database Offline".
- [ ] Sin denegaciones de SELinux relevantes en `/var/log/audit/audit.log` (`sudo ausearch -m avc -ts recent`).
- [ ] `/var/log/httpd/access_log` recibe eventos nuevos al navegar la aplicación.
- [ ] `systemctl status wazuh-agent` en `active (running)`, sin errores de configuración.

> Las pruebas de detección (SQLi, IDOR, XSS, LFI, Command Injection y su correlación con alertas de Wazuh/Suricata) se documentan por separado en `attack-scenarios/`, junto con su mapeo a MITRE ATT&CK.

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/07-Webapp/4.png" width="600"> </p>

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/07-Webapp/5.png" width="600"> </p>
