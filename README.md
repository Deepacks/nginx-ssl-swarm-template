# nginx-ssl-template

This is a how-to guide to setup your host and enable an https connection over a Docker Swarm.

## IMPORTANT STUFF HERE

A [Digital Ocean](https://www.digitalocean.com/) droplet running Ubuntu will be used.

You will need a working Docker image of your services uploaded on [hub.docker.com](https://hub.docker.com/) or any other docker registry.

This is intended to speed up the process of getting things up and running by providing a prebaked Nginx template.
You can of course tweak the Nginx configs that are not discussed in this guide.

_you should use your personal data/name/domain/host-ip where angle brackets are shown_

ex: `<domain>` -> `example.com`

## 1) Requirements

The following steps make sure all requirements are met

### 1.1) Docker

Install docker through get.docker.com (you can skip this if you chose the droplet starting image with docker from the DO marketplace)

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
docker --version #check docker is installed
```

### 1.2) Python & Certbot

Install python and then Certbot via pip

```bash
apt update
apt install python3 python3-pip
pip3 install certbot
```

### 1.2) Micro

Install micro text editor (or any other text editor you may prefer)

```bash
apt install micro
```

## 2) Confifure Nginx

Clone and setup Nginx config

### 2.1) Clone repo

Create app folder (the path to the repo will be important later)

```bash
cd /usr/src/ && \
mkdir app && \
cd app
```

Clone this repo

```bash
git clone https://github.com/Deepacks/nginx-ssl-swarm-template.git nginx
```

Remove unnecessary files

```bash
cd nginx && \
rm conf.d/sites-enabled/enabled.txt \
    certs/readme.txt \
    certs/www/readme.txt
```

### 2.2) Setup redirects

Edit redirects.conf

```bash
cd conf.d/sites-available
micro redirects.conf
```

Set the `domain` where `<domain>` is present

### 2.3) Setup services

Edit services.conf

```bash
micro services.conf
```

`<frontend-service-name>` and `<backend-service-name>`: These are the names of the docker services (next steps)

ex: `my-app-frontend` and `my-app-backend`

`<frontend-service-port>` and `<backend-service-port>`: These are the ports exposed by the docker image

### 2.4) Add symlinks inside sites-enabled

Navigate to sites-enabled and add symbolic links to the sites-available configs

```bash
cd ../sites-enabled
ln -s ../sites-available/redirects.conf .
ln -s ../sites-available/services.conf .
```

## 3) Certificates

This section is dedicated to creating certificates

### 3.1) Certbot

Create domain certificate

```bash
certbot -d <domain>  --manual --preferred-challenges dns certonly
```

Create www.domain certificate

```bash
certbot -d www.<domain>  --manual --preferred-challenges dns certonly
```

### 3.2) Import certificates

Copy the new certificates inside the nginx configuration

Non-www certificates

```bash
cp <letsencrypt-folder>/<domain>/fullchain.pem \
    <letsencrypt-folder>/<domain>/privkey.pem \
    /usr/src/app/nginx/certs
```

`<letsencrypt-folder>` : The folder that points to your letsencrypt certificates (default is /etc/letsencrypt/live/)

Www certificates (**IMPORTANT: if not copying, notice /www after /usr/src/app/nginx/certs**)

```bash
cp <letsencrypt-folder>/www.<domain>/fullchain.pem \
    <letsencrypt-folder>/www.<domain>/privkey.pem \
    /usr/src/app/nginx/certs/www
```

### 3.3) Generate DHPARAMS

Generate DHPARAM (this will take a little while) inside nginx certs folder

```bash
cd /usr/src/app/nginx/certs
openssl dhparam -out dhparams.pem 4096
```

## 4) Setup Docker Environment

Here is how to set up the docker environment

### 4.1) Docker Swarm

Initialize the docker swarm

```bash
docker swarm init --advertise-addr <your-host-ip>
```

[You want to invite other nodes to join the swarm?](https://docs.docker.com/engine/reference/commandline/swarm_join-token/)

[You have a join-token and you want to use it?](https://docs.docker.com/engine/reference/commandline/swarm_join/)

### 4.2) Configure stacks

Create folder to hold the stacks compose files

```bash
cd /usr/src/app && \
mkdir stacks && \
cd stacks
```

Create stack files

`<stack_name>`: the name you want to give to your stack

ex: `my-app`

```bash
touch base.yml && \
touch <stack_name>.yml
```

Configure base.yml (nginx bind source must match path on server)

```yml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - type: bind
        source: /usr/src/app/nginx
        target: /etc/nginx
    networks:
      - proxy_net

  mongo:
    image: mongo
    volumes:
      - mongovolume:/data/db
    networks:
      - proxy_net

volumes:
  mongovolume:
    driver: local

networks:
  proxy_net:
    driver: overlay
```

Configure <stack_name>.yml

```yml
services:
  web:
    image: <docker-image>
    networks:
      - proxy_net

  server:
    image: <docker-image>
    networks:
      - proxy_net

networks:
  proxy_net:
    name: base_proxy_net
    external: true
```

### 4.2) Deploy stacks

Deploy the base stack

```bash
docker stack deploy -c base.yml base
```

Deploy the custom stack

```bash
docker stack deploy -c <stack_name>.yml <stack_name>
```

# Authors

- [Vladimir Cuneaz](https://github.com/deepacks)
- [Jacopo De Gattis](https://github.com/jacopo-degattis)
