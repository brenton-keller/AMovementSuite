# Reference: Class System

Complete reference for AutoHotkey v2 class architecture, instantiation, inheritance, and built-in classes.

## Related Documentation
- docs/manuals/ahk2/reference/property-system.md - Property definitions and behaviors
- docs/manuals/ahk2/reference/error-types.md - Error handling in classes

---

## Core Architecture

**AutoHotkey v2 uses prototype-based inheritance**, not true class-based inheritance like Java/C#. The `class` keyword is syntactic sugar that creates runtime objects.

### The Two-Object System

Every class definition creates **two separate objects**:

1. **Class Object** - Stored in class name variable (e.g., `Counter`)
   - Contains static members
   - Has `Prototype` property pointing to prototype object
   - Has `Call` method for instance creation

2. **Prototype Object** - Accessible via `ClassName.Prototype`
   - Contains instance methods
   - Contains default instance property values
   - Has `base` property pointing to parent prototype

```
[Class Object: Counter]
├─ total: 0 (static property)
├─ Prototype → [Prototype Object]
│              ├─ count: 0 (instance default)
│              ├─ __New: <Function>
│              └─ base → Object.Prototype
└─ Call: <InstanceFactory>

[Instance: obj1]
├─ count: 1 (own property)
└─ base → Counter.Prototype
```

---

## Class Definition

### Basic Syntax

```ahk
class ClassName {
    ; Instance properties (default values)
    instanceProperty := initialValue

    ; Static properties (shared across all instances)
    static staticProperty := sharedValue

    ; Constructor
    __New(params) {
        ; Initialize instance
    }

    ; Instance methods
    InstanceMethod() {
        ; Access instance: this.property
    }

    ; Static methods
    static StaticMethod() {
        ; Access statics: ClassName.staticProperty
    }
}
```

### Complete Example

```ahk
class Counter {
    ; Static member (class-level)
    static total := 0

    ; Instance member (each instance has own)
    count := 0

    ; Constructor
    __New() {
        Counter.total++           ; ✓ Correct - use class name
        this.count := Counter.total
    }

    ; Instance method
    Increment() {
        this.count++
        Counter.total++
        return this.count
    }

    ; Static method
    static GetTotal() {
        return Counter.total
    }
}

; Usage
c1 := Counter()
c2 := Counter()
MsgBox Counter.GetTotal()  ; 2
```

---

## Instantiation

### How `obj := MyClass()` Works

1. Call `MyClass.Call()` (built-in instance factory)
2. Factory creates new object: `{base: MyClass.Prototype}`
3. Call `__Init` (built-in, initializes instance variables)
4. Call `__New` (user-defined constructor)
5. Return the instance

**Important:** `__New` can return a different object to replace the instance.

### Constructor (__New)

```ahk
class Person {
    __New(name, age) {
        this.name := name
        this.age := age
    }
}

person := Person("Alice", 30)
```

**Return Values:**
- No return / empty return → Use created instance
- Return different object → Replace instance with returned object
- Cannot return primitives (throws error)

### Destructor (__Delete)

```ahk
class Resource {
    __New(path) {
        this.file := FileOpen(path, "r")
    }

    __Delete() {
        ; Called when instance is freed
        if (this.file)
            this.file.Close()
    }
}
```

**When Called:** When reference count reaches zero (object no longer referenced).

**Cannot:** Use `return`, `goto`, `break`, or `continue` in `__Delete`.

---

## Static vs Instance Members

### Access Rules

| Context | Static Access | Instance Access |
|---------|---------------|-----------------|
| **In Constructor** | `ClassName.staticMember` | `this.instanceMember` |
| **In Instance Method** | `ClassName.staticMember` | `this.instanceMember` |
| **In Static Method** | `this.staticMember` (class is `this`) | N/A (no instance) |
| **Outside Class** | `ClassName.staticMember` | `instance.instanceMember` |

### Critical Rule: Always Use Class Name for Statics in Constructors

```ahk
class Counter {
    static total := 0
    count := 0

    __New() {
        ; WRONG - creates instance property!
        this.total++

        ; RIGHT - modifies static
        Counter.total++
        this.count := Counter.total
    }
}
```

**Why `this.total++` is wrong:** When you assign to `this.property`, AHK creates an instance property that shadows the static. The static remains unchanged.

---

## Inheritance

### Syntax

```ahk
class Child extends Parent {
    ; Child members
}
```

### How It Works

- Sets `Child.Prototype.base := Parent.Prototype`
- Child inherits all parent instance methods and properties
- Child can override parent methods
- Child does NOT automatically inherit static members

### Calling Parent Constructor

```ahk
class Parent {
    __New(name) {
        this.name := name
    }
}

class Child extends Parent {
    __New(name, age) {
        super.__New(name)  ; ✓ Must call explicitly
        this.age := age
    }
}
```

**Critical:** If parent has `__New`, child **must** call `super.__New()` to initialize parent properties.

### Method Override

```ahk
class Animal {
    Speak() {
        MsgBox "Generic sound"
    }
}

class Dog extends Animal {
    Speak() {
        MsgBox "Woof!"
    }

    SpeakParent() {
        super.Speak()  ; Call parent version
    }
}
```

### Multiple Inheritance

**NOT SUPPORTED.** AHK2 does not support multiple inheritance. Use composition instead.

```ahk
; Wrong - syntax error
class Child extends Parent1, Parent2 { }

; Right - composition
class Child extends Parent1 {
    __New() {
        this.component := Parent2()
    }
}
```

---

## Prototype Chain Resolution

When you access `obj.property`:

1. Check `obj` for own property
2. Check `obj.base` (prototype)
3. Check `obj.base.base` (parent prototype)
4. Continue up chain to `Object.Prototype`
5. If not found, call `__Get` meta-function (if defined)
6. Return empty string or throw error

```ahk
class Animal {
    species := "Unknown"
    Speak() => MsgBox(this.species)
}

class Dog extends Animal {
    species := "Canine"
}

dog := Dog()
dog.Speak()  ; "Canine"

; Resolution path:
; dog.Speak → dog.base (Dog.Prototype) → found!
; dog.species → dog (own property) → found!
```

---

## Built-In Base Classes

### Object

Base of all objects.

**Key Methods:**
- `Clone()` - Shallow copy of object
- `DefineProp(name, descriptor)` - Define property with get/set
- `DeleteProp(name)` - Remove property
- `GetOwnPropDesc(name)` - Get property descriptor
- `HasOwnProp(name)` - Check if property exists (own, not inherited)
- `OwnProps()` - Iterate own properties

### Array

Indexed collection (1-based).

**Key Properties:**
- `Length` - Number of elements

**Key Methods:**
- `Push(values*)` - Add to end
- `Pop()` - Remove from end
- `InsertAt(pos, values*)` - Insert at position
- `RemoveAt(pos, length:=1)` - Remove from position
- `Clone()` - Shallow copy

```ahk
arr := [1, 2, 3]
arr.Push(4)           ; [1, 2, 3, 4]
value := arr.Pop()    ; value = 4, arr = [1, 2, 3]
arr.InsertAt(2, 99)   ; [1, 99, 2, 3]
```

### Map

Key-value collection.

**Key Properties:**
- `Count` - Number of entries
- `CaseSense` - Case sensitivity for string keys

**Key Methods:**
- `Set(key, value)` - Add or update entry
- `Get(key, default?)` - Retrieve value
- `Has(key)` - Check if key exists
- `Delete(key)` - Remove entry
- `Clear()` - Remove all entries

```ahk
m := Map()
m["name"] := "Alice"
m.Set("age", 30)

if m.Has("name")
    MsgBox m["name"]
```

**Key Type Behavior:**
- Integer and string keys are distinct: `m[5]` ≠ `m["5"]`
- Float keys converted to string: `m[5.0]` becomes `m["5.0"]`

### Buffer

Raw memory buffer.

```ahk
buf := Buffer(100)          ; Allocate 100 bytes
NumPut("Int", 42, buf, 0)   ; Write integer at offset 0
value := NumGet(buf, 0, "Int")  ; Read integer
```

### Func

Function object.

**Key Properties:**
- `Name` - Function name
- `IsBuiltIn` - True if built-in function
- `IsVariadic` - True if variadic function
- `MinParams` - Minimum parameter count
- `MaxParams` - Maximum parameter count

**Key Methods:**
- `Call(params*)` - Call function
- `Bind(params*)` - Create BoundFunc with bound parameters

```ahk
fn := MyFunction
MsgBox fn.Name        ; "MyFunction"
MsgBox fn.MinParams   ; Minimum params

bound := fn.Bind(1, 2)  ; Bind first two parameters
bound(3)                ; Calls MyFunction(1, 2, 3)
```

### Gui

GUI window object.

See official documentation for complete GUI reference.

---

## Meta-Functions

Special methods called by the runtime for specific operations.

### Property Access

| Method | Called When | Parameters | Return |
|--------|-------------|------------|--------|
| `__Get(name, params)` | Property read (if property doesn't exist) | `name`, `params` | Property value |
| `__Set(name, params, value)` | Property write (if property doesn't exist) | `name`, `params`, `value` | Stored value |
| `__Call(name, params)` | Method call (if method doesn't exist) | `name`, `params` | Method result |

**Important:** These are only called if the property/method doesn't exist in the normal prototype chain.

```ahk
class Proxy {
    __Get(name, params) {
        return "Accessed: " . name
    }

    __Set(name, params, value) {
        MsgBox "Set " . name . " = " . value
        return value
    }
}

p := Proxy()
MsgBox p.anyProperty  ; "Accessed: anyProperty"
p.test := 42          ; MsgBox "Set test = 42"
```

### Object Lifecycle

| Method | Called When | Parameters | Notes |
|--------|-------------|------------|-------|
| `__Init()` | After instance created, before `__New` | None | Built-in, initializes instance variables |
| `__New(params*)` | After `__Init` | Constructor params | User-defined initialization |
| `__Delete()` | When object freed | None | Cleanup resources |

### Enumeration

| Method | Called When | Parameters | Return |
|--------|-------------|------------|--------|
| `__Enum(numberOfVars)` | `for` loop | `1` or `2` | Enumerator object |

```ahk
class CustomCollection {
    items := []

    __Enum(numberOfVars) {
        return this.items.__Enum(numberOfVars)
    }
}

coll := CustomCollection()
coll.items := ["a", "b", "c"]

for item in coll
    MsgBox item
```

---

## Memory Management

### Reference Counting

AHK2 uses reference counting for garbage collection:
- Each object maintains a reference count
- Assigning object to variable increments count
- Clearing variable or letting it go out of scope decrements count
- When count reaches zero, `__Delete` is called and object is freed

### Circular References

**AHK2 CANNOT detect circular references** - they cause memory leaks.

```ahk
; Memory leak - circular reference
a := {}
b := {a: a}
a.b := b  ; Both reference each other
; Neither freed until script exits
```

**Solution:** Manually break cycles in `__Delete`:

```ahk
class ParentWidget {
    __New() {
        this.children := []
    }

    AddChild(child) {
        this.children.Push(child)
        child.parent := this  ; Potential cycle
    }

    __Delete() {
        ; Break cycles
        for child in this.children
            child.parent := ""
        this.children := []
    }
}
```

### Memory Overhead

- Object overhead: ~16 bytes per property
- Object with 20 properties: ~320 bytes overhead
- For most scripts: negligible

---

## Inheritance vs Composition

### When to Use Inheritance

- Clear "is-a" relationship
- Minimal specialization (1-2 levels)
- Shared behavior and state
- Example: `Dog extends Animal`

### When to Use Composition

- "has-a" relationship
- Need flexibility
- Extending built-in classes (doesn't work well)
- Multiple inheritance needed
- Example: `Manager has Employee`

```ahk
; Composition example
class Manager {
    __New() {
        this.employee := Employee()
        this.team := []
    }
}
```

**Community Consensus:** Favor composition over deep inheritance. AHK2's prototype system makes inheritance brittle for complex hierarchies.

---

## Common Patterns

### Singleton Pattern

```ahk
class Singleton {
    static instance := ""

    __New() {
        if (Singleton.instance)
            throw Error("Use Singleton.GetInstance()")
        Singleton.instance := this
    }

    static GetInstance() {
        if (!Singleton.instance)
            Singleton.instance := Singleton()
        return Singleton.instance
    }
}

; Usage
s := Singleton.GetInstance()
```

### Factory Pattern

```ahk
class ShapeFactory {
    static Create(type, params*) {
        switch type {
            case "circle": return Circle(params*)
            case "square": return Square(params*)
            default: throw ValueError("Unknown shape: " . type)
        }
    }
}

; Usage
shape := ShapeFactory.Create("circle", 5)
```

### Manager Pattern (Static Class)

```ahk
class ToggleManager {
    static toggles := Map()

    static Create(name, initialState := true) {
        this.toggles[name] := {state: initialState}
    }

    static Toggle(name) {
        if this.toggles.Has(name)
            this.toggles[name].state := !this.toggles[name].state
    }

    static GetState(name) {
        return this.toggles.Has(name) ? this.toggles[name].state : false
    }
}

; Usage
ToggleManager.Create("AutoSave", true)
ToggleManager.Toggle("AutoSave")
MsgBox ToggleManager.GetState("AutoSave")
```

---

## Performance Considerations

- **Method calls:** Negligible overhead for typical use
- **Property access:** Prototype chain lookup is fast (O(1) per level)
- **Memory:** 16 bytes per property, acceptable for most scripts
- **Optimization:** Don't optimize prematurely - profile first

---

## Common Mistakes

### 1. Using `this` for Statics in Constructors

```ahk
; WRONG
__New() {
    this.total++  ; Creates instance property!
}

; RIGHT
__New() {
    Counter.total++  ; Modifies static
}
```

### 2. Forgetting `super.__New()`

```ahk
; WRONG - parent properties not initialized
class Child extends Parent {
    __New(params) {
        ; Missing super.__New()
        this.childProp := params
    }
}

; RIGHT
class Child extends Parent {
    __New(params) {
        super.__New(params)  ; Initialize parent
        this.childProp := params
    }
}
```

### 3. Extending Built-In Classes

```ahk
; WRONG - won't work
class MyListView extends Gui.ListView { }

; RIGHT - use composition
class MyListViewManager {
    __New(guiObj) {
        this.lv := guiObj.Add("ListView", "w500", ["Name"])
    }
}
```

### 4. Circular References

```ahk
; WRONG - memory leak
parent.child := child
child.parent := parent

; RIGHT - break cycles
parent.__Delete() {
    this.child.parent := ""
}
```

---

## See Also

- docs/manuals/ahk2/reference/property-system.md - Property definitions
- docs/manuals/ahk2/reference/error-types.md - Error handling
- Official: [Classes](https://www.autohotkey.com/docs/v2/Objects.htm)
