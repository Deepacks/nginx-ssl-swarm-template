# nginx-ssl-template

This is a how-to guide to setup your host and enable an https connection over a Docker Swarm.

## IMPORTANT STUFF HERE

For this guide, a [Digital Ocean](https://www.digitalocean.com/) droplet running Ubuntu will be used, so keep in mind that OS commands may vary.

No knowledge of Docker Swarm or Nginx is needed.

You will need a working Docker image of your services uploaded on [hub.docker.com](https://hub.docker.com/) or any other docker registry to use this guide. To get a docker image you need a Dockerfile.

[This](https://docs.docker.com/engine/reference/builder/) is the complete documentation on how to create a Dockerfile, but you can check out some more specific guides. This process varies widely depending on what your are building (_how to build a docker image for React app?_).

This example will have:

- 1 node served react app
- 1 flask + gunicorn server
- 1 mongodb database

It doesn't matter what your configuration is, as long as you have valid docker iamges

This is intended to speed up the process of getting things up and running by providing a prebaked Nginx template.
You can of course tweak the Nginx configs that are not discussed in this guide.

_you should use your personal data/name/domain/host-ip where angle brackets are shown_

`<domain>` -> `example.com`

## 1) Requirements

First things first, the following requirements will be needed on your host:

### 1.1) Docker

Install docker through get.docker.com

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
docker --version #check docker is installed
```

### 1.2) Python & Certbot

Install python and then Certbot via pip

```bash
apt-get update
apt-get install python3 python3-pip
pip3 install certbot
```

_If you get any error during the installation, first try uninstalling certbot, python3 and python3-pip, then try again. For some reason it works._

```bash
pip3 uninstall certbot
apt-get remove python3 python3-pip
```

### 1.2) Micro

Install micro text editor (or any other text editor that you prefer)

```bash
apt-get install micro
```

## 2) Docker Swarm

Here is where the swarm will be set up. This allows you to eventually scale your services to a multi replica or multi node configuration.

### 2.1) Initialise Swarm

Init the swarm

```bash
docker swarm init --advertise-addr <your-host-ip>
```

### 2.1) Create overlay network

Create network with overlay driver

```bash
docker network create --driver overlay reverse_proxy_network
docker network ls #check network has been created
```

This network is accessible to all the nodes that join the swarm. Basically, services can speak to eachother via localhost even if they are on a different host.

[You want to join the swarm from another node?](https://docs.docker.com/engine/reference/commandline/swarm_join-token/)

[You have a join-token and you want to use it?](https://docs.docker.com/engine/reference/commandline/swarm_join/)

## 3) Docker images

Download your images from the registry ([hub.docker.com](https://hub.docker.com/))

### 3.1) Login to docker

Provide username and password to docker login

```bash
docker login
```

### 3.2) Pull images

Pull your images

```bash
docker pull <docker-username>/<image-name>:<image-version>
```

Repeat for every service (es. backend and frontend)

## 4) Docker Swarm services

`<your-service-name>` : you can choose the name. Important later.

`<app-port>` : the port on which your app is lestening on (es. 3000)

```bash
docker service create --name <you-service-name> \
    -d --with-registry-auth \
    --network reverse_proxy_network \
    --publish <app-port>:<app-port> \
    <docker-username>/<image-name>:<image-version>
```

Repeat for every service (es. backend and frontend)

```bash
docker service ls #check that services are running
```

## 5) (Optional) Configure Database

Databases requires some extra attention to be run correctly as a service

### 5.1) Pull database image

Check [hub.docker.com](https://hub.docker.com/) to find your db image and config instructions

If you are running MongoDB you can use [my image](https://hub.docker.com/r/deepacks/mongodb_with_authentication), as it contains a modified CMD that allows you to secure your db behind authentication (more on it later).

```bash
docker pull <docker-username>/<image-name>:<image-version>
```

### 5.2) Create Volume

Every docker database image should provide you a path where the container internal data is stored (/data/db for mongo)

A Volume is needed to keep the db data on your host even if the service stops. If there is no volume binded to the service, should it restart or stop, the data will be lost.

```bash
docker volume create <volume-name>
docker volume ls #check volume is created
```

### 5.3) Run Database Service

`<db-port>` : your db port (27017 for MongoDB)

`<volume-name>` : the name you gave to the volume

`<db-image-data-path>` : the path provided by the db image creators where data is stored inside the container (/data/db for MongoDB)

```bash
docker service create --name mongodb \
    -d \
    --network reverse_proxy_network \
    --publish <db-port>:<db-port> \
    --mount 'type=volume,src=<volume-name>,dst=<db-image-data-path>' \
    <docker-user>/<image-name>:<image-version>
```

### (Optional) Configure MongoDB Authentication

To follow the [MongoDB SCRAM guide](https://www.mongodb.com/docs/manual/tutorial/configure-scram-client-authentication/) and run commands on the database use

```bash
docker container exec -ti <replica-identifier> mongosh
```

To get the `<replica-identifier>` start by typing the db service name and press tab to autocomplete.

If autocomplete doesn't work get db container name from

```bash
docker container ls
```

it looks like this: mongodb.1.arhhvzbklqyu58tqgagz9n68w

Once you have created the first Admin user from the guide, you shoud run mongo commands using

```bash
docker container exec -ti mongodb.1.arhhvzbklqyu58tqgagz9n68w mongosh -u <mongo-user-name>
```

At this point you can check if volume works by running

```bash
docker service scale <database-service-name>=0
docker service scale <database-service-name>=1
```

This forces the swarm to create a new container.

If you are able to authenticate to the database, navigate to the admin db and show the users you should see that the admin user you just created is still there.

## 6) NGINX template download

It's time to configure nginx to work as a reverse proxy

### 6.1) Make directory

```bash
mkdir /usr/src/app/nginx
```

### 6.2) Clone template repo

Clone repo and prepare nginx folder

```bash
git clone https://github.com/Deepacks/nginx-ssl-swarm-template.git
mv nginx-ssl-swarm-template/* nginx/
rm -rf nginx-ssl-swarm-template
cd nginx
rm conf.d/sites-available/available.txt \
    conf.d/sites-enabled/enabled.txt \
    certs/readme.txt \
    certs/www/readme.txt
```

### 6.3) Run service

The Nginx service is created with a bind mount. This allows the container to run the config that we have on a local folder of the host

Both port 80 and 443 (default for respectively http and https protocols) are published for this service

```bash
docker service create --name nginx_rp \
    --mount 'type=bind,src=/usr/src/app/nginx,dst=/etc/nginx' \
    -p 80:80 -p 443:443 \
    --network reverse_proxy_network \
    nginx:latest
```

You should now have all your services up and running

```bash
docker service ls #Every service should show 1/1 running
```

## 7) Certbot

You will need certificates both for www and non-www to allow a redirect from www to non-www

You will also need to follow the certbot instructions to obtain your certificates. In this case the manual certification option will be used.

When prompted, save certificates inside default location.

### 7.1) Create certificates

First run

```bash
certbot -d <domain>  --manual --preferred-challenges dns certonly
```

Then run

```bash
certbot -d www.<domain>  --manual --preferred-challenges dns certonly
```

### 7.2) Copy certificates

Now copy the new certificates inside their respective nginx folder

Non-www certificates

```bash
cp <letsencrypt-folder>/<domain>/fullchain.pem \
    <letsencrypt-folder>/<domain>/privkey.pem \
    /usr/src/app/nginx/certs
```

`<letsencrypt-folder>` : The folder that points to your letsencrypt live certificates (default is /etc/letsencrypt/live/)

Www certificates (**IMPORTANT: if not copying, notice /www after /usr/src/app/nginx/certs**)

```bash
cp <letsencrypt-folder>/www.<domain>/fullchain.pem \
    <letsencrypt-folder>/www.<domain>/privkey.pem \
    /usr/src/app/nginx/certs/www
```

### 7.3) Generate DHPARAMS

Generate DHPARAM (this will take a little while) inside nginx certs folder

```bash
cd /usr/src/app/nginx/certs
openssl dhparam -out dhparams.pem 4096
```

## 8) Configure Nginx

### 8.1) Configure redirects

navigate to sites-available folder, create config file and open it with text editor

```bash
cd /usr/src/app/nginx/conf.d/sites-available
touch redirects.conf
micro redirects.conf
```

Using micro paste the following code, paying attention to all the places where your `<domain>` is required (5 times)

The following allows nginx to redirect http connections to https and www to non-www

```nginx
server {
    listen 80;
    server_name <domain> www.<domain>;
    return 301 https://<domain>$request_uri;
}

server {
    listen 443 ssl http2;
    server_name www.<domain>;

    include /etc/nginx/conf.d/common.conf;
    include /etc/nginx/conf.d/ssl_www.conf;

    return 301 https://<domain>$request_uri;
}
```

### 8.2) Configure services

Create config file and edit it with micro

```bash
touch services.conf
micro services.conf
```

`<frontend-upstream-name>` and `<backend-upstream-name>` : give them watever name you like (es. "fe" and "be").

`<frontend-service-name>` and `<backend-service-name>` : These are the names you gave to the swarm services. This works because the services are accessible through the overlay network via ip addres as well as their alias.

```nginx
upstream <frontend-upstream-name> {
        server  <frontend-service-name>:<frontend-service-port>;
}

upstream <backend-upstream-name> {
        server  <backend-service-name>:<backend-service-port>;
}

server {
        listen  443 ssl;
        server_name <domain>;

        include /etc/nginx/conf.d/common.conf;
        include /etc/nginx/conf.d/ssl.conf;

        location / {
                proxy_pass http://<frontend-service-name>:<frontend-service-port>;
                include /etc/nginx/conf.d/common_location.conf;
        }

        location /api/v1 {
                proxy_pass http://<backend-service-name>:<backend-service-port>;
                include /etc/nginx/conf.d/common_location.conf;
        }
}
```

Replace all your services names and add as many locations as you need. Keep in mind that only one service can aswer to the root route. All the other services must be able to answer on different routes.

### 8.3) Add SymLinks inside sites-enabled

Navigate to sites-enabled and add symbolic links to the sites-available configs

```bash
cd ../sites-enabled
ln -s ../sites-available/redirects.conf .
ln -s ../sites-available/services.conf .
```

## 9) Reload Nginx

These are the final steps.

### 9.1) Check Nginx config

This test will respond whether your nginx config is valid

```bash
docker container exec <nginx-replica-identifier> nginx -t
```

`<nginx-replica-identifier>` : you can start typing nginx than press tab for autocompletion.

If autocomplete doesn't work get nginx container name from

```bash
docker container ls
```

it looks like this: nginx_rp.1.16s26q4e1qrmzkzvh49t5bqdm

The nginx test command shoud respond with

```bash
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### 9.2) Reload Nginx

To reload Nginx with config run

```bash
docker container exec <nginx-replica-identifier> nginx -s reload
```

# Authors

- [Vladimir Cuneaz](https://github.com/deepacks)
- [Jacopo De Gattis](https://github.com/jacopo-degattis)
