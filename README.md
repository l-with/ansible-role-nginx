# Ansible Role Nginx
Install Nginx with cofiguration using Let's Encrypt certificate

## Role Variables

#### `nginx_server_FQDN`:
the FQDN of the server for nginx_server_name and Let's Encrypt certificates

#### `nginx_location_stanza`: 
```
    # First attempt to serve request as file, then
    # as directory, then fall back to displaying a 404.
    try_files $uri $uri/ =404;
```
the content for the `location /` stanza

#### `nginx_location_stanzas`: 
```
    location / {
        {{ nginx_location_stanza}}
    }
```
all location stanzas

#### `nginx_ssl_config`: 
```
    ###########################################################################
    # start copy from/etc/nginx/sites-enabled/default managed by certbot
    ###########################################################################
    listen [::]:{{ item }} ssl ipv6only=on; # managed by Certbot
    listen {{ item }} ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/{{ nginx_server_FQDN }}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/{{ nginx_server_FQDN }}/privkey.pem;
    # include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
    ###########################################################################
    # end copy from/etc/nginx/sites-enabled/default managed by certbot
    ###########################################################################

    ###########################################################################
    # start copy from /etc/letsencrypt/options-ssl-nginx.conf
    ###########################################################################
    ssl_session_cache shared:le_nginx_SSL:10m;
    ssl_session_timeout 1440m;
    ssl_session_tickets off;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
    ###########################################################################
    # end copy from /etc/letsencrypt/options-ssl-nginx.conf
    ###########################################################################
```
the ssl configuration used in the template