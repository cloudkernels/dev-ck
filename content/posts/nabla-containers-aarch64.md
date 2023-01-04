---
title: "Experiences from porting nabla containers to an ARMv8 board"
date: 2019-01-23T14:41:44+02:00
lastmod: 2019-02-18T19:38:21+0200
---

\[**UPDATE:** Rumprun aarch64 support has now been merged in
[upstream nabla][11].\]
<br>
<br>

[Nabla containers][1] provide a new type of container designed for strong
isolation on a host system. The foundation of nabla containers lies in three
main components: [rumpkernel][2], [solo5][3], and [runnc][4]. The [team][5]
that built nabla containers extended the rumprun unikernel framework to support
solo5 (instead of hardware/baremetal or xen), so that a rumprun-baked unikernel
application can be executed on top of a lightweight monitor such as solo5. In
this post, we describe the steps we took in order to port Nabla containers to
the ARMv8 architecture.


### A bit of background ###
[Rumprun][2a] is a unikernel framework based on [rumpkernel][2], a project that
provides kernel-quality drivers for various components, e.g. file systems,
network devices, POSIX system calls. Rump kernel exposes these drivers through
the *rump kernel hypercall interface* to higher abstraction layers.  While
initially, rump kernels were designed to provide 'userspace' [drivers][3a], they
evolved to become the base of the Rumprun unikernel.

Almost any POSIX-based application can be built into a Rumprun unikernel using
the following workflow:

1. Compile and link the application against the POSIX-y interface that Rumprun
exposes.
2. *Bake* the application, in order to add the bits and pieces needed to turn
it into an image that 
is bootable on the targets that Rumprun runs on top of.

Upstream Rumprun defines the concept of *target*, i.e. the platforms on which a
Rumprun unikernel can run on top of. At the moment the upstream Rumprun
provides two targets:

* The **hw** target provides support on raw hardware which includes most
available hypervisors.
* The **Xen** target is optimized for execution on top of the Xen hypervisor

{{< figure src="/static/rumprun-solo5.png" >}}

The Nabla fork of Rumprun provides a new unikernel target, [Solo5][3]. Solo5
is, essentially, a hardware abstraction layer that provides a very thin
interface, or else a *minimal attach surface*.  Its purpose is to facilitate
the port of libOS/unikernel frameworks on various hardware platforms, i.e.  a
unikernel that runs on top of Solo5, runs on top of all the hardware targets,
or *tenders*, using the Solo5 terminology, supported by Solo5.

### ARMv8 rumprun solo5 ###

To port nabla containers to the ARMv8 architecture, one has to provide support
for each of these components: rumprun, solo5 and runnc. We decided to tackle
this challenge by separately porting each component and working on integrating
as much code as possible from upstream repositories.

#### Solo5 ####

For the solo5 port, most of the code was already in upstream solo5, although
the seccomp tender (as it is now called) provided by nabla didn’t have support
for aarch64 targets. To implement that, apart from adding the compilation
target itself, we also had to provide the arch specific lds and add the
hypercall-to-syscall mechanism used to aarch64. Finally, for everything to
actually work we implemented reading the cpu timer frequency correctly for
aarch64 and provided some missing seccomp definitions.
As of the time of writing this post, an [upstream solo5-seccomp tender][6] for
both x86 and aarch64 is under way and should be merged in really soon.

#### Rumprun ####

For rumprun, things were a bit more complicated. Both upstream and nabla
rumprun repos build necessary NetBSD parts from an old snapshot of the official
sources. NetBSD has added  aarch64 support fairly recently and with much work
still being done we decided to base our build on the latest official sources
instead of the stripped down version provided by rumprun. This presented a
challenge: integrating the changes to a not actively maintained code base is
not a walk in the park. First things first, we had to successfully compile the
whole project: Rump parts of the NetBSD kernel are not actively tested with
rumprun and so changes being made to related components are not always
guaranteed to work. After adding the aarch64 platform to the required Makefiles
for rump in NetBSD source and creating the relevant directories, we encountered
linking errors pertaining to both double symbols between the provided libc and
rump and also incorrectly linked for the rump case internal functions of the
kernel itself. Thankfully similar problems have already been solved for the arm
32-bit architecture and so we could implement a solution based on existing
code. 

Having successfully compiled NetBSD’s source we then had to implement any
missing rumpuser parts either for aarch64 or for functions introduced in
upstream kernel’s rump.

The most challenging part was to understand how thread-level switching happens
on aarch64 and implement that in the context of rumprun. Upon initial creation
of “main” threads for the rump components, rumprun stores a “bouncer” function
on top of their freshly allocated stacks and then switches between them using
its scheduler. When a thread’s turn comes to run, the “bouncer” function is
popped from the stack and the actual thread content is executed. 

The mechanism doing the actual thread switching is implemented in arch-specific
assembly. Although a basic implementation for the ARM 32-bit architecture is
provided with upstream rumprun, porting to 64-bit ARMv8 was not trivial due to
two main architectural differences of ARMv8: a) one cannot directly modify the
program counter, and b) storing and retrieving registers to/from the stack has
to be aligned if they are used for memory access. As a result, push and pop
functions have to be implemented manually, as no specific instruction exists
for that purpose. After implementing the actual thread switcher and bouncer, we
also had to make sure the TLS (Thread-Local Storage) register was set correctly
in the solo5 platform implementation.

Last but not least, with thread switching in place, the rest of the solo5
platform arch-specific bits needed to be created: the ldscript for linking, the
relevant headers and the Makefile modifications for everything to build
correctly.

NOTE: Stack protection in NetBSD source caused problems in our tests and so
it’s globally disabled for rumprun pending further investigation.

#### runnc ####

Since runnc is written in GO, the arch-specific bits were only build related,
so porting it to aarch64 was as easy as defining the architecture (arm64) and
pointing to the correct submodule repos.

We are currently working with the [nabla containers team][5] to merge in aarch64
support. In the meantime, you can [browse][7] [through][8] [the][9] [code][10].
Stay tuned for a tutorial on how to run a rumprun unikernel on a RPi3!


[1]: https://nabla-containers.github.io
[2a]: https://github.com/rumpkernel/rumprun
[2]: http://rumpkernel.org
[3a]: https://wiki.netbsd.org/projects/project/userland_pci
[3]: https://github.com/Solo5/solo5
[4]: https://github.com/nabla-containers/runnc
[5]: https://nabla-containers.github.io/people
[6]: https://github.com/Solo5/solo5/pull/310
[7]: https://github.com/cloudkernels/rumprun-aarch64
[8]: https://github.com/cloudkernels/src-netbsd
[9]: https://github.com/cloudkernels/rumprun-packages
[10]: https://github.com/cloudkernels/solo5
[11]: https://github.com/nabla-containers/rumprun
