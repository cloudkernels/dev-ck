---
title: "Building the Nabla containers toolchain for aarch64"
date: 2019-01-24T16:17:51+02:00
lastmod: 2019-02-18T19:37:57+0200
---
\[**UPDATE:** Revise instructions and links to use latest [upstream nabla repo][3]
with merged aarch64 support.\]
<br>
<br>

In [previous]({{<relref "nabla-containers-aarch64.md">}}) [posts]({{< relref
"example-rumprun-solo5-on-aarch64.md">}}), we covered a bit of background on
[rumprun][0], the [nabla containers fork][1] and our port on [aarch64][2]. In
this post, we describe how to build everything from source. In order to build a
rumprun unikernel for aarch64, the first step is to build the rumprun
toolchain.

Clone the relevant repositories:

```
git clone https://github.com/nabla-containers/rumprun
git clone https://github.com/cloudkernels/rumprun-packages
```

```
cd rumprun
git submodule update --init
```

Build rumprun with:

```
cd rumprun
make
```

The build is tested with gcc-5 but it should work with gcc versions up to 6.
If you need to explicitly set the gcc version use:

```
cd rumprun
CC=<gcc-version> make
```
Solo5 is now built by default as part of the nabla-containers code distribution.

Build (& bake) hello (both hvt and spt versions):

```
cd rumprun-packages
cd hello
make
```

Output is the following:

```
mkdir -p build
cp src/* build
CC=aarch64-rumprun-netbsd-gcc make -C build hello.spt
make[1]: Entering directory '/build/rumprun-packages/hello/build'
aarch64-rumprun-netbsd-gcc hello.c -o hello-rumprun
rumprun-bake solo5_spt hello.spt hello-rumprun

!!!
!!! NOTE: rumprun-bake is experimental. syntax may change in the future
!!!

make[1]: Leaving directory '/build/rumprun-packages/hello/build'
mkdir -p bin
cp build/hello.spt bin/hello.spt
CC=aarch64-rumprun-netbsd-gcc make -C build hello.hvt
make[1]: Entering directory '/build/rumprun-packages/hello/build'
rumprun-bake solo5_hvt hello.hvt hello-rumprun

!!!
!!! NOTE: rumprun-bake is experimental. syntax may change in the future
!!!

make[1]: Leaving directory '/build/rumprun-packages/hello/build'
mkdir -p bin
cp build/hello.hvt bin/hello.hvt
```

Try to run both (an existing dummy file and tap0 device are expected):

```
../../rumprun/solo5/tenders/hvt/solo5-hvt --disk=dummy --net=tap0 ./bin/hello.hvt
```

Output:

```
solo5-hvt: bin/hello.hvt: Warning: phdr[0] requests WRITE and EXEC permissions
solo5-hvt: WARNING: Tender is configured with HVT_DROP_PRIVILEGES=0. Not dropping any privileges.
solo5-hvt: WARNING: This is not recommended for production use.
            |      ___|
  __|  _ \  |  _ \ __ \
\__ \ (   | | (   |  ) |
____/\___/ _|\___/____/
Solo5: Memory map: 512 MB addressable:
Solo5:   reserved @ (0x0 - 0xfffff)
Solo5:       text @ (0x100000 - 0x30efff)
Solo5:     rodata @ (0x30f000 - 0x35bfff)
Solo5:       data @ (0x35c000 - 0x3cafff)
Solo5:       heap >= 0x3cb000 < stack < 0x20000000
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
rumprun: could not find start of json.  no config?
mounted tmpfs on /tmp

=== calling "rumprun" main() ===

This is the Rumprun Hello World ...
... using the Solo5 framework ...
... in a Nabla container via runnc!

=== main() of "rumprun" returned 0 ===

=== _exit(0) called ===
[   3.0281712] rump kernel halting...
[   3.0281712] syncing disks... done
[   3.0281712] unmounting file systems...
[   3.0281712] unmounted tmpfs on /tmp type tmpfs
[   3.0281712] unmounted rumpfs on / type rumpfs
[   3.0281712] unmounting done
halted
Solo5: solo5_exit(0) called
```
and the spt one:
```
../../rumprun/solo5/tenders/spt/solo5-spt --disk=dummy --net=tap0 ./bin/hello.spt
```

```
solo5-spt: bin/hello.spt: Warning: phdr[0] requests WRITE and EXEC permissions
            |      ___|
  __|  _ \  |  _ \ __ \
\__ \ (   | | (   |  ) |
____/\___/ _|\___/____/
Solo5: Memory map: 512 MB addressable:
Solo5:   reserved @ (0x0 - 0xfffff)
Solo5:       text @ (0x100000 - 0x30bfff)
Solo5:     rodata @ (0x30c000 - 0x357fff)
Solo5:       data @ (0x358000 - 0x3c6fff)
Solo5:       heap >= 0x3c7000 < stack < 0x20000000
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
rumprun: could not find start of json.  no config?
mounted tmpfs on /tmp

=== calling "rumprun" main() ===

This is the Rumprun Hello World ...
... using the Solo5 framework ...
... in a Nabla container via runnc!

=== main() of "rumprun" returned 0 ===

=== _exit(0) called ===
[   3.0211861] rump kernel halting...
[   3.0211861] syncing disks... done
[   3.0211861] unmounting file systems...
[   3.0315990] unmounted tmpfs on /tmp type tmpfs
[   3.0315990] unmounted rumpfs on / type rumpfs
[   3.0315990] unmounting done
halted
Solo5: solo5_exit(0) called
```

[0]: http://rumpkernel.org
[1]: https://nabla-containers.github.io
[2]: https://github.com/cloudkernels/rumprun
[3]: https://github.com/nabla-containers/rumprun
