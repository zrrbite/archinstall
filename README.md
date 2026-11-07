# archinstall

Notes for installing and running Arch Linux machines. The package lists and
configs themselves live in the [dotfiles](https://github.com/zrrbite/dotfiles)
repo; these guides point to them rather than copying them.

| Guide | For |
|---|---|
| [Arch + Hyprland](doc/arch-hyprland-guide.md) | A blank disk to a Hyprland desktop. Its fast path sets it up with the dotfiles |
| [Resizing partitions](doc/resize-partitions.md) | Root is full and `pacman -Syu` fails: free space now, then grow root with GParted |
| [Omarchy](doc/omarchy-guide.md) | Omarchy on a second NUC, to compare with the dotfiles setup |
| [Claude Code](doc/claude-code-install.md) | Installing Claude Code on Arch |
| [Remote Claude Code](doc/remote-cc-access.md) | Reaching Claude Code on the Arch box from a phone or laptop: Tailscale, SSH, tmux |
| [Remote UE builds](doc/remote-ue-build-guide.md) | Draft, never tried: build Unreal on the Windows workstation, run it on Arch |
