# Ansible Role Nginx
Install Nginx with configuration using Let's Encrypt certificate

## Usage
Without any parameter nginx is configured similar to a standard installation 
with Let's Encrypt installed.
Let's Encrypt is not installed, 
and the certificate is not generated by this module but assumed to be already installed.

### Port 80
The configuration for port 80 of `nginx_server_FQDN` redirects to port 443.

### Port 443
The configuration for port 443 of `nginx_server_FQDN` is either reverse proxy (`nginx_443_proxy_port`) or `nginx_location_root_stanza_content` in combination with `nginx_root_index`

## Role Variables

### `nginx_install`: yes
if nginx should be installed and enabled

### `nginx_restart`: yes
if nginx should be restarted

### `nginx_configuration_home`: `/etc/nginx`

### `nginx_443_proxy_port`: 8080
the proxy port for nginx reverse proxy for port 443 

### `nginx_server_FQDN`:
the FQDN of the server for nginx_server_name and Let's Encrypt certificates

### `nginx_server_name`: `"{{ nginx_server_FQDN }}"`

### `nginx_location_root_stanza_content`: 
```
    # First attempt to serve request as file, then
    # as directory, then fall back to displaying a 404.
    try_files $uri $uri/ =404;
```
the content for the `location /` stanza

### `nginx_root_index`:
```
  root /var/www/html;
  index index.html index.htm index.nginx-debian.html;
```
the root and index if nginx is not configured as reverse proxy


### `nginx_proxy_conf`:
```
    proxy_pass http://localhost:{{ nginx_conf.proxy_port }}/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```
the content for the `nginx_proxy_location_stanza`

### `nginx_443_proxy_port`: 8080
the proxy port for https 

### `nginx_ssl_config`: 
```
  ###########################################################################
  # start copy from/etc/nginx/sites-enabled/default managed by certbot
  ###########################################################################
  # listen [::]:{{ nginx_conf.port }} ssl ipv6only=on; # managed by Certbot
  listen [::]:{{ nginx_conf.port }} ssl; # managed by Certbot
  listen {{ nginx_conf.port }} ssl; # managed by Certbot
  ssl_certificate /etc/letsencrypt/live/{{ nginx_conf.server_name }}/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/{{ nginx_conf.server_name }}/privkey.pem;
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
the ssl configuration used in the templates

### `nginx_confs`: []
the ports and location stanzas for additional nginx ssl configurations
format: `{ port : <portnumber>, location: <path>, proxy_port: <proxy_port>, server_name: <FQDN> }`
`loop_var` should be `nginx_conf`

### `nginx_vouch_FQDN`
the FQDN of vouch-proxy

### `nginx_vouch_port`: 9090
the port of vouch-proxy

### `nginx_validate_vouch_config`:
```
  ###########################################################################
  # start copy from https://github.com/vouch/vouch-proxy
  ###########################################################################

  # send all requests to the `/validate` endpoint for authorization
  auth_request /validate;

  location = /validate {
    # forward the /validate request to Vouch Proxy
    proxy_pass http://127.0.0.1:{{ nginx_vouch_port }}/validate;

    # be sure to pass the original host header
    proxy_set_header Host $http_host;

    # Vouch Proxy only acts on the request headers
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";

    # optionally add X-Vouch-User as returned by Vouch Proxy along with the request
    auth_request_set $auth_resp_x_vouch_user $upstream_http_x_vouch_user;

    # optionally add X-Vouch-IdP-Claims-* custom claims you are tracking
    #    auth_request_set $auth_resp_x_vouch_idp_claims_groups $upstream_http_x_vouch_idp_claims_groups;
    #    auth_request_set $auth_resp_x_vouch_idp_claims_given_name $upstream_http_x_vouch_idp_claims_given_name;
    # optinally add X-Vouch-IdP-AccessToken or X-Vouch-IdP-IdToken
    #    auth_request_set $auth_resp_x_vouch_idp_accesstoken $upstream_http_x_vouch_idp_accesstoken;
    #    auth_request_set $auth_resp_x_vouch_idp_idtoken $upstream_http_x_vouch_idp_idtoken;

    # these return values are used by the @error401 call
    auth_request_set $auth_resp_jwt $upstream_http_x_vouch_jwt;
    auth_request_set $auth_resp_err $upstream_http_x_vouch_err;
    auth_request_set $auth_resp_failcount $upstream_http_x_vouch_failcount;

    # Vouch Proxy can run behind the same Nginx reverse proxy
    # may need to comply to "upstream" server naming
    # proxy_pass http://{{ nginx_vouch_FQDN }}/validate;
    # proxy_set_header Host $http_host;
  }

  # if validate returns `401 not authorized` then forward the request to the error401block
  error_page 401 = @error401;

  location @error401 {
    # redirect to Vouch Proxy for login
    return 302 https://{{ nginx_vouch_FQDN }}/login?url=$scheme://$http_host$request_uri&vouch-failcount=$auth_resp_failcount&X-Vouch-Token=$auth_resp_jwt&error=$auth_resp_err;
    # you usually *want* to redirect to Vouch running behind the same Nginx config proteced by https
    # but to get started you can just forward the end user to the port that vouch is running on
    # return 302 http://{{ nginx_vouch_FQDN }}:9090/login?url=$scheme://$http_host$request_uri&vouch-failcount=$auth_resp_failcount&X-Vouch-Token=$auth_resp_jwt&error=$auth_resp_err;
  }
  ###########################################################################
  # end copy from https://github.com/vouch/vouch-proxy
  ###########################################################################
```
the nginx validate configuration for vouch-proxy