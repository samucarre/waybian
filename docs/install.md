# Installing WayBian on a MacBook Pro 16" M3 Max

How to put the [release image](https://github.com/samucarre/waybian/releases) on a real Mac, by hand, from macOS. There is no package for the Asahi installer yet, but the image is a complete disk (GPT: an ESP and an ext4 root), so its contents can be copied onto the Mac: the root onto a new partition, and the boot files onto the ESP that the Asahi installer creates.

> **Alpha.** These are the steps this machine was installed with (from macOS, by hand), applied to the release image. The image itself has so far been tested in QEMU, not yet installed this way on another Mac. Make a backup (Time Machine) before starting. A `dd` to the wrong disk destroys it.

**Only the 16-inch MacBook Pro with the M3 Max (board `j516c`).** The `boot.bin` in the image carries the device tree for that board only. Other M3 models need their own.

**Español:** [install.es.md](install.es.md).

## What you need

- The MacBook Pro 16" M3 Max with macOS, and its administrator password.
- Free space for WayBian: at least 45 GB. The root is 40 GiB and grows on first boot to fill whatever partition you give it, so leave more if you can.
- The release files and `zstd` (`brew install zstd`). Join and decompress them as the release notes say; you get `waybian-0.15-qemu.img` (40 GB, sparse: it takes about 12 GB).
- About an hour.

## 1. The Asahi installer, "UEFI environment only"

In a macOS Terminal:

```sh
curl https://alx.sh | sh
```

- When it asks how much space to give macOS, shrink it enough to leave the free space for WayBian.
- Choose **"UEFI environment only (m1n1 + U-Boot + ESP)"** and give it a name (for example `WayBian`).
- Follow its instructions to the end: shut down, hold the power button, pick the new entry, and finish the second step in recoveryOS (your macOS password; it lowers the security of that container only, macOS stays in Full Security).

This creates a small stub container, an ESP of about 500 MB with `m1n1/boot.bin` and `vendorfw/` (the Wi‑Fi, Bluetooth and trackpad firmware of your Mac), and leaves the rest of the space free. The new entry boots to U‑Boot with nothing to load yet. Boot back into macOS (hold the power button, pick Macintosh HD) and carry on there. More about what the installer does in [boot-chain.md](boot-chain.md).

## 2. Find the two disks

Attach the image without mounting it:

```sh
hdiutil attach -imagekey diskimage-class=CRawDiskImage -nomount waybian-0.15-qemu.img
```

It prints a disk, say `/dev/disk8`, with `disk8s1` ("EFI - WAYBIAN", the image's ESP) and `disk8s2` ("WayBian", the root).

Then look at the internal disk:

```sh
diskutil list disk0
```

The Asahi installer's ESP is the ~500 MB `EFI` partition named `EFI - ` plus the first letters of the name you gave (`EFI - WAYBI`), just before the free space. Below, it is `disk0sE`: **replace E with its real number**, and `8` with the number `hdiutil` printed.

## 3. Create the Linux partition and write the root

Create a Linux partition in all the free space after that ESP:

```sh
sudo diskutil addPartition disk0sE %0FC63DAF-8483-4772-8E79-3D69D8477DE4% %noformat% 0
diskutil list disk0
```

The new partition shows as `Linux Filesystem`. Call it `disk0sR`, check that it is at least 40 GiB, and write the root onto it. **Check the number three times**: this erases whatever `disk0sR` is.

```sh
sudo dd if=/dev/rdisk8s2 of=/dev/rdisk0sR bs=4m
```

## 4. Put the boot files on the ESP

Mount both ESPs on fixed folders:

```sh
sudo mkdir -p /tmp/esp-mac /tmp/esp-img
sudo diskutil mount -mountPoint /tmp/esp-mac disk0sE
sudo diskutil mount -mountPoint /tmp/esp-img disk8s1
```

Keep the installer's `boot.bin` next to the new one, then copy WayBian's `boot.bin` (m1n1 stage 2 with Aurora's device tree for the J516C and U‑Boot; the native display and the GPU need it) and GRUB:

```sh
sudo cp /tmp/esp-mac/m1n1/boot.bin /tmp/esp-mac/m1n1/boot.bin.asahi
sudo cp /tmp/esp-img/m1n1/boot.bin /tmp/esp-mac/m1n1/boot.bin
sudo mkdir -p /tmp/esp-mac/EFI/BOOT
sudo cp /tmp/esp-img/EFI/BOOT/BOOTAA64.EFI /tmp/esp-mac/EFI/BOOT/BOOTAA64.EFI
```

Leave `vendorfw/` alone: it is your Mac's firmware, and WayBian loads it from there at every boot. GRUB finds the root by the filesystem's UUID, so no partition number has to be written anywhere.

Unmount and detach:

```sh
sudo diskutil unmount /tmp/esp-mac
sudo diskutil unmount /tmp/esp-img
hdiutil detach /dev/disk8
```

## 5. First boot

Shut down and power on. The Asahi installer left the new entry as the default; if macOS starts instead, hold the power button and pick it. The chain is m1n1 → U‑Boot (3 s) → GRUB → WayBian.

On the first boot the root grows to fill its partition, and a wizard asks for the language, keyboard, time zone and computer name, and creates your account.

**The GPU.** Its driver only starts on a firmware it recognises. If yours is not recognised, the desktop is drawn by the CPU and the kernel log shows the firmware's digest (`journalctl -k | grep "unsupported GPU firmware"`, after `sha256`). To accept it, add `asahi.g15_accept_text_prefix=0x<the first 16 hex digits of that sha256>` to the kernel line: once with `e` in the GRUB menu to try it, and for good on the `linux` line of the WayBian entry in `/etc/grub.d/40_custom`, followed by `sudo update-grub`.

**Do not install** Debian's `m1n1`, `u-boot-asahi` or `asahi-platform` packages. They do not know the M3 and their hook would overwrite `boot.bin`. The image already blocks them.

## Going back

- **Back to the installer's `boot.bin`:** from macOS, mount the ESP as in step 4 and `sudo cp /tmp/esp-mac/m1n1/boot.bin.asahi /tmp/esp-mac/m1n1/boot.bin`.
- **Removing everything:** the four `diskutil` steps in [boot-chain.md](boot-chain.md#undoing-everything-from-macos) delete the ESP, the Linux partition and the stub, and give the space back to macOS.
