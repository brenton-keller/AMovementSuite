# How-To: Debug Memory Leaks in AutoHotkey v2

## Problem

You need to detect whether your long-running AutoHotkey v2 script has memory leaks and identify what's causing them, but AHK2 provides no built-in memory profiling tools.

## Quick Context

Memory leaks in long-running scripts manifest as gradually increasing process memory that never decreases. Unlike JavaScript or Python, AHK2 uses **reference counting** without garbage collection—there's no way to force cleanup or inspect the heap. This means:

- No `System.gc()` equivalent exists
- Objects are freed immediately when refcount hits zero (deterministic)
- Leaked objects stay leaked until script exit
- Detection requires external monitoring and stress testing

## Key Concepts

### What Memory Leaks Look Like

In a leak-free script, memory usage:
1. Grows during initialization
2. Fluctuates during operation
3. Returns to baseline when operations complete
4. Plateaus at a stable level

With a memory leak:
1. Memory grows linearly with operations
2. Never returns to baseline
3. Continues growing until system resources are exhausted

### Detection Strategy

1. **Baseline**: Measure memory before suspicious operations
2. **Stress Test**: Repeat operations many times (1000-10000 iterations)
3. **Monitor**: Track memory at intervals
4. **Compare**: Check if memory returns to baseline

### Inspection Tools

- **GetProcessMemoryInfo**: Windows API for process memory stats
- **ObjAddRef/ObjRelease**: Reference count inspection (debugging only)
- **Task Manager**: Quick visual check
- **Process Explorer**: Detailed memory breakdown

## Solution: Memory Monitoring System

### Basic Memory Info Function

This uses Windows API to get accurate process memory:

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

; Basic usage:
myPID := DllCall("GetCurrentProcessId")
MsgBox("Current memory: " GetProcessMemoryInfo(myPID))
```

### Enhanced Version with Numeric Return

For calculations, return numbers instead of strings:

```ahk
GetProcessMemoryKB(PID) {
    size := 440
    pmcex := Buffer(size, 0)
    hProcess := DllCall("OpenProcess", "UInt", 0x400|0x0010, "Int", 0, "Ptr", PID, "Ptr")
    if (hProcess) {
        if (DllCall("psapi.dll\GetProcessMemoryInfo", "Ptr", hProcess, "Ptr", pmcex, "UInt", size)) {
            memKB := NumGet(pmcex, (A_PtrSize=8 ? 16 : 12), "UInt") / 1024
            DllCall("CloseHandle", "Ptr", hProcess)
            return memKB
        }
        DllCall("CloseHandle", "Ptr", hProcess)
    }
    return 0
}

; Usage:
myPID := DllCall("GetCurrentProcessId")
startMem := GetProcessMemoryKB(myPID)
; ... do work ...
endMem := GetProcessMemoryKB(myPID)
MsgBox("Memory increase: " (endMem - startMem) " KB")
```

## Stress Test Pattern

### Basic Leak Detection Loop

Test if a function leaks by running it thousands of times:

```ahk
#Requires AutoHotkey v2.0

; Function to test
SuspiciousFunction() {
    gui := Gui()
    SetTimer(() => MsgBox("Timer"), 1000)
    ; Potential leak: timer not cleaned up!
    return gui
}

; Stress test
myPID := DllCall("GetCurrentProcessId")
startMem := GetProcessMemoryKB(myPID)

MsgBox("Starting stress test...`nInitial memory: " startMem " KB")

Loop 1000 {
    SuspiciousFunction()

    if (Mod(A_Index, 100) = 0) {
        currentMem := GetProcessMemoryKB(myPID)
        OutputDebug("Iteration " A_Index ": " currentMem " KB")
    }
}

endMem := GetProcessMemoryKB(myPID)
increase := endMem - startMem
MsgBox("Stress test complete`n`nStart: " startMem " KB`nEnd: " endMem " KB`nIncrease: " increase " KB")

GetProcessMemoryKB(PID) {
    size := 440
    pmcex := Buffer(size, 0)
    hProcess := DllCall("OpenProcess", "UInt", 0x400|0x0010, "Int", 0, "Ptr", PID, "Ptr")
    if (hProcess) {
        if (DllCall("psapi.dll\GetProcessMemoryInfo", "Ptr", hProcess, "Ptr", pmcex, "UInt", size)) {
            memKB := NumGet(pmcex, (A_PtrSize=8 ? 16 : 12), "UInt") / 1024
            DllCall("CloseHandle", "Ptr", hProcess)
            return memKB
        }
        DllCall("CloseHandle", "Ptr", hProcess)
    }
    return 0
}
```

If memory increases linearly (e.g., 50KB, 100KB, 150KB at iterations 100, 200, 300), you have a leak.

### Advanced Memory Tracker

Create a reusable memory profiler:

```ahk
class MemoryProfiler {
    __New() {
        this.pid := DllCall("GetCurrentProcessId")
        this.samples := []
        this.baseMem := 0
    }

    Start() {
        this.baseMem := this.GetMemory()
        this.samples := []
        this.samples.Push({iteration: 0, memory: this.baseMem, delta: 0})
        return this.baseMem
    }

    Sample(iteration) {
        currentMem := this.GetMemory()
        delta := currentMem - this.baseMem
        this.samples.Push({iteration: iteration, memory: currentMem, delta: delta})
        return delta
    }

    GetMemory() {
        size := 440
        pmcex := Buffer(size, 0)
        hProcess := DllCall("OpenProcess", "UInt", 0x400|0x0010, "Int", 0, "Ptr", this.pid, "Ptr")
        if (hProcess) {
            if (DllCall("psapi.dll\GetProcessMemoryInfo", "Ptr", hProcess, "Ptr", pmcex, "UInt", size)) {
                memKB := NumGet(pmcex, (A_PtrSize=8 ? 16 : 12), "UInt") / 1024
                DllCall("CloseHandle", "Ptr", hProcess)
                return memKB
            }
            DllCall("CloseHandle", "Ptr", hProcess)
        }
        return 0
    }

    Report() {
        if (this.samples.Length = 0)
            return "No samples recorded"

        report := "Memory Profile Report`n"
        report .= "==================`n`n"
        report .= "Base memory: " this.baseMem " KB`n"
        report .= "Samples: " this.samples.Length "`n`n"

        ; Show every 10th sample or last 10
        step := Max(1, Floor(this.samples.Length / 10))
        for i, sample in this.samples {
            if (Mod(i-1, step) = 0 || i > this.samples.Length - 10) {
                report .= "Iteration " sample.iteration ": "
                report .= sample.memory " KB (+" sample.delta " KB)`n"
            }
        }

        ; Analysis
        lastSample := this.samples[this.samples.Length]
        totalIncrease := lastSample.delta

        report .= "`n"
        report .= "Total increase: " totalIncrease " KB`n"

        if (totalIncrease > 1000) {
            report .= "VERDICT: Likely memory leak detected!`n"
        } else if (totalIncrease > 100) {
            report .= "VERDICT: Possible leak, investigate further`n"
        } else {
            report .= "VERDICT: Memory usage appears stable`n"
        }

        return report
    }

    __Delete() {
        this.samples := []
    }
}

; Usage example:
TestForLeaks() {
    profiler := MemoryProfiler()
    profiler.Start()

    Loop 5000 {
        ; Your suspicious code here
        obj := {data: Array(100)}
        gui := Gui()
        SetTimer(() => Process(obj), 1000)
        ; Leak: timer not deleted, captures obj

        if (Mod(A_Index, 500) = 0) {
            profiler.Sample(A_Index)
        }
    }

    MsgBox(profiler.Report())
}
```

## Reference Count Inspection

### Debug-Only Refcount Checker

**WARNING**: Only use for debugging, never in production!

```ahk
GetRefCount(&obj) {
    ObjAddRef(ptr := ObjPtr(obj))
    return ObjRelease(ptr)
}

; Example: Detecting unexpected references
myObj := {name: "Test"}
MsgBox "Refcount: " GetRefCount(&myObj)  ; Should be 1

another := myObj
MsgBox "Refcount: " GetRefCount(&myObj)  ; Should be 2

another := ""
MsgBox "Refcount: " GetRefCount(&myObj)  ; Back to 1
```

### Circular Reference Detection

Use refcount inspection to detect cycles:

```ahk
#Requires AutoHotkey v2.0

; Test for circular references
TestCircularRef() {
    parent := {name: "Parent"}
    child := {name: "Child"}

    MsgBox "Initial refcounts:`nParent: " GetRefCount(&parent) "`nChild: " GetRefCount(&child)
    ; Both should be 1

    parent.child := child
    child.parent := parent

    MsgBox "After creating cycle:`nParent: " GetRefCount(&parent) "`nChild: " GetRefCount(&child)
    ; Both should be 2

    parent := ""
    child := ""
    ; Objects leaked! Can't access them to check refcount anymore

    MsgBox "Objects released from variables but leaked in memory"
}

GetRefCount(&obj) {
    ObjAddRef(ptr := ObjPtr(obj))
    return ObjRelease(ptr)
}

TestCircularRef()
```

### Safe Refcount Wrapper

Prevent accidental misuse:

```ahk
class RefCountInspector {
    static Check(obj) {
        if !IsObject(obj)
            throw ValueError("Not an object")

        try {
            ptr := ObjPtr(obj)
            ObjAddRef(ptr)
            count := ObjRelease(ptr)
            return count
        } catch as err {
            throw Error("Failed to get refcount: " err.Message)
        }
    }

    static Report(obj, label := "") {
        count := RefCountInspector.Check(obj)
        msg := label ? label ": " : ""
        msg .= "RefCount = " count

        if (count = 1)
            msg .= " (normal - only your variable)"
        else if (count = 2)
            msg .= " (2 references - check for extra assignment)"
        else
            msg .= " (multiple references - possible leak)"

        return msg
    }
}

; Usage:
obj := {data: "test"}
MsgBox RefCountInspector.Report(obj, "My object")
```

## Complete Leak Detection Framework

Full system for testing and reporting:

```ahk
#Requires AutoHotkey v2.0

class LeakDetector {
    __New(name := "Leak Test") {
        this.name := name
        this.pid := DllCall("GetCurrentProcessId")
        this.profiler := MemoryProfiler(this.pid)
        this.testResults := []
    }

    RunTest(testFunc, iterations := 1000, sampleInterval := 100) {
        MsgBox("Starting test: " this.name "`nIterations: " iterations)

        this.profiler.Start()

        Loop iterations {
            ; Run the test function
            testFunc(A_Index)

            ; Sample at intervals
            if (Mod(A_Index, sampleInterval) = 0) {
                this.profiler.Sample(A_Index)
                OutputDebug("Test '" this.name "' iteration " A_Index)
            }
        }

        ; Generate report
        report := this.profiler.Report()
        MsgBox(report, this.name " - Results")

        return report
    }

    CompareTests(tests*) {
        results := "Leak Comparison Report`n"
        results .= "====================`n`n"

        for test in tests {
            profiler := MemoryProfiler(this.pid)
            profiler.Start()

            Loop 1000 {
                test.func(A_Index)
                if (Mod(A_Index, 100) = 0)
                    profiler.Sample(A_Index)
            }

            lastSample := profiler.samples[profiler.samples.Length]
            results .= test.name ": " lastSample.delta " KB increase`n"
        }

        MsgBox(results)
    }
}

class MemoryProfiler {
    __New(pid) {
        this.pid := pid
        this.samples := []
        this.baseMem := 0
    }

    Start() {
        this.baseMem := this.GetMemory()
        this.samples := []
        this.samples.Push({iteration: 0, memory: this.baseMem, delta: 0})
        return this.baseMem
    }

    Sample(iteration) {
        currentMem := this.GetMemory()
        delta := currentMem - this.baseMem
        this.samples.Push({iteration: iteration, memory: currentMem, delta: delta})
        return delta
    }

    GetMemory() {
        size := 440
        pmcex := Buffer(size, 0)
        hProcess := DllCall("OpenProcess", "UInt", 0x400|0x0010, "Int", 0, "Ptr", this.pid, "Ptr")
        if (hProcess) {
            if (DllCall("psapi.dll\GetProcessMemoryInfo", "Ptr", hProcess, "Ptr", pmcex, "UInt", size)) {
                memKB := NumGet(pmcex, (A_PtrSize=8 ? 16 : 12), "UInt") / 1024
                DllCall("CloseHandle", "Ptr", hProcess)
                return memKB
            }
            DllCall("CloseHandle", "Ptr", hProcess)
        }
        return 0
    }

    Report() {
        if (this.samples.Length = 0)
            return "No samples recorded"

        report := "Memory Profile Report`n"
        report .= "==================`n`n"
        report .= "Base memory: " Round(this.baseMem, 2) " KB`n"
        report .= "Samples: " this.samples.Length "`n`n"

        ; Show every sample or max 20
        step := Max(1, Floor(this.samples.Length / 20))
        lastShown := 0

        for i, sample in this.samples {
            if (i = 1 || Mod(i-1, step) = 0 || i = this.samples.Length) {
                report .= "Iter " sample.iteration ": "
                report .= Round(sample.memory, 2) " KB (+"
                report .= Round(sample.delta, 2) " KB)`n"
                lastShown := i
            }
        }

        ; Analysis
        lastSample := this.samples[this.samples.Length]
        totalIncrease := lastSample.delta
        avgPerIteration := totalIncrease / lastSample.iteration

        report .= "`n--- Analysis ---`n"
        report .= "Total increase: " Round(totalIncrease, 2) " KB`n"
        report .= "Avg per iteration: " Round(avgPerIteration, 3) " KB`n"

        if (totalIncrease > 1000) {
            report .= "VERDICT: LEAK DETECTED - " Round(totalIncrease/1024, 2) " MB growth`n"
        } else if (totalIncrease > 100) {
            report .= "VERDICT: Possible leak - investigate further`n"
        } else {
            report .= "VERDICT: Memory appears stable`n"
        }

        return report
    }
}

; ====================
; Example Usage
; ====================

; Test Case 1: Known leak (timer not deleted)
LeakingFunction(iteration) {
    gui := Gui()
    SetTimer(() => MsgBox("Leak!"), 10000)  ; Never deleted!
    ; gui goes out of scope but timer keeps closure alive
}

; Test Case 2: Fixed version (timer properly cleaned)
FixedFunction(iteration) {
    static timers := []
    gui := Gui()
    timerFunc := () => MsgBox("Fixed!")
    SetTimer(timerFunc, 10000)
    timers.Push(timerFunc)

    ; Cleanup old timers
    if (timers.Length > 10) {
        old := timers.RemoveAt(1)
        SetTimer(old, "Delete")
    }
}

; Test Case 3: No leak (simple object)
SafeFunction(iteration) {
    gui := Gui()
    obj := {data: Array(100)}
    ; Everything properly released
}

; Run individual test
detector := LeakDetector("Timer Leak Test")
detector.RunTest(LeakingFunction, 500, 50)

; Compare multiple tests
; detector.CompareTests(
;     {name: "Leaking", func: LeakingFunction},
;     {name: "Fixed", func: FixedFunction},
;     {name: "Safe", func: SafeFunction}
; )
```

## Troubleshooting

### "Memory keeps growing but I can't find the leak"

**Problem**: Stress tests show growth but code looks clean.

**Solution**: Narrow down the culprit:

```ahk
; Binary search for leak source
TestSection1() {
    detector := LeakDetector("Section 1")
    detector.RunTest((*) => YourFunction1(), 1000, 100)
}

TestSection2() {
    detector := LeakDetector("Section 2")
    detector.RunTest((*) => YourFunction2(), 1000, 100)
}

TestSection3() {
    detector := LeakDetector("Section 3")
    detector.RunTest((*) => YourFunction3(), 1000, 100)
}

; Run each test separately to identify which section leaks
TestSection1()
; Wait, check results, then test next
TestSection2()
```

### "Memory increases then plateaus"

**Problem**: Memory grows initially then stabilizes.

**Verdict**: Usually **NOT a leak** - just normal caching/pooling.

**Explanation**:
```ahk
; This is normal:
; Iteration 100: 5000 KB
; Iteration 200: 5500 KB
; Iteration 500: 6000 KB
; Iteration 1000: 6000 KB  <- plateaued
; Iteration 2000: 6000 KB  <- stable

; This is a leak:
; Iteration 100: 5000 KB
; Iteration 200: 5500 KB
; Iteration 500: 6500 KB
; Iteration 1000: 7500 KB  <- keeps growing
; Iteration 2000: 9500 KB  <- linear increase
```

### "GetProcessMemoryInfo returns 0"

**Problem**: Function returns 0 or "N/A".

**Causes**:
1. Need administrator privileges
2. Incorrect PID
3. Process already exited

**Fix**:
```ahk
GetProcessMemoryKB(PID) {
    size := 440
    pmcex := Buffer(size, 0)

    ; Request PROCESS_QUERY_INFORMATION | PROCESS_VM_READ
    hProcess := DllCall("OpenProcess", "UInt", 0x400|0x0010, "Int", 0, "Ptr", PID, "Ptr")

    if (!hProcess) {
        lastErr := A_LastError
        MsgBox("Failed to open process. Error: " lastErr "`nMay need admin rights")
        return 0
    }

    result := DllCall("psapi.dll\GetProcessMemoryInfo", "Ptr", hProcess, "Ptr", pmcex, "UInt", size)

    if (!result) {
        lastErr := A_LastError
        DllCall("CloseHandle", "Ptr", hProcess)
        MsgBox("Failed to get memory info. Error: " lastErr)
        return 0
    }

    memKB := NumGet(pmcex, (A_PtrSize=8 ? 16 : 12), "UInt") / 1024
    DllCall("CloseHandle", "Ptr", hProcess)
    return memKB
}
```

### "ObjAddRef/ObjRelease crashes script"

**Problem**: Script crashes when checking refcount.

**Cause**: Manually manipulating refcounts incorrectly.

**Rules**:
1. ALWAYS pair `ObjAddRef()` with `ObjRelease()`
2. NEVER call `ObjRelease()` more times than `ObjAddRef()`
3. NEVER use on invalid pointers

**Safe pattern**:
```ahk
SafeGetRefCount(&obj) {
    if !IsObject(obj)
        return 0

    try {
        ptr := ObjPtr(obj)
        ObjAddRef(ptr)     ; Increment
        count := ObjRelease(ptr)  ; Decrement and get count
        return count
    } catch {
        return -1  ; Error
    }
}
```

### "False positive: Memory grows on first iteration"

**Problem**: First few iterations show growth, then stable.

**Cause**: AHK2 internal caching, string table, or one-time allocations.

**Solution**: Warm up before measuring:

```ahk
profiler := MemoryProfiler()

; Warm-up phase (not measured)
Loop 100 {
    TestFunction()
}

; Now start measuring
profiler.Start()
Loop 1000 {
    TestFunction()
    if (Mod(A_Index, 100) = 0)
        profiler.Sample(A_Index)
}

MsgBox profiler.Report()
```

### "Need to test GUI-heavy code"

**Problem**: GUI creation itself uses memory.

**Solution**: Test GUI cleanup explicitly:

```ahk
TestGUILeak() {
    profiler := MemoryProfiler()
    profiler.Start()
    guis := []

    ; Create many GUIs
    Loop 100 {
        gui := Gui()
        gui.Add("Edit", "w300")
        guis.Push(gui)
        profiler.Sample(A_Index)
    }

    MsgBox "Created 100 GUIs. Check memory."

    ; Destroy GUIs
    for gui in guis {
        gui.Destroy()
    }
    guis := []

    profiler.Sample(101)
    Sleep(1000)  ; Let OS reclaim
    profiler.Sample(102)

    MsgBox profiler.Report()
    ; Memory should return near baseline after destruction
}
```

## Related Documentation

- [Prevent Memory Leaks](prevent-memory-leaks.md) - Fix common leak scenarios
- [Memory Architecture Explanation](../explanation/memory-architecture.md) - How reference counting works
- [Object Lifecycle Reference](../reference/object-lifecycle.md) - `__New()` and `__Delete()` details
- [Windows API Reference](../reference/windows-api.md) - DllCall patterns for system info
