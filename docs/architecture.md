# Architecture and Design Decisions

This document explains the architectural decisions behind helix-knowledge-integration and how it relates to nushell-knowledge-tools.

## Repository Philosophy

### Separation of Concerns

This repository is a **complete reference implementation** of Helix editor integration for universal knowledge management tools. It demonstrates the architectural pattern:

```
Universal Functions (nushell-knowledge-tools)
         ↓ invoked by
Editor Integration (helix-knowledge-integration)
```

### What This Repository IS

✅ **A complete, immediately usable Helix configuration** for knowledge management
✅ **A reference implementation** showing one way to integrate universal tools
✅ **A template** for building similar integrations for other editors
✅ **Production-ready** code with comprehensive documentation

### What This Repository IS NOT

❌ **The only way** to use nushell-knowledge-tools with Helix
❌ **Prescriptive** about keybindings or workflow
❌ **A comprehensive Helix configuration** (only knowledge management features)
❌ **Required** to use the universal functions

## Design Principles

### 1. Thin Wrapper Architecture

**Principle**: Keep editor-specific logic minimal. Universal functions do the heavy lifting.

**Example**:
```toml
# In config.toml - just invoke the universal function
[keys.normal.space]
w = [
    "extend_to_line_bounds",
    ":pipe-to hx-wiki",          # Universal script
    ":sh sleep 0.1",
    ":buffer-close! /tmp/helix-current-link.md",
    ":open /tmp/helix-current-link.md"
]
```

**Why**: Makes the integration maintainable and portable. Logic lives in one place (the universal functions).

### 2. Complete Over Minimal

**Principle**: Provide a complete working configuration, not just examples.

**Decision**: Include all necessary scripts, documentation, and configuration in one repository.

**Why**: Users can clone and immediately use it. No hunting for missing pieces or assembling from examples.

### 3. Cross-Platform by Default

**Principle**: All code must work on macOS and Linux without modification.

**Implementation**:
- Platform detection in scripts: `sys | get host.name`
- Conditional clipboard commands: `pbpaste` (macOS) or `wl-paste`/`xclip` (Linux)
- Cross-platform file opening: `open` vs `xdg-open`

**Why**: Knowledge work happens on multiple platforms. No one should feel left out.

### 4. Documentation as Architecture

**Principle**: If it can't be clearly documented, the design isn't clear.

**Evidence**:
- README.md: 700+ lines
- installation.md: Complete with verification steps
- customization.md: Comprehensive adaptation guide

**Why**: Documentation forces architectural clarity and serves users.

## Technical Decisions

### Why Nushell for Scripts?

**Decision**: Write `hx-wiki` and `hx-open-system` in Nushell (not bash/sh)

**Reasoning**:
1. **Consistency**: Same language as nushell-knowledge-tools
2. **Cross-platform**: Nushell works identically on all platforms
3. **Structured data**: Better than bash text processing
4. **Already installed**: Users have Nushell (prerequisite for universal functions)

**Trade-off**: Requires Nushell (acceptable - already a dependency)

### Why /tmp/ for Communication?

**Decision**: Use `/tmp/helix-current-link.md` symlink for script-to-Helix communication

**Reasoning**:
1. **Cross-platform**: Works on macOS, Linux, BSD
2. **Auto-cleanup**: System clears on reboot
3. **No vault pollution**: Doesn't create files in knowledge base
4. **Symlink support**: Helix can open symlinks
5. **Simple protocol**: Easy to understand and debug

**Alternatives Considered**:
- Direct file paths (rejected: loses smart handling logic)
- Environment variables (rejected: Helix doesn't pass to scripts easily)
- Named pipes (rejected: more complex, platform differences)

### Why Space+ Prefix?

**Decision**: Use `Space` as leader key for knowledge management bindings

**Reasoning**:
1. **Vim tradition**: Space is common leader key in modal editors
2. **Low conflict**: Unlikely to conflict with Helix defaults
3. **Ergonomic**: Easy to press, doesn't require modifiers
4. **Semantic**: "Space" = "open up space for knowledge work"

**Alternatives Considered**:
- `Alt+` (rejected: OS/terminal often intercepts)
- `Ctrl+` (rejected: many defaults use Ctrl)
- `g` prefix (rejected: already heavily used in Helix)

### Why Separate Scripts?

**Decision**: Split functionality into `hx-wiki` and `hx-open-system`

**Reasoning**:
1. **Single responsibility**: Each script does one thing well
2. **Easier to understand**: Smaller, focused codebases
3. **Independent use**: Can use `hx-open-system` without `hx-wiki`
4. **Easier to customize**: Modify one without affecting the other

**Alternative Considered**:
- Monolithic `hx-knowledge` script (rejected: harder to maintain and customize)

### Why YAML Frontmatter?

**Decision**: Auto-created notes include YAML frontmatter

**Reasoning**:
1. **Widespread standard**: Used by Jekyll, Obsidian, many static site generators
2. **Machine-readable**: Easy to parse and process
3. **Extensible**: Can add custom fields
4. **Human-readable**: Users can see and edit
5. **Metadata separation**: Clean separation from content

**Format Chosen**:
```yaml
---
tags:
  -
date created: 2025-10-27 14:30
date modified: 2025-10-27 14:30
---
```

**Why this format**:
- ISO 8601 dates (sortable, unambiguous)
- Tags as YAML list (extensible)
- Created/modified distinction (useful for many workflows)

## Integration Patterns

### Pattern 1: Clipboard Bridge

**How it works**:
1. Run universal function in terminal (e.g., `fsl`)
2. Function copies result to clipboard
3. In Helix: `Space+l` pastes from clipboard

**Advantages**:
- Works with ANY universal function that outputs to clipboard
- Terminal and editor stay independent
- No complex IPC needed

**Trade-offs**:
- Two-step process (terminal → editor)
- Requires switching context briefly

### Pattern 2: Direct Invocation

**How it works**:
1. Helix keybinding pipes selection to script
2. Script invokes universal function
3. Script communicates result back to Helix

**Advantages**:
- Single keystroke operation
- Stays in Helix context
- Can use cursor position, selection, etc.

**Trade-offs**:
- Requires editor-specific wrapper script
- More complex communication protocol

**This repository uses both patterns** for different use cases.

## File Organization

### Why This Structure?

```
helix-knowledge-integration/
├── README.md              # Discovery and overview
├── config.toml            # The actual integration
├── scripts/               # Editor-specific wrappers
└── docs/                  # Detailed guides
```

**Reasoning**:
- **Flat root**: Easy to find key files
- **scripts/ directory**: Clear what's executable
- **docs/ directory**: Detailed docs don't clutter root
- **No src/**: These are scripts, not a software project

### Why Include LICENSE?

**Decision**: MIT License

**Reasoning**:
1. **Permissive**: Users can adapt freely
2. **Simple**: Easy to understand
3. **Compatible**: Works with nushell-knowledge-tools (also MIT)
4. **Standard**: Widely recognized and accepted

## Relationship to Other Repositories

### nushell-knowledge-tools (Parent/Sibling)

**Relationship**: This repository **invokes functions from** nushell-knowledge-tools

**Not**: A fork, a subset, or a replacement

**Dependency**: This repository **requires** nushell-knowledge-tools to function

### Personal Dotfiles (Parallel)

**Relationship**: Intentional duplication serving different purposes

**This repository**: Curated, stable, documented reference
**Dotfiles**: Personal, experimental, rapidly iterating

**Flow**: Proven features from dotfiles → distilled into → this repository

**Why both exist**: See "Duplication Philosophy" in main documentation

## Customization Points

### Designed to be Customized

The architecture explicitly supports customization:

1. **Keybindings**: Easy to remap in `config.toml`
2. **Templates**: Edit frontmatter in `hx-wiki` script
3. **Directories**: Change vault paths in scripts
4. **File types**: Modify extension handling
5. **Platform-specific**: Add support for other platforms

### How to Customize Safely

1. **Fork the repository** (keep original for reference)
2. **Modify for your needs** (document changes)
3. **Sync selectively** (pull updates you want)
4. **Share back** (PR improvements that benefit everyone)

## Future Extensibility

### Designed for Growth

The architecture supports adding:
- New keybindings (Space is just a prefix)
- New scripts (just add to `scripts/`)
- New documentation (just add to `docs/`)
- New templates (customize in scripts)

### What NOT to Add

This repository should NOT include:
- Non-knowledge-management features
- Comprehensive Helix configuration (themes, LSP, etc.)
- Features specific to one person's workflow
- Experimental/unstable features

**Why**: Keep focused. Comprehensive Helix configs belong in dotfiles or Helix configuration repos.

## Testing Philosophy

### Validation Over Tests

**Approach**: Comprehensive documentation with verification steps rather than automated tests

**Reasoning**:
1. **Simple codebase**: Scripts are straightforward
2. **Integration focus**: Testing integration is harder than unit testing
3. **Platform differences**: Would need test matrix for macOS/Linux
4. **User validation**: Users test by using

### What Could Be Tested

Future possibilities:
- Script syntax validation (Nushell parsing)
- Config syntax validation (Helix TOML)
- Documentation link checking
- Cross-platform script execution

## Version Strategy

### Semantic Versioning

**Format**: MAJOR.MINOR.PATCH

**What changes mean**:
- **MAJOR**: Breaking changes (keybinding changes, removed features)
- **MINOR**: New features (new scripts, new keybindings)
- **PATCH**: Bug fixes, documentation improvements

### Git Branches

**main**: Stable, production-ready code
**Feature branches**: Development of new features
**No develop branch**: Simple is better

## Maintenance Guidelines

### What to Maintain

✅ **Fix bugs** reported by users
✅ **Update documentation** when things change
✅ **Add proven features** from personal dotfiles
✅ **Improve cross-platform support** when possible
✅ **Keep synchronized** with nushell-knowledge-tools updates

### What NOT to Maintain

❌ **Support every Helix version** (test against current stable)
❌ **Add features speculatively** (only proven ones)
❌ **Support every workflow** (provide customization guidance instead)
❌ **Become comprehensive Helix config** (stay focused on knowledge management)

## Success Metrics

### How to Measure Success

1. **Adoption**: GitHub stars, forks, downloads
2. **Community**: Issues, PRs, discussions
3. **Inspiration**: Other editor integrations created
4. **Stability**: Few bugs, stable functionality
5. **Clarity**: Users understand and can customize

### NOT Metrics

- Total features
- Lines of code
- Comprehensiveness

**Why**: Quality over quantity. Focused over comprehensive.

## Conclusion

This repository demonstrates that **universal tools + thin editor wrappers** is a viable and valuable architecture. It provides immediate value while remaining maintainable and extensible.

The intentional separation from both nushell-knowledge-tools (universal) and personal dotfiles (personal) serves distinct purposes and benefits the community.

---

**Last Updated**: 2025-10-27
**Maintained By**: [willnapier](https://github.com/willnapier)
