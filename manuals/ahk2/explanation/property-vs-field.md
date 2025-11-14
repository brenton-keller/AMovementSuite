# Explanation: Why Properties Exist

**Understanding properties as method dispatch, not variable access**

Most developers approaching AutoHotkey v2 think properties are "variables with extra steps"—a syntactic convenience for getter/setter methods. This mental model leads to infinite recursion bugs, confusion about backing fields, and misuse of properties where simple fields suffice. The truth is more fundamental: **properties are method calls disguised as field access**, serving as hooks in the resolution chain that enable validation, caching, and reactive patterns while maintaining the syntax of simple assignment.

## The Fundamental Misconception

### What You Probably Think

- Properties are variables that store values with attached get/set methods
- `obj.property` inside a property getter accesses the stored value
- Properties and methods are fundamentally different things
- Underscore prefix has special meaning in AHK2

### The Reality

- Properties are **function pairs** that execute on access
- `obj.property` inside its own getter **calls the getter again** (infinite recursion)
- Methods ARE properties (with a `call` accessor)
- Underscores are **pure convention**—no special behavior

## Properties vs. Fields: The Architectural Difference

### Value Properties (Simple Storage)

```ahk
class Example {
    MyProperty := "initial value"  ; Value property
}

obj := Example()
obj.MyProperty := "new value"  ; Direct assignment
MsgBox obj.MyProperty          ; Direct access
```

**What happens:** Direct memory access. No function calls. This is a simple field.

### Dynamic Properties (Function Dispatch)

```ahk
class Example {
    MyProperty {
        get => "computed every time"
        set => MsgBox("setter called with: " value)
    }
}

obj := Example()
obj.MyProperty := "test"  ; Calls setter function
MsgBox obj.MyProperty     ; Calls getter function
```

**What happens:** Each access/assignment calls the corresponding function. No storage exists unless you create it explicitly.

## The Infinite Recursion Trap (And Why It Happens)

### The Classic Mistake

```ahk
class MyClass {
    property {
        set => this.property := value  ; ← INFINITE RECURSION
        get => this.property           ; ← INFINITE RECURSION
    }
}

obj := MyClass()
obj.property := "test"  ; Script freezes, crashes, or hangs forever
```

### Why This Happens: Step-by-Step

```
1. User: obj.property := "value"
2. AHK:  Call property setter
3. Setter: this.property := value
4. AHK:  "this.property" means call property setter
5. Setter: this.property := value
6. AHK:  Call property setter again...
∞. Stack overflow → crash
```

**The setter calls itself. Forever.**

### The Solution: Use a Different Name

```ahk
class MyClass {
    property {
        set => this._property := value  ; Different name!
        get => this._property           ; Accesses different storage
    }
}
```

**Why this works:** `_property` is a **different property** (or value field). No recursion because you're not calling the property setter—you're setting a simple value field.

### The Underscore Convention (Not a Language Feature)

**Community wisdom from Lexikos:**

> "You don't need to use underscores; you just need to use a name different to the name of the property. This should be obvious, since you don't want to 'call' the property again from inside the property."

```ahk
banana {
    set => this.myBanana := value    ; Works fine
    get => this.myBanana
}

Count {
    set => this.John := value        ; Also works
    get => this.John
}
```

The underscore prefix is **convention** that signals "internal backing field," but AHK doesn't enforce or recognize it. You can use any different name.

## Property Resolution: How AHK Finds Members

When you write `obj.property`, AHK follows this chain:

```
1. Check obj's own properties
2. Check obj's base (parent class)
3. Check base's base (walk up chain)
4. If found: Use that property (call getter/setter or access value)
5. If not found: Call __Get("property") if defined
6. If still not found: Return empty or error
```

**Key insight:** Properties short-circuit the chain. `__Get` is a fallback, not the primary mechanism.

**Critical detail:** `obj.property` does NOT translate to `obj.__Get("property")` if the property exists. Properties are accessed directly.

## Why Properties Exist: The Design Philosophy

### Use Case 1: Validation on Assignment

**Without properties:**
```ahk
class Person {
    age := 0

    SetAge(value) {
        if !(value is "Integer" && value >= 0)
            throw ValueError("Age must be non-negative integer")
        this.age := value
    }
}

person := Person()
person.SetAge(25)  ; Verbose, unnatural syntax
```

**With properties:**
```ahk
class Person {
    Age {
        get => this._age
        set {
            if !(value is "Integer" && value >= 0)
                throw ValueError("Age must be non-negative integer")
            this._age := value
        }
    }
}

person := Person()
person.Age := 25  ; Natural assignment syntax
```

**Why properties win:** Natural syntax while enforcing invariants. Users interact with `Age` as if it's a field, but validation runs automatically.

### Use Case 2: Computed Values with Caching

```ahk
class Rectangle {
    __New(width, height) {
        this.Width := width
        this.Height := height
    }

    Width {
        get => this._width
        set {
            this._width := value
            this._InvalidateArea()  ; Cache invalidation
        }
    }

    Height {
        get => this._height
        set {
            this._height := value
            this._InvalidateArea()
        }
    }

    ; Computed property with cache
    Area {
        get {
            if !this.HasOwnProp("_areaCache")
                this._areaCache := this._width * this._height
            return this._areaCache
        }
    }

    _InvalidateArea() {
        if this.HasOwnProp("_areaCache")
            this.DeleteProp("_areaCache")
    }
}

rect := Rectangle(10, 20)
MsgBox rect.Area    ; 200 (computed and cached)
MsgBox rect.Area    ; 200 (from cache—no recalculation)
rect.Width := 15    ; Triggers cache invalidation
MsgBox rect.Area    ; 300 (recomputed)
```

**Why properties win:** Expensive computations are cached and automatically invalidated when dependencies change. Users access `Area` naturally without knowing about the caching logic.

### Use Case 3: Observer Notifications

```ahk
class ObservableRectangle {
    __New(width, height) {
        this._observers := []
        this.Width := width
        this.Height := height
    }

    OnChange(callback) {
        this._observers.Push(callback)
    }

    Width {
        get => this._width
        set {
            oldValue := this._width ?? "unset"
            this._width := value
            this._InvalidateArea()
            this._Notify("Width", oldValue, value)
        }
    }

    Height {
        get => this._height
        set {
            oldValue := this._height ?? "unset"
            this._height := value
            this._InvalidateArea()
            this._Notify("Height", oldValue, value)
        }
    }

    Area {
        get {
            if !this.HasOwnProp("_areaCache")
                this._areaCache := this._width * this._height
            return this._areaCache
        }
    }

    _InvalidateArea() {
        if this.HasOwnProp("_areaCache")
            this.DeleteProp("_areaCache")
    }

    _Notify(propName, oldVal, newVal) {
        for callback in this._observers {
            try {
                callback(propName, oldVal, newVal)
            } catch as err {
                OutputDebug("Observer error: " err.Message)
            }
        }
    }
}

; Usage
rect := ObservableRectangle(10, 20)
rect.OnChange((prop, old, new) =>
    MsgBox(prop " changed: " old " → " new))
rect.Width := 15  ; Triggers notification
```

**Why properties win:** Changes automatically notify observers. Reactive patterns with natural assignment syntax.

## When to Use Properties vs. Methods

### The Semantic Test

**Properties feel like data. Methods feel like actions.**

| Scenario | Use Property | Use Method |
|----------|--------------|------------|
| **Naming** | Noun/Adjective (`Count`, `IsReady`) | Verb (`Calculate()`, `Save()`) |
| **Speed** | Fast (<1ms, cheap) | Any (can be expensive) |
| **Side Effects** | None (read) or minimal (write) | Expected and visible |
| **Deterministic** | Same value on repeated reads | Can vary per call |
| **Parameters** | None (or simple index) | Required or complex |
| **Returns Array** | Avoid (confusion with indexing) | ✓ Acceptable |
| **Throws in Getter** | Avoid (unexpected) | ✓ Acceptable |
| **Expensive** | No—users assume cheap | ✓ Expected to be explicit |

### Real-World Examples

**✓ Good Property Choices:**
```ahk
class ListView {
    Count => this.items.Length          ; Fast data access
    SelectedIndex => this._selectedIdx  ; State query
    BackColor := 0xFFFFFF              ; Simple attribute
}
```

**✗ Bad Property Choice (Should Be Method):**
```ahk
class Calculator {
    ; BAD - Hides 10-second computation
    Result => this.PerformLongCalculation()

    ; GOOD - Explicit action
    Calculate() => this.PerformLongCalculation()
}
```

## Performance: Properties vs. Direct Fields

### Theoretical Hierarchy

```
Fastest:  Direct field (obj.field := value)
   ↓      Value property (simple storage)
   ↓      Getter/setter property (function call)
   ↓      Method call (function lookup + call)
Slowest:  Meta-function (__Get/__Set)
```

### Reality Check

**No published AHK2 benchmarks exist** comparing property vs. method overhead. However:

**Community consensus:** "Negligible for typical usage"

**Estimated overhead:**
- Property getter/setter: Single function call (microseconds)
- Compared to direct field: Additional indirection through prototype chain

**When it matters:**
- Tight loops with >10,000 iterations
- High-frequency callbacks (>10,000/second)
- Real-time pixel searching

**When it doesn't matter (99% of code):**
- GUI applications
- User input handling
- Automation scripts
- Standard business logic

### Hot Loop Optimization

**❌ Wrong (Property Access in Loop):**
```ahk
Loop {
    if A_Index > array.Length  ; Property accessed every iteration
        break
    ProcessItem(array[A_Index])
}
```

**✅ Right (Cache Outside Loop):**
```ahk
len := array.Length  ; Cache once
Loop len {
    ProcessItem(array[A_Index])
}
```

## Caching Strategies

### Pattern 1: Always Recalculate (Simplest)

```ahk
class Circle {
    Radius := 0

    Area {
        get => 3.14159 * this.Radius ** 2
    }
}
```

**Pros:** Simple, always correct, no cache invalidation
**Cons:** Recalculates every access
**Use when:** Computation is trivial (<1ms)

### Pattern 2: Lazy Cache (Cache on First Access)

```ahk
class DataProcessor {
    Data := ""

    ProcessedData {
        get {
            if !this.HasOwnProp("_processedCache") {
                ; Expensive operation
                Sleep(500)
                this._processedCache := StrUpper(this.Data)
            }
            return this._processedCache
        }
    }

    Data {
        get => this._data
        set {
            this._data := value
            ; Invalidate cache when dependency changes
            if this.HasOwnProp("_processedCache")
                this.DeleteProp("_processedCache")
        }
    }
}
```

**Pros:** Only computes when needed, stays fresh
**Cons:** Requires manual invalidation
**Use when:** Expensive computation, infrequent changes

### Pattern 3: Time-Based Cache

```ahk
class LiveData {
    _cacheTimeout := 5000  ; 5 seconds

    CurrentPrice {
        get {
            now := A_TickCount

            ; Check if cache fresh
            if this.HasOwnProp("_priceCache")
                && this.HasOwnProp("_cacheTime")
                && (now - this._cacheTime) < this._cacheTimeout
            {
                return this._priceCache
            }

            ; Fetch new data
            this._priceCache := this._FetchPrice()
            this._cacheTime := now
            return this._priceCache
        }
    }

    _FetchPrice() {
        Sleep(200)  ; Simulate API call
        return Random(100, 200)
    }
}
```

**Pros:** Handles changing data, automatic refresh
**Cons:** May return stale data briefly
**Use when:** External data, acceptable staleness window

## Common Anti-Patterns

### Anti-Pattern 1: Validation in Getter

**❌ Wrong:**
```ahk
class BadValidation {
    Value {
        get {
            ; Too late! Invalid value already stored
            if !(this._value is "Integer")
                throw TypeError("Value must be integer")
            return this._value
        }
        set => this._value := value
    }
}
```

**✅ Right:**
```ahk
class GoodValidation {
    Value {
        get => this._value
        set {
            ; Validate BEFORE storing
            if !(value is "Integer")
                throw TypeError("Value must be integer")
            this._value := value
        }
    }
}
```

**Why:** Getters run on every read. If you stored an invalid value, every read throws. Validate on write to catch bad data at the source.

### Anti-Pattern 2: Side Effects in Getters

**❌ Wrong:**
```ahk
class SideEffectGetter {
    Count {
        get {
            this._accessCount++               ; Modifying state
            FileAppend("Access`n", "log.txt") ; I/O operation
            this.LastAccessed := A_Now        ; Changing properties
            return this._count
        }
    }
}
```

**✅ Right:**
```ahk
class CleanGetter {
    Count {
        get {
            ; Simple, fast, no side effects
            return this._count
        }
    }

    ; Explicit method for side effects
    GetCountAndLog() {
        this._accessCount++
        FileAppend("Access`n", "log.txt")
        return this._count
    }
}
```

**Why:** Getters may be called by debuggers, multiple times unexpectedly, or in hot loops. Side effects make behavior unpredictable.

**Exception:** Lazy initialization is acceptable (one-time cache population).

### Anti-Pattern 3: Not Invalidating Caches

**❌ Wrong:**
```ahk
class StaleCache {
    Width := 10
    Height := 20

    Area {
        get {
            if !this.HasOwnProp("_areaCache")
                this._areaCache := this.Width * this.Height
            return this._areaCache
        }
    }
}

obj := StaleCache()
MsgBox obj.Area  ; 200 (correct)
obj.Width := 30  ; Cache NOT invalidated
MsgBox obj.Area  ; 200 (wrong! Should be 600)
```

**✅ Right:**
```ahk
class FreshCache {
    Width {
        get => this._width
        set {
            this._width := value
            this._InvalidateArea()  ; Key line
        }
    }

    Height {
        get => this._height
        set {
            this._height := value
            this._InvalidateArea()
        }
    }

    Area {
        get {
            if !this.HasOwnProp("_areaCache")
                this._areaCache := this._width * this._height
            return this._areaCache
        }
    }

    _InvalidateArea() {
        if this.HasOwnProp("_areaCache")
            this.DeleteProp("_areaCache")
    }
}
```

## The Mental Model: Properties as Hooks

Properties are **hooks in the resolution chain** that intercept member access:

```
obj.property
   ↓
Is "property" a dynamic property? (has get/set)
   ↓ YES
Call get function
   ↓
Return result

obj.property := value
   ↓
Is "property" a dynamic property? (has set)
   ↓ YES
Call set function with value
   ↓
Store result (if setter returns)
```

**Not:**
```
obj.property → look up stored value
obj.property := value → store value
```

**Instead:**
```
obj.property → call get() function
obj.property := value → call set(value) function
```

## Key Insights

**Insight 1: Properties are method dispatch**
Dynamic properties don't contain values—they're function pairs that execute on access.

**Insight 2: Backing fields must have different names**
Using `this.property` inside property getter/setter calls the property recursively. Use a different name like `this._property`.

**Insight 3: Properties enable declarative validation**
Natural assignment syntax (`obj.Age := 25`) with automatic validation. Users don't need to call setter methods.

**Insight 4: Caching requires invalidation strategy**
Computed properties can cache results, but dependencies must trigger invalidation when they change.

**Insight 5: Properties vs. methods is semantic**
If it feels like data (noun/adjective), use property. If it feels like action (verb), use method.

**Insight 6: Performance rarely matters**
Property overhead is negligible for typical usage. Only optimize in proven bottlenecks (hot loops >10K iterations).

**Insight 7: Getters should be pure**
Avoid side effects in getters. They may be called unexpectedly by debuggers or multiple times in expressions.

## Cross-References

Related topics:
- docs/manuals/ahk2/reference/property-syntax.md - Property syntax reference
- docs/manuals/ahk2/how-to/property-patterns.md - Common property patterns
- docs/manuals/ahk2/explanation/class-system.md - Class architecture
- docs/manuals/ahk2/reference/meta-functions.md - __Get/__Set meta-functions
