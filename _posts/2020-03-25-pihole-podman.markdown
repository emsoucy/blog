---
layout: post
title:  "Using Pi-hole with Podman in Fedora 31"
date:   2020-03-25 09:09:19 -0400
categories: jekyll update
---
### Introduction

Today I wanted to free-up my Raspberry Pi 4 for other projects and it was wasteful to use it as a Pihole as it is overpowered for the task. I thought that since I already have a home server that I shall run Pihole containerized on the server that I already have.

Fedora 31 uses CGroups V2 which is not compatible with Docker. In "Fedoraland" Podman is used instead. Podman is designed to be a near drop-in replacement for Docker, so much so that they advocate:

```console
alias docker=podman
```

### Adapting Pihole docker-compose for Podman use

Pihole provides a docker-compose example at: [https://hub.docker.com/r/pihole/pihole/](https://hub.docker.com/r/pihole/pihole/)

```yaml
version: "3"

# More info at https://github.com/pi-hole/docker-pi-hole/ and https://docs.pi-hole.net/
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "67:67/udp"
      - "80:80/tcp"
      - "443:443/tcp"
    environment:
      TZ: 'America/Chicago'
      # WEBPASSWORD: 'set a secure password here or it will be random'
    # Volumes store your data between container upgrades
    volumes:
       - './etc-pihole/:/etc/pihole/'
       - './etc-dnsmasq.d/:/etc/dnsmasq.d/'
    dns:
      - 127.0.0.1
      - 1.1.1.1
    # Recommended but not required (DHCP needs NET_ADMIN)
    #   https://github.com/pi-hole/docker-pi-hole#note-on-capabilities
    cap_add:
      - NET_ADMIN
    restart: unless-stopped
```

Unfortunately there is no drop-in replacement for docker-compose as of right now
but we can gather the information we need for Podman from this example. We need
the image, ports, WEBPASSWORD, and dns fields for Podman.

Using this example we can build the following command to run Pi-hole in Podman:

```console
podman run -d -p 53:53/tcp -p 53:53/udp -p 67:67/udp \
  -p 80:80/tcp -p 443:443/tcp -e DNS1=1.1.1.1 -e DNS2=8.8.8.8 \
  -e TZ=America/New_York --name pi-hole -e WEBPASSWORD=<password> \
  docker.io/pihole/pihole
```

As we can see- I needed the docker.io/pihole/pihole address, the ports, my
timezone, and then my own unique WEBPASSWORD. I have configured my DNS to use
Cloudflare (1.1.1.1) and Google (8.8.8.8).

### Using Systemd

To configure this container to begin running at boot time I used the example
found
[here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/managing_containers/running_containers_as_systemd_services_with_podman)

If you've used Systemd this should look very familiar. I created the following
file:

/etc/systemd/system/pihole-container.service

```console
[Unit]
Description=Pihole container

[Service]
Restart=always
ExecStart=/usr/bin/podman start -a pi-hole
ExecStop=/usr/bin/podman stop -t 2 pi-hole

[Install]
WantedBy=local.target
```

Be sure that the name of the container at the end of the ExecStart and ExecStop
commands is the same that was used prior for the podman run command.

Once the service file is created you can enable it to run at boot with systemctl

```console
systemctl daemon-reload
systemctl enable pihole-container.service
```

#### Conclusion

This was the process that I used to run Pi-hole in a container on Fedora 31
using Podman. I was able to eliminate my Raspberry Pi from my home network, save
on energy, and pass the savings onto you!
