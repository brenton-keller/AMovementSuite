# AHK2 Research Request: Memory Management & Garbage Collection

## What I Need

I suspect I have memory leaks but can't find them. Objects with circular references, closures capturing large objects, timers holding references - I need to understand AHK2's memory model.

## Current Understanding (Challenge!)

- AHK2 has automatic garbage collection
- Objects are freed when no references exist
- Circular references are automatically detected
- __Delete is called when object is freed
- Memory leaks are rare in AHK2

**What am I wrong about?**

## The Leaks

**Leak 1: Circular reference**
```ahk
parent := {name: "Parent"}
child := {name: "Child"}
parent.child := child
child.parent := parent

parent := ""
child := ""
// Are they freed? Or leaked?
```

**Leak 2: Timer holding reference**
```ahk
CreateWindow() {
    largeData := Array(1000000)
    gui := Gui()
    SetTimer () => ProcessData(largeData), 1000
    return gui
}

win := CreateWindow()
win := ""  // Is largeData freed?
```

**Leak 3: Event listener**
```ahk
obj := {data: HugeArray()}
EventBus.On("event", () => Process(obj.data))
obj := ""  // Is obj freed?
```

## Questions

1. How does reference counting work? When is __Delete called?
2. Are circular references auto-detected or do they leak?
3. How do closures affect object lifetime?
4. How do I detect memory leaks?
5. Can I force garbage collection?

## Guide Should Include

- Reference counting mechanism explained
- Circular reference detection (or lack thereof)
- How closures extend variable lifetime
- Detecting and fixing memory leaks
- Complete example: Leak-free event system

## Success

Identify and fix all memory leaks in my codebase. Understand object lifetime management deeply.
