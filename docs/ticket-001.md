# Ticket #001 — White Screen of Death (WSOD) por Memory Limit

## Fecha: 2026-05-07
## Estado: ✅ RESUELTO
## Severidad: CRÍTICA — sitio completamente inaccesible

---

## Reporte del cliente
"Mi sitio muestra pantalla blanca / error 500 al navegar
a WooCommerce → Settings. El problema apareció de repente."

---

## Entorno
- WordPress: 6.x (Docker local)
- Plugin: WooCommerce 10.7.0
- PHP Memory Limit al momento del error: 8M
- Servidor: Apache + PHP-FPM en contenedor Docker

---

## Pasos para reproducir
1. Definir memory_limit = 8M en php.ini
2. Definir WP_MEMORY_LIMIT = '8M' en wp-config.php
3. Reiniciar servidor web
4. Navegar a: /wp-admin/admin.php?page=wc-settings
5. Resultado: HTTP 500 — pantalla blanca total

---

## Síntomas observados
- HTTP 500 Internal Server Error en wp-admin
- Browser: "Parece que hay un problema con este sitio"
- debug.log: solo muestra Notice de textdomain (no el fatal)
- docker logs: PHP Fatal error confirmado

---

## Error exacto (docker logs)
PHP Fatal error: Allowed memory size of 8388608 bytes
exhausted (tried to allocate 442368 bytes) in
/var/www/html/wp-content/plugins/woocommerce/vendor/
jetpack-autoloader/class-autoloader-locator.php on line 83

PHP Fatal error: Allowed memory size of 8388608 bytes
exhausted (tried to allocate 65536 bytes) in
/var/www/html/wp-includes/class-wp-recovery-mode.php
on line 182

---

## Causa raíz
PHP memory_limit insuficiente (8M) para cargar WooCommerce.
WooCommerce requiere mínimo 128M, recomendado 256M.

El fatal error ocurre ANTES de que WordPress inicialice
su sistema de logging, por eso debug.log no lo captura.
La evidencia real está en los logs del servidor (Apache/PHP-FPM).

LECCIÓN CLAVE: Cuando debug.log está vacío pero hay HTTP 500,
revisar SIEMPRE los logs del servidor web, no solo los de WP.

---

## Diagnóstico paso a paso

### Paso 1 — Revisar debug.log
docker exec wordpress-lab-wordpress-1 \
  cat /var/www/html/wp-content/debug.log | tail -20
→ Solo muestra Notice. No hay fatal. Pista: el error
  ocurre antes de que WP inicie.

### Paso 2 — Revisar logs del servidor
docker logs wordpress-lab-wordpress-1 2>&1 | \
  grep -i "memory|fatal|allowed" | tail -20
→ AQUÍ está el fatal error real con archivo y línea exacta.

### Paso 3 — Confirmar memory limit
docker exec wordpress-lab-wordpress-1 \
  php -r "echo ini_get('memory_limit');"
→ Resultado: 8M — causa confirmada.

---

## Solución aplicada

### En php.ini (obligatorio — tiene prioridad):
memory_limit = 256M
# Archivo: /usr/local/etc/php/conf.d/memory-limit.ini

### En wp-config.php (recomendado como respaldo):
define('WP_MEMORY_LIMIT', '256M');

### Comandos ejecutados:
sed -i "s/memory_limit = 8M/memory_limit = 256M/" \
  /usr/local/etc/php/conf.d/memory-limit.ini

sed -i "s/define('WP_MEMORY_LIMIT', '8M')/define('WP_MEMORY_LIMIT', '256M')/" \
  /var/www/html/wp-config.php

docker restart wordpress-lab-wordpress-1

---

## Verificación del fix
✅ php -r "echo ini_get('memory_limit');" → 256M
✅ WooCommerce → Settings carga sin error
✅ debug.log sin nuevos errores críticos
✅ HTTP 200 en /wp-admin/admin.php?page=wc-settings

---

## Lecciones aprendidas

1. debug.log vacío ≠ sin errores
   Los fatal errors de memoria ocurren antes de que
   WordPress inicie su logger. Siempre revisar server logs.

2. WP_MEMORY_LIMIT en wp-config.php NO es suficiente
   si php.ini tiene un límite menor. php.ini tiene prioridad.

3. WooCommerce requiere mínimo 128M, recomendado 256M.
   Documentado en sus requisitos oficiales.

4. Constantes duplicadas generan warnings continuos.
   Siempre verificar con grep antes de agregar defines.

---

## Referencias
- https://wordpress.org/documentation/article/common-wordpress-errors/
- https://developer.wordpress.org/advanced-administration/debug/debug-wordpress/
- https://woocommerce.com/document/server-requirements/
