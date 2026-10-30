# Hardware, component by component

State on 30 Oct 2026 with kernel `7.2.11-waybian-0.10`. "Fix" means something that had to be done on top of Debian 14 and the upstream kernel to make it work on this board (`j516c`).

## Display

- Internal panel 3456×2234, scale 2 in Hyprland. Driven by Aurora's display path (`apple-drm`), native resolution from the first frame.
- **Brightness** (`/sys/class/backlight/apple-panel-bl`): the upstream patch only admitted other 16" boards; a local kernel patch admits the J516C. Keyboard keys adjust it; a bar widget too.
- Night light: there is no colour-transform (CTM/gamma) support in the display driver yet, so `hyprsunset`-style tools do nothing. WayBian tints the screen with a Hyprland shader instead (see [desktop.md](desktop.md)).
- The notch is handled by the bar layout.

## External display (HDMI)

- The HDMI port is a second DRM card (`apple-drm-ext`) **without a render node**. Hyprland's backend (`aquamarine`) refused it; the fork `samucarre/aquamarine` (`hdmi-apple-drm-ext-0.15`, on aquamarine 0.15.1) adds a GBM renderer path for a card like that, and Hyprland is started with `AQ_DRM_DEVICES` listing both cards.
- Works at 3440×1440 @ 60 Hz on an ultrawide; 100 Hz is still pending. Audio follows the cursor's monitor.
- DisplayPort over USB‑C: investigated, not working (needs the type‑C/USB4 side, device tree and boot-loader work).

## GPU

See [kernel-gpu.md](kernel-gpu.md). Summary: Apple M3 Max G15C, 40 cores in 4 clusters, Aurora Silicon driver and Mesa, OpenGL 4.6 and Vulkan 1.4 (Mesa's Honeykrisp).

## Video decode (AVD)

- The AVD block decodes H.264, HEVC (8/10 bit), VP9 and AV1. Its firmware is free software (`aurora-silicon/avd-fw`, MIT), built with Debian's LLVM and installed under `/lib/firmware/<kernel version>/apple/` because other installed kernels carry different AVD drivers.
- User space: `libva-v4l2-request` (VA‑API over V4L2 request API). Chrome uses it with `LIBVA_DRIVER_NAME=v4l2_request`; GStreamer picks `v4l2codecs` by itself; mpv with `hwdec`. Verified frame-exact against software decoding for all five codecs.
- Effect: a browser tab streaming a remote KVM went from ~15 W to ~10 W of package power and Chrome's CPU from ~70 % to ~6 %.

## Wi‑Fi

- Broadcom BCM4388 (`brcmfmac4388c0-pcie`). Firmware and NVRAM come from macOS, extracted by the installer into the ESP (`vendorfw/`) and unpacked at boot into `/lib/firmware/vendor`.
- Fixes: the initramfs problems in [boot-chain.md](boot-chain.md) (`cp` missing in klibc, `firmware_class.path`, `modprobe` path). A rescue service re-extracts the firmware and reloads `brcmfmac`/`hci_bcm4377` if the initramfs failed.
- NetworkManager with the `iwd` backend. After dropping a network, the manager is blind for a few seconds and the next connection attempt fails; the bar's Wi‑Fi panel waits for the network to reappear and retries.
- Harmless log noise: `brcmf ... join_pref -52`, `escan_timeout`.

## Bluetooth

- Works (same chip, `hci_bcm4377`). Pairing a Logitech MX Master needed `pairable on` and answering the agent; `logiops` maps its thumb buttons to previous/next workspace.
- `libspa-0.2-bluetooth` has to be installed for Bluetooth audio (it is not pulled in by default).

## Sound

- Speakers: the J516 (16" M3) has the same amplifier/codec set as the J416 (16" M2): `cs42l84` + six `sn012776`. On Debian 13, `asahi-audio` 3.3 and `alsa-ucm-conf-asahi` did not know the J516, so the J416 UCM profiles and WirePlumber filter chain (`graph-j416.json`) were set up for it. Debian 14's `asahi-audio` 5.0 ships J516 graphs of its own, and they are the ones in use: the speakers' convolver on `RawSpeakers` and three-microphone beamforming on `RawMics` (the local Debian 13 rule, which pointed to node names that no longer exist, was removed on 29 Oct). The local UCM profile is kept with `dpkg-divert` because `alsa-ucm-conf-asahi` 10's J516 profile points to the generic laptop one, without the headphone cap. Result: the "MacBook Pro Speakers" sink with Asahi's convolver DSP.
- `speakersafetyd` 3.0 (Debian 14) ships a `j516` profile; on Debian 13 (1.1.2) a symlink `j516.conf → j316.conf` mirrored the upstream commit that added the M3 generation. Without `speakersafetyd` the kernel keeps the amplifiers limited to −20 dB, so the speakers cannot be damaged, they only sound low.
- Headphone jack: the `cs42l84` "Jack DAC Playback Volume" reaches +12 dB and nothing limits it on Linux. A local UCM profile fixes the DAC at −12 dB and leaves volume to PipeWire in software.
- Microphones: beamforming profile from the J416; the ALSA node name differs on this kernel (`platform-audio` instead of `platform-snd_soc_apple_aop`), so the filter file was adjusted.
- A gotcha: the login greeter's own PipeWire grabbed the sound card before the user session started, leaving the session with a "Dummy Output". PipeWire is masked for the greeter user.

## Input

- Keyboard and trackpad go through `dockchannel-hid` with the `tpmtfw-j516c.bin` firmware from `vendorfw`. Three-finger gestures in Hyprland switch workspaces in the macOS direction. Spanish layout (`es`) in the console, the greeter and the session.
- Keyboard backlight: `white:kbd_backlight` via `pwm_apple` + `leds_pwm`. The M3 keyboard has no keys for it (Fn+F5/F6 are microphone and sleep), so a bar widget shows a keyboard glyph whose halo follows the level: click for a slider, right click toggles, wheel ±5 %.
- Lid: still `HandleLidSwitch=lock` and `AllowSuspend=no`. Suspend works since kernel 0.9 (see [Suspend](#suspend)), but it is not wired to the lid until its battery drain is measured.

## Camera

- `apple-isp`, `/dev/video0`, works in browsers. The driver floods the console with debug output: `loglevel=3` and `kernel.printk = 3 3 3 3`.

## USB‑C, Thunderbolt

- The three ports work as USB 3 (10 Gb/s) with `deferred_probe_timeout=10`; USB4/Thunderbolt has no driver. An HDMI output through a USB‑C adapter stays in billboard mode (no DisplayPort alt mode yet).

## Storage

- Apple NVMe (`nvme-apple`, ANS2) in Asahi's tree only (mainline 7.3 has the device tree but not the controller node).
- The macOS data volume cannot be read from Linux: APFS with hardware encryption (keys in the Secure Enclave). The ESP has 4 KiB sectors and macOS does not mount it either; copying `boot.bin` from macOS is done with `dd` on the raw device.
- Linux root was enlarged from 48 to 122.5 GiB after the install by shrinking macOS from macOS and moving the partition, without reinstalling. On 28 Oct 2026 it grew to 401.9 GiB (macOS keeps 521.5 GiB): the system was copied into a new partition in the space freed from macOS (first used as `/home`), which took over the root's filesystem UUID, so GRUB and the root line of `fstab` did not change (the `/home` line went away); then the stub container and the ESP were copied to the end of the old root, keeping their PARTUUIDs (m1n1 finds the ESP by PARTUUID), the partition table was rewritten so the new root runs up to them, and `resize2fs` grew it on the next boot.

## Power

- Battery and charging report correctly; package power is read from the SMC for the bar's power widget. The time left is the SMC's own estimate (`time_to_empty_now` / `time_to_full_now`); the SMC reports battery current and power with a sign (negative when discharging), so a calculation from the current has to use its absolute value.
- Fans: with the chip at 99–107 °C during a long GPU compute job, the SMC keeps them at about 65 % (3,500 and 3,700 rpm). `macsmc_hwmon` with `fan_control=1` lets Linux set them: a small service sets 80 % from 80 °C and the maximum from 88 °C, gives control back to the SMC below 72 °C, and ignores sensors that stop changing (the GPU's keep their last value while the GPU is off).
- Short press of the power key is "power off" by default in logind; the Touch ID key sits next to Backspace, so a long press only is advisable (`HandlePowerKey=ignore`).

## Suspend

Suspend-to-idle (`s2idle`, the only sleep state here) works since kernel 0.9, started by hand:

- Ported from an open Aurora pull request for the M3 Pro (`omacom/linux-aurora#274`): the PMU's RTC alarm, so `rtcwake` can wake the machine; wake-ups from the PMU's interrupts; the external display requests released during sleep (on this board optional and off by default).
- What this board needed on top: the RTC alarm is described by the kernel from Apple's device tree at boot, so no new `boot.bin` was needed; the HDMI port's display coprocessor and its power domains stay on through sleep (its firmware starts only once per boot); and the GPU is kept powered through sleep, because nothing starts its firmware again (before that, sleep cut the GPU's power under its running firmware and the GPU never came back).
- Tested on 29 Oct 2026: three 30‑second sleeps without the GPU driver and three with it loaded, all woken by the RTC alarm. After each one the GPU kept running compute jobs at the same rate, and the panel, brightness, Bluetooth, the speaker safety daemon and the USB‑C ports came back; Wi‑Fi drops in sleep and reconnects by itself in about 15 s. In a 5‑second rehearsal (`pm_test`), with the external display lit the kernel refuses to sleep cleanly, and with it switched off (DPMS) it goes through.
- Not done yet: the lid (logind still only locks), the battery drain while asleep, and a real sleep with the HDMI display attached.

## Not working

Suspend on lid close (see [Suspend](#suspend)), USB4/Thunderbolt, DisplayPort over USB‑C, reading the macOS volume, HDMI above 60 Hz.
