# Reference: Windows Messages

Essential Windows messages with parameter decoding, usage patterns, and complete reference.

## Related Documentation
- docs/manuals/ahk2/reference/error-types.md - Errors in message operations

---

## Message Fundamentals

### MSG Structure

Every Windows message is packaged in this structure:

| Field | Type | Description |
|-------|------|-------------|
| `hwnd` | HWND | Window handle receiving message |
| `message` | UINT | Message identifier (e.g., 0x0200 for WM_MOUSEMOVE) |
| `wParam` | WPARAM | First parameter (meaning varies by message) |
| `lParam` | LPARAM | Second parameter (meaning varies by message) |
| `time` | DWORD | Milliseconds since system startup (GetTickCount) |
| `pt` | POINT | Cursor position in screen coordinates |

### Message Number Ranges

| Range | Usage | Notes |
|-------|-------|-------|
| `0x0000` - `0x03FF` | System messages | WM_* constants |
| `0x0400` - `0x7FFF` | Reserved for Windows | Application-specific |
| `0x8000` - `0xBFFF` | WM_APP range | Safe for custom messages |
| `0xC000` - `0xFFFF` | String-registered messages | RegisterWindowMessage() |

**Best Practice:** Use WM_APP range (`0x8000+`) for custom inter-script messages.

---

## Parameter Decoding Patterns

### Extracting 16-bit Values from lParam

```ahk
; For unsigned 16-bit values (e.g., window dimensions)
low := lParam & 0xFFFF           ; Low 16 bits
high := (lParam >> 16) & 0xFFFF  ; High 16 bits
```

### Extracting Signed 16-bit Values (Coordinates)

```ahk
; For signed coordinates (can be negative on multi-monitor)
x := (lParam << 48) >> 48        ; Sign-extend low 16 bits
y := (lParam << 32) >> 48        ; Sign-extend bits 16-31

; Alternative method
x := (lParam & 0xFFFF)
if (x > 0x7FFF)
    x -= 0x10000                 ; Convert to negative

y := ((lParam >> 16) & 0xFFFF)
if (y > 0x7FFF)
    y -= 0x10000
```

### Button/Key State Flags (wParam)

```ahk
MK_LBUTTON  := 0x0001  ; Left button down
MK_RBUTTON  := 0x0002  ; Right button down
MK_SHIFT    := 0x0004  ; Shift key down
MK_CONTROL  := 0x0008  ; Ctrl key down
MK_MBUTTON  := 0x0010  ; Middle button down

; Check flags
if (wParam & MK_LBUTTON)
    ; Left button is down
if (wParam & MK_CONTROL)
    ; Ctrl is pressed
```

---

## Essential Window Messages

### WM_SIZE (0x0005)

**When Sent:** After window size changes (maximize, minimize, resize).

**wParam Values:**

| Value | Constant | Meaning |
|-------|----------|---------|
| 0 | SIZE_RESTORED | Window resized (not min/max) |
| 1 | SIZE_MINIMIZED | Window minimized |
| 2 | SIZE_MAXIMIZED | Window maximized |
| 3 | SIZE_MAXSHOW | Other window restored |
| 4 | SIZE_MAXHIDE | Other window maximized |

**lParam Decoding:**

```ahk
width := lParam & 0xFFFF          ; New width
height := (lParam >> 16) & 0xFFFF ; New height
```

**Example:**

```ahk
OnMessage(0x0005, WM_SIZE_Handler)

WM_SIZE_Handler(wParam, lParam, msg, hwnd) {
    width := lParam & 0xFFFF
    height := (lParam >> 16) & 0xFFFF

    switch wParam {
        case 1:  ; SIZE_MINIMIZED
            ToolTip("Window minimized")
        case 2:  ; SIZE_MAXIMIZED
            ToolTip("Window maximized: " . width . "x" . height)
        case 0:  ; SIZE_RESTORED
            ToolTip("Window resized: " . width . "x" . height)
    }

    SetTimer(() => ToolTip(), -2000)
    return 0
}
```

---

### WM_MOVE (0x0003)

**When Sent:** After window moves.

**wParam:** Not used.

**lParam Decoding (signed coordinates):**

```ahk
x := (lParam << 48) >> 48    ; Sign-extend
y := (lParam << 32) >> 48
```

**Example:**

```ahk
OnMessage(0x0003, WM_MOVE_Handler)

WM_MOVE_Handler(wParam, lParam, msg, hwnd) {
    x := (lParam & 0xFFFF)
    y := ((lParam >> 16) & 0xFFFF)

    ; Handle negative coordinates (multi-monitor)
    if (x > 0x7FFF)
        x -= 0x10000
    if (y > 0x7FFF)
        y -= 0x10000

    ToolTip("Window moved to: " . x . ", " . y)
    SetTimer(() => ToolTip(), -2000)
    return 0
}
```

---

### WM_SYSCOMMAND (0x0112)

**When Sent:** System menu command (close, minimize, maximize, Alt+F4).

**wParam Decoding (CRITICAL - must mask low 4 bits):**

```ahk
cmd := wParam & 0xFFF0  ; MUST mask!
```

**Command Values:**

| Value | Constant | Action |
|-------|----------|--------|
| 0xF060 | SC_CLOSE | Close window |
| 0xF020 | SC_MINIMIZE | Minimize |
| 0xF030 | SC_MAXIMIZE | Maximize |
| 0xF120 | SC_RESTORE | Restore to normal |
| 0xF010 | SC_MOVE | Move window (keyboard) |
| 0xF000 | SC_SIZE | Size window (keyboard) |
| 0xF100 | SC_KEYMENU | Menu access via keyboard |
| 0xF140 | SC_SCREENSAVE | Execute screensaver |
| 0xF170 | SC_MONITORPOWER | Set monitor power state |

**lParam:** Mouse position if chosen with mouse.

**Example (Block Minimize):**

```ahk
OnMessage(0x0112, WM_SYSCOMMAND_Handler)

WM_SYSCOMMAND_Handler(wParam, lParam, msg, hwnd) {
    static SC_MINIMIZE := 0xF020

    cmd := wParam & 0xFFF0  ; MUST mask!

    if (cmd = SC_MINIMIZE) {
        ToolTip("Minimize blocked!")
        SetTimer(() => ToolTip(), -1000)
        return 0  ; Block - don't call DefWindowProc
    }

    ; Allow other commands
    return
}
```

---

### WM_CLOSE (0x0010)

**When Sent:** User clicks X or presses Alt+F4. This is a **request** to close.

**wParam/lParam:** Not used.

**Can Be Blocked:** Yes - return 0 without calling DefWindowProc.

**Default Behavior (DefWindowProc):** Calls DestroyWindow().

**Example (Confirm Close):**

```ahk
OnMessage(0x0010, WM_CLOSE_Handler)

WM_CLOSE_Handler(wParam, lParam, msg, hwnd) {
    result := MsgBox("Really close?", , "YesNo")
    if (result = "Yes")
        return  ; Allow DefWindowProc to call DestroyWindow
    else
        return 0  ; Cancel close
}
```

---

### WM_DESTROY (0x0002)

**When Sent:** Window is being destroyed (cannot be prevented).

**wParam/lParam:** Not used.

**Cannot Be Blocked:** Destruction already in progress.

**Purpose:** Cleanup resources, call PostQuitMessage() for main windows.

**Example:**

```ahk
OnMessage(0x0002, WM_DESTROY_Handler)

WM_DESTROY_Handler(wParam, lParam, msg, hwnd) {
    ; Cleanup
    FileAppend("Window destroyed at " . A_Now . "`n", "log.txt")

    ; For main window, signal app exit
    PostMessage(0x0012, 0, 0)  ; WM_QUIT
    return 0
}
```

---

### WM_QUIT (0x0012)

**Special:** Never sent to window procedure. Only retrieved by GetMessage().

**Not associated with any window** (hwnd is NULL).

**Posted by:** PostQuitMessage(exitCode).

**Effect:** Causes GetMessage() to return 0, terminating message loop.

---

### WM_WINDOWPOSCHANGING (0x0046)

**When Sent:** Before window size/position/Z-order changes.

**Purpose:** Modify pending changes, enforce constraints.

**wParam:** Not used.

**lParam:** Pointer to WINDOWPOS structure.

**WINDOWPOS Structure:**

```cpp
typedef struct tagWINDOWPOS {
    HWND hwnd;            // Window being changed
    HWND hwndInsertAfter; // Z-order position
    int x;                // New x position
    int y;                // New y position
    int cx;               // New width
    int cy;               // New height
    UINT flags;           // SWP_* flags
} WINDOWPOS;
```

**SWP Flags:**

| Value | Constant | Meaning |
|-------|----------|---------|
| 0x0001 | SWP_NOSIZE | Ignore cx, cy |
| 0x0002 | SWP_NOMOVE | Ignore x, y |
| 0x0004 | SWP_NOZORDER | Ignore hwndInsertAfter |
| 0x0040 | SWP_SHOWWINDOW | Show window |
| 0x0080 | SWP_HIDEWINDOW | Hide window |

**Example (Enforce Minimum Size):**

```ahk
OnMessage(0x0046, WM_WINDOWPOSCHANGING_Handler)

WM_WINDOWPOSCHANGING_Handler(wParam, lParam, msg, hwnd) {
    static MIN_WIDTH := 400
    static MIN_HEIGHT := 300

    ; Read WINDOWPOS structure
    flags := NumGet(lParam + 24, "UInt")

    ; Only process if size is changing
    if !(flags & 0x0001) {  ; Not SWP_NOSIZE
        cx := NumGet(lParam + 16, "Int")
        cy := NumGet(lParam + 20, "Int")

        ; Enforce minimum
        if (cx < MIN_WIDTH)
            NumPut("Int", MIN_WIDTH, lParam + 16)
        if (cy < MIN_HEIGHT)
            NumPut("Int", MIN_HEIGHT, lParam + 20)
    }

    return 0
}
```

---

### WM_NCHITTEST (0x0084)

**When Sent:** Cursor moves over window or mouse button pressed.

**Purpose:** Determine what part of window cursor is over.

**wParam:** Not used.

**lParam:** Mouse position in **screen coordinates**.

```ahk
xPos := lParam & 0xFFFF
yPos := (lParam >> 16) & 0xFFFF
```

**Return Values:**

| Value | Constant | Meaning | Cursor |
|-------|----------|---------|--------|
| 1 | HTCLIENT | Client area | Default |
| 2 | HTCAPTION | Title bar | Move cursor |
| 10 | HTLEFT | Left border | Resize left |
| 11 | HTRIGHT | Right border | Resize right |
| 12 | HTTOP | Top border | Resize up |
| 15 | HTBOTTOM | Bottom border | Resize down |
| 13 | HTTOPLEFT | Top-left corner | Resize diagonal |
| 14 | HTTOPRIGHT | Top-right corner | Resize diagonal |
| 16 | HTBOTTOMLEFT | Bottom-left corner | Resize diagonal |
| 17 | HTBOTTOMRIGHT | Bottom-right corner | Resize diagonal |
| 20 | HTCLOSE | Close button | Hand |
| 8 | HTMINBUTTON | Minimize button | Hand |
| 9 | HTMAXBUTTON | Maximize button | Hand |
| -1 | HTTRANSPARENT | Pass through to window below | N/A |

**Example (Make Entire Window Draggable):**

```ahk
OnMessage(0x0084, WM_NCHITTEST_Handler)

WM_NCHITTEST_Handler(wParam, lParam, msg, hwnd) {
    static HTCLIENT := 1, HTCAPTION := 2

    ; Get default hit test result
    result := DllCall("DefWindowProc", "Ptr", hwnd, "UInt", msg,
                      "Ptr", wParam, "Ptr", lParam, "Ptr")

    ; If cursor in client area, pretend it's in caption (draggable)
    if (result = HTCLIENT)
        return HTCAPTION

    return result
}
```

---

## Mouse Messages

### WM_MOUSEMOVE (0x0200)

**When Sent:** Mouse moves over window.

**wParam:** Button/key states (see flags below).

**lParam:** Cursor coordinates in client area (signed 16-bit).

**Frequency:** Can generate hundreds per second - keep handler fast!

**Example:**

```ahk
OnMessage(0x0200, WM_MOUSEMOVE_Handler)

WM_MOUSEMOVE_Handler(wParam, lParam, msg, hwnd) {
    static MK_LBUTTON := 0x0001

    x := (lParam << 48) >> 48
    y := (lParam << 32) >> 48

    if (wParam & MK_LBUTTON)
        ToolTip("Dragging at: " . x . ", " . y)

    return 0
}
```

### WM_LBUTTONDOWN (0x0201)

**When Sent:** Left mouse button pressed.

**wParam:** Button/key states.

**lParam:** Cursor coordinates.

### WM_LBUTTONUP (0x0202)

**When Sent:** Left mouse button released.

### WM_RBUTTONDOWN (0x0204) / WM_RBUTTONUP (0x0205)

Right mouse button.

### WM_MBUTTONDOWN (0x0207) / WM_MBUTTONUP (0x0208)

Middle mouse button.

### Mouse Button/Key State Flags (wParam)

| Value | Constant | Meaning |
|-------|----------|---------|
| 0x0001 | MK_LBUTTON | Left button down |
| 0x0002 | MK_RBUTTON | Right button down |
| 0x0004 | MK_SHIFT | Shift key down |
| 0x0008 | MK_CONTROL | Ctrl key down |
| 0x0010 | MK_MBUTTON | Middle button down |

**Check with bitwise AND:**

```ahk
if (wParam & 0x0001)  ; Left button
if (wParam & 0x0008)  ; Ctrl key
```

---

## Keyboard Messages

### WM_KEYDOWN (0x0100)

**When Sent:** Non-system key pressed.

**wParam:** Virtual-key code.

**lParam:** Packed key data (repeat count, scan code, extended key flag, etc.).

### WM_KEYUP (0x0101)

**When Sent:** Non-system key released.

### WM_CHAR (0x0102)

**When Sent:** After WM_KEYDOWN, provides character code.

**wParam:** Character code (considers Shift, Caps Lock, etc.).

### WM_SYSKEYDOWN (0x0104) / WM_SYSKEYUP (0x0105)

**When Sent:** System key pressed/released (Alt, F10, or Alt+key combinations).

---

## Message Sending Methods

### SendMessage

**Behavior:** Synchronous - blocks until window procedure returns.

**Returns:** LRESULT from window procedure.

**Use For:** Querying window state, control operations.

**AHK2 Syntax:**

```ahk
result := SendMessage(msg, wParam, lParam, control?, winTitle?)

; Examples
result := SendMessage(0x0010, 0, 0, , "ahk_class Notepad")  ; WM_CLOSE
if (result = "ERROR")
    MsgBox "SendMessage failed or timed out"

length := SendMessage(0x000E, 0, 0, , "ahk_class Notepad")  ; WM_GETTEXTLENGTH
```

**Default Timeout:** 5000ms.

**Performance:** ~0.9 microseconds (same thread).

### PostMessage

**Behavior:** Asynchronous - queues message and returns immediately.

**Returns:** Boolean success/failure of *posting* (not processing).

**Use For:** Fire-and-forget notifications, avoiding deadlocks.

**AHK2 Syntax:**

```ahk
success := PostMessage(msg, wParam, lParam, control?, winTitle?)

; Examples
success := PostMessage(0x0010, 0, 0, , "ahk_class Notepad")  ; WM_CLOSE
if (!success)
    MsgBox "Failed to post message"

; Change keyboard layout
PostMessage(0x0050, 0, 0x4090409, , "A")  ; WM_INPUTLANGCHANGEREQUEST
```

**Performance:** ~0.7 microseconds to post.

**Queue Limit:** 10,000 messages per queue.

### Quick Comparison

| Feature | SendMessage | PostMessage |
|---------|-------------|-------------|
| **Execution** | Synchronous (blocks) | Asynchronous (queues) |
| **Returns** | WndProc result | Post success/fail |
| **Use for** | Querying state | Notifications |
| **Queue** | Bypasses | Goes through |
| **Deadlock risk** | High | Low |

---

## Common Message Patterns

### Get Window Text

```ahk
; Get text length
length := SendMessage(0x000E, 0, 0, , "ahk_id " hwnd)  ; WM_GETTEXTLENGTH

; Allocate buffer
buf := Buffer((length + 1) * 2)  ; Unicode

; Get text
SendMessage(0x000D, length + 1, buf.Ptr, , "ahk_id " hwnd)  ; WM_GETTEXT

; Convert to string
text := StrGet(buf)
```

### Set Window Text

```ahk
; Create buffer with text
buf := Buffer(StrPut(text, "UTF-16"))
StrPut(text, buf, "UTF-16")

; Send WM_SETTEXT
SendMessage(0x000C, 0, buf.Ptr, , "ahk_id " hwnd)
```

### Close Window

```ahk
; Send WM_CLOSE (can be intercepted)
SendMessage(0x0010, 0, 0, , "ahk_class Notepad")

; Or post (non-blocking)
PostMessage(0x0010, 0, 0, , "ahk_class Notepad")
```

### Activate Window

```ahk
; Send WM_ACTIVATE
SendMessage(0x0006, 1, 0, , "ahk_id " hwnd)  ; WA_ACTIVE = 1
```

---

## Performance Considerations

### High-Frequency Messages

**WM_MOUSEMOVE (0x0200):**
- Can generate 100-1000 messages/second
- Keep handler extremely fast (<1ms)
- Consider throttling or deferring heavy work

```ahk
OnMessage(0x0200, QuickHandler)

QuickHandler(wParam, lParam, msg, hwnd) {
    static lastUpdate := 0

    ; Throttle to 60 Hz
    now := A_TickCount
    if (now - lastUpdate < 16)
        return 0

    lastUpdate := now

    ; Process mouse move
    ProcessMove(lParam)
    return 0
}
```

### Defer Heavy Work

```ahk
OnMessage(0x0200, FastHandler)

FastHandler(wParam, lParam, msg, hwnd) {
    ; Save data quickly
    static savedX := 0, savedY := 0
    savedX := lParam & 0xFFFF
    savedY := (lParam >> 16) & 0xFFFF

    ; Defer processing
    SetTimer(ProcessData, -50)  ; Process after 50ms, once
    return 0
}

ProcessData() {
    ; Expensive work here
}
```

---

## Debugging Messages

### Log to File

```ahk
OnMessage(0x0200, LogHandler)

LogHandler(wParam, lParam, msg, hwnd) {
    timestamp := FormatTime(, "HH:mm:ss.") . Mod(A_TickCount, 1000)
    line := Format("{1} - Msg:{2:04X} wParam:{3} lParam:{4}`n",
                   timestamp, msg, wParam, lParam)
    FileAppend(line, "message_log.txt")
    return 0
}
```

### Use OutputDebug + DebugView

```ahk
OnMessage(0x0200, DebugHandler)

DebugHandler(wParam, lParam, msg, hwnd) {
    OutputDebug(Format("AHK| Msg:{:04X} wParam:{} lParam:{}",
                      msg, wParam, lParam))
    return 0
}
```

---

## Limitations and Gotchas

### UAC/UIPI

**Problem:** Cannot send messages to higher-privilege windows.

**Solution:** Run script as administrator.

### 64-bit vs 32-bit Spy++

**Problem:** 32-bit Spy++ shows ZERO messages from 64-bit processes.

**Solution:** Use matching architecture (spyxx_amd64.exe for 64-bit).

### Message Queue Limit

**Problem:** PostMessage fails when queue exceeds 10,000 messages.

**Solution:** Check return value, implement backpressure.

### DirectX Games

**Problem:** Games often bypass Windows messages for input.

**Solution:** Use SendPlay, hardware-level input, or accept limitation.

---

## See Also

- docs/manuals/ahk2/reference/error-types.md - Error handling
- Official: [SendMessage](https://www.autohotkey.com/docs/v2/lib/SendMessage.htm)
- Official: [PostMessage](https://www.autohotkey.com/docs/v2/lib/PostMessage.htm)
- Official: [OnMessage](https://www.autohotkey.com/docs/v2/lib/OnMessage.htm)
- MSDN: [Window Messages](https://docs.microsoft.com/en-us/windows/win32/winmsg/windowing)
