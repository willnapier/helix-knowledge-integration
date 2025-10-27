# Installation Guide

## Prerequisites Check

Before installing, verify you have:

```bash
# Check Helix is installed
hx --version

# Check Nushell is installed
nu --version

# Check Rust tools are installed
rg --version
fd --version
sk --version

# Check nushell-knowledge-tools is installed
which fsl  # Should show path to function
```

If any are missing, see main README prerequisites section.

## Clean Installation

For users **without** existing Helix configuration:

### 1. Clone Repository

```bash
cd ~
git clone https://github.com/willnapier/helix-knowledge-integration.git
cd helix-knowledge-integration
```

### 2. Install Configuration

```bash
# Create Helix config directory if it doesn't exist
mkdir -p ~/.config/helix

# Copy configuration
cp config.toml ~/.config/helix/config.toml
```

### 3. Install Scripts

```bash
# Copy scripts to PATH
cp scripts/* ~/.local/bin/

# Make executable
chmod +x ~/.local/bin/hx-wiki ~/.local/bin/hx-open-system

# Verify installation
ls -la ~/.local/bin/hx-*
```

### 4. Set Environment Variable

Add to your shell configuration (`~/.zshrc`, `~/.bashrc`, or Nushell config):

```bash
export FORGE="/path/to/your/vault"
```

Reload shell or source the file:
```bash
source ~/.zshrc  # or ~/.bashrc
```

### 5. Test Installation

```bash
# Navigate to your vault
cd $FORGE

# Open a note
hx your-note.md

# Try keybindings:
# - Space+l should paste from clipboard
# - Space+m should save file
```

## Merging with Existing Configuration

For users with existing Helix `config.toml`:

### 1. Backup Current Config

```bash
cp ~/.config/helix/config.toml ~/.config/helix/config.toml.backup
```

### 2. Review This Repository's Config

```bash
cd helix-knowledge-integration
cat config.toml
```

### 3. Merge Keybindings

Open your existing `~/.config/helix/config.toml` and add:

```toml
[keys.normal.space]
# Add these lines (adjust keybindings if they conflict)
w = [
    "extend_to_line_bounds",
    ":pipe-to hx-wiki",
    ":sh sleep 0.1",
    ":buffer-close! /tmp/helix-current-link.md",
    ":open /tmp/helix-current-link.md"
]
l = ":insert-output pbpaste"  # or wl-paste/xclip on Linux
o = ["extend_to_line_bounds", ":pipe-to hx-open-system"]
b = [":write-all", "goto_previous_buffer", ":sh sleep 0.5", ":reload-all"]
m = ":write"
M = ":write-all"
q = [":write", ":buffer-previous"]
Q = ":quit!"
```

**Conflict Resolution:**

If you already use these keys, choose different ones:
```toml
# Example: Use Space+n for navigation instead of Space+w
n = [
    "extend_to_line_bounds",
    ":pipe-to hx-wiki",
    ":sh sleep 0.1",
    ":buffer-close! /tmp/helix-current-link.md",
    ":open /tmp/helix-current-link.md"
]
```

### 4. Install Scripts

```bash
# From helix-knowledge-integration directory
cp scripts/* ~/.local/bin/
chmod +x ~/.local/bin/hx-*
```

### 5. Test Merged Configuration

```bash
# Verify config is valid
hx --health

# Test in your vault
cd $FORGE
hx some-note.md
```

## Platform-Specific Setup

### macOS

Everything should work out of the box:
- Clipboard: `pbpaste` (built-in)
- File opener: `open` (built-in)

### Linux (Wayland)

Install clipboard tools:
```bash
sudo apt install wl-clipboard
```

Update `config.toml` clipboard commands:
```toml
[keys.normal.space]
l = ":insert-output wl-paste"
```

### Linux (X11)

Install clipboard tools:
```bash
sudo apt install xclip
```

Update `config.toml`:
```toml
[keys.normal.space]
l = ":insert-output xclip -o"
```

## Verification Steps

### 1. Check Scripts in PATH

```bash
which hx-wiki
which hx-open-system

# Should show: /Users/you/.local/bin/hx-wiki (or similar)
```

### 2. Check Environment Variable

```bash
echo $FORGE

# Should print your vault path
```

### 3. Check Helix Configuration

```bash
# Open Helix
hx

# Type: :config-open
# Verify knowledge management keybindings are present
```

### 4. Test Wiki Navigation

```bash
cd $FORGE
hx some-note.md
```

Add a wiki link: `[[Test Note]]`

Place cursor on line, press `Space+w`:
- Should create file if it doesn't exist
- Should open file if it exists

### 5. Test Link Insertion

In terminal:
```bash
fsl  # Search for note
```

In Helix, press `Space+l`:
- Should paste `[[Note Name]]`

## Troubleshooting Installation

### Scripts show "Permission denied"

```bash
chmod +x ~/.local/bin/hx-*
```

### "Command not found: hx-wiki"

Check PATH includes `~/.local/bin`:
```bash
echo $PATH | grep ".local/bin"
```

If not, add to shell config:
```bash
# ~/.zshrc or ~/.bashrc
export PATH="$HOME/.local/bin:$PATH"
```

### Config changes not taking effect

Reload Helix configuration:
- In Helix: `:config-reload`
- Or restart Helix

### "FORGE not set" errors

Add to shell config and reload:
```bash
export FORGE="/path/to/vault"
source ~/.zshrc  # or ~/.bashrc
```

## Uninstallation

To remove this integration:

```bash
# Remove scripts
rm ~/.local/bin/hx-wiki ~/.local/bin/hx-open-system

# Restore backup config (if you merged)
cp ~/.config/helix/config.toml.backup ~/.config/helix/config.toml

# Or remove entire config (if clean install)
rm ~/.config/helix/config.toml
```

## Next Steps

After installation:
1. Read [keybindings.md](keybindings.md) for complete keybinding reference
2. Review [customization.md](customization.md) to adapt to your workflow
3. Check [troubleshooting.md](troubleshooting.md) if you encounter issues
