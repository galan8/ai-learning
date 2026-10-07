# Hosts SBOM

## 1. What it covers

A **host SBOM** (or VM SBOM) documents all software components and dependencies within a virtual machine or physical host: the operating system, system libraries, middleware, installed agents, container runtimes, and any application containers running on it.

It is the widest of the three scopes.

```
   ┌───────────────────────────────────────────────┐
   │  HOST / VM                                    │
   │                                               │
   │   ┌─────────────┐  ┌─────────────┐            │
   │   │ Container A │  │ Container B │  ← each has │
   │   └─────────────┘  └─────────────┘    its own  │
   │                                        SBOM    │
   │   Container runtime (docker, containerd)      │
   │   Middleware, agents, monitoring              │
   │   System libraries                            │
   │   Operating system packages                   │
   │   Kernel                                      │
   │   ─────────────────────────────────           │
   │   Hypervisor (where applicable)               │
   └───────────────────────────────────────────────┘
```

The structure is close to a container SBOM, with two additions: **the hypervisor layer** where one exists, and **everything installed on the host outside any container**, which is typically where monitoring agents, SSH daemons, orchestration components and administrative tooling live.

## 2. Generating one

```bash
# Inventory the root filesystem of a host
trivy rootfs --format cyclonedx --output host-sbom.json /

# Or scan a mounted filesystem or VM image
trivy fs --format cyclonedx --output vm-sbom.json /mnt/vm-image
```

## 3. Why it matters for AI infrastructure

The host layer is where several documented AI incidents actually happened, and it is the layer AI teams are least likely to own.

**ShadowRay** is the clearest example. The Ray framework's exposed Jobs API allowed arbitrary command execution across AI cluster nodes, and on AWS could be used to retrieve cloud credentials from instance metadata. Nothing about that attack touched a model. It was **host and infrastructure level**, exploited on machines that happened to be running AI workloads.

Two host layer characteristics specific to AI:

1. **GPU drivers and toolkits** (NVIDIA driver, CUDA, container toolkit) are privileged, kernel adjacent components with their own vulnerability history. They rarely appear in application inventories.
2. **Orchestration and cluster components** (Ray, Kubernetes, Slurm, MLflow) frequently ship with permissive defaults and management interfaces that assume a trusted network.

**The principle: an AI system is not just a model and its framework. It is also every machine that runs them, and those machines are ordinary infrastructure with ordinary vulnerabilities.**

## 4. The three scopes together

| Scope | Covers | Misses |
| ----- | ------ | ------ |
| **Application** | Code and its dependencies | OS, runtime, host |
| **Container** | Application plus base image and OS packages | Host, hypervisor, GPU drivers |
| **Host** | Everything on the machine, including the hypervisor | The models and data themselves |

All three together still miss one thing, which is the subject of lesson 17: **none of them describes the model**.

## 5. Summary

1. A host SBOM inventories everything on a VM or physical machine, including the hypervisor and out of container software
2. Generate with `trivy rootfs` or by scanning a VM image
3. AI infrastructure adds GPU drivers, CUDA toolkits and cluster orchestration to the host attack surface
4. ShadowRay demonstrates that AI systems are compromised at this layer, not only at the model layer
5. Application, container and host SBOMs nest; none of them covers the model itself
