# How-To: Create Custom Window Dragging Behavior

## Problem Statement

You want to make windows draggable from areas other than the title bar - such as the entire client area, specific regions, or frameless windows. The default Windows behavior only allows dragging from the title bar, but you need custom drag zones for modern UI designs.

## Basic Concept: WM_NCHITTEST

Windows uses the `WM_NCHITTEST` message to determine what part of a window is under the cursor. By intercepting this message and returning `HTCAPTION` (title bar code), you can make any area draggable.

**Message flow:**
1. Cursor moves over window
2. Windows sends WM_NCHITTEST (0x0084)
3. Window procedure returns hit test code
4. Windows uses code to determine cursor shape and click behavior

**Key return values:**
- `HTCLIENT` (1) = Client area (normal clicks)
- `HTCAPTION` (2) = Title bar (draggable!)
- `HTLEFT/RIGHT/TOP/BOTTOM` (10/11/12/15) = Borders (resizable)
- `HTTRANSPARENT` (-1) = Pass through to window below

## Solution 1: Make Entire Client Area Draggable

```ahk
#Requires AutoHotkey v2.0

MyGui := Gui("+Caption")
MyGui.Add("Text", "cWhite", "Drag anywhere in this window to move it!")
MyGui.Show("w400 h250")

OnMessage(0x0084, WM_NCHITTEST_Handler)

WM_NCHITTEST_Handler(wParam, lParam, msg, hwnd) {
    static HTCLIENT := 1, HTCAPTION := 2

    ; Only modify our GUI windows
    if (hwnd != MyGui.Hwnd)
        return

    ; Get default hit test result
    result := DllCall("DefWindowProc", "Ptr", hwnd, "UInt", msg, "Ptr", wParam, "Ptr", lParam, "Ptr")

    ; If cursor is in client area, pretend it's in caption (title bar)
    if (result = HTCLIENT)
        return HTCAPTION

    return result  ; Return original result for borders, buttons, etc.
}
```

**How it works:**
- We call DefWindowProc to get the default hit test result
- If the default result is HTCLIENT (client area), we return HTCAPTION instead
- Windows now treats the client area as if it were the title bar
- User can click and drag from anywhere in the window

## Solution 2: Custom Drag Regions (Frameless Window)

For modern UI with custom title bars and specific drag zones:

```ahk
#Requires AutoHotkey v2.0

; Create frameless window with custom drag zones
MyGui := Gui("-Caption +Border")
MyGui.BackColor := "0x1E1E1E"
MyGui.MarginX := 0
MyGui.MarginY := 0

; Custom title bar area (draggable)
titleBar := MyGui.Add("Text", "x0 y0 w600 h40 Background0x2D2D30 cWhite Center", "Custom Title Bar (Drag Here)")
titleBar.SetFont("s12 Bold")

; Control buttons in title bar (not draggable)
closeBtn := MyGui.Add("Button", "x560 y5 w30 h30", "X")
closeBtn.OnEvent("Click", (*) => MyGui.Destroy())

; Content area (not draggable)
MyGui.Add("Text", "x20 y60 cWhite", "Content area - cannot drag from here")
MyGui.Add("Edit", "x20 y90 w560 h200", "This is content.`n`nDrag from dark title bar at top.`n`nContent area does not allow dragging.")

MyGui.Show("w600 h330")

OnMessage(0x0084, WM_NCHITTEST_Handler)

WM_NCHITTEST_Handler(wParam, lParam, msg, hwnd) {
    static HTCLIENT := 1, HTCAPTION := 2

    ; Only handle our GUI
    if (hwnd != MyGui.Hwnd)
        return

    ; Extract cursor position (screen coordinates)
    mouseX := lParam & 0xFFFF
    mouseY := (lParam >> 16) & 0xFFFF

    ; Convert to client coordinates
    POINT := Buffer(8, 0)
    NumPut("Int", mouseX, POINT, 0)
    NumPut("Int", mouseY, POINT, 4)
    DllCall("ScreenToClient", "Ptr", hwnd, "Ptr", POINT)
    clientX := NumGet(POINT, 0, "Int")
    clientY := NumGet(POINT, 4, "Int")

    ; Define draggable region (title bar area, but not the close button)
    if (clientY >= 0 && clientY <= 40 && clientX < 555) {
        return HTCAPTION  ; Make it draggable
    }

    ; Default behavior for everything else
    return HTCLIENT
}
```

**How it works:**
- We get mouse position from lParam (screen coordinates)
- Convert screen coordinates to client coordinates using ScreenToClient
- Check if cursor is in title bar region (y: 0-40, x: 0-555)
- Return HTCAPTION for title bar, HTCLIENT for content area
- Close button area (x > 555) returns HTCLIENT so it can be clicked

## Solution 3: Multiple Drag Zones

Create complex layouts with multiple draggable regions:

```ahk
#Requires AutoHotkey v2.0

MyGui := Gui("-Caption +Border")
MyGui.BackColor := "White"

; Header (draggable)
MyGui.Add("Text", "x0 y0 w400 h50 BackgroundBlue cWhite Center", "Header (Draggable)")

; Sidebar (draggable)
MyGui.Add("Text", "x0 y50 w100 h300 BackgroundGray", "Sidebar`n(Draggable)")

; Content area (not draggable, contains controls)
MyGui.Add("Text", "x120 y70", "Content Area (Not Draggable)")
MyGui.Add("Edit", "x120 y100 w250 h100")
MyGui.Add("Button", "x120 y220", "Button")

MyGui.Show("w400 h350")

OnMessage(0x0084, WM_NCHITTEST_Handler)

WM_NCHITTEST_Handler(wParam, lParam, msg, hwnd) {
    static HTCLIENT := 1, HTCAPTION := 2

    if (hwnd != MyGui.Hwnd)
        return

    ; Get client coordinates
    mouseX := lParam & 0xFFFF
    mouseY := (lParam >> 16) & 0xFFFF

    POINT := Buffer(8, 0)
    NumPut("Int", mouseX, POINT, 0)
    NumPut("Int", mouseY, POINT, 4)
    DllCall("ScreenToClient", "Ptr", hwnd, "Ptr", POINT)
    clientX := NumGet(POINT, 0, "Int")
    clientY := NumGet(POINT, 4, "Int")

    ; Header region (full width, top 50 pixels)
    if (clientY >= 0 && clientY <= 50) {
        return HTCAPTION
    }

    ; Sidebar region (left 100 pixels, below header)
    if (clientX >= 0 && clientX <= 100 && clientY > 50) {
        return HTCAPTION
    }

    ; Everything else is client area (controls work normally)
    return HTCLIENT
}
```

## Solution 4: Conditional Dragging (With Modifier Key)

Allow dragging only when a modifier key is held:

```ahk
#Requires AutoHotkey v2.0

MyGui := Gui()
MyGui.Add("Text", , "Hold Ctrl and drag to move window")
MyGui.Add("Edit", "w300 h100", "This edit control works normally.`n`nHold Ctrl to drag window.")
MyGui.Show()

OnMessage(0x0084, WM_NCHITTEST_Handler)

WM_NCHITTEST_Handler(wParam, lParam, msg, hwnd) {
    static HTCLIENT := 1, HTCAPTION := 2

    if (hwnd != MyGui.Hwnd)
        return

    ; Only make draggable when Ctrl is held
    if GetKeyState("Ctrl", "P") {
        result := DllCall("DefWindowProc", "Ptr", hwnd, "UInt", msg, "Ptr", wParam, "Ptr", lParam, "Ptr")
        if (result = HTCLIENT)
            return HTCAPTION
    }

    return  ; Use default behavior
}
```

**Use case:** Windows with many interactive controls where accidental dragging would be annoying.

## Complete Example: Modern App Window

```ahk
#Requires AutoHotkey v2.0

; Modern frameless window with custom chrome
AppGui := Gui("-Caption +Border")
AppGui.BackColor := "0x202020"
AppGui.MarginX := 0
AppGui.MarginY := 0

; Custom title bar with buttons
titleBg := AppGui.Add("Text", "x0 y0 w600 h35 Background0x2D2D30")
titleText := AppGui.Add("Text", "x10 y8 Background0x2D2D30 cWhite", "My Application")

; Window control buttons
minimizeBtn := AppGui.Add("Button", "x500 y5 w30 h25", "_")
minimizeBtn.OnEvent("Click", (*) => WinMinimize("ahk_id " AppGui.Hwnd))

maximizeBtn := AppGui.Add("Button", "x535 y5 w30 h25", "□")
maximizeBtn.OnEvent("Click", ToggleMaximize)

closeBtn := AppGui.Add("Button", "x570 y5 w25 h25", "X")
closeBtn.OnEvent("Click", (*) => AppGui.Destroy())

; Menu bar (draggable)
menuBg := AppGui.Add("Text", "x0 y35 w600 h25 Background0x383838")
AppGui.Add("Text", "x10 y38 Background0x383838 cWhite", "File")
AppGui.Add("Text", "x50 y38 Background0x383838 cWhite", "Edit")
AppGui.Add("Text", "x90 y38 Background0x383838 cWhite", "View")

; Content area
AppGui.Add("Text", "x20 y80 cWhite", "Content Area:")
AppGui.Add("Edit", "x20 y110 w560 h300")

AppGui.Show("w600 h450")

; Track maximize state
global isMaximized := false

ToggleMaximize(*) {
    global isMaximized
    if (isMaximized) {
        WinRestore("ahk_id " AppGui.Hwnd)
        isMaximized := false
    } else {
        WinMaximize("ahk_id " AppGui.Hwnd)
        isMaximized := true
    }
}

OnMessage(0x0084, WM_NCHITTEST_Handler)

WM_NCHITTEST_Handler(wParam, lParam, msg, hwnd) {
    static HTCLIENT := 1, HTCAPTION := 2

    if (hwnd != AppGui.Hwnd)
        return

    ; Get client coordinates
    mouseX := lParam & 0xFFFF
    mouseY := (lParam >> 16) & 0xFFFF

    POINT := Buffer(8, 0)
    NumPut("Int", mouseX, POINT, 0)
    NumPut("Int", mouseY, POINT, 4)
    DllCall("ScreenToClient", "Ptr", hwnd, "Ptr", POINT)
    clientX := NumGet(POINT, 0, "Int")
    clientY := NumGet(POINT, 4, "Int")

    ; Title bar (but not control buttons)
    if (clientY >= 0 && clientY <= 35 && clientX < 495) {
        return HTCAPTION
    }

    ; Menu bar
    if (clientY > 35 && clientY <= 60) {
        return HTCAPTION
    }

    ; Default behavior
    return HTCLIENT
}
```

## Troubleshooting

### Controls in draggable area don't work

**Problem:** Buttons/edits in draggable regions can't be clicked.

**Solution:** Exclude control areas from drag region:

```ahk
; Get control positions and exclude them
if (clientY >= 0 && clientY <= 40) {
    ; Check if cursor is over a button
    if (clientX >= 560 && clientX <= 590) {
        return HTCLIENT  ; Allow button clicks
    }
    return HTCAPTION  ; Drag everywhere else in title bar
}
```

### Window doesn't drag on multi-monitor setups

**Problem:** Coordinate conversion fails with negative screen coordinates.

**Solution:** The ScreenToClient API handles negative coordinates correctly. Ensure you're using it:

```ahk
; ✅ Correct - handles negative coordinates
DllCall("ScreenToClient", "Ptr", hwnd, "Ptr", POINT)

; ❌ Wrong - manual conversion fails with negative values
clientX := mouseX - winX
```

### Dragging is choppy or laggy

**Problem:** WM_NCHITTEST is called very frequently (100s of times per second).

**Solution:** Keep handler extremely fast - no heavy operations:

```ahk
; ❌ Wrong - slow operations in handler
WM_NCHITTEST_Handler(wParam, lParam, msg, hwnd) {
    WinGetPos(&x, &y, &w, &h, "ahk_id " hwnd)  ; SLOW!
    ; ... more processing
}

; ✅ Right - minimal processing
WM_NCHITTEST_Handler(wParam, lParam, msg, hwnd) {
    static HTCLIENT := 1, HTCAPTION := 2
    ; Fast coordinate checks only
    return HTCAPTION
}
```

### Can't resize window after removing caption

**Problem:** Frameless windows can't be resized.

**Solution:** Return appropriate resize codes for window edges:

```ahk
WM_NCHITTEST_Handler(wParam, lParam, msg, hwnd) {
    static HTCLIENT := 1, HTCAPTION := 2
    static HTLEFT := 10, HTRIGHT := 11, HTTOP := 12, HTBOTTOM := 15
    static HTTOPLEFT := 13, HTTOPRIGHT := 14
    static HTBOTTOMLEFT := 16, HTBOTTOMRIGHT := 17

    ; Get window size
    WinGetPos(&winX, &winY, &winW, &winH, "ahk_id " hwnd)

    ; Convert to client coordinates
    ; ... (coordinate conversion code)

    ; Define resize border width
    borderWidth := 5

    ; Check for resize regions
    if (clientX < borderWidth && clientY < borderWidth)
        return HTTOPLEFT
    if (clientX >= winW - borderWidth && clientY < borderWidth)
        return HTTOPRIGHT
    if (clientX < borderWidth && clientY >= winH - borderWidth)
        return HTBOTTOMLEFT
    if (clientX >= winW - borderWidth && clientY >= winH - borderWidth)
        return HTBOTTOMRIGHT
    if (clientX < borderWidth)
        return HTLEFT
    if (clientX >= winW - borderWidth)
        return HTRIGHT
    if (clientY < borderWidth)
        return HTTOP
    if (clientY >= winH - borderWidth)
        return HTBOTTOM

    ; Title bar drag region
    if (clientY <= 40)
        return HTCAPTION

    return HTCLIENT
}
```

## Performance Considerations

**WM_NCHITTEST frequency:**
- Called 100-200 times per second during mouse movement
- Handler must execute in <1ms to avoid UI lag
- Avoid: WinGetPos, file I/O, complex calculations
- Use: Simple coordinate comparisons, static values

**Memory usage:**
- Static variables in handler: ~100 bytes
- No memory leak risk
- Buffer for coordinate conversion: 8 bytes (reused)

## Related Concepts

- docs/manuals/ahk2/how-to/save-restore-positions.md - Save window positions
- docs/source-reports/ahk2/ahk2_sys_01_window_messages.md - Windows message architecture
- docs/manuals/ahk2/reference/window-messages.md - Complete message reference

## Key Takeaway

Custom window dragging is achieved by intercepting WM_NCHITTEST and returning HTCAPTION for regions you want to be draggable. Keep the handler fast (simple coordinate checks only), convert screen coordinates to client coordinates for accurate region detection, and exclude control areas from drag regions so buttons and other elements remain functional.
