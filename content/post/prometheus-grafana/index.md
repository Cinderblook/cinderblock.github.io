---
author: "Austin Barnes"
title: "Prometheus + Grafana monitoring"
description: "A solution for homelab monitoring of core systems."
tags: ["prometheus", "grafana", "github", "database", "homelab"]
categories: ["Monitoring","Database","Docker"]
date: 2023-07-3
image: prometheus-grafana-banner.png
---

# Utilizing Prometheus DB to create Grafana monitoring dashboards

In this guide, I'll be setting up a Prometheus database, and configuring Grafana dashboards for monitoring solutions within my homelab.

## Prerequisites

To get started, you'll need:

1. Systems that need monitored
2. Familiarity with Docker and Docker-compose.
3. An internal DNS server for name resolution.
4. SSL certificates for secure communication

Let's get started.

## Setup Prometheus
On the host you wish to run Prometheus on, go ahead and create a new directory called `Prometheus` and create two files in it.

```bash
mkdir prometheus
touch prometheus/docker-compose.yml
touch prometheus/prometheus.yml
```

Within that directory. put a docker-compose file in it, containing the following:

```yml
---
version: '3.9'

services:
  prometheus:
    image: prom/prometheus:latest # I recommend finding an image that works for you , and setting the version here.
    container_name: prometheus
    ports:
      - 9090:9090
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus-data/:/prometheus
    command: 
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'    
    restart: unless-stopped
    networks:
      - proxy
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    expose:
      - 9100
    networks:
      - proxy
      - monitoring

networks:
  proxy:
    external: true
  monitoring:
    external: false
```

Next to it, put a config file called `prometheus.yml` with this:

```yml
# This is a fairly bare bones config
global:
  scrape_interval: 1m

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 1m
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      # Put VMs that are going to be monitored with the node-exporter service, here.
      - targets: ['node-exporter:9100','IP-Node-#2:9100','...','...']

# Uncomment and fill in if using the Grafana Cloud solution
#remote_write:
#  - url: 'FQDN-HERE'
#    basic_auth:
#      username: ${GRAFANA-USER}
#      password: ${GRAFANA-PASS}
```

With those files setup, navigate to your directory and run:

```bash
docker compose up -d
```

### Prometheus Configuration
First thing to do, is to check to ensure it is running, and attempt to reach your site `https://FQDN:9090`

Next, setup your Basic Authentication user and password in that web portal. 

From here, you can also see Prometheus's connected Discovery Services and other detailed information from this panel.


## Setup Node-Exporter
[Node-Exporter](https://github.com/prometheus/node_exporter) is out solution to host monitoring. It is going to gather critical host data, such as CPU, Disk, RAM, and overall utilization data. This will vary on a case-by-case scenario, although for each system you wish to obtain data from, perform the following. (This could include the host-machine running prometheus!)

### Automated w/ Ansible
If you have [Ansible-Semaphore running like I do, and created a setup guide for](https://www.cinderblock.tech/p/ansible-semaphore-automating-updates/), or just Ansible capability in general, you could run this [ansible playbook from my GitHub repo](https://github.com/Cinderblook/tacklebox/blob/main/Network-Automation/Ansible/Playbooks/Linux/docker-node-exporter.yml) to automate the deployment of the Node-Exporter docker container on all your systems.

### Manual Effort
1. Navigate to the system in question
2. Create a directory for the docker-compose file 
    - `mkdir node-exporter`
3. Create the Docker-compose file
    - `nano docker-compose.yml`
      - ```yml
        node-exporter:
          image: prom/node-exporter:latest
          container_name: node-exporter
          restart: unless-stopped
          volumes:
            - /proc:/host/proc:ro
            - /sys:/host/sys:ro
            - /:/rootfs:ro
          command:
            - '--path.procfs=/host/proc'
            - '--path.rootfs=/rootfs'
            - '--path.sysfs=/host/sys'
            - '--collector.filesyste  mount-points-exclude=^/ (sys|proc|dev|host|et  ($$|/)'
          ports:
            - 9100:9100
        ```
4. Start the container
    - `docker compose up -d`

## Setup Cadvisor
[Cadvisor](https://github.com/google/cadvisor) is setup to monitor docker services running on hosts machines. Its biggest benefit is gaining insight into containers rather than the host itself, to monitor highest load of your virtualized instances. This is granular in comparison to Node-Exporter.

### Automated w/ Ansible
If you have a lot of hosts to run this on, I recommend an automated method. Similar to Node-Exporter, I'll be handling this with an Ansible playbook running off of [Ansible Semaphore](https://www.cinderblock.tech/p/ansible-semaphore-automating-updates/). The playbook in question can be found here on [my GitHub Repo](https://github.com/Cinderblook/tacklebox/blob/main/Network-Automation/Ansible/Playbooks/Linux/docker-cadvisor.yml). 

### Manual Effort
1. Navigate to the system in question
2. Create a directory for the docker-compose file 
    - `mkdir cadvisor`
3. Create the Docker-compose file
    - `nano docker-compose.yml`
      - ```yml
        cadvisor:
          image: gcr.io/cadvisor/cadvisor
          container_name: cadvisor
          restart: unless-stopped
          privileged: true
          volumes:
            - /:/rootfs:ro
            - /var/run:/var/run:rw
            - /sys:/sys:ro
            - /var/lib/docker/:/var/lib/docker:ro
          ports:
            - 8080:8080
          networks:
            - monitoring
        ```
4. Start the container
    - `docker compose up -d`


## Setup Grafana
On the host you wish to run Grafana on, similar to the Prometheus setup, create a directory for hosting all the files.

```bash
mkdir grafana
```

Within that grafana directory, create docker-compose.yml file and fill it with:

```yml
version: '3.9'

services:
  grafana:
    image: grafana/grafana:10.0.1
    # Set this to prevent permission errors during Grafana deployment - Sets user to Root
    user: "0" 
    container_name: grafana-monitor
    ports:
      - "3000:3000"
    volumes:
      - ./grafana-data:/var/lib/grafana
    #environment:
      #- "GF_SERVER_ROOT_URL=${GRAF_FQDN}
      #- GF_INSTALL_PLUGINS=${GRAF_PLUGIN_LIST}
      #- GF_DEFAULT_INSTANCE_NAME=${GRAF_NAME}
      #- GF_SECURITY_ADMIN_USER=${GRAF_ADMIN}
      #- GF_AUTH_GOOGLE_CLIENT_SECRET=${GRAF_SECRET}
      #- GF_PLUGIN_GRAFANA_IMAGE_RENDERER_RENDERING_IGNORE_HTTPS_ERRORS=true
    restart: unless-stopped
```

With that created, spin it up.

```bash
docker compose up -d
```

Then navigate to the URL and setup your user/pass for Grafana
  - https://FQDN:PORT

Once the account is setup, your Panel should look like this:
[!grafana-panel](grafana-example-01.png 'grafana-01')

Navgiate to `Administration --> Data Sources --> Add a New Data Source` and select `Prometheus`

Enter your URL, and Basic Auth settings for connection to the Prometheus DB
[!grafana-panel](grafana-example-02.png 'grafana-02')

At the bottom of the page, keep the HTTP method set to POST, and then hit Save & Test.


### Grafana Dashboard imports!
Now that we have the data flowing into Grafana, we need to either setup or import an existing Dashboard to view it in a managable manner. 

#### Node-Exporter Import/Dashboard

The Grafana site has plent of options. For our Node-Exporter, I will be using [Grafana Node-Exporter Full](https://grafana.com/grafana/dashboards/1860-node-exporter-full/)

For setup, navigate to the top of your grafana panel screen, hit the `+` symbol, and select `import dashboard`
[!grafana-panel](grafana-example-03.png 'grafana-03')

Put the ID/URL of the dashboard in the section for `Import via grafana.com` and hit `Load`

Once this loads in, you can navigate to `Dashboard --> General --> Node Exporter Full` and go back and forth between all of the nodes relaying data in a readable grpah formats. This dashboard can be edited and played with as need be.

[!grafana-panel](grafana-example-04.png 'grafana-04')

#### Cadvisor Import/Dashboard
Repeat steps in Node-Exporter Import/Dashboard section, except use [this dashboard designed for cadvisor](https://grafana.com/grafana/dashboards/14282-cadvisor-exporter/).


## Further benefits
The way to best benefit form this monitoring solution setup, it to see trends within the systems over-time, along with peak utilization, potential Storage issues, and to allow for visual overview all in one spot of otherwise typically tedious to obtain data.

I would suggest looking into monitoring alerts for Prometheus and Grafana, which I may cover later. Using something like a webhook to send alerts for data capacity, or CPU/RAM load of various systems can be extremely useful!


## Useful Resources

Here are some useful resources for further exploration:

* [Grafana Documentation](https://grafana.com/docs/grafana/latest/)
* [My GitHub Repository](https://github.com/Cinderblook/tacklebox/tree/)
* [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
