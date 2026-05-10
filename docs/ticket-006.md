# Ticket #006 — Backup y Restauración completa

## Fecha: 2026-05-09
## Estado: ✅ RESUELTO
## Severidad: PREVENTIVO — procedimiento estándar

---

## Reporte del cliente
"Necesito hacer un backup antes de cambios importantes.
Si algo sale mal, ¿cómo restauro exactamente como estaba?"

---

## Entorno
- WordPress: 6.9.4
- MySQL: 8.0 (contenedor wordpress-lab-db-1)
- Servidor: Apache + PHP-FPM en Docker

---

## Backup realizado

### Componente 1 — Base de datos:
docker exec wordpress-lab-db-1 \
  mysqldump -u wpuser -pwppass123 wordpress \
  > backups/db-backup-20260509-2211.sql
Tamaño: 635K
Contenido: Todas las tablas WordPress + WooCommerce

### Componente 2 — Archivos:
tar -czf /tmp/wp-content-backup.tar.gz \
  /var/www/html/wp-content/ \
  /var/www/html/wp-config.php
Tamaño: 50MB
Contenido: plugins + themes + uploads + wp-config.php

### Copiar backup al host:
docker cp wordpress-lab-wordpress-1:/tmp/[archivo] \
  ~/wordpress-lab/backups/

---

## Desastre simulado
DELETE FROM wp_posts WHERE post_status='publish';
UPDATE wp_options SET option_value=''
  WHERE option_name='blogname';

Resultado:
- posts_publicados: 0 (todos eliminados)
- blogname: vacío (nombre del sitio perdido)

---

## Restauración aplicada

### Paso 1 — Restaurar DB:
docker exec -i wordpress-lab-db-1 \
  mysql -u wpuser -pwppass123 wordpress \
  < backups/db-backup-20260509-2211.sql

### Paso 2 — Verificar datos:
SELECT COUNT(*) FROM wp_posts WHERE post_status='publish';
→ 8 posts recuperados ✅
SELECT option_value FROM wp_options
  WHERE option_name='blogname';
→ "WP LAB" restaurado ✅

### Paso 3 — Restaurar archivos:
docker cp backup.tar.gz wordpress-lab-wordpress-1:/tmp/
tar -xzf /tmp/backup.tar.gz -C /

---

## Verificación final
✅ 8 posts publicados recuperados
✅ blogname "WP LAB" restaurado
✅ wp-content completo restaurado
✅ wp-config.php restaurado
✅ Sitio funcionando correctamente

---

## Estrategia de backup recomendada para producción

### Regla 3-2-1:
- 3 copias del backup
- 2 medios diferentes (local + cloud)
- 1 copia offsite (S3, Google Drive, Dropbox)

### Frecuencia recomendada:
- DB: diario (automatizado con cron)
- Archivos: semanal (wp-content no cambia tanto)
- Backup completo: antes de CUALQUIER actualización

### Automatización con cron (ejemplo):
# Backup diario de DB a las 2am
0 2 * * * docker exec wordpress-lab-db-1 \
  mysqldump -u wpuser -pwppass123 wordpress \
  > /backups/db-$(date +\%Y\%m\%d).sql

### Plugins de backup recomendados:
- UpdraftPlus (gratuito, S3/Dropbox/GDrive)
- BackupBuddy (premium, más completo)
- Jetpack Backup (Automattic — tiempo real)

### Qué incluir en un backup completo:
✅ Base de datos (wp_* tablas)
✅ wp-content/uploads (media del cliente)
✅ wp-content/themes (themes activos y child)
✅ wp-content/plugins (plugins instalados)
✅ wp-config.php (configuración crítica)
❌ wp-core files (se pueden reinstalar con WP-CLI)
❌ wp-content/cache (se regenera automáticamente)

---

## Comandos de referencia rápida

# Backup DB completo
docker exec wordpress-lab-db-1 \
  mysqldump -u USER -pPASS DBNAME > backup.sql

# Restaurar DB
docker exec -i wordpress-lab-db-1 \
  mysql -u USER -pPASS DBNAME < backup.sql

# Backup archivos
tar -czf backup.tar.gz wp-content/ wp-config.php

# Restaurar archivos
tar -xzf backup.tar.gz -C /var/www/html/

# Backup con WP-CLI (alternativa)
wp db export backup.sql --allow-root
wp db import backup.sql --allow-root

---

## Lecciones aprendidas

1. Un backup tiene DOS partes: DB + archivos.
   Olvidar cualquiera de las dos = backup incompleto.
   La DB tiene el contenido. Los archivos tienen
   los media, plugins y themes.

2. mysqldump genera warning de PROCESS privilege.
   Es un warning, no un error — el backup se
   completa correctamente. En producción, crear
   un usuario con los permisos mínimos necesarios.

3. Siempre VERIFICAR el backup después de crearlo:
   - Abrir el .sql y confirmar que tiene datos
   - Hacer un test restore en entorno de staging
   - Un backup no verificado = no existe

4. docker cp es esencial para sacar archivos
   del contenedor al host. Sin este paso, el
   backup está atrapado dentro del contenedor
   y se pierde si el contenedor se elimina.

5. La restauración de DB usa mysql (no mysqldump)
   con redirección de entrada: < archivo.sql
   La dirección del operador importa:
   → Export: mysqldump ... > archivo.sql
   → Import: mysql ... < archivo.sql

---

## Referencias
- https://developer.wordpress.org/cli/commands/db/export/
- https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html
- https://updraftplus.com/
- https://jetpack.com/features/security/backup/
