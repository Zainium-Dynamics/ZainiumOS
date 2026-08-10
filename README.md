# ZainiumOS

[![Website](https://img.shields.io/badge/Website-zainiumdynamics.tech-00d9ff?style=flat-square&logo=internetexplorer&logoColor=white)](https://zainiumdynamics.tech)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue?style=flat-square&logo=gnu&logoColor=white)](https://www.gnu.org/licenses/gpl-3.0)
[![Built with Rust](https://img.shields.io/badge/Built%20with-Rust-f74c00?style=flat-square&logo=rust&logoColor=white)](https://www.rust-lang.org)
[![Linux](https://img.shields.io/badge/Linux-Kernel-FCC624?style=flat-square&logo=linux&logoColor=black)](https://kernel.org)
[![LFS](https://img.shields.io/badge/Base-Linux%20From%20Scratch-2c3e50?style=flat-square&logo=linuxserver&logoColor=white)](https://www.linuxfromscratch.org)
[![Platform](https://img.shields.io/badge/Platform-x86__64-6e40c9?style=flat-square&logo=intel&logoColor=white)](https://zainiumdynamics.tech)
[![musl libc](https://img.shields.io/badge/libc-musl-2e7d32?style=flat-square&logoColor=white)](https://musl.libc.org)
[![Status](https://img.shields.io/badge/Status-Active%20Development-2ea44f?style=flat-square)](https://zainiumdynamics.tech)

ZainiumOS is a next-generation operating system built from scratch on a Linux From Scratch (LFS) foundation, targeting **musl libc** rather than glibc. It is not derived from any existing distribution and does not repackage any existing distribution's binaries. The filesystem layout, package management model, environment system, cross-compilation toolchain, and every shipped package are original work, designed and built by **Ali Zain**.

---

## What Is ZainiumOS

Most Linux distributions carry decades of structural compromise — shared directories, mutable system state, dependency graphs that span the entire filesystem, package managers that can silently overwrite critical binaries, and a libc (glibc) that trades size and simplicity for feature breadth. ZainiumOS was built to eliminate all of that.

Starting from LFS, every component of the system is intentional. The filesystem is divided into a permanently clean, immutable base layer and a fully isolated userland layer. These two layers never share the same physical location on disk. They are merged at runtime through a precisely controlled union and symlink model. The result is an operating system where the core can never be broken by anything that happens in userland.

The base system (`syshub`) is compiled entirely from upstream source against **musl libc**, using a purpose-built cross-compilation toolchain (`x86_64-zainium-linux-musl`) — never assembled from another distribution's precompiled binaries. The userland package manager, Zex, sources packages the same way whenever possible, and only falls back to compatible musl binary mirrors (Void, Alpine, Chimera, in that priority order) for the long tail of software Zainium's own archive doesn't carry yet.

---

## Filesystem Architecture

```
/ (Root)
├── dev/
├── home/
├── proc/
├── run/
├── sys/
│
└── overlayer/
    │
    ├── zaisys/                        # Kernel and early-boot environment
    │   └── kernel/                    # Kernel binary and quantra-ramfs
    │
    ├── syshub/                        # Immutable Base OS — read-only at runtime
    │   ├── bin/                       # Core system binaries, cross-toolchain (gcc, clang, rustc, cargo)
    │   ├── sbin/                      # Essential system administration binaries
    │   ├── lib/                       # Base system libraries (musl, LLVM, GStreamer, AppArmor, …)
    │   ├── include/                   # Headers for the base toolchain and libraries
    │   ├── share/                     # Shared data: zoneinfo, man pages, pkgconfig, …
    │   ├── etc/                       # Core system configuration
    │   ├── x86_64-zainium-linux-musl/ # musl sysroot (loader, libc, target headers/libs)
    │   ├── drivers/                   # Hardware modules and system-level driver hooks
    │   │   ├── hardware/              # Firmware, GPU, sensor, peripheral modules
    │   │   ├── system/                # Zex operation drivers (install/detach/upgrade)
    │   │   └── modules/               # Kernel modules
    │   └── engine/                    # Init and service engine
    │       ├── services/              # Zainium System service definitions
    │       └── quantra                # PID 1 init binary
    │
    └── zexlib/                        # Writable Userland — all installed packages land here
        ├── registry/
        │   ├── packages.toml          # Master package state — versions, origins, dependency graph
        │   └── installed/             # Per-package receipt files
        │       ├── nginx.toml
        │       ├── rust.toml
        │       └── ...
        └── union/                     # Active merged layer — symlinked with syshub at runtime
            ├── bin/                   # Installed package binaries + syshub/bin symlinks
            ├── sbin/
            ├── lib/                   # Installed libraries + syshub/lib symlinks
            ├── etc/                   # Userland configuration
            ├── var/                   # Runtime and application data
            └── tmp/                   # Temporary files
```

---

## The Two Layers

### syshub — The Immutable Base

`syshub` is the complete, self-sufficient operating system: musl libc and its dynamic linker, the `x86_64-zainium-linux-musl` cross-toolchain (GCC, LLVM/Clang), coreutils, and system-critical libraries and services (GStreamer, GPGME, libseccomp, AppArmor, and more) — all built from upstream source specifically for this target. It is mounted read-only at runtime.

No package installed by the user ever enters `syshub`. It stays permanently clean. There is no dependency hell possible at the base layer because the base layer is never modified after installation. If the entire `zexlib` userland is deleted or corrupted, the system still boots fully into a working base system using only `syshub`.

### zexlib — The Writable Userland

`zexlib` is where all user-installed software lives. Its `union/` directory is symlinked with `syshub` at runtime, so installed packages appear alongside base system tools transparently. The user sees one unified system. On disk, the two layers are physically separate.

The `union/bin/`, `union/lib/`, and other subdirectories each hold either files extracted by the package manager or symlinks pointing back into `syshub`. The base system files are never duplicated — only referenced. `zexlib` packages are dynamically linked binaries that rely on `syshub` being present at its fixed runtime path, or fully static musl binaries with no runtime dependency on `syshub` at all — both are first-class, source-built targets, never foreign-distribution binaries.

---

## Package Management

ZainiumOS uses **Zex** as its package manager. Zex operates entirely within `zexlib`, and every source it pulls from is resolved through a strict, priority-ordered tier system — never a flat, unordered mirror list:

| Tier | Source | Format | Role |
|------|--------|--------|------|
| 1 | `archive.zainiumdynamics.tech` (Zainium's own core + userland archive) | `.zex` | Primary — native Zainium builds, always tried first |
| 2 | Void Linux musl ports | `xbps` | Power-user fallback, deliberately ranked above Alpine |
| 3 | Alpine Linux (main + community) | `apk` | Embedded/minimal fallback, lowest-priority binary tier |
| 4 | Chimera Linux (main + user) | `adb` | Last-resort binary fallback |
| — | crates.io source-build engine | `cargo` | Rust crates built from source, static musl (`+crt-static`), not a binary mirror at all |

If a package exists in the Tier 1 native archive, Zex uses it and never consults the lower tiers. A package only falls through to Void, then Alpine, then Chimera if Zainium's own archive doesn't carry it yet — and even then, every package that installs into `syshub` itself is still built from upstream source against `x86_64-zainium-linux-musl` by Zex's own build pipeline, never repackaged from a foreign binary. The fallback tiers exist to keep `zexlib` usable on day one for the long tail of userland software, without waiting for Zainium's own archive to carry every package first.

### packages.toml — The Master State File

When a package is installed, Zex writes a record into `zexlib/registry/packages.toml`. This file is the single source of truth for the system's desired state. Each entry records the package name, version, upstream source, and its full dependency graph inline. Dependencies are not resolved at install time by scanning the filesystem — they are declared in the graph at write time, so Zex already knows what is needed before a single file is built or extracted.

When the user runs an upgrade, Zex reads `packages.toml`, rebuilds or fetches the updated sources, and updates the entries and the files in `union/` directly. There is no separate upgrade database. The same file governs both state and update resolution.

### installed/ — Package Receipts

Every package that Zex installs also generates a receipt file at `zexlib/registry/installed/<name>.toml`. The receipt records every file that the package placed into the `union/` layer. When a package is removed, Zex reads only its receipt and deletes exactly those files. Nothing else is touched. No orphaned files, no leftover libraries, no stale configuration.

### Collision Blocking

Before extracting any package, Zex checks whether any of the incoming files conflict with files already present in `union/`. If a conflict is detected, the installation is aborted. Files from `syshub` cannot be overwritten under any circumstance. This guarantees that no package can silently replace a base system binary.

---

## Environment System

ZainiumOS manages the user environment through a dedicated Rust-based environment binary. This replaces the traditional `/etc/profile.d` shell script chain. It sets the `$PATH`, configures aliases, and manages `.bashrc` behavior from a single controlled source. The profile script and `.bashrc` equivalent are both handled by this environment layer rather than being scattered across the filesystem.

---

## Design Principles

ZainiumOS is built on a small set of non-negotiable constraints.

The base system is immutable. User-installed software has no mechanism to reach it. `syshub` is always built from its own upstream source against musl — never repackaged from another distribution. `zexlib` prefers the same, native-first, and only reaches for a musl-compatible binary fallback tier when Zainium's own archive doesn't carry a package yet. The package state is a single file, not a database scattered across `/var/lib`. Every install and every removal is fully reversible because every change is recorded in a receipt. The environment is managed programmatically, not through fragile shell script inheritance. And the entire userland can be wiped and rebuilt without touching the running OS.

**Determinism is ZainiumOS's first rule, and it is not negotiable.** Every build, every install, every upgrade produces the same result from the same inputs, every time. ZainiumOS does not favor any upstream, is not built to compete with any other distribution or the market, and does not exist to chase trends — it exists to be deterministic. Nothing in the system is allowed to compromise that, for any reason.

---

Developer and System Architect: **Ali Zain**

---

## License

ZainiumOS is free software licensed under the **GNU General Public License v3.0 (GPL-3.0)**.

You are free to use, study, modify, and distribute this software under the terms of the GPL v3. Any derivative work must also be distributed under the same license. See the [LICENSE](LICENSE) file for the full license text, or visit [https://www.gnu.org/licenses/gpl-3.0.html](https://www.gnu.org/licenses/gpl-3.0.html).
