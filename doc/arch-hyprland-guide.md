# Arch Linux + Hyprland Setup Guide

A complete guide for setting up Arch Linux with Hyprland compositor.

---

## Part 1: Preparing Installation Media

Download the Arch ISO from https://archlinux.org/download/

### Option A: Bootable USB (Bare Metal)

Use [Rufus](https://rufus.ie/) on Windows to create a bootable USB drive:

1. Download and run Rufus
2. Insert a USB drive (4GB+ recommended)
3. Select your USB drive under "Device"
4. Click "SELECT" and choose the Arch ISO
5. Partition scheme: **GPT** (for UEFI) or **MBR** (for legacy BIOS)
6. File system: **FAT32**
7. Click "START"
8. When prompted, select "Write in ISO Image mode"
9. Wait for completion

**On Linux**, you can use `dd`:

```bash
sudo dd bs=4M if=archlinux-*.iso of=/dev/sdX status=progress oflag=sync
```

Replace `/dev/sdX` with your USB device (check with `lsblk`). **Warning:** This will erase the USB drive.

Boot from the USB:
1. Insert USB into target machine
2. Enter BIOS/UEFI (usually F2, F12, Del, or Esc during boot)
3. Set USB as first boot device, or use boot menu
4. Save and reboot

### Option B: VirtualBox VM (Testing/Learning)

VirtualBox is useful for testing or learning before committing to bare metal.

1. Create new VM:
   - Name: Arch
   - Type: Linux
   - Version: Arch Linux (64-bit)

2. Resources:
   - RAM: 4096 MB (minimum 2048)
   - Hard disk: 20GB+, VDI, dynamically allocated

3. Display settings (Settings → Display):
   - Video Memory: **128 MB**
   - Graphics Controller: **VMSVGA**
   - Enable **3D Acceleration**

4. Network settings (Settings → Network → Adapter 1):
   - Enable Network Adapter: **checked**
   - Attached to: **NAT**

5. Mount Arch ISO:
   - Settings → Storage → Empty disk under Controller: IDE
   - Click disk icon → Choose your Arch ISO

6. Boot the VM

> **Note:** VirtualBox-specific steps are marked with **(VirtualBox)** throughout this guide. Skip these on bare metal.

---

## Part 2: Arch Installation

### Verify network in live ISO

```bash
ip a
ping -c 3 archlinux.org
```

### Run the installer

```bash
archinstall
```

**Critical settings in archinstall:**
- **Network configuration**: Select **Use Network Manager (default backend)** (important!)
- **Mirrors and repositories → Optional repositories**: Enable **multilib** (needed for `lib32-*` packages such as the 32-bit Vulkan loader)
- Everything else to your preference

Complete the installation and reboot.

---

## Part 3: Post-Install Setup

> **Quick Setup Alternative:** If you want a pre-configured Nord-themed Hyprland environment instead of manual configuration, see [Using Dotfiles](#using-dotfiles) after completing the post-install network setup. The dotfiles repo automates Parts 4-9 with tested configs.

### If you have no network after reboot

```bash
sudo systemctl enable --now systemd-networkd
sudo systemctl enable --now systemd-resolved

sudo tee /etc/systemd/network/20-wired.network << EOF
[Match]
Name=enp0s3

[Network]
DHCP=yes
EOF

sudo systemctl restart systemd-networkd
```

---

## Using Dotfiles

For a quick, pre-configured setup instead of manual configuration, use the [dotfiles repo](https://github.com/zrrbite/dotfiles):

```bash
git clone https://github.com/zrrbite/dotfiles.git ~/dotfiles
cd ~/dotfiles && ./install_arch.sh
```

This installs all packages and symlinks configs for:
- **Hyprland** + hyprpaper + hyprlock + hypridle + cliphist
- **Foot** terminal (Nord theme, transparency)
- **Waybar** status bar (Nord theme)
- **Rofi** app launcher (replaces wofi)
- **Mako** notifications
- **Starship** prompt
- **Neovim** IDE setup (LSP, treesitter)
- **Git** config with aliases
- Audio via pipewire

After install, log out and select Hyprland as your session.

See `~/dotfiles/README.md` for key bindings (`Super + F1` shows all) and how to manage configs with GNU Stow.

> **Note:** If you prefer manual control or want to understand each component, continue with the sections below. The dotfiles can also serve as reference configs.

> **Heads-up:** The dotfiles' Hyprland config is still `hyprland.conf`, which Hyprland 0.57 stops loading. This guide uses the new `hyprland.lua` format — see [Part 5](#part-5-hyprland-configuration).

---

## Part 4: Install Hyprland and Desktop Environment

> **Bare Metal:** Install GPU drivers first! See [Bare Metal Differences](#bare-metal-differences) for AMD/NVIDIA/Intel driver setup.

### Core packages

```bash
sudo pacman -S hyprland foot wofi waybar hyprpaper hypridle hyprlock brightnessctl asciiquarium mako
```

- `hyprland` — Wayland compositor
- `foot` — terminal emulator
- `wofi` — application launcher
- `waybar` — status bar
- `hyprpaper` — wallpaper manager
- `hypridle` — idle daemon (screensaver/lock triggers)
- `hyprlock` — screen locker
- `brightnessctl` — screen brightness control
- `asciiquarium` — ASCII art aquarium screensaver
- `mako` — notification daemon (some apps, Discord included, can freeze without one)

### Portal, polkit agent and Qt support

```bash
sudo pacman -S xdg-desktop-portal-hyprland xdg-desktop-portal-gtk hyprpolkitagent qt5-wayland qt6-wayland
```

- `xdg-desktop-portal-hyprland` — Hyprland's portal backend (screen sharing, screenshots). Use this instead of `xdg-desktop-portal-wlr`
- `xdg-desktop-portal-gtk` — file picker dialogs, which the Hyprland portal doesn't provide
- `hyprpolkitagent` — the password prompt when an app asks for elevated privileges. `polkit` alone has no prompt
- `qt5-wayland`, `qt6-wayland` — native Wayland for Qt apps

The polkit agent is started from the autostart block in Part 5.

### Fonts (fixes waybar icons)

```bash
sudo pacman -S otf-font-awesome ttf-nerd-fonts-symbols noto-fonts
```

`noto-fonts` gives you a sans-serif font; without one, some apps render squares instead of text.

### VirtualBox guest additions (VirtualBox only)

```bash
sudo pacman -S virtualbox-guest-utils
sudo systemctl enable --now vboxservice
```

Skip this on bare metal.

### Clipboard support

```bash
sudo pacman -S wl-clipboard
```

### Screenshots and screen recording

```bash
sudo pacman -S grim slurp wf-recorder
```

- `grim` — screenshot tool
- `slurp` — region selection
- `wf-recorder` — screen recording

Add keybindings to the key bindings section of `~/.config/hypr/hyprland.lua` (Part 5):

```lua
-- Screenshots
hl.bind("Print", hl.dsp.exec_cmd("grim ~/Pictures/Screenshots/$(date +%Y%m%d_%H%M%S).png"))
hl.bind(mainMod .. " + SHIFT + S", hl.dsp.exec_cmd('grim -g "$(slurp)" ~/Pictures/Screenshots/$(date +%Y%m%d_%H%M%S).png'))

-- Screen recording
hl.bind(mainMod .. " + SHIFT + R", hl.dsp.exec_cmd("wf-recorder -f ~/Videos/recording_$(date +%Y%m%d_%H%M%S).mp4"))
hl.bind(mainMod .. " + SHIFT + BackSpace", hl.dsp.exec_cmd("pkill -INT wf-recorder"))
```

Create the directories:

```bash
mkdir -p ~/Pictures/Screenshots ~/Videos
```

### Audio

```bash
sudo pacman -S pipewire pipewire-pulse wireplumber pamixer
systemctl --user enable --now pipewire pipewire-pulse wireplumber
```

Optional GUI mixer:
```bash
sudo pacman -S pavucontrol
```

Add volume keybinds to `hyprland.lua` (`locked` keeps them working on the lock screen, `repeating` lets you hold the key):
```lua
hl.bind("XF86AudioRaiseVolume", hl.dsp.exec_cmd("pamixer -i 5"), { locked = true, repeating = true })
hl.bind("XF86AudioLowerVolume", hl.dsp.exec_cmd("pamixer -d 5"), { locked = true, repeating = true })
hl.bind("XF86AudioMute",        hl.dsp.exec_cmd("pamixer -t"),   { locked = true })
```

### Bluetooth

```bash
sudo pacman -S bluez bluez-utils blueman
sudo systemctl enable --now bluetooth
```

Pair devices using `bluetoothctl`:

```bash
bluetoothctl
```

In the interactive prompt:

```
power on
agent on
default-agent
scan on
# Wait for device to appear: [NEW] Device AA:BB:CC:DD:EE:FF DeviceName
pair AA:BB:CC:DD:EE:FF
trust AA:BB:CC:DD:EE:FF
connect AA:BB:CC:DD:EE:FF
exit
```

Replace `AA:BB:CC:DD:EE:FF` with your device's address.

For a GUI, run `blueman-manager`.

---

## Part 5: Hyprland Configuration

> **Reference:** See `~/dotfiles/hypr/` for a complete Nord-themed Hyprland config with hyprlock and clipboard history (still in the old `.conf` format).

> **Config format:** Hyprland 0.55 moved its config to Lua, in `~/.config/hypr/hyprland.lua`. The old `hyprland.conf` still loads on 0.56 with a deprecation notice, and support is removed in 0.57. If both files exist, `hyprland.lua` wins. A pre-0.55 config won't carry over as-is either: 0.53 replaced the window rule syntax, so `windowrulev2` lines are now errors.

### ~/.config/hypr/hyprland.lua

```bash
mkdir -p ~/.config/hypr
nano ~/.config/hypr/hyprland.lua
```

If you launch Hyprland without a config, it writes the upstream example config here, which is a useful reference (also at `/usr/share/hypr/hyprland.lua`). Replace it with:

```lua
-- Monitor config (VirtualBox example - see "Bare Metal Differences" for real hardware)
hl.monitor({
    output   = "Virtual-1",
    mode     = "1920x1080@60",
    position = "0x0",
    scale    = 1,
})

-- Variables
local terminal = "foot"
local menu     = "wofi --show drun"
local mainMod  = "SUPER"

-- Autostart
hl.on("hyprland.start", function()
    hl.exec_cmd("hyprpaper")
    hl.exec_cmd("waybar")
    hl.exec_cmd("hypridle")
    hl.exec_cmd("mako")
    hl.exec_cmd("systemctl --user start hyprpolkitagent")
end)

-- VirtualBox only: software cursor (remove on bare metal)
hl.config({
    cursor = {
        no_hardware_cursors = 1,
    },
})

-- Danish keyboard layout
hl.config({
    input = {
        kb_layout = "dk",
    },
})

-- Key bindings
hl.bind(mainMod .. " + Q", hl.dsp.exec_cmd(terminal))
hl.bind(mainMod .. " + C", hl.dsp.window.close())
hl.bind(mainMod .. " + M", hl.dsp.exit())
hl.bind(mainMod .. " + R", hl.dsp.exec_cmd(menu))
hl.bind(mainMod .. " + V", hl.dsp.window.float({ action = "toggle" }))
hl.bind(mainMod .. " + F", hl.dsp.window.fullscreen())

-- Move focus
hl.bind(mainMod .. " + left",  hl.dsp.focus({ direction = "left" }))
hl.bind(mainMod .. " + right", hl.dsp.focus({ direction = "right" }))
hl.bind(mainMod .. " + up",    hl.dsp.focus({ direction = "up" }))
hl.bind(mainMod .. " + down",  hl.dsp.focus({ direction = "down" }))

-- Switch to workspace / move window to workspace
for i = 1, 9 do
    hl.bind(mainMod .. " + " .. i,         hl.dsp.focus({ workspace = i }))
    hl.bind(mainMod .. " + SHIFT + " .. i, hl.dsp.window.move({ workspace = i }))
end

-- Mouse bindings
hl.bind(mainMod .. " + mouse:272", hl.dsp.window.drag(),   { mouse = true })
hl.bind(mainMod .. " + mouse:273", hl.dsp.window.resize(), { mouse = true })

-- Scroll through workspaces
hl.bind(mainMod .. " + mouse_down", hl.dsp.focus({ workspace = "e+1" }))
hl.bind(mainMod .. " + mouse_up",   hl.dsp.focus({ workspace = "e-1" }))

-- Preselect where the next window opens
hl.bind(mainMod .. " + B", hl.dsp.layout("preselect d"))  -- below
hl.bind(mainMod .. " + N", hl.dsp.layout("preselect r"))  -- right

-- Window groups (tabbed stacking)
hl.bind(mainMod .. " + G",           hl.dsp.group.toggle())
hl.bind(mainMod .. " + Tab",         hl.dsp.group.next())
hl.bind(mainMod .. " + SHIFT + Tab", hl.dsp.group.prev())
hl.bind(mainMod .. " + SHIFT + left",  hl.dsp.window.move({ direction = "left",  group_aware = true }))
hl.bind(mainMod .. " + SHIFT + right", hl.dsp.window.move({ direction = "right", group_aware = true }))
hl.bind(mainMod .. " + SHIFT + up",    hl.dsp.window.move({ direction = "up",    group_aware = true }))
hl.bind(mainMod .. " + SHIFT + down",  hl.dsp.window.move({ direction = "down",  group_aware = true }))
```

Hyprland reloads the config when the file changes. Errors show up as a banner at the top of the screen. If the config fails before any binds are registered, Hyprland adds emergency binds: `Super + Q` (terminal), `Super + M` (exit).

**Editor autocompletion:** Hyprland ships Lua type stubs in `/usr/share/hypr/stubs/`. Point your Lua language server at them with a `.luarc.json` in `~/.config/hypr/`:

```json
{
    "workspace": {
        "library": ["/usr/share/hypr/stubs"]
    }
}
```

### Idle/screensaver with hypridle

Create `~/.config/hypr/hypridle.conf`:

hypridle keeps its own `.conf` format; only the `hyprctl dispatch` commands it runs use the new Lua syntax.

```ini
general {
    lock_cmd = pidof hyprlock || hyprlock
    before_sleep_cmd = loginctl lock-session
    after_sleep_cmd = hyprctl dispatch 'hl.dsp.dpms({ action = "enable" })'
}

# Dim screen after 4 minutes
listener {
    timeout = 240
    on-timeout = brightnessctl -s set 10
    on-resume = brightnessctl -r
}

# Screensaver (asciiquarium) after 4 minutes
listener {
    timeout = 240
    on-timeout = foot --fullscreen --title=screensaver asciiquarium
    on-resume = pkill -f "foot.*screensaver"
}

# Lock screen after 5 minutes (kills screensaver first)
listener {
    timeout = 300
    on-timeout = pkill -f "foot.*screensaver"; loginctl lock-session
}

# Turn off display after 15 minutes
listener {
    timeout = 900
    on-timeout = hyprctl dispatch 'hl.dsp.dpms({ action = "disable" })'
    on-resume = hyprctl dispatch 'hl.dsp.dpms({ action = "enable" })'
}

# Suspend after 30 minutes
listener {
    timeout = 1800
    on-timeout = systemctl suspend
}
```

Install asciiquarium for the screensaver:

```bash
sudo pacman -S asciiquarium
```

hypridle is already in the autostart block of `hyprland.lua` above.

### Workspace presets

Launch apps on specific workspaces at startup. `hl.exec_cmd` takes window rules as a second argument:

```lua
-- In hyprland.lua, inside the hl.on("hyprland.start", ...) autostart block
hl.exec_cmd("foot",         { workspace = "1" })
hl.exec_cmd("firefox",      { workspace = "2" })
hl.exec_cmd("discord",      { workspace = "3" })
hl.exec_cmd("foot -e btop", { workspace = "3" })  -- tiled alongside discord
```

These rules follow the launched process's PID, so they miss apps that fork before opening a window. For those, assign the app to a workspace with a window rule, which applies whenever it opens:

```lua
hl.window_rule({ match = { class = "^(firefox)$" }, workspace = "2" })
hl.window_rule({ match = { class = "^(discord)$" }, workspace = "3" })
```

Find window class names with:

```bash
hyprctl clients | grep class
```

### Auto-start Hyprland on login

Edit `~/.bash_profile`:

```bash
nano ~/.bash_profile
```

Add at the end:

```bash
if [ -z "$DISPLAY" ] && [ "$XDG_VTNR" = 1 ]; then
    exec start-hyprland
fi
```

`start-hyprland` (Hyprland 0.53+) is the supported launcher: if Hyprland crashes, it restarts it in safe mode. Running `Hyprland` directly still works but shows a warning notification.

> **Upgrading an older setup:** remove any `WLR_NO_HARDWARE_CURSORS` / `WLR_RENDERER_ALLOW_SOFTWARE` exports. Hyprland stopped using wlroots in 0.42, so these variables do nothing. The cursor workaround is now the `cursor` setting in `hyprland.lua`.

---

## Part 6: Foot Terminal Configuration

> **Reference:** See `~/dotfiles/foot/` for a Nord-themed foot config with transparency and padding.

### ~/.config/foot/foot.ini

```bash
mkdir -p ~/.config/foot
nano ~/.config/foot/foot.ini
```

```ini
[main]
font=monospace:size=14

[key-bindings]
font-increase=Control+Shift+plus
font-decrease=Control+Shift+minus
font-reset=Control+Shift+0
```

---

## Part 7: Terminal Customization

### Themes for Foot

Foot uses manual color configuration. Add to `~/.config/foot/foot.ini`:

**Dracula theme example** (foot 1.26 renamed `[colors]` to `[colors-dark]`, and 1.28 removed the old name):

```ini
[colors-dark]
background=282a36
foreground=f8f8f2
regular0=21222c
regular1=ff5555
regular2=50fa7b
regular3=f1fa8c
regular4=bd93f9
regular5=ff79c6
regular6=8be9fd
regular7=f8f8f2
bright0=6272a4
bright1=ff6e6e
bright2=69ff94
bright3=ffffa5
bright4=d6acff
bright5=ff92df
bright6=a4ffff
bright7=ffffff
```

**Pre-made themes:** foot ships dozens in `/usr/share/foot/themes/` (dracula, nord, gruvbox, catppuccin, tokyonight, ...). Include one in `foot.ini`:

```ini
include=/usr/share/foot/themes/dracula
```

### System info on launch (fastfetch)

```bash
sudo pacman -S fastfetch
echo "fastfetch" >> ~/.bashrc
```

Shows system info with ASCII logo when you open a terminal.

### ASCII text banners

```bash
sudo pacman -S figlet toilet
```

Add to `~/.bashrc`:

```bash
figlet "Arch"
# Or fancier:
toilet -f mono12 "Arch" --metal
```

### Fortune and cowsay

```bash
sudo pacman -S fortune-mod cowsay
```

Add to `~/.bashrc`:

```bash
fortune | cowsay
```

Random quotes with ASCII cow on each terminal launch.

---

## Part 8: Wallpaper Setup

### Download a wallpaper

```bash
mkdir -p ~/Pictures
cd ~/Pictures
curl -L -o wallpaper.jpg "https://images.unsplash.com/photo-1511300636408-a63a89df3482?w=1920"
```

### ~/.config/hypr/hyprpaper.conf

hyprpaper 0.8 rewrote its config format: `preload` is gone, and an old-style config makes hyprpaper refuse to start.

```bash
nano ~/.config/hypr/hyprpaper.conf
```

```ini
wallpaper {
    monitor =
    path = ~/Pictures/wallpaper.jpg
    fit_mode = cover
}

splash = false
```

An empty `monitor` makes this the fallback for every display. For a different wallpaper per display, add one `wallpaper { }` block per monitor name from `hyprctl monitors`. `path` can also be a directory, which turns it into a slideshow (see `timeout` and `order` on the [hyprpaper wiki page](https://wiki.hypr.land/Hypr-Ecosystem/hyprpaper/)).

### Reload hyprpaper

After changing the config:

```bash
pkill hyprpaper && hyprpaper &
```

Or change wallpaper on the fly without editing the config (empty monitor = all displays):

```bash
hyprctl hyprpaper wallpaper ", $HOME/Pictures/other.jpg"
```

---

## Part 9: Waybar Configuration

> **Reference:** See `~/dotfiles/waybar/` for a complete Nord-themed waybar with workspaces, clock, and system info.

The default waybar config may not show Hyprland workspaces. Create a custom config:

```bash
mkdir -p ~/.config/waybar
nano ~/.config/waybar/config
```

```json
{
    "layer": "top",
    "modules-left": ["hyprland/workspaces"],
    "modules-center": ["hyprland/window"],
    "modules-right": ["cpu", "memory", "clock"],
    
    "hyprland/workspaces": {
        "format": "{id}"
    },
    "clock": {
        "format": "{:%H:%M}"
    },
    "cpu": {
        "format": "CPU {usage}%"
    },
    "memory": {
        "format": "MEM {}%"
    }
}
```

Restart waybar:

```bash
pkill waybar; waybar &
```

---

## Part 10: Development Tools

> **Reference:** The dotfiles repo includes configs for git (with extensive aliases), neovim (LSP + treesitter), starship prompt, and clang-format/clang-tidy. See `~/dotfiles/README.md` for the full list.

> **AUR:** Some packages below (`p4`, `p4v`, `rider`) come from the AUR. Install `yay` first, see [Part 12](#part-12-aur-and-google-chrome).

### Essential packages

```bash
sudo pacman -S git base-devel cmake make neovim tmux
```

### Git configuration

The dotfiles include a comprehensive `.gitconfig` with aliases and settings. After running `install_arch.sh`, edit your name and email:

```bash
nano ~/dotfiles/git/.gitconfig
```

Update the `[user]` section:

```ini
[user]
    name = Your Name
    email = your@email.com
```

The config includes useful aliases like `git st`, `git lg`, `git up`, and more. View all aliases with `git la`.

The file also has commented-out examples for Windows (KDiff3, VS Code, wincred), macOS, and P4Merge at the bottom.

### GitHub SSH setup (recommended)

SSH keys are more convenient than tokens — no password prompts for push/pull.

```bash
sudo pacman -S openssh
```

Generate an SSH key:

```bash
ssh-keygen -t ed25519 -C "your@email.com"
```

Press Enter to accept the default location (`~/.ssh/id_ed25519`). Optionally set a passphrase.

Start the SSH agent and add your key:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

Copy your public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Add the key to GitHub:
1. Go to https://github.com/settings/keys
2. Click "New SSH key"
3. Paste your public key and save

Test the connection:

```bash
ssh -T git@github.com
```

You should see: "Hi username! You've successfully authenticated..."

**Using SSH URLs:** Clone repos with SSH URLs instead of HTTPS:

```bash
# SSH (recommended)
git clone git@github.com:username/repo.git

# Instead of HTTPS
git clone https://github.com/username/repo.git
```

**Switch existing repo from HTTPS to SSH:**

```bash
# Check current remote
git remote -v

# Update to SSH
git remote set-url origin git@github.com:username/repo.git
```

**Auto-start SSH agent:** Add to `~/.bashrc`:

```bash
if [ -z "$SSH_AUTH_SOCK" ]; then
    eval "$(ssh-agent -s)" > /dev/null
    ssh-add ~/.ssh/id_ed25519 2> /dev/null
fi
```

### Perforce (P4)

```bash
yay -S p4 p4v
```

- `p4` — command-line client
- `p4v` — visual client (GUI)

**Fix p4v symlink errors:** If you get permission errors on first run:

```bash
sudo ln -sf /usr/lib/libssl.so /usr/share/p4v/lib/libssl.so
sudo ln -sf /usr/lib/libcrypto.so /usr/share/p4v/lib/libcrypto.so
```

Configure your workspace:

```bash
p4 set P4PORT=your-server:1666
p4 set P4USER=your-username
p4 set P4CLIENT=your-workspace-name
```

Common commands:

```bash
p4 sync           # Get latest
p4 edit file      # Check out for edit
p4 add file       # Mark new file for add
p4 submit         # Submit changelist
p4 revert file    # Revert changes
p4 changes -m 10  # Recent changelists
```

### Starship prompt (git branch, status, etc.)

```bash
sudo pacman -S starship ttf-jetbrains-mono-nerd
echo 'eval "$(starship init bash)"' >> ~/.bashrc
fc-cache -fv
source ~/.bashrc
```

Update font in `~/.config/foot/foot.ini`:

```ini
[main]
font=JetBrainsMono Nerd Font Mono:size=14
```

Open a new terminal for changes to take effect.

**Presets:**

```bash
starship preset --list                              # List all presets
starship preset nerd-font-symbols -o ~/.config/starship.toml
starship preset tokyo-night -o ~/.config/starship.toml
starship preset pastel-powerline -o ~/.config/starship.toml
```

**VS Code terminal font:**

Add to `~/.config/Code - OSS/User/settings.json` (that's the `code` package; `visual-studio-code-bin` uses `~/.config/Code/User/settings.json`):

```json
{
    "terminal.integrated.fontFamily": "JetBrainsMono Nerd Font Mono"
}
```

### LLVM/Clang toolchain

```bash
sudo pacman -S llvm clang
```

The old `clang-tools-extra` package is now part of `clang`. This gives you:
- `clang` / `clang++` — compilers
- `clangd` — language server
- `clang-format` — code formatter
- `clang-tidy` — linter

### Claude Code

Use the native installer. It doesn't need Node.js and updates itself:

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

It installs to `~/.local/bin/claude`, so make sure `~/.local/bin` is on your `PATH`. Run with `claude`. It will prompt for authentication on first run.

See [claude-code-install.md](claude-code-install.md) for PATH setup and for removing an old npm install.

### VS Code

```bash
sudo pacman -S code
```

The `code` package is **Code - OSS**, the open-source build. It installs extensions from Open VSX instead of Microsoft's marketplace. Everything below is on Open VSX.

**VS Code setup:**
1. Open VS Code: `code`
2. Install the **clangd** extension (by LLVM)
3. Install **Vim** extension (by vscodevim) for vim keybindings
4. Install **CodeLLDB** extension for debugging

**Need Microsoft-only extensions?** Remote - SSH (used in [remote-ue-build-guide.md](remote-ue-build-guide.md)) and Microsoft's C/C++ extension only work in the proprietary build. Install that from the AUR instead:

```bash
yay -S visual-studio-code-bin
```

### JetBrains Rider (recommended for Unreal Engine)

Rider has superior UE support: native project handling, UnrealLink plugin, Blueprint references, and a debugger that works well with UE.

```bash
yay -S rider
```

Or install JetBrains Toolbox to manage all JetBrains IDEs:

```bash
yay -S jetbrains-toolbox
```

Note: Rider requires a license (free for students/non-commercial use).

### Generate compile_commands.json for clangd

When building projects with CMake:

```bash
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DCMAKE_CXX_COMPILER=clang++ ..
ln -s build/compile_commands.json .
```

### Optional: clang-format config

Create `~/.clang-format`:

```yaml
BasedOnStyle: LLVM
IndentWidth: 4
```

### Optional: clang-tidy config

Create `~/.clang-tidy`:

```yaml
Checks: '*,-llvmlibc-*,-fuchsia-*,-google-*,-zircon-*,-abseil-*'
WarningsAsErrors: ''
```

---

## Part 11: Unreal Engine (Bare Metal Only)

**Requirements:**
- 340+ GB disk space (190 GB final install)
- 32 GB RAM recommended (16 GB minimum)
- Several hours build time

### Link Epic Games to GitHub

1. Create/login at epicgames.com
2. Go to Account → Connected Accounts
3. Link your GitHub account
4. Accept the EpicGames GitHub organization invite

### Clone and build

```bash
git clone https://github.com/EpicGames/UnrealEngine.git
cd UnrealEngine
./Setup.sh
./GenerateProjectFiles.sh
./Engine/Build/BatchFiles/Linux/Build.sh UnrealEditor Linux Development -Progress
```

### VS Code integration

Generate project files with VS Code support:

```bash
./GenerateProjectFiles.sh -vscode
```

This creates `.vscode/` folder and `compile_commands.json` for clangd.

### Build task (`.vscode/tasks.json`)

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Build UE Editor (Development)",
            "type": "shell",
            "command": "./Engine/Build/BatchFiles/Linux/Build.sh",
            "args": [
                "UnrealEditor",
                "Linux",
                "Development",
                "-Progress"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "problemMatcher": "$gcc",
            "presentation": {
                "reveal": "always",
                "panel": "new"
            }
        },
        {
            "label": "Build UE Editor (Debug)",
            "type": "shell",
            "command": "./Engine/Build/BatchFiles/Linux/Build.sh",
            "args": [
                "UnrealEditor",
                "Linux",
                "Debug",
                "-Progress"
            ],
            "group": "build",
            "problemMatcher": "$gcc"
        },
        {
            "label": "Generate Project Files",
            "type": "shell",
            "command": "./GenerateProjectFiles.sh",
            "args": ["-vscode"],
            "group": "build",
            "problemMatcher": []
        }
    ]
}
```

**Build configurations:**
- `Development` - for regular development (recommended, default)
- `Debug` - with full debug symbols (slower, much larger)

Build with `Ctrl + Shift + B` or open command palette (`Ctrl + Shift + P`) → "Tasks: Run Task".

### VS Code debugging

Install the **CodeLLDB** extension in VS Code.

Create `.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug UE Editor",
            "type": "lldb",
            "request": "launch",
            "program": "${workspaceFolder}/Engine/Binaries/Linux/UnrealEditor",
            "args": ["/path/to/YourProject.uproject"],
            "cwd": "${workspaceFolder}",
            "env": {
                "LD_LIBRARY_PATH": "${workspaceFolder}/Engine/Binaries/Linux"
            }
        },
        {
            "name": "Attach to UE",
            "type": "lldb",
            "request": "attach",
            "program": "${workspaceFolder}/Engine/Binaries/Linux/UnrealEditor",
            "pid": "${command:pickProcess}"
        }
    ]
}
```

Build with debug symbols:

```bash
./Engine/Build/BatchFiles/Linux/Build.sh UnrealEditor Linux Debug -Progress
```

Note: Debug builds are much slower and larger than Development builds.

---

## Part 12: AUR and Google Chrome

Chrome isn't in the official repos — you need the AUR (Arch User Repository).

### Install yay (AUR helper)

```bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ..
rm -rf yay
```

### Install Google Chrome

```bash
yay -S google-chrome
```

Launch with `google-chrome-stable` or find it in wofi.

### Install Zoom

```bash
yay -S zoom
```

### Install Slack

```bash
yay -S slack-desktop
```

---

## Part 13: Other Useful Packages

```bash
sudo pacman -S firefox thunar discord
```

- `firefox` — web browser
- `thunar` — file manager
- `discord` — voice/text chat

### Discord on Wayland

Discord is an Electron app and needs flags for native Wayland support. Create a desktop entry override:

```bash
mkdir -p ~/.local/share/applications
cat > ~/.local/share/applications/discord.desktop << 'EOF'
[Desktop Entry]
Name=Discord
StartupWMClass=discord
Comment=All-in-one voice and text chat for gamers
GenericName=Internet Messenger
Exec=/usr/bin/discord --enable-features=UseOzonePlatform --ozone-platform=wayland
Icon=discord
Type=Application
Categories=Network;InstantMessaging;
EOF
```

This makes Discord run natively on Wayland instead of through XWayland.

---

## Cheat Sheet: Hyprland

| Shortcut | Action |
|----------|--------|
| `Super + Q` | Open terminal |
| `Super + C` | Close window |
| `Super + M` | Exit Hyprland |
| `Super + R` | Open app launcher (wofi) |
| `Super + V` | Toggle floating |
| `Super + F` | Toggle fullscreen |
| `Super + Arrow keys` | Move focus |
| `Super + 1-9` | Switch to workspace |
| `Super + Shift + 1-9` | Move window to workspace |
| `Super + Scroll` | Cycle workspaces |
| `Super + Left drag` | Move window |
| `Super + Right drag` | Resize window |
| `Super + B` | Preselect below (next window opens underneath) |
| `Super + N` | Preselect right (next window opens beside) |
| `Super + G` | Toggle window group (tabbed stacking) |
| `Super + Shift + Arrow` | Move window that way, into or out of a group |
| `Super + Tab` | Next tab in group |
| `Super + Shift + Tab` | Previous tab in group |

---

## Cheat Sheet: tmux

Install: `sudo pacman -S tmux`

Default prefix: `Ctrl + B`

| Shortcut | Action |
|----------|--------|
| `tmux` | Start new session |
| `tmux attach` | Reattach to session |
| `Ctrl+B, c` | New window |
| `Ctrl+B, n` | Next window |
| `Ctrl+B, p` | Previous window |
| `Ctrl+B, %` | Vertical split |
| `Ctrl+B, "` | Horizontal split |
| `Ctrl+B, Arrow keys` | Move between panes |
| `Ctrl+B, d` | Detach session |
| `Ctrl+B, x` | Kill pane |
| `Ctrl+B, &` | Kill window |
| `Ctrl+B, [` | Scroll mode (q to exit) |
| `Ctrl+B, z` | Toggle pane zoom |

---

## Cheat Sheet: Neovim

Start: `nvim` or `nvim filename`

| Shortcut | Mode | Action |
|----------|------|--------|
| `i` | Normal | Enter insert mode |
| `Esc` | Insert | Return to normal mode |
| `h/j/k/l` | Normal | Move left/down/up/right |
| `w` | Normal | Next word |
| `b` | Normal | Previous word |
| `0` | Normal | Start of line |
| `$` | Normal | End of line |
| `gg` | Normal | Start of file |
| `G` | Normal | End of file |
| `dd` | Normal | Delete line |
| `yy` | Normal | Yank (copy) line |
| `p` | Normal | Paste |
| `u` | Normal | Undo |
| `Ctrl+R` | Normal | Redo |
| `/pattern` | Normal | Search |
| `n` | Normal | Next search result |
| `N` | Normal | Previous search result |
| `:w` | Command | Save |
| `:q` | Command | Quit |
| `:wq` | Command | Save and quit |
| `:q!` | Command | Quit without saving |
| `v` | Normal | Visual mode (select) |
| `V` | Normal | Visual line mode |
| `Ctrl+V` | Normal | Visual block mode |

---

## Cheat Sheet: Foot Terminal

| Shortcut | Action |
|----------|--------|
| `Ctrl + Shift + C` | Copy |
| `Ctrl + Shift + V` | Paste |
| `Ctrl + Shift + +` | Increase font size |
| `Ctrl + Shift + -` | Decrease font size |
| `Ctrl + Shift + 0` | Reset font size |
| `Middle click` | Paste selection |

---

## VirtualBox Tips (VirtualBox only)

Skip this section on bare metal.

**Enable clipboard sharing:**
- Devices → Shared Clipboard → Bidirectional

**Resize VM display:**
- View → Auto-resize Guest Display
- Or: `Right Ctrl + F` for fullscreen

**If display doesn't auto-resize:**
```bash
sudo systemctl enable --now vboxservice
reboot
```

**Expand disk space:**

1. Shut down the VM
2. In VirtualBox: File → Tools → Virtual Media Manager
3. Select your Arch VDI
4. Drag the Size slider to your desired size
5. Click Apply
6. Boot the VM and resize the partition:

```bash
lsblk                           # Check partition layout
sudo pacman -S parted
sudo parted /dev/sda
# In parted:
print                           # See partitions
resizepart 2 100%              # Replace 2 with your root partition number
quit
```

Then resize the filesystem:

```bash
# For ext4:
sudo resize2fs /dev/sda2

# For btrfs:
sudo btrfs filesystem resize max /
```

---

## Quick Reference: Package Summary

```bash
# All official packages in one command
sudo pacman -S \
    hyprland foot wofi waybar hyprpaper hypridle hyprlock brightnessctl asciiquarium mako \
    xdg-desktop-portal-hyprland xdg-desktop-portal-gtk hyprpolkitagent qt5-wayland qt6-wayland \
    otf-font-awesome ttf-nerd-fonts-symbols ttf-jetbrains-mono-nerd noto-fonts \
    wl-clipboard grim slurp wf-recorder \
    pipewire pipewire-pulse wireplumber pamixer pavucontrol \
    bluez bluez-utils blueman \
    git base-devel cmake make neovim tmux openssh starship \
    llvm clang \
    code firefox thunar discord

# Enable audio (run as your user, not root)
systemctl --user enable --now pipewire pipewire-pulse wireplumber

# Enable Bluetooth
sudo systemctl enable --now bluetooth

# VirtualBox only
sudo pacman -S virtualbox-guest-utils
sudo systemctl enable --now vboxservice

# Claude Code (native installer, auto-updates)
curl -fsSL https://claude.ai/install.sh | bash

# AUR packages (after installing yay)
yay -S google-chrome zoom slack-desktop p4 p4v rider
# Or for JetBrains Toolbox:
# yay -S jetbrains-toolbox
```

---

## Bare Metal Differences

When installing on real hardware instead of VirtualBox, make these changes:

### Skip these packages

```bash
# Don't need these
virtualbox-guest-utils
```

### Remove the VirtualBox cursor setting

Delete the `cursor = { no_hardware_cursors = 1 }` block from `hyprland.lua`. The default (`2`, auto) picks the right cursor mode on real hardware, NVIDIA included.

### Install GPU drivers instead

**AMD:**
```bash
sudo pacman -S mesa vulkan-radeon
```

Video decode (VA-API) is built into `mesa`; the separate `libva-mesa-driver` package was merged into it.

**NVIDIA:**

Arch's main driver now uses NVIDIA's open kernel modules and only supports Turing (RTX 20 / GTX 16 series) and newer. The old closed `nvidia` package is gone.

```bash
# Turing (RTX 20 / GTX 16) and newer
sudo pacman -S nvidia-open nvidia-utils nvidia-settings
# On linux-lts or several kernels, use nvidia-open-dkms (plus the matching *-headers) instead of nvidia-open

# Maxwell, Pascal, Volta (GTX 900 / GTX 10 series): legacy driver from the AUR
# yay -S nvidia-580xx-dkms nvidia-580xx-utils

# Brand-new GPUs the repo driver doesn't support yet
# yay -S nvidia-open-beta
```

Vulkan support (needed for UE). `lib32-*` packages need the multilib repo (see Part 2):

```bash
sudo pacman -S vulkan-icd-loader lib32-vulkan-icd-loader
```

DRM kernel mode setting (`nvidia_drm.modeset=1`) is on by default since driver 560, so no bootloader edits are needed. Verify after reboot:

```bash
cat /sys/module/nvidia_drm/parameters/modeset   # should print Y
```

Early loading (optional, helps if the display comes up late or flickers at boot). Edit `/etc/mkinitcpio.conf`:

```
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

`nvidia-utils` already blacklists nouveau. To also keep it out of the initramfs, remove `kms` from the `HOOKS` array in the same file. Then regenerate the initramfs:

```bash
sudo mkinitcpio -P
```

Reboot after all changes. No Hyprland environment variables are needed on current drivers. Older guides set `GBM_BACKEND`, `__GLX_VENDOR_LIBRARY_NAME` and `WLR_NO_HARDWARE_CURSORS`; drop them.

Optional hardware video decode (per the Arch wiki, it can draw more power than CPU decoding):

```bash
sudo pacman -S libva-nvidia-driver
```

```lua
-- hyprland.lua
hl.env("LIBVA_DRIVER_NAME", "nvidia")
```

**NVIDIA tips:**
- If the cursor flickers or vanishes, force a software cursor: `hl.config({ cursor = { no_hardware_cursors = 1 } })`
- Chrome/Electron apps may need `--ozone-platform=wayland` flag
- Use `nvidia-settings` to troubleshoot screen tearing
- Hyprland's explicit sync helps with NVIDIA — update Hyprland regularly

**Intel:**
```bash
sudo pacman -S mesa vulkan-intel intel-media-driver
```

### Monitor configuration

Replace the VirtualBox monitor line with your actual displays:

```bash
# Check connected monitors (including disabled ones)
hyprctl monitors all
```

Example for real monitors:
```lua
hl.monitor({ output = "DP-1",     mode = "2560x1440@144", position = "0x0",    scale = 1 })
hl.monitor({ output = "HDMI-A-1", mode = "1920x1080@60",  position = "2560x0", scale = 1 })
```

### Terminal

Kitty should work on bare metal (GPU acceleration available):
```bash
sudo pacman -S kitty
```

Change in `hyprland.lua`:
```lua
local terminal = "kitty"
```

### Clipboard

Should work without issues. If not:
```bash
sudo pacman -S wl-clipboard
```

### Audio

Same setup, but might need firmware:
```bash
sudo pacman -S sof-firmware alsa-firmware
```

---

## Troubleshooting

**Black screen with cursor after starting Hyprland:**
- This is normal — press `Super + Q` to open terminal

**No network after install:**
- Check NetworkManager: `sudo systemctl enable --now NetworkManager`
- Or use systemd-networkd (see Part 3)

**Hyprland crashes in VirtualBox:**
- Ensure 3D acceleration is enabled. Hyprland won't run without it, and the old `WLR_RENDERER_ALLOW_SOFTWARE` workaround no longer does anything

**Red config error banner after an update:**
- Hyprland 0.53+ rejects `windowrulev2` and the old `windowrule` syntax
- Hyprland 0.56 warns that `hyprland.conf` is deprecated (removed in 0.57)
- Fix both by moving to `hyprland.lua` (see Part 5)

**Waybar shows weird symbols:**
- Install font-awesome and nerd-fonts packages

**Waybar not showing workspaces:**
- Create custom waybar config (see Part 9)

**Can't paste into terminal:**
- Use `Ctrl + Shift + V` (not `Ctrl + V`)
- Enable clipboard sharing in VirtualBox

**No wallpaper / hyprpaper exits at startup:**
- hyprpaper 0.8+ refuses to start with an old-style config (`preload = ...`, `wallpaper = monitor,path`)
- Use the `wallpaper { }` block from Part 8
- Run `hyprpaper` in a terminal to see the config error
