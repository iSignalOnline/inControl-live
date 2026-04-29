# inControl-live / MiniOS Live — Codebase Overview

> **MiniOS** is a portable, user-friendly Linux distribution with a graphical interface that runs entirely from removable media (USB drive, optical disc, etc.). This repository (`minios-live`) contains the complete toolchain for building bootable MiniOS ISO images.

---

## Table of Contents

1. [What the Repository Does](#1-what-the-repository-does)
2. [Key Technologies Used](#2-key-technologies-used)
3. [High-Level Directory Structure](#3-high-level-directory-structure)
4. [Core Files Explained](#4-core-files-explained)
5. [Supported Build Configurations](#5-supported-build-configurations)
6. [Step-by-Step: How the Live Linux ISO Is Built](#6-step-by-step-how-the-live-linux-iso-is-built)
7. [CI/CD Automation with GitHub Actions](#7-cicd-automation-with-github-actions)
8. [Submodules](#8-submodules)
9. [Internationalisation (i18n)](#9-internationalisation-i18n)
10. [Debian Packaging](#10-debian-packaging)

---

## 1. What the Repository Does

`minios-live` is a **Bash-based build system** that automates the creation of a bootable live Linux ISO image. Starting from a bare Debian or Ubuntu distribution, it:

1. Bootstraps a minimal root filesystem with `debootstrap`.
2. Installs the kernel, desktop environment, and all software packages inside a `chroot`.
3. Packs each logical layer of the system into a compressed **SquashFS** (`.sb`) module.
4. Builds an **initramfs** (using either the in-house *livekit* builder or *dracut*).
5. Assembles a complete **ISO 9660** image with BIOS (SYSLINUX) and UEFI (GRUB 2) boot loaders.

The system is fully configurable: distribution, architecture, desktop environment, locale, kernel flavour, compression algorithm, boot loader type, and more are all set via a single `build.conf` file.

---

## 2. Key Technologies Used

| Technology | Role |
|---|---|
| **Bash** | 100 % of build logic — all scripts are POSIX-compatible Bash |
| **debootstrap** | Bootstraps the Debian/Ubuntu minimal root filesystem |
| **chroot / overlayfs** | Installs packages and runs scripts inside an isolated environment |
| **SquashFS / mksquashfs** | Compresses filesystem layers into read-only `.sb` modules |
| **xorriso** | Creates the final hybrid ISO image (BIOS + UEFI) |
| **GRUB 2** | UEFI boot loader; also used as fallback BIOS loader |
| **SYSLINUX / vesamenu** | Primary BIOS boot loader with graphical language-selection menu |
| **dracut / livekit-mos** | Two alternative initramfs builders; dracut is the default |
| **APT / dpkg** | Package management inside the chroot |
| **CondinAPT** (`condinapt`) | Custom conditional APT wrapper that installs packages from a list file based on per-package conditions (distribution, DE, variant, etc.) |
| **GNU gettext / po4a** | Shell-script and man-page internationalisation |
| **GitHub Actions** | Automated builds and release uploads |
| **Debian packaging** | The toolchain itself is distributed as a `.deb` package (`minios-live`) |

Compression formats supported for SquashFS: `zstd` (default, best performance), `xz`, `lzo`, `gz`, `lz4`.

---

## 3. High-Level Directory Structure

```
inControl-live/
├── minios-live            # Main build orchestrator (Bash entry-point)
├── minios-cmd             # User-friendly CLI wrapper around minios-live
├── linux-live/            # All build internals
│   ├── build.conf         # Master configuration file
│   ├── minioslib          # ~3 000-line Bash library: ALL build functions
│   ├── install-chroot     # Script executed INSIDE the chroot
│   ├── build-initramfs    # Initramfs builder dispatcher (livekit or dracut)
│   ├── condinapt          # Conditional package installer
│   ├── condinapt.map      # Condition-to-variable mapping file
│   ├── prerequisites.list # Host packages needed before the build starts
│   ├── aptsources/        # APT sources.list templates (one per distro)
│   └── test_condinapt.sh  # Unit test for condinapt
├── submodules/            # Git submodules — companion tools and apps
├── .github/workflows/     # GitHub Actions CI/CD pipelines
├── completions/           # Bash tab-completion scripts
├── debian/                # Debian packaging metadata
├── docs/                  # Markdown source for man pages
├── manpages/              # po4a config for translated man pages
├── po/                    # Shell-script translation catalogues (.po/.pot)
├── release-notes/         # Per-version release notes
├── tools/                 # Miscellaneous helper scripts and patches
│   ├── build-linux-images # Helper to build custom kernel images
│   ├── build-package.sh   # Builds the minios-live Debian package
│   ├── depends-creator    # Generates package dependency lists
│   ├── unmount-dirs.sh    # Standalone unmount helper
│   └── aufs6.1.124-mmap.patch # Kernel patch for AUFS support
├── images/                # Repository images (logo, etc.)
├── Makefile               # Builds the minios-live .deb package
└── README.md
```

---

## 4. Core Files Explained

### `minios-live` — Build Orchestrator

The entry point of the build pipeline. It:

- Sources `minioslib` and `build.conf`.
- Defines the **ordered command array**:
  ```
  CMD=(build_bootstrap build_chroot build_live build_modules build_boot build_config build_iso remove_sources)
  ```
- Accepts flexible range syntax: `./minios-live [start_cmd] [-] [end_cmd]`
  - `./minios-live -` → run everything
  - `./minios-live build-iso` → run only one step
  - `./minios-live build-bootstrap - build-chroot` → run a range
- Logs the entire session to `build/log/build-<timestamp>.log`.
- Registers an `EXIT` trap that unmounts all bind mounts and stops any local HTTP caching server.

### `minios-cmd` — User CLI

A thin wrapper around `minios-live` that accepts **named flags** instead of requiring manual edits to `build.conf`. Useful for CI pipelines and one-liners:

```bash
sudo ./minios-cmd -d trixie -a amd64 -de xfce -pv standard -dkms -kl -ib dracut
```

Key flags:

| Flag | Meaning |
|---|---|
| `-d` / `--distribution` | Distribution codename (e.g. `trixie`, `bookworm`, `jammy`) |
| `-a` / `--architecture` | CPU architecture (`amd64`, `i386`, `arm64`) |
| `-de` / `--desktop-environment` | `xfce`, `flux`, `lxqt`, `core` |
| `-pv` / `--package-variant` | `standard`, `toolbox`, `ultra`, `minimum` |
| `-c` / `--compression-type` | `zstd`, `xz`, `lzo`, `gz`, `lz4` |
| `-ib` / `--initramfs-builder` | `livekit` or `dracut` |
| `-dkms` | Enable DKMS extra driver compilation |
| `-kl` | Keep all locales in the image |
| `--ubuntu-pro-token TOKEN` | Attach Ubuntu Pro during build (token is stripped from final image) |
| `--config-only` | Write `build.conf` only; do not start the build |

### `linux-live/build.conf` — Master Configuration

All build parameters live here. Key groups:

- **Distribution settings** — distro name, arch, desktop, package variant, compression
- **Kernel settings** — flavour (`none`, `rt`, `cloud`), AUFS support, DKMS
- **Locale & timezone** — locale code, multilingual flag, timezone
- **Boot loader** — `syslinux-native`, `syslinux-grub`, or `grub-only`; menu language
- **live-config settings** — hostname, username, user groups, hashed passwords
- **Builder settings** — verbosity, APT caching mode, snapshot builds, module filtering
- **Ubuntu Pro** — optional Pro subscription token for build-time ESM access

### `linux-live/minioslib` — The Build Library (~3 000 lines)

Every function called during the build is defined here. Major groups:

| Section | Key functions |
|---|---|
| Variable setup | `common_variables`, `declare_locales` |
| Logging / UI | `console_colors`, `spinner`, `run_with_spinner`, `information`, `warning`, `error` |
| Config I/O | `read_config`, `update_config`, `read_config_value` |
| Host prerequisites | `ensure_host_prerequisites`, `check_internet_connection` |
| chroot management | `setup_chroot_environment`, `chroot_mount_fs`, `chroot_run`, `unmount_dirs` |
| APT caching | `setup_apt_cache`, `start_http_server`, `stop_http_server` |
| Ubuntu Pro | `ubuntu_pro_attach`, `ubuntu_pro_detach`, `ubuntu_pro_cleanup_traces` |
| **Build stages** | `build_bootstrap`, `build_chroot`, `build_live`, `build_modules`, `build_boot`, `build_config`, `build_iso`, `remove_sources` |
| Initramfs | `build_initrd` (calls `build-initramfs` script inside chroot) |
| SquashFS modules | `build_module`, `mkmod_corefs` |
| Boot configs | `create_config_files` (generates SYSLINUX + GRUB configs for all languages) |
| ISO creation | `build_iso` (calls `xorriso`) |
| Translations | `parse_po_file`, `get_translation`, `generate_localized_grub_config` |
| GRUB optimisation | `remove_unused_grub_modules` |

### `linux-live/condinapt` — Conditional Package Installer

A standalone Bash program that reads a **package list file** (one entry per line) where each line may carry filter conditions:

```
# Syntax: package_name [CONDITION: VAR=value ...]
firefox         [DISTRIBUTION_TYPE=debian]
libreoffice     [DESKTOP_ENVIRONMENT=xfce PACKAGE_VARIANT!=minimum]
```

This allows a single package list to cover many different build configurations without `if/else` sprawl. `condinapt.map` maps symbolic condition names to actual environment variables.

### `linux-live/install-chroot` — In-Chroot Installer

Executed inside the chroot after `build_chroot` sets up the environment. It calls `install_core_packages` (defined in `minioslib`) which in turn invokes `condinapt` with the appropriate package list for the chosen distro/DE/variant.

### `linux-live/build-initramfs` — Initramfs Dispatcher

Runs inside the chroot; dispatches to one of two builders:

- **livekit** (`linux-live/initramfs/livekit-mos/mkinitrfs`) — lightweight, produces a ~2× smaller initramfs.
- **dracut** (`linux-live/initramfs/dracut-mos/mkdracut`) — loads significantly faster at boot; the default for newer builds.

---

## 5. Supported Build Configurations

### Distributions

| Family | Codenames |
|---|---|
| Debian | `buster` (10), `bullseye` (11), `bookworm` (12), `trixie` (13), `sid` |
| Ubuntu | `bionic` (18.04), `focal` (20.04), `jammy` (22.04), `noble` (24.04) |
| Other | `kali-rolling`, `orel` |

### Desktop Environments

| Name | Notes |
|---|---|
| `xfce` | Default; used with `standard`, `toolbox`, `ultra` variants |
| `flux` | Fluxbox-based minimal desktop; forces `LIVE_USERNAME=root` |
| `lxqt` | Available for Bookworm/Trixie |
| `core` | No GUI; server/cloud variants |

### Package Variants

| Variant | Description |
|---|---|
| `minimum` | Minimal packages; used with `flux` |
| `standard` | Typical desktop with common applications |
| `toolbox` | Standard + development/security tools; SSH enabled by default |
| `ultra` | Everything in toolbox plus AppArmor/SELinux disabled |

### Architectures

`amd64`, `i386`, `i386-pae`, `arm64`

---

## 6. Step-by-Step: How the Live Linux ISO Is Built

Below is a detailed walk-through of each build stage in sequence.

---

### Step 0 — Prerequisites Check (`ensure_host_prerequisites`)

Before any build stage runs, `minios-live` checks that all required host tools are installed (`prerequisites.list`):

```
sudo, binutils, debootstrap, squashfs-tools, xz-utils, lz4, zstd,
xorriso, mtools, rsync, grub-common, gpg
```

An internet connectivity check is performed (unless using a local APT cache repository).

---

### Step 1 — `build_bootstrap` — Bootstrap the Root Filesystem

**Purpose:** Create the bare-minimum root filesystem (`core/`) for the target distribution.

**What happens:**

1. **Clean slate:** The `core/` and `image/` work directories are wiped.
2. **Rootfs cache hit?** If `build/rootfs/<distro>-<arch>-rootfs.tar.gz` already exists and `USE_ROOTFS=true`, it is extracted directly (saves re-downloading packages).
3. **Rootfs cache miss:** `debootstrap` is called with `--no-check-gpg` to install a minimal base system from the upstream Debian/Ubuntu mirror into `build/<distro>-<variant>-<arch>/core/`.
   ```bash
   debootstrap --arch=amd64 --include=ca-certificates,wget,dbus,sudo,curl \
       trixie ./core http://deb.debian.org/debian
   ```
4. The resulting directory tree is archived to `rootfs.tar.gz` for future builds.
5. The correct `sources.list` (from `linux-live/aptsources/`) is placed into the chroot.

---

### Step 2 — `build_chroot` — Full System Installation

**Purpose:** Install the kernel, desktop environment, and all application packages inside the chroot.

**What happens:**

1. **Bind mounts:** `/dev`, `/run`, `/proc`, `/sys`, `/dev/pts`, `/tmp` are bind-mounted into the chroot so that package post-install scripts work correctly.
2. **APT configuration:** The MiniOS repository (`deb.minios.dev`) and GPG keys are added. An optional local APT cache (`USE_APT_CACHE`) is bind-mounted to avoid re-downloading.
3. **Ubuntu Pro (optional):** If a token is provided, `ubuntu_pro_attach` activates ESM repositories inside the chroot for fresher packages. The subscription is **always removed** after installation.
4. **`install-chroot` script** is executed inside the chroot:
   - Sources `minioslib` and `minios_build.conf`.
   - Calls `install_core_packages` → runs `condinapt` with the package list files matching the current `DISTRIBUTION` / `DESKTOP_ENVIRONMENT` / `PACKAGE_VARIANT`.
   - Installs the Linux kernel (if `INSTALL_KERNEL=true`).
   - Optionally compiles DKMS modules (`KERNEL_BUILD_DKMS=true`).
5. `/etc/minios-release` is written with `NAME`, `VERSION`, and `EDITION`.
6. All bind mounts are removed.

---

### Step 3 — `build_live` — Create the Core SquashFS Module

**Purpose:** Compress the installed root filesystem into a single SquashFS bundle: `00-core-amd64.sb`.

**What happens:**

1. The build scripts are copied into the chroot's `linux-live/` directory.
2. The top-level filesystem directories (`bin`, `etc`, `home`, `lib`, `lib64`, `opt`, `root`, `sbin`, `srv`, `usr`, `var`) are located using `mkmod_corefs`.
3. `mksquashfs` compresses them:
   ```bash
   mksquashfs bin etc home ... \
       image/minios/00-core-amd64.sb \
       -comp zstd -Xcompression-level 19 -b 1024K -always-use-fragments
   ```
4. The resulting `00-core-amd64.sb` is the foundation layer that the live system mounts at boot.

---

### Step 4 — `build_modules` — Build Additional SquashFS Modules

**Purpose:** Create extra `.sb` modules for optional software layers (e.g. `01-xorg.sb`, `02-xfce.sb`, `03-apps.sb`, …).

**What happens:**

1. For each module defined in the build configuration, `build_module` is called.
2. An **overlayfs** is set up on top of the already-built `core/`:
   - `upper/` — writable layer where new packages are installed
   - `work/` — overlayfs bookkeeping
   - `merged/` — the combined chroot view
3. APT installs the module's packages into the overlay's upper layer only.
4. `mksquashfs` compresses the `upper/` directory into the numbered `.sb` file.
5. The overlay is dismantled cleanly.

**Module mode:**
- `merged` — Used for `standard` and `puzzle` variants. All applications are merged into a small number of modules.
- `simple` — Used for `toolbox` and `ultra`. Each logical group gets its own module.

---

### Step 5 — `build_boot` — Build the Initramfs and Copy Boot Files

**Purpose:** Produce the kernel image (`vmlinuz`) and initramfs (`initrfs.img`) and place them in the ISO's boot directory.

**What happens:**

1. **Overlay setup:** An overlayfs is mounted on top of `core/` so that `vmlinuz` and kernel modules can be accessed without modifying the already-built SquashFS.
2. **`build-initramfs` script is executed inside the chroot** (via `chroot_run`):
   - Reads `INITRAMFS_BUILDER` (`livekit` or `dracut`).
   - **livekit path:** Runs `mkinitrfs` from `linux-live/initramfs/livekit-mos/`. Produces a lightweight cpio archive wired specifically for MiniOS boot.
   - **dracut path:** Runs `mkdracut` from `linux-live/initramfs/dracut-mos/`. Produces a standards-compliant initramfs that loads noticeably faster.
   - Both builders accept `-n` (network) and `-c` (cryptsetup) flags for non-minimal variants.
   - Output: `/boot/initrfs.img` inside the chroot.
3. `vmlinuz` and `initrfs.img` are copied to:
   ```
   image/minios/boot/vmlinuz[-<kernel-version>]
   image/minios/boot/initrfs[-<kernel-version>].img
   ```
4. Compatibility symlinks `vmlinuz` → `vmlinuz-<ver>` and `initrfs.img` → `initrfs-<ver>.img` are created for Ventoy.

---

### Step 6 — `build_config` — Generate Live System and Boot Loader Configs

**Purpose:** Write `config.conf` (live system runtime settings) and all SYSLINUX/GRUB boot menu files.

**What happens:**

**`config.conf`** (placed in `image/minios/`) is written with:
- Hostname, username, user groups
- Hashed user and root passwords
- Locale, timezone, keyboard layout
- Module mode (`merged` / `simple`)
- Enabled/disabled services
- Session persistence settings (`LIVE_LINK_USER_DIRS`, `LIVE_BIND_USER_DIRS`)

**SYSLINUX configuration** (for BIOS boot — `syslinux-native` or `syslinux-grub` mode):
- A **language-selection main menu** (`syslinux.multilang.cfg`) lets the user pick from 9 languages (EN, RU, DE, ES, FR, ID, IT, PT-BR, PT-PT) before booting.
- Per-language configs in `boot/syslinux/lang/` set locale, timezone, and keyboard on the kernel command line.
- Translations are read directly from `.po` files; text is re-encoded to CP866 (Russian) or ISO-8859-1 (Latin scripts) as required by SYSLINUX.
- PSF console fonts are copied for proper glyph rendering.

**GRUB 2 configuration** (always generated — required for UEFI):
- `grub.multilang.cfg` — language-selection menu.
- `main.cfg` — actual boot entries (`Resume`, `New session`, `Choose session`, `Fresh start`, `Copy to RAM`).
- `grub.template.cfg` — English-only template used by the installer.
- Gettext `.mo` files are compiled from the boot `po/` sources.
- Unused GRUB modules are pruned (`remove_unused_grub_modules`) to minimise ISO size.

All five boot menu entries map to the same kernel with different `perchdir=` and `toram` kernel parameters:

| Entry | Effect |
|---|---|
| Resume previous session | `perchdir=resume` — mounts existing changes directory |
| Start a new session | `perchdir=new` — creates a new changes directory |
| Choose session during startup | `perchdir=ask` — prompts at boot |
| Fresh start | No `perchdir` — fully volatile |
| Copy to RAM | `toram` — loads all modules into RAM |

---

### Step 7 — `build_iso` — Assemble the Final ISO

**Purpose:** Combine all assets (SquashFS modules, kernel, initramfs, boot loaders) into a single bootable `.iso` file.

**What happens:**

1. **SYSLINUX BIOS boot sector** files (`isolinux.bin`, `isohdpfx.bin`, etc.) are copied into `image/`.
2. **GRUB UEFI** image (`efi.img`) is assembled from the GRUB EFI modules.
3. **`xorriso`** is called to produce a hybrid ISO (bootable from both optical disc and USB):
   ```bash
   xorriso -as mkisofs \
     -iso-level 3 -full-iso9660-filenames \
     -isohybrid-mbr isohdpfx.bin \
     -b minios/boot/syslinux/isolinux.bin \
     -c minios/boot/syslinux/boot.cat \
     -eltorito-alt-boot -e efi.img -no-emul-boot \
     -isohybrid-gpt-basdat efi.img \
     -o "minios-<distro>-<variant>-<arch>-<version>.iso" \
     image/
   ```
4. A SHA-256 checksum file (`.sha256`) is generated alongside the ISO.
5. If `BUILD_TEST_ISO=true`, a second constant-filename ISO is written for test automation.

---

### Step 8 — `remove_sources` — Optional Cleanup

**Purpose:** Remove the large working directory (`build/<distro>-<variant>-<arch>/`) to reclaim disk space. Controlled by `REMOVE_SOURCES=true`.

---

## 7. CI/CD Automation with GitHub Actions

Three workflow files live in `.github/workflows/`:

| Workflow | Trigger | Build |
|---|---|---|
| `trixie-xfce-standard-amd64.yml` | `workflow_dispatch` (manual with tag) | Debian 13 / XFCE / Standard / amd64 |
| `trixie-xfce-toolbox-amd64.yml` | `workflow_dispatch` | Debian 13 / XFCE / Toolbox / amd64 |
| `trixie-xfce-ultra-amd64.yml` | `workflow_dispatch` | Debian 13 / XFCE / Ultra / amd64 |
| `release.yml` | — | Manages the GitHub Release |

Each workflow:
1. Clones the **upstream** `minios-linux/minios-live` repository at the supplied tag.
2. Installs host dependencies (`debootstrap`, `squashfs-tools`, `xorriso`, etc.).
3. Runs `minios-cmd` with the appropriate flags.
4. Uploads the ISO (and checksum) as a GitHub Actions **artifact**.
5. A downstream `release` job downloads all artifacts and uploads them to the GitHub Release page via `gh release upload`.

---

## 8. Submodules

The repository references 26 Git submodules under `submodules/`. They are companion tools that are either:

- **Installed as packages inside the image** (e.g. `minios-tools`, `minios-installer`, `minios-welcome`, `minios-configurator`, `minios-session-manager`)
- **Kernel infrastructure** (`ntfs3-dkms`, `minios-dracut`, `minios-kernel-manager`)
- **UI components** (`elementary-xfce-minios`, `xlunch`, `ncurses-menu`, `flux-tools`)
- **Documentation / website** (`minios-live.wiki`, `minios-linux.github.io`, `docs`)
- **Misc utilities** (`b43-firmware`, `rescuezilla-app`, `driveutility`, `dynfilefs-app`, `mini-commander-app`)
- **Translations** (`minios-translations`)

---

## 9. Internationalisation (i18n)

Two parallel i18n systems are used:

### Shell-script translations (`po/`)
- `po/messages.pot` — template catalogue extracted from `minios-live` and `minios-cmd`.
- Per-language `.po` files: `de`, `es`, `fr`, `id`, `it`, `pt`, `pt_BR`, `ru`.
- Used via standard GNU `gettext` / `TEXTDOMAIN=minios-live`.
- `update-po.sh` refreshes all `.po` files from the `.pot` template.

### Man-page translations (`manpages/`, `docs/`)
- Source documentation is Markdown in `docs/`.
- `manpages/po4a.cfg` configures **po4a** to generate translated man pages from the `.po` files.
- Produces man pages under `minios-live(1)`, `minios-cmd(1)`, `condinapt(1)`, `condinapt-minios(7)`.

### Boot menu translations
- Live GRUB/SYSLINUX menus read `.po` files directly from `linux-live/bootfiles/boot/grub/po/` at build time.
- Translations are embedded into per-language GRUB `.mo` files compiled with `msgfmt`.
- SYSLINUX uses direct text substitution from the same `.po` sources.

---

## 10. Debian Packaging

The `minios-live` toolchain ships as a proper Debian package. The `debian/` directory provides the full packaging metadata:

| File | Purpose |
|---|---|
| `debian/control` | Package name, arch (`amd64`), dependencies, description |
| `debian/rules` | Build rules (uses `debhelper`) |
| `debian/changelog` | Version history |
| `debian/compat` | debhelper compatibility level |

The `Makefile` at the repo root automates packaging:
```bash
make           # check-deps + build-package
make orig      # create orig.tar.gz
make lintian   # lint the resulting .deb
make clean     # remove build/
```

When installed as a `.deb`, the tools are placed at:
- `/usr/bin/minios-live`, `/usr/bin/minios-cmd`
- `/usr/share/minios-live/` (build scripts, aptsources, bootfiles, initramfs builders)
- `/etc/minios-live/build.conf` (default config)

---

## Quick-Start Build Example

```bash
# Clone and enter the repository
git clone --recurse-submodules https://github.com/minios-linux/minios-live.git
cd minios-live

# Install host prerequisites
sudo apt-get install -y sudo binutils debootstrap squashfs-tools xz-utils \
  lz4 zstd xorriso mtools rsync grub-common gpg gettext dosfstools curl

# Build Debian 13 (Trixie) XFCE Standard amd64
sudo ./minios-cmd -d trixie -a amd64 -de xfce -pv standard -dkms -kl -ib dracut

# The ISO will be at:
# build/iso/MiniOS-Linux-<version>-trixie-xfce-standard-amd64.iso
```

---

*This document was generated from a full analysis of the `inControl-live` repository source code.*
