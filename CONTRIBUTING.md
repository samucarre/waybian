# Contributing

Issues and Discussions are open to anyone invited to this repository. English and Spanish are both fine.

**Useful to us**

- Anything you know about `t6031` / G15 / the M3 generation that we got wrong or could do better: suspend, USB4/Thunderbolt, DisplayPort over USB‑C, HDMI above 60 Hz, GPU preemption (a long compute job versus the desktop), power management.
- Experience with the same board (MacBook Pro 16" M3 Max, `j516c`) or close ones (`j514`, `j516s`).
- Improvements to the forks: open the pull request on the fork (see [docs/forks.md](docs/forks.md)) and mention it here.

**When reporting something**

Include `uname -r`, the GRUB entry used, the relevant `dmesg`/`journalctl -k` lines and, for the GPU, the firmware log from `debugfs` if you have it. Say whether the Asahi installer was used in "UEFI environment only" mode and which `boot.bin` (device tree) you boot with.

**What you will not find here**

Configuration files with personal data, credentials, or the full change log. They live in a private lab repository; ask for a specific piece and it will be shared if it can be.
