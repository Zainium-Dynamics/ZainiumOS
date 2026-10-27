# zex — Zainium OS package manager

`zex` installs, removes, updates and audits packages. It is one multicall binary at `/overlayer/syshub/bin/zex`. Anything that changes the system runs as root through `elevate`:

```bash
elevate zex <command> [options] [packages]
zex help
```

## Where things go

| Layer | Path | Changed by |
| :--- | :--- | :--- |
| Base OS (immutable) | `/overlayer/syshub` | `zex syshub --u` only, applied at the next boot |
| Userland packages | `/overlayer/zexlib/union` (seen merged into `/overlayer/syshub`) | `zex install`, `remove`, `uland --u` |
| Installed-package receipts | `/overlayer/zexlib/registry/installed/<name>.toml` | zex |
| Mirrors, tiers, signature mode | `/overlayer/syshub/engine/services/zex_services.toml` | admin |
| Base package list | `/overlayer/syshub/engine/services/syshub.toml` | base upgrades |

## Repositories (tiers)

Every mirror is asked at once; the first one, in priority order, that has the package wins.

| Tier | Source | Format | Install flag |
| :--- | :--- | :--- | :--- |
| T1 | Zainium | `.zex` | `--zainium` |
| T2 | Void Linux (musl) | `.xbps` | `--void` |
| T3 | Alpine Linux | `.apk` | `--alpine` |
| T4 | Chimera Linux | `.apk` (ADB) | `--chimera` |
The flags come from the mirror ids in `zex_services.toml`: each id's first word (`--void`), or the full id for one mirror (`--alpine-community`).

## Installing

```bash
elevate zex install htop                 # best tier by priority
elevate zex install vim git curl         # several at once
elevate zex install --void wget          # only from Void
elevate zex reinstall flatpak            # download and install again, even if current
elevate zex install neovim --dry-resolve # show what would be installed, change nothing
elevate zex install foo --cache-only     # use only already-downloaded packages
elevate zex install foo --force          # overwrite files another package owns
```

What zex checks while installing:

- **Signatures and integrity**, per tier (Ed25519 + BLAKE3 for `.zex`, the upstream RSA schemes for Void, Alpine and Chimera).
- **Shared libraries**: every program and library in the package is checked for the libraries it links (`NEEDED`). A missing library is installed from the repository that provides that exact version. If no repository has it, zex lists it and asks before installing anyway (default: no).
- **Base files win**: a file the base OS already has is left out of the package. The package installs without it, and zex lists what it skipped. If every file is already in the base, the package is refused as "already provided by the base OS".
- **Shared caches are rebuilt, not copied**: files every package would overwrite (`share/info/dir`, icon, MIME, desktop and schema caches, font indexes) are dropped from packages and regenerated after the install.
- **Base names are blocked**: names the base provides (`coreutils` → userutils, `musl` → syshub) can't be installed into userland and count as satisfied when another package depends on them.
- **Conflicts**: a file owned by another userland package stops the install unless `--force` is given.

### Reproducible installs

```bash
elevate zex install musl --lock-out musl.lock.toml   # pin the exact resolved graph
elevate zex install --from-lock musl.lock.toml       # replay it elsewhere
zex install --verify-lock musl.lock.toml             # report drift, install nothing
```

### Local files

```bash
elevate zex local-install ./myapp-1.0.zex      # also .apk and .xbps; short: zex -i <file>
zex extract ./myapp-1.0.zex --output ./out     # unpack only; short: zex -e <file> -o <dir>
```

## Removing

```bash
elevate zex remove htop
elevate zex remove vim htop --dry-run   # show what would be removed
elevate zex remove foo --yes            # no confirmation
```

zex deletes exactly the files recorded in the package's receipt at install time, then any directories left empty. Base packages can't be removed.

## Updating

```bash
elevate zex update                 # refresh indexes, then userland and base
elevate zex update --yes           # no confirmation
elevate zex update --uland-only    # userland packages only
elevate zex update --syshub-only   # base only
elevate zex uland --u              # userland upgrade on its own
elevate zex syshub --u             # base upgrade on its own
elevate zex refresh                # refresh the indexes of all tiers
```

Base upgrades are staged in `/overlayer/.syshub-pending` and applied by quantra-ramfs at the next boot, before the overlay is mounted. Every step is journaled, so an interrupted upgrade recovers the next time zex runs. On a live session the base is a read-only image and can't be upgraded.

## Looking around

```bash
zex search "fuzzy finder"          # search every tier
zex search ripgrep --tier void     # one tier
zex search vim --installed         # installed packages only
zex inspect ripgrep                # version, files, shared libraries, integrity
zex depends ripgrep                # dependency tree
zex why libssl                     # what depends on it
zex history                        # install/remove/upgrade timeline
zex history --package musl --limit 5
```

## Maintenance and recovery

```bash
elevate zex doctor                 # audit registry, links, the base, musl and glibc
elevate zex doctor --fix           # repair what it can
elevate zex sweep --dry-run        # preview cache and temp cleanup
elevate zex sweep --aggressive
zex rollback --list                # transactions
elevate zex rollback <tx_id>       # undo one
elevate zex pin gcc                # hold a package at its version
elevate zex unpin gcc
zex pins
```

## Hooks

- **Package hooks**: a package's `manifest.toml` may set `[hooks] post_install = "<command>"`. zex runs it as root with `/overlayer/syshub/bin/sh` after the transaction, with the Zainium tool path. For example, flatpak and the COSMIC Store start the Flathub catalog download this way.
- **System hooks**: `*.hook` files in `/overlayer/syshub/etc/zex/hooks.d/` run after installs and removals of matching packages:

  ```toml
  [hook]
  name = "rebuild-something"
  trigger = "post-install"          # or "post-remove"
  packages = ["fonts-*"]            # globs; empty = every package
  exec = "fc-cache -s"
  timeout_secs = 30
  ```

## For agents

- Install software with `elevate zex install`; never copy files into `/overlayer/zexlib/union` or `/overlayer/syshub` by hand.
- Never edit `/overlayer/syshub` directly: it is the immutable base, protected even for root.
- Check a package with `zex search` / `zex inspect` before installing, and prefer `--dry-resolve` when unsure what an install will pull in.
