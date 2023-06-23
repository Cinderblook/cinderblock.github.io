---
author: "Austin Barnes"
title: "Rancher behind Traefik - K3S"
description: "Running Traefik, Rancher, and Mysql containers, then connecting a lightweight kubernetes cluster to it for management."
tags: ["traefik", "networking", "github", "rancher", "homelab", "k3s"]
date: 2023-06-20
thumbnail: /covers/rancher-traefik.png
---

# Using Rancher and Traefik to provide certificates and a management platform to a K3S Cluster

In this guide, we will be creating an environment consisting of Rancher, Traefik, MySQL, and a K3S cluster. By setting it up this way, we can automate certificate management through Cloudflare (For more details about Cloudflare certificates behind Traefik, refer to other posts, as this won't be covered here.)

## Prerequisites

To get started, you'll need:

1. Four Virtual Machines (VMs). In this guide, I'm using VMs running Ubuntu Server 22.04. One VM will host our Docker containers, and the other three will serve as our K3S nodes.
2. Familiarity with Docker and Docker-compose.
3. Basic understanding of load balancing and networking.
4. An internal DNS server for name resolution.

Let's get started.



## Setting up Traefik

Our first task is to set up our Traefik container, which will provide TLS certificate management for Rancher and the K3S nodes. We need to create directories to host the necessary files.

Create the diretories to host the files, and the files reqruied.

```bash
mkdir traefik
mkdir traefik/data/
touch ./traefik/docker-compose.yml ./traefik/data/traefik.yml ./traefik/data/config.yml
```

Here are the contents for the respective files:

docker-compose.yml
```yml
version: '3'

services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
    ports:
      - "80:80" #HTTP
      - "443:443" #HTTPS
    environment:
      - CF_API_EMAIL=Cloudflare-email-here
      - CF_API_KEY=Cloudflare-api-key-here
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./logs/traefik-access.log:/var/log/traefik
      - ./data/traefik.yml:/traefik.yml:ro
      - ./data/acme.json:/acme.json
      - ./data/config.yml:/config.yml:ro
      #- ./.htpasswd:/auth/.htpasswd

    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.entrypoints=http"
      - "traefik.http.routers.traefik.rule=Host(`FQDN-Traefik-Here`)"
      - "traefik.http.middlewares.traefik-auth.basicauth.users=${BASICAUTHUSER}"
      # Take note, using a $ in the .env file, will double the amount of '$' signs being pulled over.
      # Double check config variables with 'docker-compose config' to see .env results
      # Generate BASID_AUTH_PASS: echo $(htpasswd -nb "<USER>" "<PASSWORD>") | sed -e s/\\$/\\$\\$/g
      - "traefik.http.middlewares.traefik-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.middlewares.sslheader.headers.customrequestheaders.X-Forwarded-Proto=https"
      - "traefik.http.routers.traefik.middlewares=traefik-https-redirect"
      - "traefik.http.routers.traefik-secure.entrypoints=https"
      - "traefik.http.routers.traefik-secure.rule=Host(`FQDN-Traefik-Here`)"
      - "traefik.http.routers.traefik-secure.middlewares=traefik-auth"
      - "traefik.http.routers.traefik-secure.tls=true"
      - "traefik.http.routers.traefik-secure.tls.certresolver=cloudflare"
      - "traefik.http.routers.traefik-secure.tls.domains[0].main=Domain.com"
      - "traefik.http.routers.traefik-secure.tls.domains[0].sans=*.Domain.com"
      - "traefik.http.routers.traefik-secure.service=api@internal"

networks:
  proxy:
    external: true
```

traefik.yml
```yml
api:
  dashboard: true
  debug: true

entryPoints:
  http:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: "https"
          scheme: "https"
  https:
    address: ":443"

  k3s:
    address: ":6443"    
serversTransport:
  insecureSkipVerify: true
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
  file:
    filename: /config.yml
certificatesResolvers:
  cloudflare:
    acme:
      email: ${CF_API_EMAIL}
      storage: acme.json
      dnsChallenge:
        provider: cloudflare
        #disablePropagationCheck: true # uncomment this if you have issues pulling certificates through cloudflare, By setting this flag to true disables the need to wait for the propagation of the TXT record to all authoritative name servers.
        resolvers:
          - "1.1.1.1:53"
          - "1.0.0.1:53"
log:
  level: INFO
  filePath: "/var/log/traefik/traefik.log"
accessLog:
  filePath: "/var/log/traefik/access.log"
```

config.yml
```yml
http:
 #region routers
  routers:

# This is an extremely simplified Traefik configuration file, as the HTTP portion is empty since Rancher will be setting up its 
# Labeling structure within the docker-container. See my GitHub for a more advanced version of this config file.

#region services
  services:

#endregion
  middlewares:

# Redirect all http traffic to https
    https-redirectscheme:
      redirectScheme:
        scheme: https
        permanent: true
# Add default headers for redirection
    default-headers:
      headers:
        frameDeny: true
        browserXssFilter: true
        contentTypeNosniff: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 15552000
        customFrameOptionsValue: SAMEORIGIN
        customRequestHeaders:
          X-Forwarded-Proto: https

# Whitelist all private IP addresses
    default-whitelist:
      ipWhiteList:
        sourceRange:
          - "10.0.0.0/8"
          - "192.168.0.0/16"
          - "172.16.0.0/12"
#
    secured:
      chain:
        middlewares:
          - default-whitelist
          - default-headers

tcp:
  routers:
# k3s port
    k3s:
      entryPoints: k3s
      rule: "HostSNI(`*`)"
      service: k3s
      tls: {}
      middlewares:
        - "default-whitelist"
# Load balancers - Services
  services:
# k3s loadbalancer
    k3s:
      loadBalancer:
        servers:
        - address: "192.168.1.235:6443"
        - address: "192.168.1.236:6443"
        - address: "192.168.1.237:6443"

# Whitelist all private IP addresses
  middlewares:
    default-whitelist:
      ipWhiteList:
        sourceRange:
          - "10.0.0.0/8"
          - "192.168.0.0/16"
          - "172.16.0.0/12"
```

If you need clarity or assistance in setting up Cloudflare to hand out certificates for Traefik, there are plenty of guides out there! Then start up the docker-compose file.yml from the root of the ./traefik folder

```bash
docker-compose up -d 
```

## Setting up Rancher
We will now create a directory for Rancher and place a Docker-compose.yml file inside it.

``` bash
mkdir rancher
nano ./rancher/docker-compose.yml
```

Fill the docker-compose with:

```yml
version: '3.9'
services:
    rancher:
        # --no-cascerts allows for a wildcard cert to be applied, comment out if you will not be using an assigned certificate.
        command: '--no-cacerts'
        image: 'rancher/rancher:latest'
        #dns: #Put your DNS servers below
        #    - 192.168.1.253
        #    - 1.1.1.1
        privileged: true
        ports:
            - '9443:443'
            - '9080:80'
        networks:
            - proxy
        volumes:
            - ./opt/rancher:/var/lib/rancher
        restart: unless-stopped
        # If not using Traefik, comment this out for your SSL/TLS solution.
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.rancher.entrypoints=http"
          - "traefik.http.routers.rancher.rule=Host(`rancher.cinderblock.tech`)"
          - "traefik.http.middlewares.rancher-https-redirect.redirectscheme.scheme=https"
          - "traefik.http.routers.rancher.middlewares=rancher-https-redirect"
          - "traefik.http.routers.rancher-secure.entrypoints=https"
          - "traefik.http.routers.rancher-secure.rule=Host(`rancher.cinderblock.tech`)"
          - "traefik.http.routers.rancher-secure.tls=true"
          - "traefik.http.routers.rancher-secure.service=rancher"
          - "traefik.http.services.rancher.loadbalancer.server.port=80"
          - "traefik.docker.network=proxy"

networks:
  proxy:
    external: true
```

After completing the setup, you can start it up. I highly recommend setting up an SSL/TLS certificate for Rancher, otherwise you'll have to run it without a certificate, which is insecure.

```bash
docker-compose up -d
```

## Setting up Mysql
Next, we will create a MySQL Docker container. Start by creating a directory and a Docker-compose file in it.

```bash
mkdir mysql
cd mysql
nano docker-compose.yml
```

Fill the Docker-compose file with the following:

```yml
version: '3.9'

services:
  mysql:
    image: mysql:latest
    restart: unless-stopped
    container_name: k3s-mysql
    ports:
      - 3396:3306
    expose:
      - "3396"
    volumes:
      - ./mysql-data:/var/lib/mysql/
      - ./setup.sql:/docker-entrypoint-initdb.d/setup.sql
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_PASS} # Put your sql password here for Root access
```

Setup a script file to automatically create the SQL database and uses:

```bash
nano setup.sql
```

Put the following SQL command into the setup.sql file, and ensure to change `user`, `password`, to respective pieces of information:

```sql
CREATE DATABASE k3s COLLATE latin1_swedish_ci;
CREATE USER 'user'@'%' IDENTIFIED BY 'password';
GRANT ALL ON k3s.* TO 'user'@'%';
FLUSH PRIVILEGES;

```
Start it up!

```bash
docker-compuse up -d
```

## Setting up the primary node.
Ensure you have Ubuntu Server 22.04 setup and updated. Also ensure these few things are in order:

1. `curl` package is installed 
2. IPs are in order
3. DNS is pointing correctly

On the primary node, you will want to run the following:
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.25.9+k3s1 sh -s - server \
  --tls-san ${FQDN-load-balancer-HERE} \
  --datastore-endpoint="mysql://${user}:${pass}@tcp(${IP-MYSQL}:3396)/${database-name}"
# Replace '${}` with your values
```

Once this installs successfully, run the following cat command on the node, to get the required token to setup the next two nodes to the same cluster:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Copy that token, we will use it in the next part.

## Setting up the other two nodes.

SSH into the next node. Run the following command to join it to the K3S cluster:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.25.9+k3s1 K3S_URL=https://${FQDN-LOAD-BANACER-HERE}:6443 K3S_TOKEN=${TOKEN-FROM-PRIMARY-NODE-HERE} sh -s - server \
  --tls-san ${FQDN-load-balancer-HERE} \
  --datastore-endpoint="mysql://${user}:${pass}@tcp(${IP-OF-MYSQL}:3396)/${database-name}"
  #Replace values '${}` with your values
```

Repeat for the second node.

Once this is done, test it is working with 

```bash
sudo kubectl get nodes
```

If the result shows the three nodes together, congragulations, you have a K3S cluster.

## Connect K3S cluster to rancher management panel
Navigate to your Rancher Webportal, sign in, and go to the Cluster Management tab.

1. Select Import Existing Cluster
2. Select Generic
3. Give it a name, and hit create
4. Copy the top command under `Registration`, and give rancher the ability to command Kubectl. Run it on all three k3s nodes.

Then you are good and free to have Rancher manage your k3s cluster.

## Useful Resources

Here are some useful resources for further exploration:

* [Traefik site](https://traefik.io/)
* [My GitHub Repository](https://github.com/Cinderblook/tacklebox/tree/)
* [Rancher](https://www.rancher.com/)
