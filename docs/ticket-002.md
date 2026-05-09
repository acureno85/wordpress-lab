# Ticket #002 — Permalinks rotos / 404 en páginas de WooCommerce

## Fecha: 2026-05-07
## Estado: ✅ RESUELTO
## Severidad: ALTA — Shop y páginas de WooCommerce inaccesibles

---

## Reporte del cliente
"Mi tienda en /shop/ muestra error 404 Not Found.
El wp-admin funciona bien pero las URLs amigables
no cargan en ninguna página."

---

## Entorno
- WordPress: 6.x (Docker local)
- Plugin: WooCommerce 10.7.0
- Servidor: Apache 2.4.66 (Debian) en Docker
- Permalink structure: vacía al inicio

---

## Síntomas observados
- /shop/ → 404 Not Found
- Todas las URLs amigables fallan
- wp-admin carga correctamente
- WP-CLI warning: "missing permalink_structure option"
- Apache warning: "Regenerating .htaccess requires
  special configuration"

---

## Diagnóstico paso a paso

### Paso 1 — Verificar permalink structure
wp option get permalink_structure
→ Vacío = modo Plain activo, URLs amigables desactivadas

### Paso 2 — Intentar flush
wp rewrite flush --hard
→ Warning: missing permalink_structure
→ Warning: .htaccess requires special configuration

### Paso 3 — Verificar Apache
grep -r 'AllowOverride' /etc/apache2/
→ PROBLEMA: apache2.conf tiene AllowOverride None
  (sobrescribe docker-php.conf que sí tenía All)
→ mod_rewrite ya estaba habilitado (no era el problema)

### Paso 4 — Verificar .htaccess
cat /var/www/html/.htaccess
→ Archivo incompleto (heredoc falló por caracteres
  especiales ! en bash history expansion)

---

## Causa raíz
Dos problemas combinados:
1. AllowOverride None en apache2.conf impedía que
   Apache leyera el .htaccess de WordPress
2. permalink_structure vacía = WordPress en modo Plain

---

## Solución aplicada

### Fix 1 — Establecer permalink structure:
wp rewrite structure '/%postname%/'
wp rewrite flush --hard

### Fix 2 — Corregir AllowOverride:
sed -i 's/AllowOverride None/AllowOverride All/g' \
  /etc/apache2/apache2.conf

### Fix 3 — Recrear .htaccess correctamente:
cat > /var/www/html/.htaccess << 'HTACCESS'
# BEGIN WordPress
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
</IfModule>
# END WordPress
HTACCESS

### Fix 4 — Reiniciar Apache:
docker restart wordpress-lab-wordpress-1

---

## Verificación del fix
✅ http://localhost:8080/shop/ carga correctamente
✅ Query Monitor visible (barra superior)
✅ WooCommerce muestra "Tienda disponible próximamente"
✅ URLs amigables funcionando en todo el sitio

---

## Lecciones aprendidas

1. AllowOverride en apache2.conf tiene prioridad
   sobre configuraciones de directorios específicos.
   Siempre verificar la configuración global de Apache.

2. El .htaccess de WordPress es crítico para permalinks.
   Sin él, todas las URLs amigables fallan con 404.

3. wp rewrite flush --hard NO regenera .htaccess
   automáticamente si AllowOverride lo impide.
   Hay que crearlo manualmente.

4. Bash history expansion (!) puede corromper heredocs.
   Usar comillas simples en el delimitador: << 'EOF'
   o ejecutar dentro del contenedor directamente.

5. Diagnóstico correcto:
   404 en URLs amigables → revisar en este orden:
   a) permalink_structure (wp option get)
   b) .htaccess existe y está completo
   c) AllowOverride en Apache config
   d) mod_rewrite habilitado (a2enmod rewrite)

---

## Referencias
- https://wordpress.org/documentation/article/using-permalinks/
- https://httpd.apache.org/docs/2.4/mod/core.html#allowoverride
