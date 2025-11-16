# Memory Management Reference

Quick lookup for AutoHotkey v2 memory management behavior, functions, and patterns.

---

## Reference Counting Rules

### What Increments Reference Count

| Action | Count Change | Example |
|--------|--------------|---------|
| Variable assignment | +1 | `obj2 := obj` |
| Array/Map insertion | +1 | `arr.Push(obj)` |
| Property assignment | +1 | `parent.child := obj` |
| Function parameter | +1 (temporary) | `MyFunc(obj)` |
| Closure capture | +1 | `() => Process(obj)` |
| `ObjFromPtr()` call | +1 | `ObjFromPtr(ptr)` |
| `ObjAddRef()` call | +1 | `ObjAddRef(ptr)` |
| `Gui.Show()` call | +1 (self-reference) | `gui.Show()` |

### What Decrements Reference Count

| Action | Count Change | Example |
|--------|--------------|---------|
| Variable reassignment | -1 | `obj := ""` |
| Variable goes out of scope | -1 | Function returns |
| Array/Map element removed | -1 | `arr.RemoveAt(1)` |
| Property overwritten | -1 | `parent.child := ""` |
| Function return | -1 | Parameters released |
| Closure destroyed | -1 | Closure object freed |
| `ObjRelease()` call | -1 | `ObjRelease(ptr)` |
| GUI hidden/closed | -1 | GUI self-reference released |

### What Does NOT Affect Reference Count

| Action | Effect | Notes |
|--------|--------|-------|
| `ObjPtr()` call | Returns address only | Creates unmanaged pointer |
| Passing to DllCall | No change | Use `"Ptr"` type |
| Reading property | No change | Only assignment changes count |
| Accessing array element | No change | Only modification changes count |

---

## Memory Management Functions

### ObjPtr

```ahk
ObjPtr(Object) → Integer
```

**Purpose**: Get memory address without affecting reference count

**Parameters**:
- `Object`: Any AHK object

**Returns**: Integer memory address

**Use Cases**:
- Breaking circular references (weak references)
- DllCall object parameter passing
- Object identity comparison

**Warnings**:
- Returned pointer is unmanaged
- Object may be freed while pointer exists
- Do NOT store long-term without reference

**Example**:
```ahk
obj := {}
ptr := ObjPtr(obj)  ; Get address, refcount still 1
obj := ""           ; Object freed, ptr is now dangling
```

---

### ObjFromPtr

```ahk
ObjFromPtr(Pointer) → Object
```

**Purpose**: Convert address to managed object reference

**Parameters**:
- `Pointer`: Integer memory address from `ObjPtr()`

**Returns**: Object reference (increments count)

**Use Cases**:
- DllCall return value conversion
- Weak reference retrieval
- COM object wrapping

**Warnings**:
- INCREMENTS reference count
- Using freed pointer crashes script
- Always pair with `ObjRelease()` for manual management

**Example**:
```ahk
obj := {value: 42}
ptr := ObjPtr(obj)
obj2 := ObjFromPtr(ptr)  ; refcount now 2
obj := ""                ; refcount now 1
obj2 := ""               ; refcount 0, freed
```

---

### ObjAddRef

```ahk
ObjAddRef(Pointer) → Integer
```

**Purpose**: Manually increment reference count

**Parameters**:
- `Pointer`: Integer memory address from `ObjPtr()`

**Returns**: New reference count

**Use Cases**:
- Debugging reference count
- COM interop
- Advanced lifetime control

**Warnings**:
- **DEBUG ONLY** - Do not use in production
- Must pair with `ObjRelease()`
- Misuse causes crashes

**Example** (debugging only):
```ahk
GetRefCount(&obj) {
    ObjAddRef(ptr := ObjPtr(obj))
    return ObjRelease(ptr)
}

myObj := {}
MsgBox GetRefCount(&myObj)  ; Shows: 1
```

---

### ObjRelease

```ahk
ObjRelease(Pointer) → Integer
```

**Purpose**: Manually decrement reference count

**Parameters**:
- `Pointer`: Integer memory address from `ObjPtr()`

**Returns**: New reference count

**Use Cases**:
- Debugging reference count
- COM interop cleanup
- Pairing with `ObjAddRef()`

**Warnings**:
- **DEBUG ONLY** - Do not use in production
- Reducing to 0 with active refs causes crashes
- Must balance with `ObjAddRef()`

**Example** (debugging only):
```ahk
obj := {}
ptr := ObjPtr(obj)
ObjAddRef(ptr)     ; refcount 2
ObjRelease(ptr)    ; refcount 1
obj := ""          ; refcount 0, freed
```

---

## GUI Object Lifetime

### Gui.Show() Reference Behavior

| State | Reference Count | Behavior |
|-------|----------------|----------|
| Created, not shown | Normal | Can be freed if no references |
| After `Gui.Show()` | +1 (self-reference) | GUI "holds itself" open |
| Minimized/Hidden | Still +1 | Self-reference persists |
| After `Gui.Hide()` | -1 (released) | Can be freed if no other refs |
| After `Gui.Destroy()` | -1 (released) | Window destroyed immediately |
| User closes window | -1 (released) | GUI destroyed unless prevented |

### GUI Memory Patterns

```ahk
; Pattern 1: Function-scoped GUI (WRONG)
CreateGUI() {
    gui := Gui()
    gui.Add("Text", , "Hello")
    return gui  ; Returns with refcount 1
}
g := CreateGUI()
g := ""  ; Freed immediately - window vanishes

; Pattern 2: Shown GUI (CORRECT)
CreateGUI() {
    gui := Gui()
    gui.Add("Text", , "Hello")
    gui.Show()  ; Adds self-reference, refcount 2
    return gui
}
g := CreateGUI()
g := ""  ; GUI stays visible (refcount 1 from self-reference)

; Pattern 3: Explicit cleanup (BEST)
CreateGUI() {
    gui := Gui()
    gui.Add("Text", , "Hello")
    gui.OnEvent("Close", (*) => gui.Destroy())
    gui.Show()
    return gui
}
```

---

## Closure Capture Rules

### What Gets Captured

| Variable Type | Capture Behavior | Lifetime |
|--------------|------------------|----------|
| Local variable | By reference | Until closure freed |
| Parameter | By reference | Until closure freed |
| Global variable | By reference (name lookup) | Script lifetime |
| Static variable | By reference | Function lifetime |
| `this` in method | Captures entire object | Until closure freed |
| Loop counter | Shared reference | Last value |

### Closure Capture Scenarios

```ahk
; Scenario: Timer captures local variable
CreateWindow() {
    largeData := Array(1000000)
    SetTimer(() => Process(largeData), 1000)
    ; largeData captured, won't be freed until timer deleted
}

; Scenario: Event handler captures 'this'
class Component {
    __New() {
        this.data := HugeArray()
        btn.OnEvent("Click", () => this.Process())
        ; Entire 'this' captured
    }
}

; Scenario: Loop closure (shared variable)
Loop 5 {
    i := A_Index
    SetTimer(() => MsgBox(i), 1000 * A_Index)
    ; All timers share same 'i' variable (value = 5)
}
```

### Closure Memory Impact

| Captured Value | Memory Impact | Duration |
|----------------|---------------|----------|
| Small primitive | Minimal | Closure lifetime |
| Large string/buffer | Significant | Closure lifetime |
| Object reference | Prevents object freeing | Closure lifetime |
| `this` reference | Prevents class instance freeing | Closure lifetime |
| Unused variable in scope | May still be captured | Closure lifetime |

---

## Leak Pattern Catalog

### Quick Reference Table

| Pattern | Leaks? | Why | Fix |
|---------|--------|-----|-----|
| Circular reference | YES | No cycle detector | Manual break in `__Delete()` |
| Timer with closure | YES | Timer holds func indefinitely | `SetTimer(func, "Delete")` |
| Event handler closure | YES | Handler array keeps reference | Explicit unsubscribe |
| GUI event handler | MAYBE | Depends on circular reference | Use `extends Gui` pattern |
| Hotkey with closure | YES | Hotkey keeps func until reassigned | `Hotkey(key, "Off")` |
| Bound method | MAYBE | If creates circular ref | Store and release explicitly |
| `ObjPtr()` alone | NO | Doesn't affect refcount | Safe but pointer may dangle |
| `ObjFromPtr()` unpaired | NO | Increments count properly | But creates new reference |
| Loop closure capture | NO | Not a leak, logic bug | Use separate vars per iteration |
| Large temporary data | NO | Freed when scope exits | But can optimize with `var := ""` |

### Pattern Details

#### Parent-Child Circular Reference
```ahk
; LEAKS
parent := {name: "Parent"}
child := {name: "Child"}
parent.child := child  ; parent refcount: 1, child refcount: 2
child.parent := parent ; parent refcount: 2, child refcount: 2
parent := ""           ; parent refcount: 1 (child.parent still holds it)
child := ""            ; child refcount: 1 (parent.child still holds it)
; Both leak forever

; FIX
child.parent := ""     ; Break cycle first
parent := ""           ; parent freed (refcount 0)
child := ""            ; child freed (refcount 0)
```

#### Timer Closure Leak
```ahk
; LEAKS
CreateWindow() {
    largeData := Array(1000000)
    SetTimer(() => Process(largeData), 1000)
    ; largeData captured, never freed
}

; FIX
CreateWindow() {
    largeData := Array(1000000)
    timerFunc := () => Process(largeData)
    SetTimer(timerFunc, 1000)
    return timerFunc  ; Return for cleanup
}
tf := CreateWindow()
SetTimer(tf, "Delete")  ; Deletes timer, frees closure
tf := ""
```

#### Event Handler Circular Reference
```ahk
; LEAKS
class Component {
    __New() {
        this.gui := Gui()
        this.gui.OnEvent("Close", (*) => this.OnClose())
        ; this -> gui -> event handlers -> closure -> this
    }
}

; FIX
class Component extends Gui {
    __New() {
        super.__New(, "Window", this)  ; Pass this as event sink
    }
    OnClose(gui) {  ; Auto-called, no closure needed
        gui.Destroy()
    }
}
```

#### Subscription Leak
```ahk
; LEAKS
obj := {data: HugeArray()}
bus.On("event", () => Process(obj.data))
obj := ""  ; obj captured by closure, not freed

; FIX
obj := {data: HugeArray()}
subId := bus.Subscribe("event", () => Process(obj.data))
; ... later ...
bus.Unsubscribe("event", subId)  ; Remove handler
obj := ""  ; Now freed
```

---

## SetTimer Memory Behavior

### Command Comparison

| Command | Effect | Timer State | Function Reference | Memory |
|---------|--------|-------------|-------------------|--------|
| `SetTimer(func, 1000)` | Create/update | Active, 1000ms | Stored | Held |
| `SetTimer(func, "Off")` | Disable | Inactive | **Still stored** | **Still held** |
| `SetTimer(func, "Delete")` | Remove | Deleted | Released | Freed |
| `SetTimer(func, 0)` | Remove | Deleted | Released | Freed |

### Critical Distinction

```ahk
; WRONG - Memory still held
largeData := Array(1000000)
timerFunc := () => Process(largeData)
SetTimer(timerFunc, 1000)
SetTimer(timerFunc, "Off")  ; Timer disabled but func NOT released
timerFunc := ""             ; largeData STILL HELD by timer system

; CORRECT - Memory freed
largeData := Array(1000000)
timerFunc := () => Process(largeData)
SetTimer(timerFunc, 1000)
SetTimer(timerFunc, "Delete")  ; Timer deleted, func released
timerFunc := ""                ; largeData now freed
```

### Timer Cleanup Patterns

| Pattern | When to Use |
|---------|-------------|
| `"Off"` | Temporarily pause, will restart later |
| `"Delete"` | Permanent removal, cleanup memory |
| `0` | Permanent removal (same as "Delete") |

---

## Common Pitfalls

### Wrong vs Right Comparisons

#### Pitfall 1: Timer "Off" vs "Delete"

```ahk
; WRONG
class Worker {
    Start() {
        this.timer := () => this.Work()
        SetTimer(this.timer, 100)
    }
    Stop() {
        SetTimer(this.timer, "Off")  ; WRONG - still holds reference
    }
}

; RIGHT
class Worker {
    Start() {
        this.timer := () => this.Work()
        SetTimer(this.timer, 100)
    }
    Stop() {
        SetTimer(this.timer, "Delete")  ; RIGHT - releases reference
        this.timer := ""
    }
}
```

#### Pitfall 2: Circular Reference in __New

```ahk
; WRONG
class Component {
    __New(parent) {
        this.parent := parent
        parent.children.Push(this)
        ; Circular: this -> parent -> children -> this
    }
}

; RIGHT
class Component {
    __New(parent) {
        this.parent := parent
        parent.children.Push(this)
    }
    __Delete() {
        this.parent := ""  ; Break cycle
    }
}
; Or use ObjPtr() for weak reference
```

#### Pitfall 3: GUI Event Handler Closure

```ahk
; WRONG
CreateWindow() {
    gui := Gui()
    gui.OnEvent("Close", () => HandleClose(gui))  ; Captures gui
    gui.Show()
    return gui  ; Circular: returned gui -> event -> closure -> captured gui
}

; RIGHT (Method 1: Extends Gui)
class MyWindow extends Gui {
    __New() {
        super.__New(, "Window", this)
        this.Show()
    }
    OnClose(g) {  ; Auto-called
        g.Destroy()
    }
}

; RIGHT (Method 2: No Capture)
CreateWindow() {
    gui := Gui()
    gui.OnEvent("Close", (*) => A_EventInfo := "")  ; No capture needed
    gui.Show()
    return gui
}
```

#### Pitfall 4: Loop Closure Shared Variable

```ahk
; WRONG (Logic bug, not leak)
buttons := []
Loop 5 {
    i := A_Index
    btn := Gui.Add("Button", , "Button " i)
    btn.OnEvent("Click", () => MsgBox(i))  ; All closures share i
    buttons.Push(btn)
}
; All buttons show "5" when clicked

; RIGHT
buttons := []
Loop 5 {
    btn := Gui.Add("Button", , "Button " A_Index)
    btn.OnEvent("Click", (ctrl, *) => MsgBox(ctrl.Text))  ; Use parameter
    buttons.Push(btn)
}
```

#### Pitfall 5: Manual RefCount Manipulation

```ahk
; WRONG - Will crash
obj := {}
ptr := ObjPtr(obj)
ObjRelease(ptr)  ; Drops refcount to 0
; obj still exists as variable
obj := ""  ; CRASH - double free

; RIGHT - Debug only
GetRefCount(&obj) {
    ObjAddRef(ptr := ObjPtr(obj))
    return ObjRelease(ptr)  ; Balanced immediately
}
count := GetRefCount(&myObj)  ; Safe inspection
```

#### Pitfall 6: Assuming Garbage Collection

```ahk
; WRONG ASSUMPTION
CreateData() {
    parent := {children: []}
    Loop 100 {
        child := {parent: parent}
        parent.children.Push(child)
    }
    return parent
    ; "GC will clean up circular refs" - NO IT WON'T
}

; RIGHT - Manual cleanup
class ManagedParent {
    __New() {
        this.children := []
    }
    AddChild(child) {
        this.children.Push(child)
        child.parent := this
    }
    __Delete() {
        for child in this.children
            child.parent := ""  ; Break cycles
        this.children := []
    }
}
```

---

## Decision Matrices

### When to Use ObjPtr vs Managed Reference

| Scenario | Use ObjPtr | Use Managed Reference |
|----------|------------|----------------------|
| Prevent circular reference | YES | NO |
| DllCall parameter | YES | NO |
| Long-term storage | NO | YES |
| Parent-child weak link | YES | NO |
| Ensure object stays alive | NO | YES |
| Object identity check | YES | Either |
| COM interop | Context-dependent | Preferred |
| Temporary address | YES | NO |

### When to Break Circular References

| Pattern | Break When | Where to Break |
|---------|-----------|----------------|
| Parent-child | Parent deleted | Parent's `__Delete()` |
| Event subscription | Unsubscribe | Subscriber cleanup |
| GUI event handler | GUI destroyed | GUI's `__Delete()` |
| Timer callback | Timer deleted | Before `SetTimer(*, "Delete")` |
| Hotkey callback | Hotkey removed | Before `Hotkey(*, "Off")` |
| Bound method | Object destroyed | Object's `__Delete()` |

### Memory Management Strategy Selection

| Use Case | Strategy | Implementation |
|----------|----------|----------------|
| Simple object | Default refcount | Let AHK manage |
| Parent-child tree | Manual cleanup | `__Delete()` breaks cycles |
| Event system | Subscription IDs | Explicit unsubscribe |
| Timer-based | Store function refs | `SetTimer(*, "Delete")` |
| GUI with events | Extends Gui | Pass `this` to super |
| Weak reference | ObjPtr pattern | Store ptr, validate before use |
| Long-running script | Lifecycle management | Explicit init/cleanup methods |

### Debugging Approach Selection

| Symptom | Diagnosis Method | Tool |
|---------|-----------------|------|
| Growing memory | Stress test loop | Process memory monitor |
| Objects not freed | Check refcount | `ObjAddRef/Release` debug |
| `__Delete()` not called | Circular reference | Code review |
| Dangling pointer crash | ObjPtr without reference | Add try/catch |
| Timer won't stop | Check timer list | OutputDebug timer state |
| Event handler persists | Check subscription | Event bus inspection |

---

## Memory Monitoring Code

### Process Memory Tracking

```ahk
GetProcessMemoryInfo(PID) {
    size := 440
    pmcex := Buffer(size, 0)
    hProcess := DllCall("OpenProcess", "UInt", 0x400|0x0010, "Int", 0, "Ptr", PID, "Ptr")
    if (hProcess) {
        if (DllCall("psapi.dll\GetProcessMemoryInfo", "Ptr", hProcess, "Ptr", pmcex, "UInt", size))
            ret := NumGet(pmcex, (A_PtrSize=8 ? 16 : 12), "UInt")/1024 . " KB"
        DllCall("CloseHandle", "Ptr", hProcess)
        return ret ?? "N/A"
    }
    return "N/A"
}
```

### Leak Detection Pattern

```ahk
; Stress test for memory leaks
myPID := DllCall("GetCurrentProcessId")
startMem := GetProcessMemoryInfo(myPID)

Loop 10000 {
    SuspiciousFunction()  ; Function under test

    if (Mod(A_Index, 1000) = 0) {
        currentMem := GetProcessMemoryInfo(myPID)
        OutputDebug("Iteration " A_Index ": " currentMem)
    }
}

endMem := GetProcessMemoryInfo(myPID)
MsgBox("Memory increase: " (endMem - startMem))
```

### Reference Count Inspector

```ahk
; DEBUG ONLY - Do not use in production
GetRefCount(&obj) {
    ObjAddRef(ptr := ObjPtr(obj))
    return ObjRelease(ptr)
}

; Usage
myObj := {}
MsgBox "Initial: " GetRefCount(&myObj)  ; 1
another := myObj
MsgBox "After assignment: " GetRefCount(&myObj)  ; 2
another := ""
MsgBox "After release: " GetRefCount(&myObj)  ; 1
```

---

## Core Memory Facts

### What AHK2 Has

- **Reference counting**: Deterministic, immediate
- **Automatic freeing**: When count reaches 0
- **`__Delete()` callback**: Called on object destruction
- **Special GUI behavior**: `Show()` adds self-reference

### What AHK2 Does NOT Have

- **Garbage collection**: No mark-and-sweep
- **Cycle detection**: Circular refs leak forever
- **Weak references**: No built-in weak ref type
- **Manual GC trigger**: No way to force collection
- **Finalizer queue**: `__Delete()` runs immediately
- **Memory compaction**: No heap reorganization

### Key Principles

1. **Every circular reference leaks** - No exceptions
2. **Closures extend lifetime** - All captured variables
3. **Timers hold references** - Until "Delete" or 0
4. **Event handlers hold references** - Until unsubscribed
5. **ObjPtr() is unmanaged** - Doesn't prevent freeing
6. **ObjFromPtr() increments** - Creates managed reference
7. **Refcount manipulation crashes** - Debug only
8. **`__Delete()` must clean up** - Break cycles, unsubscribe

### Memory Leak Indicators

| Indicator | Likely Cause |
|-----------|--------------|
| Linear memory growth | Leak in loop |
| `__Delete()` never called | Circular reference |
| Objects "stuck" in memory | Missing cleanup |
| Timer won't stop | Used "Off" instead of "Delete" |
| Event handler fires after cleanup | Didn't unsubscribe |
| Crash on exit | RefCount manipulation |
| GUI visible but unreachable | Lost reference but Show() holds it |

---

## Quick Reference Summary

### Function Quick Lookup

| Function | Params | Returns | Affects Count | Primary Use |
|----------|--------|---------|---------------|-------------|
| `ObjPtr(obj)` | Object | Integer | NO | Get address |
| `ObjFromPtr(ptr)` | Integer | Object | YES (+1) | Address to object |
| `ObjAddRef(ptr)` | Integer | Integer | YES (+1) | Debug refcount |
| `ObjRelease(ptr)` | Integer | Integer | YES (-1) | Debug refcount |

### Cleanup Quick Reference

| What to Clean | How | When |
|---------------|-----|------|
| Timer | `SetTimer(func, "Delete")` | Before releasing func |
| Hotkey | `Hotkey(key, "Off")` | Before releasing callback |
| Event subscription | `bus.Unsubscribe(event, id)` | Before releasing object |
| GUI event | `gui.Destroy()` | On close |
| Circular reference | `obj.prop := ""` | In `__Delete()` |
| Closure capture | Release closure | Delete timer/handler |

### Leak Prevention Checklist

- [ ] All circular references broken in `__Delete()`
- [ ] All timers use `"Delete"` not `"Off"` for cleanup
- [ ] All event subscriptions return cleanup IDs
- [ ] GUI classes extend Gui instead of containing it
- [ ] Closures capture minimal scope
- [ ] Large data explicitly released with `var := ""`
- [ ] No manual `ObjAddRef/Release` in production code
- [ ] Stress tests show stable memory usage
