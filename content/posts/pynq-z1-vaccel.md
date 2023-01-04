---
title: "Remote FPGA acceleration: vAccel on PYNQ-Z1"
date: 2023-01-20T10:14:04+01:00
---

Running applications that need hardware acceleration in public clouds remains a
challenge, both for end-users and for service providers. The reasons for this are 
mostly related to the complicated hardware abstractions that acceleration devices expose,
as well as to the complicated software stacks that drive these devices.

In an effort to hide the complexity of the software stack under the hood, and
provide end-users with the capability of accelerating their applications, we
have introduced [vAccel](https://vaccel.org), a hardware acceleration
abstraction to semantically expose functions that can be accelerated to
workloads running in VMs or even remote hosts. 

In this post, we will present how we can use vAccel to remotely execute basic
FPGA operations on a PYNQ-Z1 board. First, we go through a brief description of
the hardware and software components of this example, as well as the steps to
reproduce the experiment. We install the vAccel software stack to the board
running a generic linux distribution and run a local example. Then, we run the
same example remotely, using a client machine connected to the same network as
our development board.

## Overview

As mentioned above, we are using a PYNQ-Z1 development board. The PYNQ-Z1 board
is the hardware platform for the PYNQ open-source framework. It features a
Zynq-7000 (XC7Z020-1CLG400C) All Programmable System-On-Chip (APSoC),
integrating a feature-rich dual-core Cortex-A9 based processing system (PS) and
Xilinx programmable logic (PL) in a single device. Figure 1 shows an image of
the PYNQ-Z1 development board by Digilent.

Apart from Petalinux, you can install a generic linux distribution. We recently
walked through the process of [installing debian on a PYNQ-Z1](../pynq-z1). 

## Install vAccel

We can use the binary release or build from source. For the sake of completeness we present both options.

### Install from binaries

Get the deb package for the core vAccelRT library and install it:

```sh
wget https://s3.nbfc.io/nbfc-assets/github/vaccelrt/master/aarch32/Release-deb/vaccel-0.5.0-Linux.deb
sudo dpkg -i vaccel-0.5.0-Linux.deb
```

We should be presented with a couple of libraries in `/usr/local/lib` as well
as some example binaries on `/usr/local/bin`.

Skip the next section and go directly to [Test the installation](#test-the-installation).

### Build from source

Clone the repo and prepare to build:

```sh
git clone https://github.com/cloudkernels/vaccelrt --recursive
cd vaccelrt
mkdir -p build && cd build
cmake ../ -DBUILD_PLUGIN_NOOP=ON -DBUILD_EXAMPLES=ON
```

To build and install use the following simple command:

```sh
make install
```

## Test the installation

To make sure we've got everything setup correctly, we can run a couple of examples. First we could use the `noop` plugin to do image classification on an image.

```sh
# Set the path to the vAccel libraries
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

# Set the plugin to noop
export VACCEL_BACKENDS=/usr/local/lib/libvaccel-noop.so

# enable Debug
export VACCEL_DEBUG_LEVEL=4 

# Run the classify example
/usr/local/bin/classify /usr/local/share/images/example.jpg 1
```

We should be presented with debug output and a dummy classification tag:

```console
user@debian-fpga:~$ /usr/local/bin/classify /usr/local/share/images/example.jpg 1
2023.01.21-02:46:23.09 - <debug> Initializing vAccel
2023.01.21-02:46:23.09 - <debug> Created top-level rundir: /run/user/1001/vaccel.ogcPti
2023.01.21-02:46:23.09 - <debug> Registered plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function noop from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function sgemm from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function image classification from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function image detection from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function image segmentation from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function image pose estimation from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function image depth estimation from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function exec from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function TensorFlow session load from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function TensorFlow session run from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function TensorFlow session delete from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function MinMax from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function Array copy from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function Vector Add from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function Parallel acceleration from plugin noop
2023.01.21-02:46:23.09 - <debug> Registered function Matrix multiplication from plugin noop
2023.01.21-02:46:23.09 - <debug> Loaded plugin noop from /usr/local/lib/libvaccel-noop.so
2023.01.21-02:46:23.09 - <debug> session:1 New session
Initialized session with id: 1
Image size: 79281B
2023.01.21-02:46:23.11 - <debug> session:1 Looking for plugin implementing image classification
2023.01.21-02:46:23.11 - <debug> Found implementation in noop plugin
[noop] Calling Image classification for session 1
[noop] Dumping arguments for Image classification:
[noop] len_img: 79281
[noop] will return a dummy result
classification tags: This is a dummy classification tag!
2023.01.21-02:46:23.11 - <debug> session:1 Free session
2023.01.21-02:46:23.11 - <debug> Shutting down vAccel
2023.01.21-02:46:23.11 - <debug> Cleaning up plugins
2023.01.21-02:46:23.11 - <debug> Unregistered plugin noop
```

## Run local example

An example more tailored to the board we're running on could be a vector
operation, such as a vector addition (as in
[Ichiro](https://github.com/ikwzm)'s
[example](https://github.com/ikwzm/FPGA-SoC-Linux-Example-1-PYNQ-Z1)).

We already have a pre-compiled example for a vector addition in
`pynq_vector_add_generic`. Lets try to execute it:

```sh
/usr/local/bin/pynq_vector_add_generic
```

The output is similar to the above. Since we're using the `noop` plugin, the
result is a dummy result, only for debugging.

```console
user@debian-fpga:~$ /usr/local/bin/pynq_vector_add_generic
2023.01.21-02:49:17.02 - <debug> Initializing vAccel
2023.01.21-02:49:17.02 - <debug> Created top-level rundir: /run/user/1001/vaccel.epcZBL
2023.01.21-02:49:17.02 - <debug> Registered plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function noop from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function sgemm from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function image classification from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function image detection from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function image segmentation from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function image pose estimation from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function image depth estimation from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function exec from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function TensorFlow session load from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function TensorFlow session run from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function TensorFlow session delete from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function MinMax from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function Array copy from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function Vector Add from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function Parallel acceleration from plugin noop
2023.01.21-02:49:17.02 - <debug> Registered function Matrix multiplication from plugin noop
2023.01.21-02:49:17.02 - <debug> Loaded plugin noop from /usr/local/lib/libvaccel-noop.so
2023.01.21-02:49:17.02 - <debug> session:1 New session
Initialized session with id: 1
2023.01.21-02:49:17.02 - <debug> session:1 Looking for plugin implementing fpga_vector_add operation
2023.01.21-02:49:17.02 - <debug> Found implementation in noop plugin
[noop] Calling v_vectoradd for session 1
[noop] Dumping arguments for v_vectoradd:
[noop] len_a: 5 len_b: 5 
9.100000
9.100000
9.100000
9.100000
9.100000
2023.01.21-02:49:17.02 - <debug> session:1 Free session
2023.01.21-02:49:17.02 - <debug> Shutting down vAccel
2023.01.21-02:49:17.02 - <debug> Cleaning up plugins
2023.01.21-02:49:17.02 - <debug> Unregistered plugin noop
```

## Get the PYNQ hardware plugin

Now that we have established that vAccel is working correctly on the board, we
can use the hardware plugin, built for PYNQ. It implements three vector
operations (`vector_add`, `array_copy` and `mmult`).

To get it, we grab the deb from the
[binaries](https://docs.vaccel.org/binaries) page of vAccel:

```sh
wget https://s3.nbfc.io/nbfc-assets/github/vaccelrt/plugins/pynq/main/aarch32/Release-deb/vaccelrt-plugin-pynq-0.1-Linux.deb
dpkg -i vaccelrt-plugin-pynq-0.1-Linux.deb
```

Once installed, it should place a shared object in `/usr/local/lib`: 

```console
$ ls -la /usr/local/lib/arm-linux-gnueabihf/libvaccel-pynq.so 
-rw-r--r-- 1 root root 13144 Dec 25 03:41 /usr/local/lib/arm-linux-gnueabihf/libvaccel-pynq.so
```

We use this shared object as the vAccel plugin and re-run the `pynq_vector_add_generic`
program:

```sh
# Set the path to the vAccel libraries
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

# Set the plugin to PYNQ
export VACCEL_BACKENDS=/usr/local/lib/arm-linux-gnueabihf/libvaccel-pynq.so

# enable Debug
export VACCEL_DEBUG_LEVEL=4 

# Run the vector add example
/usr/local/bin/pynq_vector_add_generic
```

The output should be like below:

```console
2023.01.21-02:56:07.30 - <debug> Initializing vAccel
2023.01.21-02:56:07.30 - <debug> Created top-level rundir: /run/user/0/vaccel.kPHyje
2023.01.21-02:56:07.31 - <debug> Registered plugin fpga_functions
2023.01.21-02:56:07.31 - <debug> Registered function Array copy from plugin fpga_functions
2023.01.21-02:56:07.31 - <debug> Registered function Vector Add from plugin fpga_functions
2023.01.21-02:56:07.31 - <debug> Registered function Parallel acceleration from plugin fpga_functions
2023.01.21-02:56:07.31 - <debug> Registered function Matrix multiplication from plugin fpga_functions
2023.01.21-02:56:07.31 - <debug> Loaded plugin fpga_functions from /usr/local/lib/arm-linux-gnueabihf/libvaccel-pynq.so
2023.01.21-02:56:07.31 - <debug> session:1 New session
Initialized session with id: 1
2023.01.21-02:56:07.31 - <debug> session:1 Looking for plugin implementing fpga_vector_add operation
2023.01.21-02:56:07.31 - <debug> Found implementation in fpga_functions plugin
Calling Vector Add function (FPGA) 1
2.800000
2.100000
8.500000
3.500000
11.299999
2023.01.21-02:56:07.31 - <debug> session:1 Free session
2023.01.21-02:56:07.31 - <debug> Shutting down vAccel
2023.01.21-02:56:07.31 - <debug> Cleaning up plugins
2023.01.21-02:56:07.31 - <debug> Unregistered plugin fpga_functions
```

As we can see, it actually performed the addition on the two vectors. See the relevant snippet from the [code](https://github.com/cloudkernels/vaccelrt/blob/master/examples/pynq_vector_add_generic.c):

```C
[...]
	float a[5] = { 5.0, 1.0, 2.1, 1.2, 5.2 };
	float b[5] = { -2.2, 1.1, 6.4, 2.3, 6.1 };
[...]
	ret = vaccel_fpga_vadd(&sess, a, b, c, len_a, len_b);
```


## Run remote example

To be able to run the above example remotely, we need two things:

1. run the `vaccelrt-agent` as a vAccel application locally
2. run the `pynq_vector_add_generic` program on the remote host, using the relevant
   plugin that enables remote execution (`vsock`). 

### Run the vAccelRT Agent

The vAccelRT agent is essentially a vAccel application that on one side
consumes the vAccel API, and on the other side listens for gRPC requests from
remote hosts. To get it use the following commands:

```sh
wget https://s3.nbfc.io/nbfc-assets/github/vaccelrt/agent/59704ec358de8f68345556a774c60788ac957183/aarch32/release/vaccelrt-agent
chmod +x vaccelrt-agent
```

To expose the above functionality, we use the exact same environment variables,
only this time we run the agent, not the `pynq_vector_add_generic` program:

```sh
# Set the path to the vAccel libraries
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

# Set the plugin to PYNQ
export VACCEL_BACKENDS=/usr/local/lib/arm-linux-gnueabihf/libvaccel-pynq.so

# enable Debug
export VACCEL_DEBUG_LEVEL=4

# set the local endpoint
export VACCEL_AGENT_ENDPOINT=tcp://0.0.0.0:8192

# Run the vAccelRT agent
./vaccelrt-agent -a $VACCEL_AGENT_ENDPOINT
```

The output should be something like the following:

```console
2023.01.21-03:03:34.06 - <debug> Initializing vAccel
2023.01.21-03:03:34.06 - <debug> Created top-level rundir: /run/user/0/vaccel.apxqms
2023.01.21-03:03:34.06 - <debug> Registered plugin fpga_functions
2023.01.21-03:03:34.06 - <debug> Registered function Array copy from plugin fpga_functions
2023.01.21-03:03:34.06 - <debug> Registered function Vector Add from plugin fpga_functions
2023.01.21-03:03:34.06 - <debug> Registered function Parallel acceleration from plugin fpga_functions
2023.01.21-03:03:34.06 - <debug> Registered function Matrix multiplication from plugin fpga_functions
2023.01.21-03:03:34.06 - <debug> Loaded plugin fpga_functions from /usr/local/lib/arm-linux-gnueabihf/libvaccel-pynq.so
vaccel ttRPC server started. address: tcp://0.0.0.0:8192
Server is running, press Ctrl + C to exit
```

### Run the application on the remote host

On the remote host, depending on the architecture and variant we need to setup vAccelRT and the relevant plugin that enables remote execution: `vaccelrt-plugin-vsock`.

Let's assume it's an `x86_64` host. The commands needed to setup vAccel are the following:

```sh
# Get & Install vAccelRT core library
wget https://s3.nbfc.io/nbfc-assets/github/vaccelrt/master/x86_64/Release-deb/vaccel-0.5.0-Linux.deb
dpkg -i vaccel-0.5.0-Linux.deb

# Get & Install the vSock plugin
wget https://s3.nbfc.io/nbfc-assets/github/vaccelrt/plugins/vsock/master/x86_64/Release-deb/vaccelrt-plugin-vsock-0.1.0-Linux.deb
dpkg -i vaccelrt-plugin-vsock-0.1.0-Linux.deb
```

Now we should have the following on `/usr/local/lib`:

```console
$ tree  /usr/local/lib/
/usr/local/lib/
├── libmytestlib.so
├── libvaccel-noop.so
├── libvaccel-python.so
├── libvaccel.so
├── libvaccel-vsock.so

0 directories, 5 files
```

As previously the process to execute the vAccel application is the same, with the only difference that we need to point the plugin to the IP address and port where the vAccelRT Agent listens:

```sh
# Set the path to the vAccel libraries
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

# enable Debug
export VACCEL_DEBUG_LEVEL=4

# Set the plugin to VSOCK
export VACCEL_BACKENDS=/usr/local/lib/libvaccel-vsock.so

# set the IP & port
export VACCEL_VSOCK=tcp://192.168.4.21:8192

# Run the program
/usr/local/bin/pynq_vector_add_generic
```

The output should be something similar to this:

```console
$ /usr/local/bin/pynq_vector_add_generic
2023.01.20-20:16:25.18 - <debug> Initializing vAccel
2023.01.20-20:16:25.18 - <debug> Created top-level rundir: /run/user/0/vaccel.X1hPel
2023.01.20-20:16:25.20 - <debug> Registered plugin vsock
2023.01.20-20:16:25.20 - <debug> vsock is a VirtIO module
2023.01.20-20:16:25.20 - <debug> Registered function sgemm from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function image classification from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function image detection from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function image segmentation from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function image depth estimation from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function image pose estimation from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function TensorFlow session load from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function TensorFlow session delete from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function TensorFlow session run from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function MinMax from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function Array copy from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function Matrix multiplication from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function Vector Add from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function Parallel acceleration from plugin vsock
2023.01.20-20:16:25.20 - <debug> Registered function exec from plugin vsock
2023.01.20-20:16:25.20 - <debug> Loaded plugin vsock from /usr/local/lib/libvaccel-vsock.so
2023.01.20-20:16:25.21 - <debug> [vsock] Initializing session
2023.01.20-20:16:25.21 - <debug> [vsock] New session 1
2023.01.20-20:16:25.21 - <debug> session:1 New session
Initialized session with id: 1
2023.01.20-20:16:25.21 - <debug> session:1 Looking for plugin implementing fpga_vector_add operation
2023.01.20-20:16:25.21 - <debug> Found implementation in vsock plugin
2.800000
2.100000
8.500000
3.500000
11.299999
2023.01.20-20:16:25.22 - <debug> [vsock] Destroying session 1
2023.01.20-20:16:25.22 - <debug> [vsock] Destroying vsock client
2023.01.20-20:16:25.22 - <debug> session:1 Free session
2023.01.20-20:16:25.22 - <debug> Shutting down vAccel
2023.01.20-20:16:25.22 - <debug> Cleaning up plugins
2023.01.20-20:16:25.22 - <debug> Unregistered plugin vsock
```

That's it! we managed to use a PYNQ-Z1 board to run simple operations on the
FPGA fabric from a remote host. In addition to that, the remote host is of
different architecture than the PYNQ board.

## Future steps

As we are far from being experts on Hardware design, we plan to build a more
elaborate example for the FPGA board (e.g. an [Image inference accelerator
using
Tensil](https://www.hackster.io/petrohi/use-tensil-and-pynq-to-run-resnet-20-on-pynq-z1-fpga-board-c71a29))
and use this as a backend for vAccel's Image inference API ;)

Give us a shout at team@cloudkernels.net if you liked it, or visit the
[vAccel](https://vaccel.org) website and drop us a note at
vaccel@nubificus.co.uk for more info!

