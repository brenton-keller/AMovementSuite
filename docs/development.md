# Development Guide

Comprehensive guide for developing A Movement Suite.

## Table of Contents
1. [Development Setup](#development-setup)
2. [Project Structure](#project-structure)
3. [Development Tools](#development-tools)
4. [Debugging](#debugging)
5. [Common Development Tasks](#common-development-tasks)
6. [Best Practices](#best-practices)
7. [Troubleshooting](#troubleshooting)

---

## Development Setup

### Prerequisites

**Required:**
- Windows 10 (Build 10240+) or Windows 11
- AutoHotkey v2.0 or later
- Text editor with AHK syntax support

**Recommended:**
- Git for version control
- VS Code with AutoHotkey Plus Plus extension
- Multi-monitor setup for testing

### Installation

1. **Install AutoHotkey v2.0**
   ```
   Download: https://www.autohotkey.com/
   Install to: C:\Program Files\AutoHotkey (default)
   Verify: Run AutoHotkey64.exe --version
   ```

2. **Clone Repository**
   ```bash
   git clone https://github.com/yourusername/AMovementSuite.git
   cd AMovementSuite
   ```

3. **Configure Editor**

   **For VS Code:**
   - Install "AutoHotkey Plus Plus" extension
   - Configure syntax highlighting for .ahk2 files
   - Set default formatter

   **For Sublime Text:**
   - Install "AutoHotkey" package via Package Control
   - Configure build system for AHK v2

4. **Run Application**
   ```
   Right-click AWindowMovementSuite.ahk2 → Run Script
   Or: Double-click AWindowMovementSuite.ahk2
   ```

---

## Project Structure

```
AMovementSuite/
├── AWindowMovementSuite.ahk2     # Main entry point
├── claude.md                      # Project documentation hub
├── README.md                      # User documentation
├── LICENSE                        # MIT License
├── .gitignore                     # Git ignore rules
│
├── src/                           # Source code
│   ├── claude.md                  # Source overview
│   ├── Features/                  # Window features
│   ├── MouseHotkeys/              # Mouse hotkey system
│   │   └── claude.md              # Hotkey development guide
│   ├── lib/                       # Core libraries
│   │   ├── Config/                # Configuration management
│   │   ├── Core/                  # Core utilities
│   │   └── Helpers/               # Helper functions
│   ├── UI/                        # User interface components
│   └── VirtualDesktopLib/         # Virtual desktop integration
│
├── docs/                          # Technical documentation
│   ├── architecture.md            # System architecture
│   ├── features.md                # Feature reference
│   ├── mouse-hotkeys.md           # Hotkey reference
│   ├── configuration.md           # Config system
│   ├── ui-components.md           # UI documentation
│   ├── virtual-desktops.md        # Virtual desktop API
│   ├── contributing.md            # Contribution guide
│   └── development.md             # This file
│
├── settings/                      # User configuration files
│   ├── user_config.ini            # User preferences
│   └── feature_states.ini         # Feature toggle states
│
└── assets/                        # Resources
    └── images/                    # Screenshots, GIFs
```

---

## Development Tools

### AutoHotkey Built-in Tools

#### 1. Window Spy (Au3_Spy.exe)
**Location:** `C:\Program Files\AutoHotkey\UX\WindowSpy.ahk`

**Usage:**
- Identify window classes
- Get window handles
- Inspect control information
- Find process names

**Launch:** AutoHotkey system tray → Window Spy

#### 2. KeyHistory
**Usage:**
- View recent keystrokes
- Debug hotkey issues
- Check modifier states

**Access:** Right-click AHK tray icon → Open → View → Key history and script info

#### 3. ListLines
**Usage:**
- View recently executed script lines
- Debug execution flow
- Find infinite loops

**Enable temporarily:**
```ahk
ListLines 1  ; Enable
; ... code to debug ...
ListLines 0  ; Disable
```

### External Tools

#### VS Code Extensions
- **AutoHotkey Plus Plus** - Syntax highlighting, IntelliSense
- **Debugger for AutoHotkey** - Interactive debugging
- **AHK++ Formatter** - Code formatting

#### Other Tools
- **Git** - Version control
- **GitHub Desktop** - Git GUI (optional)
- **Notepad++** - Lightweight editor alternative
- **Process Explorer** - Process monitoring

---

## Debugging

### Debug Output Methods

#### 1. ToolTip (Quick Debug)
```ahk
; Display variable value
ToolTip "windowID: " windowID
SetTimer () => ToolTip(), -3000  ; Auto-hide after 3s

; Multiple values
ToolTip "X: " x " Y: " y " Width: " width " Height: " height

; Conditional debug
if (debugMode)
    ToolTip "Debug: Entering function"
```

#### 2. MsgBox (Blocking Debug)
```ahk
; Simple value display
MsgBox "Current value: " value

; Detailed info
MsgBox "
(
Window ID: " windowID "
Position: " x ", " y "
Size: " width " x " height "
)"

; Error display
MsgBox "Error: " errorMessage, "Error", "Icon!"
```

#### 3. OutputDebug (Debug Console)
```ahk
; Send to debug console
OutputDebug "Function called with param: " param

; View with DebugView (SysInternals)
; Or VS Code debug console
```

#### 4. File Logging
```ahk
DebugLog(message) {
    FileAppend A_Now " - " message "`n", "debug.log"
}

; Usage
DebugLog("Window moved to: " x ", " y)
```

### Debugging Techniques

#### Isolate Issues
```ahk
; Comment out code sections to isolate problem
; DoFeatureA()
DoFeatureB()  ; Only this runs
; DoFeatureC()
```

#### Add Breakpoints (Manual)
```ahk
; Pause execution
MsgBox "Breakpoint 1: Before operation"
PerformOperation()
MsgBox "Breakpoint 2: After operation"
```

#### Trace Execution
```ahk
FunctionA() {
    MsgBox "Entering FunctionA"
    result := DoSomething()
    MsgBox "FunctionA result: " result
    return result
}
```

#### Check Variable State
```ahk
; Before operation
MsgBox "Before: value = " value

; Perform operation
value := value * 2

; After operation
MsgBox "After: value = " value
```

### Common Debug Scenarios

#### Window Not Found
```ahk
MouseGetPos , , &windowID
if (!WinExist("ahk_id " windowID)) {
    MsgBox "Window does not exist: " windowID
    return
}
MsgBox "Window found: " WinGetTitle("ahk_id " windowID)
```

#### Hotkey Not Triggering
```ahk
RAlt & F1::{
    MsgBox "Hotkey triggered!"  ; Add this to verify
    ; ... rest of code ...
}
```

#### Configuration Not Loading
```ahk
value := IniRead("settings\config.ini", "Section", "Key", "DEFAULT")
MsgBox "Loaded value: " value " (Default: DEFAULT)"
```

---

## Common Development Tasks

### Adding a New Window Feature

**1. Create Feature File**
```ahk
; src/Features/WindowNewFeature.ahk2

; Global toggle state
global newFeatureEnabled := true

; Hotkey handler
HotkeyCombo::{
    if (!newFeatureEnabled)
        return

    MouseGetPos , , &windowID

    if (!IsValidWindowForNewFeature(windowID))
        return

    ; Feature logic here
    PerformNewFeature(windowID)
}

; Window validation
IsValidWindowForNewFeature(winID) {
    if (!WinExist("ahk_id " winID))
        return false

    winClass := WinGetClass("ahk_id " winID)
    excludedClasses := ["Shell_TrayWnd", "Progman", "WorkerW"]

    for class in excludedClasses {
        if (winClass = class)
            return false
    }

    return true
}

; Feature implementation
PerformNewFeature(winID) {
    ; Implementation here
}

; Toggle function for tray menu
ToggleNewFeature(*) {
    global newFeatureEnabled
    newFeatureEnabled := !newFeatureEnabled

    if (newFeatureEnabled)
        A_TrayMenu.Check "New Feature (Hotkey)"
    else
        A_TrayMenu.Uncheck "New Feature (Hotkey)"
}
```

**2. Include in Main Script**
```ahk
; In AWindowMovementSuite.ahk2, after other #Include statements
#Include src\Features\WindowNewFeature.ahk2
```

**3. Add Tray Menu Entry**
```ahk
; In AWindowMovementSuite.ahk2, in tray menu section
A_TrayMenu.Add "New Feature (Hotkey)", ToggleNewFeature
A_TrayMenu.Add  ; Separator (optional)
```

**4. Initialize Menu State**
```ahk
; In AWindowMovementSuite.ahk2, after menu creation
A_TrayMenu.Check "New Feature (Hotkey)"
```

**5. Update Documentation**
- Add to `docs/features.md`
- Update `README.md` if user-facing
- Update `claude.md` if needed

### Adding a New Mouse Hotkey

**1. Choose Appropriate File**
- Basic navigation → `hk_basic.ahk2`
- Code editor → `hk_code.ahk2`
- Program launching → `hk_programs.ahk2`
- Tab operations → `hk_tab.ahk2`

**2. Add Hotkey Definition**
```ahk
; In appropriate hk_*.ahk2 file
RAlt & F10::{
    ; Get active application
    ActiveProcess := WinGetProcessName("A")

    ; Context-aware behavior
    switch ActiveProcess {
        case "msedge.exe", "chrome.exe", "firefox.exe":
            Send("^t")  ; Browser: New tab
        case "Code.exe":
            Send("^n")  ; VS Code: New file
        default:
            Send("{F10}")  ; Default: F10
    }
}
```

**3. Add Modifier Support (Optional)**
```ahk
RAlt & F10::{
    if GetKeyState("LCtrl", "P") {
        ; Ctrl variant
        Send("^+{F10}")
    }
    else if GetKeyState("LShift", "P") {
        ; Shift variant
        Send("+{F10}")
    }
    else {
        ; Default
        Send("{F10}")
    }
}
```

**4. Update Documentation**
- Add to `docs/mouse-hotkeys.md`
- Update hotkey reference table

### Adding a Configuration Option

**1. Add to GlobalConfig**
```ahk
; In src/lib/Config/GlobalConfig.ahk2
class GlobalConfig {
    static NewOption := "default_value"
    ; ... other options ...
}
```

**2. Add to UserConfig (if user-configurable)**
```ahk
; In src/lib/Config/UserConfig.ahk2
class UserConfig {
    static NewOption := ""  ; Empty = use default

    static Load() {
        ; ... existing code ...
        UserConfig.NewOption := IniRead(
            "settings\user_config.ini",
            "Section",
            "NewOption",
            ""
        )
    }

    static Save() {
        ; ... existing code ...
        if (UserConfig.NewOption != "")
            IniWrite(
                UserConfig.NewOption,
                "settings\user_config.ini",
                "Section",
                "NewOption"
            )
    }
}
```

**3. Create Settings GUI (if needed)**
```ahk
ShowNewOptionSettings(*) {
    gui := Gui("+AlwaysOnTop", "New Option Settings")

    gui.Add("Text", , "New Option:")
    input := gui.Add("Edit", "w200", GetConfigValue("NewOption"))

    saveBtn := gui.Add("Button", , "Save")
    saveBtn.OnEvent("Click", (*) => SaveNewOption())

    SaveNewOption() {
        UserConfig.NewOption := input.Value
        UserConfig.Save()
        gui.Destroy()
    }

    gui.Show()
}
```

**4. Update Documentation**
- Add to `docs/configuration.md`
- Document defaults and valid values

### Modifying Existing Code

**1. Read Existing Code**
- Understand current implementation
- Check for dependencies
- Note any special cases

**2. Make Changes**
- Follow existing patterns
- Maintain code style
- Add comments for clarity

**3. Test Thoroughly**
- Test new functionality
- Test existing functionality (regression testing)
- Test edge cases

**4. Update Documentation**
- Update relevant docs/
- Update inline comments
- Note any breaking changes

---

## Best Practices

### Code Organization

```ahk
; Good: Grouped by functionality
; Global variables
global feature1Enabled := true
global feature2Enabled := true

; Helper functions
HelperFunction1() { }
HelperFunction2() { }

; Main functionality
Hotkey1::{}
Hotkey2::{}

; Toggle functions
ToggleFeature1(*) { }
ToggleFeature2(*) { }
```

### Error Handling

```ahk
; Good: Graceful error handling
try {
    WinMove x, y, width, height, "ahk_id " windowID
}
catch Error as err {
    ToolTip "Failed to move window: " err.Message
    SetTimer () => ToolTip(), -3000
}

; Bad: No error handling
WinMove x, y, width, height, "ahk_id " windowID  ; May crash
```

### Performance

```ahk
; Good: Cache values
static snapDistance := GetConfigValue("SnapDistance", 10)

; Bad: Repeated calls
snapDistance := GetConfigValue("SnapDistance", 10)  ; Every time
```

### Documentation

```ahk
; Good: Clear, concise comments
; Check if window is valid for moving
if (!IsValidWindowForMove(windowID))
    return

; Bad: Obvious comments
x := 10  ; Set x to 10
```

---

## Troubleshooting

### Script Won't Run

**Check:**
1. AutoHotkey v2.0 is installed
2. File extension is .ahk2
3. No syntax errors (check AHK error dialog)
4. #Requires directive is correct

**Debug:**
```ahk
; Add at top of script
MsgBox "Script starting..."
```

### Hotkeys Not Working

**Check:**
1. No conflicting AHK scripts running
2. Hotkey syntax is correct
3. Feature is enabled
4. Application isn't blocking hotkey

**Debug:**
```ahk
; Simplify hotkey to test
RAlt & F1::MsgBox "Hotkey works!"
```

### Features Interfering

**Check:**
1. Multiple features modifying same windows
2. Hotkey conflicts
3. State management issues

**Solution:**
- Add feature exclusivity checks
- Coordinate hotkey assignments
- Implement proper state cleanup

### Performance Issues

**Check:**
1. Unnecessary WinGet* calls in loops
2. Missing SetWinDelay -1
3. Excessive ToolTip/MsgBox in production

**Solution:**
- Cache window information
- Use performance settings
- Remove debug output

---

## Testing Workflow

### Development Testing

1. **Make changes**
2. **Reload script** (right-click tray icon → Reload Script)
3. **Test feature**
4. **Check for errors**
5. **Repeat**

### Pre-Commit Testing

1. Test all modified features
2. Test related features (regression)
3. Test on different window types
4. Test on multi-monitor (if applicable)
5. Check documentation accuracy

### Release Testing

1. Fresh install test
2. Upgrade test (from previous version)
3. Multi-monitor testing
4. Windows 10 and 11 testing
5. Long-term stability test (run for extended period)

---

## Useful Code Snippets

### Get Window Under Mouse
```ahk
MouseGetPos &x, &y, &windowID
windowTitle := WinGetTitle("ahk_id " windowID)
windowClass := WinGetClass("ahk_id " windowID)
processName := WinGetProcessName("ahk_id " windowID)
```

### Get Monitor Information
```ahk
monitorCount := MonitorGetCount()
primaryMonitor := MonitorGetPrimary()
MonitorGet(1, &left, &top, &right, &bottom)
MonitorGetWorkArea(1, &workLeft, &workTop, &workRight, &workBottom)
```

### Safe Window Move
```ahk
SafeWinMove(windowID, x, y, width, height) {
    if (!WinExist("ahk_id " windowID))
        return false

    try {
        WinMove x, y, width, height, "ahk_id " windowID
        return true
    }
    catch Error as err {
        ToolTip "Move failed: " err.Message
        SetTimer () => ToolTip(), -2000
        return false
    }
}
```

### Validate Numeric Input
```ahk
ValidateNumber(value, min, max) {
    if (!IsNumber(value))
        return false

    numValue := Integer(value)
    return (numValue >= min && numValue <= max)
}
```

---

**See Also:**
- `contributing.md` - Contribution guidelines
- `architecture.md` - System architecture
- `features.md` - Feature reference
- AutoHotkey v2 documentation: https://www.autohotkey.com/docs/v2/
