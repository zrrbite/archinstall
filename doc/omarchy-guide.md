# Omarchy on a NUC

[Omarchy](https://omarchy.org) is a ready-made Arch Linux + Hyprland desktop.
This guide installs it on a second NUC, to compare with the first NUC's
Arch + dotfiles setup ([arch-hyprland-guide.md](arch-hyprland-guide.md)).

Checked against **Omarchy 4.5.0** (released 2026-11-05) and its manual at
[omarchy.org/manual](https://omarchy.org/manual/). The older manual at
learn.omacom.io is for v3; ignore it.

How it differs from the Arch guide: there's no archinstall and no dotfiles.
Omarchy installs from its own ISO, wipes the disk, encrypts it, and sets up
its own Hyprland config, bar, launcher, apps and theme. The ISO is the only
supported way to install it; the old `curl ... | bash` installer was removed
in June 2026. **Don't apply the dotfiles on this NUC.** They would overwrite
Omarchy's own files in `~/.config` and mix the two setups you're comparing.

---

## Before you start

- **A wired keyboard**, or one with a 2.4 GHz USB dongle. The disk password
  is typed at every boot, before Bluetooth is running.
- **A USB stick of 8 GB or more.** The ISO is 6.1 GiB. Writing it erases the
  stick.
- **The disk gets wiped.** A full-disk install erases the drive you choose. If
  the NUC has two drives, only the one you pick is touched.
- **The network is optional for the install.** Every package is on the ISO.
  After the first boot, wired works on its own and Wi-Fi is one menu away.

---

## 1. Download and check the ISO

On the Mac:

```bash
cd ~/Downloads
curl -LO https://iso.omarchy.org/omarchy-4.5.0-x86.iso
curl -LO https://iso.omarchy.org/omarchy-4.5.0-x86.iso.sha256
shasum -a 256 -c omarchy-4.5.0-x86.iso.sha256
```

You should see `omarchy-4.5.0-x86.iso: OK`. For a newer release, take the
link from omarchy.org's download button; the file names follow the same
pattern.

## 2. Write the USB stick

Omarchy's manual suggests [balenaEtcher](https://etcher.balena.io/) on the
Mac: pick the ISO, pick the stick, Flash. Or use `dd`. **Check the disk number
twice**, because `dd` overwrites whichever disk you name:

```bash
diskutil list                      # find the stick: "(external, physical)", e.g. /dev/disk4
diskutil unmountDisk /dev/disk4
sudo dd if=$HOME/Downloads/omarchy-4.5.0-x86.iso of=/dev/rdisk4 bs=4m status=progress
diskutil eject /dev/disk4
```

If macOS then says it can't read the disk, click **Eject**. On Windows, use
Rufus as in [Part 1](arch-hyprland-guide.md#option-a-bootable-usb-bare-metal)
of the Arch guide.

## 3. BIOS settings

Power on the NUC and press **F2** for the BIOS setup.

- **Turn Secure Boot off.** Omarchy won't install with it on.
- Omarchy's manual says to turn off "Secure Boot and/or TPM". Try with only
  Secure Boot off first. If the installer won't boot or fails, turn off TPM
  too.
- Leave the boot mode on **UEFI**.

Save and exit. On the way back up, press **F10** for the boot menu and pick
the USB stick.

## 4. The installer

It asks its questions first, then shows them on one screen to confirm. **Don't
press Ctrl+C** on the first screen: that switches to setting the machine up
for another owner.

1. **Language:** English, or whichever you like.
2. **Keyboard layout: Danish.** This matters more than it looks. It takes
   effect at once, and the **disk password is typed in this layout at every
   boot**. Choose it before you type any password.
3. **Timezone:** start typing `Copenhagen`.
4. **Username.**
5. **Password**, twice. One password is used for your user, for root and for
   the disk encryption.
6. **Hostname.** The default is `omarchy`. Give it its own name, such as
   `nuc2`, so you can tell the two NUCs apart on the network and on Tailscale.
7. **Summary:** "Does this look right?" Check the keyboard layout.
8. **Disk:** pick the NUC's SSD. Check the size against what you expect.
9. **Install mode: Full disk.** This screen only appears if the disk has
   free space.
10. **"Everything will be overwritten."** Press Enter to go ahead. Pressing
    Ctrl+C here instead installs **without** encryption: you skip the
    password at boot, but the machine no longer runs the way Omarchy is meant
    to, which matters for a comparison. Keep the encryption unless you have a
    reason.

The install takes under five minutes. Then choose **Reboot Now** and remove the
stick.

What you get: a 2 GiB boot partition, then the rest of the disk encrypted
(LUKS2) and formatted as btrfs. The boot loader is Limine, and Snapper takes a
snapshot of the system before each update. There's no separate `/home`
partition, so the small-root problem from
[resize-partitions.md](resize-partitions.md) can't happen here.

## 5. First boot

You'll see the **Limine** boot menu, then the **password screen**. Type the
disk password; Omarchy logs you in by itself after that, because the disk
password already proved who you are.

**If the screen stays black after the boot menu:** this has been reported on
a NUC (Omarchy issue #7275). The password screen may be there but not
showing. Type the password blind and press Enter. To see what's happening
instead, press **E** in the Limine menu, delete
`quiet splash loglevel=0 systemd.show_status=false` from the kernel line, add
`loglevel=7 systemd.show_status=true plymouth.enable=0`, and press **F10**
to boot.

Notifications offer the next steps: the key bindings, Wi-Fi, updating, and
Git setup (your name and email).

- **Wi-Fi:** `Super+Ctrl+N` opens the network panel, or run `nmtui` in a
  terminal. It's NetworkManager, as on the Arch NUC.
- **Scaling:** Omarchy assumes a high-density "retina" monitor and scales
  everything up. On a 1080p or 1440p monitor that makes everything too big.
  Press `Super+Alt+/` to step to a lower scale (`Super+/` steps back up).
  Or set both of these to `1` in `~/.config/hypr/monitors.lua`:
  ```lua
  local omarchy_gdk_scale = 1
  local omarchy_monitor_scale = 1
  ```
  Then close the windows that are already open; new ones get the new size.
- **Keyboard layout, if Danish was missed:** *Setup > Typing > Keyboard
  Layouts* in the menu, or `omarchy setup keyboard`. The disk password prompt
  may keep the layout from the install.

### The keys you need first

`Super+K` lists all the key bindings.

| Keys | Action |
|---|---|
| `Super+Space` | Omarchy menu: apps, settings, everything |
| `Super+K` | Key binding help |
| `Super+Return` | Terminal (foot) |
| `Super+Shift+B` | Browser (Chromium) |
| `Super+Shift+N` | Editor (Neovim) |
| `Super+W` or `Super+Q` | Close window |
| `Super+F` | Full screen |
| `Super+1/2/3/4` | Go to workspace |
| `Super+Shift+1/2/3/4` | Move window to workspace |
| `Super+Ctrl+T` | Pick a theme |
| `Super+Ctrl+N` | Network panel |
| `Super+Ctrl+L` | Lock |
| `Super+Escape` | System menu: suspend, restart, shut down |

**Coming from the dotfiles:** `Super+Q` opens a terminal there but **closes
the window** in Omarchy. The terminal is `Super+Return`.

## 6. Updating

Use **Update > Omarchy** in the menu, the circular-arrow icon next to the
clock, or `omarchy update` in a terminal. **Don't run `sudo pacman -Syu`**:
Omarchy blocks it and points you to `omarchy update`, which takes a snapshot
first, updates the packages, and runs Omarchy's migrations.

New installs are on the **stable** channel. It updates packages from an Arch
mirror Omarchy keeps about a month behind, and it holds Hyprland back until
Omarchy's config is ready for the new version. On 2026-11-07, stable runs
Hyprland **0.56.2**, while Arch itself has 0.57.1. That is exactly the update
that leaves the dotfiles' `hyprland.conf` unread on the Arch NUC. Stay on
stable for the comparison; the edge channel has open 0.57 bugs.

**Rolling back a bad update:** reboot and pick the snapshot from before the
update in the Limine menu. Inside it, click the notification or run
`omarchy-snapshot restore`. This restores the system, but **not `/home`**, so
`~/.config` stays as it is.

## 7. Where Omarchy's config lives

- **Omarchy's defaults:** `/usr/share/omarchy`. This belongs to the
  `omarchy` package and is replaced on every update. Don't edit it.
- **Your changes:** `~/.config`. The Hyprland config is already in Lua (since
  Omarchy 4.0), split into `~/.config/hypr/hyprland.lua` plus `bindings.lua`,
  `monitors.lua`, `input.lua`, `looknfeel.lua` and `autostart.lua`. The
  terminal config is `~/.config/foot/foot.ini`, and the shell is bash
  (`~/.bashrc`).
- **To start over:** `omarchy reinstall configs` puts back Omarchy's default
  files, overwriting your changes.

---

## Comparing it with the Arch + dotfiles NUC

| | Arch + dotfiles (NUC 1) | Omarchy (NUC 2) |
|---|---|---|
| Install | archinstall, then `install_arch.sh` with AUR builds: an hour or more | ISO, offline, under five minutes |
| Disk | Whatever you picked in archinstall, unencrypted | Whole disk, encrypted, btrfs |
| Hyprland config | `hyprland.conf` on `master`, which 0.57 can't read; Lua on branch `hypr-lua` | Lua since 4.0 |
| Hyprland version | Whatever Arch ships: 0.57.1 | Held back by Omarchy: 0.56.2 on stable |
| Updates | `pacman -Syu`, `yay` | `omarchy update`: snapshot, then packages, then migrations |
| Undoing an update | No snapshots | Pick a snapshot in the boot menu |
| Bar, launcher, notifications | waybar, rofi, mako | One Quickshell process |
| Terminal and shell | foot, zsh with starship | foot, bash |
| Look | Nord, fixed | Theme picker |
| Close window / terminal | `Super+C` / `Super+Q` | `Super+W` or `Super+Q` / `Super+Return` |
| Key binding list | `Super+F1` | `Super+K` |

Things worth writing down while you use it:

- What you miss from the dotfiles, and what you don't.
- What Omarchy does that the dotfiles could take over: snapshots before every
  update, holding Hyprland back until the config is ready, the Lua config
  split into files by topic.
- How the updates go over a few weeks, compared with the Arch NUC.
