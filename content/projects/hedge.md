---
title: "Hedge: a lightweight Hypervisor for the Edge"
date: 2023-01-01T14:41:40+02:00
layout: post
projects: "nbfc"
---

Running applications in the Cloud has changed the way users develop and ship
their code. During the past 15 years, applications were deployed in the cloud
using Virtual Machines (VMs) following the Infrastructure-as-a-Service (IaaS)
model. Users choose the setup of their virtual hardware, install their
preferred OS and deploy their application / service on top of that VM.

For the past 5 years, the community has given rise to [other] [approaches]
which were quickly adopted by Cloud vendors, towards solutions that follow the
paradigm of Platform-, Software-, and Function-as-a-Service (PaaS, SaaS, and
FaaS respectively). These approaches offer performance and flexibility
improvements over IaaS by decoupling the application from the infrastructure.
Providing a common OS stack, maintained by the provider and optimized for
specific hardware is much more efficient than exposing a generic interface of
virtual hardware. 

Additionally, users seek to maximize the number of requests handled,
while at the same time minimize request / response latency. Apart from the
cloud paradigm, Edge computing is slowly adopting these modes of operation,
especially in the context of IoT and [5G]. 

In the IaaS case, the burden of orchestrating and optimizing the systems
software stack running on top of virtual hardware is passed to the user, while
in the other cases, the vendor exposes a customized interface, tailored to the
application / service offered. 

The [Cloud-native] concept emerged from this trend, as a need to reduce bloated interfaces and
abstractions that introduced significant overhead for application deployment
and execution. Containers played an important role towards cloud-native
embracement; they have revolutionized deployment by facilitating application
packing and dependency tracking, and reducing the overheads of execution;
however, this comes at the cost of [security and isolation]
As a result, cloud vendors fall back to generic virtualization techniques:
Microservice offerings are essentially VMs running the vendor's custom systems
stack, exposing a language runtime, a specific service such as a DBMS, a LAMP
stack or just container host-side systems software. For instance, to provide a
secure Serverless environment where users deploy their functions at will, cloud
providers either: (a) spawn a VM per tenant, install their Serverless backends
there, and keep it hot while the user submits functions to be executed; (b)
spawn VMs which host containers per tenant, with the necessary software
installed, and execute the user function there; or (c) spawn
microVMs[lambda] per tenant where isolation is provided by the microVM
monitor[firecracker].

This is the placeholder for Hedge, our hypervisor for the edge


[other]: https://doi.org/10.1007/978-3-319-67425-4_12
[approaches]: https://doi.org/10.1109/MS.2015.11
[5G]: https://doi.org/10.1109/MCE.2016.2590118
[cloud-native]: https://www.cncf.io/about/faq
[security and isolation]: http://doi.acm.org/10.1145/3274694.3274720

