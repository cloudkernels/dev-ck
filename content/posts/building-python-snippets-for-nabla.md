---
title: "How to build a python snippet for running in a Nabla container"
date: 2019-02-23T18:17:51+02:00
lastmod: 2019-02-24T12:37:57+0200

---

In our previous posts, we saw how to [build the toolchain for a Nabla container]({{<relref
"building-nabla-aarch64.md">}}), and also how we can use this toolchain to
[run applications as unikernels using Nabla]({{<relref "nabla-containers-aarch64.md">}}).

In this post, we will be focusing on the steps we need to take into running something
actually useful using Nabla. More specifically, we will go through all the steps for
building Python3 into a [Rumprun][0] unikernel, suitable for running in a Nabla container,
and cooking a filesystem that includes a Python script that we wish to run within.

We will be using the [rumprun-packages][1] git repository, which contains a collection
of frameworks and applications that we can build on top of the Rumprun infrastructure.
We have started doing some work on updating rumprun-packages, so that we can build
and bake applications using the [recent updates](https://github.com/nabla-containers/rumprun/commits/solo5)
done by the Nabla people for Solo5 support in Rumprun and more specifically
the `spt` and `hvt` Solo5 tenders. This is work in progress and we will be porting
more packages from rumprun-packages to work on top of the upstream toolchain, both
for x86 and [aarch64]({{<relref "nabla-containers-aarch64.md">}}).

### Building Python3.5 as a unikernel ###

Once we have [built]({{<relref "building-nabla-aarch64.md">}}) the rumprun toolchain
we can build and bake Python3.5 in a Rumprun unikernel following these steps:

```
git clone https://github.com/cloudkernels/rumprun-packages.git
cd rumprun-packages

# Setting up the rumprun-packages build environment
cp config.mk.dist config.mk

# If we are building for aarch64 we should also run:
echo "RUMPRUN_TOOLCHAIN_TUPLE=aarch64-rumprun-netbsd" >> config.mk

cd python3

# Build for the spt target
make python.spt

# Build for the hvt target
make python.hvt
```

### Packing our Python script ###

We still need to be able to pack our python script so that we can run it within the 
unikernel, i.e. the equivalent of doing `python my_script.py`?

Remember, in the world of unikernels we do not have access to a terminal,
our application *is* our Linux box / VM / container.

We have two problems to solve:

1. Make our script available within the unikernel
2. Pre-setup our environment with all the package dependencies our script needs in
order to execute.

We will solve these issues by packing our script along with all its dependencies
inside a disk image which we will later provide to the unikernel at run time.

Here's how we do this:

```
# We 're sill under rumprun-packages/python3.

# this will be were we will install the python environment and our script
mkdir -p python/lib

# Our previous step has fetched all the basic Python environment
# under: ./build/pythondist/lib/python3.5
cp -r build/pythondist/lib/python3.5 python/lib/

# We add the script to Python's site-packages
cp myscript.py python/lib/python3.5/site-packages/

# And we prepare our packages dependencies
pyvenv-3.5 newpackage-env
source newpackage-env/bin/activate
pip install a_python_package
deactivate
cp -r newpackage-env/lib/python3.5/site-packages/* python/lib/python3.5/site-packages/

# Now we have everything we need under 'python', so we create the disk image
genisoimage -l -r -o disk.iso python
```

That's it! disk.iso contains all the necessary environment to run our script.

```
solo5-spt --disk=disk.iso --net=tap0 python.spt \
'{"cmdline":"python.spt -m myscript","env":"PYTHONHOME=/python","net":{"if":"ukvmif0","cloner":"True","type":"inet","method":"static","addr":"10.0.0.2","mask":"16"},"blk":{"source":"etfs","path":"/dev/ld0a","fstype":"blk","mountpoint":"/python"}}
```

We have created a Docker image in order to automate the previous procedure of
building Python as a unikernel and preparing the disk iso with our script and its
dependencies, so that instead of running the following steps you can simply do
something like:

```
docker run --rm -v $(pwd):/build cloudkernels/python3-build disk.iso myscript.py requirements.txt
```

where `requirements.txt` includes the dependencies of `myscript.py` in the form of one
package per line (this is, essentially, whatever running `pip freeze` on your python
project directory would produce).

You can find the Docker image on [Docker hub](https://hub.docker.com/r/cloudkernels/python3-build) and on [github](https://github.com/cloudkernels/python3-build).

A working example is found below. Please note that this version includes a hack to hardcode the dns server in the dummy rootfs as we haven't yet patched the config part of rumprun to include a cmdline option for dns.

We will use a simple requests example. The files needed are the python snippet and requirements.txt.

requests_main.py:
```
import requests

r = requests.get('https://www.example.com')
print(r.status_code)
print(r.text)
```

requirements.txt:
```
requests
```

Now run the command to bake the necessary python dependencies:

```
# docker run --rm -v $(pwd):/build cloudkernels/python3-build:x86_64_dns disk.iso requests_main.py requirements.txt
[...]
  7.12% done, estimate finish Sat Feb 23 18:56:49 2019
 14.25% done, estimate finish Sat Feb 23 18:56:49 2019
 21.35% done, estimate finish Sat Feb 23 18:56:49 2019
 28.48% done, estimate finish Sat Feb 23 18:56:49 2019
 35.59% done, estimate finish Sat Feb 23 18:56:49 2019
 42.70% done, estimate finish Sat Feb 23 18:56:49 2019
 49.81% done, estimate finish Sat Feb 23 18:56:51 2019
 56.94% done, estimate finish Sat Feb 23 18:56:50 2019
 64.04% done, estimate finish Sat Feb 23 18:56:50 2019
 71.16% done, estimate finish Sat Feb 23 18:56:50 2019
 78.27% done, estimate finish Sat Feb 23 18:56:50 2019
 85.38% done, estimate finish Sat Feb 23 18:56:50 2019
 92.51% done, estimate finish Sat Feb 23 18:56:50 2019
 99.61% done, estimate finish Sat Feb 23 18:56:50 2019
Total translation table size: 0
Total rockridge attributes bytes: 714249
Total directory bytes: 1473102
Path table size(bytes): 4710
Max brk space used 678000
70274 extents written (137 MB)

```
And invoke the unikernel using the following command:

```
# ./solo5-hvt --mem=64 --disk=disk.iso --net=tap0 python.hvt '{"cmdline":"python.hvt -m requests_main","env":"PYTHONHOME=/python","net":{"if":"ukvmif0","cloner":"True","type":"inet","method":"static","addr":"10.0.0.2","mask":"16", "gw":"10.0.0.1"},"blk":{"source":"etfs","path":"/dev/ld0a","fstype":"blk","mountpoint":"/"}}'
solo5-hvt: python.hvt: Warning: phdr[0] requests WRITE and EXEC permissions
solo5-hvt: WARNING: Tender is configured with HVT_DROP_PRIVILEGES=0. Not dropping any privileges.
solo5-hvt: WARNING: This is not recommended for production use.
            |      ___|
  __|  _ \  |  _ \ __ \
\__ \ (   | | (   |  ) |
____/\___/ _|\___/____/
Solo5: Memory map: 64 MB addressable:
Solo5:   reserved @ (0x0 - 0xfffff)
Solo5:       text @ (0x100000 - 0x73ae37)
Solo5:     rodata @ (0x73ae38 - 0x8cdcd7)
Solo5:       data @ (0x8cdcd8 - 0xb5b93f)
Solo5:       heap >= 0xb5c000 < stack < 0x4000000
rump kernel bare metal bootstrap

[   1.0000000] Copyright (c) 1996, 1997, 1998, 1999, 2000, 2001, 2002, 2003, 2004, 2005,
[   1.0000000]     2006, 2007, 2008, 2009, 2010, 2011, 2012, 2013, 2014, 2015, 2016, 2017,
[   1.0000000]     2018 The NetBSD Foundation, Inc.  All rights reserved.
[   1.0000000] Copyright (c) 1982, 1986, 1989, 1991, 1993
[   1.0000000]     The Regents of the University of California.  All rights reserved.

[   1.0000000] NetBSD 8.99.25 (RUMP-ROAST)
[   1.0000000] total memory = 26824 KB
[   1.0000000] timecounter: Timecounters tick every 10.000 msec
[   1.0000080] timecounter: Timecounter "clockinterrupt" frequency 100 Hz quality 0
[   1.0000090] cpu0 at thinair0: rump virtual cpu
[   1.0000090] root file system type: rumpfs
[   1.0000090] kern.module.path=/stand/amd64/8.99.25/modules
[   1.0200090] mainbus0 (root)
[   1.0200090] timecounter: Timecounter "bmktc" frequency 1000000000 Hz quality 100
[   1.0200090] ukvmif0: Ethernet address 5e:ac:bf:a1:15:09
[   1.0732133] /dev//dev/ld0a: hostpath XENBLK_/dev/ld0a (137 MB)
mounted tmpfs on /tmp

=== calling "python.hvt" main() ===

200
<!doctype html>
<html>
<head>
    <title>Example Domain</title>

    <meta charset="utf-8" />
    <meta http-equiv="Content-type" content="text/html; charset=utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <style type="text/css">
    body {
        background-color: #f0f0f2;
        margin: 0;
        padding: 0;
        font-family: "Open Sans", "Helvetica Neue", Helvetica, Arial, sans-serif;
        
    }
    div {
        width: 600px;
        margin: 5em auto;
        padding: 50px;
        background-color: #fff;
        border-radius: 1em;
    }
    a:link, a:visited {
        color: #38488f;
        text-decoration: none;
    }
    @media (max-width: 700px) {
        body {
            background-color: #fff;
        }
        div {
            width: auto;
            margin: 0 auto;
            border-radius: 0;
            padding: 1em;
        }
    }
    </style>    
</head>

<body>
<div>
    <h1>Example Domain</h1>
    <p>This domain is established to be used for illustrative examples in documents. You may use this
    domain in examples without prior coordination or asking for permission.</p>
    <p><a href="http://www.iana.org/domains/example">More information...</a></p>
</div>
</body>
</html>

rumprun: call to ``sigaction'' ignored

=== main() of "python.hvt" returned 0 ===

=== _exit(0) called ===
[   1.8632722] rump kernel halting...
[   1.8632722] syncing disks... done
[   1.8632722] unmounting file systems...
[   1.9953910] unmounted tmpfs on /tmp type tmpfs
[   1.9967528] unmounted /dev//dev/ld0a on / type cd9660
[   1.9967528] unmounted rumpfs on / type rumpfs
[   1.9967528] unmounting done
halted
Solo5: solo5_exit(0) called
```

Please note that for this to work, we have setup tap0 with an ip address of
10.0.0.1 and have setup NAT in the host for the guest to access the network.

### Building the Nabla container ###

Now baking the nabla container is a walk in the park after the above steps. You
can have a look in our [previous]({{<relref "build-a-nabla-docker-image.md">}})
post or the relevant [repo](https://github.com/cloudkernels/nabla-base), or if you're a bit lazy here's a quick summary:

Just clone [this](https://github.com/cloudkernels/nabla-base) repo:

```
git clone https://github.com/cloudkernels/nabla-base
```

add the needed files in the rootfs directory:
```
mount -o loop disk.iso /mnt
cp -avf /mnt/* nabla-base/rootfs/
umount /mnt
```

add the seccomp tender binary:

```
cp python.spt nabla-base/python.nabla
```

Replace myprog.nabla with python.nabla in the Dockerfile (careful, the runtime
expects to find a file ending in .nabla so make sure to keep the extension
name).

And build your nabla container image using the following command:

```
cd nabla-base
docker build -f Dockerfile -t python3-requests-nabla .
``` 

Assuming you have setup runnc correctly, spawning the container is as easy as:

```
docker run --rm --runtime=runnc python3-requests-nabla -m requests_main
```

Note the boot command line -- it has to match the 'cmdline':'' parameter in the json string used above.

That's it folks!

Give it a try and let us know what you think!

[0]: http://rumpkernel.org
[1]: https://github.com/cloudkernels/rumprun-packages
