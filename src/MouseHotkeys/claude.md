# MouseHotkeys/ - Mouse Hotkey System

This directory implements an extensive mouse-based hotkey system for productivity shortcuts, navigation, program launching, and system control.

## Overview

The mouse hotkey system provides:
- **Context-aware shortcuts**: Different behavior per application
- **Modifier combinations**: RAlt + Function keys with Ctrl/Shift/Alt variants
- **Mouse wheel enhancements**: Custom scroll behaviors
- **Program launching**: Quick access to frequently used programs
- **Tab management**: Browser and application tab control
- **Explorer integration**: File manager specific shortcuts

## File Organization

```
MouseHotkeys/
├── setting_mouse.ahk2        # Main loader - includes all hotkey files
├── hk_basic.ahk2              # Basic navigation (backspace, enter, delete, etc.)
├── hk_code.ahk2               # Code editor specific shortcuts
├── hk_programs.ahk2           # Program launching shortcuts
├── hk_tab.ahk2                # Tab management
├── hk_wheel.ahk2              # Mouse wheel customizations
├── windows_explorer.ahk2      # File explorer shortcuts
└── Startbar.ahk2              # Taskbar interactions
```

### File Responsibilities

**setting_mouse.ahk2**
- Entry point loaded by main script
- Includes all individual hotkey handler files
- Defines loader order

**hk_basic.ahk2**
- Fundamental navigation keys (backspace, enter, delete, arrow keys)
- RAlt + F1-F12 for basic shortcuts
- Context-aware: different behavior for browsers vs other apps
- Modifier support: Ctrl, Shift, Alt variants

**hk_code.ahk2**
- Code editor specific shortcuts
- IDE and text editor optimizations
- Programming workflow enhancements

**hk_programs.ahk2**
- Quick program launching shortcuts
- Frequently used application access
- Window activation for running programs

**hk_tab.ahk2**
- Tab navigation (next/previous tab)
- Tab creation and closing
- Browser and multi-tab application support

**hk_wheel.ahk2**
- Mouse wheel behavior customizations
- Scroll speed modifications
- Horizontal scrolling

**windows_explorer.ahk2**
- File explorer specific shortcuts
- File navigation enhancements
- Folder operations

**Startbar.ahk2**
- Taskbar interactions
- System tray access
- Window switching

## Common Patterns

### Modifier Detection Pattern

Most hotkeys follow this pattern with RAlt + Function key as base:

```ahk
RAlt & F1::{
    if GetKeyState("LCtrl", "P") {
        ; Ctrl variant behavior
        Send("^{Backspace}")  ; Example: Ctrl+Backspace
    }
    else if GetKeyState("LShift", "P") {
        ; Shift variant behavior
        Send("+{Backspace}")  ; Example: Shift+Backspace
    }
    else if GetKeyState("LAlt", "P") {
        ; Alt variant behavior
        Send("!{Backspace}")  ; Example: Alt+Backspace
    }
    else {
        ; Default behavior (no modifiers)
        Send("{Backspace}")
    }
}
```

### Context-Aware Shortcuts

Many hotkeys check the active application and adapt behavior:

```ahk
RAlt & F1::{
    ActiveProcess := WinGetProcessName("A")

    switch ActiveProcess {
        case "msedge.exe", "brave.exe", "opera.exe", "Vivaldi.exe":
            Send("!{Left}")  ; Browser back navigation
        case "XYplorer.exe":
            Send("!{Left}")  ; File manager back
        default:
            Send("{Backspace}")  ; Standard backspace
    }
}
```

### Application Process Names

Common process names used for context detection:
- **Browsers**: `msedge.exe`, `brave.exe`, `opera.exe`, `Vivaldi.exe`, `chrome.exe`, `firefox.exe`
- **File Managers**: `XYplorer.exe`, `explorer.exe`
- **Code Editors**: `Code.exe` (VS Code), `devenv.exe` (Visual Studio), `sublime_text.exe`

## How to Add New Hotkeys

### 1. Choose the Right File
- Basic navigation → `hk_basic.ahk2`
- Code/IDE specific → `hk_code.ahk2`
- Program launching → `hk_programs.ahk2`
- Tab operations → `hk_tab.ahk2`
- Mouse wheel → `hk_wheel.ahk2`
- Explorer operations → `windows_explorer.ahk2`
- Taskbar/system → `Startbar.ahk2`

### 2. Define the Hotkey

Use the RAlt + Function key pattern:

```ahk
RAlt & F10::{
    ; Check for modifiers if needed
    if GetKeyState("LCtrl", "P") {
        ; Ctrl+RAlt+F10 behavior
    }
    else {
        ; RAlt+F10 behavior
    }
}
```

### 3. Add Context Awareness (Optional)

```ahk
RAlt & F10::{
    ActiveProcess := WinGetProcessName("A")

    ; Different behavior per application
    if (ActiveProcess = "target_app.exe") {
        ; App-specific behavior
    }
    else {
        ; Default behavior
    }
}
```

### 4. Test Thoroughly
- Test with target applications
- Test with and without modifiers
- Test for conflicts with existing shortcuts
- Verify behavior across different window states

### 5. Document Changes
- **Update `/docs/mouse-hotkeys.md`** with hotkey details
- Include: key combination, behavior, context/app specific notes
- Add to hotkey reference table

## Best Practices

### Hotkey Assignment
- Use RAlt + F1-F12 for primary shortcuts
- Reserve commonly used keys (F1-F6) for frequent actions
- Use modifier combinations for related actions
- Consider muscle memory and ergonomics

### Context Detection
- Always use `WinGetProcessName("A")` for active window detection
- Use switch statements for multiple process checks
- Provide sensible defaults for unrecognized applications
- Test with common applications (browsers, editors, explorers)

### Send Command Usage
- `Send("{Key}")` - Simple key presses
- `Send("^{Key}")` - Ctrl modifier
- `Send("+{Key}")` - Shift modifier
- `Send("!{Key}")` - Alt modifier
- `Send("#{Key}")` - Win modifier

### Error Prevention
- Use try/catch for potentially failing operations
- Validate window existence before operations
- Handle edge cases (no active window, system windows, etc.)

## Global Variables

Mouse hotkey modules may use:
- `distraction_free` - Map for distraction-free mode state (defined in `hk_basic.ahk2`)

## Integration with Main Script

The mouse hotkey system is loaded via `setting_mouse.ahk2`:

```ahk
; In AWindowMovementSuite.ahk2:
#Include src\MouseHotkeys\setting_mouse.ahk2
```

This automatically includes all individual hotkey handler files.

## Common Use Cases

### Example: Adding Browser Tab Navigation
In `hk_tab.ahk2`:
```ahk
RAlt & F7::Send("^{Tab}")      ; Next tab
RAlt & F6::Send("^+{Tab}")     ; Previous tab
```

### Example: Adding Program Launcher
In `hk_programs.ahk2`:
```ahk
RAlt & F11::{
    if WinExist("ahk_exe notepad.exe") {
        WinActivate  ; Activate if already running
    }
    else {
        Run "notepad.exe"  ; Launch if not running
    }
}
```

## Troubleshooting

### Hotkey Not Working
1. Check for conflicts with other AHK scripts
2. Verify RAlt key is correctly detected
3. Test with simpler Send command
4. Check if application intercepts the hotkey

### Context Detection Failing
1. Verify process name with `WinGetProcessName("A")`
2. Check for window class instead if process name varies
3. Add debug ToolTip to display detected process

### Modifier Detection Issues
1. Use `GetKeyState("LCtrl", "P")` for physical state
2. Ensure modifiers are checked in correct order
3. Test with KeyHistory tool (`KeyHistory` command)

---

**Next Steps:**
- For detailed hotkey reference → `/docs/mouse-hotkeys.md`
- For main source overview → `../claude.md`
- For overall project → `../../claude.md`
