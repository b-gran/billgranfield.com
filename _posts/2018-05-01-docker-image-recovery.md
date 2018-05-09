---
layout: post
title: Recovering data from a broken MacOS Docker installation
description: |
  How to recover and restore data (images and volumes) from broken or corrupt MacOS Docker installations.
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

## Data recovery if the Docker daemon will not start
Data recovery is much more difficult if the Docker daemon isn't starting up correctly. In some cases, the HyperKit VM will start, but it will not be able to load the `qcow2` image containing the container data.

The basic idea here is:
0. Install dependencies (there are several)
1. Convert the `qcow2` image to a FUSE filesystem image
2. Mount the FUSE filesystem
3. Copy the data out of the FUSE filesystem onto the host

We'll be using [qcow2-fuse](https://github.com/vasi/qcow2-fuse) to convert the `Docker.qcow2` to a FUSE filesystem and [ext4fuse](https://github.com/gerard/ext4fuse) to mount the FUSE filesystem on MacOS. We'll also need [osxfuse](https://osxfuse.github.io/) which is a dependency of the other tools.

### 0 Installing dependencies
Homebrew is required to complete the following steps.

##### `osxfuse`
First, we'll need `osxfuse`. You can download the latest installer from [their downloads page](https://github.com/osxfuse/osxfuse/releases), or run the following command:
```bash
brew cask install osxfuse
```

`osxfuse` is a generic library for working with FUSE filesystems on MacOS.

##### `ext4fuse`
Next, we'll install `ext4fuse`:
```bash
brew install ext4fuse
```

`ext4fuse` is a tool for mounting FUSE filesystems on MacOS.

##### `qcow2-fuse`
Finally, we'll install `qcow2-fuse`. `qcow2-fuse` requires `pkg-config` -- if you don't already have it installed, run
```bash
brew install pkg-config
```

`qcow2-fuse` is a tool for converting `qcow2` images into FUSE filesystems.

###### Setting up the correct Rust version
`qcow2-fuse` is built with [Rust](https://www.rust-lang.org/en-US/). We'll also need a Rust specific version: 1.11.0.
To install Rust, we'll be using [rustup](https://rustup.rs/), a Rust version manager.
To install `rustup`, run the following command:
```bash
curl https://sh.rustup.rs -sSf | sh
```

Once `rustup` is installed, we can set the Rust version (`rustup` will download the necessary binaries):
```bash
rustup default 1.11.0
```

Make sure the correct Rust version is installed by running 
```bash
rustc --version
```

###### Building `qcow2-fuse`
We're now ready to install `qcow2-fuse`. Unfortunately, the published `cargo` package will no longer build, so we'll be building `qcow2-fuse` from source.
First, choose a directory somewhere to clone the `qcow2-fuse` repository. It's not especially important to keep this directory around, so a temp directory is fine.
Then, clone the `qcow2-fuse` repository and `cd` into it:
```bash
git clone https://github.com/vasi/qcow2-fuse.git
cd qcow2-fuse
```

Now we can build `qcow2-fuse` by running the following command from the `qcow2-fuse` repo root:
```bash
# Ensure you're in the root of the qcow2-fuse repo
# cd qcow2-fuse

cargo install
```

`cargo` will place the `qcow2-fuse` binaries in `$HOME/.cargo/bin` by default, and this should be on your `$PATH`. 
If not, add `$HOME/.cargo/bin` to your `$PATH`, or copy the binary to some known location.

Verify that `qcow2-fuse` was built correctly by running 
```bash
qcow2-fuse --version
```

### 1 Convert the `qcow2` image to a FUSE filesystem image
Before we can mount the Docker VM image, we need to convert it into a format we are able to mount.

Copy your `Docker.qcow2` somewhere. You'll be creating a FUSE filesystem image in the same directory.

The following command converts the `qcow2` image to a FUSE filesystem image located in the `mnt` directory relative to the current directory:
```bash
qcow2-fuse -o allow_other -o rdonly Docker.qcow2 mnt
```

The `-o allow_other -o rdonly` flags specify that any user account should be able to access the FUSE image, and that the FUSE filesystem should be read-only.

After running the command, a `mnt` directory containing the FUSE image (a file called `Docker`) should be present in the current directory.

### 2 Mount the FUSE filesystem image
We're now ready mount the FUSE image as a volume. 

The first step is to `attach` the FUSE image as a device. The following command (which assumes there's a FUSE image called `Docker` in the `mnt` directory) will attach the FUSE image as a device:
```bash
hdiutil attach -imagekey diskimage-class=CRawDiskImage -nomount mnt/Docker
```

MacOS doesn't know how to mount FUSE images, so the `-nomount` flag is required.
The output of the command will look something like this:
```bash
/dev/disk4              FDisk_partition_scheme
/dev/disk4s1            Linux_Swap
/dev/disk4s2            Linux
```

We're interested in the line that looks like this:
```bash
/dev/disk4s2            Linux
```
which is path to the EXT4 partition of the FUSE image.

We're now ready to mount the FUSE filesystem. Run the following command, replacing `/dev/disk4s2` with correct EXT4 device for your system:
```bash
sudo ext4fuse /dev/disk4s2 volume
```

This command mounts the EXT4 partition at the path `volume` relative to the current directory.
After running this command, the `Docker.qcow2` filesystem is **mounted and accessible as a normal filesystem at the `volume` path**.

### 3 Copy the data out of the FUSE filesystem onto the host
We're finally ready to copy the data out of the Docker VM image.

We can use ordinary *nix tools to work with this filesystem. We will need to run these commands as `sudo`. 
```bash
# List the files at the root of the FUSE filesystem
sudo ls -al volume

# Copy all of the volumes on the FUSE filesystem into the current directory
sudo cp -R volume/lib/docker/volumes ./docker_volumes
```

Depending on your Docker/HyperKit version, the contents of your FUSE filesystem may differ. You should explore the filesystem to figure out exactly where your volume data resides.

The `ext4fuse` is pretty unreliable, and during the course of interacting with the FUSE filesystem, you may see errors like this:
```bash
$ sudo ls volume/lib
ls: volume/lib: Device not configured
```

To resolve this issue, simply unmount and then remount the EXT4 volume (where `volume` is the path to the EXT4 volume you mounted in the previous step):
```bash
$ sudo diskutil unmount volume
Unmount successful for volume

$ sudo ext4fuse /dev/disk4s2 volume
```

## Wrapping up
With the volume data copied to your host filesystem, you can restore the volume at your leisure. 
Using these methods, you can backup and recover your Docker data whether or not the HyperKit VM is actually able to start.
