---
layout: post
title: Cross-Bootstrap Fedora
draft: true
---

I recently had to assemble linux distribution images to be run in containers and
virtual machines. While most package managers provide tools to bootstrap an
entire distribution into a target directory (e.g., `debootstrap`, `dnf
--installroot`, `zypper`, `pacstrap`, ...), I needed to do that for foreign
architectures. Fortunately, Fedora got me covered!

If you use *`dnf --installroot=/path`*, dnf will perform the given operations in
a separate directory tree, rather than your file-system root. It is easy to use
this with *`dnf install`* to install an entire Fedora distribution into some
custom directory. Unfortunately, RPM allows scripts to be run as part of the
installation process of packages. Those scripts might invoke binaries of the
target architecture as part of the installation. Hence, before we can cross
bootstrap Fedora, we need one more tool: qemu-user-static

The qemu project provides two kinds of emulators:

 * **System Emulators**: These are the commonly known emulators used to emulate
   an entire machine of a given architecture. They can be used to run virtual
   machines of any kind. The binaries are usually called
   **`qemu-system-<arch>`**.

 * **User Emulators**: These emulators are much less known. They emulate the
   linux user-space of your target architecture of choice. That is, they
   execute binaries of foreign architectures on your machine, translating on
   the syscall boundary. Hence, you can run `MIPS` binaries on your `x86_64`
   machine running a normal `x86_64` kernel, as long as you use the
   qemu-user-mips emulator. The binaries are usually called
   **`qemu-<arch>`**.

Fedora provides a package called `qemu-user-static`, which provides statically
linked qemu user-space emulators and hooks them up with the kernel-binfmt
configuration. Hence, with the package installed, you can directly execute
binaries of foreign architectures, and the kernel will use the qemu emulators
to run the binaries. Since the qemu emulators are statically linked, they will
work just fine in chroots as well.

With this in mind, you can simply add `--forcearch=<arch>` to dnf to bootstrap
Fedora in a foreign architecture. For instance, this bootstraps just `bash` and
all its dependencies as 32bit ARM targets:

```
dnf \
        -y \
        --repo=fedora \
        --repo=updates \
        --releasever=27 \
        --forcearch=armv7hl \
        --installroot=/some/path \
        install \
                bash
```

For more information, have a look at Nathaniel McCallum's
[introduction](https://npmccallum.gitlab.io/post/cross-architecture-roots-with-dnf/)
of the `--forcearch` argument to dnf.
