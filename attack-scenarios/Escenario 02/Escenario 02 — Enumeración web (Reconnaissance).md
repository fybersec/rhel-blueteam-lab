## Objetivo de la técnica

Enumerar en detalle la aplicación web Mutillidae II expuesta en el puerto 8080/tcp del servidor (identificado en el escenario 01): rutas y directorios existentes, módulos habilitados, parámetros de entrada y tecnología subyacente (versión de Apache, PHP, cabeceras). El resultado de este escenario es lo que define la superficie de ataque concreta para los escenarios 04 en adelante.

## Herramientas

- **Nmap** (scripts NSE, categoría `http-*`).
- **Gobuster** — fuerza bruta de directorios/rutas.
- **Nikto** — escáner de vulnerabilidades web genérico.
- **WFuzz / FFuF** — fuzzing de parámetros y rutas.

## Comandos ejecutados

### 1. Scripts NSE de Nmap sobre el puerto 8080

```bash
nmap -p 8080 --script=http-enum,http-title,http-headers,http-methods <IP_DEL_SERVIDOR> -oN enum_nmap.txt
```

- `http-enum`: enumera rutas comunes conocidas para aplicaciones/CMS.
- `http-title`, `http-headers`, `http-methods`: fingerprinting básico del servidor y métodos HTTP habilitados.

### 2. Descubrimiento de directorios con Gobuster

```bash
gobuster dir -u http://<IP_DEL_SERVIDOR>:8080 -w /usr/share/wordlists/dirb/common.txt -x php -o gobuster_8080.txt
```

- `-w`: wordlist de rutas comunes.
- `-x php`: agrega la extensión `.php`, relevante para Mutillidae II.

### 3. Escaneo de vulnerabilidades con Nikto

```bash
nikto -h http://<IP_DEL_SERVIDOR>:8080 -o nikto_8080.txt
```

Identifica configuraciones inseguras, archivos sensibles expuestos y cabeceras de seguridad ausentes.

### 4. Fuzzing de parámetros sobre módulos identificados

```bash
ffuf -u "http://<IP_DEL_SERVIDOR>:8080/mutillidae/index.php?page=FUZZ" -w /usr/share/wordlists/dirb/common.txt -o ffuf_params.txt
```

Fuzzing dirigido específicamente al parámetro `page` de Mutillidae II, para identificar módulos/vistas accesibles más allá de los enlazados desde el menú de la aplicación.

## Resultado observado

- **Rutas/módulos identificados:** estructura estándar de Mutillidae II (`/mutillidae/`), con módulos de SQL Injection, XSS, IDOR y DNS Lookup (candidato a Command Injection) accesibles según lo anticipado en `mitre-mapping.md`.
- **Tecnología detrás:** Apache/httpd + PHP, según cabeceras y comportamiento de los scripts NSE.
- **Candidatos a explotación priorizados:**
  - Módulos de SQL Injection y XSS con parámetros por **GET** — priorizados para los escenarios 04 y 06 por quedar visibles tanto en `access_log` como para Suricata (ver nota transversal en `mitre-mapping.md` sobre visibilidad de POST vs GET).
  - Módulo de DNS Lookup — candidato a Command Injection (escenario 07), pendiente de confirmar accesibilidad real según el nivel de seguridad configurado en la instancia.

## Técnica MITRE ATT&CK asociada

- **Táctica:** Reconnaissance
- **Técnica:** Active Scanning: Vulnerability Scanning — [T1595.002](https://attack.mitre.org/techniques/T1595/002/)
