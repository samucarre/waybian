# How the system is built and reproduced

The scripts themselves live in the private lab repository (`waybian-lab`); this page describes what they do so the process can be followed or improved.

## Base system (once, from a Mac)

| Step | Script | What it does |
|---|---|---|
| Kernel | `00-kernel.sh` | Clones the kernel tree, applies the 16 KiB Fedora Asahi configuration (today: Aurora's tree with local patches), builds with LLVM (`-j6` under `nice`), produces the `.deb` with modules and device trees. |
| Root filesystem | `01-rootfs.sh` | `debootstrap trixie` arm64 into an ext4 image: kernel package, `initramfs-tools`, `grub-efi-arm64`, firmware, NetworkManager + iwd, Asahi's `asahi-scripts`, `speakersafetyd`, `asahi-audio`; Hyprland and the desktop from `trixie-backports`; user, time zone, Spanish keyboard. |
| System tweaks | `02-sistema.sh` | greetd on tty7, `loglevel=3`, `firmware_class.path`, `deferred_probe_timeout`, pins for `m1n1`/`u-boot-asahi`/`asahi-platform`, the initramfs hooks, `kernel.printk`. |
| Image | `03-imagen.sh` | Packs the root as `debian-m3-root.img.zst` (12 GiB ext4 that grows on first boot) and a GRUB `BOOTAA64.EFI` that looks for the root by UUID. |
| Deploy | `04-desplegar-mac.sh` | From macOS: creates the Linux partition in the free space left by the Asahi installer, writes the image to the raw device, verifies it by reading it back, copies GRUB to the ESP and refuses to touch `m1n1/boot.bin` or `vendorfw/`. |

The image was tested before touching the Mac in QEMU with HVF, booting the same way (UEFI firmware → `BOOTAA64.EFI` → GRUB → kernel, `ramfb` → `simple-framebuffer`), which is also how the native partition can be booted from macOS as a VM for repairs.

## Everything after the first boot

Every change is recorded in three places in the lab repository:

1. `CAMBIOS.md`: what, why and how, with date.
2. `config/`: a mirror of each touched file at its real path (`config/home` = `~`, `config/etc`, `config/usr`).
3. `desde-cero/`: package lists (apt, backports, pip, npm), systemd units to enable, dconf/gsettings values, external apps (repositories or GitHub releases with checksums), MCP servers, and `aplicar.sh`, an idempotent script with three verbs: `comprobar` (diff the live system against the mirror and report what is missing), `aplicar` (install packages and apps, copy the files with their permissions, enable units, apply settings, reload the desktop) and `actualizar-espejo` (refresh the mirror from the live system).

`aplicar.sh` never touches the kernel, GRUB, initramfs or the ESP: those remain manual, documented steps (`instalar-kernel-waybian.sh <version>`, `instalar-bootbin.sh <letter>`, with a one-shot GRUB entry first). Programs built locally (Waybar, aquamarine, Postcard, the portal, libva, Mesa) are rebuilt from their forks with `compilados.sh compilar <name>`.

## Backups

A weekly `rsync` of the whole root (minus `/proc`, `/sys`, caches and kernel build trees) and of the ESP to a NAS over Tailscale, owners preserved in xattrs. Restoring is: format the root partition with the **same UUID** (GRUB and `fstab` use it), rsync back, restore the ESP (which brings `boot.bin` and `vendorfw/` with it), and regenerate the initramfs from a chroot if needed.
