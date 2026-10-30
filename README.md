Zainium OS
==========

Zainium OS is an independent Linux operating system by Zainium Dynamics. It
is not based on any other distribution. It uses its own Linux kernel
(linux-zainium), the musl C library, systemd as init and the COSMIC desktop.
It runs on x86_64, it is a rolling release, and it is licensed under GPL-3.0.

This repository holds the documentation that ships with the system, in
/overlayer:

  README.zainium    what Zainium OS is and how it differs
  architecture.md   how the system is put together
  zex.md            package manager usage

The website has the full documentation:

  https://zainiumdynamics.tech/docs


The layers
----------

There is no /usr, /etc, /bin or /lib at the root. The whole system lives
under /overlayer, one directory per owner:

  /overlayer/syshub    the base: programs, libraries, configuration
  /overlayer/zexlib    software you install, and the package records
  /overlayer/zaisys    the kernel, the initramfs and the boot loader
  /overlayer/agents    instructions for AI coding agents

At boot, quantra-ramfs joins the base and your software with OverlayFS into
one view at /overlayer/syshub. Nothing you install is written into the base,
and the base always boots on its own (zainium.overlay=off). The layout is
published as the Dualayer Hierarchy Standard.


Security
--------

On an installed system every base file is sealed with fs-verity: the kernel
refuses to change it, for root too. Kernel lockdown, AppArmor, Landlock,
Yama and IPE are on from the first boot, and every kernel module is signed.
elevate replaces sudo and refuses rm, mv and cp on the base and the boot
files.


Upgrades
--------

Zainium OS has no version numbers. elevate zex update keeps everything
current. Base upgrades are staged while you work and applied at the next
boot, before anything uses the base; the last one can be undone. The kernel
rolls the same way (elevate zex kernel --u), with the previous kernel kept
as the fallback entry.


musl and glibc
--------------

The base is built against musl. glibc can be installed into a directory of
its own (elevate zex install glibc), so prebuilt glibc software runs next to
the musl base without touching it.


Main components
---------------

  Boot loader      Limine (UEFI and BIOS)
  Kernel           linux-zainium
  Initramfs        quantra-ramfs
  Init             systemd
  Login            greetd with cosmic-greeter
  Desktop          COSMIC
  Packages         zex
  Root access      elevate, elevate-pam


Copyright (C) 2026 Zainium Dynamics. Licensed under GPL-3.0.
