# Basic server services in a docker compose

> This is a file for me to have a traefik, adminer, backweb docker compose file that i can copy to my server
> and run `docker compose up -d` and then have other docker containers register their subdomains to and allow them
> to be setup for backup through the same backweb service and look at the data through the same adminer service.

## TLDR

1. Rename .env.example to .env and fill out env variables with your own.

2. Create networks for docker

```sh
docker network create public-traefik
docker network create local-backweb
docker network create local-adminer
```


3. Run the containers
    * prod: `docker compose -f docker-compose.yml up -d`
    * dev: `docker compose up -d`
4. Add networks to your services in your projects docker compose file.
    * `public-traefik` and labels to services you want to expose
    * `local-backweb` for databases that backweb should reach for backup
    * `local-adminer` for databases that adminer should reach


## Services available:

Traefik: `traefik.{DOMAIN}`
Adminer: `adminer.{DOMAIN}`
Backweb: `backweb.{DOMAIN}`

ie: `traefik.localhost`
ie: `traefik.example.com`


## Setup

You need to set the env variables shown in the .env.example either by making an `.env` file, exportin variables before running
`docker compose up -d` or just add them in the `docker` command.

when running locally just add the `.env` file, update it and run `docker compose -f docker-compose.yml up -d`


## Deployed on your server

### On the server:

* Create a directory to store your Docker Compose file:

```bash
mkdir -p /root/code/server-services/
```

* Create a docker network that other docker containers will use if they need to be accessed from the outside

```bash
docker network create public-traefik
```

### Locally

* Copy the Traefik Docker Compose file to your server. You could do it by running the command `rsync` in your local terminal:

```bash
rsync -a docker-compose.yml root@your-server.example.com:/root/code/server-services/
```

* Copy the env.example file now or you could also edit it locally first if you want.

```bash
cp .env.example .env
```

* Edit vars

* Use openssl to generate the "hashed" version of the password for HTTP Basic Auth:

```bash
openssl passwd -apr1 this-will-be-your-admin-password
```

> edit the .env file with the new password that looks something like: `$apr1$Iqxl9QOz$I8UqwmzEXk3S3/2BXIYOD0`

```bash
TRAEFIK_HASHED_PASSWORD="$apr1$Iqxl9QOz$I8UqwmzEXk3S3/2BXIYOD0"
```


* Change the domain name to your domain name

```bash
DOMAIN="example.com"
```

* Create an environment variable with the email for Let's Encrypt, e.g.:

```bash
EMAIL="admin@example.com"
```

**Note**: you need to set a different email, an email `@example.com` won't work.


* Copy the edited .env file to the server by running the command `rsync` in your local terminal:

```bash
rsync -a .env root@your-server.example.com:/root/code/server-services/
```

### On the server

### Start the Docker Compose

Go to the directory where you copied the Docker Compose file in your remote server:

```bash
cd /root/code/server-services/
```

Now with the environment variables set and the `docker-compose.yml` in place, you can start the Docker Compose running the following command:

```bash
docker compose up -d
```


To register a service with this traefik instance add the following to your services docker compose file:

This requires the env variables DOMAIN and PROJECT_NAME to be present.

using env_file: required: false, is convenient for local development so you can just have the vars in a file.

For deploying, add them to your github action secrets or whatever you use to deploy.

```yml
networks:
  public-traefik:
    # Allow setting it to false for testing
    external: true

services:

  # My service has no subdomain and registeres to traefik directly to DOMAIN ie test.com
  my-service:
    networks:
      - public-traefik
      - default
    env_file:
      - path: .env
        required: false
    labels:
      - traefik.enable=true
      - traefik.docker.network=public-traefik
      - traefik.constraint-label=public-traefik

      - traefik.http.services.${PROJECT_NAME?Variable not set}.loadbalancer.server.port=8000

      - traefik.http.routers.${PROJECT_NAME?Variable not set}-http.rule=Host(`${DOMAIN?Variable not set}`)
      - traefik.http.routers.${PROJECT_NAME?Variable not set}-http.entrypoints=http

      - traefik.http.routers.${PROJECT_NAME?Variable not set}-https.rule=Host(`${DOMAIN?Variable not set}`)
      - traefik.http.routers.${PROJECT_NAME?Variable not set}-https.entrypoints=https
      - traefik.http.routers.${PROJECT_NAME?Variable not set}-https.tls=true
      - traefik.http.routers.${PROJECT_NAME?Variable not set}-https.tls.certresolver=le

      # Enable redirection for HTTP and HTTPS
      - traefik.http.routers.${PROJECT_NAME?Variable not set}-http.middlewares=https-redirect
```



# ENV VARIABLES REQUIRED

```bash
DOMAIN
TRAEFIK_USERNAME
TRAEFIK_HASHED_PASSWORD
TRAEFIK_EMAIL
BACKWEB_SECRET_KEY

```

# Example labels, change <service> with your service name and <subdomain> with the subdomain you want to register

```yml
    labels:
      - traefik.enable=true
      - traefik.docker.network=public-traefik
      - traefik.constraint-label=public-traefik
      - traefik.http.routers.{PROJECT_NAME}-<service>-http.rule=Host(`<subdomain>.${DOMAIN?Variable not set}`)
      - traefik.http.routers.{PROJECT_NAME}-<service>-http.entrypoints=http
      - traefik.http.routers.{PROJECT_NAME}-<service>-http.middlewares=https-redirect
      - traefik.http.routers.{PROJECT_NAME}-<service>-https.rule=Host(`<subdomain>.${DOMAIN?Variable not set}`)
      - traefik.http.routers.{PROJECT_NAME}-<service>-https.entrypoints=https
      - traefik.http.routers.{PROJECT_NAME}-<service>-https.tls=true
      - traefik.http.routers.{PROJECT_NAME}-<service>-https.tls.certresolver=le
      - traefik.http.services.{PROJECT_NAME}-<service>.loadbalancer.server.port=8085
```

# Example without subdomain

```yml
    labels:
      - traefik.enable=true
      - traefik.docker.network=public-traefik
      - traefik.constraint-label=public-traefik
      - traefik.http.routers.{PROJECT_NAME}-<service>-http.rule=Host(`${DOMAIN?Variable not set}`)
      - traefik.http.routers.{PROJECT_NAME}-<service>-http.entrypoints=http
      - traefik.http.routers.{PROJECT_NAME}-<service>-http.middlewares=https-redirect
      - traefik.http.routers.{PROJECT_NAME}-<service>-https.rule=Host(`${DOMAIN?Variable not set}`)
      - traefik.http.routers.{PROJECT_NAME}-<service>-https.entrypoints=https
      - traefik.http.routers.{PROJECT_NAME}-<service>-https.tls=true
      - traefik.http.routers.{PROJECT_NAME}-<service>-https.tls.certresolver=le
      - traefik.http.services.{PROJECT_NAME}-<service>.loadbalancer.server.port=8085
```
