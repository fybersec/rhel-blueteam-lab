## Contexto dentro del ciclo de ataque

Primer paso del ciclo documentado en `attack-scenarios/mitre-mapping.md`: antes de atacar cualquier servicio específico, hay que identificar qué está expuesto en `rhel-blueteam-lab`. Este escenario se ejecuta desde Linux Mint, contra la IP del servidor en la red LAN aislada del lab (sin salida a internet, según lo definido en `00-introduccion.md`).

## Objetivo de la técnica

Determinar qué puertos TCP/UDP están abiertos en el servidor y qué servicios corren detrás de ellos (versión, banner), para decidir el siguiente vector de ataque del ciclo (en este caso: SSH → escenario 02, y el puerto 8080 de Mutillidae II → escenario 03 en adelante).

## Herramienta

- **Nmap** (incluido en Kali por defecto).

## Comandos ejecutados

### 1. Descubrimiento inicial (TCP SYN scan, todos los puertos)

```bash
sudo nmap -sS -p- -T4 <IP_DEL_SERVIDOR> -oN escaneo_completo.txt
```

- `-sS`: SYN scan (half-open), más difícil de loguear a nivel de aplicación que un connect scan completo, pero perfectamente visible para un NIDS como Suricata.
- `-p-`: los 65535 puertos TCP, no solo el top-1000 por defecto — para no dejar fuera el puerto 8080 de Mutillidae.
- `-T4`: velocidad agresiva, deliberada para este ejercicio (ver nota de detección en `respuesta.md`).

### 2. Detección de versión y servicios sobre los puertos abiertos

```bash
nmap -sV -sC -p 22,8080 <IP_DEL_SERVIDOR> -oN escaneo_servicios.txt
```

- `-sV`: fingerprinting de versión de servicio.
- `-sC`: scripts NSE por defecto (categoría `default`), para banners más completos (ej. banner de SSH, cabeceras HTTP de Apache/Mutillidae).

### 3. Escaneo UDP acotado (referencia, no crítico para el ciclo)

```bash
sudo nmap -sU --top-ports 20 <IP_DEL_SERVIDOR> -oN escaneo_udp.txt
```

Se limita a los 20 puertos UDP más comunes por el costo de tiempo de un escaneo UDP completo; no se identificó ningún servicio UDP relevante para el resto del ciclo, por lo que no se profundizó más allá de este paso.

## Resultado observado

| Puerto | Estado | Servicio | Notas |
|---|---|---|---|
| 22/tcp | open | ssh (OpenSSH) | Único punto de entrada administrativo — objetivo de `02-fuerza-bruta-ssh` |
| 8080/tcp | open | http (Apache/httpd) | Mutillidae II, ver `07-despliegue-webapp.md` — objetivo de `03-enumeracion-web` en adelante |
| 443/tcp | open | https | Dashboard de Wazuh — **fuera de alcance de ataque**, es infraestructura de detección, no el objetivo |

> Nota: el puerto 443 aparece abierto porque corresponde al dashboard de Wazuh, no a un servicio del objetivo. No se ataca en ningún escenario — está documentado acá solo porque el escaneo lo revela.

## Técnica MITRE ATT&CK asociada

- **Táctica:** Reconnaissance
- **Técnica:** Active Scanning: Vulnerability Scanning / Scanning IP Blocks — [T1595](https://attack.mitre.org/techniques/T1595/)
