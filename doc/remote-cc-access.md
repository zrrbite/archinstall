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
tailscale up
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

Detach without killing it: `Ctrl-b d`

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
| Detach | `Ctrl-b d` |
| List sessions | `tmux ls` |
| Attach to session | `tmux attach -t claude` |
| New window | `Ctrl-b c` |
| Switch window | `Ctrl-b n` / `Ctrl-b p` |
| Split horizontal | `Ctrl-b "` |
| Split vertical | `Ctrl-b %` |
| Navigate panes | `Ctrl-b arrow` |
| Kill session | `tmux kill-session -t claude` |

## Step 5: Prevent suspend during remote sessions

Your hypridle suspends the system after 30 minutes of idle. This kills
SSH connections. Two options:

### Option A: Inhibit sleep for a session

Before starting a remote-accessible session:

```bash
systemd-inhibit --what=idle:sleep --who="claude" --why="Remote CC session" tmux new -s claude
```

This prevents suspend as long as the tmux session is running. When you
kill the session, normal suspend behavior resumes.

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

Make it executable and bind it in Hyprland:

```conf
# hyprland.conf
bind = $mainMod SHIFT, C, exec, ~/.local/bin/caffeine-toggle
```

### Option C: Wake-on-LAN

If the machine does suspend, wake it remotely. Requires WoL enabled
in BIOS and on your network interface:

```bash
# On your Arch box (one-time setup)
sudo ethtool -s enp0s31f6 wol g

# From another device on the same LAN (or via Tailscale subnet router)
wol AA:BB:CC:DD:EE:FF
```

Note: WoL only works over LAN, not over Tailscale directly. You'd need
a device on the same LAN (e.g. a Raspberry Pi) to send the WoL packet,
or a Tailscale subnet router.

---

## Phone workflow (Termius)

1. Open Termius
2. Connect to saved host `archbox` (Tailscale IP)
3. `tmux attach -t claude` (or `tmux ls` to see sessions)
4. Interact with Claude Code
5. When done, `Ctrl-b d` to detach (don't exit — keeps session alive)

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
# Ctrl-b d to detach when done
```

---

## Quick start

```bash
# On your Arch box — start a persistent CC session
systemd-inhibit --what=idle:sleep --who="claude" --why="Remote session" \
    tmux new -s claude -d
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
