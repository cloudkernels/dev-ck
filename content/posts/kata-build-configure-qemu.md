---
title: "Kata Containers: Build and configure QEMU"
date: 2022-03-28T10:18:06Z
draft: false
---

Picking up from where we left in our [previous
post](/posts/kata-build-source), we will now
install QEMU and configure Kata Containers to use QEMU as their hypervisor.

### Build QEMU

First, we need to build `qemu-system` for the CPU architecture of our host machine. Kata Containers provide scripts to manage the build of QEMU both for
x86 and arm64 hosts. We will be using them to make sure that our QEMU installation is suitable for usage with Kata Containers.

#### Build QEMU for x86

We need to get the correct version of QEMU from the versions file:

```bash
export GOPATH=$(go env GOPATH) && export GO111MODULE=off
source ${GOPATH}/src/github.com/kata-containers/kata-containers/tools/packaging/scripts/lib.sh
qemu_version=$(get_from_kata_deps "assets.hypervisor.qemu.version")
```

Next, we need to get the QEMU source from the matching branch:

```bash
go get -d github.com/qemu/qemu
cd ${GOPATH}/src/github.com/qemu/qemu
git checkout ${qemu_version}
your_qemu_directory=${GOPATH}/src/github.com/qemu/qemu
```

We need to see which version of QEMU we will be building.

```bash
echo ${qemu_version}
# v6.2.0
```

Now we will use the scripts provided to apply the neccessary patches to QEMU. Make sure to change the commands below to match the version of QEMU that you are targeting.

```bash
packaging_dir="${GOPATH}/src/github.com/kata-containers/kata-containers/tools/packaging"
$packaging_dir/scripts/apply_patches.sh $packaging_dir/qemu/patches/6.2.x/
```

Once the patches are successfully applied, we need to install some apt packages that are required to build `qemu-system`:

```bash
sudo apt install ninja-build pkg-config \
  libglib2.0-dev \
  libpixman-1-dev \
  libseccomp-dev \
  libcap-ng-dev \
  librados-dev \
  libpmem-dev \
  librbd-dev \
  libgcrypt-dev
```

Now, we can finally build QEMU:

```bash
cd $your_qemu_directory
$packaging_dir/scripts/configure-hypervisor.sh kata-qemu > kata.cfg
eval ./configure "$(cat kata.cfg)"
make -j $(nproc)
sudo -E make install
```

If you followed the instructions in our previous post and you have set the `PREFIX` variable to `/opt/kata` the `qemu-system-x86_64` binary will be stored under the `/opt/kata/bin/` directory, `tools/virtiofsd/virtiofsd` under the `/opt/kata/libexec/kata-qemu` directory and all the other files will be stored under the `/opt/kata/share/kata-qemu` directory.

We can also install `virtiofsd` and `qemu-system-x86_64` under the default location using the `install_qemu.sh` script.

```bash
go get -d github.com/kata-containers/tests
script -fec 'sudo -E ${GOPATH}/src/github.com/kata-containers/tests/.ci/install_qemu.sh'
```

#### Build QEMU for aarch64

Install requierements:

```bash
sudo apt-get install -y build-essential pkg-config \
  libglib2.0-dev libpixman-1-dev \
  libaio-dev libseccomp-dev \
  libcap-ng-dev librados-dev \
  librbd-dev libzstd-dev
```

Set required environment variables

```bash
export PATH=$PATH:$(go env GOPATH)/bin && \
  export GOPATH=$(go env GOPATH) && \
  export GO111MODULE=off
```

Build QEMU using the `install_qemu.sh` script:

```bash
sudo rm -rf ${GOPATH}/src/github.com/qemu/qemu && \
  go get -d github.com/kata-containers/tests && \
  script -fec 'sudo -E ${GOPATH}/src/github.com/kata-containers/tests/.ci/install_qemu.sh'
```

### Configure Kata Containers

Next, we need to install the Kata Containers configuration file. We will use this file to configure Kata Containers to use the initrd image we built in our previous post. You could also use a rootfs image, if you prefer.
We will also enable guest `seccomp`.

```bash
sudo mkdir -p /opt/kata/configs
sudo install -o root -g root -m 0640 /opt/kata/share/defaults/kata-containers/configuration.toml /opt/kata/configs
# comment out the image entry
sudo sed -i 's/^\(image =.*\)/# \1/g' /opt/kata/configs/configuration.toml
# enable seccomp
sudo sed -i '/^disable_guest_seccomp/ s/true/false/' /opt/kata/configs/configuration.toml
```

Make sure that `/opt/kata/configs/configuration.toml` has a initrd entry pointing to the initrd we created:

```bash
17   | #image = "/opt/kata/share/kata-containers/kata-containers-image.img"
18   │ initrd = "/opt/kata/share/kata-containers/kata-containers-initrd.img"
```

### Configure containerd

You need to edit the containerd config file (usually `/etc/containerd/config.toml`.

First we need to remove `cri` from the diabled plugins list:

```
#disabled_plugins = ["cri"]
disabled_plugins = []
```

We also need to add the relevant handler for the kata runtime:

```
[plugins]
  [plugins.cri]
    [plugins.cri.containerd]
      [plugins.cri.containerd.runtimes]
        [plugins.cri.containerd.runtimes.kata]
         runtime_type = "io.containerd.kata.v2"
         privileged_without_host_devices = true
         [plugins.cri.containerd.runtimes.kata.options]
           ConfigPath = "/opt/kata/configs/configuration.toml"
```

Once the changes are saved, restart containerd: `sudo systemctl restart containerd`.

To verify our installation is successfull, we can launch an Ubuntu test container:

```bash
sudo ctr images pull docker.io/library/ubuntu:latest
sudo ctr run --runtime io.containerd.run.kata.v2 -t --rm docker.io/library/ubuntu:latest ubuntu-kata-test uname -a
# Linux e678ab3c6fc3 5.15.26 #2 SMP Sat Mar 19 21:02:56 UTC 2022 aarch64 aarch64 aarch64 GNU/Linux
```

**Note 1 for aarch64**:

We noticed that, in some instances, trying to launch a container produced the following error:

```bash
# sudo ctr images pull docker.io/library/busybox:latest
# ctr run --runtime io.containerd.run.kata.v2 -t --rm docker.io/library/busybox:latest hello1 sh
ctr: failed to create shim: failed to launch qemu: exit status 1, error messages from qemu log: qemu-system-aarch64: warning: For GICv2 max-cpus must be equal to smp-cpus
qemu-system-aarch64: warning: Overriding specified max-cpus(4) with smp-cpus(1)
qemu-system-aarch64: warning: global kvm-pit.lost_tick_policy has invalid class name
Restoring 1 CPU interfaces, kernel only has 4
: unknown
```

A workaround for that problem was to set the `default_vcpus = 8` in `/opt/kata/configs/configuration.toml`.


**Note 2 for aarch64**:

We tried running on an older kernel, on an NVIDIA Jetson AGX Xavier and on an
NVIDIA Jetson Nano, versions 4.9.253-tegra-virt (getting KVM to work on these
boards is also quite a challenge, more details on a subsequent post). Apart
from the GIC issue above, we saw that virtio-fs is not backported, nor is there
support for the nvdimm stuff, so the following changes to the config file are
needed:

```patch
--- configuration-qemu-stock.toml   2022-03-24 20:55:47.010284096 +0000
+++ configuration-qemu.toml    2022-03-24 21:34:39.725554126 +0000
@@ -103,7 +103,7 @@
 # vCPUs supported by the SB/VM. In general, we recommend that you do not edit this variable,
 # unless you know what are you doing.
 # NOTICE: on arm platform with gicv2 interrupt controller, set it to 8.
-default_maxvcpus = 0
+default_maxvcpus = 1

 # Bridges can be used to hot plug devices.
 # Limitations:
@@ -151,7 +151,7 @@
 #   - virtio-fs (default)
 #   - virtio-9p
 #   - virtio-fs-nydus
-shared_fs = "virtio-fs"
+shared_fs = "virtio-9p"

 # Path to vhost-user-fs daemon.
 virtio_fs_daemon = "/opt/kata/libexec/kata-qemu/virtiofsd"
@@ -292,7 +292,7 @@
 # nvdimm is not supported when `confidential_guest = true`.
 #
 # Default is false
-#disable_image_nvdimm = true
+disable_image_nvdimm = true

 # VFIO devices are hotplugged on a bridge by default.
 # Enable hotplugging on root bus. This may be required for devices with
```

Interenstingly enough, `default_vcpus` must be equal to `default_maxvcpus` on
such platforms.
