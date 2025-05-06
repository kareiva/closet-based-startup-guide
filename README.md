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

    sudo dnf install podman
    sudo useradd -d /alot/ofspace -m apps
    sudo usermod -v 1000000-1065536 -w 1000000-1065536 apps
    sudo podman system migrate
    sudo bash -c "echo 'net.ipv4.ip_unprivileged_port_start=53' > /etc/sysctl.d/user_priv_ports.conf"
    sudo firewall-cmd --add-port=8080/tcp --permanent # for dashboards and stuff
    sudo firewall-cmd --reload

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

## Launching cloud apps in a closet

## Managing application stacks

## Closet housekeeping

## Credits

* Simonas Kareiva
