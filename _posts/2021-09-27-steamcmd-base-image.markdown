---
layout: post
title:  "Building a SteamCMD Container Base Image From Scratch"
date:   2021-09-27 09:30:00 -0400
---
Many developers on Steam provide dedicated server software that can be used to host multiplayer instances of their games.
This enables players to host and control their own game servers.
One requirement for this is the [```steamcmd```](https://developer.valvesoftware.com/wiki/SteamCMD) tool.
This command line tool allows the user to connect to Steam, download, and install dedicated server software.
For this post I will be describing my process of building a SteamCMD container base image which can be used as a starting point in the future for game servers.

## Requirements
One of the main concerns of running any Internet connected server is security.
To minimize the security footprint (and learn along the way) I will be building this image from scratch with a very minimal set of software.
The image should also be designed to be run rootless using as few privileges as possible.
To further reduce the security footprint, containers derived from this image should force the use of a non-root user inside the container itself.

For this project, I will be using [```podman```](https://podman.io/) and [```buildah```](https://buildah.io/).

## Creating the SteamCMD base image
In order to download dedicated servers from Steam I need the [```steamcmd```](https://developer.valvesoftware.com/wiki/SteamCMD) command.
Using the Valve-provided instructions on the ```steamcmd```, I wrote a build script adapting the instructions to fit my own requirements.

The current incarnation of the build script is repeated here:
```bash
#!/bin/bash
set -o errexit

# Vars
CONTAINER=$(buildah from scratch)
MOUNTPOINT=$(buildah mount $CONTAINER)

URL='https://steamcdn-a.akamaihd.net/client/installer/steamcmd_linux.tar.gz'
STEAM=/home/steam/steamcmd
OS=$(egrep '^NAME' /etc/os-release | cut -d '=' -f2)
VER=$(egrep '^VERSION_ID' /etc/os-release | cut -d '=' -f2)

# Install dependencies
if [[ $OS =~ 'Fedora' ]]; then
  dnf install -y --installroot $MOUNTPOINT --releasever $VER coreutils\
    glibc.i686 libstdc++.i686 --nodocs --setopt install_weak_deps=False
  dnf clean all -y --installroot $MOUNTPOINT --releasever $VER
else
  echo "Unsupported OS. Exiting"; exit
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
buildah unmount $CONTAINER

buildah commit --squash $CONTAINER docker.io/emsoucy/steamcmd
```
The script mounts an empty image and then uses ```dnf``` package manager to install packages into the root of the mounted container.
This way I do not need to include a package manager inside of the container, my host's package manager installs the packages into the container.
I then create the ```steam``` user and make the required changes to ```/etc/passwd``` and ```/etc/group```.
Next , I download and unpack the ```steamcmd``` files into the container and run it so that it will download and update the latest Steam files.
Lastly, the script squashes and commits the image.

Building and listing the image:
```
[ethan@fedora steamcmd]$ buildah unshare bash build.sh 
[ethan@fedora steamcmd]$ podman image list
REPOSITORY                  TAG         IMAGE ID      CREATED        SIZE
docker.io/emsoucy/steamcmd  latest      7a82922e0390  3 minutes ago  346 MB
```

# Conclusion
I now have a finished, generic base image that contains ```steamcmd``` that I can use for future dedicated servers.
