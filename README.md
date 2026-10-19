# ZainiumOS
[![Website](https://img.shields.io/badge/Website-zainiumdynamics.tech-0052cc?style=flat-square&logo=globe&logoColor=white)](https://zainiumdynamics.tech)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue?style=flat-square&logo=gnu&logoColor=white)](https://www.gnu.org/licenses/gpl-3.0)
[![Built with Rust](https://img.shields.io/badge/Built%20with-Rust-f74c00?style=flat-square&logo=rust&logoColor=white)](https://www.rust-lang.org)
[![Linux](https://img.shields.io/badge/Linux-Kernel-FCC624?style=flat-square&logo=linux&logoColor=black)](https://kernel.org)
[![LFS](https://img.shields.io/badge/Base-Linux%20From%20Scratch-2c3e50?style=flat-square&logo=linuxserver&logoColor=white)](https://www.linuxfromscratch.org)
[![Platform](https://img.shields.io/badge/Platform-x86__64-6e40c9?style=flat-square&logo=intel&logoColor=white)](https://zainiumdynamics.tech)
[![musl libc](https://img.shields.io/badge/libc-musl-2e7d32?style=flat-square&logoColor=white)](https://musl.libc.org)
[![Status](https://img.shields.io/badge/Status-Active%20Development-2ea44f?style=flat-square)](https://zainiumdynamics.tech)

ZainiumOS is an independent Linux operating system by Zainium Dynamics, built from a Linux From Scratch foundation on **musl libc**. It is not derived from any other distribution. It runs the Linux kernel (`linux-evl`), systemd as init and the COSMIC desktop on x86_64.

The base system is compiled from upstream source with Zainium's own `x86_64-zainium-linux-musl` toolchain. Upstream software such as systemd, COSMIC, Mesa and musl is built from its source, not taken from another distribution's binaries. Original to ZainiumOS are the filesystem layout and its layer model, the package manager **zex**, the initramfs **quantra-ramfs**, the root tool **elevate**, the installer, the ISO builder and the package recipes.

---

## What is different

1. **No `/usr`, `/etc`, `/bin` or `/lib` at the root.** The whole OS lives under `/overlayer`. For software that still expects those paths, the `libsys_resolver` library maps them to the Zainium paths when the requested path does not exist (dynamically linked, non-setuid programs).
2. **The base and installed software are separate layers.** The base (`syshub`) and the software you install (`zexlib/union`) are stored apart on disk and joined at boot with OverlayFS into one view at `/overlayer/syshub`. Nothing you install is written into the base, and the userland can be wiped without touching it.
3. **Base upgrades happen at boot.** They are downloaded and staged while the system runs, then applied by quantra-ramfs at the next boot, before the system is mounted. The last applied upgrade can be undone at the next boot.
4. **musl only, in its own directory.** musl lives in `/overlayer/syshub/x86_64-zainium-linux-musl/`. There is no glibc in the base, and base upgrades reject glibc-linked files.
5. **One package manager for several sources.** zex installs native `.zex` packages first and can fall back to Void Linux (musl), Alpine Linux and Chimera Linux packages, adjusting them to Zainium paths. It checks every package's library needs before installing and never replaces a base file.

---

## Filesystem layout

```text
/
├── dev/ proc/ sys/ run/ tmp/      virtual file systems
├── home/   root/                  user data
├── var -> /overlayer/syshub/var   compatibility link
└── overlayer/
    ├── syshub/        the system view: base + installed software (OverlayFS)
    │   ├── bin/ sbin/ libexec/ lib/ include/ share/ etc/ var/
    │   ├── x86_64-zainium-linux-musl/   musl loader, libc, headers, toolchain
    │   ├── drivers/                     kernel modules and firmware
    │   └── engine/services/             base package list, mirror configuration
    ├── zexlib/
    │   ├── union/     files of installed packages (the OverlayFS upper layer)
    │   ├── work/      OverlayFS work directory
    │   ├── registry/  one receipt per installed package, index, caches, trust keys
    │   └── var/       data kept outside the union (flatpak's system installation)
    ├── zaisys/        kernel, initramfs, Limine boot loader (read-only)
    ├── agents/        instructions and skills for AI coding agents
    └── README.zainium  architecture.md  zex.md
```

## Boot

1. **Limine** (UEFI or BIOS) loads `linux-evl` and `quantra-ramfs.img`.
2. **quantra-ramfs** starts the Plymouth splash, mounts the root (a disk partition, or the live ISO's zstd-compressed EROFS image), applies a staged base upgrade if one is waiting, and builds the OverlayFS view.
3. **systemd** starts as PID 1 and runs **greetd** with **cosmic-greeter**, which starts the **COSMIC** desktop.

## Packages: zex

| Tier | Source | Format |
|------|--------|--------|
| 1 | `archive.zainiumdynamics.tech`, Zainium's own archive | `.zex` |
| 2 | Void Linux (musl) | `.xbps` |
| 3 | Alpine Linux | `.apk` |
| 4 | Chimera Linux | `.apk` (ADB) |
| — | crates.io | Rust crates |

All sources are queried at once and the first one in priority order that has the package is used. Every install writes a receipt listing its files, and removal deletes exactly those files. Desktop applications are also available through Flatpak and the COSMIC Store. Full usage: [zex.md](zex.md).

## musl and glibc

ZainiumOS is musl-based. If a user installs glibc, its loader and libraries go into the `lib` and `bin` directories under different file names from musl's (`ld-linux-x86-64.so.2` and `libc.so.6`, against musl's `ld-musl-x86_64.so.1` and `libc.so`), so the two do not conflict. Each program runs with the loader it was linked for. Building your own software against musl or glibc is your choice.

## Documentation

- [README.zainium](README.zainium) — overview of the OS, as shipped in `/overlayer`
- [architecture.md](architecture.md) — layers, boot, libraries, packages and upgrades in detail
- [zex.md](zex.md) — package manager usage

## Status

ZainiumOS is in active development. Not every package from Void, Alpine or Chimera works after adjustment; zex refuses packages whose libraries no repository provides unless you confirm. Static or setuid programs that hard-code `/usr`, `/etc` or `/bin` paths are not covered by `libsys_resolver`.

## Design goal

Determinism is Zainium Dynamics' first rule: the same inputs should produce the same build, install and upgrade, every time.

---

Created and maintained by **Ali Zain** ([Zainium Dynamics](https://zainiumdynamics.tech)).

## License

Copyright (C) 2026 Ali Zain / Zainium Dynamics.

ZainiumOS is free software licensed under the **GNU General Public License v3.0 (GPL-3.0)**. You are free to use, study, modify and distribute it under the terms of the GPL v3; derivative works must be distributed under the same license. See [LICENSE](LICENSE), or [gnu.org/licenses/gpl-3.0](https://www.gnu.org/licenses/gpl-3.0.html).
