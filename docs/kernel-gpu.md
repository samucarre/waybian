# Kernel and GPU

## Why not mainline, and why not a distribution

- Mainline Linux 7.3 has the device tree for this board (`t6031-j516c.dts`), but it is a minimal one: CPUs, interrupts, serial console, pinctrl, I2C, the boot framebuffer and the keyboard backlight PWM. The NVMe controller node (`apple,t6031-nvme-ans2`), keyboard, trackpad and Wi‑Fi only exist in Asahi's tree. Mainline will catch up over time.
- Debian's Apple Silicon installer ("Bananas") accepts firmware 12.3/13.5 only, i.e. M1/M2. The Asahi installer supports this board with firmware 14.8.3 and has the "UEFI environment only" mode WayBian relies on.
- Asahi's own M3 announcement (September 2026) lists what works (keyboard, trackpad, NVMe, Wi‑Fi, Bluetooth, camera, microphones, audio, USB 3, video decode) and what does not (GPU, suspend, HDMI, full DCP).

## Kernel lineage

| Name | Base | Config | Notes |
|---|---|---|---|
| v1 (22 Oct) | `AsahiLinux/linux`, `asahi` branch, 7.1.13 | Fedora Asahi `kernel-aarch64-16k` config | Built with LLVM in an arm64 VM. The plain `defconfig + asahi.config` lacked `DRM_SIMPLEDRM`, and without it there is no desktop on the M3. Kept as GRUB's emergency entry. |
| `7.2.11-aurora-j516c` (24 Oct) | Aurora Silicon `integration/aurora` 7.2.11 | same, adapted | First kernel with the GPU. External HDMI ported. Needed a new `boot.bin` with Aurora's device tree. |
| `7.2.11-waybian-0.1` (26 Oct) | Aurora of 25 Oct | `CONFIG_LOCALVERSION=-waybian-0.1` | Local patches on top; tuned the GPU driver's in-flight job depth, which removed stutter in games. |
| `7.2.11-waybian-0.2` (26 Oct, **default**) | Aurora of 26 Oct (`integration/aurora`, 225 commits newer) | same | Rebase + 7 local patches (panel brightness on J516C, HDMI, and small fixes). |

Build: `make -C linux O=build ... -j6` under `nice` (the machine is in use while it builds; a full build takes under two minutes on the M3 Max). Install script: modules, `/boot/aurora/{vmlinuz,initrd.img,System.map,config}-<version>`, a GRUB entry, the per-version AVD firmware, and `next_entry` for a one-shot test. `deshacer` removes all of it.

Lessons that cost a boot each:

- **Nothing may depend on the kernel's name.** The GPU service (`ConditionKernelVersion`), the session script that chooses GPU vs software rendering and the firmware directory were all keyed on the previous kernel name; the first `waybian-0.1` boot was a black screen for that reason alone.
- **Every rebase must re-apply every local patch.** The brightness patch was dropped once because a branch comparison listed it as upstream; it is not.
- Keep the previous `boot.bin` and the previous kernel entry until the new one has shown GPU, display, audio and camera working.

## GPU

The M3 brings a new GPU generation (G15; this board is the G15C "Max" variant with 40 cores in 4 clusters) and firmware 14.8.3. The Asahi driver selects its implementation by (generation, variant, firmware version), and no combination for G15 + 14.8.3 existed in Asahi's tree. Gravity Linux's driver targets G16 (M4) with firmware 26.x: not mergeable.

### The port that came first

Before adopting Aurora Silicon's driver, the author ported an experimental driver written for another M3 variant to the G15C, one reboot per attempt through a single-use GRUB entry, reading the GPU firmware's own log to see what it was waiting for. 33 rounds in a day:

- The GPU waited forever for its core clusters to power up: Linux shut the power-management coprocessor down 1.5 s after boot because nothing claimed it.
- With the coprocessor alive, the firmware crashed and took the display with it: 24 configuration parameters were byte-swapped (an 8 became 134 million and the firmware read outside its memory).
- First real compute result: `1337.0` in under 10 ms. Then job 18 failed (CPU and firmware disagreed on memory because of caching), then job 37 (memory reuse); after both fixes, 300 consecutive jobs.
- A single configuration value made the GPU power itself down twice and hang the second time: 121/121 after fixing it.

At that point Aurora Silicon already had a driver drawing a desktop on this same board, so the port was retired and Aurora's driver adopted as the base, with the board-specific pieces maintained on top and re-applied with a script on every Aurora update.

### Today

- Driver: Aurora's (Rust `asahi` driver, G15 backend) in the WayBian kernel. Mesa: Aurora's build, with a small Wayland/EGL patch, installed outside the system Mesa and selected through an EGL vendor file. Applications that run their GL in a sandbox (WebKitGTK's web process) need the Mesa path exposed inside the sandbox; Postcard's fork does that.
- `glxinfo`: "Apple M3 Max (G15C)", OpenGL 4.6. Hyprland uses about 2 % CPU; games in the browser run on the GPU at 1.3–1.4 GHz.
- Power/clock behaviour, queue depth and firmware logs are read from `debugfs`; a set of local scripts measures the GPU under load after every kernel change.
- Fallback: a Hyprland profile without GPU (`llvmpipe`, no blur, no shadows) is selected automatically by the session script when the GPU driver is not present, and the login greeter always renders by CPU.

## Video decode

Covered in [hardware.md](hardware.md#video-decode-avd): free AVD firmware built locally, `libva-v4l2-request` for VA‑API, verified frame-exact.
