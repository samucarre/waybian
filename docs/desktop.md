# Desktop

Hyprland 0.55 (Wayland) on Debian 13, with the tools and habits of a Mac user kept where they made sense.

## Session

- `greetd` on tty7 starting a **custom greeter**: a minimal Hyprland (its own user, software rendering) that shows the wallpaper dimmed and a translucent `kitty` with `tuigreet` in Spanish, remembering the user. If the graphical greeter does not appear within 40 s it falls back to the plain console greeter, so a broken GPU driver never locks you out. Tested with a fake greetd and a nested Hyprland before trusting it.
- The user session starts through a wrapper that picks the **GPU profile** when the Aurora driver is loaded and the **software profile** (`llvmpipe`, no blur/shadows) otherwise, and sets `AQ_DRM_DEVICES` with both DRM cards (internal panel + HDMI).
- Workspaces 1–6 on the laptop panel, 7–12 on the external monitor; `⌘⌥←/→` moves a window between screens. A "workstation" script lays out the usual windows in one click without duplicating the ones already open and without moving the user off the current workspace.

## Bar: Waybar on GTK4

Upstream Waybar is GTK3; the GTK4 port (upstream PR #5366) plus local fixes (a segfault in `hyprland/workspaces`, module menus, surviving monitor hot-plug, `align` for custom modules) is built from the `samucarre/Waybar` fork. The panels that drop down from the bar are small Python + GTK4 programs; modules that change often are repainted on a signal instead of waiting for their interval. Widgets written for this system:

- **Workstation**: places every window on its workspace in one click.
- **Recent files**: thumbnails of the latest downloads and screenshots, draggable straight into a web page, a mail or a chat. With focus-follows-mouse the panel could not tell when you clicked outside to close it; it reads the click from the input device directly.
- **Messaging** with counters for Discord, Telegram, WhatsApp and mail, fed by D‑Bus notifications and reset when the app is looked at. Hiding is not closing: the window moves to a special workspace and keeps notifying.
- **Remote desktops** by groups, in equal columns, never duplicating open ones.
- **Wi‑Fi** panel that waits for the network to reappear and retries (see [hardware.md](hardware.md#wi-fi)).
- **Shortcuts cheat sheet** ("?").
- **Fans, temperatures and package power.** The fan icon broke the module's JSON when Waybar ran without a locale (`\U000F0210` as text); the glyph is emitted as raw bytes.
- **Keyboard backlight**, **AirDrop**, **Tailscale**, **night light**, and a **mini music player** (Subsonic/Navidrome client: cover, seek bar, search, queue, recent albums, playlists, shuffle; audio in a separate `mpv` process so the panel can close while the music goes on; the bar talks to `mpv`'s socket directly because the MPRIS bridge reported "stopped" while playing).
- The bar is split in two pieces around the notch.

## Mac habits

- **Shortcuts**: a small script (`atajo-mac`) receives `⌘`+key from Hyprland, looks at the active window's class and sends the equivalent with `sendshortcut`: ⌘C/⌘V/⌘X become Ctrl+C/V/X in applications and Ctrl+Shift+C/V in terminals (where Ctrl+C interrupts), ⌘F finds (history search in `kitty`), ⌘W closes a tab in the browser, ⌘T opens one, ⌘⏎ sends (Ctrl+⏎) in web apps and the mail client but opens a terminal from a terminal or an empty desktop, ⌘+/⌘− zoom, Ctrl+⌘F toggles full screen, ⌘L locks. Unassigned ⌘ keys would otherwise reach Chrome as plain letters.
- **Screenshots**: ⌘⇧3 screen, ⌘⇧4 area, ⌘⇧5 window (`grim`/`slurp`), with a draggable thumbnail at the bottom right that opens `satty` to annotate.
- **Trackpad**: three-finger swipes switch workspaces in the macOS direction; the MX Master's thumb buttons do the same.
- **Night light**: a Hyprland screen shader tinted according to the local sunset (NOAA solar equations), toggled with ⌘⇧N or automatically. Screenshots are un-tinted afterwards, so captures stay true to colour.
- **AirDrop**: `omdrop` over the Wi‑Fi chip's AWDL (`omdrop-awdl`; this board is the first M3 in its list). Receiving shows a card with a thumbnail that can be dragged from the first second; sending needs the iPhone's AirDrop sheet open. Early problems (opening the panel powered the radio twice, the receiver started before the AWDL interface existed, every click restarted the stack) were fixed in the integration scripts and verified with 40 rounds of random clicks. The receiver declares no HEIC support, so the iPhone sends JPEG.
- **Quick Look**: space bar in the file manager (`sushi`); HEIC thumbnails needed `heif-thumbnailer`.
- Same wallpaper as macOS, to make the dual boot feel like one machine.

## GTK4 everywhere

- File dialogs for Hyprland through a GTK4 portal (`xdg-desktop-portal-gtk4`), Nautilus, `fuzzel` as launcher, `satty`, `mako`, `hypridle`/`hyprlock`, Chrome with `--gtk-version=4`. Electron apps, Thunderbird and LibreOffice cannot follow yet.
- **Mail: Postcard.** Debian 13 ships GTK 4.18 and libadwaita 1.7; Postcard needs libadwaita 1.8 (`Adw.ShortcutsDialog`), which needs GTK ≥ 4.19.4. Both were built into a private prefix (`/opt/gtk420`) used only by Postcard, with the X11 backend enabled because Debian's WebKitGTK references `gdk_x11_*` symbols. Postcard runs on an official Python 3.15 build (installed with `uv`) with PyGObject built in a venv, because its inbox watcher uses `IMAP4.idle()` (Python ≥ 3.14). Two generic findings from the fork:
  - **iCloud IMAP answers `UID SEARCH ALL` from a stale index** that still lists messages moved or deleted long ago (129k UIDs for a mailbox of 119k). Any client that prunes its cache with `SEARCH ALL` keeps ghosts that open as "no message body". `UID SEARCH UID 1:*` (or `UNDELETED`) is exact, and faster.
  - iCloud does not push `EXISTS` to every idling session; a watcher that depends on IDLE alone can miss a message for the length of its IDLE period. Short IDLE periods with a cheap `SEARCH` after each one are the safe pattern.
- D‑Bus-activated services (Nautilus and whatever it launches) only see the environment passed with `dbus-update-activation-environment`; `DISPLAY` had to be added for Xwayland applications opened from the file manager.
- The session runs without `LANG`, so some apps appear in English; Spanish is set per application where it matters.
