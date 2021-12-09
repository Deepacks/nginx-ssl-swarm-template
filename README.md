# nginx-ssl-template

_you should use your personal data/name/domain inside the angle brackets_

`<example>` -> `example`

## 1) Configure containers

### 1.1) Create network

First of all create a network using the bridge driver

```
docker network create --driver bridge <my_bridge>
```

### 1.2) Launch services (ex. FE & BE)

```
docker container run --network my_bridge \
    --name <container_name> \
    -p <host_port>:<container_port> \
    <image_name>:<image_tag>
```

## 2) Launch NGINX with bind mount

These steps allow a bind mount beteen the nginx config inside the container and an nginx folder on your host:

- Since bind mounts work from host to container we first need to have a working nginx config on our host before binding it.
- If we bind an empty nginx config folder to a new container the container will copy the empty folder from the host, replacing the valid config.

### 2.1) Run NGINX container

```
docker container run --name nginx nginx
```

### 2.2) Clone NGINX config repo into nginx folder

```
mkdir temp && cd temp
git clone https://github.com/Dying-Lately/nginx
mv -r ./nginx/* /etc/nginx
cd .. && rm -rf temp

```

### 2.3) Copy NGINX config from container to host

```
docker cp <container_name>:/etc/nginx <path/on/your/host>
```

### 2.4) Remove NGINX container

```
docker container rm -f nginx
```

### 2.5) Start NGINX with bind mount

```
docker container run --name nginx \
    --mount type=bind,source=<path/on/your/host/nginx>,target=/etc/nginx/ \
    -p 80:80 -p 443:443 \
    --network my_bridge \
    nginx:latest
```

## 3) Create certificate and DHPARAMS

### 3.1) Install certbot inside NGINX container

**Execute bash on container**

```
docker container exec -ti nginx bash
```

**Update system dependencies**

```
apt update
```

**Install python3 and pip3**

```
apt install python3 pip3-python
```

**Install certbot**

```
pip3 install certbot
```

**Create certificate manually (follow certbot instructions)**

```
sudo certbot -d <domain_name> \
    --manual --preferred-challenges dns certonly
```

**Copy certificate to NGINX certs folder**

```
cp /etc/letsencrypt/live/<domain_name>/fullchain.pem \
    /etc/letsencrypt/live/<domain_name>/privkey.pem \
    /etc/nginx/certs/
```

**Generate DHPARAM (this will take a little while)**

```
openssl dhparam -out dhparams.pem 4096
```

**Copy DHPARAM to NGINX certs folder**

```
cp ./dhparams.pem /etc/nginx/certs
```

Inside /etc/nginx/certs there should now be 3 files:

- dhparams.pem
- fullchain.pem
- privkey.pem

## 4) Configure available-sites

_from <path/on/your/host/nginx>_

This allows nginx to listen on the ports opened when the container was started

### 4.1) Install micro

```
apt install micro
```

### 4.2) Create front-end config file

```
touch .nginx/conf.d/sites-available/fe.conf
```

### 4.3) Edit fe.conf

```
micro .nginx/conf.d/sites-available/fe.conf
```

- Add (remember to edit with your domain name):

```
upstream fe {
        server  <front_end_container_name>:<front_end_container_port>;
}

server {
    listen 80;
    server_name <domain.com> www.<domain.com>;
    return 301 https://<domain.com>$request_uri;
}

server {
    listen 443 ssl http2;
    server_name www.<domain.com>;

    include /etc/nginx/conf.d/common.conf;
    include /etc/nginx/conf.d/ssl.conf;

    return 301 https://<domain.com>$request_uri;
}

server {
        listen  443 ssl;
        server_name <domain.com>;

        include /etc/nginx/conf.d/common.conf;
        include /etc/nginx/conf.d/ssl.conf;

        location / {
                proxy_pass http://<front_end_container_name>:3000;
                include /etc/nginx/conf.d/common_location.conf;
        }
}
```

## 4.4) Add a symbolic link to inside the sites-enabled

```
cd ./nginx/conf.d/sites-enabled && \
    ln -s ../sites-available/fe.conf .
```

## 5) Check nginx config and reload nginx

_Remember to exit container bash before proceding_

### 5.1) Check nginx config

```
docker container nginx nginx -t
```

You shoud see:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### 5.2) Reload nginx

```
docker container exec nginx -s reload
```

# Authors

- [Jacopo De Gattis](https://github.com/jacopo-degattis)
- [Vladimir Cuneaz](https://github.com/deepacks)
