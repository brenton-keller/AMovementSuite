# AHK2 Research Request: Keyboard & Mouse Input Internals

## What I Need to Learn

My mouse hotkeys work but I don't understand WHY some apps ignore them. I need to understand **the Windows input stack** - from hardware to application - so I can troubleshoot when input doesn't work.

## My Current Understanding (Challenge This!)

I believe:
- Send simulates user input
- SendInput is faster than Send
- Admin windows block input for security
- Mouse hooks can intercept all mouse events
- Scan codes and virtual keys are the same thing

**What am I missing?**

## The Mystery

Why does this work sometimes but not always:
```ahk
RAlt & F1:: {
    Send "^t"  // Open new tab
    // Works in Chrome, doesn't work in some apps - WHY?
}
```

And why can't I send input to admin windows even from admin script?

## Specific Questions

1. **Input Stack**: What's the path from keyboard hardware to application? Where does AHK2 inject?

2. **Send vs SendInput vs SendEvent vs SendPlay**: What's the ACTUAL difference? When does each fail?

3. **Scan Codes vs Virtual Key Codes**: What are they? When do I need scan codes?

4. **Input Blocking**: Why do some apps ignore SendInput? Why do games ignore it? What's the defense mechanism?

5. **Raw Input**: What is raw input? How is it different? Can AHK2 use it?

## How to Write This Guide

### 1. The Complete Input Stack

Diagram the FULL path:
```
Keyboard Hardware
  ↓
USB HID Driver
  ↓
i8042prt (keyboard port driver)
  ↓
Keyboard Class Driver
  ↓
Raw Input API (if app uses it)
  ↓
Low-Level Keyboard Hook ← AHK2 hotkeys intercept here
  ↓
Window Message Queue (WM_KEYDOWN, WM_CHAR, etc.)
  ↓
Application's Message Loop
  ↓
WndProc processes input
```

Show where each Send method injects and why some apps ignore it.

### 2. Send Methods Comparison

Complete table:

| Method | Injection Point | Speed | Blockable? | Queued? | Use Case |
|--------|----------------|-------|------------|---------|----------|
| Send | Hook level | Slow | Yes | No | General use |
| SendInput | Pre-hook | Fast | Yes | Yes | Burst input |
| SendEvent | Message queue | Medium | Easy | Yes | Compatibility |
| SendPlay | Playback driver | Slow | Harder | No | Games/anti-cheat |
| ControlSend | Direct to window | Fast | Hardest | No | Background windows |

Then explain each injection point visually.

### 3. Scan Codes Deep Dive

Explain the difference:
```ahk
Send "{VK41}"  // Virtual key code for 'A'
Send "{SC01E}" // Scan code for 'A'

// On QWERTY keyboard: Both send 'A'
// On DVORAK keyboard: VK41 = 'A', SC01E = different key!

// When does this matter?
```

Show keyboard layout diagram and mapping.

### 4. Why Input Gets Blocked

Explain defense mechanisms:

**UIPI (User Interface Privilege Isolation)**:
```
Non-admin process → Admin process: ❌ Blocked
Admin process → Admin process: ✓ Works
Admin process → Non-admin process: ✓ Works

WHY: Prevents malware from controlling elevated processes
```

**DirectInput (Games)**:
```
Game uses DirectInput API
  ↓
Bypasses Windows message queue
  ↓
Reads directly from hardware
  ↓
SendInput injected into message queue
  ↓
Game never sees it ❌
```

**Anti-cheat systems**:
- Detect non-human timing
- Detect driver-level input vs hardware
- Block known automation tools

### 5. Workarounds for Blocked Input

Progressive solutions:

**Level 1: Try SendPlay**
```ahk
Send "{Tab}"     // Blocked
SendPlay "{Tab}" // Works? Why?
```

**Level 2: ControlSend**
```ahk
ControlSend "text", "Edit1", "ahk_id " hwnd
// Sends directly to control, bypasses hooks
```

**Level 3: Simulate at message level**
```ahk
; Send WM_KEYDOWN directly
WM_KEYDOWN := 0x0100
VK_TAB := 0x09
PostMessage WM_KEYDOWN, VK_TAB, 0, "ControlNN", "ahk_id " hwnd
```

**Level 4: Physical input simulation**
```ahk
// Is this possible? Hardware-level input?
// Or are we limited to software methods?
```

### 6. Real-World Examples

**Example 1: Reliable tab switch**
```ahk
; Works in most apps
SwitchTab() {
    ; Try methods in order until one works
    try SendInput("^{Tab}")
    catch try SendEvent("^{Tab}")
    catch ControlSend "^{Tab}", , "A"
}
```

**Example 2: Type in background window**
```ahk
; Type without stealing focus
TypeInBackground(text, windowID) {
    ; How to send to specific window?
    ; How to avoid activating it?
}
```

**Example 3: Macro with precise timing**
```ahk
; Game macro - must look human
GameMacro() {
    ; How to add human-like delays?
    ; How to vary timing slightly?
    ; How to avoid detection?
}
```

### 7. Input Timing & Detection

Explain timing differences:
```
Human typing:
  Key press → 20-200ms delay → Key release
  Between keys: 50-300ms variation

SendInput:
  Key press → 0ms delay → Key release
  Between keys: 0ms (perfectly uniform)

Detection: Apps can detect zero-delay as bot input
```

Show how to make input look human.

### 8. The Gotchas

**"SendInput doesn't work in game!"**
> Game uses DirectInput or Raw Input, bypassing message queue

**"Can't send to admin window!"**
> UIPI blocks cross-privilege input for security

**"Modifier keys stuck!"**
> Sent "{Control down}" but script crashed before "{Control up}"

**"Special characters wrong!"**
> Sent "^c" (Ctrl+C) but keyboard layout doesn't have 'c' at that position

## Success Criteria

After this guide:
1. ✓ Understand complete input stack from hardware to app
2. ✓ Choose right Send method for each situation
3. ✓ Debug when input is blocked and find workarounds
4. ✓ Understand scan codes vs virtual keys
5. ✓ Send input that looks human to avoid detection

## Most Important

**Teach me Windows input architecture** so I understand not just how to send input, but WHY it sometimes doesn't work and HOW to diagnose and fix it.

I want systems-level understanding that lets me troubleshoot any input problem.
