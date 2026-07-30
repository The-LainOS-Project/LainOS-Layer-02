<!--Reddit-->
<a href="https://www.reddit.com/r/LainOSdevelopers/" target="_blank">
  <img align="top" src="https://img.shields.io/badge/Reddit-FF4500?style=for-the-badge&logo=reddit&logoColor=white" alt="Reddit">
</a>
<!--Matrix-->
<a href="https://matrix.to/#/#lainos:catgirl.cloud" target="_blank">
  <img align="top" src="https://img.shields.io/badge/Matrix%20-%20%230047a7?style=for-the-badge&logo=matrix" alt="Matrix">
</a>
<!--Discord-->
<a href="https://discord.gg/JdMQvkHqwH" target="_blank">
  <img align="top" src="https://img.shields.io/badge/Discord%20-%20%234900ff?style=for-the-badge&logo=discord" alt="Discord">
</a>
<!--Web page-->
<a href="https://lainos.net" target="_blank">
  <img align="top" src="https://img.shields.io/badge/Lain%20OS%20web-3d3b93?style=for-the-badge&logo=Devbox" alt="Web">
</a>
<!--Forgejo-->
<a href="https://forgejo.lain.rocks/lainOS" target="_blank">
  <img align="top" src="https://img.shields.io/badge/Forgejo-ff6600?style=for-the-badge&logo=forgejo&logoColor=white" alt="Forgejo">
</a>

# lainOS layer 02

Available at https://forgejo.lain.rocks/lainOS/lainOS-layer-02/releases

> A systemd-free Arch Linux derivative built with OpenRC as PID 1, offering full ABI compatibility for systemd-linked software via the Protocol 7 compatibility architecture.

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Architecture](https://img.shields.io/badge/Arch-x86__64-blue)](https://archlinux.org)
[![Init](https://img.shields.io/badge/Init-OpenRC-green)](https://github.com/OpenRC/openrc)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Key Features](#key-features)
- [System Requirements](#system-requirements)
- [Building](#building)
- [Installation](#installation)
- [Session Types](#session-types)
- [Protocol 7 Compatibility Layer](#protocol-7-compatibility-layer)
- [System Hardening](#system-hardening)
- [Network Configuration](#network-configuration)
- [lainos-utils](#lainos-utils)
- [Troubleshooting](#troubleshooting)
- [Project Structure](#project-structure)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)

---

## Overview

### LainOS Layer 02 is a daily-driver distribution that aims to strike a balance between usability, privacy, and security. It includes security and privacy hardening as first-class concerns, not as an afterthought, but it is not a specialized security distribution like Qubes or Kicksecure/Whonix. It is built to be usable, with hardening that does not get in the way.

LainOS Layer 02 is a custom Arch Linux-based distribution built on a clean Arch base using [archiso](https://wiki.archlinux.org/title/Archiso). It replaces systemd with **OpenRC** as PID 1, while maintaining ABI compatibility with software that dynamically links against systemd's client libraries through the **Protocol 7** compatibility layer.

The entire OpenRC stack is self-hosted ~ no Artix repositories are used. All OpenRC service scripts, compatibility packages, and custom daemons are maintained in the Protocol 7 repository.

This is not merely a themed Arch respin ~ it is a genuine init-system replacement project with custom C daemons handling responsibilities that systemd would otherwise own.

---

## Architecture

| Layer | Component | Role |
|-------|-----------|------|
| **Boot** | GRUB/Syslinux/EFI ~ Dracut | UEFI/BIOS boot, initramfs with `dmsquash-live` and LUKS support |
| **Init** | `/sbin/openrc-init` | PID 1, explicit kernel cmdline |
| **Live Session** | `greetd` + `tuigreet` | TUI login manager on TTY1 |
| **Wayland** | Sway + i3status-rs | Tiling compositor, autotiling, themed status bar |
| **Installer** | Calamares | Graphical system installer (autolaunches on liveuser login) |
| **Compatibility** | `protocol7-core` | systemd ABI surface, D-Bus facade, session management |

### Boot Chain

```
BIOS/UEFI ~ GRUB/Syslinux ~ kernel + initramfs
  ~ Dracut: dmsquash-live mounts squashfs, execs /sbin/openrc-init
     ~ OpenRC sysinit: dbus, lainos-notifyd, lainos-machine-id
        ~ OpenRC boot: rfkill-unblock, cgroup-delegate, lainos-ghost-units, syslog-ng
           ~ OpenRC default: seatd, lainos-dbus-bridge, greetd, chrony, nftables, acpid, polkit
              ~ greetd ~ tuigreet ~ Sway session
                 ~ lainos-session-sway ~ lainos-init ~ Sway
```

`iwd` is intentionally **not** in this default startup chain ~ WiFi stays off until you deliberately turn it on. See [Key Features](#key-features).

`rfkill-unblock` unblocks the WiFi radio at the kernel level on Libreboot/`thinkpad_acpi` hardware where it may come up soft-blocked ~ it does not start `iwd` itself and does not affect the WiFi-off-by-default posture.

---

## Key Features

### System
- **systemd-free** ~ OpenRC as PID 1 with no systemd binary present
- **Protocol 7** ~ Custom compatibility layer providing `libsystemd.so.0` ABI via real systemd-libs (not mocks)
- **Self-hosted OpenRC stack** ~ all OpenRC packages maintained in protocol_7_repo, no Artix dependency
- **Live ISO** ~ Fully bootable live environment with Calamares installer (autolaunches on login)
- **Dracut initramfs** ~ Modern initramfs with live boot and LUKS/crypt support
- **BTRFS by default** ~ with separate ext4 /boot for GRUB compatibility
- **ISO Size** ~ 2.7GB (2026.07.26-RC8)

### Desktop (Sway/Wayland)
- **Sway** 1.12+ tiling compositor with custom keybindings
- **i3status-rs** themed status bar
- **wofi** application launcher
- **alacritty** terminal emulator
- **mako** notification daemon
- **swaybg** static wallpaper
- **Autotiling** automatic window tiling
- **Powerlevel10k** zsh prompt
- **CoplandOS-GTK** dark theme with StarLabs cursor
- **wlogout** session/power menu (shutdown, reboot, suspend, lock, logout)
- **swaylock** screen locker with wallpaper background
- **gnome-keyring** secrets service backend for Electron apps
- **PipeWire** audio, bundled by default (`pipewire`, `pipewire-pulse`, `pipewire-alsa`, `pipewire-jack`); `lainos-audio-init` orchestrates startup

### System Hardening
- **Protocol 7 Core fuzz tested** ~ dfuzzer 2.6 full interface PASS, AddressSanitizer PASS, libFuzzer 2M+ iterations PASS on lainos-notifyd; two purpose-built libFuzzer harnesses on lainos-init's environment-driven exec logic, 9M+ combined executions, zero crashes
- **Manual security review, not just automated tooling** ~ a logic-focused code review of lainos-dbus-bridge found and fixed a missing caller-identity check on Reboot/PowerOff D-Bus methods (tested and confirmed not independently exploitable given the existing privilege model, fixed anyway as defense-in-depth)
- **Seccomp hardening modeled on Whonix/Kicksecure's systemd sandboxing directives** ~ `socket()` restricted to `AF_UNIX` only (RestrictAddressFamilies equivalent); `personality()`/`unshare()`/`setns()` confirmed blocked by the existing default-deny policy (LockPersonality/RestrictNamespaces equivalents)
- **Capability bounding set explicitly cleared** (5.5.3-25) ~ `prctl(PR_CAPBSET_DROP, ...)` zeroes all capability fields (`CapInh`, `CapPrm`, `CapEff`, `CapBnd`, `CapAmb`) after privilege drop on both daemons; verified directly via `/proc/<pid>/status` on live production processes
- **Filesystem/mount isolation** (5.5.3-26) ~ hand-rolled mount namespaces on `lainos-dbus-bridge` and `lainos-notifyd`: read-only root filesystem (`ProtectSystem=strict` equivalent), private `/tmp` (`PrivateTmp` equivalent), hidden `/home`/`/root` (`ProtectHome` equivalent), and minimal four-node `/dev` (`PrivateDevices` equivalent). Verified incrementally via direct `/proc/<pid>/mounts` inspection, full dfuzzer regression, and Valgrind memcheck at each checkpoint
- **AppArmor mandatory access control** (lainos-apparmor 1.2.1-29) ~ per-daemon profiles for all external-input components (`lainos-dbus-bridge`, `lainos-notifyd`, `lainos-init`), plus the DNS mediation layer (`dnsmasq`, `unbound`, `dnscrypt-proxy`) and the broader system stack (networking, media, crypto, browsers, utilities), loaded at boot before the daemons start. Path-based MAC complements seccomp + mount namespace isolation: if an attacker escapes the private namespace, AppArmor still blocks access to real user data, sensitive kernel interfaces, and network sockets. Verified via a 36-test adversarial suite covering enforcement mode, privilege drop, capability/seccomp state, mount namespace isolation, a genuine AppArmor negative control (`/var/tmp` write denied despite DAC alone allowing it), ptrace mediation, network socket/profile-text checks, and `lainos-init`'s `P7_CMD` whitelist logic. 36/36 passed
- **hardened_malloc** (GrapheneOS, light variant) Systemwide toggle `lainos-hardened-malloc {enable|disable|status}`, and preloaded by default for: gnome-keyring-daemon, keepassxc, kleopatra, mpv, tor
- **ram-wipe** ~ RAM-extraction attack defense, ported from Kicksecure/Whonix's `ram-wipe`: wipes reclaimable disk cache from RAM at every shutdown/reboot via a dracut hook, on top of the always-on `init_on_alloc`/`init_on_free` continuous memory poisoning below. Toggle the shutdown-time wipe pass with `ram-wipe {enable|disable|status}`. **Not yet visually confirmed on all hardware** ~ correctly installed and baked into the initramfs, multiple reboots completed cleanly with it active, but the on-screen shutdown message could not be visually confirmed on Sway/wlroots test hardware, likely a GPU driver/console handoff quirk rather than a functional problem
- **Kernel memory hardening** ~ `init_on_alloc=1`, `init_on_free=1`, `page_alloc.shuffle=1`
- **Full ASLR** ~ `randomize_va_space=2`
- **ptrace restricted** ~ `yama.ptrace_scope=1`
- **kexec disabled** ~ `kexec_load_disabled=1`
- **Unprivileged user namespaces disabled** ~ `unprivileged_userns_clone=0` (some software's own build-time sandboxing needs this temporarily enabled ~ see [Building Software with Sandboxed Build Requirements](#system-hardening))
- **Core dumps disabled**
- **IPv6 disabled** by default (prevents VPN leaks)
- **SYN flood protection**, ICMP redirect rejection, reverse path filtering
- **Kernel pointer restriction** (`kptr_restrict=2`)
- **dmesg restricted** to root only
- **Magic SysRq disabled**
- **CPU RNG not unconditionally trusted** (`random.trust_cpu=off`)
- **Ephemeral machine-id** ~ regenerated on every boot
- **Boot clock randomization** ~ clock jittered by up to ±180 seconds before networking comes up, before any time-sync mechanism runs
- **WiFi off by default** ~ `iwd` does not start automatically at boot; toggle with `wifi`/`wifi-autostart`
- **iwd MAC randomization** ~ new MAC address every time iwd starts; `wscan` also disables `AutoConnect` on every network immediately after connecting (default), so nothing reconnects silently on its own ~ toggle with `wifi-autoconnect`
- **Ethernet MAC randomization** ~ available via `eth0` toggle
- **Optional Tor-based time sync** ~ `sdwdate` (opt-in, since it has no fallback if Tor is unreachable) replaces plaintext NTP with fingerprint-resistant, cross-validated time sync over Tor
- **Optional Tor pluggable transports** ~ `snowflake`/`obfs4` toggles to disguise Tor traffic
- **Tor stream isolation** ~ the default SocksPort (9050, used by plain `torsocks`) isolates circuits by destination automatically; four additional dedicated, permanently isolated circuits (`tor1`-`tor4`, matching the `wg1`-`wg4` WireGuard tunnel model) are available for stronger per-application isolation. Usage is identical across CLI tools, GTK apps, and Electron apps (Signal, Element) ~ `tor1`-`tor4` auto-detect Electron apps and route them via Chromium's own `--proxy-server` flag instead of `torsocks` (which cannot intercept Electron's sandboxed renderer-process networking), and automatically strip `LD_PRELOAD` and override `XDG_CURRENT_DESKTOP` to work around known `hardened_malloc`/keyring-detection conflicts on Sway. Firefox-based browsers (LibreWolf) are a known exception ~ `torsocks` is unreliable for Firefox's multi-process architecture; configure LibreWolf's SOCKS5 proxy natively instead
- **private-mode** ~ one command for the sensitive-work sequence (`private-mode {on|off|status}`): forces WiFi/ethernet down, confirms the network is actually down, enables Snowflake + sdwdate, brings network back up long enough to bootstrap the encrypted DNS chain (if applicable) and for Tor itself to bootstrap, then switches dnsmasq to Tor DNSPort. Reverses safely on shutdown: network down first, then disables sdwdate, stops Tor, and restores the previous DNS mode (plaintext or encrypted). Automates the mechanical ordering only ~ does not replace manually verifying Tor's bootstrap state and sdwdate's sync log
- **nftables firewall** ~ default-deny with established/related allowed
- **yescrypt** password hashing
- **doas** instead of sudo (minimal attack surface)
- **LUKS full disk encryption** ~ opt-in at install, confirmed working
- **Signed package repositories** ~ SigLevel = Required, full trust chain via lainos-keyring

### Networking
- **iwd** for WiFi ~ off by default, toggle with `wifi`/`wifi-autostart`
- **dhcpcd** + **openresolv** for wired DHCP; `eth0` toggle for wired interface control + MAC randomization
- **dnsmasq** as centralized DNS mediator ~ blind forwarding resolver, no caching, three modes (plaintext/encrypted/private) toggled via `lainos-dns` and `private-mode`. Encrypted mode chains `dnsmasq` → `unbound` (DNSSEC validation, QNAME minimization, caching) → `dnscrypt-proxy` (wire encryption, anonymized relay routing) ~ no single component in the chain sees both the user's IP and the query. `stubby` remains available as a lightweight DoT fallback if `unbound` is not installed
- **chrony** for NTP time synchronization (default) or **sdwdate** (opt-in Tor-based alternative)
- **nftables** for firewall management
- No NetworkManager, no systemd-networkd, no systemd-resolved

---

## System Requirements

### Minimum
- 64-bit x86_64 processor
- 2 GB RAM
- 4 GB USB drive or free disk space
- UEFI or BIOS boot support

### Recommended
- 4+ GB RAM
- GPU with Mesa drivers (Intel/AMD recommended; software rendering fallback available)
- USB 3.0 for live boot

### Tested Hardware
- QEMU/KVM with Virtio GPU (primary development environment)
- ThinkPad T480 (Libreboot) ~ baremetal confirmed working, UEFI and BIOS, including LUKS FDE

---

## Building

### Prerequisites

**Build Host:** lainOS layer 02 or Arch Linux with Protocol 7 repository configured

Required packages:
```bash
doas pacman -S archiso base-devel git
```

### Build Steps

1. Clone the repository:
```bash
git clone https://forgejo.lain.rocks/lainOS/lainos-iso-layer-02.git
cd lainos-iso-layer-02
```

2. Build the ISO:
```bash
doas rm -rf ~/lainos-work ~/lainos-out
mkdir -p ~/lainos-work ~/lainos-out
yes "" | doas mkarchiso -v -w ~/lainos-work -o ~/lainos-out protocol7-profile 2>&1 | tee ~/lainos-build.log
```

3. Hash the ISO:
```bash
lainos-hash-iso
```

The resulting ISO will be at `~/lainos-out/lainOS-layer-02-YYYY.MM.DD-x86_64.iso`.

### Protocol 7 Repository

All packages are self-hosted in `protocol_7_repo`. All packages and databases are signed with the LainOS maintainer PGP key. Current package list:

| Package | Purpose |
|---------|---------|
| `protocol7-core` | Core compatibility layer, init scripts, C daemons |
| `protocol7-core-runit` | Runit variant of Protocol 7 |
| `lainos-audio-init` | Optional PipeWire audio orchestration (split from protocol7-core) |
| `lainos-apparmor` | AppArmor profiles for the full LainOS stack (Protocol 7 daemons, DNS layer, networking, media, crypto, browsers, utilities); OpenRC init script loads only Protocol 7 profiles at boot |
| `lainos-keyring` | LainOS PGP public key for pacman verification |
| `lainos-ram-wipe` | RAM-extraction attack defense (LainOS port of Kicksecure ram-wipe): dracut shutdown hook + init_on_alloc/init_on_free kernel parameters |
| `openrc` | Init system and service manager |
| `libeinfo` | OpenRC info library |
| `eudev` | Genuine udev implementation |
| `libudev` | libudev ABI compatibility |
| `systemd` | Dummy ~ satisfies pacman dependencies, no files |
| `systemd-sysvcompat` | Dummy ~ blocks real sysvcompat |
| `elogind` | Dummy ~ blocks real elogind |
| `mkinitcpio` | Dummy ~ blocks real mkinitcpio |
| `sudo` | Dummy ~ blocks real sudo (doas is used instead) |
| `netifrc` | Network interface configuration |
| `dbus-openrc` | OpenRC service for D-Bus |
| `acpid-openrc` | OpenRC service for acpid |
| `chrony-openrc` | OpenRC service for chrony |
| `dhcpcd-openrc` | OpenRC service for dhcpcd |
| `greetd-openrc` | OpenRC service for greetd |
| `iwd-openrc` | OpenRC service for iwd |
| `nftables-openrc` | OpenRC service for nftables |
| `seatd-openrc` | OpenRC service for seatd |
| `syslog-ng-openrc` | OpenRC service for syslog-ng |
| `tor-openrc` | OpenRC service for tor |
| `lainos-hardened-malloc` | GrapheneOS hardened_malloc, light variant |
| `lainos-kvm` | KVM/QEMU virtualization ~ OpenRC init scripts, default NAT network, opt-in `lainos-kvm-enable-firewall` script |
| `lainos-utils` | LainOS utility scripts |
| `sdwdate` | Tor-based, fingerprint-resistant time sync (LainOS port of Kicksecure sdwdate); installed but not enabled by default |
| `bootclockrandomization` | Randomizes system clock at boot for fingerprinting resistance (LainOS port of Kicksecure bootclockrandomization); installed and enabled by default |
| `python-sdnotify` | Pure-Python sd_notify implementation, packaged from upstream PyPI source (dependency of sdwdate) |
| `snowflake-pt-client` | Tor's official Snowflake pluggable transport client |
| `lainos-calamares-dracut` | Calamares binary (services-openrc enabled) |
| `lainos-calamares-config-layer-02` | Calamares configuration and branding |

### Deploying packages to the repo

```bash
cp <package>.pkg.tar.zst ~/Gitlab/protocol_7_repo/x86_64/
cd ~/Gitlab/protocol_7_repo/x86_64/
bash lainos-repo-push
```

The `lainos-repo-push` script signs all unsigned packages, rebuilds the signed database, removes symlinks for GitLab compatibility, and pushes.

---

## Installation

### Live Boot

1. Write the ISO to a USB drive:
```bash
doas dd if=~/lainos-out/lainOS-layer-02-*.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

2. Boot from USB and select **LainOS Layer 02** from the bootloader.

3. At the `tuigreet` login screen, login as `liveuser` (no password required). Calamares launches automatically.

### Installing to Disk

Calamares launches automatically on liveuser login. 

Follow the installer wizard. To enable full disk encryption, select **Encrypt system** on the partition screen. LUKS unlock at boot is handled by dracut's crypt module ~ no systemd-cryptsetup required.

**Note on filesystem:** BTRFS is the default. A separate 1GB ext4 `/boot` partition is created automatically for GRUB compatibility.


### Post-Install Notes

- WiFi: **off by default** ~ run `wifi on` (or `wifi-autostart enable` to restore auto-start at boot), then `wscan`
- Privilege escalation: `doas` (not sudo)
- Power menu: `wlogout`
- Lid close: automatically locks screen with swaylock and suspends
- Audio: PipeWire is bundled by default; `lainos-audio-init` handles orchestration
- Time sync: `chrony` (plaintext NTP) is the default; `sdwdate` (Tor-based, fingerprint-resistant) is installed but not enabled ~ toggle with `lainos-sdwdate enable`
- A short quick-start guide (`lainos-quickstart-help`) opens automatically the first time a new user opens a terminal; the full guide (`lainos-help`) is always available for reference

---

## Session Types

### Sway (Wayland)

LainOS Layer 02 is a Wayland-only distribution. The desktop is Sway with i3status-rs.

**Keybindings:**
| Key | Action |
|-----|--------|
| `Mod4+Return` | Open terminal (alacritty + tmux) |
| `Mod4+Space` | Open application launcher (wofi) |
| `Mod4+Shift+q` | Close focused window |
| `Mod4+1-9` | Switch to workspace |
| `Mod4+Shift+1-9` | Move window to workspace |
| `Mod4+h/j/k/l` | Focus left/down/up/right |
| `Mod4+Shift+h/j/k/l` | Move window left/down/up/right |
| `Mod4+w` | Open LibreWolf |
| `Mod4+f` | Open Thunar |

Full keybinding list, including screenshots, scratchpad, resize mode, and power menu, is in `lainos-help`.

---

## Protocol 7 Compatibility Layer

Protocol 7 is the architectural foundation that enables systemd-free operation while maintaining compatibility with software expecting systemd interfaces.

### Philosophy

> **Protocol 7 is not in a position to own your whole system. systemd, by contrast, is.**

Real `systemd-libs` are used for ABI compatibility ~ the client libraries function fine without systemd running as PID 1. No mock or stub reimplementations are needed. eudev is a genuine, functional udev implementation ~ not a stub.

### Security

Protocol 7 Core has been fuzz tested and hardened:

- **lainos-dbus-bridge:** dfuzzer 2.6 run against the full `org.freedesktop.login1` interface ~ Exit status: 0, all methods and properties PASS. AddressSanitizer + dfuzzer ~ Exit status: 0, no memory errors detected.
- **lainos-notifyd:** libFuzzer harness, 2,000,000 iterations ~ zero crashes. Manual socket fuzz test ~ all inputs survived, log injection sanitized, MSG_TRUNC detection confirmed.
- **lainos-init:** previously verified only by manual code review, despite processing real environment-controlled input (`P7_CMD`/`P7_COMPOSITOR`) that decides what gets exec'd. Two purpose-built libFuzzer harnesses now directly test the compositor-whitelist validation logic and the path-construction overflow-detection pattern ~ 9,102,662 combined executions, zero crashes, supplemented with targeted adversarial test cases (path traversal, command injection, shell substitution).
- **lainos-ghost-units:** confirmed via source review to take zero external input (a fixed loop over eight literal paths) ~ correctly not prioritized for fuzzing, in contrast to lainos-init.
- **Valgrind memcheck:** all daemons confirmed 0 errors, 0 leaks (definite/indirect/possible) under full test passes, independently cross-validating the ASAN/dfuzzer/libFuzzer results (2026-07-09, re-confirmed 2026-07-14 after the seccomp hardening below, and again 2026-07-17 after the capability bounding set drop and filesystem isolation).
- **Manual, logic-focused security review** (2026-07-09) ~ a category of review distinct from fuzzing/memory-safety testing, since a missing authorization check doesn't crash anything. Found that `Reboot`/`PowerOff` D-Bus methods had no caller-identity check; tested and confirmed this was not independently exploitable (the daemon's own privilege drop and `openrc-shutdown`'s own root requirement already blocked it), then fixed anyway as defense-in-depth via `dbus_bus_get_unix_user()`, verified through allow-path/deny-path testing.
- **Seccomp hardening modeled on Whonix/Kicksecure** (2026-07-14) ~ `socket()` restricted to `AF_UNIX` only on both notifyd and dbus-bridge (RestrictAddressFamilies equivalent); `personality()`/`unshare()`/`setns()` confirmed already blocked by the existing default-deny policy (LockPersonality/RestrictNamespaces equivalents, requiring no code change).
- **Capability bounding set explicitly cleared** (5.5.3-25) ~ prompted by external review asking whether the privilege drop clears Linux capabilities or only UID/GID. Verified directly via `/proc/<pid>/status` on live production processes: `setuid()` already cleared `CapEff`/`CapPrm` to zero, but `CapBnd` (the ceiling on regainable capabilities) still showed the full root bounding set. Fixed via `prctl(PR_CAPBSET_DROP, ...)` on both daemons; re-verified post-fix: all five capability fields now read zero. Full dfuzzer + Valgrind regression pass confirmed no functional regression.
- **Filesystem/mount isolation** (5.5.3-26) ~ hand-rolled mount namespaces with no systemd convenience directives available: `unshare(CLONE_NEWNS)` + `MS_PRIVATE` remount, read-only root bind, fresh tmpfs over `/tmp`, empty read-only tmpfs over `/home`/`/root`, and a minimal four-node `/dev` (`null`, `zero`, `random`, `urandom`). Verified incrementally on the real `rc-service`-managed production process: each of four checkpoints confirmed active via direct `/proc/<pid>/mounts` inspection, followed by full dfuzzer regression and Valgrind memcheck, all clean. Binary hash-verified against production at each checkpoint.
- **AppArmor MAC layer** (lainos-apparmor 1.2.1-29) ~ per-daemon profiles for the three external-input components (`lainos-dbus-bridge`, `lainos-notifyd`, `lainos-init`), loaded at boot by an OpenRC init script before the daemons start. The `lainos-apparmor` init script loads only Protocol 7 profiles, not all of `/etc/apparmor.d/*` indiscriminately. Profiles mirror `isolate_filesystem()` exactly and provide path-based denials on `/home/**`, `/root/**`, `/etc/shadow`, and `/sys/kernel/security/**` as defense-in-depth beneath the mount namespace. Verified via a 36-test adversarial suite, including a genuine negative control (`/var/tmp` write denied under AppArmor despite DAC alone allowing it) and cross-daemon ptrace denial. 36/36 passed.
- All daemons drop to `nobody` within seconds of startup ~ systemd-logind runs as root for the lifetime of the system
- seccomp whitelists applied after privilege drop on all security-sensitive daemons

### Core Components

| Component | Binary | Role |
|-----------|--------|------|
| `lainos-init` | `/usr/libexec/lainos/lainos-init` | Session initializer ~ detects Wayland, sets environment, execs compositor |
| `lainos-dbus-bridge` | `/usr/libexec/lainos/lainos-dbus-bridge` | `org.freedesktop.login1` D-Bus facade ~ fuzz tested, runs as nobody |
| `lainos-notifyd` | `/usr/libexec/lainos/lainos-notifyd` | `sd_notify` socket sink ~ fuzz tested, runs as nobody |
| `lainos-ghost-units` | `/usr/libexec/lainos/lainos-ghost-units` | Creates `/run/systemd/*` ghost directories |
| `lainos-audio-init` | `/usr/libexec/lainos/lainos-audio-init` | PipeWire + WirePlumber + pipewire-pulse orchestration (optional package) |
| `lainos-machine-id` | `/etc/init.d/lainos-machine-id` | Generates random `/etc/machine-id` on every boot |
| `cgroup-delegate` | `/etc/init.d/cgroup-delegate` | cgroup2 mount + controller delegation |

### Design Decisions

| Decision | Rationale |
|----------|-----------| 
| Real `systemd-libs`, not mocks | Simpler ABI compatibility |
| Real `eudev`, not stubs | Genuine udev implementation, full device event support |
| `dbus` in `sysinit` runlevel | Ensures D-Bus starts before all dependents |
| `cgroup-delegate` and `lainos-ghost-units` in `boot` runlevel | Dependencies not available in sysinit |
| `lainos-ghost-units` after `syslog-ng` | Prevents syslog-ng treating system as systemd-based (sd_booted() fix) |
| `seatd` instead of `logind` | Minimalist seat management |
| `doas` instead of `sudo` | Smaller attack surface |
| No `elogind` | Protocol 7 daemons handle logind responsibilities directly |
| Self-hosted OpenRC stack | No Artix dependency, full control over packaging |
| `lainos-audio-init` as optional package | Separation of concerns; not all systems need PipeWire orchestration |
| `iwd` not in the default runlevel | WiFi off by default is a deliberate privacy posture, not an oversight ~ `wifi-autostart` is the sole opt-in way to change it |
| `rfkill-unblock` in the `boot` runlevel | Unblocks a soft-blocked WiFi radio at the kernel level on Libreboot/thinkpad_acpi hardware; runs before `iwd` and does not itself start networking |

---

## System Hardening

### Kernel Parameters

`GRUB_CMDLINE_LINUX`:
```
random.trust_cpu=off init_on_alloc=1 init_on_free=1 page_alloc.shuffle=1
```

`GRUB_CMDLINE_LINUX_DEFAULT` additionally carries `wiperam=skip` when RAM-wipe's shutdown-time pass is disabled via `ram-wipe disable` ~ absent (default) means the wipe runs at every shutdown/reboot.

### sysctl Configuration

`/etc/sysctl.d/99-lainos-hardening.conf`:
```ini
# Network
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.tcp_rfc1337 = 1

# Kernel
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.sysrq = 0
kernel.yama.ptrace_scope = 1
kernel.kexec_load_disabled = 1
kernel.unprivileged_userns_clone = 0
kernel.randomize_va_space = 2
kernel.perf_event_paranoid = 3

# Filesystem
fs.suid_dumpable = 0
kernel.core_pattern = |/bin/false

# Memory
vm.max_map_count = 1048576
```

**Building software with sandboxed build requirements:** some software (LibreWolf's own build process, confirmed) needs unprivileged user namespaces for its own build-time sandboxing. Temporarily enable it for builds that need it, then restore the hardened default afterward:
```bash
doas sysctl -w kernel.unprivileged_userns_clone=1
# build your software
doas sysctl -w kernel.unprivileged_userns_clone=0
```
Resets to the hardened default automatically on reboot either way.

### AppArmor Mandatory Access Control

The `lainos-apparmor` package (1.2.1-29) provides an OpenRC init script (`/etc/init.d/lainos-apparmor`) that loads only the three Protocol 7 profiles at boot, before `lainos-dbus-bridge` and `lainos-notifyd` start. It does not indiscriminately load all profiles from `/etc/apparmor.d/*`. The package as a whole ships 20+ profiles covering the full LainOS stack ~ Protocol 7 daemons, the DNS mediation layer (`dnsmasq`, `unbound`, `dnscrypt-proxy`), networking (`tor`, `iwd`, `dhcpcd`, `stubby`, `snowflake-pt-client`), media (`pipewire`, `wireplumber`, `mpv`, `vlc`), crypto/secrets (`gpg`, `gpg-agent`, `keepassxc`), browsers (`librewolf`, Tor Browser standalone), and system utilities (`chronyd`, `syslog-ng`, `nft`, `ssh`, `sshd`).

- **Seccomp + mount namespaces** restrict *what the process can do* (syscalls, filesystem layout inside its private namespace).
- **AppArmor** restricts *where the process can go* in the host filesystem, providing a second line of defense if namespace isolation is bypassed.

All three Protocol 7 profiles deny network access independently of seccomp, and deny access to sensitive paths (`/home`, `/root`, `/etc/shadow`, `/sys/kernel/security`) independently of the mount namespace. The `lainos-dbus-bridge` and `lainos-notifyd` profiles explicitly grant the capabilities required for pre-drop mount namespace setup (`sys_admin`, `chown`, `fowner`, `setgid`, `setuid`, `setpcap`, `dac_override`, `mknod`) and mirror the mount rules from `isolate_filesystem()` exactly. The `lainos-init` profile denies all capabilities (it runs as user) and whitelists compositor paths to match the C-code `compositor_is_safe()` validation.

The `dnscrypt-proxy` profile is restricted to encrypted socket operations, relay/resolver list caching, and HTTPS resolver-list downloads via the built-in CA trust store, with no access to `unbound`'s cache or validation keys. The `unbound` profile is restricted to resolver operations, cache directories, and localhost forwarding to `dnscrypt-proxy`.

Verified via a 36-test adversarial suite (`protocol7-core-security-status`) covering enforcement mode, privilege drop, capability/seccomp state, mount namespace isolation (including bidirectional `/tmp` isolation and empty `/home`/`/root`), a genuine AppArmor negative control (`/var/tmp` write denied despite DAC alone allowing it ~ the test that actually distinguishes real enforcement from coincidental DAC coverage), cross-daemon ptrace denial, network socket/profile-text checks, and `lainos-init`'s `P7_CMD` whitelist logic. 36/36 passed.

### Filesystem and Mount Isolation

Protocol 7 daemons implement systemd-style filesystem sandboxing directly via mount namespaces, since OpenRC has no `ProtectSystem=`/`PrivateTmp=`/`ProtectHome=`/`PrivateDevices=` convenience directives:

| systemd Directive | Protocol 7 Implementation | Status |
|-------------------|---------------------------|--------|
| `ProtectSystem=strict` | `unshare(CLONE_NEWNS)` + `MS_PRIVATE` remount + `MS_BIND` / `MS_RDONLY` remount of `/` | 5.5.3-26 |
| `PrivateTmp=true` | Fresh 16MB `nosuid`/`nodev` tmpfs over `/tmp` | 5.5.3-26 |
| `ProtectHome=true` | Empty read-only tmpfs over `/home` and `/root` | 5.5.3-26 |
| `PrivateDevices=true` | Minimal tmpfs over `/dev` with only `null`, `zero`, `random`, `urandom` | 5.5.3-26 |

Isolation failures on the read-only-root and `/tmp`/`/dev` steps are fatal (`_exit(1)`) rather than silently ignored. All isolation is verified via direct `/proc/<pid>/mounts` inspection on the live production process, not just at build time.

### Capability Dropping

All Protocol 7 daemons follow a consistent privilege model:

1. Start as root ~ required for socket binding, directory creation, D-Bus name acquisition, mount namespace setup
2. Complete privileged setup
3. Drop to `nobody` (UID 65534, GID 65534)
4. Explicitly clear the capability bounding set via `prctl(PR_CAPBSET_DROP, ...)` ~ closes the residual ceiling on regainable capabilities that `setuid()` alone does not clear
5. Apply seccomp whitelist
6. Enter main loop with minimal privileges, minimal syscall surface, and zero capabilities of any kind

Verified directly on live production processes via `/proc/<pid>/status`: all five capability fields (`CapInh`, `CapPrm`, `CapEff`, `CapBnd`, `CapAmb`) read zero.

`dnscrypt-proxy` follows the same pattern via OpenRC's `command_user` directive, since it has no built-in privilege-drop mechanism of its own (unlike `dnsmasq`, which drops to `nobody` internally by default): it starts as root to bind its socket and complete pre-drop setup, then `command_user="dnscrypt-proxy:dnscrypt-proxy"` hands the running process off to a dedicated unprivileged user before it enters its main loop.

### hardened_malloc

GrapheneOS hardened_malloc (light variant) is preloaded via `LD_PRELOAD` wrappers for:
- alacritty, element, gnome-keyring-daemon, keepassxc, kleopatra, mpv
- tor daemon via `/etc/conf.d/tor`

Incompatible applications (mozjemalloc or bwrap/glycin conflicts): librewolf, torbrowser-launcher, signal, thunar, virt-manager. Since `alacritty` forces `hardened_malloc` regardless of the system-wide toggle, launching an incompatible Electron app (e.g. Signal) from a terminal inherits it and crashes with `fatal allocator error: invalid uninitialized allocator usage`. Routing through `tor1`-`tor4` works around this automatically for detected Electron apps (`LD_PRELOAD` stripped before launch); launching plain `signal-desktop` from a terminal or an unmodified `.desktop` entry does not get this protection and will still crash.

### Verifying Security Posture

The `lainos-security-suite` repository ships two scripts for inspecting and testing the state described throughout this section, rather than taking it on faith:

- **`lainos-security-status`** ~ read-only status dashboard: package versions, AppArmor profile counts and enforcement state, OpenRC service status, denial counts, connectivity/NTP state, and per-daemon hardening state (capability bounding set, seccomp mode, privilege drop). Safe to run any time.
- **`protocol7-core-security-status`** ~ a 36-test adversarial suite that actively attempts namespace escapes, blocked-path access, cross-daemon ptrace, and capability abuse, then confirms each is denied ~ including a genuine AppArmor negative control (a world-writable path that DAC alone would allow, confirming AppArmor itself is the thing blocking it, not incidental permissions). Restarts both Protocol 7 daemons on completion.

```bash
doas lainos-security-status
doas protocol7-core-security-status
```

### Full Disk Encryption

LUKS encryption is supported and confirmed working via Calamares (opt-in at install time). Unlock at boot is handled by dracut's `crypt` and `crypt-lib` modules ~ no systemd-cryptsetup required.

### Privacy Features

- Ephemeral machine-id ~ regenerated every boot
- MAC randomization ~ iwd randomizes WiFi MAC per boot; `eth0` does the same for wired
- No systemd-resolved ~ centralized DNS mediation via dnsmasq, not per-application resolvers
- Encrypted DNS ~ `dnsmasq` → `unbound` → `dnscrypt-proxy` chain (`lainos-dns encrypted`), providing DNSSEC validation, QNAME minimization, and anonymized-relay routing so no single component sees both the user's IP and the query; mode preserved across private-mode sessions. `stubby` remains as a fallback if `unbound` is not installed
- IPv6 disabled ~ prevents VPN/Tor IPv6 leaks
- gnome-keyring ~ encrypted secrets storage for Electron apps (note: Electron's keyring backend detection doesn't recognize Sway as a supported desktop by default; `tor1`-`tor4` work around this for apps launched through them by overriding `XDG_CURRENT_DESKTOP`)

---

## Network Configuration

### DNS

LainOS uses a centralized DNS mediation architecture: all applications resolve through `127.0.0.1:53` (dnsmasq), which forwards to the appropriate upstream based on mode. The operating system controls resolver policy; applications require no configuration changes.

**Modes:**
- **Plaintext** (default) ~ DHCP-provided resolver with fallbacks to 1.1.1.1/9.9.9.9
- **Encrypted** ~ `dnsmasq` → `unbound` (`127.0.0.1:5053`, DNSSEC validation + caching) → `dnscrypt-proxy` (`127.0.0.1:5300`, wire encryption + anonymized relay routing) (`lainos-dns encrypted`). No single component in the chain sees both the user's IP and the query. `stubby` is used automatically instead of `unbound` if `unbound` is not installed.
- **Private** ~ Tor DNSPort on `127.0.0.1:9059` (`private-mode on`)

Mode transitions are explicit and stateful; `private-mode` remembers and restores your previous mode (plaintext or encrypted) on exit, and both `lainos-dns` and `private-mode` wait for an actual network route (not just an interface being "up") before attempting to bootstrap the encrypted chain ~ important on WiFi, where reconnecting via `wscan` after a radio toggle isn't instantaneous.

```bash
lainos-dns plaintext    # Plaintext fallbacks
lainos-dns encrypted    # Encrypted: unbound + dnscrypt-proxy (or stubby fallback)
lainos-dns private      # Tor DNSPort, via private-mode
lainos-dns status       # Show current mode and full chain status
```

### WiFi (iwd)

**Off by default.** Turn on and connect using lainos-utils:
```bash
wifi on
wscan
```

Or manually:
```bash
doas rc-service iwd start
iwctl
[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
[iwd]# station wlan0 connect SSID
```

MAC randomization happens automatically every time iwd starts. `wscan` also disables `AutoConnect` on the network immediately after connecting by default (toggle via `wifi-autoconnect`), so nothing reconnects silently on its own in a future session.

If the radio still shows soft-blocked immediately after boot on Libreboot/`thinkpad_acpi` hardware, see [Troubleshooting](#troubleshooting).

### Wired (dhcpcd)

Automatic on boot via dhcpcd in the default runlevel. Toggle the interface and randomize its MAC with `eth0 {on|off|status}`.

---

## lainos-utils

A set of utility scripts included with LainOS Layer 02:

| Script | Command | Purpose |
|--------|---------|---------|
| wscan | `wscan` | iwd scan and connect with numbered network menu; disables AutoConnect after connecting unless `wifi-autoconnect` says otherwise |
| wifi | `wifi {on\|off\|status}` | Full WiFi radio on/off (rfkill block + iwd service), not just disconnect; `wifi on` also restarts the active DNS mode's proxy chain if needed |
| wifi-autostart | `wifi-autostart {enable\|disable\|status}` | Whether iwd starts automatically at boot (off by default) |
| wifi-autoconnect | `wifi-autoconnect {enable\|disable\|status}` | Preference `wscan` reads to decide whether to leave AutoConnect on for newly-connected networks (default: disabled) |
| eth0 | `eth0 {on\|off\|status}` | Wired interface on/off + MAC randomization via macchanger |
| wg-vpn | `wg1`/`wg1d` ~ `wg4`/`wg4d` | WireGuard VPN tunnel up/down (supports up to 4 tunnels) |
| torctl | `torctl` | Start/stop Tor routing |
| snowflake | `snowflake {enable\|disable\|status}` | Toggle Snowflake pluggable transport bridge |
| obfs4 | `obfs4 {enable\|disable\|status}` | Toggle obfs4 pluggable transport bridge |
| lainos-sdwdate | `lainos-sdwdate {enable\|disable\|status}` | Toggle between chrony (default) and Tor-based sdwdate |
| tor-tunnel | `tor1`/`tor2`/`tor3`/`tor4` | Four dedicated, permanently isolated Tor circuits, matching the wg1-wg4 model. Works identically for CLI tools, GTK apps, and Electron apps (auto-detected and routed via `--proxy-server`, with `LD_PRELOAD`/`XDG_CURRENT_DESKTOP` fixes applied automatically). Firefox-based browsers need native proxy config instead |
| usb-automount | `usb-automount {enable\|disable\|status}` | Toggle ext2/3/4 removable-drive auto-mount (off by default, matching e2fsprogs' own security default) |
| private-mode | `private-mode {on\|off\|status}` | One command for the sensitive-work sequence: forces network down, enables snowflake/sdwdate, brings network back up to bootstrap the DNS chain and Tor, then switches dnsmasq to Tor DNSPort; reverses safely on shutdown, restoring previous DNS mode (plaintext or encrypted) |
| ram-wipe | `ram-wipe {enable\|disable\|status}` | Toggle the shutdown-time RAM wipe pass (continuous init_on_alloc/init_on_free protection is always on regardless) |
| kloak | `kloak` | Keystroke anonymization (auto-detects keyboard) |
| brightness | `brightness` | Screen brightness control |
| lainos-secure-messaging | `lainos-secure-messaging` | Automated onion XMPP secure messaging setup (LESME); connects via `tor1` by default |
| ani-cli | `ani-cli` | Anime streaming CLI |
| virtman | `virtman` | KVM/QEMU VM manager wrapper |
| lainos-dns | `lainos-dns {plaintext\|encrypted\|private\|status}` | Toggle between plaintext, encrypted (unbound + dnscrypt-proxy, or stubby fallback), and private (Tor) DNS; preserves mode across private-mode sessions |
| lainos-hardened-malloc | `lainos-hardened-malloc {enable\|disable\|status}` | Toggle system-wide hardened_malloc via LD_PRELOAD in /etc/environment |
| lainos-help | `lainos-help` | Opens the LainOS Layer 02 user guide with glow |
| lainos-privacy-help | `lainos-privacy-help` | Opens the dedicated privacy guide for sensitive-work sessions |
| lainos-quickstart-help | `lainos-quickstart-help` | Opens a short quick-start checklist; shown automatically on first terminal open for a new user |
| openrc-help | `openrc-help` | Opens an OpenRC cheat sheet for people new to OpenRC |


---

## Troubleshooting

### Calamares Session Log

```bash
doas grep -n "" /root/.cache/calamares/session.log
# View specific line range:
sed -n '100,200p' /root/.cache/calamares/session.log
```

### Boot Log

```bash
cat /var/log/rc.log
```

### Daemon Logs

Protocol 7 daemons log to `/var/log/daemon.log`:
```bash
doas tail -f /var/log/daemon.log
```

### Verifying AppArmor / Protocol 7 Hardening State

If something seems off with confinement, privilege drop, or seccomp (or you just want to confirm current state), use the `lainos-security-suite` scripts rather than manually checking `/proc` and `aa-status` by hand:
```bash
doas lainos-security-status              # read-only status dashboard
doas protocol7-core-security-status      # active adversarial test suite (36 tests)
```

### WiFi Soft-Blocked After Boot (Libreboot + thinkpad_acpi)

On Libreboot systems with `thinkpad_acpi force_load=1`, wifi may be soft-blocked on boot. This is handled automatically via a dedicated `rfkill-unblock` OpenRC service in the `boot` runlevel (runs before `iwd`/networking, unblocking the radio at the kernel level without starting `iwd` itself or affecting the WiFi-off-by-default posture). If manual intervention is needed:
```bash
doas rfkill unblock all
doas rc-service iwd restart
```



### Electron App Reports a Keyring/Database Encryption Mismatch

If an Electron app reports something like *"the OS encryption keyring backend has changed"*, it was likely first run with `XDG_CURRENT_DESKTOP` set differently than it's being launched with now (e.g. once via `tor1`, which overrides this for keyring-detection reasons, and once launched normally, where Sway's real value applies). The app's local database is locked to whichever backend encrypted it first. Either always launch the app the same way going forward, or reset its local profile to start fresh under a consistent backend.

---

## Project Structure

```
lainos-iso-layer-02/
~~ protocol7-profile/              # ISO build profile
   ~~ airootfs/                   # Overlay files ~ ISO chroot
   ~  ~~ etc/
   ~  ~  ~~ acpi/                 # Lid close event handler
   ~  ~  ~~ conf.d/               # Service environment (tor LD_PRELOAD)
   ~  ~  ~~ default/grub          # GRUB configuration
   ~  ~  ~~ dnsmasq.conf*         # DNS config templates (plaintext/encrypted/private)
   ~  ~  ~~ dracut.conf.d/        # Dracut initramfs config
   ~  ~  ~~ init.d/               # OpenRC init scripts
   ~  ~  ~~ runlevels/            # Service runlevel symlinks
   ~  ~  ~~ greetd/
   ~  ~  ~~ sysctl.d/             # Kernel hardening
   ~  ~  ~~ skel/                 # Default user configs (sway, swaylock, wlogout)
   ~  ~  ~~ stubby/               # DoT proxy configuration (fallback)
   ~  ~  ~~ unbound/              # DNSSEC-validating resolver configuration
   ~  ~  ~~ dnscrypt-proxy/       # Encrypted transport + anonymized relay configuration
   ~  ~  ~~ iwd/
   ~  ~  ~~ syslog-ng/
   ~  ~  ~~ dbus-1/system.d/
   ~  ~  ~~ doas.conf
   ~  ~~ usr/
   ~  ~  ~~ local/bin/            # Hardened_malloc wrappers, lainos-utils
   ~  ~  ~~ share/lainos/wallpapers/
   ~  ~~ root/                    # Root home (live ISO swaylock/wlogout configs)
   ~~ packages.x86_64             # Package list
   ~~ pacman.conf                 # Build-time pacman config
   ~~ profiledef.sh               # ISO metadata, permissions
   ~~ airootfs.sh                 # pacman-key init during build
```

---

## Contributing

LainOS is developed by Grayson Giles and the LainOS community.

- **Forgejo:** https://forgejo.lain.rocks/lainOS/
- **Codeberg:** https://codeberg.org/lainOS
- **GitLab:** https://gitlab.com/lainos
- **GitHub:** https://github.com/The-LainOS-Project
- **Website:** https://lainos.net
- **Security verification tools:** `lainos-security-suite` repository (see the LainOS org above) ~ status dashboard and adversarial test suite for Protocol 7 and AppArmor hardening

### Reporting Issues

Please include:
- ISO version/date
- Hardware/VM configuration
- `rc-status` output
- Relevant logs from `/var/log/rc.log` or `dmesg`

---

## License

LainOS Layer 02 and the Protocol 7 compatibility layer are released under the **GNU General Public License v3.0**.

Individual components (Sway, OpenRC, Calamares, etc.) retain their respective licenses.

---

## Acknowledgments

- **Arch Linux** ~ The foundation everything is built on
- **OpenRC** ~ Reliable, predictable init system
- **GrapheneOS** ~ hardened_malloc
- **Sway/wlroots** ~ Modern Wayland compositor ecosystem
- **Calamares** ~ User-friendly system installer
- **Kicksecure/Whonix** ~ sdwdate, bootclockrandomization, and ram-wipe are LainOS ports of Kicksecure's originals; Protocol 7's seccomp hardening (AF_UNIX socket restriction, personality/namespace lockdown), filesystem isolation (ProtectSystem/PrivateTmp/ProtectHome/PrivateDevices equivalents), and AppArmor profile structure are modeled on the equivalent systemd sandboxing directives Whonix applies to its own services

---

*Last updated: 2026-07-26*
*Current package: protocol7-core-5.5.3-27*
*Status: RC8 ~ release candidate*
