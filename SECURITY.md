# Política de seguridad

**RHEL Blue Team Lab** es un laboratorio personal, educativo y aislado. Todo lo documentado en este repositorio (hardening, despliegue de Wazuh/Suricata, y en particular los escenarios de ataque contra OWASP Mutillidae II) se ejecutó en una red LAN privada, sin salida a internet durante las pruebas de ataque y sin exposición a producción.

**Nada de lo documentado acá está autorizado para usarse contra sistemas, redes o aplicaciones que no sean de tu propiedad o para los que no tengas autorización explícita y por escrito.** Mutillidae II es, por diseño, una aplicación vulnerable — desplegarla o replicar los escenarios de `attack-scenarios/` fuera de un entorno aislado y controlado es responsabilidad exclusiva de quien lo haga.

Este proyecto no ofrece garantías de seguridad, hardening certificado ni compliance con ningún benchmark formal (ver limitaciones en `docs/es/00-introduccion.md`).

## Alcance de esta política

Esta política cubre únicamente vulnerabilidades reales en el **código propio** de este repositorio — por ejemplo, en el servicio web propio documentado en `webapp/` (si aplica) o en scripts/configuraciones incluidos directamente acá.

No cubre:

- Vulnerabilidades de Wazuh, Suricata, Mutillidae II, o cualquier otro software de terceros usado en el lab — repórtalas en sus repositorios oficiales.
- El hecho de que Mutillidae II sea intencionalmente vulnerable; eso es su propósito documentado.
