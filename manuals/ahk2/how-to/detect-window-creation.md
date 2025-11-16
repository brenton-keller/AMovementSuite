# How-To: Detect When Windows Are Created

## Problem Statement

You need to execute code automatically when specific windows are created system-wide - such as auto-configuring new application windows, logging window creation, or triggering actions when certain programs start. Polling with WinExist in a loop is CPU-intensive and has delayed detection.

## Solution 1: RegisterShellHookWindow (No DLL Required)

The simplest system-wide approach uses Windows Shell Hooks:

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force
Persistent()

; Target application to monitor
targetClass := "Notepad"

; Register for shell hook notifications
shellHookMsg := DllCall("RegisterWindowMessage", "Str", "SHELLHOOK", "UInt")

; Create hidden window to receive shell hook messages
DetectHiddenWindows(true)
hiddenGui := Gui("+LastFound")
guiHwnd := WinExist()

; Register as shell hook window
if !DllCall("RegisterShellHookWindow", "Ptr", guiHwnd) {
    MsgBox("Failed to register shell hook window")
    ExitApp
}

OnMessage(shellHookMsg, ShellHookHandler)

; Tray notification
TraySetIcon("Shell32.dll", 22)  ; Binoculars icon
TrayTip("Window Monitor Active", "Watching for " targetClass " windows...")

ShellHookHandler(wParam, lParam, msg, hwnd) {
    static HSHELL_WINDOWCREATED := 1
    global targetClass

    ; Only process window creation events
    if (wParam != HSHELL_WINDOWCREATED)
        return

    newHwnd := lParam  ; New window handle

    ; Get window class and title
    try {
        className := WinGetClass("ahk_id " newHwnd)
        windowTitle := WinGetTitle("ahk_id " newHwnd)

        ; Check if it matches our target
        if (className = targetClass) {
            ; Wait for window to fully initialize
            Sleep(100)

            ; Execute custom action
            PerformCustomAction(newHwnd, windowTitle)
        }
    } catch {
        ; Window may have closed already
    }
}

PerformCustomAction(hwnd, title) {
    ; Example actions:

    ; 1. Move and resize the window
    WinMove(100, 100, 800, 600, "ahk_id " hwnd)

    ; 2. Set always on top
    WinSetAlwaysOnTop(true, "ahk_id " hwnd)

    ; 3. Send some text
    Sleep(200)  ; Wait for window to be ready
    ControlSend("Hello from AHK2!{Enter}", "Edit1", "ahk_id " hwnd)

    ; 4. Show notification
    TrayTip("Action Executed", "Configured " title, 1)
}

; Cleanup on exit
OnExit(CleanupHook)

CleanupHook(*) {
    global guiHwnd
    DllCall("DeregisterShellHookWindow", "Ptr", guiHwnd)
}
```

**How it works:**
1. RegisterWindowMessage creates custom message for shell hooks
2. RegisterShellHookWindow registers our hidden GUI to receive notifications
3. When any top-level window is created, we receive message with HSHELL_WINDOWCREATED
4. lParam contains handle of new window
5. We check window class/title and execute actions

**Limitations:**
- Only detects **top-level windows**, not child windows
- Slight delay (~50-100ms) between creation and notification
- Windows only, no cross-platform support

## Solution 2: Polling with Optimized Detection

For scenarios where shell hooks don't work (child windows, specific timing requirements):

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force
Persistent()

; Track seen windows
global seenWindows := Map()

; Check for new windows every 500ms
SetTimer(CheckForNewWindows, 500)

CheckForNewWindows() {
    ; Get all visible windows
    windowList := WinGetList()

    for hwnd in windowList {
        ; Skip if we've already seen this window
        if (seenWindows.Has(hwnd))
            continue

        ; New window detected
        seenWindows[hwnd] := true

        try {
            className := WinGetClass("ahk_id " hwnd)
            title := WinGetTitle("ahk_id " hwnd)

            ; Filter for target windows
            if (className = "Notepad") {
                OnNewNotepadWindow(hwnd, title)
            }
        } catch {
            ; Window may have closed
        }
    }

    ; Cleanup - remove windows that no longer exist
    if (seenWindows.Count > 1000) {  ; Prevent unbounded growth
        CleanupClosedWindows()
    }
}

OnNewNotepadWindow(hwnd, title) {
    TrayTip("New Notepad", title)
    WinMove(100, 100, 800, 600, "ahk_id " hwnd)
}

CleanupClosedWindows() {
    global seenWindows
    toRemove := []

    for hwnd in seenWindows {
        if !WinExist("ahk_id " hwnd)
            toRemove.Push(hwnd)
    }

    for hwnd in toRemove {
        seenWindows.Delete(hwnd)
    }
}
```

**Advantages:**
- Works for child windows
- More control over detection timing
- Can detect window destruction too

**Disadvantages:**
- CPU overhead (~0.5-2% depending on interval)
- Delay between creation and detection
- May miss very short-lived windows

## Solution 3: Hybrid Approach (Best)

Combine shell hooks for instant detection with periodic cleanup:

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force
Persistent()

class WindowMonitor {
    static trackedWindows := Map()
    static shellHookMsg := 0
    static guiHwnd := 0
    static callbacks := Map()

    static Init() {
        ; Register shell hook
        this.shellHookMsg := DllCall("RegisterWindowMessage", "Str", "SHELLHOOK", "UInt")

        DetectHiddenWindows(true)
        hiddenGui := Gui("+LastFound")
        this.guiHwnd := WinExist()

        if !DllCall("RegisterShellHookWindow", "Ptr", this.guiHwnd) {
            throw Error("Failed to register shell hook")
        }

        OnMessage(this.shellHookMsg, ObjBindMethod(this, "OnShellHook"))

        ; Periodic cleanup
        SetTimer(ObjBindMethod(this, "Cleanup"), 300000)  ; Every 5 minutes

        ; Cleanup on exit
        OnExit(ObjBindMethod(this, "OnExit"))
    }

    static Watch(windowClass, callback) {
        if (!this.callbacks.Has(windowClass))
            this.callbacks[windowClass] := []

        this.callbacks[windowClass].Push(callback)
    }

    static OnShellHook(wParam, lParam, msg, hwnd) {
        static HSHELL_WINDOWCREATED := 1

        if (wParam != HSHELL_WINDOWCREATED)
            return

        newHwnd := lParam
        this.trackedWindows[newHwnd] := true

        ; Process in separate timer to avoid blocking shell hook
        SetTimer(() => this.ProcessNewWindow(newHwnd), -10)
    }

    static ProcessNewWindow(hwnd) {
        try {
            className := WinGetClass("ahk_id " hwnd)
            title := WinGetTitle("ahk_id " hwnd)

            ; Check if any callbacks registered for this class
            if (this.callbacks.Has(className)) {
                for callback in this.callbacks[className] {
                    try {
                        callback(hwnd, title, className)
                    } catch as e {
                        ErrorHandler.Log("Window monitor callback failed", e.Message)
                    }
                }
            }
        } catch {
            ; Window may have closed
        }
    }

    static Cleanup() {
        toRemove := []
        for hwnd in this.trackedWindows {
            if !WinExist("ahk_id " hwnd)
                toRemove.Push(hwnd)
        }

        for hwnd in toRemove {
            this.trackedWindows.Delete(hwnd)
        }
    }

    static OnExit(*) {
        DllCall("DeregisterShellHookWindow", "Ptr", this.guiHwnd)
    }
}

; Initialize
WindowMonitor.Init()

; Register callbacks for different window classes
WindowMonitor.Watch("Notepad", (hwnd, title, class) => {
    TrayTip("Notepad Opened", title)
    WinMove(100, 100, 800, 600, "ahk_id " hwnd)
})

WindowMonitor.Watch("Chrome_WidgetWin_1", (hwnd, title, class) => {
    if (InStr(title, "Google Chrome")) {
        TrayTip("Chrome Window", title)
    }
})

WindowMonitor.Watch("MozillaWindowClass", (hwnd, title, class) => {
    TrayTip("Firefox Window", title)
})
```

## Use Case: Auto-Configure Application Windows

Automatically configure specific applications when they open:

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force

WindowMonitor.Init()

; Visual Studio Code - set always on top and specific position
WindowMonitor.Watch("Chrome_WidgetWin_1", (hwnd, title, class) => {
    if (InStr(title, "Visual Studio Code")) {
        Sleep(500)  ; Wait for window to fully render
        WinMove(0, 0, 1920, 1080, "ahk_id " hwnd)
        WinSetAlwaysOnTop(true, "ahk_id " hwnd)
    }
})

; Notepad - insert template text
WindowMonitor.Watch("Notepad", (hwnd, title, class) => {
    if (title = "Untitled - Notepad") {
        Sleep(200)
        template := "# New Document`n`nCreated: " FormatTime() "`n`n"
        ControlSend(template, "Edit1", "ahk_id " hwnd)
    }
})

; CMD windows - set dark theme
WindowMonitor.Watch("ConsoleWindowClass", (hwnd, title, class) => {
    Sleep(100)
    ; Send command to set dark theme
    ControlSend("color 0A{Enter}", , "ahk_id " hwnd)
})
```

## Use Case: Window Creation Logger

Log all window creations for debugging or monitoring:

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force

logFile := A_ScriptDir "\window_creation.log"

WindowMonitor.Init()

; Log all window creations (no specific class filter)
OnMessage(WindowMonitor.shellHookMsg, LogAllWindows)

LogAllWindows(wParam, lParam, msg, hwnd) {
    static HSHELL_WINDOWCREATED := 1

    if (wParam != HSHELL_WINDOWCREATED)
        return

    newHwnd := lParam

    try {
        className := WinGetClass("ahk_id " newHwnd)
        title := WinGetTitle("ahk_id " newHwnd)
        processName := WinGetProcessName("ahk_id " newHwnd)

        timestamp := FormatTime(, "yyyy-MM-dd HH:mm:ss")
        logEntry := Format("[{1}] CREATED: {2} | Class: {3} | Process: {4} | HWND: {5}`n",
            timestamp, title, className, processName, newHwnd)

        FileAppend(logEntry, logFile)
    } catch {
        ; Window closed too quickly
    }
}
```

## Use Case: Application Launcher Monitor

Detect when specific applications start and perform actions:

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force

WindowMonitor.Init()

; Detect Spotify and send play command
WindowMonitor.Watch("Chrome_WidgetWin_1", (hwnd, title, class) => {
    if (InStr(title, "Spotify")) {
        Sleep(2000)  ; Wait for app to fully load
        ; Send media play key
        Send("{Media_Play_Pause}")
        TrayTip("Spotify", "Auto-started playback")
    }
})

; Detect Discord and set status
WindowMonitor.Watch("Chrome_WidgetWin_1", (hwnd, title, class) => {
    if (InStr(title, "Discord")) {
        ; Discord detected - you could send hotkey to set status
        TrayTip("Discord", "Detected Discord startup")
    }
})
```

## Troubleshooting

### Shell hook not receiving notifications

**Problem:** OnMessage handler never called.

**Solution:** Ensure hidden GUI remains in scope:

```ahk
; ❌ Wrong - GUI goes out of scope
CreateHiddenGui() {
    hiddenGui := Gui()  ; Local variable!
    return WinExist("ahk_id " hiddenGui.Hwnd)
}
guiHwnd := CreateHiddenGui()  ; GUI destroyed here!

; ✅ Right - GUI persists
global hiddenGui := Gui("+LastFound")
guiHwnd := WinExist()
```

### Callback executes too early

**Problem:** Window not fully initialized when callback runs.

**Solution:** Add delay before accessing window:

```ahk
WindowMonitor.Watch("Notepad", (hwnd, title, class) => {
    Sleep(200)  ; Wait for window to be ready
    ; Now safe to interact with window
    ControlSend("Text", "Edit1", "ahk_id " hwnd)
})
```

### Missing some window creations

**Problem:** Very short-lived windows not detected.

**Solution:** Shell hooks have inherent delay. For critical detection, combine with polling:

```ahk
; Use shell hooks for instant detection
WindowMonitor.Init()
WindowMonitor.Watch("TargetClass", ProcessWindow)

; Add polling backup for critical windows
SetTimer(CheckForMissedWindows, 100)
```

### Callback causes script to hang

**Problem:** Long-running callback blocks shell hook processing.

**Solution:** Use SetTimer to defer heavy work:

```ahk
WindowMonitor.Watch("Chrome_WidgetWin_1", (hwnd, title, class) => {
    ; Quick check only
    if (InStr(title, "MyApp")) {
        ; Defer heavy work
        SetTimer(() => HeavyConfiguration(hwnd), -100)
    }
})

HeavyConfiguration(hwnd) {
    ; Long-running operations here
    Sleep(2000)
    ; Complex window manipulation
}
```

## Performance Considerations

**Shell hook overhead:**
- Message processing: ~0.1ms per window creation
- Memory per tracked window: ~50 bytes
- Negligible CPU usage during normal operation

**Polling overhead:**
- 500ms interval: ~1-2% CPU
- 1000ms interval: ~0.5-1% CPU
- 100ms interval: ~5-10% CPU (not recommended)

**Best practices:**
- Use shell hooks as primary method
- Add polling only for critical detection needs
- Keep callbacks fast (< 50ms)
- Use SetTimer to defer heavy work
- Cleanup tracked windows periodically

## Related Concepts

- docs/manuals/ahk2/how-to/save-restore-positions.md - Auto-restore positions for new windows
- docs/manuals/ahk2/how-to/window-dragging.md - Custom window behavior
- docs/source-reports/ahk2/ahk2_sys_01_window_messages.md - Windows messaging system

## Key Takeaway

Window creation detection enables powerful automation workflows. RegisterShellHookWindow provides instant, low-overhead detection for top-level windows without requiring DLLs. The WindowMonitor class provides a clean, reusable interface for registering callbacks that execute when specific window classes are created. Always add delays before interacting with newly created windows to ensure they're fully initialized.
