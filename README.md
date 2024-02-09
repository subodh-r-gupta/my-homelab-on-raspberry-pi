# My home lab on Raspberry pi 4


I got a new raspberry pi 4 and I want to build a low cost and low power consuming home server on it. 

So, Here we go again...

## Which services do I want to run on it?

I need a home server which can provide the following services-

### Security Services

| Service | Description |
| ----------- | ----------- |
| pivpn | for accessing services over internet when i am travelling with my laptop  |
| cloudflare ddns | for providing me a dynamic dns server to connect my vpn clients to  |
| duckdns| for providing me a dynamic dns server to connect my vpn clients to  TBD in the version 2 of this guide|
| linux firewall | for blocking intrusions  |
| nginx proxy manager | all containers will be placed behind a proxy to reduce attack surface |
| traefik | all containers will be placed behind a proxy to reduce attack surface TBD in version 2 of this guide |

My previous experience with traefik was not good, the documentation was a mess to figure out, so this time i am going to have nginx proxy manager and traefik both. I will test and choose one.

Similarly i have used duckdns in past, it worked well for me. But i have heard good things about cloudflare so I would like to try both of them and then decide on one of them.

### Privacy Services

| Service | Description |
| ----------- | ----------- |
| pihole |  for Network-Wide Ad Blocking  |
| adguard |  for Network-Wide Ad Blocking  TBD in version 2 of this guide|

I am going to have pihole and adguard both for side by side comparison. I have been using pihole for years, but heard good things about adguards. So why not have then both and test which one works better for me?



### Server Management Services

| Service | Description |
| ----------- | ----------- |
| homarr |  for a nice looking server home page with links to all the services  |
| cockpit |  for a nice server manager gui  |
| portainer | for easy docker container management using web gui |

Yes, I know the for server management a CLI must be used but sometimes a GUI is good to cheat through.

### Server Monitoring and Alert services

| Service | Description |
| ----------- | ----------- |
| netdata |  for server and network monitoring and alerts   |
| zabbix |  for server and network monitoring and alerts  TBD in version 2 of this guide|
| glances | for a quick view of docker container health |
| Librenms | for server and network monitoring alerts TBD in version 2 of this guide |

This server needs to be monitored 24x7 and i should be able to get alerts when something fails or an attack happens. I would like to compare netdata, zabbix and librenms, so I will configure them all.Once i choose one over other, It will be cleaned up.


### Entertainment Services
| Service | Description |
| ----------- | ----------- |
| jellyfin |  for sharing music and video library to my laptop and tv  |
| minidlna |  for sharing music and video library to my laptop and tv  |
| calibre and calibre web |  for sharing comics and books library to my laptop and tv  |
| photoprism | for sharing photos to my laptop and tv |


In my previous tests on raspberry pi, jellyfin was maxing out CPU usage, So i will have a lighweight alternative as minidlna on docker host to fall back if it happens again.

### File Sharing Services

| Service | Description |
| ----------- | ----------- |
| samba |  for storing and sharing important personal documents within the LAN - TBD in version 2 of this guide |
|filebrowswer | storing and sharing important personal documents within the LAN |

I have a spare 500 GB HDD which I will use as network attached storage for storing all my data.

### Office Services
| Service | Description |
| ----------- | ----------- |
| nextcloud |  for office apps, document sharing and meetings with friends and family over the internet |

### Website hosting services

| Service | Description |
| ----------- | ----------- |
| wordpress |  for selfhosting my own website |

I am a cheapskate, I do not want to pay for hosting charges every year.

----

# Getting Started...

## 1 - Inventory -

- Raspberry Pi 4B 8 gb
- 128 gb SSD 
- Heat Sink
- Official Power Supply
- Rpi4 Case With Fan
- Micro HDMI to HDMI cable
- External HDD 500 GB

## 2 - Installing Ubuntu Server 2204 LTS on raspberry pi 4

I am using Ubuntu and Red Hat Linux at work on daily basis. 

But I have decided to choose Ubuntu server 2204 LTS as the base OS for this server because...

- they have the official LTS builds available for RPI
- ease of use.
- no money to buy support

This will be my docker host.

Steps -

1- download the 64-bit image from https://ubuntu.com/download/raspberry-pi/thank-you?version=22.04.3&architecture=server-arm64+raspi

2- flash the image using raspberry pi imager

3- plug the pi into your router and find it's ip

4- ssh is enabled by default so you should be able to ssh into it
     user: ubuntu
    pass: ubuntu

Upon first login ubuntu will ask you to change the password, do so then relog it over shh. 

On first boot ubuntu will expand the file system. 

after a short while update ubuntu using - 
```
> sudo apt update
> sudo apt upgrade.
```
---

## 3 - Installing docker

Now Lets install docker on this host.

Steps -

- Download the docker installer script from docker.com
> curl -fsSL https://get.docker.com -o get-docker.sh

- Execute the docker install script - 

> sudo sh get-docker.sh

- Add a regular user to docker group, so that you dont need to use sudo with docker commands

>sudo usermod -aG docker $USER

- reboot

> sudo reboot

- Test the installation

  > docker run hello-world




---

##  4 - Setting up services

Lets install server management services to begin with.

### 4.1 Installing cockpit

> sudo apt install cockpit 

### 4.1 Setup a private network

> docker create network  proxy-net

### 4.2 Setup portainer container

```

version: "3"
services:
    portainer:
      image: portainer/portainer-ce:latest
      container_name: portainer
      restart: unless-stopped
      privileged: true
      volumes:
        - ../../storage/portainer-data:/data
        - /var/run/docker.sock:/var/run/docker.sock
      ports:
        - 9443:9443

```
run 

> docker compose up -d

visit https://docker-host-ip:9443 for portainer web gui

### 4.3 Setup homarr container
docker compose file -

```
version: '3'
---
services:
  homarr:
    container_name: homarr
    image: ghcr.io/ajnart/homarr:latest
    environment:
      PUID: 1001
      PGUID: 1001
    restart: unless-stopped
    volumes:
#      - /var/run/docker.sock:/var/run/docker.sock # Optional, only if you want docker integration
      - ../../storage/homarr-data/configs:/app/data/configs
      - ../../storage/homarr-data/icons:/app/public/icons
      - ../../storage/homarr-data/data:/data
    ports:
      - '8082:7575'

```
Then run 

> docker compose up -d

navigate to http://docker-host-ip:8082 for web ui

Next up, lets install network wide security services first -

### 4.4 Setup pihole container

I need pi hole for 2 reasons -
    
- Network wide ad blocking
- a locally hosted dns server for my internal websites/apps

Therefore, pi hole container must be connected directly to home network and should have its own ip address if I plan to use it as dhcp server as well. This would be good idea if i was runnig on ethernet but unfortunately docker macvlan networks do not play well with wireless interfaces. I tried to investigate it and do not think it is worth it. 

finally I have deployed it as regular docker container but i had to do the following to get it working

- changed the pihole admin portal port from 80 to 8080 because ngnix proxy manager is already using port 80, so to avoid port conflict errors pi hole web ui need to run on 8080.
- port 53 is used by pihole but it is conflicting with systemd resolved service on ubuntu, So to make pihole work i had to stop and disable the systemd resolved service and edit the resolve.conf manually.
- I think the net side effect would be that if pohole goes down local name resolution might now work on this server anymore.
 
  

```
version: "3"

services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "67:67/udp"
      - "8080:80/tcp"
    environment:
      TZ: 'America/Chicago'
      WEBPASSWORD: 'Changeme'
      PUID: 1001
      PGUID: 1001
      VIRTUAL_HOST: 'pi.hole'
    volumes:
      - '../../storage/pihole-data/etc-pihole:/etc/pihole'
      - '../../storage/pihole-data/etc-dnsmasq.d:/etc/dnsmasq.d'
    dns:
      - 127.0.0.1
      - 1.1.1.1
      - 8.8.8.8
    cap_add:
      - NET_ADMIN
    restart: unless-stopped


```
before starting the container -

stop resolved

> sudo systemctl stop systemd-resolve

disable resolved

> sudo systemctl disable systemd-resolve

edit the resolve.conf

> sudo nano /etc/resolv.conf

find the following line in it 

> nameserver 127.0.0.53

replce it with -

> nameserver 1.1.1.1

reboot the system and then start the pi hole container.

navigate to http://docker-host-ip:8080 for pihole web ui.


### setup glances container
docker compose file -

```
version: '3'

services:
  glances:
    image: nicolargo/glances:latest
    container_name: 'glances'
    restart: always
    environment:
      - "GLANCES_OPT= -w"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      # comment the below line if you dont want glances to display host OS detail instead of container's
      - /etc/os-release:/etc/os-release:ro
    #pid: host
    ports:
      - 8081:61208

```

then run 

> docker compose up -d

navigate to http://docker-host-ip:8081 for glances web ui

### 4.5 Setup nginx proxy manager container

```
version: "3"
---
services:
 app:
  image: 'jc21/nginx-proxy-manager:latest'
  container_name: 'nginx-proxy-manager'
  environment:
    PUID: 1001
    PGID: 1001
  restart: unless-stopped
  ports:
    - '80:80'
    - '81:81'
    - '443:443'
  volumes:
    - ../../storage/nginx-proxy-manager-data/data:/data
    - ../../storage/nginx-proxy-manager-data/letsencrypt:/etc/letsencrypt

```

run

> docker compose up -d

visit http://docker-host-ip:81 for a nginx proxy manager admin portal.



---

### 4.6 setup filebrowser container

filebrowser provides a light weight web ui which i prefer to use for sharing files and folders to home users.

docker compose 

```
version: '3'
---
services:
  filebrowser:
    image: filebrowser/filebrowser
    container_name: filebrowser
    environment:
      PUID: 1001
      PGUI: 1001
    ports:
      - 8085:80
    volumes:
      - '../../storage/filebrowser-data/shared-storage:/srv'
      - '../../storage/filebrowser-data/filebrowser.db:/db:database.db'
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true



```
Now create a file called filebrowser.db in the filebrowser-data folder, before starting the container. Otherwise it will fail.

> touch filebrowser.db

then start the container -

> docker compose up -d

nagivate to http://docker-host-ip:8085 for the web ui.

### 4.7 setup nextcloud docker container

docker compose file

```
version: '3'
---
services:
  nc:
    image: nextcloud:apache
    container_name: nextcloud
    restart: always
    ports:
      - 8086:80
    volumes:
      - '../../storage/nextcloud-data/nc-web-data:/var/www/html'
    networks:
      - redisnet
      - dbnet
    environment:
      - REDIS_HOST=redis
      - MYSQL_HOST=db
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=nextcloud
  redis:
    image: redis:alpine
    container_name: redis-nc
    restart: always
    networks:
      - redisnet
    expose:
      - 6379
  db:
    image: mariadb:10.5
    container_name: nc-mariadb
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    restart: always
    volumes:
      - '../../storage/nextcloud-data/nc-db-data:/var/lib/mysql'
    networks:
      - dbnet
    environment:
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_ROOT_PASSWORD=nextcloud
      - MYSQL_PASSWORD=nextcloud
    expose:
      - 3306
volumes:
  db_data:
  nc_data:
networks:
  dbnet:
  redisnet:

```

start the container with

> docker compose up -d

navigate to http://docker-host-ip:8086 to configure nextcloud using web gui


# Testing & Troubleshooting


---


# Automating the server setup
I would use ansible to automate the server setup as much as possible so that i can re-build it easily.

---

# REFERENCES ---

[Installing ubuntu server on raspberry pi 4](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#1-overview)

[docker official documentation](https://docs.docker.com/engine/install/ubuntu/)

[how to install docker on ubuntu 2204 articles](https://www.linuxtechi.com/install-use-docker-on-ubuntu/)

[setting up pivpn](https://pimylifeup.com/raspberry-pi-vpn-server/)

[Setting up portainer in docker container](https://bobcares.com/blog/install-portainer-docker-compose/)

[Setting up nginx proxy manager in docker container](https://nginxproxymanager.com/setup/)

[nginx proxy tutorial](https://www.bogotobogo.com/DevOps/Docker/Docker-Compose-Nginx-Reverse-Proxy-Multiple-Containers.php)

[setting up pihole in docker container on ubuntu server](https://pimylifeup.com/pi-hole-docker/)

[setting up homarr in docker container](https://homarr.dev/docs/getting-started/installation/)
[setting up glances in docker container](https://github.com/nicolargo/glances/blob/develop/docs/docker.rst)

[setting up zabbix in docker container](https://medium.com/@oachuy/easy-way-of-monitoring-your-raspberry-pi-nas-and-containers-with-zabbix-bbc123a7975f)

[setting up nextcloud in container](https://medium.com/@oachuy/install-your-own-cloud-storage-with-nexcloud-in-raspberry-pi-4-fd29f9039283)

