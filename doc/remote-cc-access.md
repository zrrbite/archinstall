# Remote Claude Code Access

Access Claude Code sessions on your Arch Linux machine from your phone
or laptop, from anywhere.

---

## Architecture

```
Phone (Termius)  ──┐
                   ├── Tailscale VPN ── SSH ── tmux ── Claude Code
Laptop (terminal) ─┘
```

- **Tailscale**: Zero-config VPN mesh. No port forwarding needed.
- **tmux**: Persistent terminal sessions that survive disconnects.
- **SSH**: Encrypted shell access over Tailscale.

---

## Step 1: Install Tailscale (Arch box)

```bash
sudo pacman -S tailscale
sudo systemctl enable --now tailscaled
sudo tailscale up
```

Follow the auth URL it prints. Your machine gets a Tailscale IP
(e.g. `100.64.x.x`) and a magic DNS name (e.g. `archbox`).

Check it:

```bash
tailscale status
```

## Step 2: Install Tailscale on your devices

- **Phone**: Install Tailscale from App Store / Play Store. Log in
  with the same account.
- **Laptop**: Install Tailscale for your OS. Same account.

All devices on the same tailnet can reach each other directly.

## Step 3: SSH setup

### Enable SSH on your Arch box

```bash
sudo systemctl enable --now sshd
```

### Key-based auth (recommended)

Copy your key from your laptop:

```bash
ssh-copy-id you@100.64.x.x
```

For Termius on phone: generate a key pair in Termius and add the
public key to `~/.ssh/authorized_keys` on your Arch box.

### Test it

From phone or laptop:

```bash
ssh you@archbox    # magic DNS name via Tailscale
```

## Step 4: tmux for persistent sessions

### Start a named session before leaving

```bash
tmux new -s claude
claude
```

Detach without killing it: `Ctrl-a d`

The keys in this guide use the prefix from the dotfiles' `~/.tmux.conf`,
`Ctrl-a`. On stock tmux, use `Ctrl-b` instead.

### Reattach from anywhere

```bash
ssh you@archbox
tmux attach -t claude
```

You're back exactly where you left off — full conversation history,
working directory, everything.

### Useful tmux commands

| Action | Keys |
|---|---|
| Detach | `Ctrl-a d` |
| List sessions | `tmux ls` |
| Attach to session | `tmux attach -t claude` |
| New window | `Ctrl-a c` |
| Switch window | `Ctrl-a n` / `Ctrl-a p` |
| Split side by side | `Ctrl-a \|` |
| Split stacked | `Ctrl-a -` |
| Navigate panes | `Ctrl-a h/j/k/l` or `Ctrl-a arrow` |
| Kill session | `tmux kill-session -t claude` |

## Step 5: Prevent suspend during remote sessions

Your hypridle suspends the system after 30 minutes of idle. This kills
SSH connections. Three options:

### Option A: Inhibit sleep for a session

Start the session with the inhibitor running *inside* it, around its shell:

```bash
tmux new -s claude 'systemd-inhibit --what=sleep --who=claude --why="Remote CC session" $SHELL'
```

The machine won't suspend for as long as that shell runs, including after
you detach. Exit the shell or kill the session and normal suspend comes
back. hypridle still locks the screen and turns it off; only the suspend is
blocked (it runs `systemctl suspend`, which refuses while the inhibitor is
held).

Don't put `systemd-inhibit` in front of `tmux`
(`systemd-inhibit ... tmux new -s claude`). That holds the lock only as long
as the tmux *client* runs, and the client exits when you detach, which is
exactly when you leave.

### Option B: Caffeine toggle

Add a script to toggle sleep inhibition:

```bash
#!/bin/bash
# ~/.local/bin/caffeine-toggle
if pgrep -f "systemd-inhibit.*caffeine" > /dev/null; then
    pkill -f "systemd-inhibit.*caffeine"
    notify-send "Caffeine OFF" "System will suspend normally"
else
    systemd-inhibit --what=idle:sleep --who="caffeine" --why="Stay awake" sleep infinity &
    notify-send "Caffeine ON" "Suspend inhibited"
fi
```

Make it executable (`chmod +x ~/.local/bin/caffeine-toggle`) and bind it
in Hyprland. `Super+Shift+I` is free in the dotfiles (`Super+Shift+C` is
their screenshot-to-clipboard):

```lua
-- hyprland.lua
hl.bind(mainMod .. " + SHIFT + I", hl.dsp.exec_cmd("~/.local/bin/caffeine-toggle"))
```

### Option C: Wake-on-LAN

If the machine does suspend, wake it remotely. Requires WoL enabled
in BIOS and on your network interface:

```bash
# On your Arch box (one-time setup; NetworkManager keeps it across reboots)
nmcli connection show                    # find the wired connection's name
sudo nmcli connection modify "Wired connection 1" 802-3-ethernet.wake-on-lan magic
sudo nmcli connection up "Wired connection 1"
ip link                                  # the MAC address is after link/ether

# From another device on the same LAN (or via Tailscale subnet router)
wakeonlan AA:BB:CC:DD:EE:FF              # on the Mac: brew install wakeonlan
```

(`ethtool -s <interface> wol g` turns it on too, but only until the next
reboot.)

Note: WoL only works over LAN, not over Tailscale directly. You'd need
a device on the same LAN (e.g. a Raspberry Pi) to send the WoL packet,
or a Tailscale subnet router.

---

## Phone workflow (Termius)

1. Open Termius
2. Connect to saved host `archbox` (Tailscale IP)
3. `tmux attach -t claude` (or `tmux ls` to see sessions)
4. Interact with Claude Code
5. When done, `Ctrl-a d` to detach (don't exit — keeps session alive)

### Termius tips

- Enable "Keep Alive" in connection settings to prevent drops
- Set up a snippet for `tmux attach -t claude` for one-tap access
- Use landscape mode for more screen space
- Termius supports key auth — set it up to skip password prompts

---

## Laptop workflow

```bash
ssh archbox
tmux attach -t claude
# ... work ...
# Ctrl-a d to detach when done
```

---

## Quick start

```bash
# On your Arch box: a detached session that blocks suspend while it runs
tmux new -d -s claude 'systemd-inhibit --what=sleep --who=claude --why="Remote CC session" $SHELL'
tmux send-keys -t claude "claude" Enter

# From anywhere
ssh archbox
tmux attach -t claude
```

---

## Troubleshooting

### Can't connect via Tailscale

```bash
tailscale status        # check if both devices are online
tailscale ping archbox  # test connectivity
```

### tmux session not found

```bash
tmux ls                 # list all sessions
```

If empty, no session is running. Start one:

```bash
tmux new -s claude
```

### SSH connection drops on phone

- Enable "Keep Alive" in Termius (sends heartbeat packets)
- Or add to `~/.ssh/config` on the client:
  ```
  Host archbox
      ServerAliveInterval 60
      ServerAliveCountMax 3
  ```

### Machine suspended, can't connect

Either wait until you're home, or use WoL if set up (see Option C).
The tmux session survives suspend — once the machine wakes, just
reconnect and `tmux attach`.
