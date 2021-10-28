---
layout: post
title:  "What are containers really?"
date:   2021-10-25 21:28:08 -0600
---

Containers have become ubiquitous in software development, especially in cloud deployments. [92% of respondents to the 2020 CNCF survey said that they use containers in production](https://www.cncf.io/wp-content/uploads/2020/11/CNCF_Survey_Report_2020.pdf). But what are containers really, and why has this technology led to one of the biggest software trends over the last decade?

# TL;DR;

[Docker](https://www.docker.com/) isn’t magic: it just packages features of the Linux kernel in a user-friendly way. It's also important to understand that containers are not the same as "lightweight VMs". In addition to Docker, there are other technologies available, each with their pros and cons. No containerization or virtualization technology is a silver bullet, and the more you understand what they are doing under the hood, the less likely you are to shoot yourself in the foot with that non-silver bullet.

# Background: Common Software Problems

Software is inherently needy, and cannot be trusted.

## Needy: dependency management is hard

Most software relies on other software to do its job. Deploying versions of software at the same time as the correct versions of their dependencies [is tricky](https://en.wikipedia.org/wiki/Dependency_hell), especially when those dependencies are shared by other software running on that operating system.

There are two primary ways of managing dependencies (aside from not managing them at all 😶):
1. Providing some nifty way to automatically install needed dependencies
2. Bundling the needed dependencies with your software

### 1. Scripting the installation of dependencies
Usually this comes in the form of _manifest_ files that describe the needed dependencies, along with a _package manager_ that can interpret the manifest and install the dependencies from over a network. Examples of this scheme include:

| Package manager | Manifest file |
| --------------- | ------------- |
| `npm`           | `package.json` |
| `pip`           | `requirements.txt` |
| Apache Ivy      | `ivy.xml` |

And this list goes on and on.

### 2. Bundling software artifacts together
Some examples of software artifacts with bundled dependencies include:
- binaries statically linked to their dependencies
- `.jar` archive files for Java
- `.tar.gz` or `.zip` archive files with executables and configuration files

Archive files can be thought of as essentially a self-contained filesystem that can unpacked wherever you need it.

## Buggy: bugs can be fatal, and malicious programs can bite

All software produces unexpected behavior sometimes. It really becomes a problem when that unexpected behavior intentionally (and maliciously) or unintentionally mucks up the computing environment, for itself or for other programs on the system.

Also, if that software that you deploy on a system happens to be compromised by a malicious actor, you don’t want it to gain control of the rest of the system, or gain access to sensitive data somewhere else on the system.

### Sandboxing/Isolation

Tired of programs messing things up for other programs, or gaining access to data that they shouldn’t?

Put your programs in isolation, so their effect on the outside system is limited!

However, beware of the sandbox cycle, since sandboxing often means closing avenues of communication…

![Sandboxing Cycle](https://imgs.xkcd.com/comics/sandboxing_cycle.png)

Sandboxing can be accomplished with:

- Modern operating systems already providing a measure of isolation by [giving each process a dedicated memory address space](https://tldp.org/LDP/tlk/kernel/processes.html). 
- Operating systems implementing user permissions, so that processes cannot perform certain tasks or access certain files without the proper permissions.
- Virtualization: translating programs through a hypervisor to create a [Virtual Machine (VM)](https://en.wikipedia.org/wiki/Virtual_machine).
- [Kernel namespacing](https://en.wikipedia.org/wiki/Linux_namespaces): Linux can run processes in a special way so that in their view of the OS, they can’t see other processes or kernel resources

# Containers: filesystems, cgroups, and namespacing

Docker popularized the concept of _containers_: easily distributable and easily sandboxed applications. While it is not the first player in this space (more generally [OS-level virtualization](https://en.wikipedia.org/wiki/OS-level_virtualization)) (ex: [FreeBSD Jails](https://en.wikipedia.org/wiki/FreeBSD_jail) and [LXC](https://en.wikipedia.org/wiki/LXC)) and definitely not the last (see [Podman](https://podman.io/), many people see Docker and containers as being synonymous, so it is worth diving into its details a bit more.

Docker solved dependency management by distributing software as images, which contain immutable filesystem layers in addition to runtime configuration. These layers are unpacked at runtime with a method similar to [`chroot`](https://en.wikipedia.org/wiki/Chroot), and are exposed to the programs as the one-and-only filesystem. This enables dependency management (including managing different Linux OS versions), as well as a level of sandboxing on the filesystem.

Docker solved sandboxing by automatically setting up all the proper kernel namespacing at runtime, so that applications interact with the kernel and the rest of the system in a sandboxed way

While still requiring some performance penalty at container startup to set all this up, containers then run as any other process, executing native code on the processor without being translated through a hypervisor or emulator. Because of this, it is more correct to think of containers as "heavily modified host processes" rather than "lightweight virtual machines".

However, Docker’s design choices make it not ideal for every application, and so other technologies (including those that follow Docker’s “container” paradigm) continue to be developed.

# Digging into Docker

In order to understand Docker, it is important to understand that it is not one piece of software, but a collection of programs that work together.

## The Pieces of Docker

### `docker`: the docker CLI
This is the user-facing CLI that lets you manage containers on your system. It has commands for building images as well as pushing  them to or pulling them from remote servers. It also enables running and managing containers. However, all of these actions really just send API requests to `dockerd` the Docker server.

### `dockerd`: the Docker daemon (server)
`dockerd` accepts API requests from the Docker CLI as well as other clients (like the [`docker-py`](https://github.com/docker/docker-py) library that powers [`docker-compose`](https://docs.docker.com/compose/)). In fact, the Docker CLI and `dockerd` need not even run on the same computer (see [Docker contexts](https://docs.docker.com/engine/context/working-with-contexts/))! In a default installation of Docker, this process runs as root, so in some ways, `root` access is only an API call away! 😬


### [`containerd`](https://github.com/containerd/containerd): container runtime (high-level)

In modern Docker, `dockerd` doesn’t actually do much work for managing and running containers: it delegates this to another server: containerd. In a default install, this also runs as root.

From its own [README](https://github.com/containerd/containerd/blob/aa65faebd73205198f294aff2aa40037c7549fda/README.md), `containerd`'s stated job is to:

>...manage the complete container lifecycle of its host system: image transfer and storage, container execution and supervision, low-level storage and network attachments, etc.

### [`runC`](https://github.com/opencontainers/runc): the container runtime (low-level)

Unlike `dockerd` and `containerd`, `runC` is not a daemon or server listening on a socket for connections: it is an executable that runs when you want to run a container. It is basically a wrapper that does the heavy lifting of setting up that fancy namespacing and dedicated filesystem, and then executes the desired program in that environment. In a default install, this runs as `root`, which means that at the top-level operating system level, your application is running as `root`, so if it can find a way out of that sandbox you put it in (which has been [a known vulnerability](https://unit42.paloaltonetworks.com/breaking-docker-via-runc-explaining-cve-2019-5736/)) then it has `root` access to your system, and can do whatever it wants. Yikes!

## Addressing the root access in the room: going rootless

So we can see that a default Docker install hands out a lot of root access, especially if you add your user to the docker group so you don’t have to use sudo for every Docker command (which should be a red flag right there!)

Because of this, a lot of effort has gone into [running Docker in “rootless” mode](https://docs.docker.com/engine/security/rootless/). This is not referring to the user inside the container, from the container’s point of view, but is referring to all the processes mentioned above, and the user from the host point of view.
Another docker flaw: the daemon

Because Docker is fundamentally organized as a client-server architecture, it breaks the standard Linux process hierarchy. We have seen this already by how a REST API was used to escalate user level commands to a daemon running as `root`. However, this also makes it difficult to integrate Docker with systemd, which is integrated into most Linux distributions nowadays.