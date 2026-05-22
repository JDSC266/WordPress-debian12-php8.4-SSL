# Tutorial completo: WordPress es_CO latest con Nginx, PHP 8.4, MariaDB y SSL

Este tutorial instala WordPress en una VM con Debian 12 usando:

- Dominio: `justecno.com`
- Carpeta del sitio: `/var/www/justecno.com/public_html`
- Servidor web: Nginx
- PHP: 8.4
- Base de datos: MariaDB
- WordPress Colombia: `https://es-co.wordpress.org/latest-es_CO.tar.gz`
- SSL publico/global: Let's Encrypt con Certbot

> Importante: un certificado SSL publico se emite para un dominio, no simplemente para una maquina. Para que Let's Encrypt funcione, `justecno.com` debe apuntar a tu VM y los puertos `80` y `443` deben llegar desde internet hasta esa VM.

Si usas Cloudflare y quieres que el SSL dependa directamente de tu servidor, pon el registro DNS en modo **DNS only** con nube gris.

---

## 1. Revisar requisitos de red

Antes de instalar, confirma esto:

```text
DNS A: justecno.com -> tu IP publica
DNS A: www.justecno.com -> tu IP publica
Router: puerto 80 -> IP local de la VM Debian
Router: puerto 443 -> IP local de la VM Debian
VM: preferiblemente en modo bridge, o con port forwarding correcto
```

Si esto no esta bien, el sitio podria funcionar dentro de tu red, pero Let's Encrypt no podra validar el dominio desde internet.

---

## 2. Actualizar Debian

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

Despues del reinicio, vuelve a entrar a la VM.

---

## 3. Instalar herramientas base

```bash
sudo apt update
sudo apt install -y curl wget tar unzip rsync nano lsb-release ca-certificates gnupg
```

---

## 4. Agregar repositorio para PHP 8.4

Debian 12 no trae PHP 8.4 por defecto. Para instalarlo, agrega el repositorio de Sury:

```bash
curl -sSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb
sudo dpkg -i /tmp/debsuryorg-archive-keyring.deb

sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/debsuryorg-archive-keyring.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list'

sudo apt update
```

---

## 5. Instalar Nginx, MariaDB y PHP 8.4

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

Debe mostrar PHP 8.4.

---

## 6. Asegurar MariaDB

Ejecuta:

```bash
sudo mariadb-secure-installation
```

Respuestas sugeridas:

```text
Switch to unix_socket authentication? n
Change the root password? n
Remove anonymous users? y
Disallow root login remotely? y
Remove test database and access to it? y
Reload privilege tables now? y
```

---

## 7. Crear base de datos para WordPress

Entra a MariaDB:

```bash
sudo mariadb
```

Dentro de MariaDB ejecuta esto. Cambia `CAMBIA_ESTA_CLAVE_SEGURA` por una clave fuerte:

```sql
CREATE DATABASE justecno_wp DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'justecno_wpuser'@'localhost' IDENTIFIED BY 'CAMBIA_ESTA_CLAVE_SEGURA';
GRANT ALL PRIVILEGES ON justecno_wp.* TO 'justecno_wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## 8. Crear carpetas del sitio

La carpeta principal se llamara `justecno.com`, no `wordpress`.

```bash
sudo mkdir -p /var/www/justecno.com/public_html
sudo mkdir -p /var/www/justecno.com/backups
sudo chown -R www-data:www-data /var/www/justecno.com
```

Estructura esperada:

```text
/var/www/justecno.com/
├── backups
└── public_html
```

---

## 9. Descargar WordPress Colombia latest

Descarga la ultima version disponible en espanol de Colombia:

```bash
cd /tmp
wget https://es-co.wordpress.org/latest-es_CO.tar.gz
tar -xzf latest-es_CO.tar.gz
```

Copia WordPress a la carpeta del sitio:

```bash
sudo rsync -a /tmp/wordpress/ /var/www/justecno.com/public_html/
sudo chown -R www-data:www-data /var/www/justecno.com/public_html
```

Con esto, WordPress queda directamente en:

```text
/var/www/justecno.com/public_html
```

No quedara como:

```text
/var/www/justecno.com/public_html/wordpress
```

---

## 10. Crear `wp-config.php`

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

Dejalas asi:

```php
define( 'DB_NAME', 'justecno_wp' );
define( 'DB_USER', 'justecno_wpuser' );
define( 'DB_PASSWORD', 'CAMBIA_ESTA_CLAVE_SEGURA' );
define( 'DB_HOST', 'localhost' );
```

Usa la misma clave que pusiste al crear el usuario en MariaDB.

---

## 11. Generar claves secretas de WordPress

Ejecuta:

```bash
curl -s https://api.wordpress.org/secret-key/1.1/salt/
```

Copia el resultado y reemplaza el bloque de claves en `wp-config.php`.

Guarda con:

```text
CTRL + O
Enter
CTRL + X
```

---

## 12. Crear configuracion de Nginx

Crea el archivo del sitio:

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
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Guarda y sal.

---

## 13. Activar el sitio en Nginx

```bash
sudo ln -s /etc/nginx/sites-available/justecno.com /etc/nginx/sites-enabled/justecno.com
sudo rm -f /etc/nginx/sites-enabled/default
```

Prueba la configuracion:

```bash
sudo nginx -t
```

Si dice que todo esta bien, recarga Nginx:

```bash
sudo systemctl reload nginx
```

---

## 14. Probar HTTP antes del SSL

Antes de pedir el certificado, el dominio debe abrir por HTTP:

```text
http://justecno.com
```

Tambien puedes revisar que Nginx este escuchando:

```bash
sudo ss -tulpn | grep nginx
```

Debes ver que Nginx escucha en el puerto `80`.

Si `http://justecno.com` no abre desde internet, revisa:

```text
DNS del dominio
IP publica
Firewall
Redireccion de puertos del router
Modo de red de la VM
Cloudflare en DNS only si no quieres depender de su proxy
```

---

## 15. Instalar Certbot para SSL

Instala Snap y Certbot:

```bash
sudo apt install -y snapd
sudo snap install snapd
sudo snap install --classic certbot
sudo ln -sf /snap/bin/certbot /usr/local/bin/certbot
```

---

## 16. Descargar certificado SSL publico/global

Cuando `http://justecno.com` ya funcione desde internet, ejecuta:

```bash
sudo certbot --nginx -d justecno.com -d www.justecno.com
```

Cuando pregunte si quieres redirigir HTTP a HTTPS, elige la opcion de redireccion.

Luego prueba la renovacion automatica:

```bash
sudo certbot renew --dry-run
```

Si el comando termina bien, el certificado se renovara automaticamente.

---

## 17. Abrir instalador de WordPress

Abre en el navegador:

```text
https://justecno.com
```

Completa:

```text
Titulo del sitio
Usuario administrador
Contrasena
Correo electronico
```

Despues entra al panel:

```text
https://justecno.com/wp-admin
```

---

## 18. Ajustes recomendados en WordPress

Entra a:

```text
Ajustes -> Generales
```

Verifica que ambas URLs usen HTTPS:

```text
Direccion de WordPress: https://justecno.com
Direccion del sitio: https://justecno.com
```

Luego entra a:

```text
Ajustes -> Enlaces permanentes
```

Selecciona:

```text
Nombre de la entrada
```

Guarda los cambios.

---

## 19. Comandos utiles de verificacion

Ver estado de servicios:

```bash
sudo systemctl status nginx
sudo systemctl status mariadb
sudo systemctl status php8.4-fpm
```

Probar configuracion Nginx:

```bash
sudo nginx -t
```

Ver version PHP:

```bash
php -v
```

Ver certificados de Certbot:

```bash
sudo certbot certificates
```

Probar renovacion SSL:

```bash
sudo certbot renew --dry-run
```

Ver logs del sitio:

```bash
sudo tail -f /var/log/nginx/justecno.com.access.log
sudo tail -f /var/log/nginx/justecno.com.error.log
```

---

## 20. Resultado final esperado

Al terminar, deberias tener:

```text
https://justecno.com
```

Funcionando con:

```text
WordPress Colombia latest
Nginx
PHP 8.4
MariaDB
SSL publico de Let's Encrypt
```

Y la instalacion organizada en:

```text
/var/www/justecno.com/public_html
```

