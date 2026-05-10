# Ticket #004 — Plugin con vulnerabilidad conocida (CVE)

## Fecha: 2026-05-09
## Estado: ✅ RESUELTO
## Severidad: ALTA — vulnerabilidad de seguridad activa

---

## Reporte del cliente
"Vi en las noticias que un plugin que tengo instalado
tiene una vulnerabilidad crítica. ¿Qué hago?
¿Estoy en riesgo? ¿Cómo lo soluciono?"

---

## Entorno
- WordPress: 6.9.4 (latest)
- Plugins afectados: Wordfence 8.2.0, Akismet 5.6
- Servidor: Apache + PHP-FPM en Docker

---

## CVE de referencia documentado
CVE-2020-35489 — Contact Form 7 < 5.3.2
Tipo: Unrestricted File Upload
Severidad: CRÍTICA (CVSS 10.0)
Impacto: Subida de archivos PHP maliciosos →
         ejecución de código remoto en el servidor
Estado en nuestro lab: Contact Form 7 v6.1.5
(versión actual, no afectada por este CVE)

---

## Vulnerabilidades encontradas en el lab
1. Wordfence 8.2.0 → actualización 8.2.1 disponible
2. Akismet 5.6 → actualización 5.7 disponible

---

## Diagnóstico paso a paso

### Paso 1 — Verificar WordPress core
wp core check-update
→ "WordPress is at the latest version" ✅

### Paso 2 — Identificar plugins desactualizados
wp plugin list --fields=name,status,version,update_version
→ wordfence: 8.2.0 → 8.2.1 ⚠️
→ akismet:   5.6   → 5.7   ⚠️

### Paso 3 — Verificar CVE activos
Fuentes de consulta para vulnerabilidades WordPress:
- https://www.wordfence.com/threat-intel/vulnerabilities/
- https://wpscan.com/plugins/
- https://cve.mitre.org/
- https://nvd.nist.gov/

### Paso 4 — Wordfence CLI scan
wp wordfence scan
→ "wordfence is not a registered wp command"
→ Wordfence no expone comandos WP-CLI
→ El scan se hace desde wp-admin → Wordfence → Scan

---

## Solución aplicada

### Actualizar plugins vulnerables vía WP-CLI:
wp plugin update wordfence akismet

→ wordfence: 8.2.0 → 8.2.1 ✅
→ akismet:   5.6   → 5.7   ✅

### Proceso de actualización segura (best practice):
1. Hacer backup ANTES de actualizar (ver Ticket #006)
2. Activar modo mantenimiento
3. Actualizar en staging primero si existe
4. Actualizar plugins uno por uno en producción
5. Verificar funcionalidad después de cada update
6. Desactivar modo mantenimiento

---

## Verificación post-update
✅ wp plugin list → sin update_version pendiente
✅ Wordfence activo y en versión 8.2.1
✅ Sitio funcionando correctamente
✅ wp-admin accesible sin errores

---

## Protocolo de respuesta ante CVE (para clientes)

### Paso 1 — Evaluar el riesgo:
¿El plugin afectado está activo? ¿En qué versión?
¿El CVE requiere autenticación para explotarse?
¿Hay exploit público disponible?

### Paso 2 — Acción inmediata si hay riesgo alto:
- Desactivar el plugin afectado temporalmente
- Bloquear el vector de ataque en WAF/Wordfence
- Aplicar la actualización disponible

### Paso 3 — Si no hay update disponible:
- Desactivar y buscar alternativa
- Contactar al desarrollador del plugin
- Reportar en wordpress.org/support

### Paso 4 — Verificar si el sitio fue comprometido:
- Revisar logs de acceso de Apache
- Ejecutar scan de Wordfence completo
- Verificar integridad de archivos core
  wp core verify-checksums
- Revisar usuarios administradores no reconocidos
  wp user list --role=administrator

---

## Comandos útiles para auditoría de seguridad

# Verificar integridad de WordPress core
wp core verify-checksums --allow-root --path=/var/www/html

# Listar administradores
wp user list --role=administrator --allow-root --path=/var/www/html

# Ver últimos logins fallidos (requiere plugin)
# Wordfence → Login Security → Login attempts

# Actualizar todos los plugins de una vez
wp plugin update --all --allow-root --path=/var/www/html

---

## Lecciones aprendidas

1. Mantener plugins actualizados es la primera línea
   de defensa. La mayoría de sitios hackeados usan
   plugins o themes desactualizados.

2. WP-CLI permite auditar y actualizar plugins
   en segundos, sin tocar el wp-admin.
   Ideal para gestión de múltiples sitios.

3. Wordfence no tiene comandos WP-CLI propios.
   El scan se ejecuta desde wp-admin o mediante
   la API REST de Wordfence si está configurada.

4. CVE-2020-35489 (CF7) es un ejemplo real de cómo
   una vulnerabilidad de file upload puede comprometer
   completamente un servidor. Siempre verificar
   que los formularios validen tipo MIME real,
   no solo la extensión del archivo.

5. Fuentes de inteligencia de vulnerabilidades WP:
   → wordfence.com/threat-intel/vulnerabilities
   → wpscan.com/plugins
   → patchstack.com/database
   → nvd.nist.gov

---

## Referencias
- https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-35489
- https://www.wordfence.com/threat-intel/vulnerabilities/
- https://developer.wordpress.org/cli/commands/plugin/update/
- https://wordpress.org/documentation/article/hardening-wordpress/
