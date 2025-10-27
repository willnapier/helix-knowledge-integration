# Helix Knowledge Integration

> **Complete Helix editor configuration for [nushell-knowledge-tools](https://github.com/willnapier/nushell-knowledge-tools)**

This is a **reference implementation** showing one complete way to integrate universal knowledge management tools with Helix editor. It demonstrates cursor-aware wiki navigation, smart file handling, and seamless clipboard integration.

## What This Provides

✅ **Wiki link navigation** - Jump between notes with `Space+w`
✅ **Smart file opening** - PDFs, images, and media open in appropriate apps
✅ **Link insertion** - Fuzzy-find and insert `[[wiki links]]` from clipboard
✅ **Auto-file creation** - New notes created with YAML frontmatter templates
✅ **Daily note support** - Date-based notes with automatic templates
✅ **Cross-platform** - Works on macOS and Linux with platform detection

## Prerequisites

### Required

1. **[nushell-knowledge-tools](https://github.com/willnapier/nushell-knowledge-tools)** - Universal CLI functions
   ```bash
   # Install first! These Helix integrations invoke those functions
   git clone https://github.com/willnapier/nushell-knowledge-tools.git
   cd nushell-knowledge-tools
   # Follow installation instructions in that repository
   ```

2. **[Helix Editor](https://helix-editor.com)** (tested with v25.07+)
   ```bash
   # macOS
   brew install helix

   # Linux
   # See https://docs.helix-editor.com/install.html
   ```

3. **[Nushell](https://nushell.sh)** - Required for scripts
   ```bash
   # macOS
   brew install nushell

   # Linux (Ubuntu/Debian)
   apt install nushell
   ```

4. **Modern Rust tools** - Used by scripts
   ```bash
   # macOS
   brew install ripgrep fd skim

   # Linux
   apt install ripgrep fd-find skim
   ```

### Environment Setup

Set your knowledge base directory:
```bash
# In ~/.zshrc, ~/.bashrc, or Nushell config
export FORGE="/path/to/your/vault"  # Use any name you prefer
```

**Note**: "FORGE" is just an example - you can use `VAULT`, `NOTES`, `KNOWLEDGE_BASE`, or any name you prefer. The scripts check `$env.FORGE` by default.

## Installation

### Option 1: Fresh Helix Installation

If you don't have an existing Helix configuration:

```bash
# Clone this repository
git clone https://github.com/willnapier/helix-knowledge-integration.git
cd helix-knowledge-integration

# Copy configuration
cp config.toml ~/.config/helix/config.toml

# Install scripts
cp scripts/* ~/.local/bin/
chmod +x ~/.local/bin/hx-*

# Verify installation
helix --health
```

### Option 2: Merge with Existing Configuration

If you already have a Helix config:

```bash
# Clone this repository
git clone https://github.com/willnapier/helix-knowledge-integration.git
cd helix-knowledge-integration

# Review the configuration
cat config.toml

# Manually merge keybindings into your ~/.config/helix/config.toml
# Focus on the [keys.normal.space] section

# Install scripts
cp scripts/* ~/.local/bin/
chmod +x ~/.local/bin/hx-*
```

See [docs/installation.md](docs/installation.md) for detailed merge instructions.

## Quick Start

### 1. Basic Wiki Navigation

Open Helix in your knowledge base:
```bash
cd /path/to/your/vault
hx some-note.md
```

Navigate wiki links:
1. Place cursor on line with `[[Another Note]]`
2. Press `Space+w`
3. Note opens in Helix (or system viewer for media)

### 2. Insert Links from Fuzzy Finder

Terminal workflow:
```bash
# In terminal (outside Helix)
fsl   # Fuzzy search → copies [[Note Name]] to clipboard
```

In Helix:
```
Space+l   # Paste the copied link
```

### 3. Open Files in System Viewer

Place cursor on line with file reference:
```
Space+o   # Opens PDF, image, video in appropriate app
```

## Keybindings Reference

### Core Knowledge Management

| Key | Action | Description |
|-----|--------|-------------|
| `Space+w` | Wiki Navigation | Opens `[[wiki links]]` intelligently |
| `Space+l` | Insert Link | Pastes link from clipboard (after `fsl`) |
| `Space+o` | System Open | Opens file in default system app |
| `Space+b` | Buffer Back | Save all → previous buffer → reload |
| `Space+m` | Save | Quick save current file |
| `Space+M` | Save All | Save all modified buffers |
| `Space+q` | Save & Close | Save and return to previous buffer |

### Advanced (Optional)

| Key | Action | Description |
|-----|--------|-------------|
| `Space+[` | Previous Day | Navigate to previous daily note |
| `Space+]` | Next Day | Navigate to next daily note |
| `g R` | Reload All | Refresh all buffers (after external changes) |

See [docs/keybindings.md](docs/keybindings.md) for complete reference.

## How It Works

### Wiki Link Navigation (`Space+w`)

**Helix → Script Flow:**
1. Helix sends current line to `hx-wiki` script via pipe
2. Script extracts first `[[wiki link]]` using regex
3. Script searches for file across knowledge base
4. Creates symlink at `/tmp/helix-current-link.md`
5. Helix opens the symlink

**Smart Handling:**
- `.md` files → Open in Helix
- Images/PDFs → Open in system viewer (Preview/xdg-open)
- Missing files → Create with YAML frontmatter template
- Date links (`[[2025-10-27]]`) → Daily note template

**Supported Link Formats:**
```markdown
[[Note Name]]              # Regular wiki link
[[~/Admin/Budget]]         # Explicit path (cross-directory)
[[2025-10-27]]            # Daily note (auto-template)
![[image.png]]            # Image embed
?[[Unresolved Link]]      # Marked as unresolved
```

### Link Insertion (`Space+l`)

**Terminal → Helix Flow:**
1. Run `fsl` in terminal (fuzzy file search)
2. Select note → copies `[[Note Name]]` to clipboard
3. In Helix, press `Space+l` to paste

**Platform Detection:**
- macOS: Uses `pbpaste`
- Linux (Wayland): Uses `wl-paste`
- Linux (X11): Uses `xclip -o`

Change in `config.toml` if using different clipboard tool.

### System File Opener (`Space+o`)

**Opens any file type with system default:**
- PDFs → PDF viewer
- Images → Image viewer
- Videos → Media player
- Spreadsheets → Excel/LibreOffice
- URLs → Web browser

## Integration with nushell-knowledge-tools

This repository provides **Helix-specific keybindings** that invoke **universal CLI functions** from nushell-knowledge-tools:

```
┌─────────────────────────────────────────┐
│  Universal Functions (nushell-tools)    │
│  - fsl, fcl, fsml (search & clipboard)  │
│  - fse, fce, fsme (search & open)       │
│  - cit, cil, ciz (citations)            │
└─────────────────────────────────────────┘
                    ▲
                    │ invoked by
                    │
┌─────────────────────────────────────────┐
│  Helix Integration (this repository)    │
│  - Space+w → hx-wiki script             │
│  - Space+l → paste from clipboard       │
│  - Space+o → hx-open-system script      │
└─────────────────────────────────────────┘
```

**Why this separation?**
- Universal functions work with **any editor**
- This repo shows **one complete Helix implementation**
- You could build similar integrations for Neovim, VS Code, Emacs

## Customization

### Change Keybindings

Edit `~/.config/helix/config.toml`:

```toml
[keys.normal.space]
# Change wiki navigation from Space+w to Space+n
n = [
    "extend_to_line_bounds",
    ":pipe-to hx-wiki",
    ":sh sleep 0.1",
    ":buffer-close! /tmp/helix-current-link.md",
    ":open /tmp/helix-current-link.md"
]
```

### Customize Templates

Edit `~/.local/bin/hx-wiki` to change YAML frontmatter:

```nushell
# Around line 243 - Regular note template
let content = $"---
tags:
  - your-custom-tag
date created: ($current_date) ($current_time)
custom_field: value
---
# ($clean_link)

Your custom template here
"
```

### Change Directory Structure

Edit `~/.local/bin/hx-wiki` to match your vault layout:

```nushell
# Around line 47 - Define paths
let vault = $env.FORGE  # Your main vault
let daily_dir = $"($vault)/journal/daily"  # Your daily notes location
```

See [docs/customization.md](docs/customization.md) for comprehensive guide.

## Troubleshooting

### "Command not found: hx-wiki"

Scripts not in PATH or not executable:
```bash
# Check if scripts exist
ls -la ~/.local/bin/hx-*

# Make executable
chmod +x ~/.local/bin/hx-*

# Verify PATH includes ~/.local/bin
echo $PATH
```

### Links not opening / "File not found"

Check environment variable:
```bash
# In terminal
echo $FORGE

# Should print your vault path
# If empty, add to shell config:
export FORGE="/path/to/your/vault"
```

### Clipboard paste not working on Linux

Install clipboard tools:
```bash
# Wayland
apt install wl-clipboard

# X11
apt install xclip
```

Update `config.toml`:
```toml
# For Wayland
l = ":insert-output wl-paste"

# For X11
l = ":insert-output xclip -o"
```

### Media files open in Helix instead of system viewer

Check file extension detection in `hx-wiki` script (line 175-195). Ensure your media extensions are in the match statement.

See [docs/troubleshooting.md](docs/troubleshooting.md) for more solutions.

## Contributing

Contributions welcome for:
- **Bug fixes** in scripts or configuration
- **Documentation improvements**
- **Cross-platform compatibility** enhancements
- **Template variations** (different frontmatter styles)

**Not needed:**
- Features that belong in nushell-knowledge-tools (universal functions)
- Extensive personal customizations (keep those in your dotfiles)

This repository aims to be a **clean reference implementation** that others can adapt.

## Related Projects

- **[nushell-knowledge-tools](https://github.com/willnapier/nushell-knowledge-tools)** - Universal CLI functions (required)
- **Your dotfiles** - Your personal configuration with experimental features

**Want to build integrations for other editors?**
- Use this as a reference for patterns
- Create `neovim-knowledge-integration`, `vscode-knowledge-integration`, etc.
- Focus on thin wrappers that invoke universal functions

## Philosophy

This integration follows the **Universal Tool Architecture**:

1. **Core logic** lives in universal CLI functions (nushell-knowledge-tools)
2. **Editor integrations** are thin wrappers with keybindings
3. **Users adapt** the integration to their preferences

**This is ONE way to integrate Helix with knowledge tools** - not THE way. Adapt to your workflow!

## License

MIT License - See [LICENSE](LICENSE) for details

---

**Need help?** Open an issue or check the [docs/](docs/) directory for detailed guides.

**Want to share your adaptations?** We'd love to see how you've customized this for your workflow!
