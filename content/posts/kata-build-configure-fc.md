---
title: "Kata Containers: Build and configure Firecracker"
date: 2022-03-28T19:33:29Z
---

Picking up from where we left in our [previous post](https://blog.cloudkernels.net/posts/kata-build-source), we will now install AWS Firecracker and configure Kata Containers to use it as their hypervisor.

### Build Firecracker

Kata Containers only support AWS Firecracker v0.23.1
([yet](https://github.com/kata-containers/kata-containers/pull/1519)). To build
Firecracker, we will clone the Github repo and checkout to the 0.23.1 version:

```bash
git clone https://github.com/firecracker-microvm/firecracker.git -b v0.23.1 --depth 1 &&\
  cd firecracker &&\
  git submodule update --init
```

Now we can build the binaries:

**Note** AWS Firecracker uses docker to build the image, so make sure your user
can access the docker daemon, or just run with sudo.

```bash
sudo ./tools/devtool -y build --release
toolchain="$(uname -m)-unknown-linux-musl"
sudo cp build/cargo_target/${toolchain}/release/firecracker /opt/kata/bin/firecracker &&\
sudo cp build/cargo_target/${toolchain}/release/jailer /opt/kata/bin/jailer
```

### devmapper snapshotter

AWS Firecracker requires a block device as the backing store for a VM. To interact with containerd and kata we use the devmapper snapshotter. To check support for your containerd installation, you can run:

```
ctr plugins ls |grep devmapper
```

if the output of the above command is:

```
io.containerd.snapshotter.v1    devmapper                linux/amd64    ok
```
then you can skip this section and move on to `Configure Kata Containers to use Firecracker`

If the output of the above command is:

```
io.containerd.snapshotter.v1    devmapper                linux/amd64    error
```

then we need to setup devmapper snapshotter. Based on a [very useful
guide](https://docs.docker.com/storage/storagedriver/device-mapper-driver/)
from docker, we can set it up using the following scripts:

```bash
#!/bin/bash
set -ex

DATA_DIR=/var/lib/containerd/io.containerd.snapshotter.v1.devmapper
POOL_NAME=containerd-pool

mkdir -p ${DATA_DIR}

# Create data file
sudo touch "${DATA_DIR}/data"
sudo truncate -s 100G "${DATA_DIR}/data"

# Create metadata file
sudo touch "${DATA_DIR}/meta"
sudo truncate -s 10G "${DATA_DIR}/meta"

# Allocate loop devices
DATA_DEV=$(sudo losetup --find --show "${DATA_DIR}/data")
META_DEV=$(sudo losetup --find --show "${DATA_DIR}/meta")

# Define thin-pool parameters.
# See https://www.kernel.org/doc/Documentation/device-mapper/thin-provisioning.txt for details.
SECTOR_SIZE=512
DATA_SIZE="$(sudo blockdev --getsize64 -q ${DATA_DEV})"
LENGTH_IN_SECTORS=$(bc <<< "${DATA_SIZE}/${SECTOR_SIZE}")
DATA_BLOCK_SIZE=128
LOW_WATER_MARK=32768

# Create a thin-pool device
sudo dmsetup create "${POOL_NAME}" \
    --table "0 ${LENGTH_IN_SECTORS} thin-pool ${META_DEV} ${DATA_DEV} ${DATA_BLOCK_SIZE} ${LOW_WATER_MARK}"

cat << EOF
#
# Add this to your config.toml configuration file and restart containerd daemon
#
[plugins]
  [plugins.devmapper]
    pool_name = "${POOL_NAME}"
    root_path = "${DATA_DIR}"
    base_image_size = "10GB"
    discard_blocks = true
EOF
```

Make it executable and run it:

```
sudo chmod +x ~/scripts/devmapper/create.sh && \
  cd ~/scripts/devmapper/ && \
  sudo ./create.sh
```

Now, we can add the devmapper configuration provided from the script to `/etc/containerd/config.toml` and restart containerd.

```bash
sudo systemctl restart containerd
```

We can use `dmsetup` to verify that the thin-pool was created successfully. We should also check that devmapper is registered and running:

```bash
sudo dmsetup ls
# devpool (253:0)
sudo ctr plugins ls | grep devmapper
# io.containerd.snapshotter.v1    devmapper                linux/amd64    ok
```

This script needs to be run only once, while setting up the devmapper snapshotter for containerd. Afterwards, make sure that on each reboot, the thin-pool is initialized from the same data dir. Otherwise, all the fetched containers (or the ones that you’ve created) will be re-initialized. A simple script that re-creates the thin-pool from the same data dir is shown below:

```bash
#!/bin/bash
set -ex

DATA_DIR=/var/lib/containerd/io.containerd.snapshotter.v1.devmapper
POOL_NAME=containerd-pool

# Allocate loop devices
DATA_DEV=$(sudo losetup --find --show "${DATA_DIR}/data")
META_DEV=$(sudo losetup --find --show "${DATA_DIR}/meta")

# Define thin-pool parameters.
# See https://www.kernel.org/doc/Documentation/device-mapper/thin-provisioning.txt for details.
SECTOR_SIZE=512
DATA_SIZE="$(sudo blockdev --getsize64 -q ${DATA_DEV})"
LENGTH_IN_SECTORS=$(bc <<< "${DATA_SIZE}/${SECTOR_SIZE}")
DATA_BLOCK_SIZE=128
LOW_WATER_MARK=32768

# Create a thin-pool device
sudo dmsetup create "${POOL_NAME}" \
    --table "0 ${LENGTH_IN_SECTORS} thin-pool ${META_DEV} ${DATA_DEV} ${DATA_BLOCK_SIZE} ${LOW_WATER_MARK}"
```

We can create a systemd service to run the above script on each reboot:

```bash
sudo nano /lib/systemd/system/devmapper_reload.service
```

The service file:

```
[Unit]
Description=Devmapper reload script

[Service]
ExecStart=/path/to/script/reload.sh

[Install]
WantedBy=multi-user.target
```

Enable the newly created service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable devmapper_reload.service
sudo systemctl start devmapper_reload.service
```

### Configure Kata Containers to use Firecracker

Next, we need to install the Kata Containers-Firecracker configuration file. We will use this file to configure Kata Containers to use the rootfs image we built in our [previous post](https://blog.cloudkernels.net/posts/kata-build-source).

```bash
sudo mkdir -p /opt/kata/configs
sudo install -o root -g root -m 0640 /opt/kata/share/defaults/kata-containers/configuration-fc.toml /opt/kata/configs
sudo sed -i 's/^\(initrd =.*\)/# \1/g' /opt/kata/configs/configuration-fc.toml
# enable seccomp
sudo sed -i '/^disable_guest_seccomp/ s/true/false/' /opt/kata/configs/configuration-fc.toml
```

Make sure that /opt/kata/configs/configuration-fc.toml has an image entry pointing to the rootfs we created:

```bash
17   | image = "/opt/kata/share/kata-containers/kata-containers.img"
```

### Configure containerd

Next, we need to configure containerd. Add a file in your path (eg. `/usr/local/bin/containerd-shim-kata-fc-v2`) with the following contents:

```bash
#!/bin/bash
KATA_CONF_FILE=/opt/kata/configs/configuration-fc.toml /usr/local/bin/containerd-shim-kata-v2 $@
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/containerd-shim-kata-fc-v2
```

Add the relevant section in containerd’s config.toml file (`/etc/containerd/config.toml`):

```
[plugins.cri.containerd.runtimes]
  [plugins.cri.containerd.runtimes.kata-fc]
    runtime_type = "io.containerd.kata-fc.v2"
```

Restart containerd:

```bash
sudo systemctl restart containerd
```

### Verify the installation

We are now ready to launch a container using Kata with Firecracker to verify that everything worked:

```bash
sudo ctr images pull --snapshotter devmapper docker.io/library/ubuntu:latest
sudo ctr run --snapshotter devmapper --runtime io.containerd.run.kata-fc.v2 -t --rm docker.io/library/ubuntu:latest ubuntu-kata-fc-test uname -a
```
