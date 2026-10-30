# Zainium OS
[![Website](https://img.shields.io/badge/Website-zainiumdynamics.tech-0052cc?style=flat-square&logo=globe&logoColor=white)](https://zainiumdynamics.tech)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue?style=flat-square&logo=gnu&logoColor=white)](https://www.gnu.org/licenses/gpl-3.0)
[![Built with Rust](https://img.shields.io/badge/Built%20with-Rust-f74c00?style=flat-square&logo=rust&logoColor=white)](https://www.rust-lang.org)
[![Linux](https://img.shields.io/badge/Linux-Kernel-FCC624?style=flat-square&logo=linux&logoColor=black)](https://kernel.org)
[![LFS](https://img.shields.io/badge/Base-Linux%20From%20Scratch-2c3e50?style=flat-square&logo=linuxserver&logoColor=white)](https://www.linuxfromscratch.org)
[![Platform](https://img.shields.io/badge/Platform-x86__64-6e40c9?style=flat-square&logo=intel&logoColor=white)](https://zainiumdynamics.tech)
[![musl libc](https://img.shields.io/badge/libc-musl-2e7d32?style=flat-square&logoColor=white)](https://musl.libc.org)
[![DHS](https://img.shields.io/badge/Layout-Dualayer%20Hierarchy%20Standard-0b7285?style=flat-square)](https://zainiumdynamics.tech/dhs)
[![Release](https://img.shields.io/badge/Release-Rolling-8250df?style=flat-square)](https://zainiumdynamics.tech/news)
[![Status](https://img.shields.io/badge/Status-Active%20Development-2ea44f?style=flat-square)](https://zainiumdynamics.tech)

**Zainium OS** is an independent, enterprise-grade Linux operating system — built from scratch and immutable by design. Its core stays intact whatever you install, upgrades apply atomically, and software for musl and glibc runs side by side. It is not derived from any other distribution, and it is a rolling release: install once and stay current, with no version numbers.

It runs its own build of the Linux kernel (`linux-zainium`), which follows the rolling release like everything else, systemd as init and the COSMIC desktop on x86_64. The base system is compiled from upstream source with Zainium's own `x86_64-zainium-linux-musl` toolchain. Upstream software such as systemd, COSMIC, Mesa and musl is built from its source, not taken from another distribution's binaries. Original to ZainiumOS are the filesystem layout and its layer model, the package manager **zex**, the initramfs **quantra-ramfs**, the root tool **elevate**, the installer, the ISO builder and the package recipes.

---

## What is different

1. **A base that never breaks.** The base (`syshub`) and the software you install (`zexlib/union`) are stored apart on disk and joined at boot with OverlayFS into one view at `/overlayer/syshub`. Nothing you install is written into the base, and if anything goes wrong the base boots on its own (`zainium.overlay=off`).
2. **Even root cannot change a single byte of the base.** On an installed system every base file is sealed with fs-verity on the first boot, and the kernel refuses to rewrite it — for root too, and even when the disk is mounted directly. `elevate` replaces sudo and refuses `rm`, `mv`, `cp`, `rsync`, `shred` and `unlink` on the base, the boot files and a staged upgrade.
3. **Kernel-enforced security from the first boot.** Kernel lockdown (integrity), AppArmor profiles, Landlock, Yama and IPE are on, and every kernel module is signed. `.zex` packages are signed (Ed25519) and checked (BLAKE3).
4. **Atomic upgrades at boot.** Base upgrades are downloaded, verified and staged while the system runs, then applied by quantra-ramfs at the next boot, before anything uses the base. Every step is journaled, and the last applied upgrade can be rolled back.
5. **musl at the core, glibc when you need it.** musl lives in `/overlayer/syshub/x86_64-zainium-linux-musl/`; `elevate zex install glibc` puts glibc in a directory of its own, so prebuilt glibc software runs next to the musl base without touching it.
6. **No dependency hell.** zex installs Zainium's own signed `.zex` packages first and can take packages from Void Linux (musl), Alpine Linux and Chimera Linux, adjusting them to Zainium paths. It resolves every package and every shared library before installing anything, never leaves a half-installed package and never replaces a base file.
7. **No `/usr`, `/etc`, `/bin` or `/lib` at the root.** The whole OS lives under `/overlayer`, in layers with one owner each, published as an open standard: the [Dualayer Hierarchy Standard](https://zainiumdynamics.tech/dhs) (DHS). For software that expects the usual paths, `sys_resolver` maps them to Zainium's at run time and starts prebuilt programs with the right loader.

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
    │   ├── union/     files of installed packages (the OverlayFS upper layer); union/var holds
    │   │              their data, such as flatpak's system installation
    │   ├── work/      OverlayFS work directory
    │   └── registry/  one receipt per installed package, index, caches, trust keys
    ├── zaisys/        kernel, initramfs, Limine boot loader (read-only)
    ├── agents/        instructions and skills for AI coding agents
    └── README.zainium  architecture.md  zex.md
```

## Boot

1. **Limine** (UEFI or BIOS) loads the kernel (`linux-zainium`) and `quantra-ramfs.img`.
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

Zainium OS is musl-based. Each C library lives in a directory named after its target triplet: musl in `/overlayer/syshub/x86_64-zainium-linux-musl`, and glibc, once installed with `elevate zex install glibc`, in `/overlayer/zexlib/union/x86_64-zainium-linux-gnu` — its loader, libraries, headers and its own `x86_64-zainium-linux-gnu-gcc`/`g++`. Nothing of glibc enters the shared `bin`, `lib` or `include`, so the two never conflict. Both read the same users, DNS and time zone from the system's `etc`, and each program runs with the loader it was linked for; binaries built on other distributions start through the right loader automatically. `gcc` builds for musl; use the glibc compiler, or `sys-resolver set`, to build for glibc.

## Documentation

- [zainiumdynamics.tech/docs](https://zainiumdynamics.tech/docs) — the full documentation
- [Dualayer Hierarchy Standard](https://github.com/Zainium-Dynamics/dualayer-hierarchy-standard) — the filesystem layout as an open specification
- [README.zainium](README.zainium) — overview of the OS, as shipped in `/overlayer`
- [architecture.md](architecture.md) — layers, boot, libraries, packages and upgrades in detail
- [zex.md](zex.md) — package manager usage

## Status

Zainium OS is in active development. Not every package from Void, Alpine or Chimera works after adjustment; zex refuses packages whose libraries no repository provides unless you confirm. Static or setuid programs that hard-code `/usr`, `/etc` or `/bin` paths are not covered by `sys_resolver`. fs-verity stops changes to the content of base files, but root can still delete or replace a whole file on a directly mounted disk; Secure Boot and an IPE policy are next on the roadmap.

## Design goal

Determinism is Zainium Dynamics' first rule: the same inputs should produce the same build, install and upgrade, every time.

---

Created and maintained by **Ali Zain** ([Zainium Dynamics](https://zainiumdynamics.tech)).

## License

Copyright (C) 2026 Ali Zain / Zainium Dynamics.

Zainium OS is free software licensed under the **GNU General Public License v3.0 (GPL-3.0)**. You are free to use, study, modify and distribute it under the terms of the GPL v3; derivative works must be distributed under the same license. See [LICENSE](LICENSE), or [gnu.org/licenses/gpl-3.0](https://www.gnu.org/licenses/gpl-3.0.html).
