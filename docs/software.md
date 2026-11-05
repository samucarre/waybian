# Software: what is installed, what we build, and why GTK4

## The intention

Everything on the desktop should be drawn by the GPU, not the CPU. On this machine the GPU is the part that is new and powerful (40 cores), and the CPU is what drains the battery when it has to paint windows. So the rule for choosing software is, in this order:

1. **Wayland-native and GTK4** (or any toolkit that renders through OpenGL/Vulkan): GTK4 renders scenes on the GPU (GL or Vulkan renderer); GTK3 draws with Cairo on the CPU and uploads pixels.
2. If there is no GTK4 option, a toolkit that still uses the GPU (Qt with Vulkan, Rust GUIs on wgpu).
3. Electron/Chromium-based apps only when there is nothing else: they carry their own Chromium and ignore the system toolkit (and the `--gtk-version=4` flag).

GTK3 cannot be removed from the system (47 packages depend on it, Chrome among them), but almost nothing that is on screen all day uses it any more. The measured effects so far are in the last section; the rest is the direction, still to be measured piece by piece.

## Desktop stack

| Program | Role | Draws with | Origin | Notes |
|---|---|---|---|---|
| Hyprland 0.56.2 | Wayland compositor | OpenGL (EGL) | Debian 14 | configured in Lua; GPU profile when the Aurora driver is up, `llvmpipe` profile otherwise |
| aquamarine (patched) | Hyprland's DRM/GBM backend | — | **built from our fork** | GBM renderer for the HDMI card without a render node |
| Waybar (GTK4 port) | bar + widgets | GTK4 / gtkmm 4 (Vulkan) | **built from our fork** | upstream is GTK3; Debian's 0.15 package stays installed but unused |
| fuzzel 1.15 | clipboard history (⌘⇧V) | Wayland native (no toolkit) | Debian 14 | replaced wofi (GTK3); since 2 Nov the bar's own launcher starts the applications. Started with `LC_CTYPE=C.UTF-8`: the session has no locale, and fuzzel dropped any name with a non-ASCII character (an entry with a middle dot showed as `<no title>`) |
| SwayNotificationCenter (swaync) 0.12.6 | notification centre | GTK4 + libadwaita (GPU) | **built from our fork** | replaced mako: popups like mako's (same corner and colours), plus a panel with the history and Do Not Disturb that hangs from the bar, opened from a bell ([desktop.md](desktop.md#bar-waybar-on-gtk4)); the fork adds the bar's drawer animation. Debian's package stays for its D‑Bus and systemd files: swaync is D‑Bus-activated through `swaync.service`, so killing it only brings Debian's `/usr/bin/swaync` back, and a user drop-in points `ExecStart` at the fork. It started on GTK's CPU renderer (`GSK_RENDERER=cairo`) as a precaution; on the GPU since 3 Nov |
| kitty 0.48 | terminal | OpenGL | Debian 14 | |
| greetd + tuigreet, custom greeter | login | CPU on purpose | Debian 14 + our scripts | must work without the GPU |
| hyprlock, hypridle, hyprpicker, hyprpolkitagent | lock, idle, picker, polkit | OpenGL / Wayland | Debian 14 | |
| mpvpaper, swaybg | animated wallpaper; the still photo as fallback | mpvpaper: libmpv on OpenGL (EGL), video decoded by the AVD; swaybg: Wayland native | mpvpaper: upstream, built here (not packaged in Debian); swaybg: Debian 14 | the loop pauses while a window is active and with the screen off or locked; swaybg takes over if mpvpaper is missing or dies ([desktop.md](desktop.md#mac-habits)) |
| grim, slurp, satty 0.22 | screenshots and annotation | satty: GTK4 + libadwaita | Debian 14; satty: upstream binary | replaced swappy (GTK3) |
| xdg-desktop-portal-hyprland + xdg-desktop-portal-gtk4 | portals (screen sharing, file dialogs) | GTK4 (Rust) | Debian 14; **portal-gtk4 built from our fork** | Quick Look (space bar) in the open dialogs, PDFs and videos included through the system thumbnailers; the GTK3 portal stays only for what the GTK4 one does not cover |
| Nautilus 51, Loupe 51, sushi 50 | files, images, Quick Look | GTK4 + libadwaita | Debian 14 | HEIC thumbnails via `heif-thumbnailer`; a watcher of ours makes the thumbnails ahead of time in Desktop, Pictures, Downloads and the bookmarks, and squares all of them ([desktop.md](desktop.md#file-dialogs-thumbnails-and-quick-look)) |
| pavucontrol 6 | audio mixer | GTK4 / gtkmm 4 | Debian 14 | |
| PipeWire + WirePlumber, asahi-audio 5.0, speakersafetyd 3.0 | sound | — | Debian 14 + Asahi packages | Debian 14's J516 profiles, plus a local headphone cap (see [hardware.md](hardware.md#sound)) |
| NetworkManager + iwd | network | — | Debian 14 | `nm-applet` (GTK3) out of the session; the bar has its own Wi‑Fi panel |
| Our bar panels and tools | launcher, recent files, messaging, remote desktops, Wi‑Fi, volume, brightness and night light, keyboard light, utilities (timer), system, power off, Tailscale, AirDrop, music, shortcuts, notch notices | Python + GTK4 (PyGObject) on gtk4-layer-shell | our scripts | one shared module (`perfilado.py`) for the shape, the theme of each screen and the drawer animation; 16 KiB-page safe |

## Applications

| Program | Draws with | Origin | Notes |
|---|---|---|---|
| Google Chrome | Chromium; GTK4 for dialogs (`--gtk-version=4`) | vendor repo | hardware video decode through VA‑API (`libva-v4l2-request`) |
| Postcard (mail) | GTK 4.24 + libadwaita 1.10 + WebKitGTK 6.0 | **built from our fork** against the system GTK, on Python 3.14 | replaced Thunderbird (GTK3, no port); kitty's palette through a user stylesheet, the whole mailbox downloaded in the background ([desktop.md](desktop.md#gtk4-everywhere)) |
| GNOME Clocks 51, Dialect 2.6, GNOME Calculator 51 | GTK4 + libadwaita | Debian 14 | opened from the bar's Utilities panel as floating windows that hang from the bar, dark in each bar's colours ([desktop.md](desktop.md#bar-waybar-on-gtk4)); installed without recommends (gnome-clocks pulls ModemManager in through geoclue) |
| WordCraft, GridCraft, DeckCraft, PdfCraft, PhotoCraft | Rust (iced/winit) on Vulkan | upstream releases (checksum pinned) | office suite drawn by the GPU; each exposes an MCP server |
| LibreOffice | GTK3, headless | Debian 14 | not for the user: hidden from the menu, it is the tool the AI assistant (Claude Code) drives to build and convert Office documents (docx/xlsx/pptx skills), because the Craft apps' MCP servers are not mature enough for that yet. Its GTK4 backend crashes. Also the default for legacy .xls/.ppt/.odp |
| mpv 0.41 | OpenGL/Vulkan, `hwdec` | Debian 14 | hardware decode |
| Telegram Desktop 5.7 | Qt 5 + Vulkan | Debian 14 | |
| WhatsApp, iCloud Photos | Chrome app windows (GTK4 dialogs) | — | WhatsApp replaced a WebKitGTK wrapper that stuttered; iCloud Photos stands in for the Photos app, because the macOS library cannot be read from Linux (hardware-encrypted volume) |
| Vesktop (Discord) | Electron 44 (GTK3) | upstream release | Electron ignores `--gtk-version=4` |
| Claude Desktop | Electron (GTK3) | vendor repo | official Linux beta; minimum width patched 600 → 360 px |
| Remmina 1.4 | GTK3 | Debian 14 | RDP/VNC; no GTK4 alternative yet |
| Steam, Tailscale, Claude Code | — | upstream | |

## Built by us (not from apt)

Kept in forks with a branch each (see [forks.md](forks.md)); rebuilt with one script per program in the lab repository (`compilados.sh compilar <name>`):

| What | From | Where it lands |
|---|---|---|
| Waybar GTK4 | fork `gtk4-debian13-m3` | `/usr/local/bin/waybar` |
| libaquamarine | fork `hdmi-apple-drm-ext-0.15` | `/usr/local/lib/aarch64-linux-gnu/` |
| libva-v4l2_request | mirror | `/usr/lib/aarch64-linux-gnu/dri/` |
| Postcard | fork `debian13-m3` (meson, against the system GTK 4.24 + libadwaita 1.10) | `/usr/local/libexec/postcard`, launcher `/usr/local/bin/postcard` |
| xdg-desktop-portal-gtk4 | fork `waybian` (Rust): Quick Look, with PDFs and videos through the system thumbnailers | `/usr/libexec/` |
| SwayNotificationCenter | fork `debian13-m3` (on v0.12.6, Vala): the drawer animation | `/usr/local/bin/swaync`, started through a drop-in for its user unit |
| mpvpaper | upstream `GhostNaN/mpvpaper`, unchanged (not packaged in Debian) | `~/.local/bin/mpvpaper` |
| Mesa (Aurora) | Aurora's tree + Wayland patch | outside the system Mesa, chosen by an EGL vendor file |
| AVD firmware | `aurora-silicon/avd-fw` | `/lib/firmware/<kernel>/apple/` |
| Kernel | private fork of Aurora's tree | `/boot/aurora/`, GRUB |
| `hyprctl` compatibility layer (legacy commands → Lua) | ours | `/usr/local/bin/hyprctl`, `/usr/local/lib/hyprctl-lua/` |
| Night-light shader, bar panels, session scripts, thumbnail watcher, the wallpaper's animation (generator and day/evening/night switch) | ours | `~/.config/hypr`, `~/.local/bin`, `~/.local/lib/waybian`, `~/.config/systemd/user` |

Everything else comes from Debian 14 or from the vendors' own repositories. On Debian 13, Hyprland and its tools, greetd/tuigreet, kitty, mako, Telegram and the portals came from `trixie-backports`.

## Still not GTK4, and why

- **Electron apps** (Claude Desktop, Vesktop): Electron 44 has no GTK4 support (electron#33690); the flag is ignored.
- **LibreOffice**: its GTK4 backend crashes; it runs with GTK3 and mostly headless.
- **Remmina**: GTK3, no port; used for RDP/VNC.
- **Thunderbird**: GTK3 and no port in sight; retired in favour of Postcard.
- **Telegram Desktop**: Qt 5, but renders with Vulkan, so it is fine on the GPU.

## What has been measured

- **Compositor on the GPU**: with Aurora's driver Hyprland sits at about 2 % CPU; with `llvmpipe` the whole desktop was painted by the CPU.
- **Hardware video decode** (AVD through VA‑API) on a browser tab streaming a remote KVM, 30 s averages after settling: software 15.3 W of package power with Chrome at 72 % of a core → hardware 10.2 W with Chrome at 6 %. With the full working session open: 19.4 W → 16.2 W. Roughly 5 W saved on the video alone, 3 W with everything open.
- **GTK4 vs GTK3 on the bar and panels**: not measured yet (the port came first). The expectation is CPU time moved to the GPU; a before/after with the same session is pending.
- **Notification centre, CPU or GPU renderer**: frames drawn per opening of its drawer, with GTK's CPU renderer (`GSK_RENDERER=cairo`) 79 on the laptop panel (longest gap 50 ms) and 44 on the external monitor (18 ms); on the GPU, 82 (33 ms) and 43 (216 ms). Cairo was not the bottleneck: the long gaps are a stall of swaync's main loop when the centre maps, which the drawer now absorbs. Since 3 Nov swaync draws on the GPU like the rest of GTK4.
- **Animations in the bar**: a smooth CSS animation repaints at the panel's 120 Hz. A pulsing halo on the brightness button took Waybar from 2 to 6 % CPU and added about 200 GPU jobs per second; a blink with fixed values between steps cost 2.3 → 3.0 % and 6 more GPU jobs per second. The halo is fixed now (blinking, it looked as if the night light had frozen).
- **Animated wallpaper**: on battery, 27.8 W with the loop playing and 23.6 W with it frozen, about 4 W even behind windows; hence it only plays on an empty workspace. Decoding on the AVD costs about 4 % CPU.

## Debian 14

Upgraded in place on 27 Oct 2026 (`apt full-upgrade` from trixie to forky: 2,241 packages, 5 minutes, no broken package), after rehearsing the same upgrade in a `systemd-nspawn` container made from a copy of the root. What needed attention, in case it helps someone:

- **Service restarts during the upgrade**: a `policy-rc.d` returning 101 kept dpkg from restarting services; otherwise the `greetd` upgrade would have killed the graphical session with apt running inside it. The upgrade itself ran as a transient `systemd-run` unit.
- **GRUB**: grub 2.14's postinst regenerated `/boot/grub/grub.cfg` and dropped the hand-written entries for the kernels in `/boot/aurora/` (the only automatic entry was the Debian emergency kernel, without GPU). Restored from the backup taken just before, then moved to `/etc/grub.d/40_custom` with an id per entry and `GRUB_DEFAULT` by id, so the next grub upgrade rebuilds our menu ([boot-chain.md](boot-chain.md#grub-layout)). The GRUB image in the ESP was later rebuilt with 2.14 and `loadenv`.
- **asahi-scripts** now ships its initramfs scripts under `/usr/share/initramfs-tools/` and renamed the patched copies in `/etc/initramfs-tools/scripts/` to `.dpkg-old`. The new upstream scripts still check `/lib/firmware/vendor` and do not wait for the ESP's by-partuuid link, so the patched ones were put back (they override `/usr/share`) and the initrds regenerated.
- **aquamarine**: Hyprland 0.56 links `libaquamarine.so.14` (0.15.1); the HDMI patch applied cleanly on v0.15.1 and the rebuilt library in `/usr/local` is picked up first. **Waybar** needed a rebuild for the new libfmt.
- **Audio**: `speakersafetyd` 3.0 ships `j516.conf` natively; `alsa-ucm-conf-asahi` 10 ships a "MacBook Pro J516.conf" that points to the generic laptop profile without the headphone cap, so the local profile is kept with `dpkg-divert`; `asahi-audio` 5.0 ships J516 graphs of its own (speakers and microphone) and they are the ones in use, so the local WirePlumber rule from Debian 13 (J416 graph + a mic filter for the old node name) was removed.
- **Python 3.14** became the default: every venv created with the system 3.13 stopped working and was recreated (one MCP server's locked `pydantic-core` does not build on 3.14 yet, so it runs on a `uv`-managed 3.13). Venvs made with `uv` were unaffected.
- **Hyprland 0.56** warns at every start that 0.57 removes the `.conf` format; the configuration moved to Lua ([desktop.md](desktop.md#session)).
- **Thumbnails**: gdk-pixbuf 2.44 decodes through glycin and no longer ships `gdk-pixbuf-thumbnailer`; the AirDrop card and the Office thumbnailer lost their pictures without a single error. A small replacement with the same command line (Python + GdkPixbuf, EXIF orientation applied) brought them back.
- **Vulkan**: Aurora's Mesa, built on Debian 13, links `libdisplay-info.so.2` and forky ships `.so.3`, so Vulkan found no driver. The old library, checked against Debian's index, sits next to that Mesa until it is rebuilt on Debian 14.
- Gains: GTK 4.24 / libadwaita 1.10 and Python 3.14 system-wide (Postcard now uses them: its private GTK 4.20 prefix and the Python 3.15 build were removed), PipeWire 1.6, Mesa 26, FreeRDP 3.32, newer toolchains.
