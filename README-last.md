# Tutorial completo: WordPress 7.0 es_CO con Nginx, PHP 8.4, MariaDB y SSL en Debian 12

Este tutorial es para instalar WordPress desde cero en una VM Debian 12, usando:

- Dominio: `justecno.com`
- Carpeta del sitio: `/var/www/justecno.com/public_html`
- Servidor web: Nginx
- PHP: 8.4
- Base de datos: MariaDB
- WordPress: `7.0 es_CO`
- Publicacion: Cloudflare Tunnel
- Certificado local: Let's Encrypt con Certbot usando DNS de Cloudflare

> Importante: como usas Cloudflare Tunnel y tu proveedor bloquea los puertos `80` y `443`, no uses `sudo certbot --nginx`. Ese metodo necesita que Let's Encrypt pueda entrar directamente a tu servidor por internet. En tu caso, el metodo correcto es `dns-cloudflare`.

## 0. Idea general

Tu sitio quedara asi:

```text
Visitante
  -> https://justecno.com
  -> Cloudflare
  -> Cloudflare Tunnel
  -> https://localhost:443 en tu VM Debian
  -> Nginx
  -> PHP 8.4
  -> WordPress
  -> MariaDB
```

Esto significa:

- El visitante entra por HTTPS.
- Cloudflare Tunnel evita abrir puertos en tu router.
- Certbot sacara un certificado valido para `justecno.com` usando DNS.
- Nginx usara ese certificado en tu VM.
- Cloudflare Tunnel se conectara a tu Nginx local usando `https://localhost:443`.

## 1. Actualizar Debian

Entra por SSH o terminal de la VM y ejecuta:

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

Despues del reinicio vuelve a entrar a la VM.

## 2. Instalar herramientas base

```bash
sudo apt update
sudo apt install -y curl wget tar unzip rsync nano lsb-release ca-certificates gnupg snapd
```

Activa `snapd`:

```bash
sudo systemctl enable --now snapd
```

## 3. Agregar repositorio para PHP 8.4

Debian 12 trae PHP 8.2 por defecto. Para PHP 8.4 usaremos el repositorio de Sury.

```bash
sudo apt update
sudo apt install -y lsb-release ca-certificates curl

curl -sSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb
sudo dpkg -i /tmp/debsuryorg-archive-keyring.deb

sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/debsuryorg-archive-keyring.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list'

sudo apt update
```

## 4. Instalar Nginx, MariaDB y PHP 8.4

```bash
sudo apt install -y nginx mariadb-server \
php8.4-fpm php8.4-cli php8.4-mysql php8.4-curl php8.4-gd \
php8.4-intl php8.4-mbstring php8.4-soap php8.4-xml \
php8.4-zip php8.4-imagick php8.4-bcmath php8.4-opcache
```

Activa los servicios:

```bash
sudo systemctl enable --now nginx
sudo systemctl enable --now mariadb
sudo systemctl enable --now php8.4-fpm
```

Verifica PHP:

```bash
php -v
```

Debe mostrar `PHP 8.4`.

## 5. Ajustes recomendados de PHP

Crea un archivo de configuracion para tu sitio:

```bash
sudo nano /etc/php/8.4/fpm/conf.d/99-justecno.ini
```

Pega esto:

```ini
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
max_execution_time = 300
max_input_vars = 3000
```

Guarda y reinicia PHP-FPM:

```bash
sudo systemctl restart php8.4-fpm
```

## 6. Preparar MariaDB

Ejecuta el asistente de seguridad:

```bash
sudo mariadb-secure-installation
```

Respuestas recomendadas:

```text
Enter current password for root: presiona Enter
Switch to unix_socket authentication? n
Change the root password? n
Remove anonymous users? y
Disallow root login remotely? y
Remove test database and access to it? y
Reload privilege tables now? y
```

Genera una clave fuerte para el usuario de WordPress:

```bash
openssl rand -base64 32
```

Guarda esa clave en un lugar seguro. Ahora entra a MariaDB:

```bash
sudo mariadb
```

Dentro de MariaDB ejecuta esto, cambiando `CAMBIA_ESTA_CLAVE_SEGURA` por tu clave:

```sql
CREATE DATABASE justecno_wp DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'justecno_wpuser'@'localhost' IDENTIFIED BY 'CAMBIA_ESTA_CLAVE_SEGURA';
GRANT ALL PRIVILEGES ON justecno_wp.* TO 'justecno_wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## 7. Crear carpetas ordenadas

```bash
sudo mkdir -p /var/www/justecno.com/public_html
sudo mkdir -p /var/www/justecno.com/backups
sudo chown -R www-data:www-data /var/www/justecno.com
```

La estructura quedara asi:

```text
/var/www/
└── justecno.com/
    ├── backups/
    └── public_html/
```

## 8. Descargar WordPress 7.0 Colombia

```bash
cd /tmp
wget -O latest-es_CO.tar.gz https://es-co.wordpress.org/latest-es_CO.tar.gz
tar -xzf latest-es_CO.tar.gz

sudo rsync -a /tmp/wordpress/ /var/www/justecno.com/public_html/
sudo chown -R www-data:www-data /var/www/justecno.com/public_html
```

Ajusta permisos:

```bash
sudo find /var/www/justecno.com/public_html -type d -exec chmod 755 {} \;
sudo find /var/www/justecno.com/public_html -type f -exec chmod 644 {} \;
```

## 9. Crear `wp-config.php`

```bash
cd /var/www/justecno.com/public_html
sudo -u www-data cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
```

Busca estas lineas:

```php
define( 'DB_NAME', 'database_name_here' );
define( 'DB_USER', 'username_here' );
define( 'DB_PASSWORD', 'password_here' );
define( 'DB_HOST', 'localhost' );
```

Cambialas por:

```php
define( 'DB_NAME', 'justecno_wp' );
define( 'DB_USER', 'justecno_wpuser' );
define( 'DB_PASSWORD', 'CAMBIA_ESTA_CLAVE_SEGURA' );
define( 'DB_HOST', 'localhost' );
```

Ahora genera las claves secretas de WordPress:

```bash
curl -s https://api.wordpress.org/secret-key/1.1/salt/
```

Copia lo que salga y reemplaza el bloque de claves dentro de `wp-config.php`.

Antes de esta linea:

```php
/* That's all, stop editing! Happy publishing. */
```

agrega esto:

```php
define( 'WP_HOME', 'https://justecno.com' );
define( 'WP_SITEURL', 'https://justecno.com' );

if ( isset( $_SERVER['HTTP_X_FORWARDED_PROTO'] ) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https' ) {
    $_SERVER['HTTPS'] = 'on';
}

if ( isset( $_SERVER['HTTP_CF_VISITOR'] ) && strpos( $_SERVER['HTTP_CF_VISITOR'], 'https' ) !== false ) {
    $_SERVER['HTTPS'] = 'on';
}
```

Esto ayuda a WordPress a entender que el sitio publico funciona con HTTPS aunque pase por Cloudflare Tunnel.

## 10. Crear configuracion inicial de Nginx

Primero creamos Nginx con HTTP local. Despues del certificado lo cambiaremos a HTTPS.

```bash
sudo nano /etc/nginx/sites-available/justecno.com
```

Pega esto:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name justecno.com www.justecno.com;

    root /var/www/justecno.com/public_html;
    index index.php index.html;

    access_log /var/log/nginx/justecno.com.access.log;
    error_log /var/log/nginx/justecno.com.error.log;

    client_max_body_size 64M;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_param HTTP_X_FORWARDED_PROTO $http_x_forwarded_proto;
        fastcgi_param HTTP_CF_VISITOR $http_cf_visitor;
    }

    location ~* /(?:uploads|files)/.*\.php$ {
        deny all;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Activa el sitio:

```bash
sudo ln -s /etc/nginx/sites-available/justecno.com /etc/nginx/sites-enabled/justecno.com
sudo rm -f /etc/nginx/sites-enabled/default

sudo nginx -t
sudo systemctl reload nginx
```

Prueba localmente:

```bash
curl -I -H "Host: justecno.com" http://127.0.0.1
```

Debe responder algo como `HTTP/1.1 200 OK`, `301`, `302` o `403`. Lo importante es que Nginx responda.

## 11. Instalar Certbot con plugin de Cloudflare

Instala Certbot con snap:

```bash
sudo snap install core
sudo snap refresh core
sudo snap install --classic certbot
sudo ln -sf /snap/bin/certbot /usr/local/bin/certbot
```

Permite instalar plugins DNS:

```bash
sudo snap set certbot trust-plugin-with-root=ok
sudo snap install certbot-dns-cloudflare
```

Verifica plugins:

```bash
certbot plugins
```

Debe aparecer algo relacionado con `dns-cloudflare`.

## 12. Crear API Token en Cloudflare

En Cloudflare ve a:

```text
My Profile -> API Tokens -> Create Token -> Custom token
```

Configura permisos:

```text
Zone -> DNS -> Edit
Zone -> Zone -> Read
```

Configura el recurso de zona:

```text
Include -> Specific zone -> justecno.com
```

Crea el token y copialo. Ese token se vera solo una vez.

## 13. Guardar token de Cloudflare en Debian

```bash
sudo mkdir -p /root/.secrets/certbot
sudo nano /root/.secrets/certbot/cloudflare.ini
```

Pega esto:

```ini
dns_cloudflare_api_token = PEGA_AQUI_TU_TOKEN_DE_CLOUDFLARE
```

Protege el archivo:

```bash
sudo chmod 600 /root/.secrets/certbot/cloudflare.ini
```

## 14. Sacar certificado SSL con DNS de Cloudflare

Este es el comando correcto para tu caso:

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/certbot/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  -d justecno.com \
  -d www.justecno.com
```

Si ya tenias un certificado viejo y quieres reemplazarlo, usa:

```bash
sudo certbot certonly --force-renewal \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/certbot/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  -d justecno.com \
  -d www.justecno.com
```

Verifica el certificado:

```bash
sudo certbot certificates
```

Deberias ver rutas como:

```text
/etc/letsencrypt/live/justecno.com/fullchain.pem
/etc/letsencrypt/live/justecno.com/privkey.pem
```

Prueba renovacion:

```bash
sudo certbot renew --dry-run
```

Si esto funciona, ya tienes renovacion automatica lista.

## 15. Cambiar Nginx a HTTPS con el certificado

Edita el sitio:

```bash
sudo nano /etc/nginx/sites-available/justecno.com
```

Reemplaza todo por esto:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name justecno.com www.justecno.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name justecno.com www.justecno.com;

    root /var/www/justecno.com/public_html;
    index index.php index.html;

    access_log /var/log/nginx/justecno.com.access.log;
    error_log /var/log/nginx/justecno.com.error.log;

    client_max_body_size 64M;

    ssl_certificate /etc/letsencrypt/live/justecno.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/justecno.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options SAMEORIGIN always;
    add_header Referrer-Policy strict-origin-when-cross-origin always;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_param HTTPS on;
        fastcgi_param HTTP_X_FORWARDED_PROTO $http_x_forwarded_proto;
        fastcgi_param HTTP_CF_VISITOR $http_cf_visitor;
    }

    location ~* /(?:uploads|files)/.*\.php$ {
        deny all;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Verifica y recarga:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Prueba HTTPS local con SNI correcto:

```bash
curl -I --resolve justecno.com:443:127.0.0.1 https://justecno.com
```

Debe responder desde Nginx sin error de certificado.

## 16. Configurar Cloudflare Tunnel

Ahora el tunel debe apuntar a HTTPS local, no a HTTP.

### Opcion A: tunnel administrado desde dashboard

En Cloudflare:

```text
Zero Trust -> Networks -> Tunnels -> tu tunnel -> Configure -> Public Hostnames
```

Para `justecno.com`:

```text
Subdomain: dejar vacio
Domain: justecno.com
Type: HTTPS
URL: localhost:443
```

En opciones avanzadas del hostname:

```text
TLS -> Origin Server Name: justecno.com
No TLS Verify: desactivado
```

Para `www.justecno.com`:

```text
Subdomain: www
Domain: justecno.com
Type: HTTPS
URL: localhost:443
```

Opciones avanzadas:

```text
TLS -> Origin Server Name: justecno.com
No TLS Verify: desactivado
```

### Opcion B: tunnel administrado por archivo `config.yml`

Si tu tunnel usa archivo local, revisa:

```bash
sudo nano /etc/cloudflared/config.yml
```

Ejemplo:

```yaml
tunnel: TU_UUID_DEL_TUNNEL
credentials-file: /etc/cloudflared/TU_UUID_DEL_TUNNEL.json

ingress:
  - hostname: justecno.com
    service: https://localhost:443
    originRequest:
      originServerName: justecno.com

  - hostname: www.justecno.com
    service: https://localhost:443
    originRequest:
      originServerName: justecno.com

  - service: http_status:404
```

Reinicia Cloudflare Tunnel:

```bash
sudo systemctl restart cloudflared
sudo systemctl status cloudflared --no-pager
```

## 17. Configuracion recomendada en Cloudflare

En Cloudflare, para el dominio `justecno.com`:

```text
SSL/TLS -> Overview -> Encryption mode: Full (strict)
```

Tambien puedes activar:

```text
SSL/TLS -> Edge Certificates -> Always Use HTTPS: On
SSL/TLS -> Edge Certificates -> Automatic HTTPS Rewrites: On
```

Con Cloudflare Tunnel no necesitas abrir puertos `80` ni `443` en tu router.

## 18. Terminar instalacion de WordPress

Abre:

```text
https://justecno.com
```

Completa el instalador:

```text
Titulo del sitio
Usuario administrador
Contrasena
Correo
```

Luego entra:

```text
https://justecno.com/wp-admin
```

Dentro de WordPress ve a:

```text
Ajustes -> Generales
```

Verifica:

```text
Direccion de WordPress: https://justecno.com
Direccion del sitio: https://justecno.com
```

Luego ve a:

```text
Ajustes -> Enlaces permanentes
```

Elige:

```text
Nombre de la entrada
```

Guarda cambios.

## 19. Comandos de verificacion

Servicios:

```bash
sudo systemctl status nginx --no-pager
sudo systemctl status php8.4-fpm --no-pager
sudo systemctl status mariadb --no-pager
sudo systemctl status cloudflared --no-pager
```

Nginx:

```bash
sudo nginx -t
```

PHP:

```bash
php -v
```

Certificados:

```bash
sudo certbot certificates
sudo certbot renew --dry-run
```

HTTPS local:

```bash
curl -I --resolve justecno.com:443:127.0.0.1 https://justecno.com
```

HTTPS publico:

```bash
curl -I https://justecno.com
```

## 20. Errores comunes y solucion

### Error: `no valid A records found`

Eso pasa con:

```bash
sudo certbot --nginx -d justecno.com -d www.justecno.com
```

En tu caso es normal porque usas Cloudflare Tunnel. Solucion: usa el metodo DNS:

```bash
sudo certbot certonly --dns-cloudflare ...
```

### Error: `Certbot failed to authenticate some domains`

Posibles causas:

- Token de Cloudflare incorrecto.
- El token no tiene permiso `Zone -> DNS -> Edit`.
- El token no esta limitado a la zona correcta.
- El TXT de validacion tarda mas en propagarse.

Prueba con mas espera:

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/certbot/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 120 \
  -d justecno.com \
  -d www.justecno.com
```

### Error 502 en Cloudflare

Revisa:

```bash
sudo systemctl status nginx --no-pager
sudo systemctl status cloudflared --no-pager
sudo nginx -t
```

Tambien revisa que el tunnel apunte a:

```text
https://localhost:443
```

### Error 525 o problema TLS en Cloudflare

Revisa en el hostname del tunnel:

```text
Origin Server Name: justecno.com
No TLS Verify: desactivado
```

Tambien verifica localmente:

```bash
curl -I --resolve justecno.com:443:127.0.0.1 https://justecno.com
```

### WordPress carga, pero CSS o scripts salen por HTTP

Revisa que `wp-config.php` tenga:

```php
define( 'WP_HOME', 'https://justecno.com' );
define( 'WP_SITEURL', 'https://justecno.com' );
```

Y tambien el bloque:

```php
if ( isset( $_SERVER['HTTP_X_FORWARDED_PROTO'] ) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https' ) {
    $_SERVER['HTTPS'] = 'on';
}
```

### `Package php8.4-fpm has no installation candidate`

El repositorio Sury no quedo bien instalado. Repite la seccion 3 y luego:

```bash
sudo apt update
apt-cache policy php8.4-fpm
```

### Nginx muestra la pagina por defecto

Revisa:

```bash
ls -l /etc/nginx/sites-enabled/
```

Debe estar `justecno.com` y no debe estar `default`.

Si aparece `default`, quitalo:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

## 21. Backup basico

Crear backup de base de datos:

```bash
sudo mysqldump justecno_wp > /var/www/justecno.com/backups/justecno_wp.sql
sudo chown www-data:www-data /var/www/justecno.com/backups/justecno_wp.sql
```

Comprimir archivos del sitio:

```bash
sudo tar -czf /var/www/justecno.com/backups/public_html.tar.gz -C /var/www/justecno.com public_html
sudo chown www-data:www-data /var/www/justecno.com/backups/public_html.tar.gz
```

## 22. Resumen de comandos clave

Certificado correcto para tu caso:

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/certbot/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  -d justecno.com \
  -d www.justecno.com
```

Renovacion:

```bash
sudo certbot renew --dry-run
```

Tunnel:

```text
justecno.com -> https://localhost:443
www.justecno.com -> https://localhost:443
Origin Server Name -> justecno.com
```

Nginx PHP socket:

```text
unix:/run/php/php8.4-fpm.sock
```

Carpeta publica:

```text
/var/www/justecno.com/public_html
```

## Fuentes

- WordPress Colombia, archivo de versiones: https://es-co.wordpress.org/download/releases/
- WordPress requirements: https://wordpress.org/about/requirements/
- Repositorio PHP Sury para Debian: https://packages.sury.org/php/README.txt
- Certbot DNS Cloudflare: https://certbot-dns-cloudflare.readthedocs.io/en/stable/
- Snap `certbot-dns-cloudflare`: https://snapcraft.io/certbot-dns-cloudflare
- Cloudflare Tunnel origin parameters: https://developers.cloudflare.com/tunnel/advanced/origin-parameters/
"# WordPress-debian12-php8.4-SSL" 
