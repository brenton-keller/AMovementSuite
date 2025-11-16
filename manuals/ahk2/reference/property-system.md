# Reference: Property System

Complete reference for AutoHotkey v2 property definitions, getters/setters, computed properties, and patterns.

## Related Documentation
- docs/manuals/ahk2/reference/class-system.md - Class definitions and inheritance
- docs/manuals/ahk2/reference/error-types.md - PropertyError and validation errors

---

## Property Types

### Value Properties (Simple Storage)

Direct storage with default value.

```ahk
class Example {
    MyProperty := "initial value"
}

obj := Example()
obj.MyProperty := "new value"  ; Direct storage
MsgBox obj.MyProperty          ; "new value"
```

**Characteristics:**
- Simple field storage
- No function calls on access
- Fast (direct memory access)
- Use for simple data storage

### Dynamic Properties (Getter/Setter Functions)

Function pairs executed on access.

```ahk
class Example {
    MyProperty {
        get => this._value
        set => this._value := value
    }
}
```

**Characteristics:**
- Execute functions on access
- Can validate, transform, compute values
- Can have side effects (logging, notifications)
- Require separate backing field

---

## Property Definition Syntax

### Getter Only (Read-Only)

```ahk
class Rectangle {
    Width := 0
    Height := 0

    Area {
        get => this.Width * this.Height
    }
}

rect := Rectangle()
rect.Width := 10
rect.Height := 20
MsgBox rect.Area       ; 200
rect.Area := 100       ; PropertyError - no setter
```

### Setter Only (Write-Only)

```ahk
class Logger {
    LogLevel {
        set {
            this._level := value
            FileAppend("Level changed to: " . value . "`n", "log.txt")
        }
    }
}
```

### Both Getter and Setter

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
```

### Arrow Function Syntax (Concise)

```ahk
; Single expression (no validation)
Property {
    get => this._property
    set => this._property := value
}

; Multiple statements require v2.1+ block syntax
Property {
    set => (
        this.Validate(value),
        this._property := value
    )
}
```

### Full Function Syntax (Flexible)

```ahk
Property {
    get {
        ; Multiple statements
        ; Complex logic
        return this._property
    }

    set {
        ; Validation
        if !IsValid(value)
            throw ValueError("Invalid value")

        ; Processing
        this._property := Transform(value)

        ; Side effects
        this.NotifyChange()

        ; Must return value for chaining
        return value
    }
}
```

---

## The Backing Field Pattern

**Critical Rule:** Property getter/setter CANNOT reference itself - causes infinite recursion.

```ahk
; WRONG - infinite recursion
Property {
    get => this.Property        ; Calls getter again!
    set => this.Property := value  ; Calls setter again!
}

; RIGHT - use different backing field name
Property {
    get => this._property       ; Different name
    set => this._property := value
}
```

**Naming Convention:** Prefix backing field with underscore (`_property`) to indicate internal storage.

**Important:** Underscore has NO special meaning in AHK2 - it's pure convention.

---

## Property Resolution Chain

When accessing `obj.property`:

1. Check `obj` for own property named "property"
2. Walk up `base` chain (parent classes)
3. If found property, use its getter (if dynamic) or value (if simple)
4. If NOT found, call `__Get("property")` meta-function (if defined)
5. Return empty string or throw error

**Key Insight:** Properties are checked BEFORE `__Get` is called. `__Get` is only a fallback for undefined properties.

---

## Validation Patterns

### Type Validation

```ahk
Age {
    get => this._age

    set {
        if !(value is "Integer")
            throw TypeError("Age must be integer")
        if (value < 0)
            throw ValueError("Age cannot be negative")
        this._age := value
    }
}
```

### Range Validation

```ahk
Volume {
    get => this._volume

    set {
        if (value < 0 || value > 100)
            throw ValueError("Volume must be 0-100")
        this._volume := value
    }
}
```

### Constrained Values (Clamping)

```ahk
Brightness {
    get => this._brightness

    set => this._brightness := Max(0, Min(value, 255))
}
```

### Pattern Validation (Regex)

```ahk
Email {
    get => this._email

    set {
        if !RegExMatch(value, "^[\w.]+@[\w.]+\.\w+$")
            throw ValueError("Invalid email format")
        this._email := value
    }
}
```

---

## Computed Properties

### Simple Computation

```ahk
class Circle {
    Radius := 0

    Area {
        get => 3.14159 * this.Radius ** 2
    }

    Circumference {
        get => 2 * 3.14159 * this.Radius
    }
}
```

**Use When:** Computation is cheap (<1ms), result changes frequently, dependencies are simple.

### Cached Computation (Lazy Evaluation)

```ahk
class DataProcessor {
    Data := []

    ProcessedData {
        get {
            ; Only compute once
            if !this.HasOwnProp("_processedCache")
                this._processedCache := this.ExpensiveProcessing()
            return this._processedCache
        }
    }

    ExpensiveProcessing() {
        Sleep(500)  ; Simulate expensive operation
        return Transform(this.Data)
    }
}
```

**Use When:** Computation is expensive (>10ms), result accessed multiple times, dependencies don't change often.

### Cache Invalidation

```ahk
class Rectangle {
    Width {
        get => this._width
        set {
            this._width := value
            this._InvalidateArea()  ; Invalidate dependent cache
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

**Pattern:** Invalidate caches when dependencies change, recompute on next access.

---

## Observable Properties (Change Notifications)

### Basic Observer Pattern

```ahk
class ObservableValue {
    __New(initialValue := "") {
        this._value := initialValue
        this._observers := []
    }

    Value {
        get => this._value

        set {
            oldValue := this._value
            this._value := value
            this._Notify(oldValue, value)
        }
    }

    OnChange(callback) {
        this._observers.Push(callback)
    }

    _Notify(oldValue, newValue) {
        for callback in this._observers {
            try {
                callback(oldValue, newValue)
            } catch as err {
                ; Log but don't fail other observers
                OutputDebug("Observer error: " . err.Message)
            }
        }
    }
}

; Usage
obs := ObservableValue(10)
obs.OnChange((old, new) => MsgBox("Changed: " . old . " -> " . new))
obs.Value := 20  ; Triggers notification
```

### Multi-Property Observer

```ahk
class ObservableModel {
    __New() {
        this._observers := []
    }

    Name {
        get => this._name
        set {
            oldValue := this._name ?? ""
            this._name := value
            this._NotifyChange("Name", oldValue, value)
        }
    }

    Age {
        get => this._age
        set {
            oldValue := this._age ?? 0
            this._age := value
            this._NotifyChange("Age", oldValue, value)
        }
    }

    OnChange(callback) {
        this._observers.Push(callback)
    }

    _NotifyChange(propName, oldVal, newVal) {
        for callback in this._observers
            callback(propName, oldVal, newVal)
    }
}

; Usage
model := ObservableModel()
model.OnChange((prop, old, new) =>
    ToolTip(prop . ": " . old . " -> " . new))
model.Name := "Alice"
model.Age := 30
```

---

## Read-Only Properties

### Computed Read-Only

```ahk
FullName {
    get => this.FirstName . " " . this.LastName
}
```

### Stored Read-Only (Set Only in Constructor)

```ahk
class Person {
    __New(id, name) {
        this._id := id  ; Set once
        this.Name := name
    }

    ID {
        get => this._id
        set => throw PropertyError("ID is read-only")
    }
}
```

### True Immutable (No Setter)

```ahk
CreatedAt {
    get => this._createdAt
    ; No setter - cannot be changed
}
```

---

## Property vs Method Decision Matrix

| Use Property When | Use Method When |
|-------------------|-----------------|
| Represents state/data | Represents action |
| Fast (<1ms) | Can be slow (>10ms) |
| No side effects (read) | Has side effects |
| Deterministic (same value on repeated reads) | Result can vary per call |
| No parameters needed | Requires parameters |
| Natural as noun/adjective (`Count`, `IsReady`) | Natural as verb (`Calculate()`, `Save()`) |

### Examples

```ahk
; GOOD - Properties
Count => this.items.Length          ; Data access
IsEmpty => this.items.Length = 0    ; State query
BackColor := 0xFFFFFF              ; Simple attribute

; BAD - Should be methods
Result => this.LongCalculation()    ; Expensive (should be Calculate())
Items => this.FetchFromAPI()        ; Side effects (should be FetchItems())
```

---

## Property Performance

### Read Access

- **Value property:** Direct memory access (~nanoseconds)
- **Getter without cache:** Function call + computation
- **Getter with cache:** Function call + cache check

### Write Access

- **Value property:** Direct memory write (~nanoseconds)
- **Setter:** Function call + validation + side effects

### Recommendations

- Properties add negligible overhead for normal use
- Only matters in tight loops (>10,000 iterations/second)
- Profile before optimizing
- Semantic correctness > micro-optimization

---

## Common Patterns

### Toggle Property

```ahk
IsEnabled {
    get => this._enabled
    set => this._enabled := !!value  ; Force boolean
}

Toggle() {
    this.IsEnabled := !this.IsEnabled
}
```

### Dependent Properties

```ahk
class Temperature {
    Celsius {
        get => this._celsius
        set => this._celsius := value
    }

    Fahrenheit {
        get => this._celsius * 9/5 + 32
        set => this._celsius := (value - 32) * 5/9
    }
}

temp := Temperature()
temp.Celsius := 100
MsgBox temp.Fahrenheit  ; 212
```

### Lazy Initialization

```ahk
ExpensiveData {
    get {
        if !this.HasOwnProp("_expensiveData")
            this._expensiveData := this.LoadExpensiveData()
        return this._expensiveData
    }
}
```

### Property Chaining (Return `this`)

```ahk
class Builder {
    Width {
        get => this._width
        set {
            this._width := value
            return this  ; Enable chaining
        }
    }

    Height {
        get => this._height
        set {
            this._height := value
            return this
        }
    }
}

; Usage
builder := Builder()
builder.Width := 100
       .Height := 200
```

---

## Anti-Patterns

### 1. Validation in Getter

```ahk
; WRONG - validates on read, but invalid value already stored
Property {
    get {
        if !IsValid(this._property)
            throw ValueError("Invalid")
        return this._property
    }
    set => this._property := value
}

; RIGHT - validate on write
Property {
    get => this._property
    set {
        if !IsValid(value)
            throw ValueError("Invalid")
        this._property := value
    }
}
```

### 2. Side Effects in Getter

```ahk
; WRONG - getter modifies state
Count {
    get {
        this._accessCount++  ; Side effect!
        return this._count
    }
}

; RIGHT - getters should be pure
Count {
    get => this._count
}

GetCountAndLog() {
    this._accessCount++
    return this._count
}
```

**Exception:** Lazy initialization (one-time computation) is acceptable.

### 3. Silent Failure in Setter

```ahk
; WRONG - silently ignores invalid values
Age {
    set {
        if (value >= 0)
            this._age := value
        ; If value < 0, nothing happens - bug hidden!
    }
}

; RIGHT - throw explicit error
Age {
    set {
        if (value < 0)
            throw ValueError("Age cannot be negative")
        this._age := value
    }
}
```

### 4. Not Returning from Setter

```ahk
; WRONG - breaks assignment chaining
Property {
    set {
        this._property := value
        ; No return
    }
}

x := obj.Property := 42  ; x is empty!

; RIGHT - return value
Property {
    set {
        this._property := value
        return value  ; Enable chaining
    }
}

x := obj.Property := 42  ; x is 42
```

### 5. Not Invalidating Dependent Caches

```ahk
; WRONG - stale cache
Width := 10

Area {
    get {
        if !this.HasOwnProp("_areaCache")
            this._areaCache := this.Width * this.Height
        return this._areaCache
    }
}

obj.Width := 20      ; Cache NOT invalidated
MsgBox obj.Area      ; Shows old value!

; RIGHT - invalidate on dependency change
Width {
    get => this._width
    set {
        this._width := value
        if this.HasOwnProp("_areaCache")
            this.DeleteProp("_areaCache")
    }
}
```

---

## Debugging Properties

### Log Property Access

```ahk
Property {
    get {
        OutputDebug("Get Property: " . this._property)
        return this._property
    }

    set {
        OutputDebug("Set Property: " . value)
        this._property := value
        return value
    }
}
```

### Trace Property Changes

```ahk
Property {
    set {
        if (A_IsCompiled = 0) {  ; Debug mode only
            stack := Exception("", -1).Stack
            FileAppend(A_Now . " - Set to " . value . "`n" . stack . "`n",
                       "property_trace.log")
        }
        this._property := value
        return value
    }
}
```

---

## Built-In Property Descriptor

Get property metadata using `GetOwnPropDesc()`:

```ahk
class Example {
    MyProperty {
        get => this._value
        set => this._value := value
    }
}

desc := Example.Prototype.GetOwnPropDesc("MyProperty")
MsgBox "Has Get: " . desc.HasProp("Get")   ; True
MsgBox "Has Set: " . desc.HasProp("Set")   ; True
MsgBox "Has Call: " . desc.HasProp("Call") ; False (not a method)
```

**Descriptor Properties:**
- `Get` - Getter function object (if defined)
- `Set` - Setter function object (if defined)
- `Call` - Call method (for methods)
- `Value` - Direct value (for value properties)

---

## Advanced: Dynamic Property Definition

Use `DefineProp()` to add properties at runtime:

```ahk
obj := {}
obj.DefineProp("DynamicProp", {
    get: () => "Getter value",
    set: (this, value) => MsgBox("Set to: " . value)
})

MsgBox obj.DynamicProp      ; "Getter value"
obj.DynamicProp := "test"   ; Shows MsgBox
```

---

## See Also

- docs/manuals/ahk2/reference/class-system.md - Class definitions
- docs/manuals/ahk2/reference/error-types.md - PropertyError
- Official: [Properties](https://www.autohotkey.com/docs/v2/Objects.htm#Custom_Properties)
