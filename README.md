# Ansible Role Nginx
Install Nginx with simple site using Let's Encrypt certificate

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