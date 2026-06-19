# kde-install-nvidia-configured

A modified FreeBSD KDE Plasma installer script, forked from
[kde-installer-dialogs](https://gitlab.com/alfix/kde-installer-dialogs)
by Alfonso Sabato Siciliano, developed under sponsorship from the FreeBSD Foundation.

This fork fixes critical NVIDIA driver configuration issues that cause kernel panics
on boot, and bundles a set of opinionated extras (Alacritty, Fish shell, Firefox ESR,
Fastfetch) with out-of-the-box theming.

---

## Why This Exists

The upstream installer works great for AMD/Intel GPUs and VM setups. For NVIDIA —
especially modern cards like the RTX 30/40 series — it has two bugs that cause an
immediate kernel panic on the first boot into KDE:

**Bug 1: Wrong kernel module name in `rc.conf`**

The upstream script sets:
```
kld_list="nvidia-drm"
```

`nvidia-drm` is a Linux kernel module name. FreeBSD's NVIDIA driver doesn't use it.
The correct modules on FreeBSD are `nvidia` and `nvidia-modeset`. Using the wrong name
means the kernel attempts to load a nonexistent module at boot, which triggers a page
fault in the DRM/Xorg initialization path.

**Bug 2: `loader.conf` entries missing or malformed**

The upstream script writes `hw.nvidiadrm.modeset="1"` to `loader.conf` but doesn't
write the `nvidia_load` and `nvidia-modeset_load` entries that tell the bootloader to
load the kernel modules early. Without these, Xorg tries to access the GPU before the
driver is present, causing a fatal trap 12 (page fault while in kernel mode) with the
current process shown as Xorg.

**Bug 3: Linux compat not loaded before package installation**

When installing `linux-nvidia-libs` (required for the NVIDIA driver), the package
triggers scripts that need the `linux64` kernel module loaded. If it isn't, pkg throws
errors mid-installation. The upstream script enables `linux_enable="YES"` in `rc.conf`
but doesn't `kldload linux linux64` before the install begins.

**Bug 4: No Xorg device section for NVIDIA**

Without an explicit `Driver "nvidia"` entry in an Xorg config file, Xorg tries to
autodetect the driver. On FreeBSD with the proprietary NVIDIA driver, this autodetection
can fall back to the wrong driver or panic during device open. The fix is a minimal
`/etc/X11/xorg.conf.d/20-nvidia.conf` with the driver explicitly named.

---

## What This Script Does Differently

### NVIDIA Fixes

- Sets `kld_list="nvidia nvidia-modeset"` in `/etc/rc.conf` (correct FreeBSD module names)
- Writes proper `loader.conf` entries:
  ```
  nvidia_load="YES"
  nvidia-modeset_load="YES"
  hw.nvidiadrm.modeset=1
  ```
- Runs `kldload linux linux64` before any package installation begins
- Always enables `linux_enable="YES"` for NVIDIA (required for `linux-nvidia-libs`)
- Creates `/etc/X11/xorg.conf.d/20-nvidia.conf` with an explicit `Driver "nvidia"` device section
- Installs the full NVIDIA package set: `nvidia-driver`, `nvidia-kmod`, `nvidia-drm-kmod`,
  `nvidia-settings`, `nvidia-xconfig`, `nvidia-texture-tools`, `linux-nvidia-libs`, `linux-nvidia-libs32`

### Added Packages (installed by default)

| Package | Why |
|---|---|
| `alacritty` | GPU-accelerated terminal, replaces Konsole |
| `fish` | User-friendly shell with better defaults than sh/bash |
| `firefox-esr` | Stable browser, no Flatpak dependency |
| `fastfetch` | System info on terminal launch |

### Shell & Terminal Setup (per user)

- Fish is set as the default login shell via `chsh` for each selected user
- `$FISH_BIN` is added to `/etc/shells` if not already present
- Fish config includes a minimal custom prompt (`❯ path HH:MM`) and launches
  `fastfetch` on every new interactive session
- Alacritty is configured with Catppuccin Mocha colors, semi-transparency, and
  cursor blinking
- KDE is told to use Alacritty as its default terminal application via `kdeglobals`

---

## Requirements

- FreeBSD 14.x or 15.x (tested on 15.1-RC2)
- amd64 architecture
- NVIDIA GPU (RTX 20 series or newer — use `nvidia-drm-kmod`)
- Internet access for `pkg`
- At least one non-root user account created before running

---

## Usage

Run as root:

```sh
sh kde-install-nvidia-configured
```

The script is interactive and uses `bsddialog` (bundled with FreeBSD base) for its UI.
You will be prompted to:

1. Confirm you want to proceed
2. Select your GPU type (NVIDIA autodetected if present)
3. Select your NVIDIA driver version
4. Select a display manager (SDDM recommended)
5. Select which users to configure

After completion, reboot and SDDM will start automatically.

---

## File Layout After Installation

```
/boot/loader.conf              ← nvidia_load, nvidia-modeset_load, hw.nvidiadrm.modeset
/etc/rc.conf                   ← kld_list="nvidia nvidia-modeset", linux_enable, sddm_enable
/etc/X11/xorg.conf.d/
  20-nvidia.conf               ← Driver "nvidia" device section
~/.config/alacritty/
  alacritty.toml               ← Catppuccin Mocha theme, opacity, font config
~/.config/fish/
  config.fish                  ← Custom prompt, fastfetch on launch, aliases
~/.config/kdeglobals           ← Sets Alacritty as KDE default terminal
```

---

## License

This script is derived from `kde-installer-dialogs` by Alfonso Sabato Siciliano,
copyright (c) 2025-2026 The FreeBSD Foundation, and is redistributed under the
**BSD 2-Clause License**. The full license text is reproduced below as required
by the license terms.

```
Copyright (c) 2025-2026 The FreeBSD Foundation

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions
are met:

1. Redistributions of source code must retain the above copyright
   notice, this list of conditions and the following disclaimer.

2. Redistributions in binary form must reproduce the above copyright
   notice, this list of conditions and the following disclaimer in
   the documentation and/or other materials provided with the
   distribution.

THIS SOFTWARE IS PROVIDED BY THE AUTHOR AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE AUTHOR OR CONTRIBUTORS
BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY,
OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT
OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR
BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE
OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE,
EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```

Modifications in this fork are by Sriram ([@thatguyspr](https://github.com/thatguyspr))
and are released under the same BSD 2-Clause License.

---

## Upstream

Original project: https://gitlab.com/alfix/kde-installer-dialogs  
Author: Alfonso Sabato Siciliano  
Sponsor: The FreeBSD Foundation
