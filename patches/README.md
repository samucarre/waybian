# Patches

Every code commit of ours on each fork, as plain patch files (what `git format-patch` produces), grouped by program and in order. The same commits are in the forks mounted under `forks/`.

| Program | Patches | Base |
|---|---|---|
| [Waybar](https://github.com/samucarre/Waybar/tree/gtk4-debian13-m3) | 14 | upstream PR Alexays/Waybar#5366 (GTK4 port) |
| [aquamarine](https://github.com/samucarre/aquamarine/tree/hdmi-apple-drm-ext-0.15) | 2 | hyprwm/aquamarine v0.15.1: apply only 0002; 0001 is the same change on v0.11.0 (branch `hdmi-apple-drm-ext`, Debian 13) |
| [postcard](https://github.com/samucarre/postcard/tree/debian13-m3) | 9 | gxanshu/postcard 1.15.0 |
| [omdrop-awdl](https://github.com/samucarre/omdrop-awdl/tree/discoverable-rapido) | 1 | netmojo-systems/omdrop-awdl main |
| [libva-v4l2_request](https://github.com/samucarre/libva-v4l2_request/tree/avd-fixes) | 0 | omacom/libva-v4l2_request (mirror, no changes) |
| [xdg-desktop-portal-gtk4](https://github.com/samucarre/xdg-desktop-portal-gtk4/tree/waybian) | 5 | mahkoh/xdg-desktop-portal-gtk4 master (`2372e15`); the branch's three README commits (screenshots) are left out |
| [linux-aurora](https://github.com/samucarre/linux-aurora) | 4 | omacom/linux-aurora `integration/aurora` (`30a6518`); one pull request each, #258–#261 |

To rebuild a fork from scratch: clone the upstream, check out the base, then `git am patches/<program>/*.patch`. Details per program in [docs/forks.md](../docs/forks.md).
