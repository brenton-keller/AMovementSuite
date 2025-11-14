# How-To: Save and Restore Window Positions

## Problem Statement

You want windows to remember their position and size between script runs. Every time the user positions a window perfectly, they expect it to reappear in the same location next time. Manually repositioning windows on every launch creates friction and poor user experience.

## Basic Solution: Save on Close, Restore on Show

Intercept `WM_CLOSE` to save position before window closes, then restore it when creating the window:

```ahk
#Requires AutoHotkey v2.0

; INI file for storing positions
iniFile := A_ScriptDir "\WindowPositions.ini"

; Create GUI
MyGui := Gui(, "Position Saver Demo")
MyGui.Add("Text", , "Close this window to save its position.`nRun script again to restore.")
MyGui.Add("Button", "w200", "Close & Save").OnEvent("Click", (*) => MyGui.Destroy())

; Try to restore saved position
if FileExist(iniFile) {
    x := IniRead(iniFile, "WindowPos", "X", "")
    y := IniRead(iniFile, "WindowPos", "Y", "")
    w := IniRead(iniFile, "WindowPos", "Width", "")
    h := IniRead(iniFile, "WindowPos", "Height", "")

    if (x != "" && y != "" && w != "" && h != "")
        MyGui.Show("x" x " y" y " w" w " h" h)
    else
        MyGui.Show("w300 h150")
} else {
    MyGui.Show("w300 h150")
}

; Intercept WM_CLOSE to save position before closing
OnMessage(0x0010, WM_CLOSE_Handler)

WM_CLOSE_Handler(wParam, lParam, msg, hwnd) {
    ; Only handle our GUI
    if (hwnd != MyGui.Hwnd)
        return

    ; Get current position and size
    try {
        WinGetPos(&x, &y, &w, &h, "ahk_id " hwnd)

        ; Save to INI file
        IniWrite(x, iniFile, "WindowPos", "X")
        IniWrite(y, iniFile, "WindowPos", "Y")
        IniWrite(w, iniFile, "WindowPos", "Width")
        IniWrite(h, iniFile, "WindowPos", "Height")

        ToolTip("Position saved: " x "," y " " w "x" h)
        SetTimer(() => ToolTip(), -1500)
    } catch {
        ; If window already destroyed, just continue
    }

    ; Allow normal close to proceed
    return
}
```

**How it works:**
- OnMessage intercepts WM_CLOSE (sent when user clicks X or calls Destroy)
- We get window position with WinGetPos before it's destroyed
- Position saved to INI file
- On next run, we check if INI exists and restore position
- try-catch protects against window already being destroyed

## Enhanced Solution: Multiple Windows

Track positions for multiple windows using window identifiers:

```ahk
#Requires AutoHotkey v2.0

iniFile := A_ScriptDir "\WindowPositions.ini"

; Create multiple windows
mainWin := CreateRestoredWindow("MainWindow", "Main Window")
settingsWin := CreateRestoredWindow("SettingsWindow", "Settings Window")
toolsWin := CreateRestoredWindow("ToolsWindow", "Tools Window")

CreateRestoredWindow(windowName, title) {
    gui := Gui(, title)
    gui.Add("Text", , "Window: " windowName)
    gui.Add("Button", "w200", "Close").OnEvent("Click", (*) => gui.Destroy())

    ; Store window name for later
    gui.windowName := windowName

    ; Restore position
    RestoreWindowPosition(gui, windowName)

    ; Register close handler
    OnMessage(0x0010, SaveOnClose)

    gui.Show()
    return gui
}

RestoreWindowPosition(gui, windowName) {
    global iniFile

    try {
        x := IniRead(iniFile, windowName, "X", "")
        y := IniRead(iniFile, windowName, "Y", "")
        w := IniRead(iniFile, windowName, "Width", "")
        h := IniRead(iniFile, windowName, "Height", "")

        if (x != "" && y != "" && w != "" && h != "") {
            ; Validate position is on screen
            if IsOnScreen(Integer(x), Integer(y)) {
                return gui.Show("x" x " y" y " w" w " h" h)
            }
        }
    } catch {
        ; No saved position or read failed
    }

    ; Default position
    gui.Show("w300 h200")
}

SaveOnClose(wParam, lParam, msg, hwnd) {
    global iniFile

    ; Find which window this is
    windowName := ""
    for win in [mainWin, settingsWin, toolsWin] {
        if (hwnd = win.Hwnd) {
            windowName := win.windowName
            break
        }
    }

    if (windowName = "")
        return  ; Not our window

    ; Save position
    try {
        WinGetPos(&x, &y, &w, &h, "ahk_id " hwnd)
        IniWrite(x, iniFile, windowName, "X")
        IniWrite(y, iniFile, windowName, "Y")
        IniWrite(w, iniFile, windowName, "Width")
        IniWrite(h, iniFile, windowName, "Height")
    }

    return  ; Allow close to proceed
}

IsOnScreen(x, y) {
    ; Check if position is on any monitor
    MonitorCount := MonitorGetCount()
    Loop MonitorCount {
        MonitorGet(A_Index, &left, &top, &right, &bottom)
        if (x >= left && x <= right && y >= top && y <= bottom)
            return true
    }
    return false
}
```

## Production Solution: WindowPositionManager Class

Reusable class for managing window positions:

```ahk
class WindowPositionManager {
    static iniFile := A_ScriptDir "\WindowPositions.ini"
    static windows := Map()

    ; Register window for position tracking
    static Register(gui, windowName) {
        gui.windowName := windowName
        this.windows[gui.Hwnd] := gui

        ; Register message handler
        OnMessage(0x0010, ObjBindMethod(this, "OnClose"))

        ; Restore position
        this.Restore(gui, windowName)
    }

    ; Restore saved position
    static Restore(gui, windowName) {
        try {
            x := IniRead(this.iniFile, windowName, "X", "")
            y := IniRead(this.iniFile, windowName, "Y", "")
            w := IniRead(this.iniFile, windowName, "Width", "")
            h := IniRead(this.iniFile, windowName, "Height", "")
            state := IniRead(this.iniFile, windowName, "State", "")

            if (x = "" || y = "" || w = "" || h = "")
                return false

            ; Validate position is on screen
            if (!this.IsOnScreen(Integer(x), Integer(y)))
                return false

            ; Show window at saved position
            gui.Show("x" x " y" y " w" w " h" h)

            ; Restore maximized state
            if (state = "Maximized")
                WinMaximize("ahk_id " gui.Hwnd)

            return true

        } catch {
            return false
        }
    }

    ; Save position on close
    static OnClose(wParam, lParam, msg, hwnd) {
        if (!this.windows.Has(hwnd))
            return

        gui := this.windows[hwnd]
        windowName := gui.windowName

        try {
            ; Check if maximized
            isMaximized := WinGetMinMax("ahk_id " hwnd) = 1

            ; Get position (restore first if maximized to get non-maximized size)
            if (isMaximized)
                WinRestore("ahk_id " hwnd)

            WinGetPos(&x, &y, &w, &h, "ahk_id " hwnd)

            ; Save to INI
            IniWrite(x, this.iniFile, windowName, "X")
            IniWrite(y, this.iniFile, windowName, "Y")
            IniWrite(w, this.iniFile, windowName, "Width")
            IniWrite(h, this.iniFile, windowName, "Height")
            IniWrite(isMaximized ? "Maximized" : "Normal", this.iniFile, windowName, "State")

        } catch {
            ; Window may already be destroyed
        }

        return  ; Allow close to proceed
    }

    ; Check if position is visible on any monitor
    static IsOnScreen(x, y) {
        MonitorCount := MonitorGetCount()
        Loop MonitorCount {
            MonitorGet(A_Index, &left, &top, &right, &bottom)
            if (x >= left && x <= right && y >= top && y <= bottom)
                return true
        }
        return false
    }

    ; Manual save (for periodic auto-save)
    static SaveNow(gui) {
        this.OnClose(0, 0, 0x0010, gui.Hwnd)
    }

    ; Clear saved position
    static Clear(windowName) {
        try {
            IniDelete(this.iniFile, windowName)
        }
    }
}

; Usage example
MyGui := Gui(, "Managed Window")
MyGui.Add("Text", , "This window remembers its position")
MyGui.Add("Button", "w200", "Close").OnEvent("Click", (*) => MyGui.Destroy())

; Register for position management
WindowPositionManager.Register(MyGui, "MyMainWindow")

; If restore failed, show with defaults
; (Register handles restore, so just show if it failed)
; Show was already called in Register
```

## Auto-Save Every N Seconds

Save position periodically in case of crashes:

```ahk
#Requires AutoHotkey v2.0

MyGui := Gui(, "Auto-Saving Window")
MyGui.Add("Text", , "Position auto-saves every 30 seconds")
MyGui.Show()

; Register for position management
WindowPositionManager.Register(MyGui, "AutoSaveWindow")

; Auto-save every 30 seconds
SetTimer(() => WindowPositionManager.SaveNow(MyGui), 30000)
```

## Handling Multi-Monitor Changes

Validate saved positions are still visible:

```ahk
ValidateAndRestorePosition(gui, windowName) {
    try {
        x := IniRead(iniFile, windowName, "X")
        y := IniRead(iniFile, windowName, "Y")
        w := IniRead(iniFile, windowName, "Width")
        h := IniRead(iniFile, windowName, "Height")

        x := Integer(x)
        y := Integer(y)
        w := Integer(w)
        h := Integer(h)

        ; Check if any part of window would be visible
        if (IsRectVisible(x, y, w, h)) {
            gui.Show("x" x " y" y " w" w " h" h)
            return true
        } else {
            ; Position is off-screen (monitor removed?) - use center of primary
            MonitorGetWorkArea(1, &left, &top, &right, &bottom)
            centerX := (right - left - w) / 2
            centerY := (bottom - top - h) / 2
            gui.Show("x" centerX " y" centerY " w" w " h" h)
            return false
        }

    } catch {
        gui.Show("w300 h200")  ; Default
        return false
    }
}

IsRectVisible(x, y, w, h) {
    ; At least 100 pixels must be on a visible monitor
    requiredVisibleWidth := Min(100, w)
    requiredVisibleHeight := Min(100, h)

    MonitorCount := MonitorGetCount()
    Loop MonitorCount {
        MonitorGet(A_Index, &left, &top, &right, &bottom)

        ; Calculate visible intersection
        visibleLeft := Max(x, left)
        visibleTop := Max(y, top)
        visibleRight := Min(x + w, right)
        visibleBottom := Min(y + h, bottom)

        visibleWidth := Max(0, visibleRight - visibleLeft)
        visibleHeight := Max(0, visibleBottom - visibleTop)

        if (visibleWidth >= requiredVisibleWidth && visibleHeight >= requiredVisibleHeight)
            return true
    }

    return false
}
```

## Complete Example: Application with Saved Positions

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force

; Initialize position manager
WindowPositionManager.iniFile := A_ScriptDir "\AppPositions.ini"

; Create main window
MainWin := Gui(, "My Application")
MainWin.Add("Text", , "Main application window")
MainWin.Add("Button", "w150", "Open Settings").OnEvent("Click", ShowSettings)
MainWin.Add("Button", "w150", "Close").OnEvent("Click", (*) => MainWin.Destroy())
WindowPositionManager.Register(MainWin, "MainWindow")

; Settings window (created on demand)
global SettingsWin := ""

ShowSettings(*) {
    global SettingsWin

    if (SettingsWin && WinExist("ahk_id " SettingsWin.Hwnd)) {
        WinActivate("ahk_id " SettingsWin.Hwnd)
        return
    }

    SettingsWin := Gui(, "Settings")
    SettingsWin.Add("Text", , "Settings window")
    SettingsWin.Add("Button", "w150", "Close").OnEvent("Click", (*) => SettingsWin.Destroy())
    WindowPositionManager.Register(SettingsWin, "SettingsWindow")
}

; Auto-save positions every minute
SetTimer(SaveAllPositions, 60000)

SaveAllPositions() {
    if (MainWin && WinExist("ahk_id " MainWin.Hwnd))
        WindowPositionManager.SaveNow(MainWin)

    if (SettingsWin && WinExist("ahk_id " SettingsWin.Hwnd))
        WindowPositionManager.SaveNow(SettingsWin)
}

; Save on script exit
OnExit(SaveOnExit)

SaveOnExit(*) {
    SaveAllPositions()
}
```

## Troubleshooting

### Positions not saving

**Problem:** WM_CLOSE handler not being called.

**Solution:** Ensure OnMessage is registered BEFORE showing window:

```ahk
; ✅ Correct order
OnMessage(0x0010, SaveHandler)
MyGui.Show()

; ❌ Wrong order
MyGui.Show()
OnMessage(0x0010, SaveHandler)  ; Too late!
```

### Window appears off-screen after monitor change

**Problem:** Saved position is on a disconnected monitor.

**Solution:** Validate position with `IsOnScreen()` before restoring.

### Maximized state not restored

**Problem:** Only saving position, not window state.

**Solution:** Check and save maximized state:

```ahk
isMaximized := WinGetMinMax("ahk_id " hwnd) = 1
IniWrite(isMaximized ? "Maximized" : "Normal", iniFile, section, "State")

; On restore:
if (state = "Maximized")
    WinMaximize("ahk_id " gui.Hwnd)
```

### Positions saved while window is minimized

**Problem:** Saving minimized window coordinates (-32000, -32000).

**Solution:** Restore window before getting position:

```ahk
minMax := WinGetMinMax("ahk_id " hwnd)
if (minMax = -1) {  ; Minimized
    WinRestore("ahk_id " hwnd)
    Sleep(50)  ; Wait for restore
}
WinGetPos(&x, &y, &w, &h, "ahk_id " hwnd)
```

## Performance Considerations

**Save frequency:**
- On close: Zero runtime overhead, may lose position if crash
- Every 30-60s: Minimal overhead (<1ms), protects against crashes
- On every move: High overhead, not recommended

**INI file size:**
- ~100 bytes per window
- 100 windows = ~10KB file
- No performance impact

## Related Concepts

- docs/manuals/ahk2/how-to/window-dragging.md - Custom drag behavior
- docs/manuals/ahk2/how-to/detect-window-creation.md - Detect new windows
- docs/source-reports/ahk2/ahk2_sys_01_window_messages.md - Window messages

## Key Takeaway

Saving window positions creates a polished user experience. Intercept WM_CLOSE to save position before destruction, validate saved positions are still on-screen (monitor changes!), and consider periodic auto-save to protect against crashes. The WindowPositionManager class provides a reusable, production-ready solution for any script.
