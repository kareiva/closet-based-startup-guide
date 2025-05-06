# Closet-based startup guide

Everyone wants to be cloud-native these days. Let's talk about being able to run a startup in a closet. 
If you don't know what's exactly a closet, you might be better off by looking it up on Google Images
(don't forget to add something like "server closet" or "network closet" otherwise the results might vary).

TODO: _put a picture of a closet here_

## Contents

* [Installing a server in a closet](#Installing-a-server-in-a-closet)
* [Picking an engine for containerized apps](#Picking-an-engine-for-containerized-apps)
* [Using a closet-native application proxy](#Using-a-closet-native-application-proxy)
* [Installing a closet container registry (CCR)](#Installing-a-closet-container-registry-CCR)
* [Launching cloud apps in a closet](#Launching-cloud-apps-in-a-closet)
* [Managing application stacks](#Managing-application-stacks)
* [Closet housekeeping](#Closet-housekeeping)

## Things not covered

* High availability
* Disaster recovery
* What to do after an startup investment
* Closet-to-cloud migrations (should be seamless)

## Installing a server in a closet

If you're doing this in a closet, you are probably not rich enough to afford an Enterprise Linux licence. But we're
going to aim for the stars, therefore picking something close to enterprise (and not so costly) would be a good idea.
My advice is to go by the upstream and use Fedora Server or CentOS Stream (both are FOSS). I will go with CentOS Stream
version 10 because it's super close to RHEL except that it receives updates in YOLO (rolling) mode. So, YOLO.

* CentOS Stream Downloads: https://www.centos.org/download/ ([direct v10 download](https://mirrors.centos.org/mirrorlist?path=/10-stream/BaseOS/x86_64/iso/CentOS-Stream-10-latest-x86_64-dvd1.iso&redirect=1&protocol=https))
* Python module for kickstart configs: https://github.com/pykickstart/pykickstart

Remember our server is a pet and not a cattle, so we should really *care* about it. If you want to make few cattles, 
the first CentOS server will have a /root/anakdonda-ks.cfg file that can be reused for subsequent automated installs.

You might consider RAID levels during the installation, but this is not necessary, because of the nature of the closet-based
installation. You will probably need backups only after you're successful and this is going to happen sooner than somebody
nukes your closet with a intercontinental ballistic missile. With backups, you will also need restores.

## Picking an engine for containerized apps

Now we can probably run Docker on Linux, but Docker is already dead at the time of this writing. Also, Docker will happily 
send you a thick bill once your annual revenue surpasses $10,000,000.00. This can happen anytime now, so let's go with a 
free alternative - Podman. Podman also has a lesser chance of exposing your pet server underwear in case your first
startup hire makes a typo when launching a containerized app. No swarm, no kubernetes, no cloud, just a podman in a closet.

    sudo dnf install podman python3-pip
    sudo useradd -d /alot/ofspace -m apps
    sudo usermod -v 1000000-1065536 -w 1000000-1065536 apps
    sudo podman system migrate
    sudo bash -c "echo 'net.ipv4.ip_unprivileged_port_start=53' > /etc/sysctl.d/user_priv_ports.conf"
    sudo firewall-cmd --add-port=8080/tcp --permanent # for dashboards and stuff
    sudo firewall-cmd --reload
    sudo pip3 install podman-compose

Create a two-factor SSH authentication mechanism for the user 'apps':

    sudo dnf install -y epel-release && sudo dnf install -y google-authenticator qrencode-libs
    
You can follow the full guide [here](https://www.digitalocean.com/community/tutorials/how-to-set-up-multi-factor-authentication-for-ssh-on-centos-7). There are a plenty of bugs to solve too ([here](https://github.com/google/google-authenticator-libpam/issues/101#issuecomment-997533681)).

Okay, just use [duo.com](https://duo.com).

## Using a closet-native application proxy

Let's pick something that is both bullet-proof and hard to pronounce for the script kiddies. Traefik (which is both a cloud and
closet-native proxy) will do just fine and it works nicely with Podman. Once your downtime will cost you around $10.000,00/minute,
Traefik will be there to support you at a presumably lesser rate. As your unprivileged user:

    systemctl --user enable podman.socket
    touch ~/acme.json
    chmod 0600 ~/acme.json
    podman run -d \
      --name=traefik \
      --net podman \
      --security-opt label=type:container_runtime_t \
      -v /run/user/1000/podman/podman.sock:/var/run/docker.sock:z \
      -v /home/apps/acme.json:/acme.json:z \
      -p 80:80 \
      -p 443:443 \
      -p 8080:8080 \
      docker.io/library/traefik:latest \
      --api.dashboard=true \
      --api.insecure=true \
      --certificatesresolvers.lets-encrypt.acme.email="your@startup.com" \
      --certificatesresolvers.lets-encrypt.acme.storage=/acme.json \
      --certificatesresolvers.lets-encrypt.acme.tlschallenge=true \
      --entrypoints.http.address=":80" \
      --entrypoints.http.http.redirections.entryPoint.to=https \
      --entrypoints.http.http.redirections.entryPoint.scheme=https \
      --entrypoints.https.address=":443" \
      --providers.docker=true

Don't forget to plug this beast into systemd:

    mkdir -p ~/.config/systemd/user/ && cd ~/.config/systemd/user/
    podman generate systemd --files --new --name traefik
    systemctl --user daemon-reload 
    systemctl --user enable --now traefik
    loginctl enable-linger
    

This [web page](https://blog.cthudson.com/2023-11-02-running-traefik-with-podman/) has brilliant guidance on how to run Podman with Traefik.

## Installing a closet container registry (CCR)

You have now a functioning closet-native platform and it is time to serve your first app that does something more than just "Hello World".
Since we're betting on containers, we need to create a place where to store them. Of course we don't want anything so advanced like Quay, 
so we will just run a simple registry wrapped in TLS behind Traefik. Did we already install podman-compose? I think so.

    mkdir -p ~/registry/auth
    echo 'apps:$2y$10$3RxCvl4s1kSQBssGLpKgDOYz4JHWiabd4vQBgwjZQPzLYOw.IBPkq' > ~/registry/auth/registry.passwd
    cd ~/registry && cat << EOF > compose.yaml
    version: '3.8'
    services:
      registry:
        image: registry:2
        container_name: registry
        restart: always
        environment:
          REGISTRY_AUTH: htpasswd
          REGISTRY_AUTH_HTPASSWD_REALM: Registry-Realm
          REGISTRY_AUTH_HTPASSWD_PATH: /auth/registry.passwd
          REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
        volumes:
          - registrydata:/data
          - ./auth:/auth
        labels:
          - "traefik.enable=true"
          - "traefik.docker.network=registry"
          - "traefik.http.routers.registry.entrypoints=http"
          - "traefik.http.routers.registry.rule=Host(`registry.yourstartup.com`)"
          - "traefik.http.middlewares.registry-https-redirect.redirectscheme.scheme=https"
          - "traefik.http.routers.registry.middlewares=registry-https-redirect"
          - "traefik.http.services.registry-secure.loadbalancer.server.port=5000"
          - "traefik.http.routers.registry-secure.entrypoints=https"
          - "traefik.http.routers.registry-secure.tls.certresolver=lets-encrypt"
          - "traefik.http.routers.registry-secure.tls=true"
          - "traefik.http.routers.registry-secure.rule=Host(`registry.yourstartup.com`)"
        networks:
          - registry
    
      portainer:
        image: portainer/portainer-ce
        container_name: portainer
        restart: always
        volumes:
          - /run/user/1000/podman/podman.sock:/var/run/docker.sock:z
          - portainer_data:/data:z
        networks:
          - registry
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.portainer.entrypoints=http"
          - "traefik.http.routers.portainer.rule=Host(`portainer.yourstartup.com`)"
          - "traefik.http.middlewares.portainer-https-redirect.redirectscheme.scheme=https"
          - "traefik.http.routers.portainer.middlewares=portainer-https-redirect"
          - "traefik.http.services.portainer-secure.loadbalancer.server.port=9000"
          - "traefik.http.routers.portainer-secure.entrypoints=https"
          - "traefik.http.routers.portainer-secure.tls.certresolver=lets-encrypt"
          - "traefik.http.routers.portainer-secure.tls=true"
          - "traefik.http.routers.portainer-secure.rule=Host(`portainer.yourstartup.com`)"
    
    networks:
      registry:
        driver: bridge
    
    volumes:
      portainer_data:
        driver: local
      registrydata:
        driver: local
    EOF
    podman compose up -d
    
Now try to login to your registry and push something:

    $ podman login registry.yourstartup.com
    Username: apps
    Password: 
    Error: authenticating creds for "registry.example.com": pinging container registry registry.example.com: Get "https://registry.example.com/v2/": dial tcp: lookup registry.example.com: i/o timeout

LOL, we need to add DNS records and your closet server must have a routable external IPv4 (or IPv6) address where all the above DNS records are pointed. 
Fix that and try again. See you in few weeks!

## Launching cloud apps in a closet

The classical way of running apps inside a container works just fine. Here's an example command to start a webapp that runs inside a container on port 9000:

    podman run --detach \
             --name myapp \
             --network podman \
             --volume data:/var/opt/something \
             --restart always \
             -l traefik.enable="true" \
             -l traefik.http.routers.myapp.rule=Host'(`myapp.yourstartup.com`)' \
             -l traefik.http.middlewares.myapp-https-redirect.redirectscheme.scheme="https" \
             -l traefik.http.routers.myapp.middlewares="myapp-https-redirect" \
             -l traefik.http.routers.myapp-secure.entrypoints="https" \
             -l traefik.http.routers.myapp-secure.rule=Host'(`myapp.yourstartup.com`)' \
             -l traefik.http.routers.myapp-secure.tls="true" \
             -l traefik.http.routers.myapp-secure.tls.certresolver=lets-encrypt \
             -l traefik.http.services.myapp.loadbalancer.server.port="9000" \
             registry.yourstartup.com/myapp:latest

Nothing happens. Let's check the proxy logs:

    $ podman logs traefik
    2025-04-26T19:43:25Z ERR error="service \"myapp\" error: unable to find the IP address for the container \"/myapp\": the server is ignored" container=myapp-4fe88f4a4350cab8bd5a69b182696e75d760c18f4740873269293d5010c9be81 providerName=docker

Oh shit we forgot to enable DNS on the default podman network! 

## Managing application stacks

Now based on the above `compose.yaml` template you are ready to launch your apps in stacks that you will be able to see in Portainer.

## Closet housekeeping

TODO: Closet housekeeping

## Credits

* Simonas Kareiva
