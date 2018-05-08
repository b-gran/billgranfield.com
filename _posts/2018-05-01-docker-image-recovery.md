---
layout: post
title: Recovering data from a broken MacOS Docker installation
description: |
  How to recover and retrieve data (images and volumes) from broken or corrupt MacOS Docker installation.
---

# Recovering data from a broken MacOS Docker installation

This guide explains how to recover data from [Docker](https://docs.docker.com/docker-for-mac/), even if the underlying installation is corrupt.

The guide is written for MacOS specifically, although many of the steps are applicable to *nix. 

## How is data stored in _Docker for Mac_?
_Docker for Mac_ starts a [HyperKit](https://github.com/moby/hyperkit) daemon which runs a [LinuxKit](https://github.com/linuxkit/linuxkit) VM. LinuxKit is an extremely light-weight Linux distro. The HyperKit VM runs [containerd](https://github.com/containerd/containerd), which is actually responsible for creating Docker containers.

The HyperKit VM's data is stored on disk in [qcow2](https://people.gnome.org/~markmc/qcow-image-format.html) format. Depending on how you installed _Docker for Mac_, this disk image is located at `$HOME/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/Docker.qcow2`.

The HyperKit VM image contains all of your container data, including images and volumes. Recovering data from _Docker for Mac_ is a matter of extracting data from the `qcow2` image.

## Exploring the HyperKit VM filesystem
_Docker for Mac_ creates a `tty` in the VM directory. Usually this file is located at `$HOME/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty`.
This file is the `tty` for the HyperKit VM. To get a shell in the HyperKit VM, run
```bash
screen $HOME/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty
```

Once inside the HyperKit VM, you can explore the data stored.
The volumes are stored in `/var/lib/docker/volumes`, and the volume contents are present as ordinary files and directories.

The location and format of the images depends on the storage driver - see [this link](https://stackoverflow.com/questions/19234831/where-are-docker-images-stored-on-the-host-machine) for more information.

## Data recovery if the Docker daemon still starts
If the `qcow2` image is uncorrupted and the HyperKit daemon still starts, recovering data is somewhat straightforward. It will still be possible to issue Docker commands and back up the data using Docker's own tooling.

The basic idea is:
1. Identify a volume to back up
2. Start a _new_ container that mounts the volume
3. Copy the volume to the host filesystem using `docker cp`

### 1 Identify the volume you want to backup
We need to find the **volume name** of the volume we want to back up.

Run this command to list all of the available volumes:
```bash
docker volume ls
```

The output looks like this:
```
DRIVER              VOLUME NAME
local               f52cf7a10fbd02d3e932f37074f92e3949f5a186581023b233691cde09e87352
local               f99f37c656d20b253785c1615a063ee8c5ca6ee7c39d53f897bb184ec4dfc745
local               some-volume-name
```

You can get _a bit_ more information about the volume by running
```bash
docker volume inspect some-volume-name
```

The output looks like this:
```
[
  {
    "CreatedAt": "2018-04-29T20:41:25Z",
    "Driver": "local",
    "Labels": {
      "com.docker.compose.project": "some-project-name",
      "com.docker.compose.volume": "volume-name"
    },
    "Mountpoint": "/var/lib/docker/volumes/some-project-name_volume-name/_data",
    "Name": "some-project-name_volume-name",
    "Options": null,
    "Scope": "local"
  }
]
```
> the labels field may not be present, depending on how you created the volume.

The `CreatedAt` and `Mountpoint` fields may be helpful.
`Mountpoint` is the mount point of the volume _within the HyperKit VM_. You can explore the data using the tty above. 

Once you've identified the volume you want to back up, **take note of the volume name**. You'll be using this volume name in later commands.

### 2 Start a new container that mounts the volume
In order to use `docker cp`, a simple way to copy data from a container to the host, we first need to start a container that contains the volume data.
We don't really care what this container does, as long as it mounts the correct volume and stays running while we copy the data to the host.

Run the following command, substituting `$your_volume_name` with the volume from the previous step:
```bash
docker run `# docker run starts a new container with the specified image and command` \
  -d `# Run the container in the background` \
  --rm `# Remove the container when it exits` \
  --mount source=$your_volume_name,target=/volume `# Mount your chosen volume at the` \
                                                  `# /volume path on the containers filesystem` \
  busybox `# Use the busybox image, an extremely minimal Linux` \
  tail -f /dev/null `# A command that runs indefinitely without exiting`
```

For convenience, here's the command without comments or newlines:
```bash
docker run -d --rm --mount source=$your_volume_name,target=/volume busybox tail -f /dev/null
```

This command will print the id of the container, which we will use in the next step.
You can also run the following command to find the human-readable name of the container you just created:
```bash
docker ps
```

Take note of **the container id or the container name** for use in the next step.
Within this container, the volume data is mounted at `/volume`.

### 3 Copy the data onto the host filesystem
Now that we have a container running with our volume mounted, we can copy the data out using `docker cp`.

Run the following command, replacing `$your_container_name` with the name or id from the previous step:
```bash
docker cp \
  $your_container_name:/volume `# Copies the /volume directory (recursively) from the container` \
  ./volume-on-host `# Copies the volume into the ./volume-on-host directory on the host` \
```

For convenience, here's the command without comments or newlines:
```bash
docker cp $your_container_name:/volume ./volume-on-host
```

### Cleanup
After running this command, the data in the volume will be present in the `./volume-on-host` directory in the current directory.
You can stop the running container with 
```bash
docker kill $your_container_name
```

## What do we mean by _"broken"_

For the purposes of this guide, a Docker installation is called _broken_ if the Docker daemon is unable to start, e.g.
* the Docker daemon hangs when starting up
* the Docker daemon exits during startup
* the Docker daemon does not even begin to start up

If your Docker installation is _missing files_, you probably will not be able to follow up.

If you are able to start the Docker daemon, there are simpler and more reliable guides you can follow to recover data, e.g.
* [Recover orphaned volumes](http://blog.idetailaid.co.uk/how-to-recover-an-orphaned-docker-volume-for-a-data-container/)
* 

## How data is stored in Docker
