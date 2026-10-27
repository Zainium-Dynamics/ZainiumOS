# Zainium OS architecture

This document describes how Zainium OS is put together: the directory layout, how the system view is built at boot, how libraries are found, and how software is installed and upgraded.

## 1. Directory layout

### At runtime

```text
/
├── dev/ proc/ sys/ run/ tmp/      virtual file systems
├── home/                          user data
├── root/                          root's home
├── var -> /overlayer/syshub/var   compatibility link (systemd state, logs)
└── overlayer/
    ├── syshub/            the system view: base + installed software (OverlayFS)
    ├── zexlib/            installed software, package records, flatpak data
    ├── zaisys/            kernel, initramfs, boot loader (read-only)
    ├── agents/            instructions and skills for AI coding agents
    ├── README.zainium     overview of the OS
    ├── architecture.md    this document
    └── zex.md             package manager usage
```

There is no `/usr`, `/etc`, `/bin`, `/sbin`, `/lib` or `/boot` at the root.

### On disk

```text
<root partition>/
├── home/
├── root/
└── overlayer/
    ├── syshub/                         the base system
    │   ├── bin/ sbin/ libexec/         programs
    │   ├── lib/                        libraries (and lib/systemd, lib/udev, …)
    │   ├── include/                    headers
    │   ├── share/                      data files
    │   ├── etc/                        system configuration
    │   ├── var/                        state of base services
    │   ├── x86_64-zainium-linux-musl/  musl: loader, libc, headers, toolchain
    │   ├── drivers/
    │   │   ├── modules/<kernel>/       kernel modules
    │   │   └── hardware/firmwares/     firmware
    │   └── engine/services/            syshub.toml (base package list), zex_services.toml (mirrors)
    ├── zexlib/
    │   ├── union/                      files of installed packages (bin, lib, etc, share, …)
    │   │   └── x86_64-zainium-linux-gnu/  glibc (when installed): loader, libc, headers
    │   ├── work/                       OverlayFS work directory
    │   ├── registry/                   one receipt per installed package, package index, caches, trust keys
    │   └── var/                        data kept outside the union (flatpak's system installation)
    ├── zaisys/
    │   ├── kernel/                     linux-evl, linux-panic, quantra-ramfs.img
    │   └── limine/                     BOOTX64.EFI, limine-bios.sys, limine.conf
    ├── agents/
    ├── README.zainium  architecture.md  zex.md
    └── .syshub-pending/                a staged base upgrade (only while one is waiting)
```

## 2. The layers

| Layer | Path | Contents | Changed by |
| :--- | :--- | :--- | :--- |
| Base | `overlayer/syshub` | The complete OS: musl, toolchain, systemd, desktop, drivers | Base upgrades only, applied at boot |
| Userland | `overlayer/zexlib/union` | Software installed with `zex` | `zex install`, `remove`, `uland --u` |
| Boot | `overlayer/zaisys` | Kernel, initramfs, boot loader | Base upgrades, the installer |
| Agents | `overlayer/agents` | Agent instructions, skills, policies | Admin |

The base never receives a file from the userland. If the userland is deleted, the base still boots on its own.

## 3. How the system view is built at boot

1. **Limine** (UEFI or BIOS) loads `linux-evl` and `quantra-ramfs.img` from `overlayer/zaisys`.
2. **quantra-ramfs** (the initramfs) starts the Plymouth splash, waits for devices, and mounts the root:
   - installed system: the root partition;
   - live ISO: the image `zaisys/zairoot.img` (EROFS, zstd-compressed), loop-mounted read-only.
3. If a base upgrade is staged in `overlayer/.syshub-pending`, it is applied to `overlayer/syshub` now, before anything else uses it. A requested rollback undoes the last applied upgrade at this point instead.
4. The system view is mounted with OverlayFS at `/overlayer/syshub`:
   - lower layers: `overlayer/syshub` (base, first) and `overlayer/zaisys`;
   - upper layer: `overlayer/zexlib/union`, work directory `overlayer/zexlib/work`.

   Programs see one system. A package's files appear in `/overlayer/syshub/bin`, `/overlayer/syshub/lib`, … while they are stored in `zexlib/union`.
5. `zaisys` (read-only), `zexlib`, `agents` and the other entries of `overlayer/` are mounted at their own paths. `/home` and `/root` come from the disk. `/var` becomes a link to `/overlayer/syshub/var`.
6. The root is switched (`pivot_root`) and **systemd** starts as PID 1. It runs **greetd** with **cosmic-greeter**, which starts the **COSMIC** desktop.

Rescue options on the kernel command line: `zainium.overlay=off` boots the base without the userland layer; `rd.break=pre-mount|pre-overlay|pre-pivot` stops in a shell at that step.

### Live ISO and installed system

| | Live ISO | Installed |
| :--- | :--- | :--- |
| Root | `zaisys/zairoot.img`, read-only EROFS | disk partition |
| Installed software (`zexlib`) | in memory, lost at reboot | on disk |
| `/home`, `/root` | in memory over the image | on disk |
| Base upgrades | not available | staged, applied at the next boot |

## 4. Programs and libraries

### musl

The base is built from source against musl with the `x86_64-zainium-linux-musl` toolchain. musl lives in its own directory:

```text
/overlayer/syshub/x86_64-zainium-linux-musl/lib/ld-musl-x86_64.so.1   loader
/overlayer/syshub/x86_64-zainium-linux-musl/lib/libc.so                C library
```

Every program records this loader in its ELF header. The loader searches `/overlayer/syshub/lib`, `/overlayer/syshub/x86_64-zainium-linux-musl/lib` and `/overlayer/zexlib/union/lib`, in that order. Programs also carry a library path (`RPATH`) of `/overlayer/syshub/lib`.

There is no glibc in the base. Base upgrades reject files linked against glibc.

### glibc next to musl

glibc is a userland package (`glibc`, with `glibc-dev` and `glibc-toolchain` for building) and lives in its own directory, as musl does in the base:

| | musl | glibc |
| :--- | :--- | :--- |
| Directory | `/overlayer/syshub/x86_64-zainium-linux-musl` | `/overlayer/zexlib/union/x86_64-zainium-linux-gnu` |
| Loader | `lib/ld-musl-x86_64.so.1` | `lib/ld-linux-x86-64.so.2` |
| C library | `lib/libc.so` | `lib/libc.so.6` |
| Compiler | `gcc`, `g++`, `cc`, `c99` | `x86_64-zainium-linux-gnu-gcc`, `x86_64-zainium-linux-gnu-g++` |
| `ldd` | on `PATH` | in glibc's `bin`, not on `PATH` |

Nothing of glibc goes into the shared `bin`, `lib` or `include` directories, so the compiler, headers, `ldd` and programs of the base stay musl's. Each program runs with the loader it was linked for. A glibc program loads libraries only from glibc's directory (glibc's `libstdc++` and `libgcc_s` are there too); the libraries in `/overlayer/syshub/lib` are built for musl and cannot load into it. glibc's compiled-in paths point at the system's files: users, groups, hosts, DNS and the timezone come from `/overlayer/syshub/etc`, `/bin/sh` is `/overlayer/syshub/bin/sh`. zex installs glibc programs and libraries unchanged, without the musl loader and library path it gives packages from other distributions. Zainium OS is musl-based by default. Whether to build your own software against musl or glibc is the user's choice.

### Programs that expect FHS paths

Software written for other distributions may open `/usr/...`, `/etc/...` or `/bin/...`. `libsys_resolver.so` is preloaded (`LD_PRELOAD`) into systemd services, login shells and PAM sessions. When such a path does not exist, it maps it to the Zainium path. A program built elsewhere whose loader is missing (`/lib64/ld-linux-x86-64.so.2`, `/lib/ld-musl-x86_64.so.1`) is started with this system's copy of that loader. The same library works in musl and glibc programs: it has no libc of its own. It cannot help statically linked or setuid programs, because the loader does not preload into those.

## 5. Software and upgrades

**zex** is the package manager (usage: `/overlayer/zex.md`).

- **Sources, in priority order:** Zainium's own archive (`.zex`), then Void Linux (musl), Alpine Linux, Chimera Linux, and crates.io. The first source with the package is used.
- **Packages from other distributions** are adjusted when installed: program loader and library path (`RPATH`), `pkg-config` prefixes, and script interpreters are rewritten to Zainium paths.
- **Libraries:** before a package is installed, every library its programs need must exist on the system or come from a repository. Otherwise zex asks first.
- **Base protection:** a package never replaces a base file. Files the base already has are left out of the package.
- **Records:** every install writes a receipt (`zexlib/registry/installed/<name>.toml`) listing its files. Removal deletes exactly those files.
- **Base upgrades** (`zex syshub --u`): new base packages are downloaded, verified and staged in `/overlayer/.syshub-pending` (root only, immutable while waiting), then applied by quantra-ramfs at the next boot.
- **Desktop applications** are also available through Flatpak, whose system installation lives in `zexlib/var/lib/flatpak`, and the COSMIC Store.

## 6. Privileges

`elevate` replaces sudo; authentication goes through `elevate-pam`. Even as root, `elevate` refuses `rm`, `rmdir`, `mv`, `cp`, `rsync`, `shred` and `unlink` on `/overlayer/syshub`, `/overlayer/zaisys`, `/overlayer/.syshub-pending` and `/overlayer` itself. The base changes only through zex base upgrades.

## 7. How the OS is built

- Packages are built from recipes (`ZEXBUILD` + `manifest.toml`) in three repositories: `syshub-recipes` (base), `userland-recipes` and `cosmic-zainium` (desktop). CI builds them in a container against the Zainium musl loader, checks every program's loader, and publishes them to `archive.zainiumdynamics.tech`.
- The ISO is made by eclipse-iso-builder: the root tree is packed into a zstd-compressed EROFS image (`zaisys/zairoot.img`), then written with Limine into an ISO 9660 image (Rock Ridge names) that boots on UEFI and BIOS.
