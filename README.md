# docker-compose-wordpress-ubuntu
A description for how to install docker-compose to wordpress uing nginx with ssl certs
curl -fsSL https://get.docker.com -o get-docker.sh

sudo sh get-docker.sh

sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

mkdir -p /root/docker/letsencrypt

cd /root/docker/letsencrypt

docker run --rm -ti -v $PWD/log/:/var/log/letsencrypt/ -v $PWD/etc/:/etc/letsencrypt/ -p 80:80 deliverous/certbot certonly --standalone -d www.example.com

mkdir -p /var/ds/repo/nginx-conf-folder/ssl

cd /var/ds/repo/nginx-conf-folder/ssl

openssl dhparam -out dhparam.pem 2048

cd /var/ds/repo/nginx-conf-folder

nano nginx.conf (with the following nginx configuration)

server {

  listen 80;

  listen [::]:80;

   server_name www.example.com;

   location ~ /.well-known/acme-challenge {

          allow all;

          root /var/www/html;

  }

  location / {

          rewrite ^ https://$host$request_uri? permanent;

  }

}

server {

  listen 443 ssl http2;

  listen [::]:443 ssl http2;

  server_name www.example.com;

  index index.php index.html index.htm;

  root /var/www/html;

  server_tokens off;

  # add our paths for the certificates Certbot created 

  ssl_certificate /etc/letsencrypt/live/www.example.com/fullchain.pem;

  ssl_certificate_key /etc/letsencrypt/live/www.example.com/privkey.pem;

  ssl_protocols TLSv1.2 TLSv1.3;

  # some security headers ( optional )

add_header X-Frame-Options "SAMEORIGIN" always;

add_header X-XSS-Protection "1; mode=block" always;

add_header X-Content-Type-Options "nosniff" always;

add_header Referrer-Policy "no-referrer-when-downgrade" always;

add_header Content-Security-Policy "default-src * data: 'unsafe-eval' 'unsafe-inline'" always;

ssl_dhparam /etc/nginx/ssl/dhparam.pem;

ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    ssl_prefer_server_ciphers off;

ssl_stapling on;

ssl_stapling_verify on;

# resolver set to Cloudflare

resolver 8.8.8.8 8.8.4.4;

resolver_timeout 5s;

ssl_session_tickets off;

#ssl_ecdh_curve x25519:sect571r1:secp52r1:secp384r1;

ssl_ecdh_curve secp384r1;

add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload;" always;

  location / {

          try_files $uri $uri/ /index.php$is_args$args;

  }

  location ~ \.php$ {

          try_files $uri =404;

          fastcgi_split_path_info ^(.+\.php)(/.+)$;

          fastcgi_pass wordpress:9000;

          fastcgi_index index.php;

          include fastcgi_params;

          fastcgi_param SCRIPT_FILENAME  $document_root$fastcgi_script_name;

          fastcgi_param PATH_INFO $fastcgi_path_info;

  }

  location ~ /\.ht {

          deny all;

  }

   location = /favicon.ico { 

          log_not_found off; access_log off; 

  }

  location = /favicon.svg { 

          log_not_found off; access_log off; 

  }

  location = /robots.txt { 

          log_not_found off; access_log off; allow all; 

  }

  location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {

          expires max;

          log_not_found off;

  }

}

cd /var/ds/

nano docker-compose.yml (with the followings)

version: '3'

services:

  db:

    image: mysql:8.0

    container_name: db

    restart: unless-stopped

    command: '--default-authentication-plugin=mysql_native_password'

    env_file: .env

    environment:

      - MYSQL_DATABASE=wordpress

    volumes: 

      - dbdata:/var/lib/mysql

  wordpress:

    image: wordpress:5-fpm-alpine

    depends_on:

      - db

    container_name: wordpress

    restart: unless-stopped

    volumes:

      - wordpress:/var/www/html

    env_file: .env

    environment:

      - WORDPRESS_DB_HOST=db:3306

      - WORDPRESS_DB_USER=$MYSQL_USER

      - WORDPRESS_DB_PASSWORD=$MYSQL_PASSWORD

      - WORDPRESS_DB_NAME=wordpress

  webserver:

    depends_on:

      - wordpress

    image: nginx:1.15.12-alpine

    container_name: webserver

    restart: unless-stopped

    ports:

      - "80:80"

      - "443:443"

    volumes:

      - wordpress:/var/www/html

      - /root/docker/letsencrypt/etc/:/etc/letsencrypt/

      - /var/ds/repo/nginx-conf-folder/:/etc/nginx/conf.d

      - /var/ds/repo/nginx-conf-folder/ssl/:/etc/nginx/ssl

    networks:

      default:

create .env file (/var/ds)

MYSQL_ROOT_PASSWORD=1234password 
MYSQL_USER=wordpress 
MYSQL_PASSWORD=1234password

cd /var/ds

docker-compose up

If you see mysql socket is ready for connection then it should be working

https://www.example.com to continue with the wordpress installation

I took the example from : https://zactyh.medium.com/hosting-wordpress-in-docker-with-ssl-2020-fa9391881f3

check your grading at www.ssllabs.com www.securityheaders and wwww.keycdn.com (for http2)

## there will be permission
## use the following code to lock wordpress from over-written

sudo find /var/lib/docker/volumes/ds_wordpress/_data/wp-content/themes/ -type f -exec chmod 644 {} +
sudo find /var/lib/docker/volumes/ds_wordpress/_data/wp-content/themes/ -type d -exec chmod 755 {} +

## this will lock the directory to 755 and file to 644
## in the event you need to install new plugins, do the following

sudo find /var/lib/docker/volumes/ds_wordpress/_data/wp-content/themes/ -type f -exec chmod 664 {} +
sudo find /var/lib/docker/volumes/ds_wordpress/_data/wp-content/themes/ -type d -exec chmod 775 {} +

## In the event this does not work:

sudo docker exec -it (container id) /bin/bash
go to /var/www/html
chmod 777 -R wp-content/

## once you had successfully installed the plugin then do the above 755 and 644



