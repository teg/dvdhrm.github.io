---
layout: post
name: Goodbye Gnu-EFI!
categories: [fedora]
tags: [efi, uefi, boot, gnu, gnuefi, gnu-efi]
---

The recommended way to link [UEFI](https://www.uefi.org/) applications on linux
was until now through
*GNU-EFI*, a toolchain provided by the *GNU Project* that bridges from the
*ELF* world into *COFF/PE32+*. But why don't we compile directly to native
UEFI? A short dive into the past of *GNU Toolchains*, its remnants, and a
surprisingly simple way out.

The *Linux World* (and many *UNIX Derivatives* for that matter) is modeled
around
[ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format). With
statically linked languages becoming more prevalent, the impact of the ABI
diminishes, but it still defines properties far beyond just how to call
functions. The ABI your system uses also effects how compiler and linker
interact, how binaries export information (especially symbols), and what
features application developers can make use of. We have become used to ELF,
and require its properties in places we didn't expect.

UEFI does not use ELF. For all that matters,
[UEFI follows Microsoft Windows](https://www.uefi.org/uefi). This means, UEFI
uses
[*COFF/PE32+*](https://www.microsoft.com/whdc/system/platform/firmware/PECOFF.mspx)
(or short *PE+*). If we compile binaries for UEFI, they must target *PE+*. And
the *GNU Compiler Collection* can do this… somewhat.

Conceptually, *GCC* supports many languages, ABIs, targets, and architectures
in a single code-base. Technically, though, every compiled instance of *GCC*
compiles from one language to one target. Your compiler that takes *C* and
produces *x86-64* is actually specific to *x86_64-pc-linux-gnu*. You cannot
tweak it to compile UEFI binaries. Instead, you need another instance of *GCC*,
one that takes *C* and produces *x86_64-windows-msvc*. You probably know this
combination under the name *MinGW*.

But this is not what *GNU* went for. Instead, to what still puzzles me to this
day, the *GNU* project decided against using its own software and instead
produced something named
[*GNU-EFI*](https://sourceforge.net/projects/gnu-efi/). The goal of *GNU-EFI*
is to allow writing UEFI applications using the common *GNU Toolchain* (meaning
you compile *ELF* binaries for *Linux*). They achieve this by linking a
*PE+ Stub*, which at runtime performs required relocations, parameter
translations, and jumps into the *ELF* application. You effectively write a
free-standing *Linux Application*, add a wrapping layer and then execute it on
*UEFI*. It works, but is needlessly complex.

*Is this really the best way to compile for UEFI?* **Not anymore!**

The *LLVM* toolchain (*clang* compiler plus *lld* linker) combines all
supported targets in a single toolchain, offering a target selector `--target`
to let *LLVM* know what to compile for. So as long as you have *clang* and
*lld* installed, you can compile native UEFI binaries just like normal local
compilation:

```sh
# Normal local compile+link
$ clang \
        $CFLAGS \
        -o OBJECT \
        -c [SOURCES…]
$ clang \
        $LDFLAGS \
        -o BINARY \
        [OBJECTS…]
```

To make this compile for UEFI targets, you simply set:

```sh
CFLAGS+= \
        --target x86_64-unknown-windows \
        -ffreestanding \
        -fshort-wchar \
        -mno-red-zone

LDFLAGS+= \
        --target x86_64-unknown-windows \
        -nostdlib \
        -Wl,-entry:efi_main \
        -Wl,-subsystem:efi_application \
        -fuse-ld=lld-link
```

The two things special are `--target <TRIPLE>` and `--fuse-ld=<LINKER>`. The
former instructs both compiler and linker to produce *COFF/PE32+* objects
compatible to the *Microsoft Windows Platform* (which matches the UEFI
platform). The latter selects the linker to use. Mind you, using the default
linker will very likely fail (default being *ld* or *ld-gold*). Currently, you
either have to use *lld-link* (*PE+* backend of the *LLVM* linker), or you need
a version of *GNU-ld* compiled for a *PE+* toolchain. I recommend *LLVM lld*.

Voilà! No need for *GNU-EFI*, no need to mess with separated toolchains. With
*LLVM* you get all this through your local toolchain.

If you use *Meson Build*, the
[**c-efi**](https://c-util.github.io/c-efi) project even provides you an
example *cross-file*. A native meson C project can then be compiled for UEFI by
nothing more than passing `--cross-file x86_64-unknown-uefi` to `meson`. See
its
[sources](https://github.com/c-util/c-efi/blob/master/src/x86_64-unknown-uefi.mesoncross.ini)
for details.

The [**c-efi**](https://c-util.github.io/c-efi) project also provides the
protocol contants and definitions from the UEFI specification, so you don't
have to extract them yourself.
