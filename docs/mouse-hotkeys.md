# Mouse Hotkey System

Complete technical reference for the mouse hotkey system in A Movement Suite.

## System Overview

The mouse hotkey system provides context-aware keyboard shortcuts using RAlt + Function keys with modifier support. Different applications receive different commands for the same hotkey combination.

**Base Pattern:** `RAlt + F1-F12` with optional `LCtrl`, `LShift`, or `LAlt` modifiers

## Module Organization

| Module | Purpose | File Location |
|--------|---------|---------------|
| Loader | Includes all hotkey modules | src/MouseHotkeys/setting_mouse.ahk2 |
| Basic Navigation | Backspace, Enter, Delete, Arrows | src/MouseHotkeys/hk_basic.ahk2 |
| Code Editor | Programming shortcuts | src/MouseHotkeys/hk_code.ahk2 |
| Program Launcher | Application launching | src/MouseHotkeys/hk_programs.ahk2 |
| Tab Management | Tab navigation & control | src/MouseHotkeys/hk_tab.ahk2 |
| Mouse Wheel | Scroll customization | src/MouseHotkeys/hk_wheel.ahk2 |
| Windows Explorer | File manager shortcuts | src/MouseHotkeys/windows_explorer.ahk2 |
| Startbar | Taskbar interactions | src/MouseHotkeys/Startbar.ahk2 |
| Transparency | Window transparency toggle | src/MouseHotkeys/hk_transparency.ahk2 |
| WezTerm | Terminal shortcuts | src/MouseHotkeys/hk_wezterm.ahk2 |

---

## Hotkey Reference

### hk_basic.ahk2 - Basic Navigation

#### RAlt + F1: Backspace Variants

| Modifiers | Browsers/XYplorer | Other Applications |
|-----------|-------------------|-------------------|
| None | `Backspace` | `Backspace` |
| LCtrl | `Alt+Left` (Back) | `Ctrl+Backspace` |
| LShift | `Alt+Left` (Back) | `Shift+Backspace` |
| LAlt | `Alt+Left` (Back) | `Alt+Backspace` |

**Context Applications:**
- `msedge.exe` (Microsoft Edge)
- `brave.exe` (Brave Browser)
- `opera.exe` (Opera)
- `XYplorer.exe` (File Manager)
- `Vivaldi.exe` (Vivaldi Browser)

**Source:** `src/MouseHotkeys/hk_basic.ahk2:4-44`

#### RAlt + F2: Enter Variants

| Modifiers | Browsers/XYplorer | Other Applications |
|-----------|-------------------|-------------------|
| None | `Enter` | `Enter` |
| LCtrl | `Alt+Right` (Forward) | `Ctrl+Enter` |
| LShift | `Alt+Right` (Forward) | `Shift+Enter` |
| LAlt | `Alt+Right` (Forward) | `Enter` |

**Source:** `src/MouseHotkeys/hk_basic.ahk2:46-87`

### Additional Basic Hotkeys

The `hk_basic.ahk2` file likely contains additional navigation shortcuts for:
- Delete key variants
- Arrow key combinations
- Page Up/Down
- Home/End

**Pattern:** Each follows the same modifier detection pattern with context-aware behavior.

---

## hk_code.ahk2 - Code Editor Shortcuts

### Purpose
Optimized shortcuts for programming and text editing workflows, plus system-wide volume control.

### Common IDE/Editor Targets
- `Code.exe` (Visual Studio Code)
- `cursor.exe` (Cursor)
- `devenv.exe` (Visual Studio)
- `sublime_text.exe` (Sublime Text)
- `notepad++.exe` (Notepad++)

### Hotkey Reference

#### RAlt + F7: Go to Definition
| Modifiers | Behavior |
|-----------|----------|
| None | Go to definition (`F12`) |
| LCtrl | Go to definition (`F12`) |

#### RAlt + F10: Unfold / Mute Toggle
| Modifiers | Behavior |
|-----------|----------|
| None | Unfold code section (`Ctrl+Shift+]`) |
| LCtrl | Unfold all (`Ctrl+K, Ctrl+J`) |
| LAlt | **Toggle system mute** with visual tooltip |

#### RAlt + F11: Comment Toggle / Volume Down
| Modifiers | Behavior |
|-----------|----------|
| None | Toggle comment in VS Code/Cursor (`Ctrl+/`) |
| LAlt | **Decrease system volume by 1%** with visual tooltip |

### Volume Control Feature
The LAlt modifier transforms F10/F11 into system-wide volume controls:
- Shows centered tooltip with current volume percentage
- Mute toggle shows "Volume: MUTED" when muted
- Tooltips auto-dismiss after 1 second

**Source:** `src/MouseHotkeys/hk_code.ahk2`

---

## hk_programs.ahk2 - Program Launcher

### Purpose
Quick launching and activation of frequently used applications.

### Pattern
```ahk
RAlt & F{N}::{
    programPath := "C:\Program Files\Application\app.exe"
    programClass := "ahk_exe app.exe"

    if WinExist(programClass) {
        WinActivate  ; Activate if already running
    }
    else {
        Run programPath  ; Launch if not running
    }
}
```

### Benefits
- Single hotkey to launch or switch to application
- No duplicate instances
- Fast application switching

**Source:** `src/MouseHotkeys/hk_programs.ahk2`

---

## hk_tab.ahk2 - Tab Management

### Purpose
Navigate and manage tabs in browsers and multi-tab applications.

### Common Operations

#### Tab Navigation
- `Ctrl+Tab` - Next tab
- `Ctrl+Shift+Tab` - Previous tab
- `Ctrl+1-8` - Jump to specific tab
- `Ctrl+9` - Last tab

#### Tab Management
- `Ctrl+T` - New tab
- `Ctrl+W` - Close tab
- `Ctrl+Shift+T` - Reopen closed tab

### Target Applications
- Web browsers (Chrome, Firefox, Edge, Opera, Brave, Vivaldi)
- File managers with tabs
- Code editors with tab support
- Terminal applications

**Source:** `src/MouseHotkeys/hk_tab.ahk2`

---

## hk_wheel.ahk2 - Mouse Wheel Customization

### Purpose
Enhance mouse wheel functionality with modifiers and context.

### Common Customizations

#### Horizontal Scrolling
```ahk
Shift + Wheel Up/Down → Horizontal scroll
```

#### Zoom Control
```ahk
Ctrl + Wheel Up/Down → Zoom in/out
```

#### Volume Control
```ahk
Alt + Wheel Up/Down → Volume adjustment
```

#### Context-Specific Wheel
Different wheel behavior per application (e.g., faster scrolling in code editors, precision scrolling in image editors).

**Source:** `src/MouseHotkeys/hk_wheel.ahk2`

---

## windows_explorer.ahk2 - File Explorer

### Purpose
Enhanced file navigation and operations in Windows Explorer and file managers.

### Target Applications
- `explorer.exe` (Windows Explorer)
- `XYplorer.exe`
- `TotalCmd.exe` (Total Commander)
- `dopus.exe` (Directory Opus)

### Common Operations
- Quick navigation shortcuts
- File operations (copy, move, delete)
- Folder navigation (up, back, forward)
- View switching
- Search activation

**Source:** `src/MouseHotkeys/windows_explorer.ahk2`

---

## Startbar.ahk2 - Taskbar Interactions

### Purpose
Quick access to taskbar and system tray functionality.

### Common Operations
- Show/hide taskbar
- Access system tray icons
- Quick launch pinned applications
- Window switching via taskbar
- Virtual desktop navigation

**Source:** `src/MouseHotkeys/Startbar.ahk2`

---

## hk_transparency.ahk2 - Window Transparency

### Purpose
Temporarily make all visible windows semi-transparent to see through them.

### Hotkey
`Alt + \`` (backtick) - Hold to activate transparency

### Behavior
1. **Press and hold** `Alt + \`` to activate
2. All valid windows become 70% transparent (opacity 178/255)
3. Orange indicator appears in top-left corner
4. Tooltip shows count of affected windows
5. **Release keys** to restore all windows to full opacity

### Features
- Uses shared `GetValidWindows()` function for consistent window filtering
- Excludes system windows, taskbar, desktop, own script windows
- Visual indicator GUI while active
- Graceful error handling for protected windows

### Global Variables
- `transparencyEnabled` - State tracking variable

### Use Cases
- Quickly peek at windows behind the current view
- Find hidden content without rearranging windows
- Visual debugging of window positions

**Source:** `src/MouseHotkeys/hk_transparency.ahk2`

---

## hk_wezterm.ahk2 - WezTerm Terminal

### Purpose
WezTerm-specific shortcuts and smart terminal launching.

### Features
- Terminal tab management
- Smart directory detection for new terminals
- WezTerm-specific key mappings

**Source:** `src/MouseHotkeys/hk_wezterm.ahk2`

---

## Global Variables

### distraction_free Map
```ahk
global distraction_free := Map()
```

**Purpose:** Track distraction-free mode state per application or window.

**Usage:**
```ahk
distraction_free[windowID] := true  ; Enable for window
if distraction_free.Has(windowID) {
    ; Window is in distraction-free mode
}
```

**Source:** `src/MouseHotkeys/hk_basic.ahk2:1`

---

## Implementation Patterns

### 1. Modifier Detection Pattern

```ahk
RAlt & F{N}::{
    if GetKeyState("LCtrl", "P") {
        ; Ctrl variant
        Send("^{Action}")
    }
    else if GetKeyState("LShift", "P") {
        ; Shift variant
        Send("+{Action}")
    }
    else if GetKeyState("LAlt", "P") {
        ; Alt variant
        Send("!{Action}")
    }
    else {
        ; No modifiers
        Send("{Action}")
    }
}
```

**Key Points:**
- Use `GetKeyState(key, "P")` for physical state
- Check in order: Ctrl → Shift → Alt → None
- Use `Send()` with modifier symbols: `^` (Ctrl), `+` (Shift), `!` (Alt), `#` (Win)

### 2. Context Detection Pattern

```ahk
RAlt & F{N}::{
    ActiveProcess := WinGetProcessName("A")

    switch ActiveProcess {
        case "msedge.exe", "brave.exe", "opera.exe":
            Send("^t")  ; Browser-specific action
        case "Code.exe":
            Send("^p")  ; VS Code specific action
        default:
            Send("{F3}")  ; Default action
    }
}
```

**Key Points:**
- `WinGetProcessName("A")` gets active window process
- Use `switch` for multiple process checks
- Always provide `default` case
- List related processes together (all browsers, all file managers, etc.)

### 3. Launch or Activate Pattern

```ahk
RAlt & F{N}::{
    targetExe := "ahk_exe application.exe"
    targetPath := "C:\Path\To\Application.exe"

    if WinExist(targetExe) {
        WinActivate
    }
    else {
        Run targetPath
    }
}
```

**Key Points:**
- Check existence before launching
- Use `WinActivate` without parameters (acts on last found window)
- Handle errors with try/catch for Run command

### 4. Conditional Hotkey Pattern

```ahk
#HotIf WinActive("ahk_exe explorer.exe")
RButton::CustomExplorerAction()
#HotIf
```

**Key Points:**
- Use `#HotIf` for application-specific hotkeys
- Close with `#HotIf` (no condition) to end scope
- Useful for overriding default behaviors in specific apps

---

## Application Process Names Reference

### Browsers
- `msedge.exe` - Microsoft Edge
- `chrome.exe` - Google Chrome
- `firefox.exe` - Mozilla Firefox
- `brave.exe` - Brave
- `opera.exe` - Opera
- `Vivaldi.exe` - Vivaldi
- `iexplore.exe` - Internet Explorer (legacy)

### Code Editors
- `Code.exe` - Visual Studio Code
- `devenv.exe` - Visual Studio
- `sublime_text.exe` - Sublime Text
- `notepad++.exe` - Notepad++
- `atom.exe` - Atom
- `idea64.exe` - IntelliJ IDEA

### File Managers
- `explorer.exe` - Windows Explorer
- `XYplorer.exe` - XYplorer
- `TotalCmd.exe` - Total Commander
- `dopus.exe` - Directory Opus

### Terminals
- `WindowsTerminal.exe` - Windows Terminal
- `cmd.exe` - Command Prompt
- `powershell.exe` - PowerShell
- `pwsh.exe` - PowerShell Core

---

## Send Command Reference

### Modifier Symbols
- `^` - Ctrl
- `+` - Shift
- `!` - Alt
- `#` - Win

### Special Keys
- `{Enter}` - Enter key
- `{Backspace}` - Backspace
- `{Delete}` - Delete
- `{Tab}` - Tab
- `{Esc}` - Escape
- `{Space}` - Space
- `{Up}`, `{Down}`, `{Left}`, `{Right}` - Arrow keys
- `{Home}`, `{End}`, `{PgUp}`, `{PgDn}` - Navigation keys
- `{F1}`-`{F24}` - Function keys

### Combined Examples
```ahk
Send("^c")           ; Ctrl+C (copy)
Send("^+s")          ; Ctrl+Shift+S (save as)
Send("!{F4}")        ; Alt+F4 (close window)
Send("#{Left}")      ; Win+Left (snap left)
Send("^{Home}")      ; Ctrl+Home (document start)
```

---

## Troubleshooting

### Hotkey Not Triggering

**Check:**
1. RAlt key is functioning: `KeyHistory` in AHK window tray icon
2. No conflicting hotkeys in other running AHK scripts
3. Target application isn't blocking the hotkey
4. Function key isn't mapped to hardware action (F-Lock, media keys)

**Debug:**
```ahk
RAlt & F1::{
    ToolTip "Hotkey triggered!"
    SetTimer () => ToolTip(), -2000
}
```

### Context Detection Not Working

**Check:**
1. Verify process name: Add debug output
   ```ahk
   ActiveProcess := WinGetProcessName("A")
   MsgBox ActiveProcess
   ```
2. Check for process name variations (32-bit vs 64-bit: `app.exe` vs `app64.exe`)
3. Try window class instead: `WinGetClass("A")`

**Alternative:**
```ahk
; Use window class for more reliable detection
ActiveClass := WinGetClass("A")
if (ActiveClass = "CabinetWClass")  ; Windows Explorer
    ; Action
```

### Send Command Not Working

**Check:**
1. Application accepts the key combination
2. Try `SendInput` or `SendPlay` instead of `Send`
3. Add delays: `Send("{Ctrl down}c{Ctrl up}")`
4. Check if admin privileges required

**Alternative:**
```ahk
; More reliable for some applications
SendInput "^c"

; With explicit delays
Send "{Ctrl down}"
Sleep 50
Send "c"
Sleep 50
Send "{Ctrl up}"
```

### Modifier Detection Failing

**Check:**
1. Use physical state: `GetKeyState("LCtrl", "P")` not `GetKeyState("Ctrl")`
2. Check key timing: Modifier must be held when RAlt+F{N} pressed
3. Test with simple ToolTip output

**Debug:**
```ahk
RAlt & F1::{
    ctrlState := GetKeyState("LCtrl", "P") ? "Down" : "Up"
    shiftState := GetKeyState("LShift", "P") ? "Down" : "Up"
    altState := GetKeyState("LAlt", "P") ? "Down" : "Up"

    ToolTip "Ctrl: " ctrlState "`nShift: " shiftState "`nAlt: " altState
    SetTimer () => ToolTip(), -3000
}
```

---

## Performance Considerations

### Minimize Process Checks
```ahk
; Cache process name if checking multiple times
static cachedProcess := ""
if (cachedProcess = "" || A_TickCount > lastCheck + 1000) {
    cachedProcess := WinGetProcessName("A")
    lastCheck := A_TickCount
}
```

### Efficient Context Detection
```ahk
; Use static arrays for process lists
static browsers := ["msedge.exe", "chrome.exe", "firefox.exe", "brave.exe"]

ActiveProcess := WinGetProcessName("A")
for browser in browsers {
    if (ActiveProcess = browser) {
        ; Browser-specific action
        return
    }
}
```

### Avoid Excessive Send Commands
```ahk
; Instead of multiple Send calls
Send("{Ctrl down}")
Send("a")
Send("{Ctrl up}")

; Use single Send with modifier
Send("^a")
```

---

## Adding New Hotkeys

### Step-by-Step Guide

1. **Choose the Module**
   - Navigation → `hk_basic.ahk2`
   - Code editing → `hk_code.ahk2`
   - Program launching → `hk_programs.ahk2`
   - Tab operations → `hk_tab.ahk2`
   - Mouse wheel → `hk_wheel.ahk2`
   - File explorer → `windows_explorer.ahk2`
   - Taskbar → `Startbar.ahk2`

2. **Select Available Function Key**
   Check existing hotkeys to avoid conflicts. F1-F12 available with RAlt.

3. **Implement Hotkey**
   ```ahk
   RAlt & F10::{
       ; Add your implementation
       ActiveProcess := WinGetProcessName("A")

       if (ActiveProcess = "target.exe") {
           Send("^+p")  ; Context-specific action
       }
       else {
           Send("{F10}")  ; Default action
       }
   }
   ```

4. **Add Modifier Support (Optional)**
   ```ahk
   RAlt & F10::{
       if GetKeyState("LCtrl", "P") {
           Send("^{F10}")
       }
       else {
           Send("{F10}")
       }
   }
   ```

5. **Test Thoroughly**
   - Test in target applications
   - Test with all modifier combinations
   - Test for conflicts
   - Test in different window states

6. **Document**
   - Update this file (mouse-hotkeys.md)
   - Add to hotkey reference table
   - Document context-specific behaviors
   - Note any known limitations

---

**See Also:**
- `src/MouseHotkeys/claude.md` - Development guide
- `architecture.md` - System architecture
- `configuration.md` - Configuration system
