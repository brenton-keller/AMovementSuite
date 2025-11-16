# How-To: Prevent Memory Leaks in AutoHotkey v2

## Problem

Circular references, event handlers, and timers in AutoHotkey v2 cause permanent memory leaks that aren't automatically cleaned up, leading to memory growth in long-running scripts.

## Quick Context

AutoHotkey v2 uses **reference counting**, not garbage collection. When you create an object, it maintains an internal counter tracking how many references point to it. When the counter reaches zero, the object is immediately freed. However, AHK2 has **no cycle detector**—circular references prevent the counter from ever reaching zero, causing permanent memory leaks until script exit.

This is fundamentally different from JavaScript or Python's garbage collection. Once a circular reference is created, those objects are leaked forever unless you manually break the cycle.

## Key Concepts

### Reference Counting Basics
- Creating a variable pointing to an object increments its reference count
- Reassigning or releasing a variable decrements the count
- At count zero, the object is immediately freed and `__Delete()` is called
- Circular references keep counts above zero permanently

### Three Common Leak Scenarios
1. **Circular object references** - Parent/child relationships
2. **Closure-captured variables** - Timers holding large data via closures
3. **Event handler cycles** - Objects subscribing to events that capture `this`

## Solution 1: Breaking Circular References

### The Problem Code

This creates a permanent leak:

```ahk
parent := {name: "Parent"}
child := {name: "Child"}
parent.child := child    ; parent refcount: 1, child refcount: 2
child.parent := parent   ; parent refcount: 2, child refcount: 2
parent := ""             ; parent refcount: 1 (still referenced by child)
child := ""              ; child refcount: 1 (still referenced by parent)
; Both objects leak - neither can be freed
```

Both objects now have refcount 1, but you have no variables pointing to either. They're permanently leaked and their `__Delete()` methods will never run.

### The Fix: Manual Cycle Breaking

Break the cycle before releasing references:

```ahk
parent := {name: "Parent"}
child := {name: "Child"}
parent.child := child
child.parent := parent

; Break the cycle FIRST
child.parent := ""       ; Now parent has only 1 reference
parent := ""             ; Parent refcount reaches 0, freed
child := ""              ; Child refcount reaches 0, freed
```

### Production Pattern: __Delete Destructor

Implement automatic cycle breaking in `__Delete()`:

```ahk
class ManagedNode {
    __New(name) {
        this.name := name
        this.children := []
    }

    AddChild(child) {
        this.children.Push(child)
        child.parent := this  ; Creates potential circular reference
    }

    __Delete() {
        ; Break all circular references before destruction
        for child in this.children {
            child.parent := ""
        }
        this.children := []
    }
}

; Usage - now safe:
parent := ManagedNode("Parent")
child := ManagedNode("Child")
parent.AddChild(child)
parent := ""  ; Calls __Delete, breaks cycles, frees both objects
child := ""
```

The `__Delete()` method automatically breaks all child references when the parent is freed, preventing leaks.

## Solution 2: Timer Cleanup

### The Problem Code

Timers with closures leak captured variables:

```ahk
CreateWindow() {
    largeData := Array(1000000)
    gui := Gui()
    SetTimer(() => ProcessData(largeData), 1000)
    return gui
}

win := CreateWindow()
win := ""  ; largeData is NOT freed - timer still holds it
```

The anonymous function `() => ProcessData(largeData)` captures `largeData` by reference. The timer stores that function object globally. Even though `CreateWindow()` returned and `largeData` went out of scope, the closure keeps it alive. **That million-element array leaks.**

### The Fix: Explicit Timer Deletion

Store the timer function for cleanup:

```ahk
CreateWindow() {
    largeData := Array(1000000)
    gui := Gui()
    timerFunc := () => ProcessData(largeData)
    SetTimer(timerFunc, 1000)
    gui.timerFunc := timerFunc  ; Store reference for cleanup
    return gui
}

win := CreateWindow()
; Later, to properly clean up:
SetTimer(win.timerFunc, "Delete")  ; "Delete", not "Off"!
win.timerFunc := ""
win := ""  ; Now everything can be freed
```

**Critical distinction**: `SetTimer(func, "Off")` disables the timer but retains the function reference. Only `SetTimer(func, "Delete")` or `SetTimer(func, 0)` actually removes it and breaks the reference chain.

### Production Pattern: Timer Management Class

```ahk
class ManagedWindow {
    __New() {
        this.largeData := Array(1000000)
        this.gui := Gui()
        this.timerFunc := () => this.ProcessData()
        SetTimer(this.timerFunc, 1000)
    }

    ProcessData() {
        ; Process this.largeData
        if (this.largeData.Length > 0) {
            ; Do work...
        }
    }

    Close() {
        SetTimer(this.timerFunc, "Delete")  ; Critical: "Delete" not "Off"
        this.timerFunc := ""
        this.gui.Destroy()
        this.largeData := []
    }

    __Delete() {
        this.Close()
    }
}

; Usage:
win := ManagedWindow()
; When done:
win.Close()  ; Explicitly clean up
win := ""    ; Now safe to release
```

### Generic Timer Manager

For classes with multiple timers:

```ahk
class Repeater {
    __New() {
        this.timers := []
    }

    AddTimer(callback, interval) {
        timerFunc := callback.Bind(this)
        SetTimer(timerFunc, interval)
        this.timers.Push(timerFunc)
        return timerFunc
    }

    RemoveTimer(timerFunc) {
        for i, func in this.timers {
            if (func = timerFunc) {
                SetTimer(func, "Delete")
                this.timers.RemoveAt(i)
                return
            }
        }
    }

    StopAll() {
        for timerFunc in this.timers {
            SetTimer(timerFunc, "Delete")
        }
        this.timers := []
    }

    __Delete() {
        this.StopAll()
    }
}

; Usage:
repeater := Repeater()
timer1 := repeater.AddTimer(() => MsgBox("Timer 1"), 1000)
timer2 := repeater.AddTimer(() => MsgBox("Timer 2"), 2000)

; Remove specific timer
repeater.RemoveTimer(timer1)

; Cleanup all
repeater := ""  ; Automatically stops all timers
```

## Solution 3: Event Handler Cleanup

### The Problem Code

Event handlers create hidden cycles:

```ahk
obj := {data: HugeArray()}
EventBus.On("event", () => Process(obj.data))
obj := ""  ; obj is NOT freed - event handler closure captures it
```

The closure `() => Process(obj.data)` captures `obj`. EventBus stores that closure in its listeners. Setting `obj := ""` releases your variable, but the EventBus still holds a reference. **The object and its huge array remain in memory until you explicitly unregister the handler.**

### Compound Problem: Self-Referential Event Handlers

```ahk
class Component {
    __New(eventBus) {
        this.bus := eventBus
        ; This creates a circular reference:
        ; this -> bus -> listeners -> closure -> this
        this.bus.On("update", () => this.HandleUpdate())
    }
    HandleUpdate() { }
}

comp := Component(bus)
comp := ""  ; Leaks! Component never freed due to circular reference
```

### The Fix: Return Subscription IDs

Implement an event system that returns handles for cleanup:

```ahk
class EventBus {
    __New() {
        this.events := Map()
        this.nextId := 1
    }

    Subscribe(eventName, callback) {
        if (!this.events.Has(eventName))
            this.events[eventName] := Map()

        id := this.nextId++
        this.events[eventName][id] := callback
        return id  ; Return subscription ID
    }

    Unsubscribe(eventName, subscriptionId) {
        if (!this.events.Has(eventName))
            return

        if (this.events[eventName].Has(subscriptionId))
            this.events[eventName].Delete(subscriptionId)

        ; Clean up empty event maps
        if (this.events[eventName].Count = 0)
            this.events.Delete(eventName)
    }

    Publish(eventName, args*) {
        if (!this.events.Has(eventName))
            return

        for id, callback in this.events[eventName] {
            callback(args*)
        }
    }

    __Delete() {
        this.events.Clear()
    }
}

; Correct usage:
bus := EventBus()
obj := {data: HugeArray()}

; Store subscription ID
subId := bus.Subscribe("event", () => Process(obj.data))

; Do work...

; Unsubscribe before releasing obj
bus.Unsubscribe("event", subId)
obj := ""  ; Now safe to release - no references remain
```

### Production Pattern: Subscription Manager

Automate subscription tracking:

```ahk
class Subscriber {
    __New(eventBus) {
        this.eventBus := eventBus
        this.subscriptions := []
    }

    Subscribe(eventName, callback) {
        id := this.eventBus.Subscribe(eventName, callback)
        this.subscriptions.Push({event: eventName, id: id})
        return id
    }

    Unsubscribe(eventName, subscriptionId) {
        this.eventBus.Unsubscribe(eventName, subscriptionId)

        ; Remove from tracking
        for i, sub in this.subscriptions {
            if (sub.event = eventName && sub.id = subscriptionId) {
                this.subscriptions.RemoveAt(i)
                break
            }
        }
    }

    UnsubscribeAll() {
        for sub in this.subscriptions {
            this.eventBus.Unsubscribe(sub.event, sub.id)
        }
        this.subscriptions := []
    }

    __Delete() {
        this.UnsubscribeAll()
    }
}

; Usage in components:
class DataComponent {
    __New(eventBus) {
        this.subscriber := Subscriber(eventBus)
        this.data := []

        ; Subscribe to events - automatically cleaned up
        this.subscriber.Subscribe("data_update", (newData) => this.OnDataUpdate(newData))
        this.subscriber.Subscribe("data_clear", (*) => this.OnDataClear())
    }

    OnDataUpdate(newData) {
        this.data.Push(newData)
    }

    OnDataClear() {
        this.data := []
    }

    __Delete() {
        ; subscriber.__Delete() automatically unsubscribes
        this.data := []
    }
}

; No memory leaks:
bus := EventBus()
comp := DataComponent(bus)
bus.Publish("data_update", "Item 1")
comp := ""  ; Automatically unsubscribes
```

## Complete Leak-Free Example

Full event system implementation combining all patterns:

```ahk
#Requires AutoHotkey v2.0

class EventBus {
    __New() {
        this.events := Map()
        this.nextId := 1
    }

    Subscribe(eventName, callback) {
        if (!this.events.Has(eventName))
            this.events[eventName] := Map()

        id := this.nextId++
        this.events[eventName][id] := callback
        return id
    }

    Unsubscribe(eventName, subscriptionId) {
        if (!this.events.Has(eventName))
            return

        if (this.events[eventName].Has(subscriptionId))
            this.events[eventName].Delete(subscriptionId)

        if (this.events[eventName].Count = 0)
            this.events.Delete(eventName)
    }

    Publish(eventName, args*) {
        if (!this.events.Has(eventName))
            return

        ; Copy callbacks to avoid modification during iteration
        callbacks := []
        for id, callback in this.events[eventName]
            callbacks.Push(callback)

        for callback in callbacks {
            try {
                callback(args*)
            } catch as err {
                OutputDebug("EventBus error in " eventName ": " err.Message)
            }
        }
    }

    Clear(eventName) {
        if (this.events.Has(eventName))
            this.events.Delete(eventName)
    }

    ClearAll() {
        this.events.Clear()
    }

    __Delete() {
        this.ClearAll()
    }
}

class Subscriber {
    __New(eventBus) {
        this.eventBus := eventBus
        this.subscriptions := []
    }

    Subscribe(eventName, callback) {
        id := this.eventBus.Subscribe(eventName, callback)
        this.subscriptions.Push({event: eventName, id: id})
        return id
    }

    Unsubscribe(eventName, subscriptionId) {
        this.eventBus.Unsubscribe(eventName, subscriptionId)

        for i, sub in this.subscriptions {
            if (sub.event = eventName && sub.id = subscriptionId) {
                this.subscriptions.RemoveAt(i)
                break
            }
        }
    }

    UnsubscribeAll() {
        for sub in this.subscriptions {
            this.eventBus.Unsubscribe(sub.event, sub.id)
        }
        this.subscriptions := []
    }

    __Delete() {
        this.UnsubscribeAll()
    }
}

class Application {
    __New() {
        this.bus := EventBus()
        this.subscriber := Subscriber(this.bus)
        this.gui := Gui(, "Leak-Free Application")

        ; Subscribe to events
        this.subscriber.Subscribe("process_data", (data) => this.ProcessData(data))

        ; Setup GUI
        this.gui.Add("Button", "w200", "Process Data").OnEvent("Click", (*) => this.OnProcess())
        this.gui.Add("Button", "w200", "Exit").OnEvent("Click", (*) => this.OnExit())

        ; Setup timer
        this.timerFunc := () => this.OnTimer()
        SetTimer(this.timerFunc, 5000)
    }

    ProcessData(data) {
        MsgBox("Processing: " data)
    }

    OnProcess(*) {
        this.bus.Publish("process_data", "User clicked process")
    }

    OnTimer() {
        this.bus.Publish("process_data", "Timer tick")
    }

    OnExit(*) {
        this.Close()
    }

    Close() {
        ; Stop timer
        if this.HasProp("timerFunc") {
            SetTimer(this.timerFunc, "Delete")
            this.timerFunc := ""
        }

        ; Unsubscribe from all events
        this.subscriber.UnsubscribeAll()

        ; Destroy GUI
        this.gui.Destroy()
    }

    __Delete() {
        this.Close()
    }

    Show() {
        this.gui.Show()
    }
}

; Run application with no memory leaks
app := Application()
app.Show()
return

; Cleanup on exit
^Escape:: {
    global app
    app := ""
    ExitApp
}
```

## Troubleshooting

### "My script's memory keeps growing"

**Symptom**: Task Manager shows increasing memory usage over time.

**Common causes**:
1. Timer with closure not being deleted
2. Event handlers not unsubscribed
3. Circular references in parent-child objects

**Diagnosis**:
```ahk
; Add to your script
myPID := DllCall("GetCurrentProcessId")
SetTimer(() => OutputDebug("Memory: " GetProcessMemoryInfo(myPID)), 60000)

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

### "SetTimer 'Off' doesn't free memory"

**Problem**: Using `SetTimer(func, "Off")` but memory still leaking.

**Solution**: Use `"Delete"` instead:
```ahk
; Wrong:
SetTimer(myFunc, "Off")  ; Disables but doesn't free

; Correct:
SetTimer(myFunc, "Delete")  ; Actually removes the timer
```

### "__Delete never gets called"

**Problem**: Your `__Delete()` method isn't executing.

**Cause**: Circular reference preventing refcount from reaching zero.

**Diagnosis**:
```ahk
GetRefCount(&obj) {
    ObjAddRef(ptr := ObjPtr(obj))
    return ObjRelease(ptr)
}

myObj := MyClass()
MsgBox GetRefCount(&myObj)  ; Should be 1
; If > 1, you have extra references somewhere
```

**Fix**: Review object for:
- Properties pointing back to parent
- Event handlers capturing `this`
- Timers capturing `this`
- Bound methods stored elsewhere

### "GUI extending class leaks"

**Problem**: `class MyWindow extends Gui` still leaks.

**Common mistake**: Not calling parent `__Delete()`:
```ahk
; Wrong:
class MyWindow extends Gui {
    __Delete() {
        ; Missing super.__Delete()
        this.CleanupStuff()
    }
}

; Correct:
class MyWindow extends Gui {
    __Delete() {
        this.CleanupStuff()
        super.__Delete()  ; Call parent destructor
    }
}
```

### "Hotkey with closure leaks"

**Problem**: Registered hotkey keeps objects alive.

**Solution**: Store function and cleanup:
```ahk
class HotkeyManager {
    __New() {
        this.hotkeys := Map()
    }

    Register(key, callback) {
        Hotkey(key, callback)
        this.hotkeys[key] := callback
    }

    Unregister(key) {
        if this.hotkeys.Has(key) {
            Hotkey(key, "Off")
            this.hotkeys.Delete(key)
        }
    }

    __Delete() {
        for key in this.hotkeys {
            Hotkey(key, "Off")
        }
        this.hotkeys.Clear()
    }
}
```

## Related Documentation

- [Memory Architecture Explanation](../explanation/memory-architecture.md) - Deep dive into reference counting vs garbage collection
- [Object Lifecycle Reference](../reference/object-lifecycle.md) - Complete `__New()` and `__Delete()` behavior
- [Closure Behavior Explanation](../explanation/closures.md) - How variable capture works
- [Debug Memory Leaks](debug-memory-leaks.md) - Detect and diagnose leaks in your scripts
