---
title: "Build Kata Containers from source on x86 and arm64"
date: 2022-03-24T10:18:06Z
draft: false
---

[Kata Containers](https://katacontainers.io) enable containers to be seamlessly
executed in Virtual Machines. Kata Containers are as light and fast as
containers and integrate with the container management layers, while also
delivering the security advantages of VMs. Kata Containers is the result of
merging two existing open source projects: Intel Clear Containers and Hyper
runV.

Kata Containers consist of several components. For amd64 machines,
[binaries](https://github.com/kata-containers/kata-containers/releases) are
provided through the
[formal](https://github.com/kata-containers/kata-containers/blob/main/docs/Release-Process.md)
release process. However, for arm64, binary files are not available ([just
yet](https://github.com/kata-containers/kata-containers/issues/2639)).

In this post, we will be going through the steps to build kata containers from
source, both for amd64 and arm64 architectures. In
[follow-up](https://cloudkernels.github.io/posts/kata-build-configure-qemu/)
[posts](https://cloudkernels.github.io/posts/kata-build-configure-fc/), we go
through the steps to build and configure [QEMU](https://github.com/qemu/qemu)
and [AWS Firecracker](https://github.com/firecracker-microvm/firecracker) as
VMMs for Kata Containers.


### Install requirements

To build Kata Containers we need to install Rust v1.58.1, Go v1.16.10, Docker and some apt/snap packages. The specific versions may change,
so make sure to check the [versions database](https://github.com/kata-containers/kata-containers/blob/main/versions.yaml).

#### Apt/Snap Packages:

We need to install `gcc`, `make` and `yq v3`. `containerd` and `runc` are installed by the Docker install script, in the following steps.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install gcc make snapd -y
sudo snap install yq --channel=v3/stable
```

#### Rust (version 1.58.1):

We will use `rustup` to install and set Rust 1.58.1 as our default toolchain:

```bash
down_dir=$(mktemp -d)
pushd $down_dir
wget -q https://static.rust-lang.org/rustup/dist/$(uname -p)-unknown-linux-gnu/rustup-init
sudo chmod +x rustup-init
./rustup-init -q -y --default-toolchain 1.58.1
source $HOME/.cargo/env
popd
rm -rf $down_dir
```

#### Go (version 1.16.10)

We will download the appropriate Go binaries and add them to the `PATH` environment variable:

```bash
down_dir=$(mktemp -d)
pushd $down_dir
wget -q https://go.dev/dl/go1.16.10.linux-$(dpkg --print-architecture).tar.gz
sudo mkdir -p /usr/local/go1.16
sudo tar -C /usr/local/go1.16 -xzf go1.16.10.linux-$(dpkg --print-architecture).tar.gz
echo 'export PATH=$PATH:/usr/local/go1.16/go/bin' >> $HOME/.profile
source $HOME/.profile
popd
rm -rf $down_dir
```

#### Docker

We will install Docker using the provided convenience script:

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc -y > /dev/null 2>&1
sudo rm -rf /var/lib/docker/
down_dir=$(mktemp -d)
pushd $down_dir
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
# Optionally add user to docker group to run docker without sudo
# sudo usermod -aG docker $USER
popd
rm -rf $down_dir
```

### Build Kata components

#### Build kata-runtime

First, we need to set the correct Go environment variables:

```bash
export PATH=$PATH:$(go env GOPATH)/bin && \
  export GOPATH=$(go env GOPATH) && \
  export GO111MODULE=off
```

We will use `go get` to download kata-containers source code:

```bash
go get -d -u github.com/kata-containers/kata-containers
```

We are now ready to build the `kata-runtime`:

```bash
pushd $GOPATH/src/github.com/kata-containers/kata-containers/src/runtime
export GO111MODULE=on
export PREFIX=/opt/kata
make
popd
```
To install the binaries to a specific path (say `/opt/kata`) we need to specify
the `PREFIX` environment variable prior to installing:

```bash
pushd $GOPATH/src/github.com/kata-containers/kata-containers/src/runtime
export PREFIX=/opt/kata
sudo -E PATH=$PATH -E PREFIX=$PREFIX make install
popd
```
Kata binaries are now installed in `/opt/kata/bin` and configs are installed in
`/opt/kata/share/defaults/kata-containers/`.

It is recommended you add a symbolic link to /opt/kata/bin/kata-runtime and
/opt/kata/bin/containerd-shim-kata-v2 in order for containerd to reach these
binaries from the default system `PATH`.

```bash
sudo ln -s /opt/kata/bin/kata-runtime /usr/local/bin
sudo ln -s /opt/kata/bin/containerd-shim-kata-v2 /usr/local/bin
```
#### Create a rootfs & initrd image

We can use either a rootfs or initrd image to launch Kata Containers with Qemu.
However, AWS Firecracker does not work with initrd images, so we will be using a
rootfs image for Kata with Firecracker. If you do not want to use QEMU or QEMU
with initrd, you can skip building the initrd image.

Create the rootfs base image:

```bash
export ROOTFS_DIR=${GOPATH}/src/github.com/kata-containers/kata-containers/tools/osbuilder/rootfs-builder/rootfs
cd $GOPATH/src/github.com/kata-containers/kata-containers/tools/osbuilder/rootfs-builder
# you may change the distro (in this case we used ubuntu). to get supported distros list, run ./rootfs.sh -l
script -fec 'sudo -E GOPATH=$GOPATH AGENT_INIT=yes USE_DOCKER=true ./rootfs.sh ubuntu'
```

**Note for arm64**:

We noticed that in some instances the kata-agent compilation failed.
A possible workaround was to remove the USE_DOCKER variable. This requires `qemu-img` command to be available on your system.
You can install it with `sudo apt install -y qemu-utils`.

```bash
export ROOTFS_DIR="${GOPATH}/src/github.com/kata-containers/kata-containers/tools/osbuilder/rootfs-builder/rootfs"
sudo rm -rf ${ROOTFS_DIR}
cd $GOPATH/src/github.com/kata-containers/kata-containers/tools/osbuilder/rootfs-builder
script -fec 'sudo -E GOPATH=$GOPATH AGENT_INIT=yes ./rootfs.sh ubuntu'
```

Build a kata rootfs image:

```bash
cd $GOPATH/src/github.com/kata-containers/kata-containers/tools/osbuilder/image-builder && \
  script -fec 'sudo -E USE_DOCKER=true -E AGENT_INIT=yes ./image_builder.sh ${ROOTFS_DIR}'
```

Install the kata rootfs image:

```bash
export PREFIX=/opt/kata
commit=$(git log --format=%h -1 HEAD) && \
  date=$(date +%Y-%m-%d-%T.%N%z) && \
  image="kata-containers-${date}-${commit}" && \
  sudo install -o root -g root -m 0640 -D kata-containers.img "$PREFIX/share/kata-containers/${image}" && \
  (cd $PREFIX/share/kata-containers && sudo ln -sf "$image" kata-containers.img)
```

(OPTIONAL) Next we will build an initrd image:

```bash
cd $GOPATH/src/github.com/kata-containers/kata-containers/tools/osbuilder/initrd-builder
script -fec 'sudo -E AGENT_INIT=yes USE_DOCKER=true ./initrd_builder.sh ${ROOTFS_DIR}'
```

(OPTIONAL) Once the image is built, we install it:

```bash
export PREFIX=/opt/kata
commit=$(git log --format=%h -1 HEAD) && \
  date=$(date +%Y-%m-%d-%T.%N%z) && \
  image="kata-containers-initrd-${date}-${commit}" && \
  sudo install -o root -g root -m 0640 -D kata-containers-initrd.img "$PREFIX/share/kata-containers/${image}" && \
  (cd $PREFIX/share/kata-containers && sudo ln -sf "$image" kata-containers-initrd.img)
```

#### Build Kata Containers kernel

First, we need some additional packages to build the kernel:

```bash
sudo apt install -y libelf-dev bison flex
```

Setup the kernel source code:

```bash
cd $GOPATH/src/github.com/kata-containers/kata-containers/tools/packaging/kernel
./build-kernel.sh -d setup
```

Build the kernel:

```bash
./build-kernel.sh -d build
```

Install the kernel in the default path for Kata:

```bash
export PREFIX=/opt/kata
sudo -E PATH=$PATH -E PREFIX=$PREFIX ./build-kernel.sh -d install
```

**Note**:

We noticed that in some instances the installation or build process failed with the following error: `ERROR: path to kernel does not exist, use build-kernel.sh setup`. We mitigated this problem by specifying the version:
```bash
./build-kernel.sh -d -v 5.15.26 build
export PREFIX=/opt/kata
sudo -E PATH=$PATH -E PREFIX=$PREFIX ./build-kernel.sh -d -v 5.15.26 install
``` 

### Next steps

At this point we have succesfully built all the Kata components. All the binaries we built are stored under the `/opt/kata/bin` dir:

```bash
$ ls -l /opt/kata/bin/
total 142296
-rwxr-xr-x 1 root root 50919312 Mar  25 15:32 containerd-shim-kata-v2
-rwxr-xr-x 1 root root    16691 Mar  25 15:32 kata-collect-data.sh
-rwxr-xr-x 1 root root 42093616 Mar  25 15:32 kata-monitor
-rwxr-xr-x 1 root root 52673784 Mar  25 15:32 kata-runtime
```

The rootfs image, the initrd image and the kernel are stored under the `/opt/kata/share/defaults/kata-containers` dir:

```bash
$ ls -l /opt/kata/share/kata-containers/
total 221972
-rw-r--r-- 1 root root     72480 Μαρ  25 15:51 config-5.15.26
-rw-r----- 1 root root 134217728 Μαρ  25 15:41 kata-containers-2022-03-25-15:41:55.534872004+0200-486322a0
lrwxrwxrwx 1 root root        59 Μαρ  25 15:41 kata-containers.img -> kata-containers-2022-03-25-15:41:55.534872004+0200-486322a0
-rw-r----- 1 root root  27627256 Μαρ  24 14:43 kata-containers-initrd-2022-03-24-14:43:59.501993241+0200-853dd98b
-rw-r----- 1 root root  27626874 Μαρ  25 15:42 kata-containers-initrd-2022-03-25-15:42:28.034074480+0200-486322a0
lrwxrwxrwx 1 root root        66 Μαρ  25 15:42 kata-containers-initrd.img -> kata-containers-initrd-2022-03-25-15:42:28.034074480+0200-486322a0
-rw-r--r-- 1 root root  38736168 Μαρ  25 15:51 vmlinux-5.15.26-90
lrwxrwxrwx 1 root root        18 Μαρ  25 15:51 vmlinux.container -> vmlinux-5.15.26-90
-rw-r--r-- 1 root root   5795664 Μαρ  25 15:51 vmlinuz-5.15.26-90
lrwxrwxrwx 1 root root        18 Μαρ  25 15:51 vmlinuz.container -> vmlinuz-5.15.26-90
```

The configuration files are stored under the `/opt/kata/share/defaults/kata-containers` dir:

```bash
ls -l /opt/kata/share/defaults/kata-containers
total 72
-rw-r--r-- 1 root root  9717 Μαρ  25 15:32 configuration-acrn.toml
-rw-r--r-- 1 root root 13535 Μαρ  25 15:32 configuration-clh.toml
-rw-r--r-- 1 root root 15364 Μαρ  25 15:32 configuration-fc.toml
-rw-r--r-- 1 root root 25701 Μαρ  25 15:32 configuration-qemu.toml
lrwxrwxrwx 1 root root    23 Μαρ  25 15:32 configuration.toml -> configuration-qemu.toml
```

Now, we need to install a hypervisor (eg QEMU or Firecracker) and configure
Kata Containers and containerd accordingly.  In our upcoming posts we will go
through the process of building and configuring both
[QEMU](/posts/kata-build-configure-qemu/) and
[AWS
Firecracker](/posts/kata-build-configure-fc/) for
x86 and arm64 hosts. Happy hacking!
