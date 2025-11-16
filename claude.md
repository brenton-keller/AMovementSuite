# A Movement Suite

**AutoHotkey v2.0 window management suite with mouse-driven controls and extensive productivity shortcuts.**

## Quick Start

- **Tech Stack**: AutoHotkey v2.0
- **Entry Point**: `AWindowMovementSuite.ahk2`
- **Run**: Execute the main script, use tray menu to toggle features

## Key Capabilities

- **Window Management**: Dynamic scaling (width, height, proportional), moving with snapping, grouping, always-on-top, rolling, dimming, cascading
- **Mouse Hotkeys**: Extensive shortcuts for navigation, program launching, tab management, system control
- **Virtual Desktop Integration**: Advanced virtual desktop management
- **Configuration**: Persistent user settings and preferences

## Architecture

See `docs/claude/architecture-overview.md` for detailed structure.

Quick map:
- `src/Features/` - Window management features
- `src/MouseHotkeys/` - Mouse hotkey handlers (`src/MouseHotkeys/claude.md`)
- `src/lib/` - Core utilities, config, helpers
- `src/UI/` - GUI components
- `docs/` - Technical reference documentation

## Development

See `docs/claude/development-guide.md` for workflow.

Quick reference:
- **Adding features**: See `docs/features.md` and `src/Features/`
- **Adding hotkeys**: See `src/MouseHotkeys/claude.md`
- **Configuration**: See `docs/configuration.md`
- **Code patterns**: See `docs/claude/code-conventions.md`

## Slash Commands

Essential workflows via Claude Code commands:

- `/plan [task]` - Plan features with complexity tracking
- `/commit [focus]` - Create properly formatted commits
- `/doc [component]` - Update documentation with guidance
- `/search [query]` - Find code patterns intelligently
- `/cleanup` - Remove dead code safely

## Git Workflow

See `docs/claude/git-workflow.md` for commit philosophy.

**Quick rules:**
- Descriptive commits: clear title + organized body
- No fluff: no bot signatures or "generated with" messages
- Use `/commit` command for proper formatting

## Documentation Map

**For AI/Claude Code context:**
- Project overview → This file
- Source organization → `src/claude.md`
- Mouse hotkeys system → `src/MouseHotkeys/claude.md`

**For technical reference:** (load explicitly when needed)
- Architecture details → `docs/architecture.md`
- Feature specifications → `docs/features.md`
- Configuration system → `docs/configuration.md`
- AHK2 memory management → `docs/ahk2-memory-management.md`
- Mouse hotkey reference → `docs/mouse-hotkeys.md`
- UI components → `docs/ui-components.md`
- Virtual desktops → `docs/virtual-desktops.md`
- Contributing guidelines → `docs/contributing.md`
- Development setup → `docs/development.md`

**For focused guides:**
- Architecture overview → `docs/claude/architecture-overview.md`
- Code conventions → `docs/claude/code-conventions.md`
- Development workflow → `docs/claude/development-guide.md`
- Git workflow → `docs/claude/git-workflow.md`

**For comprehensive manuals:** (load when deep understanding needed)
- AHK2 reference guide → `manuals/ahk2/index.md`
- Claude Code guide → `manuals/claude/index.md`
- All manuals directory → `manuals/*/index.md`

## Documentation Philosophy

- **CLAUDE.md**: High-level navigation (~100 lines)
- **docs/claude/**: Focused guides for AI context
- **docs/*.md**: Comprehensive technical reference
- **manuals/**: Deep-dive reference manuals (AHK2, Claude Code, etc.)
- **src/claude.md**: Source code organization
- **Subsystem claude.md**: Detailed subsystem guides

Use `path/to/file.md` references for progressive disclosure.

## Project Goals

1. **Modularity**: Independent, toggleable features
2. **Performance**: Minimal overhead, responsive UX
3. **Reliability**: Robust error handling, safe operations
4. **Usability**: Intuitive hotkeys, visual feedback
5. **Maintainability**: Clear structure, comprehensive docs

---

**Next Steps:**
- Developers → `docs/claude/development-guide.md`
- Architecture → `docs/claude/architecture-overview.md`
- Code conventions → `docs/claude/code-conventions.md`
- Technical deep dives → `docs/*.md`
- Reference manuals → `manuals/*/index.md`
