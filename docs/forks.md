# Forks and local builds

Everything that needed a change is kept as a fork with a branch, under github.com/samucarre, so the system can be rebuilt without depending on upstream merges. Reasons and dates are in the private lab repository's log.

| Program | Fork and branch | What changed | Installed as |
|---|---|---|---|
| Waybar | [samucarre/Waybar](https://github.com/samucarre/Waybar) `gtk4-debian13-m3` (on upstream PR Alexays/Waybar#5366) | GTK4 port plus: three fixes to build on Debian 13, a segfault in `hyprland/workspaces`, module menus, not dying when a monitor is removed or added, `align` for custom modules, clicks on a module's label or icon, tooltips that closed on every refresh, workspace clicks with Hyprland 0.55 and with a Lua configuration. `DEBIAN-M3.md` in the fork has the build recipe. | `/usr/local/bin/waybar` (remove it to return to Debian's) |
| aquamarine | [samucarre/aquamarine](https://github.com/samucarre/aquamarine) `hdmi-apple-drm-ext-0.15` (v0.15.1; `hdmi-apple-drm-ext` on v0.11.0 was the Debian 13 one) | GBM renderer for a DRM card without a render node, which is what the M3's HDMI (`apple-drm-ext`) is. Applied cleanly on 0.15.1 for Hyprland 0.56. | `/usr/local/lib/aarch64-linux-gnu/libaquamarine.so.14` |
| omdrop-awdl (AirDrop) | [samucarre/omdrop-awdl](https://github.com/samucarre/omdrop-awdl) `discoverable-rapido` | Faster discoverable on/off from the bar. | `/usr/lib/omdrop/` |
| libva-v4l2_request | [samucarre/libva-v4l2_request](https://github.com/samucarre/libva-v4l2_request) | Mirror, no changes: VA‑API driver for the AVD. | `/usr/lib/aarch64-linux-gnu/dri/v4l2_request_drv_video.so` |
| xdg-desktop-portal-gtk4 | [samucarre/xdg-desktop-portal-gtk4](https://github.com/samucarre/xdg-desktop-portal-gtk4/tree/waybian) `waybian` (on upstream `2372e15`) | Quick Look in the open dialogs, like the Finder: space opens the selected file in its own window, the arrows move through the files, space or Esc closes it and Enter then opens the file that was on screen. New module `src/gui/quick_look.rs` (Rust), with unit tests for its image-header reader. | `/usr/libexec/xdg-desktop-portal-gtk4` |
| Postcard | [samucarre/postcard](https://github.com/samucarre/postcard) `debian13-m3` | Google OAuth with a stored refresh token (no GOA); a minimal MCP server over its database; "Group Conversations" switch; IMAP IDLE for Python < 3.14; a reading pane that stays still when a click focuses a message (links open at the first click); prune with `UID SEARCH UID 1:*` instead of `ALL` (iCloud); WebKit sandbox exposure of a non-system GL driver; `DEBIAN-M3.md` with the GTK 4.20 + libadwaita 1.8 + Python 3.15 recipe for Debian 13 and the note that on Debian 14 the system GTK and Python are enough. | `/usr/local/libexec/postcard` + `/usr/local/bin/postcard` (meson, against the system) |
| Kernel | private fork of `omacom/linux-aurora` `integration/aurora` | Local patches (panel brightness on J516C, HDMI, small fixes), configs, build/install/update scripts. Four of the patches are proposed upstream from the public fork [samucarre/linux-aurora](https://github.com/samucarre/linux-aurora) ([below](#kernel-patches-proposed-upstream)). | `/boot/aurora/`, one GRUB entry per kernel, by id (`waybian-0.4` is the default) |
| Mesa (Aurora) | Aurora's Mesa | Small Wayland/EGL patch; installed outside the system Mesa, selected by an EGL vendor file. | `~/m3gpu/mesa-aurora` |
| AVD firmware | [aurora-silicon/avd-fw](https://github.com/aurora-silicon/avd-fw) | No changes; built with Debian's LLVM, SHA‑256 pinned. | `/lib/firmware/<kernel>/apple/avd-fw-v4-t0.bin` |

Installed from upstream unchanged: `satty` (release binary), `fuzzel` and everything else from Debian 14 (Hyprland 0.56.2 among them).

## Our commits, as patches
Each fork is mounted as a git submodule under `forks/` (pinned to the exact commit in use on the machine) and every code commit of ours is also here as a plain patch file, readable on GitHub. Apply with `git am` on top of the base named for each fork. The kernel fork stays private.

### Waybar: GTK4 port + fixes for Debian 13 / M3

Fork [samucarre/Waybar](https://github.com/samucarre/Waybar/tree/gtk4-debian13-m3), branch `gtk4-debian13-m3`, pinned at `a1343a4` as `forks/Waybar`. Base: upstream PR Alexays/Waybar#5366 (GTK4 port).

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
| 12 | 2026-10-27 | [ALabel: serve the tooltip through query-tooltip on the module widget](https://github.com/samucarre/Waybar/commit/f2007d2c748c1bfdfacbdc5488b188136a366976) | [0012-ALabel-serve-the-tooltip-through-query-tooltip-on-the-module.patch](../patches/Waybar/0012-ALabel-serve-the-tooltip-through-query-tooltip-on-the-module.patch) |
| 13 | 2026-10-27 | [hyprland/backend: try the legacy dispatch before the Lua protocol](https://github.com/samucarre/Waybar/commit/cef6e68d9b17050f3b99086c547a9187bdbe67bd) | [0013-hyprland-backend-try-the-legacy-dispatch-before-the-Lua-prot.patch](../patches/Waybar/0013-hyprland-backend-try-the-legacy-dispatch-before-the-Lua-prot.patch) |
| 14 | 2026-10-27 | [hyprland/backend: treat a Lua parse error reply as a rejected legacy dispatch](https://github.com/samucarre/Waybar/commit/a1343a481778550793d31daa338f99650095833c) | [0014-hyprland-backend-treat-a-Lua-parse-error-reply-as-a-rejected.patch](../patches/Waybar/0014-hyprland-backend-treat-a-Lua-parse-error-reply-as-a-rejected.patch) |


### aquamarine: GBM renderer for a DRM card without a render node (HDMI)

Fork [samucarre/aquamarine](https://github.com/samucarre/aquamarine/tree/hdmi-apple-drm-ext-0.15), branch `hdmi-apple-drm-ext-0.15`, pinned at `888d536` as `forks/aquamarine`. Base: hyprwm/aquamarine v0.15.1 (Debian 14); the branch `hdmi-apple-drm-ext` on v0.11.0 was the Debian 13 one. On v0.15.1 apply only patch 2; patch 1 is the same change on v0.11.0.

| # | Date | Commit | Patch |
|---|---|---|---|
| 1 | 2026-10-26 | [drm: fall back to a GBM renderer for GPUs without a render node (Apple dcpext/HDMI on the M3)](https://github.com/samucarre/aquamarine/commit/cd69728a118f128e75c8b6558326cb213547eddd) | [0001-drm-fall-back-to-a-GBM-renderer-for-GPUs-without-a-render-no.patch](../patches/aquamarine/0001-drm-fall-back-to-a-GBM-renderer-for-GPUs-without-a-render-no.patch) |
| 2 | 2026-10-27 | [drm: fall back to a GBM renderer for GPUs without a render node (Apple dcpext/HDMI on the M3) — rebased on v0.15.1 for Debian 14 (Hyprland 0.56)](https://github.com/samucarre/aquamarine/commit/888d53663522e8e8123d38953ef8ec9dbd764b22) | [0002-drm-fall-back-to-a-GBM-renderer-for-GPUs-without-a-render-no.patch](../patches/aquamarine/0002-drm-fall-back-to-a-GBM-renderer-for-GPUs-without-a-render-no.patch) |


### Postcard: OAuth without GOA, MCP, IMAP fixes, GTK 4.20 + Python 3.15 recipe

Fork [samucarre/postcard](https://github.com/samucarre/postcard/tree/debian13-m3), branch `debian13-m3`, pinned at `051ba85` as `forks/postcard`. Base: gxanshu/postcard 1.15.0.

| # | Date | Commit | Patch |
|---|---|---|---|
| 1 | 2026-10-26 | [DEBIAN-M3.md: native build on Debian 13 with GTK 4.20 + libadwaita 1.8 in a private prefix](https://github.com/samucarre/postcard/commit/d08599906ac6f1a8ecb1cb335b0073da0ce494f4) | [0001-DEBIAN-M3-md-native-build-on-Debian-13-with-GTK-4-20-libadwa.patch](../patches/postcard/0001-DEBIAN-M3-md-native-build-on-Debian-13-with-GTK-4-20-libadwa.patch) |
| 2 | 2026-10-26 | [oauth: sign a Google account in with a stored refresh token (no GOA)](https://github.com/samucarre/postcard/commit/c106d6c61307bc7c7a456e65953564d196589d55) | [0002-oauth-sign-a-Google-account-in-with-a-stored-refresh-token-n.patch](../patches/postcard/0002-oauth-sign-a-Google-account-in-with-a-stored-refresh-token-n.patch) |
| 3 | 2026-10-26 | [contrib: a minimal MCP server over Postcard's DB + IMAP](https://github.com/samucarre/postcard/commit/9ed4a4ce8aff0542163798777df091123fd9cd0c) | [0003-contrib-a-minimal-MCP-server-over-Postcard-s-DB-IMAP.patch](../patches/postcard/0003-contrib-a-minimal-MCP-server-over-Postcard-s-DB-IMAP.patch) |
| 4 | 2026-10-27 | [sandbox: expose the session's EGL vendor and driver paths to the web process](https://github.com/samucarre/postcard/commit/5141119578a2347d96ea5a67c113840249450009) | [0004-sandbox-expose-the-session-s-EGL-vendor-and-driver-paths-to-.patch](../patches/postcard/0004-sandbox-expose-the-session-s-EGL-vendor-and-driver-paths-to-.patch) |
| 5 | 2026-10-27 | [prefs: a "Group Conversations" switch so every message can be its own row](https://github.com/samucarre/postcard/commit/4eba53e7f27ca2a7e9283ee3b13e4c93705d0d34) | [0005-prefs-a-Group-Conversations-switch-so-every-message-can-be-i.patch](../patches/postcard/0005-prefs-a-Group-Conversations-switch-so-every-message-can-be-i.patch) |
| 6 | 2026-10-27 | [imap: IDLE on Python < 3.14, and prune with UID 1:* instead of SEARCH ALL (iCloud)](https://github.com/samucarre/postcard/commit/962b38c8b88f83c9281e6914207c0997b9e2389d) | [0006-imap-IDLE-on-Python-3-14-and-prune-with-UID-1-instead-of-SEA.patch](../patches/postcard/0006-imap-IDLE-on-Python-3-14-and-prune-with-UID-1-instead-of-SEA.patch) |
| 7 | 2026-10-27 | [DEBIAN-M3.md: run Postcard on an official Python 3.15 build (uv) with PyGObject built in a venv](https://github.com/samucarre/postcard/commit/f3349f0a9ba334ade373a0674a902759de028afd) | [0007-DEBIAN-M3-md-run-Postcard-on-an-official-Python-3-15-build-u.patch](../patches/postcard/0007-DEBIAN-M3-md-run-Postcard-on-an-official-Python-3-15-build-u.patch) |
| 8 | 2026-10-27 | [DEBIAN-M3.md: on Debian 14 the system GTK and Python are enough](https://github.com/samucarre/postcard/commit/3dc993ebcc51111f099cf56952c72401c6b66f9f) | [0008-DEBIAN-M3-md-on-Debian-14-the-system-GTK-and-Python-are-enou.patch](../patches/postcard/0008-DEBIAN-M3-md-on-Debian-14-the-system-GTK-and-Python-are-enou.patch) |
| 9 | 2026-10-29 | [fix: keep the reading pane still when a click focuses a message body](https://github.com/samucarre/postcard/commit/051ba85910b78f8fbb0ffe886e04ec93b7023096) | [0009-fix-keep-the-reading-pane-still-when-a-click-focuses-a-messa.patch](../patches/postcard/0009-fix-keep-the-reading-pane-still-when-a-click-focuses-a-messa.patch) |


### omdrop-awdl (AirDrop): faster discoverable toggle

Fork [samucarre/omdrop-awdl](https://github.com/samucarre/omdrop-awdl/tree/discoverable-rapido), branch `discoverable-rapido`, pinned at `ce623f5` as `forks/omdrop-awdl`. Base: netmojo-systems/omdrop-awdl main.

| # | Date | Commit | Patch |
|---|---|---|---|
| 1 | 2026-10-26 | [discoverable: faster visible/invisible toggle for the Waybar AirDrop button (M3 Max, Debian 13)](https://github.com/samucarre/omdrop-awdl/commit/ce623f52eed51bdfa267058bf0a694fd72cd962e) | [0001-discoverable-faster-visible-invisible-toggle-for-the-Waybar-.patch](../patches/omdrop-awdl/0001-discoverable-faster-visible-invisible-toggle-for-the-Waybar-.patch) |

### libva-v4l2_request: VA-API driver for the AVD (mirror)

Fork [samucarre/libva-v4l2_request](https://github.com/samucarre/libva-v4l2_request/tree/avd-fixes), branch `avd-fixes`, pinned at `4f60c51` as `forks/libva-v4l2_request`. Base: omacom/libva-v4l2_request (mirror, no changes).

No commits of ours: a mirror.

### xdg-desktop-portal-gtk4: Quick Look in the GTK4 file dialogs

Fork [samucarre/xdg-desktop-portal-gtk4](https://github.com/samucarre/xdg-desktop-portal-gtk4/tree/waybian), branch `waybian` (its default branch), pinned at `b369398` as `forks/xdg-desktop-portal-gtk4`. Base: mahkoh/xdg-desktop-portal-gtk4 master (`2372e15`).

| # | Date | Commit | Patch |
|---|---|---|---|
| 1 | 2026-10-29 | [FileChooser: preview panel for the selected file](https://github.com/samucarre/xdg-desktop-portal-gtk4/commit/55b6519ab1c8a9cba6ea755f625bcb7482227f1e) | [0001-FileChooser-preview-panel-for-the-selected-file.patch](../patches/xdg-desktop-portal-gtk4/0001-FileChooser-preview-panel-for-the-selected-file.patch) |
| 2 | 2026-10-29 | [FileChooser: Quick Look with the space bar](https://github.com/samucarre/xdg-desktop-portal-gtk4/commit/83d326d3b77a19df5e2f7922c22725742449e2ef) | [0002-FileChooser-Quick-Look-with-the-space-bar.patch](../patches/xdg-desktop-portal-gtk4/0002-FileChooser-Quick-Look-with-the-space-bar.patch) |
| 3 | 2026-10-29 | [FileChooser: keep only Quick Look, drop the side panel](https://github.com/samucarre/xdg-desktop-portal-gtk4/commit/6c61876fe1858b129b12598c6066253382832d2c) | [0003-FileChooser-keep-only-Quick-Look-drop-the-side-panel.patch](../patches/xdg-desktop-portal-gtk4/0003-FileChooser-keep-only-Quick-Look-drop-the-side-panel.patch) |
| 4 | 2026-10-29 | [FileChooser: Quick Look window of a fixed size](https://github.com/samucarre/xdg-desktop-portal-gtk4/commit/5c894800139e6c427e7b0ac03267e9d83d5edf3d) | [0004-FileChooser-Quick-Look-window-of-a-fixed-size.patch](../patches/xdg-desktop-portal-gtk4/0004-FileChooser-Quick-Look-window-of-a-fixed-size.patch) |
| 5 | 2026-10-29 | [FileChooser: Quick Look fixes after a review](https://github.com/samucarre/xdg-desktop-portal-gtk4/commit/68d7a420a29e912884f14a86f502e2efec87cb43) | [0005-FileChooser-Quick-Look-fixes-after-a-review.patch](../patches/xdg-desktop-portal-gtk4/0005-FileChooser-Quick-Look-fixes-after-a-review.patch) |

Commit 3 removes the side panel that commit 1 added; commit 5 fixes what a review found (a crash with an empty file list, a leak when the dialog closed with the preview open, the list's cursor left behind by the arrows, the selection lost with several files, huge images). The last three commits of the branch ([763190a](https://github.com/samucarre/xdg-desktop-portal-gtk4/commit/763190a60d33ecac13c61e9753268af75a7d809e), [65bca80](https://github.com/samucarre/xdg-desktop-portal-gtk4/commit/65bca805e948fba95f81ab781b0945dd58ab329a) and [b369398](https://github.com/samucarre/xdg-desktop-portal-gtk4/commit/b36939883ed9632c04bb619be0174cb1c1711eb8)) only touch the README (a section with screenshots); they are not kept as patches because they carry images.

### Kernel: patches proposed upstream

The kernel that runs is built from a private fork (local patches, configurations, install scripts). Four of its patches were sent to Aurora as pull requests from the public fork [samucarre/linux-aurora](https://github.com/samucarre/linux-aurora), one branch each; state as of 29 Oct 2026:

| Pull request | Branch | Title | State |
|---|---|---|---|
| [#258](https://github.com/omacom/linux-aurora/pull/258) | `t6031-gpu-pmp-report` | soc: apple: report the T6031 GPU to the PMP | open |
| [#259](https://github.com/omacom/linux-aurora/pull/259) | `j516c-panel-brightness` | drm/apple: v14.7: admit panel brightness on the J516C | open |
| [#260](https://github.com/omacom/linux-aurora/pull/260) | `avd-av1-invalid-ref-idx` | media: apple: avd: av1: do not index reference_frame_ts with ref_frame_idx -1 | open |
| [#261](https://github.com/omacom/linux-aurora/pull/261) | `g15-max-inflight` | drm/asahi: g15: allow 8 prefixes in flight per VM (was 2), tunable with g15_max_inflight | open |
