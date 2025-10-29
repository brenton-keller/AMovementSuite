# A Movement Suite - Documentation Guide

## Project Overview

**A Movement Suite** (formerly "A Window Movement Suite") is an AutoHotkey v2.0 application that provides advanced window management and mouse hotkey functionality for Windows. It enables users to resize, move, snap, and control windows through intuitive mouse and keyboard combinations, along with extensive mouse-based productivity shortcuts.

### Key Capabilities
- **Window Management**: Dynamic window scaling (width, height, proportional), moving with snapping, always-on-top, window rolling, dimming, and cascading
- **Mouse Hotkeys**: Extensive mouse-based shortcuts for navigation, program launching, tab management, and system control
- **Virtual Desktop Integration**: Advanced virtual desktop management capabilities
- **Configuration Management**: Persistent user settings and preferences

## Quick Start

1. **Entry Point**: `AWindowMovementSuite.ahk2` - Main script that loads all modules
2. **Core Logic**: `src/` directory contains all source code
3. **Documentation**: See `/docs/` for detailed technical reference
4. **Settings**: User configurations stored in `settings/`

## Architecture at a Glance

```
AMovementSuite/
├── AWindowMovementSuite.ahk2    # Main entry point, tray menu setup
├── src/                          # All source code (see src/claude.md)
│   ├── Features/                 # Window management features
│   ├── MouseHotkeys/             # Mouse hotkey handlers (see src/MouseHotkeys/claude.md)
│   ├── lib/                      # Core utilities, config, helpers
│   ├── UI/                       # GUI components
│   └── VirtualDesktopLib/        # Virtual desktop integration
├── docs/                         # Technical reference documentation
├── settings/                     # User configuration files
└── assets/                       # Images, resources
```

## Directory Structure Guide

### `/src/Features/`
Window management feature modules. Each feature is self-contained:
- `WindowMove.ahk2` - Window moving with snapping and grid positioning
- `WindowScaleWidth.ahk2`, `WindowScaleHeight.ahk2`, `WindowScaleXY.ahk2` - Window resizing
- `WindowAlwaysOnTop.ahk2` - Toggle always-on-top state
- `WindowRollUp.ahk2` - Roll up windows to title bar
- `WindowDimmer.ahk2` - Dim inactive windows
- `WindowCascade.ahk2` - Cascade window arrangement
- `DisableWindowModifications.ahk2` - Disable modifications for specific windows

See `docs/features.md` for technical details.

### `/src/lib/`
Core libraries and utilities:
- **Config/**: `GlobalConfig.ahk2`, `UserConfig.ahk2` - Configuration management
- **Core/**: `CoreUtils.ahk2`, `ColorUtils.ahk2`, `MoveUtils.ahk2`, `ToggleUtils.ahk2` - Shared utilities
- **Helpers/**: `ErrorHandler.ahk2`, `LoggingHelper.ahk2` - Support functions

See `docs/configuration.md` for config system details.

### `/src/MouseHotkeys/`
Mouse-based hotkey handlers for productivity shortcuts:
- `hk_basic.ahk2` - Basic navigation (backspace, enter, delete, etc.)
- `hk_code.ahk2` - Code editor shortcuts
- `hk_programs.ahk2` - Program launching shortcuts
- `hk_tab.ahk2` - Tab management
- `hk_wheel.ahk2` - Mouse wheel customizations
- `setting_mouse.ahk2` - Mouse hotkey loader
- `windows_explorer.ahk2` - File explorer shortcuts
- `Startbar.ahk2` - Taskbar interactions

See `src/MouseHotkeys/claude.md` and `docs/mouse-hotkeys.md` for details.

### `/src/UI/`
User interface components:
- `CustomDialogs.ahk2` - Custom dialog boxes
- `SettingsGUI.ahk2` - Settings interfaces

See `docs/ui-components.md` for UI system details.

### `/src/VirtualDesktopLib/`
Virtual desktop management library with COM interfaces and desktop actions.
See `docs/virtual-desktops.md` for technical reference.

## Where to Find What

### For Development Tasks
- **Adding a new window feature**: See `docs/features.md` and `src/Features/`
- **Adding mouse hotkeys**: See `src/MouseHotkeys/claude.md` and `docs/mouse-hotkeys.md`
- **Modifying configuration**: See `docs/configuration.md`
- **Understanding architecture**: See `docs/architecture.md`
- **UI/GUI changes**: See `docs/ui-components.md`
- **Contributing**: See `docs/contributing.md`
- **Development setup**: See `docs/development.md`

### For Understanding Code
1. Start with this file (claude.md) for the big picture
2. Read `src/claude.md` for source code organization
3. Dive into specific `docs/*.md` files for technical details
4. Check subsection `claude.md` files (e.g., `src/MouseHotkeys/claude.md`) for focused context

## Development Workflow

### Adding New Features
1. Identify which subsystem (Features, MouseHotkeys, UI, etc.)
2. Create new file or modify existing module
3. Follow established patterns (see relevant `docs/*.md`)
4. Update configuration if needed
5. **Update documentation** (see below)
6. Test thoroughly with various window types

### Common Patterns
- **Feature Toggle**: Features have `{name}Enabled` global variables controlled via tray menu
- **Hotkey Pattern**: Mouse hotkeys check modifiers (Ctrl, Shift, Alt) for different behaviors
- **Window Validation**: Use `IsValidWindowFor*` functions to filter valid windows
- **Configuration**: Use GlobalConfig for system defaults, UserConfig for user preferences

See `docs/architecture.md` for detailed patterns.

## Documentation Update Reminders

**IMPORTANT**: When making changes, update documentation:

### When Adding/Modifying Features
- [ ] Update `docs/features.md` with technical details
- [ ] Update this file if it affects overall architecture
- [ ] Update `README.md` if it changes user-facing functionality

### When Adding/Modifying Mouse Hotkeys
- [ ] Update `docs/mouse-hotkeys.md` with hotkey details
- [ ] Update `src/MouseHotkeys/claude.md` if patterns change

### When Changing Configuration
- [ ] Update `docs/configuration.md` with new config options
- [ ] Document default values and valid ranges

### When Modifying Architecture
- [ ] Update `docs/architecture.md` with structural changes
- [ ] Update relevant subsection `claude.md` files
- [ ] Update this file if directory structure changes

### When Adding UI Components
- [ ] Update `docs/ui-components.md` with GUI details
- [ ] Document user interaction patterns

## Code Conventions

- **AutoHotkey v2.0**: All code uses AHK v2.0 syntax
- **Naming**: PascalCase for functions, camelCase for variables, UPPER_CASE for constants
- **Global Variables**: Minimize use, prefix with descriptive names (e.g., `windowMoveEnabled`)
- **Comments**: Document non-obvious logic, function purposes, and complex interactions
- **File Organization**: One feature per file, related utilities grouped in lib/

## Git Commit Philosophy

**Descriptive commits with clear title and detailed body. No fluff.**

### Commit Message Format
```
Title line (capitalized, imperative mood, concise)

Organized explanation of changes in 2-4 logical sections:
- Thorough but concise - clarity over comprehensiveness
- Use bullets to highlight key points and improvements
- Explain WHAT changed and WHY, not HOW (code shows how)
- NO "authored by", "generated with", or bot signatures
```

### Good Example
```
Add comprehensive documentation system

Created hierarchical documentation with AI context and technical reference:

Documentation Structure:
- Claude.md files: Project overview, source organization, subsystem guides
- /docs/ directory: Architecture, features, config, UI, contributing guides

Key Features:
- Layered detail: High-level overview in claude.md, technical depth in docs/
- Cross-referenced with update reminders for code changes
- Workflow-focused for both AI assistants and human developers
```

### Bad Examples
```
✗ WIP: working on calculator stuff

✗ feat: add calculator enhancements (#123)

✗ Update files
  🤖 Generated with Claude Code
  Co-Authored-By: Bot <bot@example.com>
```

### Guidelines
- **Title**: Capitalized, imperative mood ("Add feature" not "Added feature")
- **Body**: 2-4 organized sections with bullets - thorough but concise
- **Length**: Detailed enough to understand changes, not exhaustive documentation
- **No fluff**: No bot signatures, no "generated with" messages
- **Focused**: One logical change per commit when possible
- **Purpose-driven**: Explain what changed and why, not implementation details

## Testing & Debugging

- **Manual Testing**: Test with various window types (browsers, explorer, system windows)
- **Edge Cases**: Test with maximized, minimized, multi-monitor scenarios
- **Error Handling**: Use ErrorHandler helper for consistent error reporting
- **Logging**: Use LoggingHelper for debugging (when implemented)

See `docs/development.md` for detailed testing procedures.

## Quick Reference Links

- **Main Entry Point**: `AWindowMovementSuite.ahk2`
- **Tray Menu Setup**: `AWindowMovementSuite.ahk2:33-53`
- **Feature Modules**: `src/Features/`
- **Mouse Hotkeys**: `src/MouseHotkeys/`
- **Core Config**: `src/lib/Config/GlobalConfig.ahk2`
- **Documentation Index**: `docs/` directory

## Project Goals

1. **Modularity**: Each feature should be independent and toggleable
2. **Performance**: Minimal overhead, responsive user experience
3. **Reliability**: Robust error handling, safe window operations
4. **Usability**: Intuitive hotkeys, visual feedback, sensible defaults
5. **Maintainability**: Clear code structure, comprehensive documentation

---

**Next Steps for AI/Developers:**
1. Read `src/claude.md` for source code overview
2. Check subsection `claude.md` files for focused areas
3. Consult `docs/*.md` for technical deep dives
4. Follow documentation update reminders when making changes
