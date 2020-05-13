nginx-rp
===============================

NGINX reverse proxy for docker-compose.

Base requirements
------------

* docker
* docker-compose


For a fresh ubuntu you can install the above with:
```
groupadd docker
snap install docker
apt install docker-compose
```

Upstream servers
------------
NGINXes in upstream servers should not expose ports on the main host.

Change:
```
...

    ports:
      - 80:80
      - 443:443

...
```

to:
```
...

    expose:
      - 80
      - 443

...
```

Adding new site
------------

1. Add proper docker network to the `docker-compose.yml`:

> :warning: Change `test` and `test_default` according to your needs.

```
...

services:
  nginx-rp:
    ...
    networks:
      - test
    ...

...

networks:
  test:
    external:
      name: test_default
```

2. Add proper server configuration to the `nginx/conf/default.conf`. Example configuration:

> :warning: Change `server_name` and `proxy_pass` in `default.conf` according to your needs.

```
server {
    listen      80;

    listen      443           ssl http2;

    server_name               test.localhost.local;

    ssl_session_cache         shared:SSL:20m;
    ssl_session_timeout       10m;

    ssl_protocols             TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers               "ECDH+AESGCM:ECDH+AES256:ECDH+AES128:!ADH:!AECDH:!MD5;";

    ssl_stapling              on;
    ssl_stapling_verify       on;

    ssl_certificate           /etc/letsencrypt/live/$server_name/fullchain.pem;
    ssl_certificate_key       /etc/letsencrypt/live/$server_name/privkey.pem;
    ssl_trusted_certificate   /etc/letsencrypt/live/$server_name/chain.pem;

    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    gzip off;

    access_log                /dev/stdout;
    error_log                 /dev/stderr info;

    client_max_body_size 100M;

    location ^~ /.well-known {
        allow all;
        root  /data/letsencrypt/;
    }

    location / {
        proxy_pass_header Server;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X_SCHEME $scheme;
        proxy_redirect off;
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_http_version 1.1;
        proxy_intercept_errors on;
        proxy_pass https://test_nginx_1;
    }
}
```

3. Restart or reload NGINX service:

**WITH RESTART**:
```
$ docker-compose down
$ docker-compose up -d
```

**WITHOUT RESTART:** 
> :warning: Change `test_default` in below command according to your needs.

```
$ docker network connect test_default nginx-rp_nginx-rp_1
$ docker-compose exec nginx-rp nginx -s reload
```

Removing the site
------------------------------------

1. Remove the network from the `docker-compose.yml`. See [adding new site](#adding-new-site)

2. Remove the server from the `nginx/conf/default.conf`. See [adding new site](#adding-new-site)

3. Restart or reload NGINX service:

**WITH RESTART**:
```
$ docker-compose down
$ docker-compose up -d
```

**WITHOUT RESTART:** 
> :warning: Change `test_default` in below command according to your needs.

```
$ docker-compose exec nginx-rp nginx -s reload
$ docker network disconnect test_default nginx-rp_nginx-rp_1
```
