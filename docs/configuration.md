# Configuration System

Technical reference for the configuration management system in A Movement Suite.

## Overview

The configuration system uses a two-tier approach:
1. **GlobalConfig** - System defaults and constants (read-only)
2. **UserConfig** - User preferences and overrides (read-write, persistent)

## Architecture

```
Configuration Hierarchy:
    GlobalConfig (Defaults)
         ↓
    UserConfig (Overrides)
         ↓
    Runtime State (In-Memory)
```

---

## GlobalConfig (src/lib/Config/GlobalConfig.ahk2)

### Purpose
Defines system-wide defaults, constants, and configuration values that apply to all features.

### Structure
```ahk
; Example structure (actual implementation may vary)
class GlobalConfig {
    ; Window management
    static SnapDistance := 10
    static MinWindowWidth := 100
    static MinWindowHeight := 100

    ; Colors
    static Colors := {
        Preview: 0x3399FF,
        Border: 0xFF6600,
        Highlight: 0x00FF00
    }

    ; Feature defaults
    static Features := {
        WindowMove: true,
        WidthScaling: true,
        HeightScaling: true,
        XYScaling: true,
        AlwaysOnTop: true
    }
}
```

### Key Configuration Categories

#### Window Management Settings
- `SnapDistance` - Pixel threshold for edge snapping (default: 10)
- `MinWindowWidth` - Minimum allowed window width (default: 100)
- `MinWindowHeight` - Minimum allowed window height (default: 100)
- `CascadeOffset` - Offset for cascading windows (default: 30)

#### Color Definitions
- `Preview` - Preview GUI color
- `Border` - Border highlight color
- `Overlay` - Overlay transparency color
- Color format: 0xRRGGBB (hexadecimal RGB)

#### Feature Default States
- Initial enabled/disabled state for each feature
- Used on first run or config reset

### Usage in Code
```ahk
#Include %A_ScriptDir%\src\lib\Config\GlobalConfig.ahk2

; Access configuration values
snapDist := GlobalConfig.SnapDistance
previewColor := GlobalConfig.Colors.Preview
defaultState := GlobalConfig.Features.WindowMove
```

### File Location
`src/lib/Config/GlobalConfig.ahk2`

---

## UserConfig (src/lib/Config/UserConfig.ahk2)

### Purpose
Stores user-specific preferences that override GlobalConfig defaults. Persists across sessions.

### Structure
```ahk
; Example structure
class UserConfig {
    ; User preferences
    static CustomSnapDistance := 0  ; 0 = use GlobalConfig default
    static EnabledFeatures := []

    ; Methods
    static Load() {
        ; Load from settings file
    }

    static Save() {
        ; Save to settings file
    }

    static Get(key, default := "") {
        ; Get config value with fallback
    }

    static Set(key, value) {
        ; Set config value and mark dirty
    }
}
```

### Persistent Storage

#### Settings File Location
`settings/user_config.ini` or similar

#### File Format (INI)
```ini
[Window]
SnapDistance=15
MinWidth=150
MinHeight=100

[Features]
WindowMove=1
WidthScaling=1
HeightScaling=1
XYScaling=0

[Colors]
PreviewColor=0x3399FF

[Hotkeys]
DisableWindowMove=0
```

### Configuration Lifecycle

#### 1. Startup
```ahk
; On application start
UserConfig.Load()  ; Load from file

; Use config values
if (UserConfig.CustomSnapDistance > 0)
    snapDistance := UserConfig.CustomSnapDistance
else
    snapDistance := GlobalConfig.SnapDistance
```

#### 2. Runtime Modification
```ahk
; User changes setting via GUI
UserConfig.Set("SnapDistance", 15)
UserConfig.Save()  ; Persist to file
```

#### 3. Configuration Reset
```ahk
; Reset to defaults
UserConfig.Reset()
UserConfig.Save()

; Or delete settings file
FileDelete "settings\user_config.ini"
```

### File Location
`src/lib/Config/UserConfig.ahk2`

---

## Configuration Access Patterns

### 1. Direct Access Pattern
```ahk
; Simple, direct access to global config
snapDistance := GlobalConfig.SnapDistance
```

### 2. User Override Pattern
```ahk
; Check user config first, fall back to global
snapDistance := UserConfig.CustomSnapDistance > 0
    ? UserConfig.CustomSnapDistance
    : GlobalConfig.SnapDistance
```

### 3. Helper Function Pattern
```ahk
GetConfigValue(key, default := "") {
    ; Check user config first
    if UserConfig.Has(key)
        return UserConfig.Get(key)

    ; Fall back to global config
    if GlobalConfig.Has(key)
        return GlobalConfig.Get(key)

    ; Return default if not found
    return default
}

; Usage
snapDistance := GetConfigValue("SnapDistance", 10)
```

---

## Settings Storage

### Directory Structure
```
AMovementSuite/
├── settings/
│   ├── user_config.ini       # User preferences
│   ├── feature_states.ini    # Feature enable/disable states
│   └── hotkey_config.ini     # Custom hotkey mappings (future)
```

### File Management

#### Creating Settings Directory
```ahk
; Ensure settings directory exists
if !DirExist("settings")
    DirCreate "settings"
```

#### Reading INI Files
```ahk
; Read value from INI
IniRead, value, settings\user_config.ini, Window, SnapDistance, 10

; Or with AHK v2 syntax
value := IniRead("settings\user_config.ini", "Window", "SnapDistance", "10")
```

#### Writing INI Files
```ahk
; Write value to INI
IniWrite value, settings\user_config.ini, Window, SnapDistance

; Or with AHK v2 syntax
IniWrite(value, "settings\user_config.ini", "Window", "SnapDistance")
```

---

## Feature State Management

### Feature Toggle States

Each feature has a global variable tracking its enabled state:

```ahk
global windowMoveEnabled := true
global widthScalingEnabled := true
global heightScalingEnabled := true
global xyScalingEnabled := true
global alwaysOnTopEnabled := true
global windowRollUpEnabled := true
global windowDimmerEnabled := true
global windowCascadeEnabled := true
```

### Persisting Feature States

#### Save on Toggle
```ahk
ToggleWindowMove(*) {
    global windowMoveEnabled
    windowMoveEnabled := !windowMoveEnabled

    ; Update tray menu
    if (windowMoveEnabled)
        A_TrayMenu.Check "Window Move (LAlt+RButton)"
    else
        A_TrayMenu.Uncheck "Window Move (LAlt+RButton)"

    ; Persist state
    IniWrite(windowMoveEnabled, "settings\feature_states.ini", "Features", "WindowMove")
}
```

#### Load on Startup
```ahk
; Load feature states
LoadFeatureStates() {
    global windowMoveEnabled, widthScalingEnabled, heightScalingEnabled

    windowMoveEnabled := IniRead("settings\feature_states.ini", "Features", "WindowMove", "1") = "1"
    widthScalingEnabled := IniRead("settings\feature_states.ini", "Features", "WidthScaling", "1") = "1"
    heightScalingEnabled := IniRead("settings\feature_states.ini", "Features", "HeightScaling", "1") = "1"
    ; ... etc.
}
```

---

## Settings GUI Integration

### Window Move Settings Example

From `AWindowMovementSuite.ahk2:43`:
```ahk
A_TrayMenu.Add "Window Move Settings", ShowWindowMoveSettings
```

### Settings GUI Function
```ahk
ShowWindowMoveSettings(*) {
    ; Create settings GUI
    settingsGui := Gui("+AlwaysOnTop", "Window Move Settings")

    ; Add snap distance input
    settingsGui.Add("Text", "x10 y10", "Snap Distance (pixels):")
    snapInput := settingsGui.Add("Edit", "x150 y10 w100", GetConfigValue("SnapDistance", 10))

    ; Add save button
    saveBtn := settingsGui.Add("Button", "x10 y40 w100", "Save")
    saveBtn.OnEvent("Click", (*) => SaveSettings())

    SaveSettings() {
        newSnapDist := snapInput.Value
        UserConfig.Set("SnapDistance", newSnapDist)
        UserConfig.Save()
        settingsGui.Destroy()
    }

    settingsGui.Show()
}
```

See `ui-components.md` for more on settings GUIs.

---

## Configuration Best Practices

### 1. Use GlobalConfig for System Defaults
```ahk
; DO: Define in GlobalConfig
class GlobalConfig {
    static SnapDistance := 10
}

; DON'T: Hard-code in features
snapDistance := 10  ; Bad - should use config
```

### 2. Allow User Overrides
```ahk
; DO: Check user config first
snapDistance := UserConfig.CustomSnapDistance > 0
    ? UserConfig.CustomSnapDistance
    : GlobalConfig.SnapDistance

; DON'T: Ignore user preferences
snapDistance := GlobalConfig.SnapDistance  ; Bad - ignores user setting
```

### 3. Validate Configuration Values
```ahk
; DO: Validate input
SetSnapDistance(value) {
    if (value < 1 || value > 100) {
        MsgBox "Snap distance must be between 1 and 100"
        return false
    }

    UserConfig.Set("SnapDistance", value)
    return true
}

; DON'T: Accept invalid values
SetSnapDistance(value) {
    UserConfig.Set("SnapDistance", value)  ; Bad - no validation
}
```

### 4. Provide Sensible Defaults
```ahk
; DO: Always provide defaults
value := GetConfigValue("SnapDistance", 10)  ; Good - has default

; DON'T: Assume config exists
value := GetConfigValue("SnapDistance")  ; Bad - might return empty
```

### 5. Persist Important Settings
```ahk
; DO: Save after user changes
UserConfig.Set("SnapDistance", newValue)
UserConfig.Save()  ; Persist immediately

; DON'T: Lose user changes
UserConfig.Set("SnapDistance", newValue)  ; Bad - not saved
```

---

## Adding New Configuration Options

### Step-by-Step Guide

1. **Add to GlobalConfig (Default Value)**
   ```ahk
   ; src/lib/Config/GlobalConfig.ahk2
   class GlobalConfig {
       static NewFeatureOption := "default_value"
   }
   ```

2. **Add to UserConfig (User Override)**
   ```ahk
   ; src/lib/Config/UserConfig.ahk2
   class UserConfig {
       static NewFeatureOption := ""  ; Empty = use default
   }
   ```

3. **Add to Settings File Structure**
   ```ini
   ; settings/user_config.ini
   [NewFeature]
   FeatureOption=custom_value
   ```

4. **Implement Load/Save**
   ```ahk
   ; In UserConfig.Load()
   UserConfig.NewFeatureOption := IniRead(
       "settings\user_config.ini",
       "NewFeature",
       "FeatureOption",
       ""
   )

   ; In UserConfig.Save()
   IniWrite(
       UserConfig.NewFeatureOption,
       "settings\user_config.ini",
       "NewFeature",
       "FeatureOption"
   )
   ```

5. **Create Settings GUI (if user-configurable)**
   ```ahk
   ShowNewFeatureSettings(*) {
       ; Create settings GUI for new option
       ; See ui-components.md for details
   }
   ```

6. **Use in Feature Code**
   ```ahk
   ; In feature module
   optionValue := UserConfig.NewFeatureOption != ""
       ? UserConfig.NewFeatureOption
       : GlobalConfig.NewFeatureOption
   ```

7. **Update Documentation**
   - Add to this file (configuration.md)
   - Document default value
   - Document valid ranges/options
   - Add to settings GUI documentation

---

## Configuration Migration

### Handling Config Version Changes

```ahk
class ConfigMigration {
    static CurrentVersion := "1.1"

    static Migrate() {
        configVersion := IniRead("settings\user_config.ini", "Meta", "Version", "1.0")

        if (configVersion = "1.0") {
            this.MigrateFrom1_0To1_1()
            configVersion := "1.1"
        }

        ; Save updated version
        IniWrite(configVersion, "settings\user_config.ini", "Meta", "Version")
    }

    static MigrateFrom1_0To1_1() {
        ; Rename old config keys
        oldValue := IniRead("settings\user_config.ini", "Window", "OldKeyName", "")
        if (oldValue != "") {
            IniWrite(oldValue, "settings\user_config.ini", "Window", "NewKeyName")
            IniDelete("settings\user_config.ini", "Window", "OldKeyName")
        }
    }
}

; Run on startup
ConfigMigration.Migrate()
```

---

## Troubleshooting

### Settings Not Persisting

**Check:**
1. Settings directory exists and is writable
2. INI file exists and has correct permissions
3. `UserConfig.Save()` is called after changes
4. No errors during file write operations

**Debug:**
```ahk
try {
    UserConfig.Save()
    MsgBox "Settings saved successfully"
}
catch Error as err {
    MsgBox "Failed to save settings: " err.Message
}
```

### Settings Not Loading

**Check:**
1. Settings file exists in expected location
2. INI section and key names match exactly
3. Default values provided for missing keys
4. File format is valid INI

**Debug:**
```ahk
value := IniRead("settings\user_config.ini", "Window", "SnapDistance", "NOT_FOUND")
MsgBox "Loaded value: " value
```

### Invalid Configuration Values

**Check:**
1. Validation logic in place
2. Type conversions correct (string to number, etc.)
3. Range checks for numeric values
4. Enum validation for string values

**Solution:**
```ahk
; Add validation
SetSnapDistance(value) {
    ; Convert to number
    value := Integer(value)

    ; Validate range
    if (value < 1 || value > 100) {
        MsgBox "Invalid snap distance. Using default: 10"
        value := 10
    }

    UserConfig.Set("SnapDistance", value)
}
```

---

## Performance Considerations

### Minimize File I/O
```ahk
; DO: Batch saves
UserConfig.Set("Option1", value1)
UserConfig.Set("Option2", value2)
UserConfig.Set("Option3", value3)
UserConfig.Save()  ; Single save operation

; DON'T: Save after each change
UserConfig.Set("Option1", value1)
UserConfig.Save()
UserConfig.Set("Option2", value2)
UserConfig.Save()  ; Multiple saves - slower
```

### Cache Configuration Values
```ahk
; DO: Cache frequently accessed values
static snapDistance := GetConfigValue("SnapDistance", 10)

; DON'T: Read from config every time
; (in hot path like mouse move handler)
snapDistance := GetConfigValue("SnapDistance", 10)  ; Slow if called repeatedly
```

---

**See Also:**
- `architecture.md` - System architecture
- `ui-components.md` - Settings GUI components
- `features.md` - Feature configuration options
