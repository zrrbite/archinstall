# Remote UE Build Workflow

Cross-compile Unreal Engine for Linux on a Windows workstation, run the editor natively on Arch Linux.

```
[Arch NUC]                         [Windows Workstation]
    │                                     │
 thin client ◄──── SSH/rsync ────► UE source + cross-compile
    │                                     │
 run Linux editor                    Threadripper does heavy lifting
```

---

## Part 1: Enable SSH on Windows

On your **Windows machine**, open **PowerShell as Administrator**:

### Install and start OpenSSH Server

```powershell
# Install OpenSSH Server
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# Start and enable the service
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic

# Verify it's running
Get-Service sshd
```

### Allow through firewall

```powershell
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

### Get your IP address

```powershell
ipconfig
```

Look for `IPv4 Address` (e.g., `192.168.1.100`).

---

## Part 2: Test SSH from Arch

On your **Arch machine**:

### Test connection

```bash
ssh your-windows-username@192.168.1.100
```

Replace with your actual Windows username and IP.

### Set up SSH config for convenience

```bash
nano ~/.ssh/config
```

Add:

```
Host workstation
    HostName 192.168.1.100
    User your-windows-username
```

Now you can just use: `ssh workstation`

### Copy SSH key (optional, for passwordless login)

```bash
ssh-copy-id workstation
```

Or manually copy your public key to Windows:

```bash
cat ~/.ssh/id_ed25519.pub | ssh workstation "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

---

## Part 3: VS Code Remote-SSH

On your **Arch machine**:

1. Open VS Code
2. Install the **Remote - SSH** extension
3. Press `Ctrl+Shift+P` → "Remote-SSH: Connect to Host"
4. Select `workstation` (or enter `user@ip`)
5. VS Code opens a new window connected to Windows

You can now edit files on Windows, run terminal commands, and build—all from Arch.

---

## Part 4: Install UE Linux Cross-Compile Toolchain (Windows)

On your **Windows machine** (via SSH or directly):

### Download the toolchain

Epic provides a clang-based Linux toolchain. Download from:
- UE GitHub releases, or
- Inside UE: `Engine/Extras/ThirdPartyNotUE/SDKs/HostLinux/`

### Set environment variable

```powershell
# In System Environment Variables, add:
LINUX_MULTIARCH_ROOT=C:\UnrealToolchains\v22_clang-16.0.6-centos7
```

Or in PowerShell:

```powershell
[System.Environment]::SetEnvironmentVariable("LINUX_MULTIARCH_ROOT", "C:\UnrealToolchains\v22_clang-16.0.6-centos7", "User")
```

### Verify in UE

Regenerate project files:

```cmd
GenerateProjectFiles.bat
```

---

## Part 5: Cross-Compile UE for Linux (Windows)

On your **Windows machine**:

```cmd
cd C:\Path\To\UnrealEngine
Engine\Build\BatchFiles\Build.bat UnrealEditor Linux Development
```

This uses the Threadripper's power to compile Linux binaries.

---

## Part 6: Sync Binaries to Arch

On your **Arch machine**:

```bash
# Sync the Linux binaries from Windows to Arch
rsync -avz --progress workstation:/c/Path/To/UnrealEngine/Engine/Binaries/Linux/ ~/dev/ue/Engine/Binaries/Linux/
```

Or use a script:

```bash
#!/bin/bash
REMOTE="workstation:/c/UE/Engine/Binaries/Linux/"
LOCAL="~/dev/ue/Engine/Binaries/Linux/"
rsync -avz --progress "$REMOTE" "$LOCAL"
```

---

## Part 7: Run Linux UE Editor (Arch)

On your **Arch machine**:

```bash
cd ~/dev/ue
./Engine/Binaries/Linux/UnrealEditor
```

---

## Part 8: Debugging (Arch)

### Option A: VS Code + CodeLLDB

1. Install **CodeLLDB** extension in VS Code
2. Create `.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug UE Editor",
            "type": "lldb",
            "request": "launch",
            "program": "${workspaceFolder}/Engine/Binaries/Linux/UnrealEditor",
            "args": [],
            "cwd": "${workspaceFolder}",
            "env": {
                "LD_LIBRARY_PATH": "${workspaceFolder}/Engine/Binaries/Linux"
            }
        }
    ]
}
```

### Option B: JetBrains Rider

Rider has excellent Linux remote debugging support and UE integration.

```bash
yay -S rider
```

---

## Troubleshooting

### SSH connection refused

- Check Windows firewall: `Get-NetFirewallRule -Name sshd`
- Check service is running: `Get-Service sshd`

### Permission denied (publickey)

- Ensure your public key is in `C:\Users\YourUser\.ssh\authorized_keys` on Windows
- Check permissions on the authorized_keys file

### Cross-compile toolchain not found

- Verify `LINUX_MULTIARCH_ROOT` environment variable is set
- Restart terminal/VS Code after setting env vars

---

## Quick Reference

```bash
# SSH to workstation
ssh workstation

# Build UE for Linux (on Windows via SSH)
ssh workstation "cd /c/UE && Engine/Build/BatchFiles/Build.bat UnrealEditor Linux Development"

# Sync binaries
rsync -avz workstation:/c/UE/Engine/Binaries/Linux/ ~/dev/ue/Engine/Binaries/Linux/

# Run editor
~/dev/ue/Engine/Binaries/Linux/UnrealEditor
```
