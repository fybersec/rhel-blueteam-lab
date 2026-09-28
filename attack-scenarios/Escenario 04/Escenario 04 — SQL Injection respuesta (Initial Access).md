## Fuente de detección esperada

Según `attack-scenarios/mitre-mapping.md`: **Suricata** (el payload viaja por GET/URL) + **Wazuh** (`access_log` de Apache). Este escenario se ejecutó deliberadamente por GET, priorizando el vector con mejor visibilidad para ambas fuentes — de haber sido explotable solo por POST, la detección hubiera recaído únicamente en Wazuh, según el criterio transversal documentado en `mitre-mapping.md`.

## Comportamiento observado en Suricata / Wazuh

**Suricata** dispara firmas de la categoría `ET WEB_SPECIFIC_APPS` / `ET WEB_SERVER` ante los patrones característicos de SQL Injection en la URL (comillas simples, `UNION SELECT`, comentarios SQL `--`, etc.), tanto en el payload manual de confirmación como en el volumen generado por SQLmap.

**Alerta representativa de Suricata:**

```
[**] [1:2019401:5] ET WEB_SPECIFIC_APPS SQL Injection Attempt in URI [**]
[Classification: Web Application Attack] [Priority: 1]
{TCP} <IP_KALI>:XXXXX -> <IP_SERVIDOR>:8080
```

```
[**] [1:2013504:5] ET WEB_SERVER SQLMAP SQL Injection Scan Detected [**]
[Classification: Web Application Attack] [Priority: 1]
{TCP} <IP_KALI>:XXXXX -> <IP_SERVIDOR>:8080
```

**Wazuh** correlaciona el `access_log` de Apache, que registra la URL completa con el payload SQL incluido en cada petición — a diferencia del escenario 03, acá el contenido del payload en sí mismo es evidencia, no solo el volumen de peticiones.

**Alerta representativa de Wazuh:**

```
** Alert <timestamp>: - web,accesslog,attack,
Rule: 31103 (level 6) -> 'Common web attack (SQL injection).'
Src IP: <IP_KALI>
URL: /mutillidae/index.php?page=user-info.php&username=admin'--&password=
```

Ambas fuentes son visibles en el dashboard de Wazuh en **Threat Hunting → Events**, filtrando por `rule.groups:suricata` o `rule.groups:web,sql_injection`.

## Limitaciones observadas

- La detección de Suricata para este escenario depende **exclusivamente de que el vector sea GET**; si Mutillidae II solo hubiera sido explotable por POST en este parámetro, el payload no habría quedado visible en la URL y la firma de red no se hubiera disparado — la detección habría recaído enteramente en Wazuh vía `access_log`.
- Si el atacante ofusca el payload (encoding, comentarios alternativos, técnicas de evasión de WAF) o usa `--random-agent` en SQLmap, se reduce la efectividad tanto de la firma de Suricata como de la correlación por patrón conocido en Wazuh.
- La extracción de datos en sí (el `--dump` de SQLmap) no genera una alerta diferenciada de "exfiltración" — Wazuh y Suricata detectan el patrón de inyección en la petición, no el contenido de la respuesta del servidor.

## Mitigación / hardening relacionado

| Medida                                                                        | Ya aplicada en este lab     | Notas                                                                                                                                   |
| ----------------------------------------------------------------------------- | --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Sanitización de entradas / consultas parametrizadas                           | No aplicada                 | Mutillidae II se mantiene deliberadamente vulnerable para sostener el ciclo de ataque completo del lab.                                 |
| WAF delante de Apache (ej. ModSecurity con reglas OWASP CRS)                  | No aplicada                 | Pendiente para una futura iteración — bloquearía el payload antes de llegar a la aplicación, en vez de solo detectarlo.                 |
| Principio de mínimo privilegio en la cuenta de base de datos                  | No aplicada                 | La cuenta de base de Mutillidae II mantiene privilegios amplios por defecto — limitarlos reduciría el impacto de una inyección exitosa. |
| Alerta correlacionada por umbral en Wazuh (agrupando intentos de SQLi por IP) | Parcial (regla por defecto) | La regla 31103 ya cubre el caso base; se podría afinar para elevar el nivel de alerta ante un volumen sostenido desde la misma IP.      |

## Checklist de verificación

- [ ] Las alertas de Suricata (`SQL Injection Attempt in URI` / `SQLMAP Scan Detected`) aparecen en el dashboard de Wazuh dentro de los segundos posteriores al ataque.
- [ ] La IP origen y el user-agent registrados coinciden con los de Kali/SQLmap.
- [ ] Los datos extraídos según `ataque.md` (tabla `accounts`) son consistentes con el esquema real de la base `mutillidae`.
- [ ] No se generaron falsos positivos por tráfico de Docker (`172.17.0.0/16`, `172.18.0.0/16`) durante la ventana del ataque.

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/videos/04/wazuh.png" width="600"> </p>
