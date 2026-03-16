# Claude Code Installation (Arch Linux)

## Standalone installer (recommended)

The standalone installer is the preferred method. It auto-updates and
doesn't depend on Node.js/npm.

```bash
curl -fsSL https://claude.ai/install.sh | sh
```

This installs to `~/.local/share/claude/versions/` and symlinks
`~/.local/bin/claude`.

### PATH setup

Make sure `~/.local/bin` is on your PATH. Add to `~/.bashrc` or
`~/.zshrc`:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Then `source ~/.bashrc` or restart your shell.

### Verify

```bash
which claude        # should be ~/.local/bin/claude
claude --version
```

## Removing a conflicting npm install

If you previously installed via npm, you'll have two installs and
Claude Code will nag you to switch. To clean up:

```bash
# Remove the npm global install
npm uninstall -g @anthropic-ai/claude-code

# Remove any stale standalone symlink/versions
rm -f ~/.local/bin/claude
rm -rf ~/.local/share/claude/versions

# Reinstall standalone
curl -fsSL https://claude.ai/install.sh | sh
```

### Why not npm?

- npm install requires Node.js as a dependency
- Updates are manual (`npm update -g @anthropic-ai/claude-code`)
- Installs to `/usr/lib` which can conflict with `~/.local/bin`
- The standalone installer auto-updates itself

## Updating

The standalone installer handles updates automatically. If you need to
force an update:

```bash
claude update
```
