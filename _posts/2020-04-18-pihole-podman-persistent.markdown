---
layout: post
title:  "Adding Persistent Storage to Pi-hole in Podman"
date:   2020-04-18 17:00:00 -0400
categories: jekyll update
---
#### Introduction
In a previous post I set up pihole to run in a container on Fedora 31 server. I realized early that it does not save configuration. All of my blacklists and whitelists would be wiped each time the container was re-created. The solution to this is to add "volumes" for the container to use.

There is also an easier method to create the service file used by Systemd that I will explain.

#### Setup

Adding persistent storage area for my pihole container required adding the following arguments to my create command:
```
-v $(pwd)/etc-pihole:/etc/pihole/:z -v $(pwd)/etc-dnsmasq.d:/etc/dnsmasq.d/:z
```

My full command is now as follows:
```
$ podman run -d -p 53:53/tcp -p 53:53/udp -p 67:67/udp \
-p 80:80/tcp -p 443:443/tcp -e DNS1=1.1.1.1 -e DNS2=8.8.8.8 \
-e TZ=America/New_York --name pi-hole -e WEBPASSWORD=<PASSWORD> \
-v $(pwd)/etc-pihole:/etc/pihole/:z -v $(pwd)/etc-dnsmasq.d:/etc/dnsmasq.d/:z \
docker.io/pihole/pihole
```
These lines will place the storage location in the user's home directory that the container runs as. In my case this means the persistent storage is created in:
```
/home/homeuser/etc-pihole
/home/homesuer/etc-dnsmasq.d
```
Now that our command to create is good to go I can generate the service file for systemd with the following command:
````
$ podman generate systemd > /etc/systemd/system/pihole-container.service
```

This will create a service file for systemd to use by redirecting the output of the generate command to a new service file in /etc/systemd/system/

Next we need to reload systemctl and enable the service to start at boot:

```
$ systemctl enable pihole-container.service
$ systemctl start pihole-container.service
```
#### Conclusion
I now have persistent storage for Pihole. My blacklists, whitelists, and DNS settings should persist if our container is recreated.