---
title: "Run a rumprun unikernel on a RPi3"
date: 2019-01-24T00:09:27+02:00
lastmod: 2019-02-22T19:37:57+0200

---

\[**UPDATE:** Revise instructions to reflect upstream nabla changes.\]
<br>
<br>

In this post, we will walk through the steps of compiling, baking, and running an
application as a rumprun unikernel on a Rasrberry Pi 3.

In our [previous post]({{< relref "nabla-containers-aarch64.md" >}}), we
provided some background for [Rumprun][1]/[rump kernels][2] and
[Solo5][5]. In short, Rumprun provides the necessary components to run a POSIX
compatible application as a unikernel. Solo5 is, essentially, a hardware
abstraction layer that provides a very thin interface, or else a *minimal
attack surface*.

![Rumprun High-level stack](/static/rumprun-solo5-high.png)

The [Nabla Containers][4] fork of Rumprun provides a solo5 target for the rump
kernel unikernel stack. Additionally, they provide a Docker runtime that spawns
unikernel images as containers. We will be talking about Nabla Containers in
more detail in future posts. Stay tuned :)

For this tutorial, we will use our current [aarch64 port]({{< relref
"nabla-containers-aarch64.md" >}}). We will show you how to build and run a
unikernel application on a RPi3. Keep in mind, that our port should work
without issues on any aarch64 platform. Feel free to run this tutorial on your
favourite ARM board and let us know about your experience!

However, enough with the boring theory, let's get our hands dirty.

### Hands-on ###

First of all, we need the rumprun toolchain to bake our unikernel image. One
option is to  use our [docker image][8] with the
toolchain pre-installed. 

Another option is to build the rumprun toolchain from source. We will provide
the necessary information on how to build all components for aarch64 in the
coming days.

So, log in to your RPi3 and, after you've [installed docker-ce][9], try running
the following command:

```
docker run -ti --rm -v /build:/build cloudkernels/debian-rumprun-build:aarch64 /bin/bash
```

you should be presented with something like the following:


```
Unable to find image 'cloudkernels/debian-rumprun-build:aarch64' locally
aarch64: Pulling from cloudkernels/debian-rumprun-build
e10919c546c2: Pull complete 
6b3f0a4d7b10: Pull complete 
473e207e8cf0: Pull complete 
0deecc1ceca2: Pull complete 
628025a81431: Pull complete 
25fd95c63d4f: Pull complete 
Digest: sha256:0221ba1c3a120bde1fd83b6d9c267fb4379c33d8f0012b9a64826afd476faf72
Status: Downloaded newer image for cloudkernels/debian-rumprun-build:aarch64
root@184fa9ecd15d:/#
```

now move to the build directory, and clone the rumprun-packages repo:

```
root@184fa9ecd15d:/# cd build
root@184fa9ecd15d:/build# git clone https://github.com/cloudkernels/rumprun-packages
```

Make sure your config.mk file contains the correct toolchain tuple:

```
root@184fa9ecd15d:/build# cd rumprun-packages
root@184fa9ecd15d:/build/rumprun-packages# echo RUMPRUN_TOOLCHAIN_TUPLE=aarch64-rumprun-netbsd > config.mk
``` 

and go to an example package, say hello, and type make

```
root@184fa9ecd15d:/build/rumprun-packages# cd hello
root@184fa9ecd15d:/build/rumprun-packages# make
```

The output should be the following:

```
# make
mkdir -p build
cp src/* build
make -C build hello.spt
make[1]: Entering directory '/build/rumprun-packages/hello/build'
aarch64-rumprun-netbsd-gcc hello.c -o hello-rumprun
rumprun-bake solo5_spt hello.spt hello-rumprun

!!!
!!! NOTE: rumprun-bake is experimental. syntax may change in the future
!!!

make[1]: Leaving directory '/build/rumprun-packages/hello/build'
mkdir -p bin
cp build/hello.spt bin/hello.spt
rumprun-bake solo5_hvt hello.hvt hello-rumprun

!!!
!!! NOTE: rumprun-bake is experimental. syntax may change in the future
!!!

make[1]: Leaving directory '/build/rumprun-packages/hello/build'
mkdir -p bin
cp build/hello.hvt bin/hello.hvt

```

Now, exit the container (Ctrl-D) and cd into this directory:

```
root@rpi3:~# cd /build/rumprun-packages/hello
root@rpi3:/build/rumprun-packages/hello# 
```

Make sure there's a dummy file for the disk image, and a tap interface:

```
root@rpi3:/build/rumprun-packages/hello# dd if=/dev/zero of=dummy count=1 bs=512
root@rpi3:/build/rumprun-packages/hello# ip tuntap add tap0 mode tap
root@rpi3:/build/rumprun-packages/hello# ip link set dev tap0 up
```

And make sure you've got the solo5-hvt/spt binaries (solo5). If not, do the following:

```
root@rpi3:/build# git clone https://github.com/solo5/solo5
root@rpi3:/build# cd solo5
root@rpi3:/build/solo5# apt-get install libseccomp-dev && make
```

If everything was successful, you should have two binaries: solo5-spt (seccomp-tender) and solo5-hvt (kvm tender) at:

```
/build/solo5/tenders/spt/solo5-spt
/build/solo5/tenders/hvt/solo5-hvt
```

So, returning to the previous dir/evn, you can execute the hello unikernel:

```
root@rpi3:/build/rumprun-packages/hello# /build/solo5/tenders/spt/solo5-spt --mem=32 --net=tap0 --disk=dummy ./bin/hello.spt
            |      ___|
  __|  _ \  |  _ \ __ \
\__ \ (   | | (   |  ) |
____/\___/ _|\___/____/
Solo5: Memory map: 32 MB addressable:
Solo5:     unused @ (0x0 - 0xfffff)
Solo5:       text @ (0x100000 - 0x30dfff)
Solo5:     rodata @ (0x30e000 - 0x356fff)
Solo5:       data @ (0x357000 - 0x3c3fff)
Solo5:       heap >= 0x3c4000 < stack < 0x2000000
rump kernel bare metal bootstrap

[   1.0000000] Copyright (c) 1996, 1997, 1998, 1999, 2000, 2001, 2002, 2003, 2004, 2005,
[   1.0000000]     2006, 2007, 2008, 2009, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017,
[   1.0000000]     2018 The NetBSD Foundation, Inc.  All rights reserved.
[   1.0000000] Copyright (c) 1982, 1986, 1989, 1991, 1993
[   1.0000000]     The Regents of the University of California.  All rights reserved.

[   1.0000000] NetBSD 8.99.25 (RUMP-ROAST)
[   1.0000000] total memory = 14328 KB
[   1.0000000] timecounter: Timecounters tick every 10.000 msec
[   1.0000080] timecounter: Timecounter "clockinterrupt" frequency 100 Hz quality 0
[   1.0000090] cpu0 at thinair0: rump virtual cpu
[   1.0000090] root file system type: rumpfs
[   1.0000090] kern.module.path=/stand/evbarm/8.99.25/modules
[   1.0200090] mainbus0 (root)
[   1.0200090] timecounter: Timecounter "bmktc" frequency 1000000000 Hz quality 100
rumprun: could not find start of json.  no config?
mounted tmpfs on /tmp

=== calling "rumprun" main() ===

This is the Rumprun Hello World ...
... using the Solo5 framework ...
... in a Nabla container via runnc!

=== main() of "rumprun" returned 0 ===

=== _exit(0) called ===
[   3.0285155] rump kernel halting...
[   3.0285155] syncing disks... done
[   3.0285155] unmounting file systems...
[   3.0412055] unmounted tmpfs on /tmp type tmpfs
[   3.0412055] unmounted rumpfs on / type rumpfs
[   3.0412055] unmounting done
halted
Solo5: solo5_exit(0) called
```

In case you want to run the non-seccomp tender (KVM_RUN), then all you need to
do is bake the hello-rumprun binary with solo5_hvt. Return to the docker env
and bake the file accordingly:

```
docker run -ti --rm -v /build:/build cloudkernels/debian-rumprun-build:aarch64 /bin/bash
root@184fa9ecd15d:/# cd /build/rumprun-packages/hello
root@184fa9ecd15d:/build/rumprun-packages/hello# rumprun-bake solo5-hvt ./bin/hello.hvt ./build/hello-rumprun
```

and now, exit the docker env (Ctrl-D) and run the hello.hvt unikernel image with the solo5-hvt binary:

```
root@rpi3:~# cd /build/rumprun-packages/hello
root@rpi3:~/build/rumprun-packages/hello# /build/solo5/tenders/hvt/solo5-hvt --mem=32 --net=tap0 --disk=dummy ./bin/hello.hvt
solo5-hvt: ./bin/hello.hvt: Warning: phdr[0] requests WRITE and EXEC permissions
            |      ___|
  __|  _ \  |  _ \ __ \
\__ \ (   | | (   |  ) |
____/\___/ _|\___/____/
Solo5: Memory map: 32 MB addressable:
Solo5:     unused @ (0x0 - 0xfffff)
Solo5:       text @ (0x100000 - 0x30ffff)
Solo5:     rodata @ (0x310000 - 0x359fff)
Solo5:       data @ (0x35a000 - 0x3c6fff)
Solo5:       heap >= 0x3c7000 < stack < 0x2000000
rump kernel bare metal bootstrap

[   1.0000000] Copyright (c) 1996, 1997, 1998, 1999, 2000, 2001, 2002, 2003, 2004, 2005,
[   1.0000000]     2006, 2007, 2008, 2009, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017,
[   1.0000000]     2018 The NetBSD Foundation, Inc.  All rights reserved.
[   1.0000000] Copyright (c) 1982, 1986, 1989, 1991, 1993
[   1.0000000]     The Regents of the University of California.  All rights reserved.

[   1.0000000] NetBSD 8.99.25 (RUMP-ROAST)
[   1.0000000] total memory = 14322 KB
[   1.0000000] timecounter: Timecounters tick every 10.000 msec
[   1.0000080] timecounter: Timecounter "clockinterrupt" frequency 100 Hz quality 0
[   1.0000090] cpu0 at thinair0: rump virtual cpu
[   1.0000090] root file system type: rumpfs
[   1.0000090] kern.module.path=/stand/evbarm/8.99.25/modules
[   1.0200090] mainbus0 (root)
[   1.0200090] timecounter: Timecounter "bmktc" frequency 1000000000 Hz quality 100
rumprun: could not find start of json.  no config?
mounted tmpfs on /tmp

=== calling "rumprun" main() ===

This is the Rumprun Hello World ...
... using the Solo5 framework ...
... in a Nabla container via runnc!

=== main() of "rumprun" returned 0 ===

=== _exit(0) called ===
[   3.0408694] rump kernel halting...
[   3.0408694] syncing disks... done
[   3.0408694] unmounting file systems...
[   3.0548072] unmounted tmpfs on /tmp type tmpfs
[   3.0548072] unmounted rumpfs on / type rumpfs
[   3.0548072] unmounting done
halted
Solo5: solo5_exit(0) called
```

Now, to run an end-to-end example, take a look at [runnc][10], the
runtime container environment to spawn Nabla containers.

```
root@rpi3:/build# git clone https://github.com/cloudkernels/runnc
root@rpi3:/build# cd runnc
root@rpi3:/build/runnc# make container-build
root@rpi3:/build/runnc# make container-install
```

Now setup the daemon.json file and restart docker

```
root@rpi3:~# cat >> /etc/docker/daemon.json << EOF
{
    "runtimes": {
        "runnc": {
                "path": "/usr/local/bin/runnc"
        }
    }
}
EOF

root@rpi3:~# systemctl restart docker
```

And spawn an example nabla container via the docker cli:

```
root@rpi3:~# docker run --rm --runtime=runnc cloudkernels/hello-nabla:aarch64
Unable to find image 'cloudkernels/hello-nabla:aarch64' locally
aarch64: Pulling from cloudkernels/hello-nabla
271c6521e5a2: Pull complete 
Digest: sha256:309f8cd277a833958ec5055c0e7b0c7ca84c0b0413c691b8bd6392e30fbd9b71
Status: Downloaded newer image for cloudkernels/hello-nabla:aarch64
Running with args: [/opt/runnc/bin/runnc-cont -k8s -nabla-run /opt/runnc/bin/nabla-run -tap tap5cc88ff904a1 -cwd / -volume /var/run/docker/runtime-runnc/moby/5cc88ff904a1f97eaea9b0cb590575e57fce5227854ef77ecd[
            |      ___|
  __|  _ \  |  _ \ __ \
\__ \ (   | | (   |  ) |
____/\___/ _|\___/____/
Solo5: Memory map: 512 MB addressable:
Solo5:     unused @ (0x0 - 0xfffff)
Solo5:       text @ (0x100000 - 0x30dfff)
Solo5:     rodata @ (0x30e000 - 0x356fff)
Solo5:       data @ (0x357000 - 0x3c3fff)
Solo5:       heap >= 0x3c4000 < stack < 0x20000000
rump kernel bare metal bootstrap

[   1.0000000] Copyright (c) 1996, 1997, 1998, 1999, 2000, 2001, 2002, 2003, 2004, 2005,
[   1.0000000]     2006, 2007, 2008, 2009, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017,
[   1.0000000]     2018 The NetBSD Foundation, Inc.  All rights reserved.
[   1.0000000] Copyright (c) 1982, 1986, 1989, 1991, 1993
[   1.0000000]     The Regents of the University of California.  All rights reserved.

[   1.0000000] NetBSD 8.99.25 (RUMP-ROAST)
[   1.0000000] total memory = 253 MB
[   1.0000000] timecounter: Timecounters tick every 10.000 msec
[   1.0000080] timecounter: Timecounter "clockinterrupt" frequency 100 Hz quality 0
[   1.0000090] cpu0 at thinair0: rump virtual cpu
[   1.0000090] root file system type: rumpfs
[   1.0000090] kern.module.path=/stand/evbarm/8.99.25/modules
[   1.0200090] mainbus0 (root)
[   1.0200090] timecounter: Timecounter "bmktc" frequency 1000000000 Hz quality 100
[   1.0200090] ukvmif0: Ethernet address 02:42:ac:11:00:02
[   1.0695796] /dev//dev/ld0a: hostpath XENBLK_/dev/ld0a (19710 KB)
mounted tmpfs on /tmp

=== calling "/dev/shm/docker/overlay2/18f96a896573deb8de0883c46ea1b077be4dceeec10aaa00fb1d75565c642d47/merged/hello.nabla" main() ===

This is the Rumprun Hello World ...
... using the Solo5 framework ...
... in a Nabla container via runnc!

=== main() of "/dev/shm/docker/overlay2/18f96a896573deb8de0883c46ea1b077be4dceeec10aaa00fb1d75565c642d47/merged/hello.nabla" returned 0 ===

=== _exit(0) called ===
[   3.0895680] rump kernel halting...
[   3.0895680] syncing disks... done
[   3.0895680] unmounting file systems...
[   3.0895680] unmounted tmpfs on /tmp type tmpfs
[   3.0895680] unmounted /dev//dev/ld0a on / type cd9660
[   3.0895680] unmounted rumpfs on / type rumpfs
[   3.0895680] unmounting done
halted
Solo5: solo5_exit(0) called
```

Easy ? :D Let us know how it went!


[1]: https://github.com/rumpkernel/rumprun
[2]: http://rumpkernel.org
[3]: https://wiki.netbsd.org/projects/project/userland_pci
[4]: https://nabla-containers.github.io
[5]: https://github.com/Solo5/solo5
[8]: https://hub.docker.com/r/cloudkernels/debian-rumprun-build
[9]: https://docs.docker.com/install/linux/docker-ce/ubuntu
[10]: https://github.com/cloudkernels/runnc
