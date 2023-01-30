---
title: "Running containers on Firecracker microVMs using kata on kubernetes"
date: 2021-07-09T10:14:04+01:00
---

This is the first of a number of posts regarding the orchestration, deployment
and scaling of containerized applications in VM sandboxes using kubernetes,
kata-containers and AWS Firecracker microVMs. We have gathered some notes
during the installation and configuration of the necessary components and we
thought they might be useful to the community, especially with regards to the
major pain points in trying out recent open-source projects and technologies.

### About Orchestration, the Edge, and Kata Containers

To manage and orchestrate containers in a cluster, the community is using
[kubernetes][k8s] (k8s), a powerful, open-source system for automating the
deployment, scaling and management of containerized applications. To
accommodate the vast majority of options and use-cases, k8s supports a number
of container runtimes: the piece of software that sets up all the necessary
components in the system to run the containerized application. For more
information on the container runtime support or k8s visit the [official
documentation][k8s-docs]. 

#### k8s at the edge

A stripped down version of k8s is available through [K3s][k3s], a kubernetes
distribution designed for production workloads in unattended,
resource-constrained, remote locations or inside IoT appliances. K3s is
packaged as a single <40MB binary that reduces the dependencies and steps
needed to install, run and auto-update a production Kubernetes cluster. K3s
supports various architectures (amd64, ARMv8, ARMv8) with binaries and
multiarch images available for all. K3s works great from something as small as
a Raspberry Pi to an AWS a1.4xlarge 32GiB server. K3s is also great for trying
out k8s on a local machine without messing up the OS.

#### container sandboxing

[Kata Containers][kata] enable containers to be seamlessly executed in Virtual
Machines. Kata Containers are as light and fast as containers and integrate
with the container management layers, while also delivering the security
advantages of VMs. Kata Containers is the result of merging two existing open
source projects: [Intel Clear Containers][clear] and [Hyper runV][runv].

Kata Containers integrate with k8s easily; however, there are some (minor) pain
points when adding more options to the mix. For instance, choosing AWS
Firecracker as the VMM for the sandbox environment, brings a storage backend
dependency: device mapper. In this post, we will be going through the steps
needed to setup kata containers with Firecracker, focusing on the device mapper
setup for k8s and k3s. 

### Kata containers and containerd

First, lets start of by installing kata containers and add the relevant handler
to containerd. Following the [docs][kata-doc] is pretty straightforward,
especially [this][kata-containerd-doc] one. To make sure everything is working
as expected, you could try out a couple of examples found
[here][kata-containerd-examples].

In short, the needed steps are:

##### install kata binaries

Download a release from: `https://github.com/kata-containers/kata-containers/releases`

(using v2.1.1, released in June 2021):

```
$ wget https://github.com/kata-containers/kata-containers/releases/download/2.1.1/kata-static-2.1.1-x86_64.tar.xz
```

Unpack the binary

```
$ xzcat kata-static-2.1.1-x86_64.tar.xz | sudo tar -xvf - -C /
```

by default, kata is being installed in `/opt/kata`. Check the installed version by running:

```
$ /opt/kata/bin/kata-runtime --version
```

It should output something like the following:

```
$ /opt/kata/bin/kata-runtime --version
kata-runtime  : 2.1.1
   commit   : 0e2be438bdd6d213ac4a3d7d300a5757c4137799
   OCI specs: 1.0.1-dev
```

It is recommended you add a symbolic link to `/opt/kata/bin/kata-runtime` and
`/opt/kata/bin/containerd-shim-kata-v2` in order for containerd to reach these
binaries from the default system `PATH`.

```
$ sudo ln -s /opt/kata/bin/kata-runtime /usr/local/bin
$ sudo ln -s /opt/kata/bin/containerd-shim-kata-v2 /usr/local/bin
```

##### install containerd

To install containerd, you can grab a release from
`https://github.com/containerd/containerd/releases` or use the package manager
of your distro:

```
$ sudo apt-get install containerd
```

A fairly recent containerd version is recommended (e.g. we've only tested with
containerd versions `v1.3.9` and above).

Add the kata configuration to containerd' `config.toml` (`/etc/containerd/config.toml`):

```
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "kata"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
          runtime_type = "io.containerd.kata.v2"
```

If `/etc/containerd/config.toml` is not present, create it using the following
command:

```
$ sudo containerd config default > /etc/containerd/config.toml
```

and add the above snippet to the relevant section.

##### test kata with containerd

Now, after we restart containerd:

```
sudo systemctl restart containerd
```

we should be able to launch a Ubuntu test container using kata containers and
containerd:

```
$ sudo ctr image pull docker.io/library/ubuntu:latest
docker.io/library/ubuntu:latest:                                                  resolved       |++++++++++++++++++++++++++++++++++++++| 
index-sha256:82becede498899ec668628e7cb0ad87b6e1c371cb8a1e597d83a47fac21d6af3:    done           |++++++++++++++++++++++++++++++++++++++| 
manifest-sha256:1e48201ccc2ab83afc435394b3bf70af0fa0055215c1e26a5da9b50a1ae367c9: done           |++++++++++++++++++++++++++++++++++++++| 
layer-sha256:16ec32c2132b43494832a05f2b02f7a822479f8250c173d0ab27b3de78b2f058:    done           |++++++++++++++++++++++++++++++++++++++| 
config-sha256:1318b700e415001198d1bf66d260b07f67ca8a552b61b0da02b3832c778f221b:   done           |++++++++++++++++++++++++++++++++++++++| 
elapsed: 7.2 s                                                                    total:  27.2 M (3.8 MiB/s)                                       
unpacking linux/amd64 sha256:82becede498899ec668628e7cb0ad87b6e1c371cb8a1e597d83a47fac21d6af3...
done
$ ctr run --runtime io.containerd.run.kata.v2 -t --rm docker.io/library/ubuntu:latest ubuntu-kata-test /bin/bash
root@clr-d25fa567d2f440df9eb4316de1699b51:/# uname -a
Linux clr-d25fa567d2f440df9eb4316de1699b51 5.10.25 #1 SMP Fri Apr 9 18:18:14 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
```

#### Kata containers and AWS Firecracker

The above example is using the QEMU/KVM hypervisor. To use a different
hypervisor (eg. AWS Firecracker), we need to make sure we honor all the needed
prerequisites, so that the VM sandbox is spawned correctly and is able to host
the containerized application we want to run. Let's take a step back and see how
kata containers fetch the container image and run the application in the
sandboxed environment.

##### kata containers execution flow

As mentioned earlier, Kata containers provide a way for containerized
application to run inside a VM sandbox. Additionally, kata containers manage
the container execution via a runtime system on the Host and an agent running
in the sandbox.  The containerized application is packaged in a container
image, which is pulled outside the sandbox environment, along with its metadata
description (the `json` file). These two components comprise the *container
bundle*. In order to run the container inside the sandbox environment (that is,
the VM) the container rootfs (the layer stack) must be somehow exposed from the
host to the VM. Then, the *kata agent*, which runs inside the VM, creates a
container from the exposed rootfs. 

In the case of QEMU/KVM, the container rootfs is exposed using a virtiofs
shared directory between the host and the guest. In the case of AWS
Firecracker, however, virtiofs is [not supported][fc-virtiofs]. So, the only
option is to use a virtio block device. 

##### devmapper

In order to expose the container inside an AWS Firecracker sandbox environment,
the kata runtime system expects that containerd uses the [devmapper
snapshotter][devmapper-doc]. Essentially, the container rootfs is a device
mapper snapshot, hot-plugged to AWS Firecracker as a virtio block device. The
*kata agent* running in the VM finds the mount point inside the guest and
issues the relevant command to libcontainerd to create and spawn the container.


So, in order to glue all the above together, we need containerd configured with
the devmapper snapshotter. The first step is to setup a device mapper
thin-pool. On a local dev environment you can use loopback devices.

A simple script from the [devmapper snapshotter][devmapper-doc] documentation
is show below:


```sh
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
sudo truncate -s 40G "${DATA_DIR}/meta"

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
    base_image_size = "40GB"
EOF
```

This script needs to be run only once, while setting up the devmapper
snapshotter for containerd. Afterwards, make sure that on each reboot, the
thin-pool is initialized from the same data dir. Otherwise, all the fetched
containers (or the ones that you've created will be re-initialized). As simple
script that re-creates the thin-pool from the same data dir is show below:

```sh
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

After a containerd restart (`systemctl restart containerd`), we should be able
to see the plugin registered and working correctly:

```
$ ctr plugins ls |grep devmapper
io.containerd.snapshotter.v1    devmapper                linux/amd64    ok     
```

##### containerd handler setup

The last step to setup AWS Firecracker with kata and containerd, is add the
relevant handler to containerd `config.toml`. To use a separate handler and
keep both QEMU/KVM and AWS Firecracker options as the hypervisor for kata
containers, we create a simple script that calls the kata containers runtime
with a different config file. Add a file in your path (eg.
`/usr/local/bin/containerd-shim-kata-fc-v2`) with the following contents:

```sh
#!/bin/bash
KATA_CONF_FILE=/opt/kata/share/defaults/kata-containers/configuration-fc.toml /opt/kata/bin/containerd-shim-kata-v2 $@
```

make it executable:

```
$ chmod +x /usr/local/bin/containerd-shim-kata-fc-v2
```

and add the relevant section in containerd's `config.toml` file (`/etc/containerd/config.toml`):

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata-fc]
    runtime_type = "io.containerd.kata-fc.v2"
```

After a containerd restart (`systemctl restart containerd`), we should be able
to launch a Ubuntu test container using kata containers and AWS Firecracker:

```
root@nuc8:~# ctr images pull --snapshotter devmapper docker.io/library/ubuntu:latest
docker.io/library/ubuntu:latest:                                                  resolved       |++++++++++++++++++++++++++++++++++++++| 
index-sha256:82becede498899ec668628e7cb0ad87b6e1c371cb8a1e597d83a47fac21d6af3:    done           |++++++++++++++++++++++++++++++++++++++| 
manifest-sha256:1e48201ccc2ab83afc435394b3bf70af0fa0055215c1e26a5da9b50a1ae367c9: done           |++++++++++++++++++++++++++++++++++++++| 
config-sha256:1318b700e415001198d1bf66d260b07f67ca8a552b61b0da02b3832c778f221b:   done           |++++++++++++++++++++++++++++++++++++++| 
layer-sha256:16ec32c2132b43494832a05f2b02f7a822479f8250c173d0ab27b3de78b2f058:    done           |++++++++++++++++++++++++++++++++++++++| 
elapsed: 7.9 s                                                                    total:  27.2 M (3.4 MiB/s)                                       
unpacking linux/amd64 sha256:82becede498899ec668628e7cb0ad87b6e1c371cb8a1e597d83a47fac21d6af3...
done
root@nuc8:~# ctr run --snapshotter devmapper --runtime io.containerd.run.kata-fc.v2 -t --rm docker.io/library/ubuntu:latest ubuntu-kata-fc-test uname -a
Linux clr-f083d4640978470da04f091181bb9e95 5.10.25 #1 SMP Fri Apr 9 18:18:14 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
```


### Configure k8s to use AWS Firecracker with kata containers

Following the above steps, integrating kata containers with k8s is a piece of
cake ;-)

Two things are needed to create a pod with kata and AWS Firecracker:

  - make the containerd snapshotter default
  - add the kata containers runtime class to k8s

To configure the devmapper snapshotter as the default snapshotter in containerd add the following lines to `/etc/containerd/config.toml`:

```
[plugins.cri.containerd]
  snapshotter = "devmapper"
```

To add the kata containers runtime class to k8s, we need to create a simple YAML file (`kata-fc-rc.yaml`) containing the name of the `kata-fc` handler we configured earlier:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-fc
handler: kata-fc
```

and apply it to our k8s cluster:

```
$ kubectl apply -f kata-fc-rc.yaml
```

After a containerd restart (`systemctl restart containerd`), we should be able to create a pod 
to see the plugin registered and working correctly. Take for instance the following YAML file (`nginx-kata-fc.yaml`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-kata-fc
spec:
  runtimeClassName: kata-fc
  containers:
  - name: nginx
    image: nginx
```

and apply it:

```
$ kubectl apply -f nginx-kata-fc.yaml
```

the output should be something like the following:

```
$ kubectl apply -f nginx-kata-fc.yaml
pod/nginx-kata-fc created
```

Inspecting the pod in the node created should give us the following:

```
$ kubectl describe pod nginx-kata-fc
Name:         nginx-kata-fc
[snipped]
Containers:
  nginx:
    Container ID:   containerd://bbb6dc73c3a0c727dae81b4be0b93d853c9e2a7843e3f037b934bcb5aea89ece
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:8f335768880da6baf72b70c701002b45f4932acae8d574dedfddaf967fc3ac90
    Port:           <none>
    Host Port:      <none>
    State:          Running
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-psxw6 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-psxw6:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              katacontainers.io/kata-runtime=true
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason   Age   From     Message
  ----    ------   ----  ----     -------
  Normal  Pulling  107s  kubelet  Pulling image "nginx"
  Normal  Pulled   87s   kubelet  Successfully pulled image "nginx" in 20.37180329s
  Normal  Created  87s   kubelet  Created container nginx
  Normal  Started  87s   kubelet  Started container nginx
```

Digging in a bit deeper:

```
$ ps -ef |grep firecracker
root     2705170 2705157  1 [snipped] ?        00:00:00 /opt/kata/bin/firecracker --api-sock /run/vc/firecracker/00fedc54a0bb0e55bd0513e0cf3c551f/root/run/firecracker.socket --config-file /run/vc/firecracker/00fedc54a0bb0e55bd0513e0cf3c551f/root/fcConfig.json
$ kubectl exec -it nginx-kata-fc -- /bin/bash
root@nginx-kata-fc:/# uname -a
Linux nginx-kata-fc 5.10.25 #1 SMP Fri Apr 9 18:18:14 UTC 2021 x86_64 GNU/Linux
root@nginx-kata-fc:/# cat /proc/cpuinfo |head -n 10
processor	: 0
vendor_id	: GenuineIntel
cpu family	: 6
model		: 142
model name	: Intel(R) Xeon(R) Processor @ 3.00GHz
stepping	: 10
microcode	: 0x1
cpu MHz		: 3000.054
cache size	: 4096 KB
physical id	: 0
root@nginx-kata-fc:/# cat /proc/meminfo |head -n 10
MemTotal:        2043696 kB
MemFree:         1988208 kB
MemAvailable:    1997152 kB
Buffers:            2172 kB
Cached:            31224 kB
SwapCached:            0 kB
Active:            16336 kB
Inactive:          21264 kB
Active(anon):         40 kB
Inactive(anon):     4204 kB
```

Looking into the container id via `ctr`, we can see that is using the kata-fc handler:

```
$ ctr -n k8s.io c ls |grep bbb6dc73c3a0c727dae81b4be0b93d853c9e2a7843e3f037b934bcb5aea89ece
bbb6dc73c3a0c727dae81b4be0b93d853c9e2a7843e3f037b934bcb5aea89ece docker.io/library/nginx:latest  io.containerd.kata-fc.v2          
```

As a side note, containerd matches the handler notation with an executable like this:

```
io.containerd.kata-fc.v2 -> containerd-shim-kata-fc-v2
```

so the name of the script we created above
(`/usr/local/bin/containerd-shim-kata-fc-v2`) must match the name of the
handler in containerd `config.toml` using the above notation.



### Configure k3s to use AWS Firecracker with kata containers

k3s ships with a minimal version of containerd where the devmapper snapshotter
plugin is not included.

As a result, in order to add AWS Firecracker support to k3s, we either need to
configure k3s to use a different containerd binary or build k3s-containerd from
[source](https://github.com/k3s-io/k3s), patch it and inject it to
`/var/lib/rancher/k3s/data/current/bin`.

#### Use external containerd

Make sure you have configured containerd to use the cri devmapper plugin
correctly (see above). Then just install k3s using the system's containerd.

```
curl -sfL https://get.k3s.io  | INSTALL_K3S_EXEC="--container-runtime-endpoint unix:///run/containerd/containerd.sock" sh -
```

#### Build k3s' containerd

To build k3s' containerd we need to patch the source in order to enable the devmapper snapshotter. So:

(i) get the source:

```
$ git clone https://github.com/k3s-io/k3s
```

(ii) patch with the following file (`devmapper_patch.txt`):

```diff
diff --git a/pkg/containerd/builtins_linux.go b/pkg/containerd/builtins_linux.go
index 280b09ce..89c700eb 100644
--- a/pkg/containerd/builtins_linux.go
+++ b/pkg/containerd/builtins_linux.go
@@ -25,5 +25,6 @@ import (
        _ "github.com/containerd/containerd/runtime/v2/runc/options"
        _ "github.com/containerd/containerd/snapshots/native"
        _ "github.com/containerd/containerd/snapshots/overlay"
+       _ "github.com/containerd/containerd/snapshots/devmapper"
        _ "github.com/containerd/fuse-overlayfs-snapshotter/plugin"
 )
diff --git a/vendor/modules.txt b/vendor/modules.txt
index 91554f14..054d04bf 100644
--- a/vendor/modules.txt
+++ b/vendor/modules.txt
@@ -307,6 +307,8 @@ github.com/containerd/containerd/services/snapshots
 github.com/containerd/containerd/services/tasks
 github.com/containerd/containerd/services/version
 github.com/containerd/containerd/snapshots
+github.com/containerd/containerd/snapshots/devmapper
+github.com/containerd/containerd/snapshots/devmapper/dmsetup
 github.com/containerd/containerd/snapshots/native
 github.com/containerd/containerd/snapshots/overlay
 github.com/containerd/containerd/snapshots/proxy
```

```
$ cd k3s
$ patch -p1 < ../devmapper_patch.txt
patching file pkg/containerd/builtins_linux.go
patching file vendor/modules.txt
```

and (iii) build:

```
$ sudo apt-get install btrfs-progs libbtrfs-dev
$ mkdir -p build/data && ./scripts/download && go generate
$ go get -d github.com/containerd/containerd/snapshots/devmapper
$ SKIP_VALIDATE=true make # because we have local changes
```

Hopefully, you'll be presented with a k3s binary ;-)


#### install kata-containers on k3s

After a successful build & run of k3s, you should be able to list the available nodes for your local cluster:

```
$ k3s kubectl get nodes --show-labels
NAME                     STATUS   ROLES                  AGE   VERSION        LABELS
mycluster.localdomain   Ready    control-plane,master   11m   v1.21.3+k3s1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=mycluster.localdomain,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=true,node-role.kubernetes.io/master=true,node.kubernetes.io/instance-type=k3s
```

Apart from the manual install we saw above for k8s, we could just use the
well documented process in the [kata-deploy][kata-deploy] section of the
kata-containers repo.

So, just run the following:

```
$ k3s kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-rbac/base/kata-rbac.yaml
serviceaccount/kata-label-node created
clusterrole.rbac.authorization.k8s.io/node-labeler created
clusterrolebinding.rbac.authorization.k8s.io/kata-label-node-rb created
$ k3s kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-deploy/base/kata-deploy.yaml
daemonset.apps/kata-deploy created
```

This will fetch the necessary binaries and install them on /opt/kata (via the
daemonset). It will also add the relevant handlers on
`/etc/containerd/config.toml`. However, in order to use these handlers we need to
create the respective [runtime classes][runtime-class]. To add them run the following:

```
$ k3s kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/runtimeclasses/kata-runtimeClasses.yaml
runtimeclass.node.k8s.io/kata-qemu-virtiofs created
runtimeclass.node.k8s.io/kata-qemu created
runtimeclass.node.k8s.io/kata-clh created
runtimeclass.node.k8s.io/kata-fc created
```

Query the system to see if kata-deploy was successful:

```
$ k3s kubectl get nodes --show-labels
NAME                     STATUS   ROLES                  AGE   VERSION        LABELS
mycluster.localdomain   Ready    control-plane,master   62m   v1.21.3+k3s1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=mycluster.localdomain,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=true,node-role.kubernetes.io/master=true,node.kubernetes.io/instance-type=k3s,katacontainers.io/kata-runtime=true
```

See the last label? its there because kata containers installation was successful! So we're ready to spawn a sandboxed container on k3s:

```
$ k3s kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/examples/test-deploy-kata-fc.yaml
deployment.apps/php-apache-kata-fc created
service/php-apache-kata-fc created
```

Lets see what has been created:

```
$ k3s kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
php-apache-kata-fc-5ccb8df89-v8hqj   1/1     Running   0          31s
```

```
$ ps -ef |grep firecracker
root      128201  128187  3 15:21 ?        00:00:02 /opt/kata/bin/firecracker --api-sock /run/vc/firecracker/2050be93ed33df24537f6f8274e3c66c/root/run/firecracker.socket --config-file /run/vc/firecracker/2050be93ed33df24537f6f8274e3c66c/root/fcConfig.json
```

```
$ k3s kubectl describe pod php-apache-kata-fc-5ccb8df89-v8hqj
Name:         php-apache-kata-fc-5ccb8df89-v8hqj
Namespace:    default
Priority:     0
Node:         mycluster.localdomain
Labels:       pod-template-hash=5ccb8df89
              run=php-apache-kata-fc
Annotations:  <none>
Status:       Running
IP:           10.42.0.192
IPs:
  IP:           10.42.0.192
Controlled By:  ReplicaSet/php-apache-kata-fc-5ccb8df89
Containers:
  php-apache:
    Container ID:   containerd://13718fd3820efb1c0260ec9d88c039c37325a1e76a3edca943c62e6dab02c549
    Image:          k8s.gcr.io/hpa-example
    Image ID:       k8s.gcr.io/hpa-example@sha256:581697a37f0e136db86d6b30392f0db40ce99c8248a7044c770012f4e8491544
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:        200m
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-k62m5 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-k62m5:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              katacontainers.io/kata-runtime=true
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  82s   default-scheduler  Successfully assigned default/php-apache-kata-fc-5ccb8df89-v8hqj to mycluster.localdomain
  Normal  Pulling    80s   kubelet            Pulling image "k8s.gcr.io/hpa-example"
  Normal  Pulled     80s   kubelet            Successfully pulled image "k8s.gcr.io/hpa-example" in 745.344896ms
  Normal  Created    80s   kubelet            Created container php-apache
  Normal  Started    79s   kubelet            Started container php-apache
```

To tear things down on k3s run the following:

```
$ k3s kubectl delete -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/examples/test-deploy-kata-fc.yaml
deployment.apps "php-apache-kata-fc" deleted
service "php-apache-kata-fc" deleted
$ k3s kubectl delete -f  https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/runtimeclasses/kata-runtimeClasses.yaml
runtimeclass.node.k8s.io "kata-qemu-virtiofs" deleted
runtimeclass.node.k8s.io "kata-qemu" deleted
runtimeclass.node.k8s.io "kata-clh" deleted
runtimeclass.node.k8s.io "kata-fc" deleted
$ k3s kubectl delete -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-deploy/base/kata-deploy.yaml
daemonset.apps "kata-deploy" deleted
$ k3s kubectl delete -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-rbac/base/kata-rbac.yaml
serviceaccount "kata-label-node" deleted
clusterrole.rbac.authorization.k8s.io "node-labeler" deleted
clusterrolebinding.rbac.authorization.k8s.io "kata-label-node-rb" deleted
```

This concludes a first take on running containers as microVMs in k8s using
Kata Containers and AWS Firecracker. In the next posts we will explore hardware
acceleration options, serverless frameworks and unikernel execution! Stay tuned
for more!

NOTE I: *If you choose the first option for k3s, you'll need to manually
append the handlers in `/etc/containerd/config.toml` as the k3s denotes a
relative path for containerd's config file, so it resides in
`/var/lib/rancher/k3s/agent/etc/containerd/config.toml`. The easiest way to
work around this is*: 
```
cat /var/lib/rancher/k3s/agent/etc/containerd/config.toml >> /etc/containerd/config.toml
systemctl restart containerd
```

NOTE II: *If you are already running k8s/k3s with a different containerd
snapshotter, after configuring devmapper and restarting containerd, you
probably need to restart kubelet/k3s services and make sure that all the
containers running in `k8s.io` containerd namespace are now running using
devicemapper.*


[kata]: https://katacontainers.io
[k8s]: https://kubernetes.io/
[clear]: https://software.intel.com/content/www/us/en/develop/articles/intel-clear-containers-1-the-container-landscape.html
[runv]: https://github.com/hyperhq/runv
[k8s-docs]: https://kubernetes.io/docs/home/
[k3s]: https://k3s.io/
[kata-doc]: https://github.com/kata-containers/kata-containers/tree/main/docs/install#readme
[kata-k8s-doc]: https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/run-kata-with-k8s.md
[kata-containerd-doc]: https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/containerd-kata.md
[kata-containerd-examples]: https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/containerd-kata.md#run
[fc-virtiofs]: https://github.com/firecracker-microvm/firecracker/pull/1351
[devmapper-doc]: https://pkg.go.dev/github.com/containerd/containerd/snapshots/devmapper
[kata-deploy]: https://github.com/kata-containers/kata-containers/tree/main/tools/packaging/kata-deploy
[runtime-class]: https://kubernetes.io/docs/concepts/containers/runtime-class/#runtime-class


