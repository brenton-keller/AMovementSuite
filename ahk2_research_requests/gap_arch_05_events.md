# AHK2 Research Request: Event-Driven Architecture

## What I Need to Learn

My features can't communicate - when snapping mode changes, nothing else knows. When a window is grouped, the dimmer feature doesn't know to skip it. I need event-driven architecture to decouple features.

## My Current Understanding (Challenge This!)

I believe:
- AHK2 has no built-in event system
- I'd need to build pub/sub from scratch
- Events are slower than direct calls
- Observer pattern is overkill for simple scripts

**Am I underestimating what's possible?**

## The Core Problem

Current tight coupling:
```ahk
; Feature A directly modifies Feature B's state
global enableWindowSnapping := false

ToggleSnapping() {
    global enableWindowSnapping
    enableWindowSnapping := !enableWindowSnapping
    // No way to notify other features
    // They have to poll or check directly
}

; Feature B needs to know when snapping changes
; Currently: Global variable check everywhere (tight coupling)
```

What I want:
```ahk
// Feature A fires event
EventBus.Emit("snapping:changed", {enabled: false})

// Feature B subscribes
EventBus.On("snapping:changed", (data) => {
    // React to change
})

// Features don't know about each other (loose coupling)
```

## Specific Questions

1. **Observer Pattern**: How do I implement observer/observable in AHK2? Array of callbacks?

2. **Event Bus**: How do I create centralized event bus? How to namespace events?

3. **Unsubscribe**: How do I remove listeners? Prevent memory leaks?

4. **Event Order**: If multiple listeners, what order do they execute?

5. **Async Events**: Can events be async? Queue events for later processing?

6. **Performance**: What's the overhead vs direct function calls?

## How to Write This Guide

### 1. Start with "The Problem Events Solve"

Show tight coupling problem:
```ahk
// Before: Feature A knows about Feature B
class FeatureA {
    DoSomething() {
        // ... work ...
        FeatureB.Update()  // ❌ Tight coupling
        FeatureC.Refresh() // ❌ Tight coupling
    }
}

// After: Feature A knows nothing about listeners
class FeatureA {
    DoSomething() {
        // ... work ...
        EventBus.Emit("featureA:completed")  // ✓ Decoupled
    }
}
```

### 2. Progressive Implementation

**Level 1: Simple callback**
```ahk
class EventEmitter {
    __New() {
        this.listeners := []
    }

    On(callback) {
        this.listeners.Push(callback)
    }

    Emit(data := "") {
        for listener in this.listeners
            listener(data)
    }
}
```

**Level 2: Named events**
```ahk
class EventBus {
    __New() {
        this.events := Map()  // event name → array of callbacks
    }

    On(eventName, callback) {
        if !this.events.Has(eventName)
            this.events[eventName] := []
        this.events[eventName].Push(callback)
    }

    Emit(eventName, data := "") {
        if !this.events.Has(eventName)
            return

        for callback in this.events[eventName]
            callback(data)
    }
}
```

**Level 3: Unsubscribe support**
```ahk
class EventBus {
    On(eventName, callback) {
        // ... register ...
        ; Return unsubscribe function
        return () => this.Off(eventName, callback)
    }

    Off(eventName, callback) {
        // How to remove specific callback from array?
    }
}

// Usage:
unsubscribe := EventBus.On("change", handler)
// Later:
unsubscribe()  // Remove listener
```

**Level 4: Once, priority, async**
```ahk
class EventBus {
    On(eventName, callback, priority := 0) { }
    Once(eventName, callback) { }  // Auto-unsubscribe after first call
    EmitAsync(eventName, data) { }  // Queue for later
    Clear(eventName) { }  // Remove all listeners
}
```

### 3. Real-World Application

Transform my actual features:

**Current: Polling pattern**
```ahk
; WindowDimmer checks every 250ms
SetTimer CheckActiveWindow, 250

CheckActiveWindow() {
    ; Has active window changed?
    ; Manually check state
}
```

**After: Event-driven**
```ahk
; WindowDimmer subscribes to focus events
EventBus.On("window:focused", (hwnd) => {
    DimOtherWindows(hwnd)
})

; Window movement emits event
OnWindowMoved(hwnd) {
    EventBus.Emit("window:focused", hwnd)
}
```

### 4. Memory Management

The memory leak problem:
```ahk
; Creating closure in listener
obj := {name: "Test"}
EventBus.On("event", () => MsgBox(obj.name))

obj := ""  // Event bus still holds reference via closure!
// Memory leak - obj never freed
```

How to prevent this? Weak references? Cleanup patterns?

### 5. Error Handling in Events

What happens here:
```ahk
EventBus.Emit("event", data)
// Listener 1: succeeds
// Listener 2: throws error  ← What happens?
// Listener 3: never called?

// How to handle errors without breaking other listeners?
```

Show robust implementation.

### 6. Patterns Comparison

| Pattern | Coupling | Flexibility | Performance |
|---------|----------|-------------|-------------|
| Direct call | Tight | Low | Fastest |
| Callback param | Medium | Medium | Fast |
| Event bus | Loose | High | Slower |
| Global state | Tight | Low | Fast |

When to use each?

### 7. Complete Event System

Show production-ready implementation:
```ahk
class EventBus {
    static events := Map()

    static On(eventName, callback, options := "") {
        // Support options: priority, once, filter
    }

    static Off(eventName, callback := "") {
        // Remove specific or all listeners
    }

    static Emit(eventName, data := "") {
        // Safe execution (catch errors)
        // Respect priority
        // Log event in debug mode
    }

    static Clear() {
        // Remove all listeners (cleanup)
    }
}
```

### 8. The Gotchas

**"Events firing in wrong order!"**
> Didn't consider listener registration order matters without priority system

**"Memory leak from closures!"**
> Event listeners captured objects, preventing garbage collection

**"Error in one listener broke all"**
> Didn't wrap listener calls in try/catch - one failure stopped all

**"Can't unsubscribe!"**
> Used anonymous function as listener - no reference to remove it later

## Success Criteria

After this guide:
1. ✓ Implement event bus for feature communication
2. ✓ Decouple features (no direct dependencies)
3. ✓ Prevent memory leaks from event listeners
4. ✓ Handle errors in listeners gracefully
5. ✓ Understand when events are better than direct calls

## Most Important

Teach me to think in **events and reactions** instead of **direct calls and coupling**.

I want to design systems where features don't know about each other but still coordinate perfectly through events.
