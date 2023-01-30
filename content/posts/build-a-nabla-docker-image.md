---
title: "Build a Nabla Docker Image"
date: 2019-02-23T14:41:40+02:00
---

In this post, we go through the basic steps for containerizing a unikernel
application and running it on nabla runnc. Checkout [nabla
containers](https://github.com/nabla-containers) and in
particular [runnc](https://github.com/nabla-containers/runnc).

### How to

In order to build a docker image for nabla containers, we have to build:

- the nabla toolstack
- the unikernel image
- the docker image

#### Build the nabla toolstack

There's an informative blog post on how to build the nabla rumprun toolstack
[here](https://blog.cloudkernels.net/posts/building-nabla-aarch64/). Now that
our aarch64 changes are upstream, the process should be as easy as the
following:

```
git clone https://github.com/nabla-containers/rumrpun
cd rumprun
git submodule update --init
make
. obj/config-PATH.sh
```
and you should end up with the toolstack (${ARCH}-rumprun-netbsd-) in your PATH variable.


Alternatively you can use one of the (unofficial) docker images with the
toolstack embedded at /usr/local:

- [x86_64-rumprun-netbsd-](https://hub.docker.com/r/cloudkernels/debian-rumprun-build)
- [aarch64-rumprun-netbsd-](https://hub.docker.com/r/cloudkernels/debian-rumprun-build)

You can run these containers using the following command:

```
docker run --runtime=runc --rm -v ${PWD}:/build -it cloudkernels/debian-rumprun-build:${ARCH} /bin/bash
```

where ${ARCH} could be x86_64 or aarch64.

#### Build the unikernel Image

building the image is as easy as running the following:

```
${ARCH}-rumprun-netbsd-gcc myprog.c -o myprog-rumprun
```

```
rumprun-bake solo5-spt myprog.spt myprog-rumprun
```

So we have ourselves an .spt file (a unikernel). To try it out, assuming you
have a [solo5-spt](https://github.com/Solo5/solo5) binary lying around you can
do the following:

```
solo5-spt ./myprog.spt
```

Asumming the unikernel image is successful, its time to construct the docker image.

#### Build the docker image

Rumprun is a quirky unikernel framework, with a number of assumptions we can't
ignore. Thus, in order to get the unikernel running correctly when in a nabla
container environmnent, we need to be careful and include the correct stubs for
a dummy root filesystem. So, clone [this
repo](https://github.com/cloudkernels/nabla-base), which contains the basic
rootfs from rumprun's lib/librumprunfs_base/rootfs, and a template Dockerfile
shown below:

```
FROM scratch
COPY myprog.spt /myprog.nabla
COPY rootfs/etc /etc
ENTRYPOINT [ "myprog.nabla" ]
```

Copy the spt file in this directory and run:

```
docker build -f Dockerfile -t myprog-nabla:${ARCH} .
```

You should end up with a local docker image named myprog-nabla, tagged with
your current architecture variant (x86_64 or aarch64).

Assuming you have setup [runnc](https://github.com/nabla-containers/runnc)
correctly, spawning the container is as easy as:

```
docker run --rm --runtime=runnc myprog-nabla:${ARCH}
```

