# Forks and local builds

Everything that needed a change is kept as a fork with a branch, under github.com/samucarre, so the system can be rebuilt without depending on upstream merges. Reasons and dates are in the private lab repository's log.

| Program | Fork and branch | What changed | Installed as |
|---|---|---|---|
| Waybar | [samucarre/Waybar](https://github.com/samucarre/Waybar) `gtk4-debian13-m3` (on upstream PR Alexays/Waybar#5366) | GTK4 port plus: three fixes to build on Debian 13, a segfault in `hyprland/workspaces`, module menus, not dying when a monitor is removed or added, `align` for custom modules. `DEBIAN-M3.md` in the fork has the build recipe. | `/usr/local/bin/waybar` (remove it to return to Debian's) |
| aquamarine | [samucarre/aquamarine](https://github.com/samucarre/aquamarine) `hdmi-apple-drm-ext` (v0.11.0) | GBM renderer for a DRM card without a render node, which is what the M3's HDMI (`apple-drm-ext`) is. | `/usr/local/lib/aarch64-linux-gnu/libaquamarine.so.10` |
| omdrop-awdl (AirDrop) | [samucarre/omdrop-awdl](https://github.com/samucarre/omdrop-awdl) `discoverable-rapido` | Faster discoverable on/off from the bar. | `/usr/lib/omdrop/` |
| libva-v4l2_request | [samucarre/libva-v4l2_request](https://github.com/samucarre/libva-v4l2_request) | Mirror, no changes: VA‑API driver for the AVD. | `/usr/lib/aarch64-linux-gnu/dri/v4l2_request_drv_video.so` |
| xdg-desktop-portal-gtk4 | [samucarre/xdg-desktop-portal-gtk4](https://github.com/samucarre/xdg-desktop-portal-gtk4) | Mirror, no changes (Rust): GTK4 file dialogs through the portal. | `/usr/libexec/xdg-desktop-portal-gtk4` |
| Postcard | [samucarre/postcard](https://github.com/samucarre/postcard) `debian13-m3` | Google OAuth with a stored refresh token (no GOA); a minimal MCP server over its database; "Group Conversations" switch; IMAP IDLE for Python < 3.14; prune with `UID SEARCH UID 1:*` instead of `ALL` (iCloud); WebKit sandbox exposure of a non-system GL driver; `DEBIAN-M3.md` with the GTK 4.20 + libadwaita 1.8 + Python 3.15 recipe. | `/opt/gtk420` + `/usr/local/bin/postcard` |
| Kernel | private fork of `omacom/linux-aurora` `integration/aurora` | Local patches (panel brightness on J516C, HDMI, small fixes), configs, build/install/update scripts. | `/boot/aurora/`, GRUB entries 2–4 |
| GTK 4.20.4 + libadwaita 1.8.8 | upstream tarballs | No changes; built into `/opt/gtk420` for Postcard only. | `/opt/gtk420` |
| Mesa (Aurora) | Aurora's Mesa | Small Wayland/EGL patch; installed outside the system Mesa, selected by an EGL vendor file. | `~/m3gpu/mesa-aurora` |
| AVD firmware | [aurora-silicon/avd-fw](https://github.com/aurora-silicon/avd-fw) | No changes; built with Debian's LLVM, SHA‑256 pinned. | `/lib/firmware/<kernel>/apple/avd-fw-v4-t0.bin` |

Installed from upstream unchanged: `satty` (release binary), `fuzzel` and everything else from Debian 13 (some from `trixie-backports`, such as Hyprland 0.55).
