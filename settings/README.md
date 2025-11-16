# Hardware Configuration Settings

This directory contains hardware-specific configuration files for peripheral devices used with A Movement Suite.

## Razer Synapse Configurations

These files configure macro buttons and key bindings on Razer gaming peripherals (mouse, keyboard, keypad) to work seamlessly with the AutoHotkey window management system.

### Files

| File | Purpose | Device |
|------|---------|--------|
| `chroma_functions_f.synapse3` | Function key (F1-F12) macros | Razer Synapse 3 devices |
| `chroma_functions_num.synapse3` | Numpad key macros | Razer Synapse 3 devices |
| `synapse4_trinity_numpad.synapse4` | Trinity numpad configuration | Razer Synapse 4 devices |
| `trinity_f.synapse4` | Trinity function key configuration | Razer Synapse 4 devices |

### Why Version Controlled?

These configurations are tracked in git because:

1. **Team Consistency**: Team members using compatible Razer hardware can use the same optimized hotkey layouts
2. **Backup & Restore**: Easy recovery after hardware replacements or system reinstalls
3. **Documentation**: Shows the recommended peripheral setup for optimal workflow
4. **Reference**: Even if you use different hardware, these files document the intended hotkey mappings

### Usage

#### If You Have Compatible Razer Hardware:

1. Install Razer Synapse (version 3 or 4, depending on your device)
2. Import the appropriate `.synapse3` or `.synapse4` file into Synapse
3. The macro buttons will be pre-configured to work with A Movement Suite

#### If You Don't Have Razer Hardware:

- These files serve as **reference documentation** for the intended hotkey layout
- You can configure your own peripherals to match the same key bindings
- The actual window management functionality is in the `.ahk2` scripts, not these configs

### Updating These Files

If you modify your Razer Synapse configuration:

1. Export your updated profile from Razer Synapse
2. Replace the appropriate file in this directory
3. Commit with a clear message describing what changed (e.g., "Add hotkey for new feature")
4. Consider documenting major changes in this README

### Related Documentation

- Mouse hotkeys are documented in `src/MouseHotkeys/claude.md`
- Feature-specific hotkeys are in `docs/features.md`
- Keyboard shortcuts are in `docs/mouse-hotkeys.md`

---

**Note**: While these configs are version controlled as shared team resources, you can use A Movement Suite with any keyboard/mouse setup. The Razer configs are just one optimized configuration option.
