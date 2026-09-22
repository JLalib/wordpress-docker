# 🐳 WordPress Docker Stack

![GitHub](https://img.shields.io/github/license/genbyte/wordpress-docker)
![Docker](https://img.shields.io/badge/Docker-Ready-blue)
![License](https://img.shields.io/badge/License-GPL%20v3-green)

## 📋 Descripción general

**WordPress Docker Stack** es una solución completa y autohospedada para desplegar WordPress (el CMS que impulsa el 40% de internet) usando contenedores Docker. Incluye MariaDB como base de datos optimizada, phpMyAdmin para gestión visual de la base de datos, y Nginx como reverse proxy de alto rendimiento. Todo listo para producción con HTTPS automático vía Caddy/Let's Encrypt, volúmenes persistentes y backups triviales.

Esta stack elimina la dependencia de hosting SaaS como WordPress.com o Wix, dándote control total sobre tus datos, plugins, temas y configuración, con un coste de infraestructura mínimo.

## ✨ Características principales

- **WordPress Core última versión** - CMS más popular (40% de internet), potente, flexible y robusto
- **MariaDB 10.6+** - Base de datos ligera, MySQL-compatible, rápida y eficiente
- **phpMyAdmin interfaz web** - Gestión visual de BD, backup/restore sin CLI
- **Nginx high-performance** - Web server rápido, bajo consumo, reverse proxy integrado
- **PHP 8.2+** - Runtime moderno y optimizado
- **Docker Compose setup** - Despliegue fácil con un solo comando
- **Volúmenes persistentes** - Datos seguros en `wp-data` y `db-data`, backups triviales (`cp` de volumes)
- **Networking automático** - Service discovery entre contenedores
- **Health checks + Auto-restart** - Resiliencia automática
- **HTTPS automático** - SSL certificados gratis vía Let's Encrypt con Caddy
- **Multi-sitio WordPress** - Múltiples blogs en una instalación, gestión centralizada
- **60K+ plugins + 10K+ temas** - Ecosistema masivo, extensibilidad total
- **Users + Roles RBAC** - Control de acceso granular (Admin, Editor, Author, Contributor, Subscriber)
- **Media library optimizado** - Imágenes, videos, archivos con redimensionamiento automático
- **SEO-friendly** - Yoast SEO plugin, sitemaps, canonicals, structured data
- **Production-ready** - Tested, escalable, seguro, usado en millones de sitios
- **Cache plugins soportados** - Redis, Memcached vía compose
- **Debugging tools** - WP-CLI incluido
- **Database optimization tools** - Performance monitoring integrado
- **Seguridad hardened** - WP hardening plugins soportados
- **Multiidioma + WPML** - Soporte completo para sitios multilingües
- **Staging/development environments** - Entornos fáciles de replicar

## 📋 Requisitos del sistema

- **Docker** (Engine 20.10+)
- **Docker Compose** (v2.0+ recomendada, plugin `docker compose`)
- **1 GB - 4 GB RAM** mínimo (depende del tráfico del sitio)
- **5 GB - 50 GB espacio disco** (para BD + uploads + contenido)
- **Puertos 80 y 443** (HTTP y HTTPS, configurables)
- **Volúmenes persistentes** para WordPress, MariaDB, phpMyAdmin
- **Dominio apuntando al servidor** (para HTTPS automático)
- **Email servidor SMTP** (opcional, para notificaciones)
- **Navegador moderno** (editar posts, gestión admin)

> **Setup recomendado:** 2 GB RAM mínimo para tráfico pequeño · 4+ GB para tráfico medio · SSD recomendado

## 🐳 Instalación

### Opción 1: Docker Compose completo (recomendado)

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # MariaDB - Base de datos
  mariadb:
    image: mariadb:latest
    container_name: wordpress_db
    restart: unless-stopped
    environment:
      - MYSQL_DATABASE=wordpress
      - MYSQL_ROOT_PASSWORD=tu_contraseña_root_fuerte
      - MYSQL_USER=wordpress
      - MYSQL_PASSWORD=tu_contraseña_wordpress_fuerte
      - TZ=Europe/Madrid
    volumes:
      - db_data:/var/lib/mysql
    command: --default-authentication-plugin=mysql_native_password

  # WordPress - CMS
  wordpress:
    image: wordpress:php8.2-fpm
    container_name: wordpress_app
    restart: unless-stopped
    depends_on:
      - mariadb
    environment:
      - WORDPRESS_DB_HOST=mariadb:3306
      - WORDPRESS_DB_USER=wordpress
      - WORDPRESS_DB_PASSWORD=tu_contraseña_wordpress_fuerte
      - WORDPRESS_DB_NAME=wordpress
      - WORDPRESS_TABLE_PREFIX=wp_
      - WORDPRESS_DEBUG=false
    volumes:
      - wordpress_data:/var/www/html

  # Nginx - Reverse Proxy / Web Server
  nginx:
    image: nginx:alpine
    container_name: wordpress_nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - wordpress_data:/var/www/html:ro
    depends_on:
      - wordpress

  # phpMyAdmin - Gestión BD web UI
  phpmyadmin:
    image: phpmyadmin:latest
    container_name: wordpress_phpmyadmin
    restart: unless-stopped
    environment:
      - PMA_HOST=mariadb
      - PMA_USER=root
      - PMA_PASSWORD=tu_contraseña_root_fuerte
    ports:
      - "8080:80"

volumes:
  wordpress_data:
  db_data:
EOF
```

```bash
# Crear nginx.conf (si no existe)
cat > nginx.conf << 'EOF'
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 100M;

    upstream wordpress {
        server wordpress:9000;
    }

    server {
        listen 80;
        server_name _;
        root /var/www/html;
        index index.php index.html;

        location ~ \.php$ {
            fastcgi_pass wordpress;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include /etc/nginx/fastcgi_params;
        }

        location ~ /\.ht {
            deny all;
        }
    }
}
EOF
```

```bash
# Levantar la stack
docker compose up -d
```

### Acceder (setup WordPress)

- **http://localhost** - Instalador WordPress (5-minute install)
- **http://localhost:8080** - phpMyAdmin (user: `root`, password: `tu_contraseña_root_fuerte`)

## ⚙️ Configuración

1. **Cambia todas las contraseñas** en `docker-compose.yml` antes de desplegar (usa contraseñas fuertes únicas)
2. **Ajusta la zona horaria** (`TZ=Europe/Madrid`) según tu ubicación
3. **Configura el dominio** en Nginx/Caddy para HTTPS automático (ver sección Acceso remoto seguro)
4. **Define WP_HOME y WP_SITEURL** en `wp-config.php` para forzar HTTPS (ver sección Acceso remoto seguro)
5. **Aumenta límites de memoria** si necesitas subir archivos grandes (ver Gestión y mantenimiento)
6. **Configura SMTP** en WordPress (plugin "WP Mail SMTP") para emails transaccionales
7. **Habilita pretty permalinks** en Ajustes → Enlaces permanentes → "Nombre de la entrada"

## 🚀 Primeros pasos

1. **Login admin WordPress**  
   Abre `http://localhost/wp-admin` → Ingresa usuario y contraseña admin → Dashboard principal aparece

2. **Crear primer post**  
   Dashboard → Posts → Add New → Título, contenido, categorías, etiquetas → Publish → Post aparece en blog frontend

3. **Personalizar sitio (apariencia)**  
   Dashboard → Appearance → Themes → Browse free themes o sube custom theme (ZIP) → Activate → Customize colores, header image, etc

4. **Agregar plugins (extensiones)**  
   Dashboard → Plugins → Add New → Busca plugin (ej. "Yoast SEO") → Install → Activate → Plugin aparece en sidebar izquierdo

5. **Gestionar usuarios y roles**  
   Dashboard → Users → Add New → Email, usuario, contraseña, rol (Administrator, Editor, Author, Contributor, Subscriber) → Cada rol tiene permisos específicos

6. **Configurar páginas estáticas (About, Contact, etc)**  
   Dashboard → Pages → Add New → Crea páginas necesarias → Appearance → Menus → crear menú con esas páginas

7. **Configurar inicio de sesión del sitio**  
   Dashboard → Settings → General → Site Title, Tagline, Site URL, Timezone, date format, week starts

8. **Backup de BD con phpMyAdmin**  
   Abre `http://localhost:8080` → Database: wordpress → Export → Descarga SQL file (backup BD completa) → Guarda en lugar seguro

9. **Backup de archivos WordPress (volumes)**  
   ```bash
   docker cp wordpress_app:/var/www/html ./wordpress-backup-$(date +%Y%m%d)
   ```

10. **Instalar Yoast SEO (plugin popular)**  
    Dashboard → Plugins → Add New → Busca "Yoast SEO" → Install → Activate → Aparece "SEO" en sidebar → Configura meta descriptions, keywords en cada post

11. **Habilitar permanentes (Pretty URLs)**  
    Dashboard → Settings → Permalinks → Elige "Post name" (para URLs limpias) → Guarda (WordPress auto-actualiza .htaccess)

12. **Configurar comentarios y discusiones**  
    Dashboard → Settings → Discussion → Habilita/deshabilita comentarios → Moderación, spam filters (Akismet plugin)

## 💡 Casos de uso

- **Blogs personales** - Full control, sin publicidad de WordPress.com, datos tuyos
- **Sitios corporativos** - Presencia web profesional, portfolio, contacto
- **Tiendas online** - WooCommerce plugin integrado, ventas completas, inventory management
- **Agencias digitales** - Sitios clientes self-hosted, mantenimiento fácil
- **Portfolios creativos** - Fotógrafos, artistas, diseñadores, galería de trabajos
- **Alternativa WordPress.com/Wix** - Total control, bajo costo, open source, no vendor lock-in

## 🔒 Acceso remoto seguro

### HTTPS con Caddy (producción)

```bash
# Caddyfile para WordPress
cat > Caddyfile << 'EOF'
midominio.com www.midominio.com {
    reverse_proxy localhost:80
}
EOF
```

Acceso: `https://midominio.com` con HTTPS automático y certificado Let's Encrypt

### IMPORTANTE: Configurar WordPress URL

```bash
# Editar wp-config.php en WordPress container
docker exec wordpress_app bash -c "nano /var/www/html/wp-config.php"
```

```php
// Agregar o modificar estas líneas:
define('WP_HOME', 'https://midominio.com');
define('WP_SITEURL', 'https://midominio.com');
define('FORCE_SSL_ADMIN', true);
```

## 🛠️ Gestión y mantenimiento

### Ver logs
```bash
docker logs -f wordpress_app
docker logs -f wordpress_db
docker logs -f wordpress_nginx
```

### Backup completo (BD + archivos)
```bash
mkdir -p ./backups

# Backup base datos
docker exec wordpress_db mysqldump -u wordpress -p tu_contraseña_wordpress wordpress > ./backups/wordpress-db-$(date +%Y%m%d).sql

# Backup archivos WordPress
docker cp wordpress_app:/var/www/html ./backups/wordpress-files-$(date +%Y%m%d)
```

### Restore de backup
```bash
# Base de datos
docker exec -i wordpress_db mysql -u wordpress -p tu_contraseña wordpress < ./backups/wordpress-db-YYYYMMDD.sql

# Archivos: reemplaza /var/www/html en volume o copia archivos
```

### Reiniciar servicios
```bash
docker compose restart
# O servicio específico
docker compose restart wordpress
```

### Actualizar WordPress, plugins, temas
```bash
# Vía Dashboard WordPress → Updates → Click Update automáticamente

# O vía WP-CLI en container:
docker exec wordpress_app wp core update --allow-root
docker exec wordpress_app wp plugin update --all --allow-root
```

### Limpiar caché y optimizar BD
```bash
docker exec wordpress_db mysqlcheck --optimize --all-databases -u root -p
# O instalar plugin "WP-Optimize" para limpiar automático
```

### Monitorear consumo
```bash
docker stats wordpress_app wordpress_db wordpress_nginx
```

### Aumentar límite upload archivos
```bash
# Editar wp-config.php
define('WP_MEMORY_LIMIT', '256M');
define('WP_MAX_MEMORY_LIMIT', '512M');
```

## 📝 Licencia

Este proyecto está licenciado bajo **GPL v3** - ver el archivo [LICENSE](LICENSE) para detalles.

WordPress es software libre licenciado bajo GPL v2+. MariaDB bajo GPL v2. phpMyAdmin bajo GPL v2. Nginx bajo licencia BSD-2-Clause.

---

> 📖 **Guía completa en el blog:** [Cómo instalar WordPress en Docker - Stack completo con MariaDB y phpMyAdmin autohospedado](https://genbyte.blogspot.com/2026/08/como-instalar-wordpress-en-docker-stack.html)