---
title: "Playing with a Raspberry Pi 4 64-bit"
date: 2019-07-10T14:41:44+02:00
lastmod: 2019-10-17T19:38:21+0200
---

Lightweight virtualization is a natural fit for low power devices and, so,
seeing that the extremely popular Raspberry Pi line got an upgrade, we were
very keen on trying the newly released Raspberry Pi 4 model B.

Getting the board up and running with a 64bit kernel (and a 64bit userland)
proved to be kind of a challenge, given that currently there is a number of
[limitations][0] (SD card not fully working for > 1GB RAM, coherent memory
allocations etc.).
With the help of some [very][1] useful [posts][2] we were able to successfully
boot a 64bit kernel with KVM support. Please note we also had to enable KVM &
VIRTIO options in the kernel config to support QEMU/KVM & solo5-hvt instances.

For now, we can safely boot it with 1GB of RAM and play with our various
configurations.

What's interesting about the RPI4, as far as virtualization is concerned, is
that it now comes with ARM's standard GIC (Generic Interrupt Controller). Thus,
unlike previous RPI generations that were wired with custom interrupt handling,
virtualization is supported natively without any patching needed for interrupt
emulation (and the overhead that this can incur...).

To get a first taste of the performance of different virtualization options we
ran a simple set of benchmarks for common use-case scenarios: a simple
key-value store (REDIS), and a popular web server (NGINX). We used the stock
tool for redis (redis-benchmark) and the apache-benchmark (ab), both running on
the host OS. 

The figures below show measurements from the following configurations: native
(bare-metal execution on the linux host), nabla (or solo5-spt, runnc with SPT
backend), solo5-hvt (runnc with HVT backend), and QEMU/KVM (generic linux
guest, running Debian Buster with vhost enabled).

First we ran the full redis-benchmark test on a RPi3 & a RPi4, for the various
configurations above. Results for the linux guest are shown below:

{{< figure src="/static/redis-linux-guest.png" >}}

Clearly, apart from the A72 vs A53 upgrade, the GIC addition boosted the
guests' performance significantly. 

To dig into the virtualization internals we tried running a rumprun unikernel
on top of solo5-hvt & solo5-spt (nabla). To facilitate deployment we used a
[modified][3] version of [runnc][4] that includes a solo5-hvt backend and the
ability to "mount" host fs files/directories in the container.

{{< figure src="/static/redis-unikernel-containers.png" >}}

The performance is much better than in the Linux guest. To further investigate the overhead of running virtualized workloads on such SoCs, we examined the SET case. The figure below shows results from native, nabla, solo5-hvt and qemu/KVM cases.

{{< figure src="/static/redis-set.png" >}}

[Nabla containers][5] exhibit the lowest overhead of all cases (~20%): this is
expected as there is no virtualization involved, just seccomp filtering. The
source of the overhead is the minimal I/O interface that solo5-spt offers.
Solo5-hvt exhibits significant overhead (~40%), mainly due to the unoptimized
I/O interface. In addition to that, trapping into user-space to handle I/O
requests could possible be the main source of overhead. Qemu/KVM runs at 41% of
the boards capabilities, albeit the use of the vhost mechanism which removes
unnecessary traps to user-space for I/O handling.

Finally, we plot the apache-benchmark results we got from an NGINX web server,
running on a linux guest and a solo5-hvt unikernel.

{{< figure src="/static/nginx-ab.png" >}}

We had fun playing with a RPi4 and its systems support! Clearly there is a need
to study the systems stack and optimize the various missing pieces to be able
to reduce or even eliminate the overheads for simple, lightweight workload
execution. 

Evaluating virtualization options in the diverse low power SoC ecosystem is a
non-trivial task; the popularity and the evolvement of such systems makes them
very interesting to explore, especially when it comes to identifying the
bottlenecks and providing optimizations that help the community design and
implement energy-efficient, low-footprint, and high-performance solutions.

If you would like to try it out on your RPi 4, check out our custom-made [RPi4
image][6] (sha1sum: 1b96a6be5256182eaceb5894fb993c8ffce8c2a2), based on [ubuntu
18.04.2][7], with KVM, docker & nabla container runtimes pre-installed!

**UPDATE**

Actually, links to the older builds were removed, so here's the updated links for the stock [18.04.3 image][8], and our KVM-enabled ubuntu [18.04.3 image][9].

Also, many people have reached out to us about the username and password for
our custom image. The issue is that we assumed people would have a TTL2USB
cable to play with, so the username is root, without any password. That (of
course) doesn't work if you try to SSH to the RPi.

[0]: https://github.com/raspberrypi/linux/commit/cdb78ce891f6c6367a69c0a46b5779a58164bd4b#diff-634f284364ba43ef69912111615b08ef
[1]: https://andrei.gherzan.ro/linux/raspbian-rpi-64/
[2]: https://andrei.gherzan.ro/linux/raspbian-rpi4-64/
[3]: https://github.com/cloudkernels/runnc/tree/hvt%2Bvolumes
[4]: https://github.com/nabla-containers/runnc
[5]: https://nabla-containers.github.io/
[6]: https://cloudkernels.net/rpi4-64-bit-kvm-docker.img.xz
[7]: http://cdimage.ubuntu.com/ubuntu/releases/18.04.2/release/ubuntu-18.04.2-preinstalled-server-arm64+raspi3.img.xz 
[8]: http://cdimage.ubuntu.com/ubuntu/releases/18.04.3/release/ubuntu-18.04.3-preinstalled-server-arm64+raspi3.img.xz 
[9]: https://cloudkernels.net/ubuntu-18.04.3-preinstalled-server-arm64+raspi4+kvm.img.xz



