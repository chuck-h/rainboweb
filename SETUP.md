## Cheat sheet for setting up token manager on an AWS linux instance

### Install nginx - Option 1, for use with Cloudflare https proxy

`sudo yum install nginx`

generate self-signed cert and key, save in `/etc/pki/nginx/`
```
sudo mkdir -p /etc/pki/nginx/private/
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout /etc/pki/nginx/private/nginx-selfsigned.key -out /etc/pki/nginx/nginx-selfsigned.crt
```

nginx configuration at `/etc/nginx/nginx.conf`:
```
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log notice;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80;
        listen       [::]:80;
        server_name  localhost;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
          # Automatically route HTTP to HTTPS
                return 301 https://$server_name$request_uri;
        }
    }
# Settings for a TLS enabled server.
#
    upstream server {
      server 127.0.0.1:3000;
    }

    server {
        listen       443 ssl http2;
        listen       [::]:443 ssl http2;
        server_name  localhost;
        root         /usr/share/nginx/html;

        ssl_certificate "/etc/pki/nginx/nginx-selfsigned.crt";
        ssl_certificate_key "/etc/pki/nginx/private/nginx-selfsigned.key";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_protocols       TLSv1.1 TLSv1.2;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;
        location / {
            proxy_pass http://localhost:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
            proxy_set_header X-Real-IP       $proxy_protocol_addr;
            proxy_set_header X-Forwarded-For $proxy_protocol_addr;
            # Very important, controls proxied websocket connection timeout
            proxy_read_timeout 600s;
            add_header "Access-Control-Allow-Origin" * always;

            # Preflighted requests
            if ($request_method = OPTIONS ) {
                add_header "Access-Control-Allow-Origin"  *;
                add_header "Access-Control-Allow-Methods" "GET, POST, OPTIONS, HEAD";
                add_header "Access-Control-Allow-Headers" "Authorization, Origin, X-Requested-With, Content-Type, Accept";
                return 200;
            }

        }

        location /rainbo/ {
            root /usr/share/nginx;
            index index.html;
        }

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }

}
```

Start nginx
`sudo systemctl start ngnix.service`



### Install nginx - Option 2, for use with ssl host cert & key

`sudo yum install nginx`

Acquire ssl certificate and key, install at
`/etc/pki/nginx/server.crt` and `/etc/pki/nginx/private/server.key`

nginx configuration at `/etc/nginx/nginx.conf`

Note that `server_name` must be entered in two places

```
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log notice;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80;
        listen       [::]:80;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
          # Automatically route HTTP to HTTPS
                return 301 https://$server_name$request_uri;
        }
    }
# Settings for a TLS enabled server.
#
    server {
        listen       443 ssl http2;
        listen       [::]:443 ssl http2;
        server_name  _;
        root         /usr/share/nginx/html;

        ssl_certificate "/etc/pki/nginx/server.crt";
        ssl_certificate_key "/etc/pki/nginx/private/server.key";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_protocols       TLSv1.1 TLSv1.2;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;
        location / {
            proxy_pass http://localhost:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
            add_header "Access-Control-Allow-Origin" * always;

            # Preflighted requests
            if ($request_method = OPTIONS ) {
                add_header "Access-Control-Allow-Origin"  *;
                add_header "Access-Control-Allow-Methods" "GET, POST, OPTIONS, HEAD";
                add_header "Access-Control-Allow-Headers" "Authorization, Origin, X-Requested-With, Content-Type, Accept";
                return 200;
            }

        }

        location /rainbo/ {
            root /usr/share/nginx;
            index index.html;
        }

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }

}
```

Start nginx
`sudo systemctl start ngnix.service`

### Install token manager


```
sudo yum install git
mkdir ~/Projects
cd ~/Projects
git clone https://github.com/chuck-h/rainboweb.git

sudo mkdir /usr/share/nginx/rainbo
sudo cp ~/Projects/rainboweb/manage.html /usr/share/nginx/rainbo
```

## install nodejs & pm2

https://medium.com/monstar-lab-bangladesh-engineering/deploying-node-js-apps-in-amazon-linux-with-pm2-7fc3ef5897bb

## install encode-transaction-service

```
cd ~/Projects
git clone https://github.com/chuck-h/encode-transaction-service.git
```

