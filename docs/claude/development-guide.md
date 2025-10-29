# Development Guide

## Development Workflow

### Adding New Features

1. **Identify subsystem**: Features, MouseHotkeys, UI, etc.
2. **Create/modify file**: Follow established patterns
3. **Follow patterns**: See @docs/claude/code-conventions.md
4. **Update configuration**: If feature needs settings
5. **Update documentation**: See below
6. **Test thoroughly**: Various window types and scenarios

### Adding a New Window Feature

1. Create new file in `Features/` (e.g., `WindowNewFeature.ahk2`)
2. Define global enabled variable: `global newFeatureEnabled := true`
3. Implement hotkey handler(s)
4. Create toggle function: `ToggleNewFeature(*)`
5. Add to main script: `#Include src\Features\WindowNewFeature.ahk2`
6. Add tray menu entry in `AWindowMovementSuite.ahk2`
7. **Update @docs/features.md** with technical details

### Adding a New Mouse Hotkey

See @src/MouseHotkeys/claude.md for detailed instructions.

Summary:
1. Choose appropriate file (`hk_basic.ahk2`, `hk_code.ahk2`, etc.)
2. Define hotkey with RAlt + Function key pattern
3. Add context awareness if needed (application-specific)
4. Test with and without modifiers
5. **Update @docs/mouse-hotkeys.md** with hotkey details

### Modifying Core Utilities

1. Identify appropriate module (CoreUtils, ColorUtils, MoveUtils, etc.)
2. Add function with clear documentation
3. Ensure no side effects (pure functions preferred)
4. **Update @docs/configuration.md** if adding config options

### UI Changes

1. Modify or create components in `UI/`
2. Follow existing GUI patterns from `SettingsGUI.ahk2`
3. **Update @docs/ui-components.md** with changes

## Documentation Update Workflow

Use the `/doc` command to guide documentation updates:

```bash
/doc                    # Check what changed, guide through updates
/doc WindowDimmer       # Focus on specific component
```

### When to Update Documentation

**After modifying Features/:**
- [ ] `docs/features.md` - Technical details of the feature
- [ ] `CLAUDE.md` - If architecture/patterns changed
- [ ] `README.md` - If user-facing behavior changed

**After modifying Config/:**
- [ ] `docs/configuration.md` - Document new settings
- [ ] `docs/claude/code-conventions.md` - If affects common patterns

**After modifying UI/:**
- [ ] `docs/ui-components.md` - GUI changes

**After modifying MouseHotkeys/:**
- [ ] `docs/mouse-hotkeys.md` - Hotkey reference
- [ ] `src/MouseHotkeys/claude.md` - If patterns changed

**After modifying architecture/structure:**
- [ ] `docs/architecture.md` - System structure
- [ ] `docs/claude/architecture-overview.md` - High-level overview
- [ ] `CLAUDE.md` - Directory structure if changed

## Using Slash Commands

The project provides helpful development commands:

- `/plan` - Plan features with complexity tracking before building
- `/commit` - Create properly formatted commits
- `/doc` - Update documentation with guidance
- `/search` - Find code patterns and implementations
- `/cleanup` - Remove dead code safely

## Testing & Debugging

- **Manual Testing**: Test with various window types (browsers, explorer, system windows)
- **Edge Cases**: Test with maximized, minimized, multi-monitor scenarios
- **Error Handling**: Use ErrorHandler helper for consistent error reporting
- **Logging**: Use LoggingHelper for debugging (when implemented)

See @docs/development.md for detailed testing procedures.

## Quick File Reference

**Most frequently modified:**
- `Features/Window*.ahk2` - Window features
- `MouseHotkeys/hk_*.ahk2` - Mouse shortcuts
- `lib/Config/GlobalConfig.ahk2` - Configuration options
- `UI/SettingsGUI.ahk2` - Settings interfaces

**Core dependencies:**
- `lib/Core/CoreUtils.ahk2` - Shared utilities
- `lib/Core/ToggleUtils.ahk2` - Feature toggle management
- `lib/Config/GlobalConfig.ahk2` - System configuration

## Project Goals

1. **Modularity**: Each feature should be independent and toggleable
2. **Performance**: Minimal overhead, responsive user experience
3. **Reliability**: Robust error handling, safe window operations
4. **Usability**: Intuitive hotkeys, visual feedback, sensible defaults
5. **Maintainability**: Clear code structure, comprehensive documentation
