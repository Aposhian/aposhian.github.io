---
layout: post
title:  "Comparison of software packaging methods"
date:   2021-10-27 21:01:19 -0600
---

## TL;DR;
All packaging formats are filesystem trees in disguise. Different "package managers" perform function

## Preface
Here is the big question: how do I create and collect software from multiple sources into applications and systems that I can run?

All of these solutions are designed to turn a directed graph of software source projects into executable filesystem trees. Why filesystems? Because that is the language of Unix-like operating systems. Programs are files. Programs load libraries that are files. Programs read configuration data that are files. The minimal set of things required for most programs to run is a specific filesystem structure and content.

## Concepts
There are four interrelated concepts:
- [Build system](#build-system)
  - How do you build the dependencies you want?
- [Dependency management](#dependency-management)
  - How do you get the dependencies that you want?
- [Packaging format](#packaging-format)
  - What format do you package the dependencies?
- [Application runtime](#application-runtime)
  - A framework for running applications

Many projects bring one or more of these functions under one umbrella. For example, Flatpak is both a tool for managing dependencies, and a specific packging format.

### Classifying technologies

| Project     | Build system         | Dependency Management | Packaging format   | Application runtime |
| ----------- | -------------------- | --------------------- | ------------------ | ------------------- |
| `apt`       | ❌                   | ✅                    | ✅ `DEB` packages  | ❌                  |
| `dnf`       | ❌                   | ✅                    | ✅ `RPM` packages  | ❌                  |
| `flatpak`   | ❌                   | ✅                    | ✅                 | ✅                  |
| `docker`*   | ℹ️ Buildkit          | ✅                    | ✅**               | ✅                  |
| `snap`      | ❌                   | ✅                    | ✅                 | ✅                  |
| `conan`     | ℹ️ Meta-build system | ✅                    | ✅                 | ❌                  |
| `nix`       | ℹ️ Meta-build system | ✅                    | ✅                 | ✅                  |
| `colcon`    | ℹ️ Meta-build system | ❌                    | ❌                 | ❌                  |
| `cmake`     | ✅                   | ❌                    | ❌                 | ❌                  |
| `pip`       | ❌                   | ✅                    | ✅                 | ❌                  |
| `cargo`     | ✅                   | ✅                    | ✅ "crates"        | ❌                  |
| `npm`       | ✅                   | ✅                    | ✅                 | ❌                  |
| `rpm-ostree`| ❌                   | ✅                    | ✅                 | ✅                  |
| AppImage    | ❌                   | ❌                    | ✅                 | ✅                  |

*Docker is called out specifically, but its paradigm is also found in Podman and other projects for managing and running "containers"

**Docker does have its own packaging format, but also works with OCI format

## Build system
Most build systems are programming language-specific, or are at least designed with a particular ecosystem in mind. Build systems like CMake and NPM are extensible enough that they can be adapted to fit different needs, but other projects take a different approach: that of a _meta build system_.

### Meta build systems

## Dependency Management

### Basic features
- Downloading dependencies from hosted repositories
- Deduplication and caching of downloaded dependencies
- Dependency resolution (what is needed for what)

### Advanced features
- Delta downloads (only download what changed)
- Atomic updates (impossible to leave the system in a bad state)
- Content trust (verifying that the package comes from who you think it does)
- Content integrity (verifying that you downloaded what you asked for)
- Dependency sharing (don't duplicate bandwidth and storage for dependencies that multiple things use)

| Project     | Delta downloads | Atomic updates | Content trust | Content integrity | Dependency sharing |
| ----------- | --------------- | -------------- | ------------- | ----------------- | ------------------ |
| `apt`       | ❌              | ❌             | ✅            | ✅                | ✅ by package      |
| `dnf`       | ✅ by file      | ❌             | ✅            | ✅                | ✅ by package      |
| `flatpak`   | ✅ by file      | ✅             | ✅            | ✅                | ✅ by runtime      |
| `docker`    | ✅ by layer     | ✅             | ✅            | ✅                | ✅ by layer        |
| `snap`      | ✅ binary       | ✅             | ✅            | ✅                | ❌                 |
| `conan`     | ❌              | ❌             | ❌            | ✅                | ✅ by package      |
| `nix`       | ❌              | ✅             | ❌            | ✅                | ✅ by package      |
| `pip`       | ❌              | ❌             | ❌            | ✅                | ✅ by package      |
| `cargo`     | ❌              | ❌             | ❌            | ✅                | ✅ by package      |
| `npm`       | ❌              | ❌             | ✅            | ✅                | ✅ by package      |
| `rpm-ostree`| ✅ by file      | ✅             | ✅            | ✅                | ❌                 |
| AppImage    | ❌              | ✅             | ✅            | ❌                | ❌                 |

## Technologies

### Just distribute a tar file
Distributing a tar file is a filesystem image that is either self-contained, or contains a script to move its contents into various places in the filesystem. There is no inherent dependency management or resolution. There is no delta-diff updates.

### Docker and OCI containers
docker (and OCI containers in general) are based on the idea of union filesystems. Artifacts are distributed as a sequence of layers, which are filesystem diffs (changesets) that are merged together at runtime to create the desired filesystem tree for operation. This may rely on a number of underlying storage systems, but usually Docker uses overlayFS in the Linux kernel to accomplish this.

Each layer is addressed with a digest, so when you download a collection of layers (an image), tools like Docker are smart enough to only download layers that have changed. So that is not the bottleneck. Here are bottlenecks:
Docker images are usually built with Dockerfiles, which have a dumb, linear caching model by default (no dependency metadata other than the order of operations)
Since it uses overlayFS, it may be infeasible to have a layer for every dependency, since there is a hard limit at 128 layers

Docker pull is smart. But docker build, even with buildkit, is not as smart. Since `RUN` steps allow running arbitrary code, they are assumed to have a requirement on the exact state of the filesystem. That means that if any layers before a RUN command change, then the RUN command must be rerun.

### Flatpak
Flatpak is “OStree for desktop applications”. However, since it is tailored to work with desktops, the non-graphical case is not supported.

### OStree
OStree is “Git for operating systems”. It is primarily used for bootable images of a system. However, flatpak adapts it for userspace, so it must be possible to use it in other ways…

https://witekio.com/blog/ostree-tutorial-system-updates/

### rpm-ostree
Ostree but with packages

### DEB and RPM
DEB and RPM packages are very similar to OCI container layers: they contain a filesystem changeset. However, they also have metadata that places them in a dependency graph. And they support execution of arbitrary scripts on install or uninstall, etc.

Apparently dnf does deltas for download, whereas apt downloads the whole deb?

https://michael.stapelberg.ch/posts/2019-07-20-hooks-and-triggers/

https://news.ycombinator.com/item?id=29835245


### Nix

Nix manages caching of binary derivations and only rebuilding what is “needed”, even though it takes the most conservative approach of rebuilding all downstream dependencies from something that changed.

### Snaps
While snaps are a non-starter since they don’t have private hosting for a package store, I don’t know how they store data.

Snaps use squashfs images, which are mounted at boot time, regardless of when the program is run.

### AppImage
Does not have any dependency sharing between AppImages. “one app = one file”

squashfs mounted with fuse.

https://docs.appimage.org/reference/architecture.html

https://en.wikipedia.org/wiki/SquashFS

Language dependency managers

Language package managers achieve a lot of functionality at the cost of flexibility: packages must be a in a specific format, and generally must be packaged for use by a single language. These include:
Cargo (Rust)
Pull all dependencies into a local tree, which then compiles all dependencies
Npm and Yarn (JavaScript)
Pull down all dependencies into a tree at `node_modules`, which then can be accessed by the program
Conan (C++)
?
Pip (Python)
Install packages into a path for python packages, which is generally only exposed to python packages.

Note: rosdep is not a package manager. It is a unified interface to system package managers + pip on different OSes.

OS block replication (Disk Images)
One way to distribute software to a dedicated device is to ship entire images. However, this doesn’t support OTA updates, or many other features.
