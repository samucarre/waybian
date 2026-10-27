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
| Hyprland 0.55.2 | Wayland compositor | OpenGL (EGL) | Debian 13 backports | GPU profile when the Aurora driver is up, `llvmpipe` profile otherwise |
| aquamarine (patched) | Hyprland's DRM/GBM backend | — | **built from our fork** | GBM renderer for the HDMI card without a render node |
| Waybar (GTK4 port) | bar + widgets | GTK4 / gtkmm 4 (Vulkan) | **built from our fork** | upstream is GTK3; Debian's 0.12 package stays installed but unused |
| fuzzel 1.12 | launcher | Wayland native (no toolkit) | Debian 13 | replaced wofi (GTK3) |
| mako 1.10 | notifications | Wayland native | Debian 13 backports | |
| kitty 0.41 | terminal | OpenGL | Debian 13 | |
| greetd + tuigreet, custom greeter | login | CPU on purpose | backports + our scripts | must work without the GPU |
| hyprlock, hypridle, hyprpicker, hyprpolkitagent | lock, idle, picker, polkit | OpenGL / Wayland | backports | |
| swaybg | wallpaper | Wayland native | Debian 13 | |
| grim, slurp, satty 0.22 | screenshots and annotation | satty: GTK4 + libadwaita | Debian 13; satty: upstream binary | replaced swappy (GTK3) |
| xdg-desktop-portal-hyprland + xdg-desktop-portal-gtk4 | portals (screen sharing, file dialogs) | GTK4 (Rust) | backports; **portal-gtk4 built from mirror** | the GTK3 portal stays only for what the GTK4 one does not cover |
| Nautilus 48, Loupe 48, sushi 46 | files, images, Quick Look | GTK4 + libadwaita | Debian 13 | HEIC thumbnails via `heif-thumbnailer` |
| pavucontrol 6 | audio mixer | GTK4 / gtkmm 4 | Debian 13 | |
| PipeWire + WirePlumber, asahi-audio 3.3, speakersafetyd | sound | — | Debian 13 + Asahi packages | J516 profiles added by hand (see [hardware.md](hardware.md)) |
| NetworkManager + iwd | network | — | Debian 13 | `nm-applet` (GTK3) out of the session; the bar has its own Wi‑Fi panel |
| Our bar panels and tools | recent files, messaging, remote desktops, Wi‑Fi, keyboard light, AirDrop, music, night light, shortcuts | Python + GTK4 (PyGObject) | our scripts | ~30 scripts, 16 KiB-page safe |

## Applications

| Program | Draws with | Origin | Notes |
|---|---|---|---|
| Google Chrome | Chromium; GTK4 for dialogs (`--gtk-version=4`) | vendor repo | hardware video decode through VA‑API (`libva-v4l2-request`) |
| Postcard (mail) | GTK 4.20 + libadwaita 1.8 + WebKitGTK 6.0 | **built from our fork**, into `/opt/gtk420`, on Python 3.15 | replaced Thunderbird (GTK3, no port) |
| WordCraft, GridCraft, DeckCraft, PdfCraft, PhotoCraft | Rust (iced/winit) on Vulkan | upstream releases (checksum pinned) | office suite drawn by the GPU; each exposes an MCP server |
| LibreOffice | GTK3, headless | Debian 13 | not for the user: hidden from the menu, it is the tool the AI assistant (Claude Code) drives to build and convert Office documents (docx/xlsx/pptx skills), because the Craft apps' MCP servers are not mature enough for that yet. Its GTK4 backend crashes. Also the default for legacy .xls/.ppt/.odp |
| mpv 0.40 | OpenGL/Vulkan, `hwdec` | Debian 13 | hardware decode |
| Telegram Desktop 5.7 | Qt 5 + Vulkan | backports | |
| WhatsApp | Chrome app window (GTK4 dialogs) | — | replaced a WebKitGTK wrapper that stuttered |
| Vesktop (Discord) | Electron 44 (GTK3) | upstream release | Electron ignores `--gtk-version=4` |
| Claude Desktop | Electron (GTK3) | vendor repo | official Linux beta; minimum width patched 600 → 360 px |
| Remmina 1.4 | GTK3 | Debian 13 | RDP/VNC; no GTK4 alternative yet |
| Steam, Tailscale, Claude Code | — | upstream | |

## Built by us (not from apt)

Kept in forks with a branch each (see [forks.md](forks.md)); rebuilt with one script per program in the lab repository (`compilados.sh compilar <name>`):

| What | From | Where it lands |
|---|---|---|
| Waybar GTK4 | fork `gtk4-debian13-m3` | `/usr/local/bin/waybar` |
| libaquamarine | fork `hdmi-apple-drm-ext` | `/usr/local/lib/aarch64-linux-gnu/` |
| libva-v4l2_request | mirror | `/usr/lib/aarch64-linux-gnu/dri/` |
| GTK 4.20.4 + libadwaita 1.8.8 + Postcard | upstream tarballs + fork `debian13-m3` | `/opt/gtk420` (private prefix, Postcard only) |
| Python 3.15.2 + PyGObject 3.58 | `uv python install`, pip | `~/.local/share/uv`, venv `/opt/gtk420/py315` |
| xdg-desktop-portal-gtk4 | mirror (Rust) | `/usr/libexec/` |
| Mesa (Aurora) | Aurora's tree + Wayland patch | outside the system Mesa, chosen by an EGL vendor file |
| AVD firmware | `aurora-silicon/avd-fw` | `/lib/firmware/<kernel>/apple/` |
| Kernel | private fork of Aurora's tree | `/boot/aurora/`, GRUB |
| Night-light shader, bar panels, session scripts | ours | `~/.config/hypr`, `~/.local/bin` |

Everything else comes from Debian 13 (some packages from `trixie-backports`: Hyprland and its tools, greetd/tuigreet, kitty, mako, Telegram, the portals) or from the vendors' own repositories.

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
