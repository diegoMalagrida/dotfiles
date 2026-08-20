# Notes on the package lists

`pacman.txt` and `aur.txt` are a snapshot of the **explicitly installed**
packages (`pacman -Qqen` and `pacman -Qqem`) on the laptop as of 2026-08-13.
Dependencies are not listed, because pacman pulls those in by itself.

## What to look at before installing on another machine

**Hardware-bound packages.** This laptop has an Intel CPU/GPU. If the new
machine is not Intel, drop these from `pacman.txt`:

    intel-ucode  intel-media-driver  intel-compute-runtime  vulkan-intel  level-zero-loader

and put the equivalents in (`amd-ucode`, `vulkan-radeon`, `libva-mesa-driver`...).
Installing the wrong vendor's microcode breaks nothing, but it does nothing
either.

**Base system packages.** `base`, `base-devel`, `linux`, `linux-firmware`,
`grub`, `efibootmgr`, `os-prober` and `dosfstools` are already on any freshly
installed Arch. They stay in the list on purpose: pacman skips whatever is
already there, and that way the list remains the complete snapshot. If you start
from a minimal `archinstall`, it fills them in for you as a bonus.

**The bootloader.** `grub` is in the list, but the installer **does not touch**
`/boot` and never runs `grub-install`. If the new machine uses systemd-boot,
ignore that package; the installer is not going to argue with your boot setup.

## The `yay` chicken-and-egg problem

`yay` shows up in `aur.txt` but cannot be installed with `yay`. The installer
builds it from the AUR with `makepkg` the first time (it needs `git` and
`base-devel`) and from then on uses yay for the rest of `aur.txt`.

## The packages holding the rice up

If you ever trim the list, these are the ones you **cannot** remove without the
desktop failing to start or losing pieces of itself:

| Package | What for |
|---|---|
| `hyprland`, `uwsm`, `xdg-desktop-portal-hyprland` | the compositor and its session |
| `quickshell`, `qt6-quicktimeline`, `qt6-sensors` | the bar and the notch (all the QML) |
| `hypridle`, `hyprlock`, `hyprpolkitagent`, `hyprsunset` | idling, locking, polkit, colour temperature |
| `awww` | the thing that paints the wallpaper |
| `python-pywal`, `python-colorthief` | the global recolouring from the wallpaper |
| `sddm` + `sddm-silent-theme` | the login (the `hyprisland` theme itself comes from your config) |
| `cliphist`, `grim`, `slurp`, `satty`, `hyprshot` | clipboard history and screenshots. The launcher and the power menu are the shell's own — `rofi` and `wlogout` were dropped on 2026-08-19, once nothing called them |
| `cava`, `playerctl` | audio visualiser and playback control |
| `kitty`, `zsh`, `ttf-jetbrains-mono-nerd` | terminal and its font |
| `ydotool` | the gestures and automation that synthesise keystrokes |
| `zram-generator` | used by the OOM protection in `system/etc` |

## Careful

`ttf-jetbrains-mono-nerd` is the font the shell and the terminal ask for, and it
does **not** show up in `pacman -Qqen` because here it arrived as a dependency.
The installer installs it separately, by hand. Same goes for
`ttf-nerd-fonts-symbols-common`.
