---
layout: post
title:  "Building a Valheim Dedicated Server Container"
date:   2021-10-04 09:30:00 -0400
---
I enjoy gaming with friends and the latest game that we have been playing is
[Valheim](https://en.wikipedia.org/wiki/Valheim). Valheim is a sandbox/survival
game that takes place in a large procedurally generated world inspired by Norse
mythology. This game is particularly interesting to me because the developers
provide a dedicated server that players can run locally. In my previous post I
built a container base image that was to serve as a starting point for future
dedicated server containers. I will be using this base image and adding the
Valheim dedicated server software so that I can host a Valheim server for
friends.

## Requirements

Most of the work has already been done but there is still some game-specific
work to do.
I would like the container to:

- Run rootless
- Download and update the game on startup
- Game worlds shall persist on the host
- Server admin files, such as adminlist.txt, shall persist on the host

## Building the Container Image

This time around the build script is quite small, however we gain an entry point
script. I'll repeat them here, since they are short, but they can be found on
[GitHub](https://github.com/emsoucy/valheim).

Build script:

```bash
#!/bin/bash
set -o errexit

CONTAINER=$(buildah from steamcmd)

buildah run $CONTAINER -- sh -c \
  'mkdir -p /home/steam/.config/unity3d/IronGate/Valheim/worlds'
buildah copy $CONTAINER $(dirname $(realpath "$0"))/startServer.sh '/home/steam'
buildah config --cmd '/home/steam/startServer.sh' $CONTAINER

buildah commit --squash $CONTAINER valheim 
```

This creates the full path to worlds directory which we will need for the volume
mount and copies over the entry point startServer.sh script. The start server
script will server as the container entrypoint and will download, update, and
run the game server software.

Container entry point script:

```bash
#!/bin/bash
set -o errexit

# Check for required variables
if [ ! "$SERVER_NAME" ] || [ ! "$WORLD_NAME" ] || [ ! "$SERVER_PASS" ]; then
  echo 'Requires SERVER_NAME, WORLD_NAME, and SERVER_PASS environment variables. Exiting'
  exit
fi

# Check for optional variable 
if [ -z "$SERVER_PORT" ]; then
  echo 'SERVER_PORT not defined. Using default port 2456'
  export SERVER_PORT=2456
fi

export LD_LIBRARY_PATH=$HOME/Steam/steamapps/common/Valheim\ dedicated\ server/linux64:$LD_LIBRARY_PATH
export SteamAppId=892970
export ValheimAppId=896660

# Update Steam, download/update Valheim
$HOME/steamcmd/steamcmd.sh +login anonymous +app_update $ValheimAppId validate \
  +exit

# Run Valheim
$HOME/Steam/steamapps/common/Valheim\ dedicated\ server/valheim_server.x86_64 \
  -name "$SERVER_NAME" -port "$SERVER_PORT" -world "$WORLD_NAME" \
  -password "$SERVER_PASS" -nographics -batchmode -public 1
```

The image can now be built:

```console
$ buildah unshare bash build.sh
88c245db4118411a0f4ef190740ded42dda6cef49fd137cca05ad42806acb9f6
Getting image source signatures
Copying blob 845d603ce1eb done
Copying config a87c9cc6f7 done
Writing manifest to image destination
Storing signatures
a87c9cc6f734d48f77e07674f6286ba1ba77fe459ab7eb7ae10140e8e2f9303c
```

This builds quickly because I already have the ```steamcmd``` base image built.
If the build is successful I should see the new Valheim container listed.

```console
$ podman image list
REPOSITORY          TAG         IMAGE ID      CREATED            SIZE
localhost/valheim   latest      a87c9cc6f734  About an hour ago  356 MB
localhost/steamcmd  latest      42404647a957  2 hours ago        356 MB
```

## Running the Server

The new container can be run with the following:

```console
podman run -d \
  -e SERVER_NAME="Ethan's Server" \
  -e WORLD_NAME="Ethan's World" \
  -e SERVER_PASS="secret password" \
  -v ./data:/home/steam/.config/unity3d/IronGate/Valheim:Z,U \
  -p 2456-2457:2456-2457/udp \
  --name valheim \
  valheim
```

The ```data``` directory can be anything that I choose. Valheim will use the
data directory for things like worlds and admin files. The directory tree of
```data``` can be seen as follows:

```console
data/
├── adminlist.txt
├── bannedlist.txt
├── permittedlist.txt
├── prefs
└── worlds
    ├── Ethan's World.db
    ├── Ethan's World.db.old
    ├── Ethan's World.fwl
    └── Ethan's World.fwl.old
```

Once the container is running I am able to connect and play.
![Valheim New World](/photos/valheim/valheim.png)
