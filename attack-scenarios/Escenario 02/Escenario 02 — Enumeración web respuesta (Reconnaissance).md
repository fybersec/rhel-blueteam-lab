## Fuente de detección esperada

Según `attack-scenarios/mitre-mapping.md`: **Wazuh** (`access_log` de Apache) + **Suricata** (firmas HTTP de Emerging Threats). A diferencia del escenario 01 (donde solo Suricata tenía visibilidad), acá la enumeración web genera tráfico HTTP en texto plano con peticiones y respuestas completas, por lo que ambas fuentes aportan señal en paralelo.

## Comportamiento observado en Suricata / Wazuh

**Suricata** dispara firmas de la categoría `ET WEB_SERVER` / `ET SCAN` ante el volumen y patrón de peticiones típico de un scanner (Gobuster, Nikto, WFuzz), incluyendo coincidencias por user-agent por defecto de estas herramientas cuando no se ofusca.

**Alerta representativa de Suricata:**

```
[**] [1:2101201:10] GPL WEB_SERVER 403 Forbidden [**]
[Classification: Attempted Information Leak] [Priority: 2]
{TCP} <IP_Linux Mint>:XXXXX -> <IP_SERVIDOR>:8080
```

```
[**] [1:2016153:5] ET WEB_SERVER Nikto Web App Scan Detected [**]
[Classification: Web Application Attack] [Priority: 1]
{TCP} <IP_Linux Mint>:XXXXX -> <IP_SERVIDOR>:8080
```

**Wazuh** correlaciona el `access_log` de Apache recolectado por el agente, y eleva el nivel de alerta ante el volumen de códigos `404`/`403` en poco tiempo desde el mismo origen — patrón característico de un directory brute-force.

**Alerta representativa de Wazuh:**

```
** Alert <timestamp>: - web,accesslog,
Rule: 31151 (level 8) -> 'Multiple web server 400 error codes from same source IP.'
Src IP: <IP_Linux Mint>
URL: /mutillidae/<ruta_probada>
```

Ambas fuentes son visibles en el dashboard de Wazuh en **Threat Hunting → Events**, filtrando por `rule.groups:suricata` (para las firmas de red) y `rule.groups:web` (para la correlación de `access_log`).

## Información recopilada sobre el posible atacante

- **IP origen:** la de la VM de Linux Mint (única IP en la red aislada del lab).
- **User-agent:** por defecto, Gobuster/Nikto/FFuF envían un user-agent identificable de la propia herramienta si no se lo modifica explícitamente — dato de alto valor para perfilar al atacante.
- **Patrón de peticiones:** alto volumen de rutas no existentes (`404`) en corto tiempo, seguido de picos puntuales de `200` sobre rutas reales de Mutillidae II — permite reconstruir qué rutas/módulos terminó descubriendo el atacante.
- **Parámetros probados:** el fuzzing sobre `?page=FUZZ` queda registrado en `access_log` con cada valor probado, lo que revela el intento de descubrir módulos no enlazados desde el menú principal.

## Limitaciones observadas

- El volumen de eventos generado por un fuzzing agresivo (Gobuster/FFuF con wordlists grandes) puede saturar de ruido el dashboard si no hay una regla de correlación por umbral — se vuelve más difícil distinguir una petición de reconocimiento puntual de una campaña de enumeración activa sin agrupar por IP y ventana de tiempo.
- Si el atacante rota o falsea el user-agent, se pierde ese dato como indicador — la detección pasa a depender exclusivamente del patrón de volumen/tasa de errores 404, que es más ruidoso.
- Las peticiones por **POST** (relevantes recién a partir de escenarios de explotación como el 04) no dejarían el mismo nivel de visibilidad para Suricata; en este escenario de enumeración el tráfico es mayormente GET, por lo que la limitación no aplica todavía, pero queda anotada como antecedente para los escenarios siguientes.

## Mitigación / hardening relacionado

| Medida                                                          | Ya aplicada en este lab | Notas                                                                                                                                 |
| ----------------------------------------------------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| Reducción de superficie de ataque (módulos innecesarios de Mutillidae desactivados) | No aplicada              | Mutillidae II se mantiene con su configuración por defecto deliberadamente, para sostener el ciclo de ataque completo del lab.       |
| Cabeceras de seguridad (CSP, X-Frame-Options, etc.)                | No aplicada              | Nikto reporta su ausencia como parte del resultado esperado del escenario — queda documentado como hallazgo, no como pendiente urgente. |
| Rate limiting / WAF delante de Apache                              | No aplicada              | Pendiente para una futura iteración — permitiría frenar un fuzzing agresivo sin bloquear tráfico legítimo.                          |
| Alerta correlacionada por umbral en Wazuh (agrupando 404 por IP)   | Parcial (regla por defecto) | La regla 31151 ya cubre el caso base; se podría afinar el umbral para reducir falsos positivos ante tráfico legítimo con errores ocasionales. |

## Checklist de verificación

- [ ] Las alertas de Suricata (`ET WEB_SERVER` / scan detectado) aparecen en el dashboard de Wazuh dentro de los segundos posteriores a la enumeración.
- [ ] La correlación de `access_log` en Wazuh muestra el volumen de códigos 404/403 esperado.
- [ ] La IP origen y el user-agent registrados coinciden con los de Kali/las herramientas usadas.
- [ ] Las rutas y módulos reportados como encontrados coinciden con los que efectivamente responden `200` en Mutillidae II.
- [ ] No se generaron falsos positivos por tráfico de Docker (`172.17.0.0/16`, `172.18.0.0/16`) durante la ventana de enumeración.

<p align="center"> <img src="captura de las alertas de enumeración web en el dashboard de Wazuh" width="600"> </p>
