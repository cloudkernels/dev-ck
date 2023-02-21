---
title: "Isolated, hardware-accelerated functions on Jetson AGX Orin"
date: 2023-02-16T10:14:04+01:00
---

Following up on a successful [VM boot](/orin-vm) on a [Jetson AGX
Orin](https://www.nvidia.com/en-il/autonomous-machines/embedded-systems/jetson-orin/),
we continue exploring the capabilities of this edge device, focusing on the
cloud-native aspect of application deployment.

As a team, we've built [vAccel](https://docs.vaccel.org), a hardware
acceleration framework that decouples the operation from its hardware
implementation. One of the awesome things vAccel offers is the ability to
execute hardware-accelerated applications in a VM that has no direct access to
a hardware accelerator. Given the Jetson Orin board has an Ampere GPU, with
1792 cores and 56 Tensor Cores, it sounds like a perfect edge device to try
isolated hardware-accelerated workload execution through VM sandboxes. 

Additionally, coupled with our downstream kata-containers port, we can invoke a
VM sandboxed container through CRI that can execute compute-intensive tasks
faster by using the GPU without having direct access to it!

## Overview

In this post we will first go through the steps to run a vAccel application on
a number of VMMs ([QEMU](https://github.com/qemu/qemu), [AWS
Firecracker](https://github.com/firecracker-microvm/firecracker), [Cloud
hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor), and
[Dragonball](https://github.com/kata-containers/kata-containers/)) on the
Jetson AGX Orin board. Then, we will walk through the steps to run a VM
sandboxed container that uses vAccel on our
[custom](https://github.com/nubificus/kata-containers) kata-containers runtime.
Then, we [patch](/orin-vm#Solution) the device tree to include the interrupt
register definitions, in order to enable proper setup of the GICv3
initialization. Then, we go through the build process for the vAccel-enabled
kata-containers runtime. And finally, we spawn stock sandboxed containers using
the `go` and `rust` runtime of kata, as well as vAccel-enabled ones, using the
`go` runtime.


#### Requirements

The steps to build & install the stock `kata-containers` runtime are shown below:

```sh



```


