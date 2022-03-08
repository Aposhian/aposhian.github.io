---
layout: post
title:  "Digging into Docker"
date:   2021-01-26 8:30:00 -0600
---

Once you have a basic understanding of the kinds of things that container technologies are doing under the hood, it is helpful to dive into specific tools and how they are structured. This kind of knowledge will enable you to
- make more secure deployments
- understand your options to optimize for your use case

Before we continue, let's get one thing straight: Docker is _not_ the same thing as the more general concept of _containers_. If this is unclear to you, see my post [_What are containers really?_]({% post_url 2021-10-27-what-are-containers %}). Many people see Docker and containers as being synonymous, so it is worth diving into its details a bit more.

# TL;DR;

Docker is not one program: it is a collection of programs that work together. Its client-server architecture poses some problems, as does the default method of setting up docker, which gives out too much `root` access. You can also customize your Docker installation by using different container runtimes.

# The Pieces of Docker

### `docker`: the Docker CLI
This is the user-facing CLI that lets you manage containers on your system. It has commands for building images as well as pushing  them to or pulling them from remote servers. It also enables running and managing containers. However, all of these actions really just send API requests to `dockerd` the Docker server.

### `dockerd`: the Docker daemon (server)
`dockerd` accepts API requests from the Docker CLI as well as other clients (like the [`docker-py`](https://github.com/docker/docker-py) library that powers [`docker-compose`](https://docs.docker.com/compose/)). In fact, the Docker CLI and `dockerd` need not even run on the same computer (see [Docker contexts](https://docs.docker.com/engine/context/working-with-contexts/))! In a default installation of Docker, this process runs as root, so in some ways, `root` access is only an API call away! 😬


### [`containerd`](https://github.com/containerd/containerd): container runtime (high-level)

In modern Docker, `dockerd` doesn’t actually do much work for managing and running containers: it delegates this to another server: `containerd`. In a default install, this also runs as root.

From its own [README](https://github.com/containerd/containerd/blob/aa65faebd73205198f294aff2aa40037c7549fda/README.md), `containerd`'s stated job is to:

>...manage the complete container lifecycle of its host system: image transfer and storage, container execution and supervision, low-level storage and network attachments, etc.

### [`runC`](https://github.com/opencontainers/runc): container runtime (low-level)

Unlike `dockerd` and `containerd`, `runC` is not a daemon or server listening on a socket for connections: it is an executable that runs when you want to run a container. It is basically a wrapper that does the heavy lifting of setting up that fancy namespacing and dedicated filesystem, and then executes the desired program in that environment. In a default install, this runs as `root`, which means that at the top-level operating system level, your application is running as `root`, so if it can find a way out of that sandbox you put it in (which has been [a known vulnerability](https://unit42.paloaltonetworks.com/breaking-docker-via-runc-explaining-cve-2019-5736/)) then it has `root` access to your system, and can do whatever it wants. Yikes!

# Addressing the root access in the room: going rootless

As we can see, a default Docker install hands out a lot of root access, especially if you add your user to the docker group so you don’t have to use `sudo` for every `docker` command (which should be a red flag right there!).

Because of this, a lot of effort has gone into [running Docker in “rootless” mode](https://docs.docker.com/engine/security/rootless/). This is not referring to the user inside the container (from the container’s point of view) but is referring to all the processes mentioned above (`docker`, `dockerd`, `containerd`, and `runC`), and the user from the host point of view.

Another docker flaw: the daemon

Because Docker is fundamentally organized as a client-server architecture, it breaks the standard Linux process hierarchy. We have seen this already by how a REST API was used to escalate user level commands to a daemon running as `root`. However, this also makes it difficult to integrate Docker with systemd, which is integrated into most Linux distributions nowadays.

# Alternative container runtimes

While `runC` is the default container runtime for Docker, you can use other container runtimes to actually launch the processes. Here are few examples, with different use-cases.

### [`crun`](https://github.com/containers/crun)
From its README:
> A fast and low-memory footprint OCI Container Runtime fully written in C.

`crun` is for those looking to get the most out of their containerized system by getting faster startup time for containers, and using less memory to manage the containers.

### [`nvidia-container-runtime`](https://github.com/NVIDIA/nvidia-container-runtime)

A modified version of `runc` that adds support for Nvidia GPUs.

### [Kata Containers](https://katacontainers.io/)

The kata containers runtime actually launches lightweight virtual machines, rather than namespaced processes. This provides a higher level of sandboxing and security.