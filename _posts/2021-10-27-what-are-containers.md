---
layout: post
title:  "What are containers really?"
date:   2021-10-27 21:01:19 -0600
---

Containers have become ubiquitous in software development, especially in cloud deployments. [92% of respondents to the 2020 CNCF survey said that they use containers in production](https://www.cncf.io/wp-content/uploads/2020/11/CNCF_Survey_Report_2020.pdf). But what are containers really, and why has this technology led to one of the biggest software trends over the last decade?

# TL;DR;

[Docker](https://www.docker.com/) isn’t magic: it just packages features of the Linux kernel in a user-friendly way, and it's not the only software that exposes those capabilities. It's also important to understand that containers are not the same as "lightweight VMs". No containerization or virtualization technology is a silver bullet, and the more you understand what they are doing under the hood, the less likely you are to shoot yourself in the foot with that non-silver bullet.

# Background: Common Software Problems

Software is inherently needy, and cannot be trusted.

## Dependency Management

Most software relies on other software to do its job. Deploying versions of software at the same time as the correct versions of their dependencies [is tricky](https://en.wikipedia.org/wiki/Dependency_hell), especially when those dependencies are shared by other software running on that operating system.

There are two primary ways of managing dependencies (aside from not managing them at all 😶):
1. Providing some nifty way to automatically install needed dependencies
2. Bundling the needed dependencies with your software

### 1. Scripting the installation of dependencies
Usually this comes in the form of _manifest_ files that describe the needed dependencies, along with a _package manager_ that can interpret the manifest and install the dependencies from over a network. Examples of this scheme include:

| Package manager | Manifest file |
| --------------- | ------------- |
| [`npm`](https://docs.npmjs.com/cli/v7/commands/npm)           | [`package.json`](https://docs.npmjs.com/cli/v7/configuring-npm/package-json) |
| [`pip`](https://pip.pypa.io/en/stable/)           | [`requirements.txt`](https://pip.pypa.io/en/stable/user_guide/#requirements-files) |
| [Apache Ivy](https://ant.apache.org/ivy/)      | [`ivy.xml`](https://ant.apache.org/ivy/history/latest-milestone/ivyfile.html) |

...and the list goes on and on.

### 2. Bundling software artifacts together
Some examples of software artifacts with bundled dependencies include:
- binaries statically linked to their dependencies
- `.jar` archive files for Java
- `.tar.gz` or `.zip` archive files with executables and configuration files

[Archive files](https://en.wikipedia.org/wiki/Archive_file) can be thought of as essentially a self-contained filesystem that can unpacked wherever you need it.

### 3. ...And a combination of the two

Some deployment formats use manifests and bundling together, primarily to bundle everything needed for that package, while providing a manifest for external dependencies. `.deb` and `.rpm` packages are examples of this.

## Isolation and Sandboxing

> “If debugging is the process of removing software bugs, then programming must be the process of putting them in.”
    - Edsger W. Dijkstra

All software produces unexpected behavior sometimes. It really becomes a problem when that unexpected behavior intentionally (and maliciously) or unintentionally mucks up the computing environment, for itself or for other programs on the system.

Also, if that software that you deploy on a system happens to be compromised by a malicious actor, you don’t want it to gain control of the rest of the system, or gain access to sensitive data somewhere else on the system.

These problems can be mitigated by placing programs in a "sandbox", or a state where their effect on the outside is limited.

There are few common ways to put your software in a safe sandbox:

- Modern operating systems already providing a measure of isolation by [giving each process a dedicated memory address space](https://tldp.org/LDP/tlk/kernel/processes.html).
- Every process runs under a user, so that processes capabilities can be controlled somewhat by using permissions.
- Virtualization: translating programs through a hypervisor to create a [Virtual Machine (VM)](https://en.wikipedia.org/wiki/Virtual_machine).
- [Kernel namespacing](https://en.wikipedia.org/wiki/Linux_namespaces): Linux can run processes in a special way so that in their view of the OS, they can’t see other processes or kernel resources.

It is important to note that sandboxing and communication are constantly in tension. To get around process isolation, there are various methods for [inter-process communication (IPC)](https://en.wikipedia.org/wiki/Inter-process_communication). Containers can be made to share portions of their filesystem with the host.

Moral of the story? Be thoughtful about putting things in boxes, because you will probably want to open them later (although that's not necessarily a bad thing).

![Sandboxing Cycle](https://imgs.xkcd.com/comics/sandboxing_cycle.png)


# Containers: filesystems, cgroups, and namespacing

Docker popularized the concept of _containers_: easily distributable and easily sandboxed applications. While it is not the first player in this space (more generally [OS-level virtualization](https://en.wikipedia.org/wiki/OS-level_virtualization)) (ex: [FreeBSD Jails](https://en.wikipedia.org/wiki/FreeBSD_jail) and [LXC](https://en.wikipedia.org/wiki/LXC)) and definitely not the last (see [Podman](https://podman.io/)). Docker’s design choices make it not ideal for every application, and so other technologies (including those that follow the “container” paradigm) continue to be developed.

Containers solve the dependency management problem by distributing software as _images_, which contain immutable filesystem layers in addition to runtime configuration. These layers are unpacked into a filesystem at runtime, which is then exposed to the programs as the one-and-only filesystem (using [`pivot_root`](https://man7.org/linux/man-pages/man2/pivot_root.2.html) or [`chroot`](https://man7.org/linux/man-pages/man2/chroot.2.html))[^runC], both of which are standard Linux [system calls](https://en.wikipedia.org/wiki/System_call)). Doing this sort of work is supported This enables dependency management (including managing different Linux OS versions! ), as well as a level of sandboxing on the filesystem.

Containers enable sandboxing by automatically setting up all the proper kernel namespacing at runtime, so that applications interact with the kernel and the rest of the system in a sandboxed way.

While still requiring some performance penalty at container startup to set all this up, containers then run as any other process, executing native code on the processor without being translated through a hypervisor or emulator. Because of this, it is more correct to think of containers as "heavily modified host processes" rather than "lightweight virtual machines".

# But wait, I thought containers weren't just for Linux!

Yes, you can run Docker containers on Windows and macOS with [Docker Desktop](https://www.docker.com/products/docker-desktop). However, since Docker uses Linux kernel features to make containers, Docker Desktop has to run a Linux virtual machine to make this happen[^DockerDesktopVM]! So this should be seen as development environment tool, rather than a production scenario.

Also, Docker supports building [Windows containers](https://www.docker.com/products/windows-containers) to be run on Windows hosts. But we don't talk about that. 😉

# References

[^runC]: [From the `runC` spec](https://github.com/opencontainers/runc/blob/master/libcontainer/SPEC.md)
    > A `pivot_root` is used to change the root for the process, effectively jailing the process inside the rootfs... For container's running with a rootfs inside ramfs a `MS_MOVE` combined with a `chroot` is required as `pivot_root` is not supported in `ramfs`.

[^DockerDesktopVM]: 
    > [Some of the magic Docker Desktop takes care of for developers includes... A secure, optimized Linux VM that runs Linux tools and containers](https://www.docker.com/blog/the-magic-behind-the-scenes-of-docker-desktop/)
