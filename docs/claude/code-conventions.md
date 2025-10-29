# Code Conventions

## Naming Conventions

- **Functions**: PascalCase (e.g., `IsValidWindowForResize()`)
- **Variables**: camelCase (e.g., `windowMoveEnabled`)
- **Constants**: UPPER_CASE (e.g., `MOTION_PAUSE_THRESHOLD`)
- **Global Variables**: Prefix with descriptive names (e.g., `windowMoveEnabled`, not just `enabled`)

## File Organization

- **One feature per file**: Each feature module is self-contained
- **Related utilities grouped**: Core utilities organized by function in `lib/`
- **Hierarchical includes**: Features include core libraries as needed

## Common Patterns

### Feature Toggle Pattern
```ahk
global featureEnabled := true  ; Default enabled state

; Hotkey handler checks enabled state
Hotkey::{
    if (!featureEnabled)
        return
    ; Feature logic...
}

; Toggle function for tray menu
ToggleFeature(*) {
    global featureEnabled
    featureEnabled := !featureEnabled
    ; Update tray menu check...
}
```

### Window Validation Pattern
```ahk
IsValidWindowForFeature(winID) {
    ; Check if window exists
    if (!WinExist("ahk_id " winID))
        return false

    ; Exclude system windows, desktop, etc.
    winClass := WinGetClass("ahk_id " winID)
    excludedClasses := ["Shell_TrayWnd", "Progman", "WorkerW"]

    for class in excludedClasses {
        if (winClass = class)
            return false
    }

    return true
}
```

### Mouse Position & Window Detection
```ahk
MouseGetPos &mouseX, &mouseY, &windowID
WinGetPos &winX, &winY, &winWidth, &winHeight, "ahk_id " windowID
```

### Configuration Access
```ahk
; Access global config values
snapDistance := GlobalConfig.SnapDistance
previewColor := GlobalConfig.Colors.Preview
```

## Global State Management

- **Feature toggle states**: Stored as global variables
- **Configuration access**: Through GlobalConfig/UserConfig objects
- **Minimize global state**: Prefer parameters over globals

## Hotkey Scope

- **Most hotkeys**: Global (system-wide)
- **Application-specific**: Use `#HotIf` directives
- **Mouse hotkeys**: Check modifier keys for variant behaviors

## Error Handling

- Use `try/catch` for potentially failing operations
- Window operations may fail if window closes mid-operation
- Always validate windows before manipulation

## Comments

- Document non-obvious logic and function purposes
- Explain complex interactions and algorithms
- Keep comments updated with code changes

## AutoHotkey v2.0 Specifics

- All code uses AHK v2.0 syntax
- Use modern v2 features (e.g., fat arrow functions `=>`)
- Consistent indentation (4 spaces or tabs)
