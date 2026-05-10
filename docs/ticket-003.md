# Ticket #003 — Error establishing a database connection

## Fecha: 2026-05-09
## Estado: ✅ RESUELTO
## Severidad: CRÍTICA — sitio completamente inaccesible

---

## Reporte del cliente
"Mi sitio muestra 'Error establishing a database connection'
en el frontend. En wp-admin dice 'One or more database tables
are unavailable. The database may need to be repaired.'
No hice ningún cambio reciente."

---

## Entorno
- WordPress: 6.x (Docker local)
- MySQL: 8.0 (contenedor separado wordpress-lab-db-1)
- Plugin: WooCommerce 10.7.0
- DB: wordpress | User: wpuser | Host: db

---

## Síntomas observados
- Frontend: "Error establishing a database connection"
- wp-admin: "One or more database tables are unavailable"
- Sitio completamente inaccesible para visitantes
- wp-admin parcialmente accesible con mensaje de error

---

## Diagnóstico paso a paso

### Paso 1 — Identificar contenedor de MySQL
docker ps --format "table {{.Names}}\t{{.Image}}"
→ wordpress-lab-db-1 (mysql:8.0)
→ IMPORTANTE: mysql client NO está en el contenedor
  de WordPress — está en el contenedor de MySQL

### Paso 2 — Verificar opciones críticas en DB
docker exec wordpress-lab-db-1 \
  mysql -u wpuser -pwppass123 wordpress \
  -e "SELECT option_name, option_value
      FROM wp_options
      WHERE option_name IN ('siteurl','home');"
→ PROBLEMA ENCONTRADO:
  siteurl = (vacío)
  home    = http://localhost:8080

### Paso 3 — Verificar engine de la tabla
docker exec wordpress-lab-db-1 \
  mysql -u wpuser -pwppass123 wordpress \
  -e "SHOW TABLE STATUS LIKE 'wp_options';"
→ Engine: MyISAM (propenso a corrupción)
→ InnoDB es el engine recomendado para WordPress

---

## Causa raíz
Dos problemas combinados:
1. wp_options.siteurl vacío → WordPress no puede
   determinar la URL base del sitio → falla la
   inicialización completa
2. Engine MyISAM en wp_options → más vulnerable
   a corrupción que InnoDB (no tiene transacciones
   ni recuperación automática ante fallos)

---

## Solución aplicada

### Fix 1 — Restaurar siteurl:
docker exec wordpress-lab-db-1 \
  mysql -u wpuser -pwppass123 wordpress \
  -e "UPDATE wp_options
      SET option_value='http://localhost:8080'
      WHERE option_name='siteurl';"

### Fix 2 — Reparar tabla MyISAM:
docker exec wordpress-lab-db-1 \
  mysql -u wpuser -pwppass123 wordpress \
  -e "REPAIR TABLE wp_options;"
→ Resultado: status OK

### Fix 3 — Migrar a InnoDB (prevención):
docker exec wordpress-lab-db-1 \
  mysql -u wpuser -pwppass123 wordpress \
  -e "ALTER TABLE wp_options ENGINE=InnoDB;"

---

## Verificación del fix
✅ Frontend carga correctamente
✅ wp-admin accesible sin mensajes de error
✅ siteurl = http://localhost:8080 confirmado
✅ home = http://localhost:8080 confirmado
✅ wp_options Engine = InnoDB

---

## Lecciones aprendidas

1. mysql client NO está en el contenedor de WordPress.
   Siempre ejecutar comandos MySQL desde el contenedor
   de la DB: docker exec wordpress-lab-db-1 mysql ...

2. siteurl y home son las opciones más críticas de WP.
   Si siteurl está vacío o incorrecto, WordPress no
   puede inicializar y muestra "Error establishing
   a database connection" aunque la DB esté accesible.

3. MyISAM vs InnoDB:
   - MyISAM: sin transacciones, propenso a corrupción,
     requiere REPAIR TABLE manual
   - InnoDB: transacciones ACID, auto-recovery,
     recomendado para todas las tablas de WordPress

4. Diagnóstico correcto para "Error DB connection":
   a) Verificar credenciales en wp-config.php
   b) Verificar que el contenedor MySQL está corriendo
   c) Verificar siteurl y home en wp_options
   d) Verificar engine y estado de tablas críticas
   e) Ejecutar REPAIR TABLE si engine es MyISAM

5. WordPress tiene una URL de reparación nativa:
   http://tudominio.com/wp-admin/maint/repair.php
   (requiste definir WP_ALLOW_REPAIR = true en wp-config.php)

---

## Referencias
- https://wordpress.org/documentation/article/faq-troubleshooting/
- https://dev.mysql.com/doc/refman/8.0/en/repair-table.html
- https://wordpress.org/documentation/article/creating-database-for-wordpress/
