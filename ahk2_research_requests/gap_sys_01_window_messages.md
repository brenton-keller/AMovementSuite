# AHK2 Research Request: Window Messages & Message Hooks

## What I Need to Learn

I'm moving windows with while loops polling mouse position. I suspect there's a better way using Windows messages. I need to understand **how Windows is fundamentally message-driven** and how to intercept/send/manipulate messages for powerful automation.

## My Current Understanding (Challenge This!)

I believe:
- Everything in Windows is a message (clicks, moves, resizes)
- WM_MOVE is sent when window moves
- I can intercept messages with hooks
- SendMessage is like SendInput for messages

**Where am I wrong? What am I missing?**

## The Core Problem I'm Solving

**Current code** (from my project):
```ahk
While (GetKeyState("RButton", "P") && GetKeyState("LAlt", "P")) {
    MouseGetPos &currentX, &currentY
    newX := Round(currentX - offsetX)
    newY := Round(currentY - offsetY)
    WinMove newX, newY, , , "ahk_id " mwin
    // Polling 60+ times per second, CPU intensive
}
```

**What I want**:
- React to WM_MOVE messages instead of polling
- Intercept window close (WM_CLOSE) to save position
- Detect when ANY window is created/destroyed system-wide
- Know when monitor configuration changes

## Specific Questions

1. **Message Flow**: When I move a window, what messages are sent, in what order, and can I intercept them?

2. **Hooks vs Polling**: I'm polling mouse position in a loop. Could a WH_MOUSE hook eliminate this? What's the trade-off?

3. **SendMessage vs PostMessage**: When do I use each? What's the difference conceptually?

4. **WH_CBT Hook**: I want to detect when ANY window is created. How do I set up a shell hook? What's the performance impact?

5. **Message Parameters**: Messages have WPARAM and LPARAM. How do I decode these? Show examples for common messages.

## Example Messages I Need to Understand

Research these IN DEPTH with examples:

**WM_SIZE (0x0005)**: Window resized
- When is it sent vs WM_SIZING?
- What are wParam values (SIZE_MINIMIZED, SIZE_MAXIMIZED, etc.)?
- How to get new dimensions from lParam?

**WM_MOVE (0x0003)**: Window moved
- When sent (after move or during drag)?
- How to get position from lParam?
- Can I prevent the move?

**WM_SYSCOMMAND (0x0112)**: System menu/window operations
- What are the wParam values (SC_MINIMIZE, SC_MAXIMIZE, etc.)?
- How to intercept window minimize and cancel it?
- How to add custom system menu items?

**WM_CLOSE (0x0010)**: Window closing
- Difference vs WM_DESTROY vs WM_QUIT?
- How to intercept and prevent close?
- How to save state before close?

## How to Write This Guide

### 1. Start with Windows Fundamentals
Before showing AHK2 code, explain **how Windows actually works**:

```
User clicks button
  ↓
Hardware generates interrupt
  ↓
Driver creates WM_LBUTTONDOWN message
  ↓
Message posted to thread's message queue
  ↓
GetMessage retrieves from queue
  ↓
DispatchMessage calls window procedure (WndProc)
  ↓
WndProc handles message or calls DefWindowProc
```

Explain WHERE hooks intercept this flow.

### 2. Message Anatomy
Show the structure:
```c
MSG {
    HWND hwnd;      // Window that receives message
    UINT message;   // Message ID (WM_*)
    WPARAM wParam;  // Additional info (often flags/IDs)
    LPARAM lParam;  // Additional info (often packed coordinates)
    DWORD time;     // When posted
    POINT pt;       // Mouse position when posted
}
```

Then show how to read wParam/lParam for each message type.

### 3. Hook Types Comparison Table

| Hook Type | Scope | Use Case | Performance Impact |
|-----------|-------|----------|-------------------|
| WH_KEYBOARD | System | Global key intercept | Low |
| WH_MOUSE | System | Global mouse intercept | Low |
| WH_CBT | System | Window creation/focus | Medium |
| WH_CALLWNDPROC | Thread | Monitor messages | High |
| WH_GETMESSAGE | Thread | Modify before dispatch | Medium |

Then explain WHEN to use each.

### 4. Progressive Examples

**Level 1: Send a Simple Message**
```ahk
; Close a window by sending WM_CLOSE
WM_CLOSE := 0x0010
SendMessage(WM_CLOSE, 0, 0, "ahk_id " windowID)
// Explain every parameter
```

**Level 2: Read Message Parameters**
```ahk
; Get window position from WM_MOVE
OnWM_MOVE(wParam, lParam, msg, hwnd) {
    x := lParam & 0xFFFF          ; Low word = X
    y := (lParam >> 16) & 0xFFFF  ; High word = Y
    MsgBox "Window moved to " x "," y
}

// How to register this handler?
// How does bit manipulation work here?
```

**Level 3: Intercept and Modify**
```ahk
; Prevent window from moving to certain position
OnWM_WINDOWPOSCHANGING(wParam, lParam, msg, hwnd) {
    ; lParam points to WINDOWPOS structure
    ; How do I read the structure?
    ; How do I modify it?
    ; How do I signal "handled"?
}
```

**Level 4: System-Wide Hook**
```ahk
; Detect ALL windows created system-wide
SetWindowsHookEx(WH_CBT, callback, hMod, threadID)
// Complete working example
// How to install hook
// How to process events
// How to uninstall safely
```

### 5. The "How I'd Actually Use This" Section

Show complete, production-ready solutions:

**Use Case 1: Save Window Positions on Close**
```ahk
; Intercept WM_CLOSE, save position, allow close
// Complete implementation
// Error handling
// Restoration on next launch
```

**Use Case 2: Auto-Launch When Specific App Opens**
```ahk
; Use WH_CBT to detect when Chrome launches
// Execute action when detected
// Clean up hook when done
```

**Use Case 3: Custom Window Drag**
```ahk
; Make entire client area draggable (not just title bar)
// Intercept WM_NCHITTEST
// Return HTCAPTION to make area draggable
```

### 6. Performance Reality

Benchmark:
```
Polling in While loop (60Hz): 1.5% CPU, 16ms response time
Message hook: 0.1% CPU, 1ms response time

Conclusion: Messages are 15x more efficient and 16x more responsive
```

### 7. Debugging Messages

Teach me how to debug:
- How to log all messages a window receives
- How to identify which message does what
- How to use Spy++ or AHK2 alternatives
- How to test message handlers

### 8. The Gotchas

**"Why doesn't my hook work?"**
- Hooks must be in persistent script
- System hooks require DLL (can't do pure AHK2)
- UAC windows filter messages
- Some messages can't be blocked

**"Why is my app slow?"**
- Processing every WM_MOUSEMOVE kills performance
- Filter messages you don't care about
- Don't block in message handlers

### 9. When NOT to Use Messages

Be honest about limitations:
- Can't hook UAC/admin windows from non-admin script
- Can't hook DirectX/game windows reliably
- System hooks complex (DLL required)
- Polling sometimes simpler for quick scripts

## Success Criteria

After this guide:
1. ✓ Can explain the message flow from hardware to application
2. ✓ Know when to use SendMessage vs PostMessage vs SendInput
3. ✓ Can intercept window close and save state
4. ✓ Can detect window creation system-wide
5. ✓ Understand wParam/lParam decoding for common messages
6. ✓ Refactor my polling loops to use messages where appropriate

## Most Important

Don't just show me how to call SendMessage. Teach me **how Windows itself works** so I can reason about message flow and debug when things don't work.

I want to understand the message queue, the window procedure, and where hooks intercept - deeply enough to troubleshoot any message-related issue.
