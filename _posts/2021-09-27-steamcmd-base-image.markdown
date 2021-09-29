---
layout: post
title:  "Building a SteamCMD Container Base Image From Scratch"
date:   2021-09-27 09:30:00 -0400
---
Many developers on Steam provide dedicated server software that can be used by players to host their own multiplayer games.
The [```steamcmd```](https://developer.valvesoftware.com/wiki/SteamCMD) tool can be used to fetch the server software from Steam.
This command-line tool allows users to connect to Steam, download, and install dedicated server software.
For this post I will be describing my process of building a container base image with the SteamCMD tool preinstalled.

## Requirements
One of the main concerns of running any Internet connected server is security.
To minimize the security footprint, and learn along the way, I will be building this image from scratch with a very minimal set of software.
The image should be designed for rootless opearation with as few privileges as possible.
To further isolate the server from the host, containers derived from this image will use an unprivileged user inside of the container itself.

For this project, I will be using [```podman```](https://podman.io/) and [```buildah```](https://buildah.io/).

## Creating the SteamCMD base image
In order to download dedicated servers from Steam, I need the [```steamcmd```](https://developer.valvesoftware.com/wiki/SteamCMD) command.
Using the Valve-provided instructions on ```steamcmd```, I wrote a build script adapting the instructions to fit my own requirements.

The build script is as follows:
```bash
#!/bin/bash
set -o errexit

# Vars
CONTAINER=$(buildah from --ulimit nofile=2048 scratch)
MOUNTPOINT=$(buildah mount $CONTAINER)

URL='https://steamcdn-a.akamaihd.net/client/installer/steamcmd_linux.tar.gz'
STEAM=/home/steam/steamcmd
VER=$(egrep '^VERSION_ID' /etc/os-release | cut -d '=' -f2)

# Install dependencies
if command -v 'dnf' &> /dev/null; then
  dnf install -y --installroot $MOUNTPOINT --releasever $VER\
    SDL2.i686 coreutils glibc-langpack-en glibc.i686 libstdc++.i686\
    --nodocs --refresh --setopt install_weak_deps=False
  dnf clean all -y --installroot $MOUNTPOINT --releasever $VER
else
  echo "Build script requires dnf package manager. Exiting."; exit
fi

# Create steam user
mkdir $MOUNTPOINT/home/steam
echo 'steam:x:1000:1000::/home/steam:/bin/bash' >> $MOUNTPOINT/etc/passwd
echo 'steam:x:1000:' >> $MOUNTPOINT/etc/group
echo "PATH=\$PATH:$STEAM" >> $MOUNTPOINT/home/steam/.bashrc
buildah config --user steam:steam $CONTAINER
buildah config --workingdir '/home/steam' $CONTAINER

# Get steamcmd, unpack, and update
mkdir $MOUNTPOINT$STEAM
wget -qO- $URL | tar xvzf - -C $MOUNTPOINT$STEAM
chmod -R 700 $MOUNTPOINT/home/steam
chown -R 1000:1000 $MOUNTPOINT/home/steam
buildah run $CONTAINER -- sh\
  -c "$STEAM/steamcmd.sh +login anonymous validate +exit"
```

The script mounts an empty image and then uses ```dnf``` package manager to install packages into the root of the mounted container.
This way I do not need to include a package manager inside of the container, my host's package manager installs the packages into the container.
I then create the ```steam``` user and make the required changes to ```/etc/passwd``` and ```/etc/group```.
Next I download and unpack the ```steamcmd``` files into the container and run it so that it will download and update the latest Steam files.
Lastly the script squashes and commits the image.

Building and listing the image:
```
[ethan@fedora steamcmd]$ buildah unshare bash build.sh 
[ethan@fedora steamcmd]$ podman image list
REPOSITORY                  TAG         IMAGE ID      CREATED        SIZE
docker.io/emsoucy/steamcmd  latest      7a82922e0390  3 minutes ago  346 MB
```
Installing a game:
```
[ethan@fedora ~]$ podman run -it steamcmd bash
bash-5.1$ steamcmd.sh +login anonymous +app_update <app id> validate +exit
```

# Resources
The most recent build script can be found on [GitHub](https://github.com/emsoucy/steamcmd).

[Fedora Magazine: Build Smaller Containers](https://fedoramagazine.org/build-smaller-containers/)
