# ZainiumOS

ZainiumOS is an independent Linux operating system by [Zainium Dynamics](https://zainiumdynamics.tech). It is built from a Linux From Scratch foundation on musl libc, with its own `x86_64-zainium-linux-musl` toolchain, and is not derived from any other distribution. Upstream software such as the Linux kernel, systemd, musl and the COSMIC desktop is compiled from source; the filesystem layout, the package manager zex, the initramfs quantra-ramfs, the root tool elevate, the installer, the ISO builder and the package recipes are our own work.

ZainiumOS takes a different approach from other distributions. There is no `/usr`, `/etc` or `/bin` at the root: the whole system lives under `/overlayer`. The base system and the software you install are kept apart on disk and joined at boot with OverlayFS, so nothing you install is ever written into the base. Base upgrades are staged while the system runs and applied at the next boot, and the last one can be undone. zex installs native packages first and can also use Void, Alpine and Chimera packages, adjusting them to Zainium's paths. The base is musl-only and keeps musl in its own directory, so glibc can be installed next to it without conflict.

You can read more in [README.zainium](README.zainium), how the system is put together in [architecture.md](architecture.md), and how to use the package manager in [zex.md](zex.md). ZainiumOS is in active development. Our first rule is determinism: the same inputs should give the same build, install and upgrade, every time.

Created and maintained by Ali Zain. Copyright (C) 2026 Ali Zain / Zainium Dynamics. ZainiumOS is free software under the [GNU General Public License v3.0](LICENSE).
