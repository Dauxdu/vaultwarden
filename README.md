# **🤔 HOW TO install vaultwarden using docker & traefik**
###### Thank you mrgukki ❤️

## Official repository
The guide is based on the [official](https://github.com/dani-garcia/vaultwarden) dani-garcia repository.

### 0. ✏️ DNS A record
Before proceeding, it is advisable to immediately enter a DNS A record in your domain editor with the format vw.your.domain

### 1. 🐧 Linux
Updating repositories and installing kernel updates
```
apt-get update
apt-get upgrade -y
```
Then reboot the operating system, and after the reboot navigate to /home/deploy
```
cd /home/deploy
```

### 2. 🐳 Docker
Install Docker
```
curl -fsSL get.docker.com -o get-docker.sh
CHANNEL=stable sh get-docker.sh
rm get-docker.sh
```
Initialise the docker swarm to run services as in docker-compose
```
docker swarm init
```

### 3. 🦫 Traefik
Creating a network for Traefik to communicate with the outside world
```
docker network create --driver=overlay traefik-public
```
Retrieve the variable as a node ID and store it in the cache
```
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
```
Specify a tag for Traefik on this node
```
docker node update --label-add traefik-public.traefik-public-certificates=true $NODE_ID
```
Exporting **EMAIL** and **DOMAIN** variables to obtain certificates in Traefik
```
export EMAIL=*email@mail.domain*
export DOMAIN=traefik.*your.domain*
```
Create a configuration file
```
nano traefik.yml
```
Insert the docker-compose text and save
```
version: '3.5'

services:

  traefik:
    # Use the latest v2.2.x Traefik image available
    image: traefik:v2.7
    ports:
      # Listen on port 80, default for HTTP, necessary to redirect to HTTPS
      - 80:80
      # Listen on port 443, default for HTTPS
      - 443:443
    deploy:
      placement:
        constraints:
          # Make the traefik service run only on the node with this label
          # as the node with it has the volume for the certificates
          - node.labels.traefik-public.traefik-public-certificates == true
      labels:
        # Enable Traefik for this service, to make it available in the public network
        - traefik.enable=true
        # Use the traefik-public network (declared below)
        - traefik.docker.network=traefik-public
        # Use the custom label "traefik.constraint-label=traefik-public"
        # This public Traefik will only use services with this label
        # That way you can add other internal Traefik instances per stack if needed
        - traefik.constraint-label=traefik-public
        # admin-auth middleware with HTTP Basic auth
        # Using the environment variables USERNAME and HASHED_PASSWORD
#        - traefik.http.middlewares.admin-auth.basicauth.users=${USERNAME?Variable not set}:${HASHED_PASSWORD?Variable not set}
        # https-redirect middleware to redirect HTTP to HTTPS
        # It can be re-used by other stacks in other Docker Compose files
        - traefik.http.middlewares.https-redirect.redirectscheme.scheme=https
        - traefik.http.middlewares.https-redirect.redirectscheme.permanent=true
        # traefik-http set up only to use the middleware to redirect to https
        # Uses the environment variable DOMAIN
        - traefik.http.routers.traefik-public-http.rule=Host(`${DOMAIN?Variable not set}`)
        - traefik.http.routers.traefik-public-http.entrypoints=http
        - traefik.http.routers.traefik-public-http.middlewares=https-redirect
        # traefik-https the actual router using HTTPS
        # Uses the environment variable DOMAIN
        - traefik.http.routers.traefik-public-https.rule=Host(`${DOMAIN?Variable not set}`)
        - traefik.http.routers.traefik-public-https.entrypoints=https
        - traefik.http.routers.traefik-public-https.tls=true
        # Use the special Traefik service api@internal with the web UI/Dashboard
        - traefik.http.routers.traefik-public-https.service=api@internal
        # Use the "le" (Let's Encrypt) resolver created below
        - traefik.http.routers.traefik-public-https.tls.certresolver=le
        # Enable HTTP Basic auth, using the middleware created above
#        - traefik.http.routers.traefik-public-https.middlewares=admin-auth
        # Define the port inside of the Docker service to use
        - traefik.http.services.traefik-public.loadbalancer.server.port=8080
    volumes:
      # Add Docker as a mounted volume, so that Traefik can read the labels of other services
      - /var/run/docker.sock:/var/run/docker.sock:ro
      # Mount the volume to store the certificates
      - ./certificates:/certificates
    command:
      # Enable Docker in Traefik, so that it reads labels from Docker services
      - --providers.docker
      # Add a constraint to only use services with the label "traefik.constraint-label=traefik-public"
      - --providers.docker.constraints=Label(`traefik.constraint-label`, `traefik-public`)
      # Do not expose all Docker services, only the ones explicitly exposed
      - --providers.docker.exposedbydefault=false
      # Enable Docker Swarm mode
      - --providers.docker.swarmmode
      # Create an entrypoint "http" listening on port 80
      - --entrypoints.http.address=:80
      # Create an entrypoint "https" listening on port 443
      - --entrypoints.https.address=:443
      # Create the certificate resolver "le" for Let's Encrypt, uses the environment variable EMAIL
      - --certificatesresolvers.le.acme.email=${EMAIL?Variable not set}
      # Store the Let's Encrypt certificates in the mounted volume
      - --certificatesresolvers.le.acme.storage=/certificates/acme.json
      # Use the TLS Challenge for Let's Encrypt
      - --certificatesresolvers.le.acme.tlschallenge=true
      # Enable the access log, with HTTP requests
      - --accesslog
      # Enable the Traefik log, for configurations and errors
      - --log
      # Enable the Dashboard and API
      - --api
    networks:
      # Use the public network created to be shared between Traefik and
      # any other service that needs to be publicly available with HTTPS
      - traefik-public

#volumes:
  # Create a volume to store the certificates, there is a constraint to make sure
  # Traefik is always deployed to the same Docker node with the same volume containing
  # the HTTPS certificates
#  traefik-public-certificates:

networks:
  # Use the previously created public network "traefik-public", shared with other
  # services that need to be publicly available via this Traefik
  traefik-public:
    external: true
```
Create a directory for certificates and deploy traefik
```
mkdir certificates
docker stack deploy -c traefik.yml traefik
```
Find out the container ID with ```docker ps``` and look at the logs
```
docker logs -f --tail 50 Container ID
```
If the deployment is successful, you will receive a message like this
```
time="2022-10-30T19:00:00Z" level=info msg="Configuration loaded from flags."
10.0.0.2 - - [30/Oct/2022:19:00:00 +0000] "GET /api/overview HTTP/2.0" 200 497 "-" "-" 1 "traefik-public-https@docker" "-" 0ms
10.0.0.2 - - [30/Oct/2022:19:00:00 +0000] "GET /api/http/routers?search=&status=&per_page=10&page=1 HTTP/2.0" 200 437 "-" "-" 2 "traefik-public-https@docker" "-" 0ms
```
### 4. 🛡 Vaultwarden
Exporting variables for Vaultwarden. ⚠️ **ADMIN_TOKEN** and **DOMAIN** should be changed to yours
```
export ADMIN_TOKEN=your_password
export SIGNUPS_ALLOWED=true
export DOMAIN=vw.your.domain
```
Create a configuration file
```
nano vw.yml
```
Insert the docker-compose text and save
```
version: '3.4'
services:
  vaultwarden:
    image: vaultwarden/server:latest
    environment:
      - ADMIN_TOKEN=${ADMIN_TOKEN}
      - SIGNUPS_ALLOWED=${SIGNUPS_ALLOWED}
#      - I_REALLY_WANT_VOLATILE_STORAGE=true
    volumes:
      - ./vw_data:/data
    deploy:
      labels:
        - traefik.enable=true
        - traefik.docker.network=traefik-public
        - traefik.constraint-label=traefik-public
        - traefik.http.routers.vaultwarden-http.rule=Host(`${DOMAIN?Variable not set}`)
        - traefik.http.routers.vaultwarden-http.entrypoints=http
        - traefik.http.routers.vaultwarden-http.middlewares=https-redirect
        - traefik.http.routers.vaultwarden-https.rule=Host(`${DOMAIN?Variable not set}`)
        - traefik.http.routers.vaultwarden-https.entrypoints=https
        - traefik.http.routers.vaultwarden-https.tls=true
        - traefik.http.routers.vaultwarden-https.tls.certresolver=le
        - traefik.http.services.vaultwarden.loadbalancer.server.port=80
    networks:
      - traefik-public
networks:
  traefik-public:
    external: true
```
Deploy vaultwarden
```
docker stack deploy -c vw.yml vaultwarden
```
If the deployment is successful, you will receive a message like this
```
/--------------------------------------------------------------------\
|                        Starting Vaultwarden                        |
|                           Version 1.26.0                           |
|--------------------------------------------------------------------|
| This is an *unofficial* Bitwarden implementation, DO NOT use the   |
| official channels to report bugs/features, regardless of client.   |
| Send usage/configuration questions or feature requests to:         |
|   https://vaultwarden.discourse.group/                             |
| Report suspected bugs/issues in the software itself at:            |
|   https://github.com/dani-garcia/vaultwarden/issues/new            |
\--------------------------------------------------------------------/

[INFO] No .env file found.

[2022-10-30 19:00:00.394][start][INFO] Rocket has launched from http://0.0.0.0:80
```
