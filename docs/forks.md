# Forks and local builds

Everything that needed a change is kept as a fork with a branch, under github.com/samucarre, so the system can be rebuilt without depending on upstream merges. Reasons and dates are in the private lab repository's log.

| Program | Fork and branch | What changed | Installed as |
|---|---|---|---|
| Waybar | [samucarre/Waybar](https://github.com/samucarre/Waybar) `gtk4-debian13-m3` (on upstream PR Alexays/Waybar#5366) | GTK4 port plus: three fixes to build on Debian 13, a segfault in `hyprland/workspaces`, module menus, not dying when a monitor is removed or added, `align` for custom modules. `DEBIAN-M3.md` in the fork has the build recipe. | `/usr/local/bin/waybar` (remove it to return to Debian's) |
| aquamarine | [samucarre/aquamarine](https://github.com/samucarre/aquamarine) `hdmi-apple-drm-ext-0.15` (v0.15.1; `hdmi-apple-drm-ext` on v0.11.0 was the Debian 13 one) | GBM renderer for a DRM card without a render node, which is what the M3's HDMI (`apple-drm-ext`) is. Applied cleanly on 0.15.1 for Hyprland 0.56. | `/usr/local/lib/aarch64-linux-gnu/libaquamarine.so.14` |
| omdrop-awdl (AirDrop) | [samucarre/omdrop-awdl](https://github.com/samucarre/omdrop-awdl) `discoverable-rapido` | Faster discoverable on/off from the bar. | `/usr/lib/omdrop/` |
| libva-v4l2_request | [samucarre/libva-v4l2_request](https://github.com/samucarre/libva-v4l2_request) | Mirror, no changes: VA‑API driver for the AVD. | `/usr/lib/aarch64-linux-gnu/dri/v4l2_request_drv_video.so` |
| xdg-desktop-portal-gtk4 | [samucarre/xdg-desktop-portal-gtk4](https://github.com/samucarre/xdg-desktop-portal-gtk4) | Mirror, no changes (Rust): GTK4 file dialogs through the portal. | `/usr/libexec/xdg-desktop-portal-gtk4` |
| Postcard | [samucarre/postcard](https://github.com/samucarre/postcard) `debian13-m3` | Google OAuth with a stored refresh token (no GOA); a minimal MCP server over its database; "Group Conversations" switch; IMAP IDLE for Python < 3.14; prune with `UID SEARCH UID 1:*` instead of `ALL` (iCloud); WebKit sandbox exposure of a non-system GL driver; `DEBIAN-M3.md` with the GTK 4.20 + libadwaita 1.8 + Python 3.15 recipe. | `/opt/gtk420` + `/usr/local/bin/postcard` |
| Kernel | private fork of `omacom/linux-aurora` `integration/aurora` | Local patches (panel brightness on J516C, HDMI, small fixes), configs, build/install/update scripts. | `/boot/aurora/`, GRUB entries 2–4 |
| GTK 4.20.4 + libadwaita 1.8.8 | upstream tarballs | No changes; built into `/opt/gtk420` for Postcard only. | `/opt/gtk420` |
| Mesa (Aurora) | Aurora's Mesa | Small Wayland/EGL patch; installed outside the system Mesa, selected by an EGL vendor file. | `~/m3gpu/mesa-aurora` |
| AVD firmware | [aurora-silicon/avd-fw](https://github.com/aurora-silicon/avd-fw) | No changes; built with Debian's LLVM, SHA‑256 pinned. | `/lib/firmware/<kernel>/apple/avd-fw-v4-t0.bin` |

Installed from upstream unchanged: `satty` (release binary), `fuzzel` and everything else from Debian 13 (some from `trixie-backports`, such as Hyprland 0.55).

## Our commits, as patches
Each fork is mounted as a git submodule under `forks/` (pinned to the exact commit in use on the machine) and every commit of ours is also here as a plain patch file, readable on GitHub. Apply with `git am` on top of the base named for each fork. The kernel fork stays private.

### Waybar: GTK4 port + fixes for Debian 13 / M3

Fork [samucarre/Waybar](https://github.com/samucarre/Waybar/tree/gtk4-debian13-m3), branch `gtk4-debian13-m3`, pinned at `fff9bf3` as `forks/Waybar`. Base: upstream PR Alexays/Waybar#5366 (GTK4 port).

| # | Date | Commit | Patch |
|---|---|---|---|
| 1 | 2026-10-26 | [meson: build against Debian 13 (gtkmm 4.18, gtk4-layer-shell 1.0.4, giomm-2.68)](https://github.com/samucarre/Waybar/commit/61179f7f0920727356a146fee1c2480022373f71) | [0001-meson-build-against-Debian-13-gtkmm-4-18-gtk4-layer-shell-1-.patch](../patches/Waybar/0001-meson-build-against-Debian-13-gtkmm-4-18-gtk4-layer-shell-1-.patch) |
| 2 | 2026-10-26 | [glibmm-2.68: catch Glib::Error instead of the removed Glib::Exception](https://github.com/samucarre/Waybar/commit/ba0809ea179ce560127f46cc23788518d85782f3) | [0002-glibmm-2-68-catch-Glib-Error-instead-of-the-removed-Glib-Exc.patch](../patches/Waybar/0002-glibmm-2-68-catch-Glib-Error-instead-of-the-removed-Glib-Exc.patch) |
| 3 | 2026-10-26 | [hyprland/workspaces: guard controller_scroll_ like sway/workspaces does](https://github.com/samucarre/Waybar/commit/bc5754a97bda13a42215fa52c1f05332af987294) | [0003-hyprland-workspaces-guard-controller-scroll-like-sway-worksp.patch](../patches/Waybar/0003-hyprland-workspaces-guard-controller-scroll-like-sway-worksp.patch) |
| 4 | 2026-10-26 | [DEBIAN-M3.md: build notes for Debian 13 on the M3 Max](https://github.com/samucarre/Waybar/commit/7eff727eaf14056e5e48b0704b37fb2c929b523f) | [0004-DEBIAN-M3-md-build-notes-for-Debian-13-on-the-M3-Max.patch](../patches/Waybar/0004-DEBIAN-M3-md-build-notes-for-Debian-13-on-the-M3-Max.patch) |
| 5 | 2026-10-26 | [DEBIAN-M3.md: note the GTK4 CSS pitfall with universal font rules](https://github.com/samucarre/Waybar/commit/bb05fe74906dfb154a4492c04f17312d8330b9f1) | [0005-DEBIAN-M3-md-note-the-GTK4-CSS-pitfall-with-universal-font-r.patch](../patches/Waybar/0005-DEBIAN-M3-md-note-the-GTK4-CSS-pitfall-with-universal-font-r.patch) |
| 6 | 2026-10-26 | [AModule: bring back menu/menu-file/menu-actions on GTK4 with a PopoverMenu](https://github.com/samucarre/Waybar/commit/1b40f4721efa0310d337db69f1d1759951618c7c) | [0006-AModule-bring-back-menu-menu-file-menu-actions-on-GTK4-with-.patch](../patches/Waybar/0006-AModule-bring-back-menu-menu-file-menu-actions-on-GTK4-with-.patch) |
| 7 | 2026-10-27 | [client: survive output removal (GTK4 items-changed)](https://github.com/samucarre/Waybar/commit/736f1c1883917fb271537d9bc0176113db64ec3a) | [0007-client-survive-output-removal-GTK4-items-changed.patch](../patches/Waybar/0007-client-survive-output-removal-GTK4-items-changed.patch) |
| 8 | 2026-10-27 | [AIconLabel/ALabel: honour "align" inside the module box](https://github.com/samucarre/Waybar/commit/7f9329eacf897dd9e45f5e6056395981098ede5e) | [0008-AIconLabel-ALabel-honour-align-inside-the-module-box.patch](../patches/Waybar/0008-AIconLabel-ALabel-honour-align-inside-the-module-box.patch) |
| 9 | 2026-10-27 | [DEBIAN-M3.md: notes for the output-removal and align fixes](https://github.com/samucarre/Waybar/commit/9fdd7e87fcac082c302d97a6279083bdc2926c0b) | [0009-DEBIAN-M3-md-notes-for-the-output-removal-and-align-fixes.patch](../patches/Waybar/0009-DEBIAN-M3-md-notes-for-the-output-removal-and-align-fixes.patch) |
| 10 | 2026-10-27 | [AIconLabel: let clicks on the label/image reach the module's gestures](https://github.com/samucarre/Waybar/commit/175999c35ad1f2d0b8cf943462af91c4140fe594) | [0010-AIconLabel-let-clicks-on-the-label-image-reach-the-module-s-.patch](../patches/Waybar/0010-AIconLabel-let-clicks-on-the-label-image-reach-the-module-s-.patch) |
| 11 | 2026-10-27 | [AIconLabel: bind the event controllers to the box, not the label](https://github.com/samucarre/Waybar/commit/fff9bf3ecc16de7db5fa4d20e283bd68f2bdb3d6) | [0011-AIconLabel-bind-the-event-controllers-to-the-box-not-the-lab.patch](../patches/Waybar/0011-AIconLabel-bind-the-event-controllers-to-the-box-not-the-lab.patch) |

### aquamarine: GBM renderer for a DRM card without a render node (HDMI)

Fork [samucarre/aquamarine](https://github.com/samucarre/aquamarine/tree/hdmi-apple-drm-ext), branch `hdmi-apple-drm-ext`, pinned at `cd69728` as `forks/aquamarine`. Base: hyprwm/aquamarine v0.11.0.

| # | Date | Commit | Patch |
|---|---|---|---|
| 1 | 2026-10-26 | [drm: fall back to a GBM renderer for GPUs without a render node (Apple dcpext/HDMI on the M3)](https://github.com/samucarre/aquamarine/commit/cd69728a118f128e75c8b6558326cb213547eddd) | [0001-drm-fall-back-to-a-GBM-renderer-for-GPUs-without-a-render-no.patch](../patches/aquamarine/0001-drm-fall-back-to-a-GBM-renderer-for-GPUs-without-a-render-no.patch) |

### Postcard: OAuth without GOA, MCP, IMAP fixes, GTK 4.20 + Python 3.15 recipe

Fork [samucarre/postcard](https://github.com/samucarre/postcard/tree/debian13-m3), branch `debian13-m3`, pinned at `ff9a126` as `forks/postcard`. Base: gxanshu/postcard 1.15.0.

| # | Date | Commit | Patch |
|---|---|---|---|
| 1 | 2026-10-26 | [DEBIAN-M3.md: native build on Debian 13 with GTK 4.20 + libadwaita 1.8 in a private prefix](https://github.com/samucarre/postcard/commit/d08599906ac6f1a8ecb1cb335b0073da0ce494f4) | [0001-DEBIAN-M3-md-native-build-on-Debian-13-with-GTK-4-20-libadwa.patch](../patches/postcard/0001-DEBIAN-M3-md-native-build-on-Debian-13-with-GTK-4-20-libadwa.patch) |
| 2 | 2026-10-26 | [oauth: sign a Google account in with a stored refresh token (no GOA)](https://github.com/samucarre/postcard/commit/c106d6c61307bc7c7a456e65953564d196589d55) | [0002-oauth-sign-a-Google-account-in-with-a-stored-refresh-token-n.patch](../patches/postcard/0002-oauth-sign-a-Google-account-in-with-a-stored-refresh-token-n.patch) |
| 3 | 2026-10-26 | [contrib: a minimal MCP server over Postcard's DB + IMAP](https://github.com/samucarre/postcard/commit/9ed4a4ce8aff0542163798777df091123fd9cd0c) | [0003-contrib-a-minimal-MCP-server-over-Postcard-s-DB-IMAP.patch](../patches/postcard/0003-contrib-a-minimal-MCP-server-over-Postcard-s-DB-IMAP.patch) |
| 4 | 2026-10-27 | [sandbox: expose the session's EGL vendor and driver paths to the web process](https://github.com/samucarre/postcard/commit/5141119578a2347d96ea5a67c113840249450009) | [0004-sandbox-expose-the-session-s-EGL-vendor-and-driver-paths-to-.patch](../patches/postcard/0004-sandbox-expose-the-session-s-EGL-vendor-and-driver-paths-to-.patch) |
| 5 | 2026-10-27 | [prefs: a "Group Conversations" switch so every message can be its own row](https://github.com/samucarre/postcard/commit/01aa53a4af9e1fc7b17d98d17b6f01ec41b77ab9) | [0005-prefs-a-Group-Conversations-switch-so-every-message-can-be-i.patch](../patches/postcard/0005-prefs-a-Group-Conversations-switch-so-every-message-can-be-i.patch) |
| 6 | 2026-10-27 | [imap: IDLE on Python < 3.14, and prune with UID 1:* instead of SEARCH ALL (iCloud)](https://github.com/samucarre/postcard/commit/77af0e591ad48efd74d606d97b1257aa86adb45b) | [0006-imap-IDLE-on-Python-3-14-and-prune-with-UID-1-instead-of-SEA.patch](../patches/postcard/0006-imap-IDLE-on-Python-3-14-and-prune-with-UID-1-instead-of-SEA.patch) |
| 7 | 2026-10-27 | [DEBIAN-M3.md: run Postcard on an official Python 3.15 build (uv) with PyGObject built in a venv](https://github.com/samucarre/postcard/commit/ff9a1262d7e246242115936e68a3c3ee341cbbb0) | [0007-DEBIAN-M3-md-run-Postcard-on-an-official-Python-3-15-build-u.patch](../patches/postcard/0007-DEBIAN-M3-md-run-Postcard-on-an-official-Python-3-15-build-u.patch) |

### omdrop-awdl (AirDrop): faster discoverable toggle

Fork [samucarre/omdrop-awdl](https://github.com/samucarre/omdrop-awdl/tree/discoverable-rapido), branch `discoverable-rapido`, pinned at `ce623f5` as `forks/omdrop-awdl`. Base: netmojo-systems/omdrop-awdl main.

| # | Date | Commit | Patch |
|---|---|---|---|
| 1 | 2026-10-26 | [discoverable: faster visible/invisible toggle for the Waybar AirDrop button (M3 Max, Debian 13)](https://github.com/samucarre/omdrop-awdl/commit/ce623f52eed51bdfa267058bf0a694fd72cd962e) | [0001-discoverable-faster-visible-invisible-toggle-for-the-Waybar-.patch](../patches/omdrop-awdl/0001-discoverable-faster-visible-invisible-toggle-for-the-Waybar-.patch) |

### libva-v4l2_request: VA-API driver for the AVD (mirror)

Fork [samucarre/libva-v4l2_request](https://github.com/samucarre/libva-v4l2_request/tree/avd-fixes), branch `avd-fixes`, pinned at `4f60c51` as `forks/libva-v4l2_request`. Base: omacom/libva-v4l2_request (mirror, no changes).

No commits of ours: a mirror.

### xdg-desktop-portal-gtk4: GTK4 file dialogs (mirror)

Fork [samucarre/xdg-desktop-portal-gtk4](https://github.com/samucarre/xdg-desktop-portal-gtk4/tree/master), branch `master`, pinned at `2372e15` as `forks/xdg-desktop-portal-gtk4`. Base: mahkoh/xdg-desktop-portal-gtk4 (mirror, no changes).

No commits of ours: a mirror.
