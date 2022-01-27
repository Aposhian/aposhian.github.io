---
layout: post
title:  "What is an operating system?"
date:   2021-01-28 8:30:00 -0600
---

An operating system is like a landlord. It doesn't really care what it's tenants do, as long as they stay in their space and don't break things.

When the tenants need access to the underlying infrastructure, like plumbing, electricity, etc., they need to call in their landlord to solve their problem. Similarly, processes are not trusted to have direct access to the filesystem, network interfaces, or memory management. Instead, they must transfer control to the operating system by making a **system call**.

Like a landlord may delegate maintenance work to contractors, the operating system is typically organized into modular **drivers**.

# Processor modes
Processors that can host an operating system have different modes, or [**protection rings**](https://en.wikipedia.org/wiki/Protection_ring). A CPU core can only be in one mode at a time.

# Deep dive: Windows vs Unix-like Operating Systems

Windows works differently than Unix-like operating systems in that it is a **hybrid kernel**, that is, some of the operating system runs in kernelspace, and some runs in userspace.