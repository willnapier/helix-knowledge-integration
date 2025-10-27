# Customization Guide

This guide shows how to adapt the Helix knowledge integration to your specific workflow.

## Configuration Files

### Main Configuration
- **`~/.config/helix/config.toml`** - Helix keybindings
- **`~/.local/bin/hx-wiki`** - Wiki link navigation script
- **`~/.local/bin/hx-open-system`** - System file opener script

## Keybinding Customization

### Change Keybindings

Edit `~/.config/helix/config.toml`:

```toml
[keys.normal.space]
# Default: Space+w for wiki navigation
# Change to: Space+n
n = [
    "extend_to_line_bounds",
    ":pipe-to hx-wiki",
    ":sh sleep 0.1",
    ":buffer-close! /tmp/helix-current-link.md",
    ":open /tmp/helix-current-link.md"
]
```

### Add Additional Keybindings

```toml
[keys.normal.space]
# Add Space+W (capital) for wiki navigation in new pane (requires Zellij)
W = [
    ":pipe-to hx-wiki",
    ":sh sleep 0.1",
    ":sh zellij action new-pane --direction right",
    ":sh zellij action write-chars 'hx /tmp/helix-current-link.md'"
]
```

### Remove Optional Features

Comment out unwanted keybindings:

```toml
[keys.normal.space]
# Disable daily note navigation
# "[" = [...]
# "]" = [...]
```

## Template Customization

### Regular Note Template

Edit `~/.local/bin/hx-wiki` around line 243:

```nushell
let content = $"---
tags:
  - inbox  # Add default tag
  - zettelkasten
author: Your Name
date created: ($current_date) ($current_time)
date modified: ($current_date) ($current_time)
status: draft  # Add status field
---
# ($clean_link)

## Summary
<!-- One-sentence summary -->

## Content

## References

## Backlinks

"
```

### Daily Note Template

Edit `~/.local/bin/hx-wiki` around line 280:

```nushell
let content = $"---
tags:
  - daily
  - journal
date created: ($clean_link) ($current_time)
date modified: ($clean_link) ($current_time)

# Your custom metrics
mood:
energy_level:
focus_score:
---
# ($readable_date)

## Morning Routine
- [ ] Exercise
- [ ] Meditation
- [ ] Journaling

## Priorities
1.
2.
3.

## Daily Log

## Evening Reflection

## Links
- Previous: [[($prev_date)]]
- Next: [[($next_date)]]

## Backlinks

"
```

## Directory Structure Customization

### Change Vault Paths

Edit `~/.local/bin/hx-wiki` around line 47:

```nushell
# Default
let vault = $env.FORGE
let daily_dir = $"($vault)/daily"

# Custom structure
let vault = $env.FORGE
let daily_dir = $"($vault)/journal/daily-notes"  # Your structure
let projects_dir = $"($vault)/projects"
let areas_dir = $"($vault)/areas"
```

### Multi-Directory Support

Add additional search locations:

```nushell
# Around line 92 - Search across multiple directories
let existing_file = try {
    fd -t f $"^($clean_link).md$" $vault $daily_dir $projects_dir $areas_dir
    | lines
    | first
} catch {
    ""
}
```

## Platform-Specific Customization

### Clipboard Commands

Edit `config.toml` for your platform:

```toml
[keys.normal.space]
# macOS (default)
l = ":insert-output pbpaste"

# Linux (Wayland)
l = ":insert-output wl-paste"

# Linux (X11)
l = ":insert-output xclip -o -selection clipboard"

# Windows (WSL)
l = ":insert-output powershell.exe Get-Clipboard"
```

### File Opener

Edit `~/.local/bin/hx-wiki` around line 198:

```nushell
def open_file [file: string] {
    # Default
    if (sys | get host.name) == "Darwin" {
        ^open $file
    } else {
        ^xdg-open $file
    }

    # Custom for specific file types
    let extension = ($file | path parse | get extension)
    match $extension {
        "pdf" => { ^zathura $file }  # Custom PDF viewer
        "png" | "jpg" => { ^feh $file }  # Custom image viewer
        _ => {
            if (sys | get host.name) == "Darwin" {
                ^open $file
            } else {
                ^xdg-open $file
            }
        }
    }
}
```

## Link Format Customization

### Support Additional Link Formats

Edit `~/.local/bin/hx-wiki` around line 26:

```nushell
# Default: Supports [[link]], ![[embed]], ?[[unresolved]]
let first_link = try {
    $line | rg -o '[!?]?\[\[[^\]]+\]\]' | lines | first
} catch {
    ""
}

# Add support for Obsidian [[link|alias]] format
let first_link = try {
    $line | rg -o '[!?]?\[\[[^\]|]+(\|[^\]]+)?\]\]' | lines | first
} catch {
    ""
}

# Extract link and alias
let parts = ($first_link | str replace -r '[!?]*\[\[([^\]|]+)(\|([^\]]+))?\]\]' '$1|$3')
let link = ($parts | split row '|' | first)
let alias = ($parts | split row '|' | get 1?)
```

### Custom Link Patterns

Add support for hashtag links:

```nushell
# After wiki link extraction, check for hashtags
if ($first_link | is-empty) {
    # Try hashtag pattern
    let hashtag = try {
        $line | rg -o '#[a-zA-Z0-9_-]+' | lines | first | str replace '#' ''
    } catch {
        ""
    }

    if not ($hashtag | is-empty) {
        # Search for notes with this tag
        let found = try {
            rg -l $"tags:.*($hashtag)" $vault | lines | first
        } catch {
            ""
        }

        if not ($found | is-empty) {
            ln -sf $found $target_file
            return
        }
    }
}
```

## File Type Handling

### Add Custom File Types

Edit `~/.local/bin/hx-wiki` around line 210:

```nushell
# Default media extensions
let media_extensions = ["jpg", "jpeg", "png", "gif", "bmp", "webp", "svg", "pdf", "mp4", "mov", "avi", "mp3", "wav", "m4a"]

# Add more
let media_extensions = ["jpg", "jpeg", "png", "gif", "bmp", "webp", "svg", "tiff", "pdf", "mp4", "mov", "avi", "mkv", "webm", "mp3", "wav", "m4a", "flac", "ogg"]
```

### Custom Handlers for Specific Types

```nushell
def handle_existing_file [file: string, target_file: string] {
    let extension = ($file | path parse | get extension)

    match $extension {
        "md" | "txt" => { ln -sf $file $target_file }
        "pdf" => { ^zathura $file }  # Custom PDF viewer
        "png" | "jpg" => { ^feh $file }  # Custom image viewer
        "mp4" | "mkv" => { ^mpv $file }  # Custom video player
        "org" => {
            # Convert org to markdown and open
            ^pandoc -f org -t markdown $file | hx -
        }
        _ => { open_file $file }
    }
}
```

## Auto-Save Timing

Edit `config.toml` auto-save interval:

```toml
[editor]
# Default: 500ms (good for background link watchers)
idle-timeout = 500

# Longer interval for less aggressive auto-save
idle-timeout = 2000  # 2 seconds

# Shorter for instant saving
idle-timeout = 100  # 100ms
```

## Disabling Features

### Disable Auto-Save

```toml
[editor]
# Remove or set to false
# auto-save = true
auto-save = false
```

### Disable Backlinks Section

Edit templates in `hx-wiki` to remove:

```nushell
# Remove this line from templates
## Backlinks

```

## Advanced: Custom Actions

### Add Pre-Navigation Hook

Edit `hx-wiki` to run custom code before navigation:

```nushell
def main [] {
    # ... existing code ...

    # Custom pre-navigation hook
    if ($env.HX_WIKI_HOOK? | is-not-empty) {
        ^($env.HX_WIKI_HOOK) $clean_link
    }

    # Continue with normal navigation
    # ...
}
```

Usage:
```bash
export HX_WIKI_HOOK="~/.local/bin/my-custom-hook"
```

### Add Post-Creation Hook

```nushell
def create_new_file [file: string, clean_link: string, target_file: string] {
    # ... create file ...

    # Custom post-creation hook
    if ($env.HX_WIKI_POST_CREATE? | is-not-empty) {
        ^($env.HX_WIKI_POST_CREATE) $file
    }
}
```

## Testing Your Customizations

### Debug Mode

Enable debug logging:

```nushell
# In hx-wiki script
def main [] {
    # ... existing code ...

    # Add verbose logging
    $line | save --append /tmp/hx-wiki-debug.log
    $clean_link | save --append /tmp/hx-wiki-debug.log
    $file | save --append /tmp/hx-wiki-debug.log
}
```

View logs:
```bash
tail -f /tmp/hx-wiki-debug.log
```

### Test Individual Scripts

```bash
# Test hx-wiki directly
echo "[[Test Note]]" | ~/.local/bin/hx-wiki

# Check what was created
cat /tmp/helix-current-link.md
```

## Backup Your Customizations

After customizing:

```bash
# Backup config
cp ~/.config/helix/config.toml ~/helix-config-backup.toml

# Backup scripts
cp ~/.local/bin/hx-* ~/helix-scripts-backup/

# Or version control them
cd ~/.config/helix
git init
git add config.toml
git commit -m "My custom Helix knowledge config"
```

## Share Your Customizations

If you create useful customizations:
1. Document them clearly
2. Open an issue or PR in the repository
3. Help others benefit from your adaptations!

The beauty of this system is its flexibility - make it yours!
