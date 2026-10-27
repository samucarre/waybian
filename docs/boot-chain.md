# Boot chain

How WayBian starts, what the installer creates on the disk, and why none of it touches macOS.

## The chain

```
Power button
 └─ Boot ROM → iBoot1 (Apple, signed)                                  never touched
     └─ picks an APFS container (the startup selector)
         ├─ macOS (Full Security)                                      never touched
         └─ "Debian": a stub APFS container in Permissive Security
             └─ iBoot2 + macOS 14.8.3 firmware (from the stub)
                 └─ m1n1 stage 1 (signed with a key local to this Mac)
                     └─ ESP/m1n1/boot.bin = m1n1 stage 2 + device trees + U‑Boot
                         └─ U‑Boot (UEFI, 3 s menu) → ESP/EFI/BOOT/BOOTAA64.EFI = GRUB
                             └─ Linux kernel + initramfs on the ext4 root → systemd → greetd → Hyprland
```

Power on with nothing pressed boots Debian (the installer set it as default). Holding the power button shows Apple's startup selector, where macOS is one click away.

## What the Asahi installer put on the disk

The official Asahi installer supports this board (`j516cap`) since 15 Sep 2026 and offers **"UEFI environment only (m1n1 + U‑Boot + ESP)"**. That is the only thing it is used for. It:

1. shrinks the macOS APFS container (macOS keeps ~800 GiB; a local Time Machine snapshot has to be deleted first or the container refuses to shrink by a single byte);
2. creates a **stub APFS container** (~2.5 GB) with a minimal macOS 14.8.3 that provides iBoot2 and the peripheral firmware, and installs m1n1 inside it as the "kernel";
3. creates an **ESP** (FAT32, ~500 MB) with `m1n1/boot.bin`, U‑Boot and `vendorfw/` (Wi‑Fi/Bluetooth/trackpad firmware extracted from macOS);
4. leaves free space for the Linux root.

Lowering the stub to Permissive Security happens in recoveryOS (1TR), with the owner's password, during the installer's second step. Security on Apple Silicon is per container, so macOS stays in Full Security with FileVault and updates working as before. Nothing in the system firmware is modified.

Debian itself was written afterwards from macOS: an ext4 image onto a new partition, GRUB into the ESP (`EFI/BOOT/BOOTAA64.EFI` with a `grub.cfg` that looks for the root by UUID). `boot.bin` and `vendorfw/` were left exactly as the installer made them.

## Undoing everything (from macOS)

1. `diskutil list` — identify the ESP, the Linux partition and the stub container.
2. `diskutil eraseVolume free free diskXsN` for the ESP and the Linux partition.
3. `diskutil apfs deleteContainer diskY` for the stub (this also removes the entry in the startup selector).
4. `diskutil apfs resizeContainer disk3 0` so macOS gets the space back.

Worst case (very unlikely, since the system firmware is never touched) is a DFU restore with Apple Configurator from another Mac. Keep a backup before installing.

## Pinned packages

Debian ships `m1n1`, `u-boot-asahi` and `asahi-platform`, built for M1/M2. Their post-install hook (`zz-update-m1n1`) regenerates `boot.bin` with a m1n1 that does not know the M3, which would leave the Mac unable to boot Linux. They are blocked with `apt-mark hold` plus an apt pin, and `/etc/default/update-m1n1` has `M1N1_UPDATE_DISABLED=1` as a second lock.

`boot.bin` is rebuilt by hand only when a new kernel brings new device trees (m1n1 + the kernel's DTBs + U‑Boot, gzip-concatenated), and the previous `boot.bin` is always kept next to it. Since 25 Oct 2026 it also carries the WayBian logo and a 3 s U‑Boot timeout.

## GRUB layout

| Entry | Kernel | Purpose |
|---|---|---|
| 0–1 | Asahi `asahi` branch 7.1.13 (Fedora Asahi 16k config) | emergency: no GPU, but boots whatever happens |
| 2 | `7.2.11-aurora-j516c` | Aurora kernel with the first local patches |
| 3 | `7.2.11-waybian-0.1` | Aurora of 25 Oct + local patches |
| 4 | `7.2.11-waybian-0.2` (**default**) | Aurora of 26 Oct + 7 local patches |

Any new kernel is installed as a new entry and tried **once** with `grub-reboot` (`next_entry` in `grubenv`): if it hangs, the next reboot returns by itself to the previous default. Only after checking GPU, display, audio and camera does it become `GRUB_DEFAULT`.

Kernel command line, besides the root by UUID:

- `quiet loglevel=3` — the camera driver (`apple-isp`) dumps debug output on the console and would overwrite the login prompt.
- `firmware_class.path=/lib/firmware/vendor` — where the extracted Apple firmware lands; without it the Wi‑Fi and trackpad firmware are not found.
- `deferred_probe_timeout=10` — the USB‑C controllers (`cd321x`) wait for a USB4 driver that does not exist; the timeout lets USB 3 come up anyway.

## initramfs

Debian's initramfs uses klibc, and the upstream Asahi hook that unpacks `vendorfw/firmware.cpio` from the ESP assumed a shell with `cp`. Local hooks (`zz-asahi-m3`):

- copy `cp` into the initramfs and wait for the ESP's `/dev/disk/by-partuuid` link before mounting it;
- check `/vendorfw/.vendorfw.manifest` as upstream does (Debian's hook looked elsewhere) and log the result to `/var/log/asahi-vendorfw.log`;
- provide `/usr/bin/modprobe` — the Fedora kernel config hard-codes that path and Debian has it in `/usr/sbin`, so without the link the kernel could not autoload modules;
- keep `dockchannel-hid` (trackpad/keyboard) out of the initramfs, so it is not probed before its firmware exists. That driver does not survive an unbind/bind (kernel BUG), so it is never reloaded at runtime either.
