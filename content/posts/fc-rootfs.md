---
title: "Build a single-app rootfs for Firecracker MicroVMs"
date: 2019-10-21T10:02:51+01:00
---

Spawning applications in the cloud has been made super easy using container frameworks such as docker. For instance running a simple command like the following

```bash
docker run --rm -v /path/to/nginx-files:/etc/nginx nginx
```

spawns an NGINX web server, provided you customize config files and the actual HTML files to be served.

This process, inherits NGINX's stock docker hub rootfs, and spawns it as a docker container in a generic Linux container host.

However, what happens when someone wants to create the absolute mimimum rootfs for such a process? 

In what follows we describe two straightforward approaches that came up handy when playing with firecraker and qemu rootfs images on low-power devices, where resources are scarse (memory footprint, storage footprint, bw etc.).

#### Option 1: base the rootfs on a minimal linux container distro

The firecracker guide provides a guide on how to build a rootfs based on Alpine
Linux. We will re-use the same steps for building our rootfs, but to spice it up
we will create a rootfs that upon boot runs immediately nginx.

We will use the following Dockerfile to build nginx:

```docker
FROM alpine:latest

MAINTAINER Babis Chalios <mail@bchalios.io>

ENV NGINX_VERSION nginx-1.17.4

WORKDIR /build
RUN apk --update add openssl-dev pcre-dev zlib-dev wget build-base && \
    wget http://nginx.org/download/${NGINX_VERSION}.tar.gz && \
    tar -zxvf ${NGINX_VERSION}.tar.gz && \
    cd ${NGINX_VERSION} && \
    ./configure \
        --with-http_ssl_module \
        --with-http_gzip_static_module \
        --prefix=/etc/nginx \
        --http-log-path=/var/log/nginx/access.log \
        --error-log-path=/var/log/nginx/error.log \
        --sbin-path=/usr/local/sbin/nginx && \
    make && \
    make install && \
    apk del build-base && \
    rm -rf /tmp/src && \
    rm -rf /var/cache/apk/*

WORKDIR /
```

We build nginx:

```bash
$ docker build -t build_static_nginx
```

and use the container to build our rootfs:

```bash
$ dd if=/dev/zero of=nginx_fc.ext4 bs=1M count=100
$ mkfs.ext4 nginx_fc.ext4
$ mkdir mnt
$ sudo mount nginx_fc.ext4 mnt
$ docker --rm -ti -v $(pwd)/mnt:/my-rootfs build_static_nginx
```

and in the container:

```
# for d in bin etc lib root sbin usr; do tar c "/$d" | tar x -C /my-rootfs; done
# for dir in dev proc run sys var; do mkdir /my-rootfs/${dir}; done
# exit
```

As the final step, we will substitute the `/sbin/init` binary of our rootfs with
the following script which launches the nginx binary:

```bash
$ cat nginx_init
#!/bin/sh

mkdir -p /var/log/nginx

# the init process should never exit
/usr/local/sbin/nginx -g "daemon off;"
$ sudo mv mnt/sbin/init mnt/sbin/init.old
$ sudo cp nginx_init mnt/sbin/init
$ sudo chmod a+x mnt/sbin/init
$ sudo umount mnt
```


#### Option 2: ditch the rootfs, replace /sbin/init with the application

Well, this is more of a hack than an actual solution, but comes in handy when memory and storage space is critical.

Based on an [excellent tutorial][1] by Rob Landley, we come up with a single kernel image that spawns an NGINX web server.

##### Step 1: Build the application as a static binary

There are a couple of tutorials on how to build NGINX statically. We followed [this one][2]. After successful compilation, we come up with the following files/dirs:

```
objs/nginx
conf/
html/
```

##### Step 2: generate the list of files/dirs for initramfs

To embed all files in the Linux kernel build, we use the initramfs option in
the kernel config, where the kernel build process generates the cpio archive to
be appended to the kernel binary, and used as initramfs.

First step is to create a file that contains the list of all files to be included in the cpio archive:

```bash
$ cat >> initramfs_list < EOF
dir /dev 755 0 0
nod /dev/console 644 0 0 c 5 1
nod /dev/loop0 644 0 0 b 7 0
dir /bin 755 1000 1000
dir /proc 755 0 0
dir /sys 755 0 0
dir /mnt 755 0 0
file /init usr/nginx 755 0 0
dir /opt 755 0 0
dir /opt/nginx 755 0 0
dir /opt/nginx/logs 755 0 0
dir /opt/nginx/conf 755 0 0
dir /opt/nginx/html 755 0 0
file /opt/nginx/conf/fastcgi.conf usr/conf/fastcgi.conf 644 0 0
file /opt/nginx/conf/fastcgi_params usr/conf/fastcgi_params 644 0 0
file /opt/nginx/conf/koi-utf usr/conf/koi-utf 644 0 0
file /opt/nginx/conf/koi-win usr/conf/koi-win 644 0 0
file /opt/nginx/conf/mime.types usr/conf/mime.types 644 0 0
file /opt/nginx/conf/nginx.conf usr/conf/nginx.conf 644 0 0
file /opt/nginx/conf/scgi_params usr/conf/scgi_params 644 0 0
file /opt/nginx/conf/uwsgi_params usr/conf/uwsgi_params 644 0 0
file /opt/nginx/conf/win-utf usr/conf/win-utf 644 0 0
file /opt/nginx/html/index.html usr/html/index.html 644 0 0
EOF
```

We then place all necessary files at the `usr/` dir in the linux kernel build dir.

##### Step 3: build the kernel

```bash
$ git clone https://github.com/torvalds/linux.git
$ cd linux
$ git checkout v4.14
$ wget https://raw.githubusercontent.com/firecracker-microvm/firecracker/b1e48beaea7b917fef1e4846f1d75a2c1a136517/resources/microvm-kernel-aarch64-config -O .config
$ make oldconfig
```
In this step, we can select any kernel version we want, starting from version
4.14, e.g. `git checkout v4.20`.

The only thing left is to add the option `CONFIG_INITRAMFS_SOURCE="usr/initramfs_list"` to the kernel config file, and point it to where our `initramfs_list` file is. A simple `make` will build the necessary stuff and we end up with `arch/arm64/boot/Image` which contains the Linux kernel, as well as this initramfs.cpio.gz archive with the files we already specified.

and voila! Here's a dump of this `Image` booting on firecracker on a Rpi4:

```bash
# ./build/cargo_target/aarch64-unknown-linux-musl/release/firecracker --api-sock /tmp/fireananos.sock 
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd083]
[    0.000000] Linux version 5.4.0-rc3 (root@baremetal.nubificus.com) (gcc version 6.3.0 20170516 (Debian 6.3.0-18)) #2 SMP PREEMPT Sat Oct 19 06:13:48 CDT 2019
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] PCI: Unknown option `off'
[    0.000000] earlycon: uart0 at MMIO 0x0000000040003000 (options '')
[    0.000000] printk: bootconsole [uart0] enabled
[    0.000000] efi: Getting EFI parameters from FDT:
[    0.000000] efi: UEFI not found.
[    0.000000] cma: Reserved 16 MiB at 0x0000000086c00000
[    0.000000] NUMA: No NUMA configuration found
[    0.000000] NUMA: Faking a node at [mem 0x0000000080000000-0x0000000087ffffff]
[    0.000000] NUMA: NODE_DATA [mem 0x87fb9800-0x87fbafff]
[    0.000000] Zone ranges:
[    0.000000]   DMA32    [mem 0x0000000080000000-0x0000000087ffffff]
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000080000000-0x0000000087ffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000080000000-0x0000000087ffffff]
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv1.0 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: Trusted OS migration not required
[    0.000000] psci: SMC Calling Convention v1.1
[    0.000000] percpu: Embedded 22 pages/cpu s52760 r8192 d29160 u90112
[    0.000000] Detected PIPT I-cache on CPU0
[    0.000000] CPU features: detected: EL2 vector hardening
[    0.000000] ARM_SMCCC_ARCH_WORKAROUND_1 missing from firmware
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 32256
[    0.000000] Policy zone: DMA32
[    0.000000] Kernel command line: console=ttyS0 root=/dev/vda reboot=k panic=1 pci=off ip=10.0.0.2::10.0.0.1:255.255.255.0::eth0:off net.ifnames=0 root=/dev/vda rw earlycon=uart,mmio,0x40003000
[    0.000000] Dentry cache hash table entries: 16384 (order: 5, 131072 bytes, linear)
[    0.000000] Inode-cache hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 72448K/131072K available (11004K kernel code, 1588K rwdata, 5552K rodata, 6656K init, 381K bss, 42240K reserved, 16384K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=64 to nr_cpu_ids=1.
[    0.000000] 	Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=1
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] random: get_random_bytes called from start_kernel+0x2bc/0x45c with crng_init=0
[    0.000000] arch_timer: cp15 timer(s) running at 54.00MHz (virt).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0xc743ce346, max_idle_ns: 440795203123 ns
[    0.000004] sched_clock: 56 bits at 54MHz, resolution 18ns, wraps every 4398046511102ns
[    0.003165] Console: colour dummy device 80x25
[    0.004704] Calibrating delay loop (skipped), value calculated using timer frequency.. 108.00 BogoMIPS (lpj=216000)
[    0.008179] pid_max: default: 32768 minimum: 301
[    0.009775] LSM: Security Framework initializing
[    0.011308] Mount-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.013772] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.042261] ASID allocator initialised with 32768 entries
[    0.055248] rcu: Hierarchical SRCU implementation.
[    0.068193] EFI services will not be available.
[    0.080179] smp: Bringing up secondary CPUs ...
[    0.083868] smp: Brought up 1 node, 1 CPU
[    0.087308] SMP: Total of 1 processors activated.
[    0.091120] CPU features: detected: 32-bit EL0 Support
[    0.093094] CPU features: detected: CRC32 instructions
[    0.095989] CPU: All CPU(s) started at EL1
[    0.097545] alternatives: patching kernel code
[    0.099975] devtmpfs: initialized
[    0.106349] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.109513] futex hash table entries: 256 (order: 2, 16384 bytes, linear)
[    0.112244] pinctrl core: initialized pinctrl subsystem
[    0.115499] DMI not present or invalid.
[    0.121449] NET: Registered protocol family 16
[    0.125835] DMA: preallocated 256 KiB pool for atomic allocations
[    0.128200] audit: initializing netlink subsys (disabled)
[    0.133094] cpuidle: using governor menu
[    0.134911] audit: type=2000 audit(0.116:1): state=initialized audit_enabled=0 res=1
[    0.138336] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    0.143418] Serial: AMBA PL011 UART driver
[    0.190417] HugeTLB registered 1.00 GiB page size, pre-allocated 0 pages
[    0.192858] HugeTLB registered 32.0 MiB page size, pre-allocated 0 pages
[    0.196031] HugeTLB registered 2.00 MiB page size, pre-allocated 0 pages
[    0.198512] HugeTLB registered 64.0 KiB page size, pre-allocated 0 pages
[    0.205111] cryptd: max_cpu_qlen set to 1000
[    0.217815] ACPI: Interpreter disabled.
[    0.224317] iommu: Default domain type: Translated 
[    0.227342] vgaarb: loaded
[    0.228696] SCSI subsystem initialized
[    0.231005] usbcore: registered new interface driver usbfs
[    0.233680] usbcore: registered new interface driver hub
[    0.236052] usbcore: registered new device driver usb
[    0.238306] pps_core: LinuxPPS API ver. 1 registered
[    0.240438] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.243424] PTP clock support registered
[    0.245042] EDAC MC: Ver: 3.0.0
[    0.247254] Advanced Linux Sound Architecture Driver Initialized.
[    0.250766] clocksource: Switched to clocksource arch_sys_counter
[    0.253053] VFS: Disk quotas dquot_6.6.0
[    0.254530] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    0.257177] pnp: PnP ACPI: disabled
[    0.268272] thermal_sys: Registered thermal governor 'step_wise'
[    0.268275] thermal_sys: Registered thermal governor 'power_allocator'
[    0.271870] NET: Registered protocol family 2
[    0.276694] tcp_listen_portaddr_hash hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.279823] TCP established hash table entries: 1024 (order: 1, 8192 bytes, linear)
[    0.282285] TCP bind hash table entries: 1024 (order: 2, 16384 bytes, linear)
[    0.284760] TCP: Hash tables configured (established 1024 bind 1024)
[    0.287067] UDP hash table entries: 256 (order: 1, 8192 bytes, linear)
[    0.289706] UDP-Lite hash table entries: 256 (order: 1, 8192 bytes, linear)
[    0.292468] NET: Registered protocol family 1
[    0.307079] RPC: Registered named UNIX socket transport module.
[    0.313313] RPC: Registered udp transport module.
[    0.315397] RPC: Registered tcp transport module.
[    0.317154] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.319655] PCI: CLS 0 bytes, default 64
[    0.504111] kvm [1]: HYP mode not available
[    0.507069] Initialise system trusted keyrings
[    0.510147] workingset: timestamp_bits=44 max_order=15 bucket_order=0
[    0.526272] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    0.530011] NFS: Registering the id_resolver key type
[    0.532603] Key type id_resolver registered
[    0.534191] Key type id_legacy registered
[    0.535985] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    0.538740] 9p: Installing v9fs 9p2000 file system support
[    0.550468] Key type asymmetric registered
[    0.552206] Asymmetric key parser 'x509' registered
[    0.554107] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 246)
[    0.557229] io scheduler mq-deadline registered
[    0.558966] io scheduler kyber registered
[    0.574957] EINJ: ACPI disabled.
[    0.589178] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
[    0.596662] printk: console [ttyS0] disabled
[    0.599695] 40003000.uart: ttyS0 at MMIO 0x40003000 (irq = 7, base_baud = 1500000) is a 16550A
[    0.603515] printk: console [ttyS0] enabled
[    0.603515] printk: console [ttyS0] enabled
[    0.607076] printk: bootconsole [uart0] disabled
[    0.607076] printk: bootconsole [uart0] disabled
[    0.611276] SuperH (H)SCI(F) driver initialized
[    0.613660] msm_serial: driver initialized
[    0.615984] cacheinfo: Unable to detect cache hierarchy for CPU 0
[    0.631690] loop: module loaded
[    0.634019] virtio_blk virtio0: [vda] 204800 512-byte logical blocks (105 MB/100 MiB)
[    0.648631] libphy: Fixed MDIO Bus: probed
[    0.652999] tun: Universal TUN/TAP device driver, 1.6
[    0.658605] thunder_xcv, ver 1.0
[    0.660878] thunder_bgx, ver 1.0
[    0.662279] nicpf, ver 1.0
[    0.664058] e1000e: Intel(R) PRO/1000 Network Driver - 3.2.6-k
[    0.666284] e1000e: Copyright(c) 1999 - 2015 Intel Corporation.
[    0.669114] igb: Intel(R) Gigabit Ethernet Network Driver - version 5.6.0-k
[    0.671961] igb: Copyright (c) 2007-2014 Intel Corporation.
[    0.674176] igbvf: Intel(R) Gigabit Virtual Function Network Driver - version 2.4.0-k
[    0.677465] igbvf: Copyright (c) 2009 - 2012 Intel Corporation.
[    0.680142] sky2: driver version 1.30
[    0.682282] VFIO - User Level meta-driver version: 0.3
[    0.690342] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    0.693832] ehci-pci: EHCI PCI platform driver
[    0.695936] ehci-platform: EHCI generic platform driver
[    0.698111] ehci-orion: EHCI orion driver
[    0.700178] ehci-exynos: EHCI EXYNOS driver
[    0.701992] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    0.704917] ohci-pci: OHCI PCI platform driver
[    0.706835] ohci-platform: OHCI generic platform driver
[    0.709036] ohci-exynos: OHCI EXYNOS driver
[    0.711184] usbcore: registered new interface driver usb-storage
[    0.716697] rtc-pl031 40004000.rtc: registered as rtc0
[    0.720525] i2c /dev entries driver
[    0.725890] sdhci: Secure Digital Host Controller Interface driver
[    0.729845] sdhci: Copyright(c) Pierre Ossman
[    0.732261] Synopsys Designware Multimedia Card Interface Driver
[    0.735862] sdhci-pltfm: SDHCI platform and OF driver helper
[    0.739259] ledtrig-cpu: registered to indicate activity on CPUs
[    0.743495] usbcore: registered new interface driver usbhid
[    0.745688] usbhid: USB HID core driver
[    0.751079] NET: Registered protocol family 17
[    0.754152] 9pnet: Installing 9P2000 support
[    0.756397] Key type dns_resolver registered
[    0.758475] registered taskstats version 1
[    0.760525] Loading compiled-in X.509 certificates
[    0.763104] rtc-pl031 40004000.rtc: setting system clock to 2019-10-19T14:46:13 UTC (1571496373)
[    0.786860] IP-Config: Complete:
[    0.790000]      device=eth0, hwaddr=c2:6b:5f:69:cc:12, ipaddr=10.0.0.2, mask=255.255.255.0, gw=10.0.0.1
[    0.800389]      host=10.0.0.2, domain=, nis-domain=(none)
[    0.802853]      bootserver=255.255.255.255, rootserver=255.255.255.255, rootpath=
[    0.806010] ALSA device list:
[    0.807295]   No soundcards found.
[    0.816070] Freeing unused kernel memory: 6656K
[    0.817939] Run /init as init process
``` 

and from a GET / command:

```html
curl 172.16.0.42
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```



[1]: http://landley.net/writing/rootfs-howto.html
[2]: https://gist.github.com/rjeczalik/7057434
