# Resizing Partitions: Making Root Bigger (Arch)

When `/` is full, `sudo pacman -Syu` stops with `error: Partition / too full`
and `not enough free disk space`, and the system can't be updated until it has
room again.

On a machine installed with archinstall, the usual cause is its disk layout.
With **ext4 (or xfs/f2fs) and a separate `/home` partition**, which archinstall
offers by default, it makes the root partition only **32 GiB on any disk under
320 GiB** (10% of the disk up to 500 GiB, 50 GiB above that). The rest of the
disk goes to `/home`. 32 GiB fills up quickly once the desktop, the
development tools and the AUR packages are installed, because pacman keeps a
copy of every package version it has downloaded.
(Source: archinstall 4.5, `archinstall/lib/disk/default_layouts.py`.)

The plan:

1. [Find out which layout you have](#1-find-out-which-layout-you-have).
2. [Free some space now](#2-free-some-space-now), enough to get today's
   update through. Takes minutes, no reboot.
3. [Grow root with GParted from a USB stick](#3-make-a-gparted-live-usb-stick)
   so it doesn't happen again. Takes an hour or so.
4. [Keep the package cache small](#7-keep-the-package-cache-small).

---

## 1. Find out which layout you have

```bash
lsblk -f
df -h / /home
```

The main case looks like this: three partitions on one disk, root nearly full,
`/home` mostly empty.

```
NAME        FSTYPE FSVER LABEL UUID      FSAVAIL FSUSE% MOUNTPOINTS
nvme0n1
├─nvme0n1p1 vfat   FAT32       ...        850M    17% /boot
├─nvme0n1p2 ext4   1.0         ...        1.1G    96% /
└─nvme0n1p3 ext4   1.0         ...        171G     8% /home
```

NVMe disks are named `nvme0n1` with partitions `nvme0n1p1`, `p2`, ...; SATA
disks are `sda` with `sda1`, `sda2`, .... The commands below use `nvme0n1`;
use the names your `lsblk` shows.

| What `lsblk -f` and `df` show | What it means | What to do |
|---|---|---|
| `/` and `/home` are two **ext4** partitions on the same disk | archinstall's separate `/home` | Steps 2 to 7 |
| `/` and `/home` show the **same device**, FSTYPE **btrfs** | One btrfs partition with subvolumes. There's nothing to resize: the whole disk is full | Step 2, then see [The whole disk is full](#the-whole-disk-is-full) |
| `/` and `/home` are on **different disks** (say `nvme0n1` and `sda`) | archinstall's two-disk layout. Root has its whole disk | Step 2, then see [The whole disk is full](#the-whole-disk-is-full) |
| `/home` is **xfs** | XFS can't shrink, so GParted can't take space from it | Step 2. Growing root means backing up `/home`, recreating it smaller, and restoring; not covered here |
| FSTYPE says **crypto_LUKS** or **LVM2_member** | Encrypted or LVM install | Step 2. Resizing goes through more layers; not covered here, so don't follow steps 3 to 6 |

---

## 2. Free some space now

Look at what takes the room on `/`:

```bash
sudo du -sh /usr /opt /var/cache/pacman/pkg /var/log /var/lib /root 2>/dev/null
```

The package cache (`/var/cache/pacman/pkg`) is usually the big one: pacman
keeps every version of every package it has downloaded. Clear it:

```bash
sudo pacman -Sc                       # drop cached packages that are no longer installed
sudo pacman -Scc                      # or empty the cache completely (it asks twice)
sudo journalctl --vacuum-size=200M    # trim the system log
```

The only cost of `-Scc` is that you can't downgrade a package from the cache
until it fills up again.

If you've built AUR packages, `~/.cache/yay` can be big too. It's on `/home`,
though, so it doesn't help root.

Then check, and update:

```bash
df -h /
sudo pacman -Syu
```

A few GiB free is usually enough for an update. If the cache was the problem,
you're unblocked now, and the steps below are about not landing here again.

> **Before you update a machine with the dotfiles:** Arch has shipped
> Hyprland 0.57 since 2026-11-03, and 0.57 no longer reads `hyprland.conf`.
> If the machine runs the dotfiles from `master`, Hyprland starts without
> your config after the update. See step 7 of the
> [fast path](arch-hyprland-guide.md#fast-path-arch-with-the-dotfiles-step-by-step).

---

## 3. Make a GParted Live USB stick

Making root bigger means taking space from the start of `/home`, which means
moving `/home`'s data along the disk. That can't happen while the system runs
from that disk, so you boot a live system from a USB stick. GParted Live is a
small one (about 600 MB) with a graphical partition editor that does the
moving for you. The Arch install stick has `parted` and `resize2fs`, but no
tool that moves a partition safely.

1. Download the **amd64** ISO, `gparted-live-<version>-amd64.iso`, from
   https://gparted.org/download.php (1.8.1-6 as of 2026-11).
2. Write it to a USB stick. **This erases the stick.** Use another stick than
   the Arch one if you have two; otherwise you can write Arch back onto it
   afterwards.

**On the Mac:**

```bash
diskutil list                      # find the stick: "(external, physical)", e.g. /dev/disk4
diskutil unmountDisk /dev/disk4
sudo dd if=$HOME/Downloads/gparted-live-1.8.1-6-amd64.iso of=/dev/rdisk4 bs=4m status=progress
diskutil eject /dev/disk4
```

**Check the disk number twice:** `dd` overwrites whichever disk you give it,
without asking. `rdisk` (raw disk) is the same stick as `disk`, just much
faster to write. If macOS then says it can't read the disk, click **Eject**;
it's a Linux stick.

**On Windows:** Rufus, as in [Part 1](arch-hyprland-guide.md#option-a-bootable-usb-bare-metal)
of the main guide. **On Linux:** `dd`, also in Part 1.

---

## 4. Back up first

GParted moves `/home`'s data block by block. That's reliable, but if the
power goes or the machine hangs halfway through, the partition is lost. A NUC
has no battery, so a power cut is a real risk. Before you start, get
everything you can't lose off the machine: push your git repos, and copy
anything else to another disk or machine.

---

## 5. Grow root with GParted

1. **Boot the stick.** Plug it in, power on, and press **F10** for the NUC's
   boot menu (F2 opens the BIOS setup). Pick the USB stick. Other machines
   use F12, F11 or Esc. Secure Boot must be off, as it was for the Arch
   install.

2. **Get through GParted Live's questions.** Press **Enter** at each one to
   accept the default: the boot menu (*GParted Live (Default settings)*), the
   keymap, the language, and the video mode. The US keymap is fine here:
   you'll mostly click, and type numbers. GParted opens by itself.

3. **Pick the disk.** At the top right, choose the system disk, e.g.
   `/dev/nvme0n1`. You should see the same three partitions as in `lsblk`:
   boot (fat32), root (ext4), home (ext4).

4. **Shrink `/home` from the left.** Right-click the home partition (the last
   one) → **Resize/Move**. In **Free space preceding (MiB)**, type how much
   root should grow, e.g. `65536` for 64 GiB. Leave the other fields alone;
   GParted shrinks `/home` from its start to make that room. Click
   **Resize/Move**.
   - How much? Root at about 100 GiB holds Arch, the dotfiles' packages and
     the AUR builds with room to spare: 32 + 64 = 96 GiB.
   - GParted won't let you take space that `/home` is using.

5. **Grow root.** Right-click the root partition → **Resize/Move**. Set
   **Free space following (MiB)** to `0`, or drag the right edge as far as it
   goes. Click **Resize/Move**.

6. **Apply.** Nothing has happened yet: the two operations wait in the list
   at the bottom. Click **Apply All Operations** (the green tick) → **Apply**.
   GParted warns that moving a partition can stop an operating system from
   booting. That's about partitions that hold the boot loader; the boot
   partition here is the first one, and it isn't touched. Go ahead.

7. **Wait.** Moving `/home` takes about as long as copying the data on it:
   minutes to an hour. GParted checks both filesystems and resizes them as part
   of the job. Don't power off. When it says *All operations successfully
   completed*, close GParted, double-click **Exit** on the desktop, choose
   **Reboot**, and remove the stick when it tells you to.

---

## 6. Check the result

Back in Arch:

```bash
df -h / /home
```

You should see root about 64 GiB bigger and `/home` that much smaller.
Nothing else needs changing: root only grew where it was, and `/home` kept
its filesystem UUID, which is what `/etc/fstab` uses to mount it. The boot
partition wasn't touched.

---

## 7. Keep the package cache small

pacman never clears its cache by itself. `paccache` (in `pacman-contrib`) with
its weekly timer keeps the three newest versions of each package and deletes
the rest:

```bash
sudo pacman -S pacman-contrib
sudo systemctl enable --now paccache.timer
```

The dotfiles' `install_arch.sh` doesn't set this up (as of 2026-11-07).

---

## The whole disk is full

With one btrfs partition, or with root on its own disk, there's no space
elsewhere on the disk to move into root. What helps:

- Step 2, and step 7 so the cache stays small.
- Look for anything big that doesn't need to be on that disk.
- A bigger SSD. Copying an existing install onto a new disk isn't covered
  here; reinstalling with the [fast path](arch-hyprland-guide.md#fast-path-arch-with-the-dotfiles-step-by-step)
  is often the simpler route.

---

## Next install: avoid the small root

In archinstall's disk configuration, choose **btrfs** with the default
subvolumes (one partition; `/` and `/home` share all the space), or **ext4**
and answer **No** to *Would you like to create a separate partition for
/home?*. Part 2 of the [main guide](arch-hyprland-guide.md#run-the-installer)
says the same.
