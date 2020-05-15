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
The NGINX instances on the upstream servers should not expose ports on the main host.

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

Default configuration
------------

1. The domain is set in the `nginx/nginx.conf` separetly for HTTPS and HTTP. By default it is `localhost.local`.

```
stream {
    ...

    map $ssl_preread_server_name $targetUpstream {
        ~^(?<upstream>.+).localhost.local $upstream;
    }

    ...
}

http {
    ...

    map $http_host $targetUpstream {
        ~^(?<upstream>.+).localhost.local $upstream;
        ...
    }

    ...
}
```

2. By default all configuration is preconfigured for `test.localhost.local` and `test2.localhost.local` with:
* `test_nginx_1` and `test2_nginx_1` NGINX containers (for both HTTP and HTTPS)
* `test_default` and `test2_default` networks


Adding new site
------------

1. Add proper docker network to the `docker-compose.yml`.

> :warning: Change `test` to a unique name of your choice and `test_default` to the upstream's container network.

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

2. Add proper upstream configuration to the `nginx/nginx.conf`.

> :warning: Change `test` to match your subdomain and `test_nginx_1` to the upstream's NGINX container name.

**For HTTPS**:
```
stream {
    ...

    upstream test {
        server test_nginx_1:443;
    }

    ...
}
```

**For HTTP**:
```
http {
    ...

    upstream test {
        server test_nginx_1:80;
    }

    ...
}
```

You can also change the default upstream server which is used for `docker-compose` healhchecking:
```
http {
    ...

    map $http_host $targetUpstream {
        ...
        default test;
    }

    ...
}
```

3. Restart or reload the NGINX service.

**With restart**:
```
$ docker-compose down
$ docker-compose up -d
```

**Without restart:** 
> :warning: Change `test_default` in below command to the upstream's container network.

```
$ docker network connect test_default nginx-rp_nginx-rp_1
$ docker-compose exec nginx-rp nginx -s reload
```

Removing the site
------------------------------------

1. Remove the network from the `docker-compose.yml`. See [adding new site](#adding-new-site)

2. Remove the upstream configuration from the `nginx/nginx.conf`. See [adding new site](#adding-new-site)

3. Restart or reload the NGINX service:

**With restart**:
```
$ docker-compose down
$ docker-compose up -d
```

**Without restart:** 
> :warning: Change `test_default` in below command to the upstream's container network.

```
$ docker-compose exec nginx-rp nginx -s reload
$ docker network disconnect test_default nginx-rp_nginx-rp_1
```
