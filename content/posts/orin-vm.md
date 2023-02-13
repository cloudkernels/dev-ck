---
title: "Boot a VM on an NVIDIA Jetson AGX Orin"
date: 2023-02-13T10:14:04+01:00
---

In 2022, NVIDIA released the [Jetson
Orin](https://www.nvidia.com/en-il/autonomous-machines/embedded-systems/jetson-orin/)
modules, specifically designed for extreme computation at the Edge. The NVIDIA
Jetson AGX Orin modules deliver up to 275 TOPS of AI performance with power
configurable between 15W and 60W. 

{{< figure src="/static/nvidia-jetson-agx-orin.jpg#center"
caption="Figure 1: The NVIDIA Jetson AGX Orin devkit & module" width="80%">}}

These powerful machines, apart from bleeding edge GPUs, feature 12 ARMv8 cores
and come with 16GB, 32GB or 64GB of memory. The number of cores, combined with
these amounts of RAM appear ideal for some use-cases where multi-tenancy is
essential; making use of the virtualization extensions in these cores, we
enforce stronger isolation among the workloads running at the Edge. Moreover,
with the use of [kata-containers](https://katacontainers.io), we maintain the
cloud-native aspect of the application deployment at the Edge.

Following up on our work with [kata-containers](https://katacontainers.io) and
[vAccel](https://docs.vaccel.org), we got a couple of Orin nodes and started
experimenting. Unfortunately, things are a bit different than with the original
Jetson AGX Xavier boards we were used to. The SoC is slightly different, with a
fully-featured, [multi-core
GICv3](https://developer.nvidia.com/orin-series-soc-technical-reference-manual)
implementation.

That's great news! Right? well... not really, as the necessary control
structures for GICv3 are not created during boot, caused by an [incomplete
interrupts declaration in the device
tree](https://forums.developer.nvidia.com/t/fix-interrupts-declarations-for-gicv3/240566).

[Vadim](https://forums.developer.nvidia.com/u/vadim.likholetov/summary) &
[Alexey](https://forums.developer.nvidia.com/u/alexey13/summary) identified the
issue and provided the necessary patches to the device tree source files and ..
tada! we can boot a non-emulated VM, using GICv3 on Jetson Orin! 

### Walk through the issue

So, initially, we started to see if the stock kernel has KVM enabled:

```console
# dmesg |grep -i kvm
[    0.362967] kvm [1]: IPA Size Limit: 48 bits
[    0.363130] kvm [1]: VHE mode initialized successfully
```

Looks like there is support. So we moved on to try running an AWS Firecracker
VM. Got our `vmlinux`, `rootfs.img` and `config.json` built for our tests and
tried to boot the VM:

```sh
wget https://s3.nbfc.io/nbfc-assets/github/vaccelrt/vm-example/aarch64/rust-vmm/vmlinux
wget https://s3.nbfc.io/nbfc-assets/github/vaccelrt/vm-example/aarch64/rootfs.img
wget https://s3.nbfc.io/nbfc-assets/github/vaccelrt/vm-example/aarch64/fc/config_vsock.json
wget https://s3.nbfc.io/nbfc-assets/github/vaccelrt/vm-example/aarch64/fc/firecracker
```

```console
# ./firecracker --api-sock fc.sock --config-file config_vsock.json 
2023-02-13T18:54:29.061153660 [anonymous-instance:main:ERROR:src/firecracker/src/main.rs:480] Building VMM configured from cmdline json failed: Internal(Vm(VmCreateGIC(CreateGIC(Error(19)))))
```

We were a bit troubled as the exact same setup was working fine on a Jetson AGX
Xavier. We also tried
[QEMU](https://forums.developer.nvidia.com/t/jetson-agx-orin-devkit-34-1-1-gicv3-vgic-creation-failed/216192),
with the same results:


```console
# qemu-system-aarch64 -cpu max -machine virt,gic-version=3,kernel-irqchip=on -m 1024 -nographic -monitor none -kernel /boot/Image -enable-kvm
qemu-system-aarch64: gic-version=3 is not supported with kernel-irqchip=off
```

but if we disable KVM on qemu, we can see the kernel booting:

```console
# qemu-system-aarch64 -cpu max -machine virt,gic-version=3,kernel-irqchip=on -m 1024 -nographic -monitor none -kernel /boot/Image
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x000f0510]
[    0.000000] Linux version 5.10.65-tegra (buildbrain@mobile-u64-5266-d7000) (aarch64-buildroot-linux-gnu-gcc.br_real (Buildroot 2020.08) 9.3.0, GNU ld (GNU Binutils) 2.33.1) #1 SMP PREEMPT Tue Mar 15 00:53:43 PDT 2022
[    0.000000] OF: fdt: memory scan node memory@40000000, reg size 16,
[    0.000000] OF: fdt:  - 40000000 ,  40000000
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] efi: UEFI not found.
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000040000000-0x000000007fffffff]
[    0.000000]   DMA32    empty
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[snipped]
```

So after digging into this issue, googling and trying various workarounds, we
came across the points raised above about the GICv3 initialization.

### Solution

Following the
[instructions](https://docs.nvidia.com/jetson/archives/r35.2.1/DeveloperGuide/text/SD/Kernel/KernelCustomization.html)
on how to build a kernel for the Jetson AGX Orin series, we get the sources and
patch the device tree sources using the following snippet:

```patch
--- Linux_for_Tegra/source/public/hardware/nvidia/soc/t23x/kernel-dts/tegra234-soc/tegra234-soc-minimal.dtsi.orig	2022-08-11 03:14:51.000000000 +0000
+++ Linux_for_Tegra/source/public/hardware/nvidia/soc/t23x/kernel-dts/tegra234-soc/tegra234-soc-minimal.dtsi		2023-02-12 09:07:10.259761186 +0000
@@ -43,6 +43,10 @@
 		reg = <0x0 0x0f400000 0x0 0x00010000    /* GICD */
 		       0x0 0x0f440000 0x0 0x00200000>;  /* GICR CPU 0-15 */
 		ranges;
+		interrupts = <GIC_PPI 9
+                       (GIC_CPU_MASK_SIMPLE(8) | IRQ_TYPE_LEVEL_HIGH)>;
+                interrupt-parent = <&intc>;
+
 		status = "disabled";
 
 		gic_v2m: v2m@f410000 {
```

We build the kernel using the following command:

```sh
Linux_for_Tegra/source/public# ./nvbuild.sh -o kernel_out_updated
```

Essentially, just the device tree is needed:

```console
kernel_out_updated/arch/arm64/boot/dts/nvidia/tegra234-p3701-0000-p3737-0000.dtb
```

We copy this file to the board at the following directory:

```
/boot/dtb
```

and tweak the boot loader config to load the updated device-tree file, instead
of the default one:

```patch
--- /boot/extlinux/extlinux.conf.orig	2023-02-13 18:41:26.208771762 +0000
+++ /boot/extlinux/extlinux.conf	2023-02-13 18:41:37.452854874 +0000
@@ -1,6 +1,6 @@
 LABEL primary
       MENU LABEL primary kernel
       LINUX /boot/Image
-      FDT /boot/dtb/kernel_tegra234-p3701-0000-p3737-0000.dtb
+      FDT /boot/dtb/tegra234-p3701-0000-p3737-0000.dtb
       INITRD /boot/initrd
       APPEND ${cbootargs} root=/dev/mmcblk0p1 rw rootwait rootfstype=ext4 mminit_loglevel=4 console=ttyTCU0,115200 console=ttyAMA0,115200 console=tty0 firmware_class.path=/etc/firmware fbcon=map:0 net.ifnames=0
```

And now we're ready for reboot! Assuming all went well, you'll end up with the
following in the kernel boot logs:

```console
# dmesg |grep -i kvm
[    3.048360] kvm [1]: IPA Size Limit: 48 bits
[    3.052646] kvm [1]: GICv3: no GICV resource entry
[    3.057437] kvm [1]: disabling GICv2 emulation
[    3.061891] kvm [1]: GIC system register CPU interface enabled
[    3.067830] kvm [1]: vgic interrupt IRQ9
[    3.071899] kvm [1]: VHE mode initialized successfully
```

And if we try spawning a firecracker VM as above, we get the following:

```console
# ./bin/firecracker --config-file config_vsock.json --api-sock fc.sock
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd421]
[    0.000000] Linux version 5.10.0 (runner@gh-cloud-pod-ckp6h) (gcc (Ubuntu/Linaro 8.4.0-1ubuntu1~18.04) 8.4.0, GNU ld (GNU Binutils for Ubuntu) 2.30) #1 SMP Wed May 4 06:10:52 UTC 2022
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] earlycon: uart0 at MMIO 0x0000000040003000 (options '')
[    0.000000] printk: bootconsole [uart0] enabled
[    0.000000] efi: UEFI not found.
[    0.000000] NUMA: No NUMA configuration found
[    0.000000] NUMA: Faking a node at [mem 0x0000000080000000-0x000000017fffffff]
[    0.000000] NUMA: NODE_DATA [mem 0x17f6d7900-0x17f6f8fff]
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000080000000-0x00000000bfffffff]
[    0.000000]   DMA32    [mem 0x00000000c0000000-0x00000000ffffffff]
[    0.000000]   Normal   [mem 0x0000000100000000-0x000000017fffffff]
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000080000000-0x000000017fffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000080000000-0x000000017fffffff]
[    0.000000] On node 0 totalpages: 1048576
[    0.000000]   DMA zone: 4096 pages used for memmap
[    0.000000]   DMA zone: 0 pages reserved
[    0.000000]   DMA zone: 262144 pages, LIFO batch:63
[    0.000000]   DMA32 zone: 4096 pages used for memmap
[    0.000000]   DMA32 zone: 262144 pages, LIFO batch:63
[    0.000000]   Normal zone: 8192 pages used for memmap
[    0.000000]   Normal zone: 524288 pages, LIFO batch:63
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv1.0 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: Trusted OS migration not required
[    0.000000] psci: SMC Calling Convention v1.1
[    0.000000] percpu: Embedded 22 pages/cpu s49944 r8192 d31976 u90112
[    0.000000] pcpu-alloc: s49944 r8192 d31976 u90112 alloc=22*4096
[    0.000000] pcpu-alloc: [0] 0 [0] 1 
[    0.000000] Detected PIPT I-cache on CPU0
[    0.000000] CPU features: detected: GIC system register CPU interface
[    0.000000] CPU features: detected: Hardware dirty bit management
[    0.000000] CPU features: detected: Spectre-v4
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 1032192
[    0.000000] Policy zone: Normal
[    0.000000] Kernel command line: console=ttyS0 reboot=k panic=1 pci=off loglevel=8 root=/dev/vda ip=172.42.0.2::172.42.0.1:255.255.255.0::eth0:off random.trust_cpu=on root=/dev/vda rw earlycon=uart,mmio,0x40003000
[    0.000000] Dentry cache hash table entries: 524288 (order: 10, 4194304 bytes, linear)
[    0.000000] Inode-cache hash table entries: 262144 (order: 9, 2097152 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] software IO TLB: mapped [mem 0x00000000bbfff000-0x00000000bffff000] (64MB)
[    0.000000] Memory: 4024740K/4194304K available (8064K kernel code, 7700K rwdata, 2060K rodata, 1408K init, 3005K bss, 169564K reserved, 0K cma-reserved)
[    0.000000] random: get_random_u64 called from __kmem_cache_create+0x2c/0x4a0 with crng_init=0
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
[    0.000000] rcu: Hierarchical RCU implementation.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=128 to nr_cpu_ids=2.
[    0.000000] 	Tracing variant of Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=2
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] GICv3: 96 SPIs implemented
[    0.000000] GICv3: 0 Extended SPIs implemented
[    0.000000] GICv3: Distributor has no Range Selector support
[    0.000000] GICv3: 16 PPIs implemented
[    0.000000] GICv3: CPU0: found redistributor 0 region 0:0x000000003ffb0000
[    0.000000] arch_timer: cp15 timer(s) running at 31.25MHz (virt).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0xe6a171046, max_idle_ns: 881590405314 ns
[    0.000003] sched_clock: 56 bits at 31MHz, resolution 32ns, wraps every 4398046511088ns
[    0.001027] Console: colour dummy device 80x25
[    0.001600] Calibrating delay loop (skipped), value calculated using timer frequency.. 62.50 BogoMIPS (lpj=125000)
[    0.003003] pid_max: default: 32768 minimum: 301
[    0.003621] LSM: Security Framework initializing
[    0.004239] SELinux:  Initializing.
[    0.004705] Mount-cache hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.005642] Mountpoint-cache hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.007399] rcu: Hierarchical SRCU implementation.
[    0.008182] EFI services will not be available.
[    0.008825] smp: Bringing up secondary CPUs ...
[    0.016089] Detected PIPT I-cache on CPU1
[    0.016135] GICv3: CPU1: found redistributor 1 region 0:0x000000003ffd0000
[    0.016251] CPU1: Booted secondary processor 0x0000000001 [0x410fd421]
[    0.016842] smp: Brought up 1 node, 2 CPUs
[    0.019483] SMP: Total of 2 processors activated.
[    0.020055] CPU features: detected: Privileged Access Never
[    0.020721] CPU features: detected: LSE atomic instructions
[    0.021326] CPU features: detected: User Access Override
[    0.021913] CPU features: detected: 32-bit EL0 Support
[    0.022464] CPU features: detected: Common not Private translations
[    0.023120] CPU features: detected: RAS Extension Support
[    0.023723] CPU features: detected: Data cache clean to the PoU not required for I/D coherence
[    0.024694] CPU features: detected: CRC32 instructions
[    0.025269] CPU features: detected: Speculative Store Bypassing Safe (SSBS)
[    0.051711] CPU: All CPU(s) started at EL1
[    0.052455] alternatives: patching kernel code
[    0.054839] devtmpfs: initialized
[    0.056234] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.057936] futex hash table entries: 512 (order: 3, 32768 bytes, linear)
[    0.059329] DMI not present or invalid.
[    0.060220] NET: Registered protocol family 16
[    0.062118] DMA: preallocated 512 KiB GFP_KERNEL pool for atomic allocations
[    0.064139] DMA: preallocated 512 KiB GFP_KERNEL|GFP_DMA pool for atomic allocations
[    0.066196] DMA: preallocated 512 KiB GFP_KERNEL|GFP_DMA32 pool for atomic allocations
[    0.067848] audit: initializing netlink subsys (disabled)
[    0.069643] audit: type=2000 audit(0.068:1): state=initialized audit_enabled=0 res=1
[    0.069818] thermal_sys: Registered thermal governor 'fair_share'
[    0.071171] thermal_sys: Registered thermal governor 'step_wise'
[    0.071923] thermal_sys: Registered thermal governor 'user_space'
[    0.072795] cpuidle: using governor ladder
[    0.074720] cpuidle: using governor menu
[    0.075499] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    0.076945] ASID allocator initialised with 65536 entries
[    0.085849] HugeTLB registered 1.00 GiB page size, pre-allocated 0 pages
[    0.087077] HugeTLB registered 32.0 MiB page size, pre-allocated 0 pages
[    0.088149] HugeTLB registered 2.00 MiB page size, pre-allocated 0 pages
[    0.089190] HugeTLB registered 64.0 KiB page size, pre-allocated 0 pages
[    0.093766] iommu: Default domain type: Translated 
[    0.094597] SCSI subsystem initialized
[    0.095080] pps_core: LinuxPPS API ver. 1 registered
[    0.095636] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.096684] PTP clock support registered
[    0.097441] NetLabel: Initializing
[    0.097830] NetLabel:  domain hash size = 128
[    0.098321] NetLabel:  protocols = UNLABELED CIPSOv4 CALIPSO
[    0.098967] NetLabel:  unlabeled traffic allowed by default
[    0.099750] clocksource: Switched to clocksource arch_sys_counter
[    0.100568] VFS: Disk quotas dquot_6.6.0
[    0.101037] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    0.101952] FS-Cache: Loaded
[    0.102771] CacheFiles: Loaded
[    0.104955] NET: Registered protocol family 2
[    0.105781] tcp_listen_portaddr_hash hash table entries: 2048 (order: 3, 32768 bytes, linear)
[    0.107212] TCP established hash table entries: 32768 (order: 6, 262144 bytes, linear)
[    0.108659] TCP bind hash table entries: 32768 (order: 7, 524288 bytes, linear)
[    0.110424] TCP: Hash tables configured (established 32768 bind 32768)
[    0.111486] UDP hash table entries: 2048 (order: 4, 65536 bytes, linear)
[    0.112545] UDP-Lite hash table entries: 2048 (order: 4, 65536 bytes, linear)
[    0.113614] NET: Registered protocol family 1
[    0.114900] Initialise system trusted keyrings
[    0.115467] Key type blacklist registered
[    0.116112] workingset: timestamp_bits=36 max_order=20 bucket_order=0
[    0.118568] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    0.132869] Key type asymmetric registered
[    0.133466] Asymmetric key parser 'x509' registered
[    0.134217] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 251)
[    0.136154] Serial: 8250/16550 driver, 1 ports, IRQ sharing disabled
[    0.137435] printk: console [ttyS0] disabled
[    0.138118] 40003000.uart: ttyS0 at MMIO 0x40003000 (irq = 14, base_baud = 1500000) is a 16550A
[    0.139487] printk: console [ttyS0] enabled
[    0.139487] printk: console [ttyS0] enabled
[    0.140875] printk: bootconsole [uart0] disabled
[    0.140875] printk: bootconsole [uart0] disabled
[    0.142604] cacheinfo: Unable to detect cache hierarchy for CPU 0
[    0.146080] loop: module loaded
[    0.146816] virtio_blk virtio0: [vda] 2097152 512-byte logical blocks (1.07 GB/1.00 GiB)
[    0.147764] vda: detected capacity change from 0 to 1073741824
[    0.149279] Loading iSCSI transport class v2.0-870.
[    0.150525] iscsi: registered transport (tcp)
[    0.151107] tun: Universal TUN/TAP device driver, 1.6
[    0.152381] rtc-pl031 40004000.rtc: designer ID = 0x41
[    0.153012] rtc-pl031 40004000.rtc: revision = 0x1
[    0.153898] rtc-pl031 40004000.rtc: char device (254:0)
[    0.154530] rtc-pl031 40004000.rtc: registered as rtc0
[    0.155158] rtc-pl031 40004000.rtc: setting system clock to 2023-02-13T19:08:49 UTC (1676315329)
[    0.156324] hid: raw HID events driver (C) Jiri Kosina
[    0.157485] Initializing XFRM netlink socket
[    0.158395] NET: Registered protocol family 10
[    0.160444] Segment Routing with IPv6
[    0.160971] NET: Registered protocol family 17
[    0.161565] Key type dns_resolver registered
[    0.162179] NET: Registered protocol family 40
[    0.163868] registered taskstats version 1
[    0.164554] Loading compiled-in X.509 certificates
[    0.166411] Loaded X.509 cert 'Build time autogenerated kernel key: 5c87d35d601eb9a30312d062d8891ae920951e61'
[    0.167750] Key type ._fscrypt registered
[    0.168431] Key type .fscrypt registered
[    0.169023] Key type fscrypt-provisioning registered
[    0.170251] Key type encrypted registered
2023-02-13T19:08:49.402741111 [anonymous-instance:ERROR:src/devices/src/virtio/net/device.rs:390] Failed to write to tap: Os { code: 5, kind: Other, message: "Input/output error" }
[    0.187729] IP-Config: Complete:
[    0.188127]      device=eth0, hwaddr=aa:fc:00:00:00:01, ipaddr=172.42.0.2, mask=255.255.255.0, gw=172.42.0.1
[    0.189265]      host=172.42.0.2, domain=, nis-domain=(none)
[    0.189908]      bootserver=255.255.255.255, rootserver=255.255.255.255, rootpath=
[    0.190773] 
[    0.193054] EXT4-fs (vda): mounted filesystem with ordered data mode. Opts: (null)
[    0.194395] VFS: Mounted root (ext4 filesystem) on device 254:0.
[    0.195952] devtmpfs: mounted
[    0.197005] Freeing unused kernel memory: 1408K
[    0.215773] Run /sbin/init as init process
[    0.216663]   with arguments:
[    0.217281]     /sbin/init
[    0.217844]   with environment:
[    0.218496]     HOME=/
[    0.218989]     TERM=linux
[    0.219552]     pci=off
SELinux:  Could not open policy file <= /etc/selinux/targeted/policy/policy.33:  No such file or directory
[    0.258127] systemd[1]: Failed to find module 'autofs4'
[    0.263505] systemd[1]: systemd 245.4-4ubuntu3.11 running in system mode. (+PAM +AUDIT +SELINUX +IMA +APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +SECCOMP +BLKID +ELFUTILS +KMOD +IDN2 -IDN +PCRE2 default-hierarchy=hybrid)
[    0.266780] systemd[1]: Detected architecture arm64.

Welcome to Ubuntu 20.04.2 LTS!

[snipped]
```

Initially, we got an RTC initialization error, so in case you get such an
issue, disable `pl031_driver_init` through the `initcall_blacklist` kernel
cmdline option:

```patch
--- config_vsock.json.orig	2023-02-13 19:17:43.750699285 +0000
+++ config_vsock.json	2023-02-13 19:17:57.154793334 +0000
@@ -1,7 +1,7 @@
 {
 	"boot-source": {
 		"kernel_image_path": "vmlinux",
-		"boot_args": "console=ttyS0 reboot=k panic=1 pci=off loglevel=8 root=/dev/vda ip=172.42.0.2::172.42.0.1:255.255.255.0::eth0:off random.trust_cpu=on"
+		"boot_args": "console=ttyS0 reboot=k panic=1 pci=off loglevel=8 root=/dev/vda ip=172.42.0.2::172.42.0.1:255.255.255.0::eth0:off random.trust_cpu=on initcall_blacklist=pl031_driver_init"
 	},
 	"drives": [
 		{
```

## Future steps

Give us a shout at team@cloudkernels.net if you liked it! The plan is to use
these boards to expose acceleration functionality in isolated workloads running
on sandboxed containers using [kata-containers](https://katacontainers.io) and
[vAccel](https://vaccel.org). Take a sneak peek on what we are working on
[here](https://docs.vaccel.org/container_runtimes) and stay tuned for the next
post where we describe the process to build and run a vAccel-enabled sandboxed
container on a Jetson Orin! 
