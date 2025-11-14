# Explanation: Windows Message Architecture

**Understanding the complete message flow from hardware to your code**

Windows is fundamentally event-driven, but most developers never learn how messages actually flow through the system. This creates confusion about posted vs. sent messages, hook interception points, and why polling is inefficient. Understanding the message architecture transforms you from writing wasteful polling loops to crafting efficient, responsive automation that works *with* the operating system.

## The Complete Message Journey

When you press a key or move your mouse, the input travels through a precise chain before reaching your application code:

```
HARDWARE
   ↓
Device Driver (converts electrical signal → message)
   ↓
System Message Queue (central clearinghouse)
   ↓
Thread Message Queue (per-thread, holds messages for all windows on that thread)
   ↓
GetMessage() (retrieves message from queue, BLOCKS if empty)
   ↓
DispatchMessage() (routes to appropriate window procedure)
   ↓
Window Procedure (WndProc) - Your handler code
   ↓
DefWindowProc() (default behavior for unhandled messages)
```

### Critical Insight: Thread Queues, Not Window Queues

**There is no such thing as a window message queue.** Each GUI thread has **one queue** that holds messages for **all windows** on that thread.

This explains why:
- A hung window freezes all windows on the same thread
- Blocking in a message handler prevents other windows from updating
- Cross-thread SendMessage can deadlock

## Posted vs. Sent Messages: The Architecture Difference

### Posted Messages (PostMessage)

**Path:** Message → Thread Queue → GetMessage → DispatchMessage → WndProc

```
┌──────────────┐
│ PostMessage  │
└──────┬───────┘
       ↓
┌────────────────────────┐
│ Thread Message Queue   │
│ [Msg1][Msg2][Msg3]...  │
└──────┬─────────────────┘
       ↓ (when thread pumps message loop)
┌──────────────┐
│ GetMessage   │
└──────┬───────┘
       ↓
┌─────────────────┐
│ DispatchMessage │
└──────┬──────────┘
       ↓
┌──────────┐
│ WndProc  │
└──────────┘
```

**Characteristics:**
- **Asynchronous** - PostMessage returns immediately
- **Cannot get return value** from window procedure
- **Goes through message queue** - follows GetMessage/DispatchMessage cycle
- **Used for:** WM_KEYDOWN, WM_LBUTTONDOWN, WM_TIMER

**Performance:** ~0.7 microseconds to post
**Queue limit:** 10,000 messages per thread queue

### Sent Messages (SendMessage)

**Path:** Direct call to WndProc (bypasses queue entirely)

```
┌──────────────┐
│ SendMessage  │
└──────┬───────┘
       ↓ (DIRECT CALL - no queue)
┌──────────┐
│ WndProc  │
└──────┬───┘
       ↓
┌──────────────┐
│ SendMessage  │ Returns the WndProc result
└──────────────┘
```

**Characteristics:**
- **Synchronous** - SendMessage blocks until WndProc returns
- **Returns actual result** from window procedure
- **Bypasses message queue** completely
- **Used for:** WM_SETTEXT, WM_ACTIVATE, WM_SETCURSOR

**Performance:** ~0.9 microseconds (same-thread)
**Risk:** Can deadlock if both threads send to each other

### Same-Thread vs. Cross-Thread

**Same-thread SendMessage:** Literally just a function call on the stack. No special handling.

**Cross-thread SendMessage:**
1. Sender thread calls SendMessage
2. **Sender blocks** (waits)
3. Target thread pumps message loop
4. Target thread processes message
5. Result returned to sender
6. Sender unblocks and receives result

**Deadlock scenario:**
```
Thread A: SendMessage to Thread B (blocks waiting)
Thread B: SendMessage to Thread A (blocks waiting)
→ Both threads stuck forever
```

## The MSG Structure: Anatomy of a Message

Every message is packaged in this structure:

```c
typedef struct tagMSG {
    HWND   hwnd;       // Window that receives message
    UINT   message;    // Message identifier (WM_KEYDOWN, etc.)
    WPARAM wParam;     // First parameter (meaning varies)
    LPARAM lParam;     // Second parameter (meaning varies)
    DWORD  time;       // Timestamp (milliseconds since boot)
    POINT  pt;         // Cursor position (screen coordinates)
} MSG;
```

### Parameter Encoding Patterns

**wParam** - Originally 16-bit, now pointer-sized (32/64-bit)
- Button/key states (MK_LBUTTON, MK_SHIFT, MK_CONTROL)
- Virtual key codes
- Control IDs

**lParam** - Originally 32-bit, now pointer-sized
- **Packed coordinates:** Low 16 bits = X, High 16 bits = Y
- **Structure pointers:** WINDOWPOS, NCCALCSIZE_PARAMS
- **Repeat counts and scan codes**

**Critical:** Multi-monitor systems have negative coordinates. Simple `lParam & 0xFFFF` fails—you must cast to signed 16-bit:

```ahk
; WRONG - Negative coordinates become huge positive values
xPos := lParam & 0xFFFF

; RIGHT - Handles negative coordinates
xPos := (lParam << 48) >> 48  ; Sign-extend low 16 bits
```

## Where Hooks Intercept Messages

Hooks insert themselves at specific points in the message flow:

```
HARDWARE EVENT
   ↓
┌────────────────────────────────────┐
│ WH_KEYBOARD_LL / WH_MOUSE_LL       │ ← Low-level hooks (before queue)
└────────────────────────────────────┘
   ↓
System Queue → Thread Queue
   ↓
┌────────────────────────────────────┐
│ WH_GETMESSAGE                      │ ← After GetMessage retrieves
└────────────────────────────────────┘
   ↓
┌────────────────────────────────────┐
│ WH_CALLWNDPROC                     │ ← Before SendMessage calls WndProc
└────────────────────────────────────┘
   ↓
Window Procedure
```

### Low-Level Hooks (WH_KEYBOARD_LL, WH_MOUSE_LL)

**Interception point:** Between system queue and thread queue

**Characteristics:**
- See events **before** Windows posts to any application
- Run in **your process** (no DLL injection needed)
- System-wide monitoring without DLL
- **High performance impact** - processes every keystroke/mouse move

**AHK2 support:** Native via hotkeys with `$` prefix or `#UseHook`

### Message Hooks (WH_GETMESSAGE)

**Interception point:** After GetMessage retrieves, before DispatchMessage

**Characteristics:**
- Intercepts **posted messages** only (not sent)
- Thread-specific or system-wide (requires DLL)
- Sees message after it leaves queue

### Call Hooks (WH_CALLWNDPROC)

**Interception point:** Before SendMessage invokes WndProc

**Characteristics:**
- Intercepts **sent messages** only (not posted)
- Extremely high impact on Windows 10+ (6x slower than Win7)
- Requires DLL for system-wide

### CBT Hooks (WH_CBT)

**Interception point:** Before window creation, activation, destruction

**Characteristics:**
- Fires **before WM_CREATE** sent
- Can **prevent** window creation by returning 1
- Can modify creation parameters
- Requires DLL for system-wide

**Use case:** Detecting window creation across all apps

## Decoding Common Messages

### WM_SIZE (0x0005) - Window Resized

**wParam values:**
```ahk
SIZE_RESTORED   := 0  ; Resized but not min/max
SIZE_MINIMIZED  := 1  ; Window minimized
SIZE_MAXIMIZED  := 2  ; Window maximized
```

**lParam decoding:**
```ahk
width  := lParam & 0xFFFF        ; Low 16 bits
height := (lParam >> 16) & 0xFFFF ; High 16 bits
```

**Sent:** After size change completes (not during drag)

### WM_MOVE (0x0003) - Window Moved

**lParam decoding (handles negative coordinates):**
```ahk
xPos := (lParam << 48) >> 48  ; Sign-extend low 16 bits
yPos := (lParam << 32) >> 48  ; Sign-extend bits 16-31
```

**Why sign extension?** Multi-monitor setups can place windows at negative coordinates (e.g., -100, -50 on left monitor).

### WM_SYSCOMMAND (0x0112) - System Menu Command

**Critical detail:** Must mask low 4 bits!

```ahk
cmd := wParam & 0xFFF0  ; MUST mask - Windows uses low 4 bits internally
```

**Command values:**
```ahk
SC_CLOSE    := 0xF060  ; Close window
SC_MINIMIZE := 0xF020  ; Minimize
SC_MAXIMIZE := 0xF030  ; Maximize
SC_RESTORE  := 0xF120  ; Restore
```

**Sent:** Before action occurs - you can intercept and cancel

**Blocking minimize:**
```ahk
OnMessage(0x0112, WM_SYSCOMMAND_Handler)

WM_SYSCOMMAND_Handler(wParam, lParam, msg, hwnd) {
    if ((wParam & 0xFFF0) = 0xF020)  ; SC_MINIMIZE
        return 0  ; Cancel - don't call DefWindowProc
}
```

### Window Shutdown Sequence

Three messages form the close lifecycle:

**WM_CLOSE (0x0010):**
- User clicks X or presses Alt+F4
- This is a **request** to close
- You can intercept and cancel
- DefWindowProc calls DestroyWindow()

**WM_DESTROY (0x0002):**
- Window is being destroyed
- Cannot cancel - already in progress
- Window removed from screen, handle still valid
- Call PostQuitMessage(0) for main window

**WM_QUIT (0x0012):**
- Special message that terminates message loop
- **Never sent to WndProc** - only retrieved by GetMessage
- Causes GetMessage to return 0
- Must be posted with PostQuitMessage(), not SendMessage

### WM_NCHITTEST (0x0084) - Hit Test

**Purpose:** Windows asks "what part of the window is the cursor over?"

**Return values determine behavior:**
```ahk
HTCAPTION  := 2   ; Title bar (draggable)
HTCLIENT   := 1   ; Client area
HTLEFT     := 10  ; Left border (resize)
HTRIGHT    := 11  ; Right border
HTTOP      := 12  ; Top border
HTBOTTOM   := 15  ; Bottom border
HTTRANSPARENT := -1  ; Pass through to window below
```

**Making entire client area draggable:**
```ahk
OnMessage(0x0084, WM_NCHITTEST_Handler)

WM_NCHITTEST_Handler(wParam, lParam, msg, hwnd) {
    static HTCLIENT := 1, HTCAPTION := 2

    ; Get default hit test result
    result := DllCall("DefWindowProc", "Ptr", hwnd, "UInt", msg,
                      "Ptr", wParam, "Ptr", lParam, "Ptr")

    ; If cursor in client area, pretend it's in caption
    if (result = HTCLIENT)
        return HTCAPTION

    return result  ; Preserve border/button behavior
}
```

## OnMessage() in AutoHotkey v2

AHK2's message interception mechanism for script windows.

### Syntax

```ahk
OnMessage(MsgNumber, FunctionObject, MaxThreads := 1)
```

**Handler signature:**
```ahk
HandlerFunction(wParam, lParam, msg, hwnd) {
    ; Process message
    return 0  ; or integer to suppress further processing
}
```

### Return Value Behavior

**return 0 or empty:** Allow message to continue to DefWindowProc
**return non-zero integer:** Suppress further processing, return this value

### Multiple Handlers

If you register multiple handlers for the same message:
- Called in **reverse registration order** (newest first)
- If any returns non-empty, later handlers are skipped

### Thread Priority and Critical

OnMessage handlers always run at **priority 0**.

If your script executes high-priority code (`Priority > 0`), messages are **not monitored**—they're lost.

**Critical behavior:**
```ahk
HandlerFunction(wParam, lParam, msg, hwnd) {
    Critical 1000  ; Buffer messages ≥0x312 for 1 second
    ; Your code here
}
```

**Messages below 0x0312:** Cannot be buffered. Must finish processing in **under 10ms** or subsequent messages are missed.

**Default check interval:** 8ms between queue checks

## Performance: Polling vs. Message-Driven

### Polling Approach (Wasteful)

```ahk
; CPU-intensive polling at 60Hz
While (GetKeyState("RButton", "P") && GetKeyState("LAlt", "P")) {
    MouseGetPos(&currentX, &currentY)
    newX := Round(currentX - offsetX)
    newY := Round(currentY - offsetY)
    WinMove(newX, newY, , , "ahk_id " mwin)
    Sleep(16)  ; ~60 FPS
}
```

**Costs:**
- **CPU usage:** 1-5% continuously (even when idle)
- **Latency:** Average 8ms, worst case 16ms
- **Missed events:** Events between polls lost
- **Power consumption:** CPU never truly idle
- **Battery impact:** High on laptops

### Message-Driven Approach (Efficient)

```ahk
OnMessage(0x0200, WM_MOUSEMOVE_Handler)

WM_MOUSEMOVE_Handler(wParam, lParam, msg, hwnd) {
    static dragging := false

    if (GetKeyState("RButton", "P") && GetKeyState("LAlt", "P")) {
        if (!dragging) {
            ; Start drag
            MouseGetPos(&startX, &startY, &targetHwnd)
            WinGetPos(&winX, &winY, , , "ahk_id " targetHwnd)
            dragging := true
        }

        ; Update position
        x := (lParam & 0xFFFF) - offsetX
        y := ((lParam >> 16) & 0xFFFF) - offsetY
        WinMove(x, y, , , "ahk_id " targetHwnd)
    } else if (dragging) {
        dragging := false
    }
}
```

**Benefits:**
- **CPU usage:** <0.1% when idle
- **Latency:** <1ms (near-instant response)
- **Missed events:** None - all buffered in queue
- **Power consumption:** Minimal
- **Battery impact:** Low

### Quantified Comparison

| Metric | Polling (60Hz) | Message-Driven |
|--------|---------------|----------------|
| Idle CPU | 1-5% | <0.1% |
| Active CPU | 5-10% | 2-5% |
| Avg Latency | 8ms | <1ms |
| Max Latency | 16ms | <2ms |
| Events/sec | 60 | 100-1000+ |

**Efficiency improvement:** 70-90% CPU reduction in typical scenarios

## Hook Capabilities in AHK2

### Native Support (No DLL)

**WH_KEYBOARD_LL:**
- Via hotkeys with `$` prefix
- `#UseHook` directive
- System-wide keyboard monitoring

**WH_MOUSE_LL:**
- `#InstallMouseHook` directive
- System-wide mouse monitoring

**Thread-specific hooks:**
- Limited usefulness (only affect your windows)

### Requires DLL

**System-wide:**
- WH_CBT (window creation monitoring)
- WH_SHELL
- WH_CALLWNDPROC
- Any global hook except low-level

### Detecting Window Creation

**Method 1: RegisterShellHookWindow (No DLL)**

```ahk
; Register to receive shell hook messages
shellMsg := DllCall("RegisterWindowMessage", "Str", "SHELLHOOK", "UInt")
DllCall("RegisterShellHookWindow", "Ptr", hwnd)

OnMessage(shellMsg, ShellHookHandler)

ShellHookHandler(wParam, lParam, msg, hwnd) {
    static HSHELL_WINDOWCREATED := 1
    if (wParam = HSHELL_WINDOWCREATED)
        MsgBox("Window created: " lParam)  ; lParam is new window handle
}
```

**Limitation:** Only fires for **top-level windows**, not child windows.

**Method 2: WH_CBT with HCBT_CREATEWND (Requires DLL)**
- Fires **before** WM_CREATE
- Can prevent creation by returning 1
- Sees all windows (top-level and child)

## Critical Limitations

### UAC/UIPI (User Interface Privilege Isolation)

Windows blocks messages between different integrity levels:

**Scenario:** Normal user script tries to send to administrator window
**Result:** SendMessage/PostMessage **silently fail** or return ERROR

**Affected:** Elevated windows, UAC prompts, Windows Defender

**Solutions:**
1. Run script as administrator
2. Use Task Scheduler with highest privileges
3. Accept limitation—some windows deliberately unreachable

### DirectX and Game Windows

Most games don't use standard Windows messaging for input:
- DirectX uses driver-level input
- Fullscreen exclusive mode bypasses Windows
- Anti-cheat detects automation

**Limited workarounds:**
- Use Windowed or Borderless Windowed mode
- Try SendPlay (lower-level simulation)
- Increase SetKeyDelay to 25-50ms

**Reality check:** Many games deliberately resist automation

### High-Frequency Message Performance

**WM_MOUSEMOVE (0x0200)** generates hundreds per second:

**Problem:**
- Each message takes 1-2ms to process
- Handlers must return in **under 10ms** (for messages <0x312)
- Subsequent messages lost if handler too slow

**Best practices:**
```ahk
OnMessage(0x0200, QuickMouseHandler)

QuickMouseHandler(wParam, lParam, msg, hwnd) {
    ; Save data quickly
    static savedX := 0, savedY := 0
    savedX := lParam & 0xFFFF
    savedY := (lParam >> 16) & 0xFFFF

    ; Defer heavy processing
    SetTimer(ProcessMouseData, -50)  ; Process after 50ms, once
    return 0
}

ProcessMouseData() {
    ; Do expensive work here
}
```

## When to Use Each Approach

### Use OnMessage when:
- Monitoring messages to your own GUI windows
- Custom inter-script communication (WM_COPYDATA)
- System notifications (WM_POWERBROADCAST, WM_DISPLAYCHANGE)
- You control the windows receiving messages

### Use hooks when:
- System-wide keyboard/mouse monitoring
- Detecting window creation/destruction across all apps
- Low-level input capture before applications see it
- Need to intercept before queueing

### Use polling when:
- Game state that doesn't generate messages
- Continuous analog input (joystick positions)
- Simple scripts where complexity isn't justified
- Fixed-rate update loops (animations)
- Testing/prototyping for quick iterations

## Key Insights

**Insight 1: Thread queues, not window queues**
Understanding that each thread has one queue for all its windows explains blocking behavior and cross-thread issues.

**Insight 2: Posted vs. sent is architectural**
Posted goes through queue (async), sent bypasses queue (sync). This determines performance, return values, and deadlock risks.

**Insight 3: Hooks intercept at specific points**
Low-level hooks see input before queueing. GetMessage hooks see after retrieval. CallWndProc sees sent messages before WndProc.

**Insight 4: Message-driven is drastically more efficient**
70-90% CPU reduction compared to polling, with lower latency and no missed events.

**Insight 5: wParam/lParam encoding varies by message**
No universal rule—you must decode based on specific message documentation. Watch for signed coordinates and packed values.

## Cross-References

Related topics:
- docs/manuals/ahk2/how-to/gui-event-handling.md - Practical GUI patterns
- docs/manuals/ahk2/reference/onmessage.md - OnMessage() reference
- docs/manuals/ahk2/how-to/window-automation.md - SendMessage/PostMessage usage
- docs/manuals/ahk2/explanation/error-handling-philosophy.md - Handling message failures
