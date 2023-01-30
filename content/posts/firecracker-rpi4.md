---
title: "Porting Firecracker to a Raspberry Pi 4"
date: 2019-10-21T09:58:31+02:00
---

Since we got our hands on the new Raspberry Pi 4, we started exploring how
various virtualization technologies behave on the board. First thing we tried is how to run [Nabla][0] on it and how it compares to native KVM.

Next thing we wanted to try is [firecracker][1], the notorious micro-VMM that
Amazon Lambda & Fargate run on. To our disappointment, firecracker was not
[yet][2] running on RPi4. So we started looking into coding in the necessary
changes :)

After a bit of investigation, we figured out that the key missing piece is
support for the GICv2 ARM interrupt controller. In fact, firecracker only
supports GIC version 3 since it was open-sourced, but not version 2, which is
the version appearing in the Raspberry Pi series, as well as in other popular
boards, like the Hikey 970 we got our hands on, or Khadas VIM3. A bit more
digging into the internals of firecracker and the Linux kernel, helped us
work out the details and open a [pull-request][3] which adds support for the
missing parts.

#### Changes in Firecracker

The Generic Interrupt Controller (GIC) is the IP block in the ARM processors
that implements interrupt handling. The Linux kernel supports user- and
kernel-space emulation for GICv2 as well as GICv3 and v4. However, Firecracker
only handles the GICv3-related configuration of the virtual GIC (VGIC).
Similarly, setting up the FDT for the guest microVM from Firecracker only
handles GIC-v3 devices.

In terms of code organization, the GIC related code currently exposes a function
for setting up GICv3 performing the corresponding `ioctl` KVM commands. The
first part of our PR changes this by introducing a GIC Trait which defines the
common interface for all GIC implementations. Even though it is still under
discussion, what exactly will be in the Trait in its final form, it will be
something along the following lines:

```rs
/// Trait for GIC devices.
pub trait GICDevice {
    /// Returns the GIC version of the device
    fn version(&self) -> u32;

    /// Returns the file descriptor of the GIC device
    fn device_fd(&self) -> &DeviceFd;

    /// Returns an array with GIC device properties
    fn device_properties(&self) -> &[u64];
}
```

With this in place, we can define objects per GIC version, which implement this
Trait, and still have the rest of the code deal with the `GICDevice` Trait,
which is transparent to the GIC version.

The implementations for each version, implement the Trait and a `new` function
which is used to create the new object:

```rs
pub fn new(vm: &VmFd, vcpu_count: u64) -> Result<Box<GICDevice>>
```

The differentiation between the two versions lays in the VGIC register
attributes that each device exposes. As a result, the work we need to do is essentially to `mmap` their addresses.

GICv2 relevant addresses include the **distributor** and **cpu** groups:

```rs
/* Setting up the distributor attribute.
 We are placing the GIC below 1GB so we need to substract the size of the distributor.
*/
set_device_attribute(
    &vgic_fd,
    kvm_bindings::KVM_DEV_ARM_VGIC_GRP_ADDR,
    u64::from(kvm_bindings::KVM_VGIC_V2_ADDR_TYPE_DIST),
    &get_dist_addr() as *const u64 as u64,
    0,
)?;

/* Setting up the CPU attribute.
 */
set_device_attribute(
    &vgic_fd,
    kvm_bindings::KVM_DEV_ARM_VGIC_GRP_ADDR,
    u64::from(kvm_bindings::KVM_VGIC_V2_ADDR_TYPE_CPU),
    &get_cpu_addr() as *const u64 as u64,
    0,
)?;
```

whereas the GICv3 includes the **distributor** and **redistributor** groups:

```rs
/* Setting up the distributor attribute.
 We are placing the GIC below 1GB so we need to substract the size of the distributor.
*/
set_device_attribute(
    &vgic_fd,
    kvm_bindings::KVM_DEV_ARM_VGIC_GRP_ADDR,
    u64::from(kvm_bindings::KVM_VGIC_V3_ADDR_TYPE_DIST),
    &get_dist_addr() as *const u64 as u64,
    0,
)?;

/* Setting up the redistributors' attribute.
We are calculating here the start of the redistributors address. We have one per CPU.
*/
set_device_attribute(
    &vgic_fd,
    kvm_bindings::KVM_DEV_ARM_VGIC_GRP_ADDR,
    u64::from(kvm_bindings::KVM_VGIC_V3_ADDR_TYPE_REDIST),
    &get_redists_addr(u64::from(vcpu_count)) as *const u64 as u64,
    0,
)?;
```

Finally, for both versions of GIC we finalize the device initialization by
setting the number of supported interrupts and the *control_init* group.

```rs
/// Finalize the setup of a GIC device
pub fn finalize_device(fd: &DeviceFd) -> Result<()> {
    /* We need to tell the kernel how many irqs to support with this vgic.
     * See the `layout` module for details.
     */
    let nr_irqs: u32 = super::layout::IRQ_MAX - super::layout::IRQ_BASE + 1;
    let nr_irqs_ptr = &nr_irqs as *const u32;
    set_device_attribute(
        fd,
        kvm_bindings::KVM_DEV_ARM_VGIC_GRP_NR_IRQS,
        0,
        nr_irqs_ptr as u64,
        0,
    )?;

    /* Finalize the GIC.
     * See https://code.woboq.org/linux/linux/virt/kvm/arm/vgic/vgic-kvm-device.c.html#211.
     */
    set_device_attribute(
        fd,
        kvm_bindings::KVM_DEV_ARM_VGIC_GRP_CTRL,
        u64::from(kvm_bindings::KVM_DEV_ARM_VGIC_CTRL_INIT),
        0,
        0,
    )?;

    Ok(())
}
```

Regarding the FDT creation, there are differences between v2 and v3 regarding
the interrupt controller `intc` node.

First, we need to define the GICv2 `compatible` property to be `arm,gic-400`,
since it is the GIC-400 chip which implements GICv2. Next is the `reg`
property of the FDT, which includes the addresses and the corresponding sizes of
the GIC registers, i.e. distributor and CPU for GICv2, distributor and
redistributor for GICv3. Finally, we fix the `interrupts` property which
determines the interrupt source of the VGIC maintenance interrupt. The
corresponding values for GICv2 were taken from the respective [Linux Kernel entries][6].

#### Build firecracker with RPi4 support

While waiting for the patch to merge upstream you can go ahead and check out it
yourself.

Clone and build our fork of firecracker (keep in mind that while the PR review
is going on, we might force-update the branch).

```bash
$ git clone https://github.com/cloudkernels/firecracker.git
$ cd firecracker
$ ./tools/devtool build
```

Next you need a kernel image and a root filesystem to run with your firecracker
build. You can grab the pre-built kernel image and rootfs here:

```bash
$ wget https://s3.amazonaws.com/spec.ccfc.min/img/aarch64/ubuntu_with_ssh/kernel/vmlinux.bin
$ wget https://s3.amazonaws.com/spec.ccfc.min/img/aarch64/ubuntu_with_ssh/fsfiles/xenial.rootfs.ext4 
```

Alternatively, you can build your own kernel and rootfs using the following
steps provided by the firecracker docs
[here][4]


#### Launch our image

To launch our image we will use the [firectl][5] tool. We need, to setup a tap
device for the networking or our firecracker microVM.

```bash
$ sudo ip tuntap add dev tap0 mode tap
$ sudo ip addr add 172.16.0.1/24 dev tap0
$ sudo ip link set tap0 up
```

And we launch the microVM using firectl:

```bash
firectl --firecracker-binary=${PATH_TO_FC_BIN} --kernel=vmlinux.bin \
	--tap-device=tap0/aa:fc:00:00:00:01 \
	--kernel-opts="console=ttyS0 reboot=k panic=1 pci=off ip=172.16.0.42::172.16.0.1:255.255.255.0::eth0:off" \
	--root-drive=./nginx_fc.ext4
```

Here, we used here the `ip` kernel boot parameter to give the `172.16.0.42`.

Finally, we try out our nginx server:

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

[0]: https://cloudkernels.github.io/posts/rpi4-64bit-virt/
[1]: https://github.com/firecracker-microvm/firecracker
[2]: https://github.com/firecracker-microvm/firecracker/issues/1196
[3]: https://github.com/firecracker-microvm/firecracker/pull/1235
[4]: https://github.com/cloudkernels/firecracker/blob/master/docs/rootfs-and-kernel-setup.md 
[5]: https://github.com/firecracker-microvm/firectl
[6]: https://github.com/torvalds/linux/blob/master/Documentation/devicetree/bindings/interrupt-controller/arm%2Cgic.yaml
