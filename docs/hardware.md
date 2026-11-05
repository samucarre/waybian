# Hardware, component by component

State on 5 Nov 2026 with kernel `7.2.11-waybian-0.14`. "Fix" means something that had to be done on top of Debian 14 and the upstream kernel to make it work on this board (`j516c`).

## Display

- Internal panel 3456×2234, scale 2 in Hyprland. Driven by Aurora's display path (`apple-drm`), native resolution from the first frame, 120 Hz.
- **Brightness** (`/sys/class/backlight/apple-panel-bl`): the upstream code only admitted other 16" boards; this project's patch for the J516C was merged into Aurora on 30 Oct 2026 (#259). Keyboard keys adjust it; a bar widget too.
- **Up to 1000 nits.** Linux stopped at 500 nits because of the kernel, not the panel (Apple rates it at 600 nits SDR and 1000 nits sustained full-screen): a driver constant and `apple,max-brightness = <500>` in the panel's device tree. A local patch (kernel 0.12) adds `appledrm.panel_nits_max`, which replaces the device tree's value on a mini-LED panel, up to 1000; without it, nothing changes. Measured from the SMC, the power outside the chip goes from about 14–17 W at 500 nits to 16.2 W at 600, 16.9 at 800, 20.4 at 900 and 21.5 at 1000 (a noisy reading right after boot), so the firmware does follow requests above 600. The bar's brightness panel keeps a normal range up to 500 nits and an extra one up to 1000, and the bar icon shows the "overload" state above 500 ([desktop.md](desktop.md#bar-waybar-on-gtk4)).
- Night light: there is no colour-transform (CTM/gamma) support in the display driver yet, so `hyprsunset`-style tools do nothing. WayBian tints the screen with a Hyprland shader instead (see [desktop.md](desktop.md)).
- The notch is handled by the bar layout: on the laptop panel the bar is black, so the notch blends into it.
- Since kernel 0.14 (Aurora #325) the panel path no longer keeps a framebuffer reference for good on every swap ([kernel-gpu.md](kernel-gpu.md#kernel-lineage)).

## External display (HDMI)

- The HDMI port is a second DRM card (`apple-drm-ext`) **without a render node**. Hyprland's backend (`aquamarine`) refused it; the fork `samucarre/aquamarine` (`hdmi-apple-drm-ext-0.15`, on aquamarine 0.15.1) adds a GBM renderer path for a card like that, and Hyprland is started with `AQ_DRM_DEVICES` listing both cards.
- Works at 3440×1440 @ 60 Hz on an ultrawide; 100 Hz is still pending. Audio follows the cursor's monitor. Since kernel 0.13 Aurora turns DP/HDMI audio on by default (#300), but this board's device tree does not describe that audio path yet, so there is no HDMI audio here; the external mode is set again whenever the display powers on (#295).
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
- **Regulatory database.** Every boot logged `cfg80211: loaded regulatory.db is malformed or signature is missing/invalid`. The WayBian kernel (Aurora's configuration) requires a signed database and trusts only the upstream keys (`CFG80211_REQUIRE_SIGNED_REGDB`, `CFG80211_USE_KERNEL_REGDB_KEYS`), while Debian's `regulatory.db` alternative pointed to the Debian-signed copy, the one Debian's own kernel accepts. cfg80211 was left in the minimal world domain ("country 00": 2.4 GHz and 5170–5250 MHz only, with passive scanning). Fixed without a new kernel or a reboot: `update-alternatives --set regulatory.db /lib/firmware/regulatory.db-upstream`, `iw reg reload`, and the local regulatory domain set at boot with `options cfg80211 ieee80211_regdom=<country>` in `/etc/modprobe.d/`. The Broadcom firmware kept its own world domain until kernel 0.13 and the matching `boot.bin` passed the country to it (Aurora #311; see [boot-chain.md](boot-chain.md#pinned-packages)).
- **iwd ignores the AirDrop interface** (`iwd -I awdl0`, in a drop-in for `iwd.service`). Restarted while `awdl0` existed, iwd adopted it as well and gave it the `dormant` link mode it uses on `wlan0`; in that mode the kernel keeps the interface DORMANT even with carrier and never assigns its IPv6 link-local address, so AirDrop had nowhere to listen and failed with "awdl0 has no link-local". At boot it cannot happen (`awdl0` is created after iwd starts, and iwd ignores interfaces that appear later), which is why it looked like a regression of the new kernel; it was not. The AirDrop helper also puts `awdl0` back to the default link mode if it ever finds it dormant.
- Harmless log noise: `brcmf ... join_pref -52`, `escan_timeout`, and one `iface validation failed: err=-16` per iwd restart (brcmfmac counts `awdl0` as a second station).

## Bluetooth

- Works (same chip, `hci_bcm4377`). Pairing a Logitech MX Master needed `pairable on` and answering the agent; `logiops` maps its thumb buttons to previous/next workspace. At boot `logid` can time out on the mouse (`Device timed out`) and never retries, so its udev rule restarts it once more 5 s later (a transient `systemd-run --on-active=5` timer).
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
- Lid: logind still only locks (`HandleLidSwitch=lock`, `AllowSuspend=no`); suspend works since kernel 0.9 (see [Suspend](#suspend)) but is not wired to the lid until its battery drain is measured. What happens to the panel on closing and opening: [Lid and black screens](#lid-and-black-screens).

## Lid and black screens

On 3 Nov 2026 the screen stayed black on opening the lid after about 50 minutes closed, with the system alive, and the machine had to be forced off with the power button. Two things were found:

- **Nothing turned the panel back on.** With the session locked and the panel off by DPMS, opening the lid is not activity for hypridle (it resumes on keyboard or mouse input only), Hyprland does not wake DPMS by itself, and there was no lid bind. Hyprland's `misc:key_press_enables_dpms` and `mouse_move_enables_dpms` are no way out: with them on, `dpms off` never applied at all, even with zero input events (checked with evdev). Upstream has reports with this version and the same lock + DPMS combination (Hyprland #16259 and #16315, hyprlock #953).
- **That time the panel had never been switched off.** The display coprocessor logs a `brightness_changed` line on every DPMS off and on, and there was none: a browser window inhibiting idle kept hypridle from reaching its 20-minute timeout, and the panel stayed lit with black content. That is the signature of two earlier black screens (one in use, one on the login screen), whose cause is still unknown.

The fix covers both. Hyprland binds on the lid switch (`switch:on` / `switch:off` of the "Apple SMC power/lid events" device) turn the internal panel off by DPMS 5 s after closing (time for hyprlock to start) and every output on 0.5 s after opening, through timers that cancel each other (closing and opening within 5 s used to leave the panel off with the lid open), and they work while locked and while a window grabs the shortcuts (`locked`, `dont_inhibit`). The DPMS cycle on opening also makes the display coprocessor present frames again, which is the rescue for the lit-but-black state; `⌘⇧R` runs the same cycle blind, and `misc:allow_session_lock_restore` lets a crashed hyprlock be started again. To leave evidence next time, Hyprland's full log, with timestamps, is kept on disk for every session ([desktop.md](desktop.md#session)), and hypridle and hyprlock log to the journal. Tested with a virtual lid switch (a uinput device with the same name): closed for 9 s, the panel goes off after ~5 s and comes back in under 1 s on opening; closed for 1 s, it stays on. Since then it has worked the same with the real lid (4 and 5 Nov, closed overnight and for an hour or so): the panel goes off 5–6 s after closing and comes back on opening.

## Camera

- `apple-isp`, `/dev/video0`, works in browsers. The driver floods the console with debug output: `loglevel=3` and `kernel.printk = 3 3 3 3`.

## USB‑C, Thunderbolt

- The three ports work as USB 3 (10 Gb/s) with `deferred_probe_timeout=10`; USB4/Thunderbolt has no driver. An HDMI output through a USB‑C adapter stays in billboard mode (no DisplayPort alt mode yet).

## Storage

- Apple NVMe (`nvme-apple`, ANS2) in Asahi's tree only (mainline 7.3 has the device tree but not the controller node).
- The macOS data volume cannot be read from Linux: APFS with hardware encryption (keys in the Secure Enclave). The ESP has 4 KiB sectors and macOS does not mount it either; copying `boot.bin` from macOS is done with `dd` on the raw device.
- Linux root was enlarged from 48 to 122.5 GiB after the install by shrinking macOS from macOS and moving the partition, without reinstalling. On 28 Oct 2026 it grew to 401.9 GiB (macOS kept 521.5 GiB): the system was copied into a new partition in the space freed from macOS (first used as `/home`), which took over the root's filesystem UUID, so GRUB and the root line of `fstab` did not change (the `/home` line went away); then the stub container and the ESP were copied to the end of the old root, keeping their PARTUUIDs (m1n1 finds the ESP by PARTUUID), the partition table was rewritten so the new root runs up to them, and `resize2fs` grew it on the next boot.
- On 4 Nov 2026 it grew again, to 560.2 GiB (macOS keeps 363.2 GiB; 207 GB free on the root afterwards), in three steps. From macOS, with a script, the space was taken from the APFS container and left as a temporary FAT partition right in front of the Linux root. From Linux, a second script removed that partition from the GPT and moved the start of the root's entry back to where it began (same PARTUUID, type and name), without letting the running kernel re-read the table. Debian cannot boot between this step and the next, because the filesystem still sits 158 GiB into the partition and GRUB does not find its UUID (copies of the previous table were kept, one of them on the ESP). From macOS again, the script moved the ext4 filesystem down to the partition's new start, and on the next boot a one-shot service checked the partition's start and size, ran `resize2fs` online and disabled itself. Partition numbers moved during the operation (the temporary partition took `p3` and pushed the root to `p4` until the second script removed it) and ended as they were before (root `p3`, ESP `p5`); nothing depends on them: `fstab`, GRUB and the configuration embedded in `BOOTAA64.EFI` find the root by filesystem UUID, and U‑Boot finds the ESP by its type. Layout afterwards: Apple's 500 MiB iBoot System Container, macOS 363.2 GiB, Linux root 560.2 GiB, stub container 2.3 GiB, ESP 477 MiB, and Apple's 128 MiB boot and 5 GiB recovery partitions.

## Power

- Battery and charging report correctly; package power is read from the SMC for the bar's power widget. The time left is the SMC's own estimate (`time_to_empty_now` / `time_to_full_now`); the SMC reports battery current and power with a sign (negative when discharging), so a calculation from the current has to use its absolute value.
- Fans: with the chip at 99–107 °C during a long GPU compute job, the SMC keeps them at about 65 % (3,500 and 3,700 rpm). `macsmc_hwmon` with `fan_control=1` lets Linux set them: a small service sets 80 % from 80 °C and the maximum from 88 °C, gives control back to the SMC below 72 °C, and ignores sensors that stop changing (the GPU's keep their last value while the GPU is off).
- Power key (the Touch ID key, next to Backspace): a short press locks the session (`HandlePowerKey=lock`), like closing the lid, and does nothing on the login screen. logind's default is to power off, and a short press on a black login screen did exactly that. Holding the key for about 5 s is the SMC's forced shutdown, which does not sync the disks; logind's long-press action cannot take its place, because the SMC reports the long press at 4.95 s and logind waits 5.0 s (`HandlePowerKeyLongPress=ignore`).

## Suspend

Suspend-to-idle (`s2idle`, the only sleep state here) works since kernel 0.9, started by hand:

- Ported from an open Aurora pull request for the M3 Pro (`omacom/linux-aurora#274`): the PMU's RTC alarm, so `rtcwake` can wake the machine; wake-ups from the PMU's interrupts; the external display requests released during sleep (on this board optional and off by default).
- What this board needed on top: the RTC alarm was described by the kernel from Apple's device tree at boot, so no new `boot.bin` was needed at first (since `boot.bin` F, 2 Nov, it comes from the Linux device tree, #285); the HDMI port's display coprocessor and its power domains stay on through sleep (its firmware starts only once per boot); and the GPU is kept powered through sleep, because nothing starts its firmware again (before that, sleep cut the GPU's power under its running firmware and the GPU never came back).
- Tested on 29 Oct 2026: three 30‑second sleeps without the GPU driver and three with it loaded, all woken by the RTC alarm. After each one the GPU kept running compute jobs at the same rate, and the panel, brightness, Bluetooth, the speaker safety daemon and the USB‑C ports came back; Wi‑Fi drops in sleep and reconnects by itself in about 15 s. In a 5‑second rehearsal (`pm_test`), with the external display lit the kernel refuses to sleep cleanly, and with it switched off (DPMS) it goes through. The same test with the GPU passed again on kernels 0.10, 0.11 and 0.13 (three out of three each; on 0.11 two of the sleeps were ended by touching the keyboard, one by the alarm).
- Not done yet: the lid (logind still only locks), the battery drain while asleep, and a real sleep with the HDMI display attached.

## Not working

Suspend on lid close (see [Suspend](#suspend)), USB4/Thunderbolt, DisplayPort over USB‑C, reading the macOS volume, HDMI above 60 Hz.
