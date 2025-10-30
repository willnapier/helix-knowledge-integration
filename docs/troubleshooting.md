# Troubleshooting Guide

**Date**: 2025-10-30
**Status**: ✅ Active troubleshooting reference
**Related**: [Architecture Documentation](./architecture.md)

---

## Table of Contents

1. [Wiki Link Corruption](#wiki-link-corruption)
2. [Navigation Issues](#navigation-issues)
3. [Link Insertion Problems](#link-insertion-problems)
4. [File Creation Issues](#file-creation-issues)
5. [Service Management](#service-management)

---

## Wiki Link Corruption

### Diagnosing File Corruption

**Symptoms**:
- Files display as `??????????????????[[Link Text]]` with many repeated `?` markers
- Helix becomes slow when opening certain files
- Unusually high number of wiki link markers in a file

**How to Identify Corrupted Files**:

```bash
# Quick check: Count link markers in a file
rg -o "\\[\\[" ~/Forge/suspicious-file.md | wc -l

# If count is >1000, file is likely corrupted
# Normal files have <500 link markers maximum
```

**Corruption Characteristics**:

- **Normal file**: `[[Link Text]]` or `?[[Unresolved Link]]`
- **Corrupted file**: `??????????????????[[Link Text]]` (many repeated `?` markers)
- **Diagnostic threshold**: Files with >1000 link markers are almost certainly corrupted

**Link Distribution in Healthy Vault**:
- 90% of files: <100 links
- 9% of files: 100-300 links
- 1% of files: 300-500 links
- 0% of files: >500 links (anything higher indicates corruption)

### Comprehensive Corruption Scan

Run this to find all corrupted files across your entire vault:

```nushell
# Nushell command - scan for files with >1000 links
fd -t f ".md" ~/Forge ~/Admin
  | lines
  | each { |file|
      let count = (rg -o "\\[\\[" $file | lines | length)
      if $count > 1000 {
        {file: (basename $file), count: $count, path: $file}
      }
    }
  | compact
  | sort-by count -r
```

If this returns any results, you have corrupted files that need recovery.

### Recovery Procedure

If corruption is detected:

**Step 1: Stop the Link Management Service**

```bash
link-service stop
```

This prevents further corruption while you recover.

**Step 2: Identify All Corrupted Files**

```bash
# Create list of corrupted files
fd -t f ".md" ~/Forge ~/Admin | while IFS= read -r file; do
    count=$(rg -o "\\[\\[" "$file" | wc -l)
    if [ $count -gt 1000 ]; then
        echo "$file" >> /tmp/corrupted-files.txt
    fi
done

# Review the list
cat /tmp/corrupted-files.txt
```

**Step 3: Restore from Backup**

```bash
# Find most recent clean backup
ls -lt ~/.Trash/ForgeBackup_* | head -1

# Restore each corrupted file
while IFS= read -r file; do
    # Find the file in backup (search by basename)
    backup_file=$(find ~/.Trash/ForgeBackup_* -type f -name "$(basename "$file")" -print -quit)

    if [ -n "$backup_file" ]; then
        # Verify backup is not corrupted
        backup_count=$(rg -o "\\[\\[" "$backup_file" | wc -l)

        if [ $backup_count -lt 1000 ]; then
            cp "$backup_file" "$file"
            echo "✅ Restored: $(basename "$file")"
        else
            echo "⚠️ Backup also corrupted: $(basename "$file")"
        fi
    else
        echo "❌ No backup found: $(basename "$file")"
    fi
done < /tmp/corrupted-files.txt
```

**Step 4: Verify Recovery**

```bash
# Re-run corruption scan to confirm zero remaining corruption
nu -c 'fd -t f ".md" ~/Forge ~/Admin | lines | each { |file| let count = (rg -o "\\[\\[" $file | lines | length); if $count > 1000 { {file: (basename $file), count: $count} } } | compact'

# Should return empty result
```

**Step 5: Restart Service**

```bash
link-service start
```

### Prevention Measures

**Automated Corruption Monitoring**:

Create a weekly cron job to detect corruption early:

```nushell
# ~/scripts/check-vault-corruption.nu
def main [] {
    fd -t f ".md" ~/Forge ~/Admin
      | lines
      | each { |file|
          let count = (rg -c "\\[\\[" $file | into int)
          if $count > 1000 {
            print $"⚠️ CORRUPTED: ($file) has ($count) link markers"
          }
        }
}
```

Add to crontab:
```bash
# Run every Sunday at 2 AM
0 2 * * 0 nu ~/scripts/check-vault-corruption.nu
```

**Regular Backups**:

```bash
# Create dated backup before major changes
rsync -av ~/Forge/ ~/.Trash/ForgeBackup_$(date +%Y-%m-%d)/
echo "✅ Backup created: ~/.Trash/ForgeBackup_$(date +%Y-%m-%d)/"

# Verify backup
backup_count=$(fd -t f ".md" ~/.Trash/ForgeBackup_$(date +%Y-%m-%d)/ | wc -l)
source_count=$(fd -t f ".md" ~/Forge/ | wc -l)
echo "Backup: $backup_count files | Source: $source_count files"
```

---

## Navigation Issues

### Wiki Link Navigation Not Working (`Space+w`)

**Symptom**: Pressing `Space+w` on a wiki link doesn't open the target file

**Check 1: Is the script available?**

```bash
which hx-wiki
# Should show: /Users/williamnapier/.local/bin/hx-wiki

# If not found:
ls -la ~/.local/bin/hx-wiki
chmod +x ~/.local/bin/hx-wiki  # Make executable if needed
```

**Check 2: Does the symlink get created?**

```bash
# Place cursor on a wiki link in Helix, press Space+w
# Then check:
ls -la /tmp/helix-current-link.md

# Should be a symlink pointing to the target file
```

**Check 3: Is the target file in the vault?**

```bash
# Find a file by name
fd -t f "^filename.md$" ~/Forge ~/Admin ~/Archives

# If file doesn't exist, the ?[[ marker is correct
```

**Manual Testing**:

```bash
# Test the hx-wiki script directly
echo "[[Test Note]]" | hx-wiki

# Should create symlink at /tmp/helix-current-link.md
ls -la /tmp/helix-current-link.md
```

### System File Opener Not Working (`Space+o`)

**Symptom**: Pressing `Space+o` doesn't open files in appropriate applications

**Check 1: Is the script available?**

```bash
which hx-open-system
# Should show: /Users/williamnapier/.local/bin/hx-open-system

# Test manually:
hx-open-system /path/to/file.pdf
```

**Platform-Specific Issues**:

**macOS**:
```bash
# Should use `open` command
which open
# Should show: /usr/bin/open
```

**Linux**:
```bash
# Should use `xdg-open` command
which xdg-open
# If not found: sudo apt install xdg-utils (Debian/Ubuntu)
# or: sudo dnf install xdg-utils (Fedora)
```

---

## Link Insertion Problems

### Fuzzy Link Picker Not Appearing (`Alt+l`)

**Symptom**: Pressing `Alt+l` doesn't show the fuzzy picker

**Check 1: Are you in a Zellij session?**

```bash
# Check if Zellij is running
echo $ZELLIJ

# If empty, you're not in Zellij
# Start Zellij: zellij
```

**Check 2: Is the keybinding configured?**

```bash
# Check Zellij config
grep -A 5 "Alt l" ~/.config/zellij/config.kdl
```

**Check 3: Is the obsidian-linker script available?**

```bash
ls -la ~/.config/helix/obsidian-linker.sh
chmod +x ~/.config/helix/obsidian-linker.sh  # Make executable if needed
```

**Check 4: Is OBSIDIAN_VAULT set?**

```bash
echo $OBSIDIAN_VAULT
# Should show: /Users/williamnapier/Forge

# If not set, add to shell config:
# ~/.zshrc or ~/.config/nushell/env.nu
export OBSIDIAN_VAULT="/Users/williamnapier/Forge"
```

**Check 5: Are dependencies installed?**

```bash
# Required: skim (fuzzy finder)
which sk
# If not found: brew install sk (macOS)
# or: cargo install skim (any platform)

# Optional but recommended: bat (syntax highlighting)
which bat
# If not found: brew install bat
```

### Link Not Pasting (`Space+l`)

**Symptom**: After selecting a link with `Alt+l`, pressing `Space+l` doesn't insert it

**Check 1: Is link in clipboard?**

```bash
# After selecting with Alt+l, check clipboard:
pbpaste  # macOS
# or
xclip -o  # Linux

# Should show: [[Note Name]]
```

**Check 2: Is Helix keybinding configured?**

```bash
# Check Helix config
grep "space.l" ~/.config/helix/config.toml
```

Should show:
```toml
l = ":insert-output pbpaste"  # macOS
# or
l = ":insert-output xclip -o"  # Linux
```

---

## File Creation Issues

### New Notes Not Getting Templates

**Symptom**: Creating a new file doesn't add YAML frontmatter template

**Check**: Is the file being created with the right script?

```bash
# When using file creation script
which create-daily-note  # or similar
```

**Manual Template Addition**:

```bash
# Template for new permanent notes
cat > new-note.md << 'EOF'
---
created: $(date +%Y-%m-%d)
tags: []
---

# Note Title

EOF
```

### Daily Notes Not Working

**Symptom**: Daily note command doesn't create properly dated files

**Check 1: Is the script available?**

```bash
which create-daily-note
ls -la ~/.local/bin/create-daily-note
```

**Check 2: Is the format correct?**

Daily notes should follow format: `YYYY-MM-DD.md`

```bash
# Create daily note manually
touch ~/Forge/fleeting/$(date +%Y-%m-%d).md
```

---

## Service Management

### Link Service Won't Start

**Symptom**: `link-service start` fails or shows errors

**Check 1: Are the watcher scripts available?**

```bash
which wiki-backlinks
which wiki-resolve-mark

# Should both show paths like:
# /Users/williamnapier/.local/bin/wiki-backlinks
```

**Check 2: Are directories being watched?**

```bash
# Check if Forge exists
ls -la ~/Forge/

# Check if Admin exists (optional)
ls -la ~/Admin/
```

**Check 3: Are there stale PIDs?**

```bash
# Check for stale PID file
cat ~/scripts/wiki-link-management/logs/link-service.pid

# If service shows running but isn't:
link-service stop  # Force stop
rm ~/scripts/wiki-link-management/logs/link-service.pid  # Remove stale PID
link-service start  # Restart
```

**Check 4: View error logs**

```bash
# Check for errors
link-service errors

# Or view logs directly
cat ~/scripts/wiki-link-management/logs/backlinks-Forge.err.log
cat ~/scripts/wiki-link-management/logs/resolve-Forge.err.log
```

### Service Running but Not Processing

**Symptom**: Service status shows running, but files aren't being updated

**Check 1: Are the processes actually running?**

```bash
link-service status

# Should show PIDs and process status
# Verify with ps:
ps aux | grep wiki-backlinks
ps aux | grep wiki-resolve-mark
```

**Check 2: Are logs showing activity?**

```bash
# Monitor logs in real-time
tail -f ~/scripts/wiki-link-management/logs/backlinks-Forge.out.log

# In another terminal, edit a file and save
# Logs should show processing activity
```

**Check 3: Is debounce delay too high?**

Default debounce is 2000ms (2 seconds). If you modified this:

```bash
# Check running command
ps aux | grep wiki-backlinks

# Should show: --debounce-ms 2000 (or similar)
```

---

## Common Error Messages

### "Error: File not found"

**Cause**: Target file doesn't exist or is outside watched directories

**Solution**:
- Verify file exists: `fd -t f "filename.md" ~/Forge ~/Admin`
- If file is outside Forge/Admin, move it or add directory to watcher config

### "Error: Permission denied"

**Cause**: Script doesn't have execute permissions or file is read-only

**Solution**:
```bash
chmod +x ~/.local/bin/hx-wiki
chmod +x ~/.local/bin/wiki-backlinks
chmod +x ~/.local/bin/wiki-resolve-mark
```

### "Error: Command not found: pbpaste"

**Cause**: On Linux, but config uses macOS clipboard command

**Solution**: Update Helix config to use `xclip -o` instead of `pbpaste`

### "Error: OBSIDIAN_VAULT not set"

**Cause**: Environment variable not configured

**Solution**:
```bash
# Add to shell config (~/.zshrc or ~/.config/nushell/env.nu)
export OBSIDIAN_VAULT="$HOME/Forge"

# Reload shell config
source ~/.zshrc  # or restart terminal
```

---

## Performance Issues

### Helix Slow When Opening Files

**Possible Causes**:
1. **File corruption**: Check link count with `rg -o "\\[\\[" file.md | wc -l`
2. **Very large file**: Check file size with `ls -lh file.md`
3. **Too many watchers**: Check process count with `ps aux | grep wiki- | wc -l`

**Solutions**:
1. If corrupted: Restore from backup (see corruption section above)
2. If large: Consider splitting into smaller notes
3. If too many watchers: Verify only expected watchers are running

### Link Management Causing High CPU

**Check watcher CPU usage**:

```bash
# Monitor CPU usage
top -p $(pgrep -d',' wiki-backlinks)

# If high CPU, check for:
1. Large batch operations (many files created/modified at once)
2. Runaway loop (rapidly incrementing link markers)
3. Very large files being processed
```

**Solutions**:
1. Increase debounce delay: `wiki-backlinks --debounce-ms 5000`
2. Check for corruption (runaway loops)
3. Exclude large directories if not needed

---

## Getting Help

If issues persist after trying these troubleshooting steps:

1. **Check logs** for detailed error messages
2. **Enable verbose logging** by adding `--verbose` flag to watcher commands
3. **Create a minimal test case** to isolate the issue
4. **Check GitHub issues** for similar problems
5. **Open a new issue** with:
   - Detailed description of the problem
   - Steps to reproduce
   - Relevant log excerpts
   - System information (OS, Helix version, etc.)

---

## Related Documentation

- [Architecture Documentation](./architecture.md) - System design and components
- [Customization Guide](./customization.md) - Configuration options
- [Installation Guide](./installation.md) - Setup instructions
- [File Watcher Safety](https://github.com/willnapier/nushell-knowledge-tools/blob/main/docs/file-watcher-safety.md) - Prevention measures

---

**Document Status**: ✅ Complete
**Last Updated**: 2025-10-30
**Covers**: Corruption diagnosis, navigation issues, service management, common errors
