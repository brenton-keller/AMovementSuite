# Explanation: Why AHK2 Classes Are Prototype-Based

**Understanding JavaScript-style prototypes, not Java-style classes**

AutoHotkey v2 classes are **runtime objects masquerading as compile-time blueprints**. When you write `class MyClass {}`, you're not creating a compile-time blueprint like Java or C#—you're creating a **runtime object** stored in a variable with special properties (`Prototype` and `Call`) that make it behave like a class factory. This prototype-based architecture explains every confusing behavior: why `this.staticVar++` creates instance properties instead of modifying statics, why circular references leak memory permanently, and why composition often beats inheritance in AHK2.

## The Mind-Bending Reveal

### What You Probably Think

- "AHK2 has classes similar to Java/C#"
- "Classes are compile-time blueprints"
- "`class MyClass {}` defines a template"
- "Static members are truly separate from instances"

### The Reality

- AHK2 uses **JavaScript-style prototypes**
- Classes are **runtime objects** created at script load
- `class MyClass {}` creates an object immediately
- Static members live on the class object, instances access them through lookup

**Direct quote from Lexikos (AHK creator):**

> "Prototype-based model: Every object inherits properties and methods from its prototype or base object."

The `class` keyword is **syntactic sugar** that constructs a prototype object automatically. Everything you can do with classes, you can do with plain objects and prototype chains—classes just make it cleaner.

## What Actually Happens: `obj := MyClass()`

When you execute `obj := MyClass()`, you're calling `MyClass.Call()`, which runs an instance factory:

```
MyClass.Call()
   ↓
Create new object: {base: MyClass.Prototype}
   ↓
Call __Init (initialize instance variables from prototype)
   ↓
Call __New (your constructor if defined)
   ↓
Return instance
```

The resulting object doesn't "have" methods—it **inherits** them by following `base` references through the prototype chain until finding what it needs.

## Memory Layout: The Two-Object System

Every class definition creates **two separate objects**:

```
[Class Object: Counter]
├─ total: 0 (static property stored here)
├─ Prototype → [Prototype Object]
│              ├─ count: 0 (default value)
│              ├─ __New: <Function>
│              └─ base → Object.Prototype
└─ Call: <InstanceFactory>

[Instance: obj1]
├─ count: 1 (own property, set in constructor)
└─ base → Counter.Prototype

[Instance: obj2]
├─ count: 2
└─ base → Counter.Prototype
```

**Critical distinction:**
- **Class Object** (stored in `Counter`): Contains static members
- **Prototype Object** (at `Counter.Prototype`): Contains instance methods
- **Instances** get `{base: Counter.Prototype}` pointing to prototype, not class

## Static vs. Instance: The Real Mental Model

### The Dangerous Pattern

```ahk
class Counter {
    static total := 0
    count := 0

    __New() {
        this.total++  // Creates instance property, doesn't modify static!
    }
}

c1 := Counter()
c2 := Counter()
c3 := Counter()

MsgBox Counter.total  ; 0 (static unchanged)
MsgBox c1.total       ; 1 (instance property)
MsgBox c2.total       ; 1 (instance property)
```

### What's Happening Under the Hood

**Step-by-step execution of `this.total++` in constructor:**

1. Lookup `total` through prototype chain
2. Find `Counter.total` (static) with value 0
3. Read value: 0
4. Increment: 0 + 1 = 1
5. **Create NEW instance property** `this.total = 1`
6. Static `Counter.total` remains 0

**Why this happens:** Reading `this.total` walks the chain successfully (finds static). But **assignment creates a property on `this`**, not modifying what was found in the chain.

### The Correct Pattern

```ahk
class Counter {
    static total := 0
    count := 0

    __New() {
        Counter.total++           // ✓ Modifies static on class object
        this.count := Counter.total  // ✓ Instance gets correct value
    }

    GetTotal() {
        return Counter.total      // ✓ Always read statics directly
        // return this.total      // ⚠️ Could read stale instance property
    }
}
```

**Rule:** **Always use `ClassName.staticMember` when reading or writing statics**, especially in constructors.

### The Asymmetry

**Reading (walks chain):**
```ahk
this.property → check this → check this.base → check this.base.base → ...
```

**Writing (creates local):**
```ahk
this.property := value → create/modify property on this
```

This asymmetry between reading and writing explains most prototype-related bugs.

## Garbage Collection: Reference Counting Without Cycle Detection

AHK2 uses **reference counting**, not mark-and-sweep. When an object's reference count reaches zero, it's immediately freed and `__Delete` is called.

**The problem:** Circular references leak memory permanently.

**Quote from Lexikos:**

> "Resolving circular references would take far more work than I'm willing to put in any time soon."

### Circular Reference Example

```ahk
while true {
    a := {}
    b := {a: a}
    a.b := b  // Circular reference created
    // Objects NEVER freed even when loop continues
}
```

Each object holds a reference to the other. Neither reference count ever reaches zero. Memory leaks until script exit.

### Manual Cleanup Required

```ahk
x := {}, y := {}
x.child := y
y.parent := x

// Before letting them go out of scope:
y.parent := ""  // Break the cycle
x := ""
y := ""  // Now both can be freed
```

### Cleanup in Destructors

```ahk
class ParentWidget {
    __New() {
        this.children := []
    }

    AddChild(child) {
        this.children.Push(child)
        child.parent := this  // Creates potential cycle
    }

    __Delete() {
        // Break cycles before destruction
        for child in this.children
            child.parent := ""
        this.children := []
    }
}
```

**Memory overhead:** 16 bytes per property (key-value pair). An object with 20 properties has 320 bytes overhead beyond actual data.

## Inheritance vs. Composition

**Community consensus:** Favor composition over inheritance.

Multiple experienced developers report that inheritance in AHK2 leads to tight coupling, fragile base classes, and unexpected behavior. The language lacks true polymorphism support.

### Inheritance Fails for Built-in Classes

```ahk
class ListViewExtensions extends Gui.ListView  // WRONG - fails
```

**Problem:** You cannot instantiate `Gui.ListView` directly—it must be created through `Gui.Add()`. Extending it creates unusable classes.

### Composition Solution

```ahk
class MyListViewManager {
    __New(guiObj) {
        this.lv := guiObj.Add("ListView", "w500 h300", ["Name", "Value"])
        this.lv.OnEvent("Click", this.OnClick.Bind(this))
    }

    OnClick(ctrl, info) {
        // Custom behavior
    }

    AddRow(data) {
        this.lv.Add("", data*)
    }
}
```

**Why composition wins:**
- No tight coupling to parent implementation
- Flexibility to change underlying objects
- Clearer dependencies
- No inheritance hierarchy depth issues

### When Inheritance Works

Clear "is-a" relationships with minimal specialization:

```ahk
class Character {
    __New(name, health) {
        this.name := name
        this.health := health
    }

    TakeDamage(amount) {
        this.health -= amount
    }
}

class Wizard extends Character {
    __New(name, health, mana) {
        super.__New(name, health)  // Must call parent constructor
        this.mana := mana
    }

    CastSpell(cost) {
        if (this.mana >= cost) {
            this.mana -= cost
            return true
        }
        return false
    }
}
```

**Guideline:** 2-3 levels deep maximum. Beyond that, use composition.

## Common Pitfalls

### Pitfall 1: Forgetting `super.__New()`

```ahk
class PopLayers extends PopupWin {
    __New() {
        // Properties defined in parent __New not available!
        // this.found is undefined
        this.layers := []
    }
}
```

**Problem:** Parent constructor defines `this.found := false` in `__New()`, but child's `__New()` never calls parent.

**Solution:**
```ahk
class PopLayers extends PopupWin {
    __New() {
        super.__New()  // Call parent constructor first
        this.layers := []
    }
}
```

**Alternative:** Move property definitions to class body (inherited automatically):
```ahk
class PopupWin {
    found := false  // Defined in class body, not constructor
}
```

### Pitfall 2: Infinite Recursion with Property Getters

```ahk
property {
    set => this.property := value  // INFINITE RECURSION
    get => this.property           // INFINITE RECURSION
}
```

**Solution:**
```ahk
property {
    set => this._property := value  // Different name
    get => this._property
}
```

See docs/manuals/ahk2/explanation/property-vs-field.md for complete explanation.

### Pitfall 3: Circular References in COM Objects

Real-world example from GitHub issue:

```ahk
ComObjConnect(WB, WB_events)  // Creates circular reference
```

The `ComObjConnect` creates a cycle that crashed the debugger. While this was fixed in the engine, it highlights that circular references cause real problems even in professional code.

### Pitfall 4: Over-Engineering Simple Tasks

Forum discussion where user asked about creating Employee class for data:

**Community response:**

> "Why use a class? A simple object `{age: 22, name: 'Timothy'}` works fine unless you need methods or multiple instances sharing behavior."

Classes should solve actual complexity, not demonstrate programming knowledge.

### Pitfall 5: Assuming Classes Work Like Java/C#

The most fundamental misunderstanding:
- Static members are NOT truly separate
- Constructors don't always run (can return early)
- Inheritance behaves differently
- No compile-time type checking

**Mental shift required:** Think JavaScript prototypes, not Java classes.

## Prototype Chain Visualization

### Property Lookup

```ahk
obj1.count
   ↓
Check obj1 for "count" property
   ↓ Found: count = 1
Return 1

obj1.__New
   ↓
Check obj1 for "__New"
   ↓ Not found
Check obj1.base (Counter.Prototype) for "__New"
   ↓ Found: __New function
Return function reference
```

### Static Access Through Instance

```ahk
obj1.total++
   ↓
Lookup "total":
   Check obj1 → Not found
   Check obj1.base (Counter.Prototype) → Not found
   Check Counter (class object) → Found: total = 0
   ↓
Read value: 0
Increment: 0 + 1 = 1
   ↓
Create NEW PROPERTY obj1.total = 1
(Counter.total remains 0)
```

**Key insight:** Assignment creates property on receiving object, not modifying what was found in lookup.

## Performance Realities

**Class vs. function overhead:** Negligible for typical automation.

Method calls have additional indirection (prototype chain lookup), but we're talking microseconds.

### When Performance Matters

- Tight loops iterating 100,000+ times
- Real-time input processing at sub-50ms intervals
- Large data structure operations with 10,000+ objects
- Timer callbacks at intervals below 50ms

### When It Doesn't Matter (99% of Use Cases)

- GUI automation responding to human input
- Scripts running fewer than 1000 operations
- Standard business logic and workflows

### Critical Optimization

```ahk
SetBatchLines -1  // Remove artificial delays - MOST IMPORTANT
```

This single line has more impact than any class-versus-function decision.

### Memory Overhead

**Per-object:** 16 bytes per property minimum. An object with 10 properties uses at least 160 bytes overhead plus actual data.

For typical scripts with dozens/hundreds of objects: negligible. For tight loops creating thousands of temporary objects per second: measurable.

## When to Use Classes

### Decision Framework

**Use plain functions when:**
- 1-2 toggles total
- No shared state between functions
- Scripts under 100 lines
- One-off utility scripts
- Learning AHK basics

**Use static class methods when:**
- 2-4 similar functions
- Need simple namespace grouping
- Want basic encapsulation without instances

**Use static manager pattern when:**
- 5+ similar functions with shared behavior
- Need error handling and diagnostics
- Want maintainable code over time
- Multiple developers

**Use instance-based classes when:**
- Need multiple independent sets of objects
- Different contexts (separate windows/menus)
- Complex per-instance state
- True object-oriented modeling

### Brutal Honesty: When NOT to Use Classes

- Scripts under 50 lines
- One-off personal automation
- Learning phase (master functions first)
- Single toggles that don't need coordination
- Any case where simple object literal `{key: value}` suffices

**Community consensus:** "Most AHK users get along fine without ever using classes."

Classes become valuable at the complexity level where managing global state becomes painful.

## Key Insights

**Insight 1: Classes are syntactic sugar over prototypes**
Everything with `class MyClass {}` could be done manually with object literals and base references.

**Insight 2: The prototype chain resolves ALL member access**
`obj.method()` walks `obj → obj.base → obj.base.base → ...` until finding `method`.

**Insight 3: Assignment creates properties on the receiving object**
`this.property := value` creates/modifies a property on `this`, not walking the chain.

**Insight 4: Reference counting without cycle detection is leaky**
Long-running scripts with temporary object graphs need manual cycle breaking.

**Insight 5: Composition beats inheritance without polymorphism**
AHK2's prototype system simulates inheritance, but without interfaces/abstract methods, inheritance becomes brittle.

**Insight 6: Static access requires ClassName.member**
Never use `this.static` in constructors. Always `ClassName.static`.

**Insight 7: Classes add value at 5+ related functions**
Below this threshold, overhead exceeds benefits. Above it, centralized control improves maintainability.

## The Prototype Truth

Understanding that classes are runtime objects with prototype chains explains:

- Why `class MyClass {}` executes at script load (creates object)
- Why `MyClass.staticVar` and `instance.staticVar` can differ (shadowing)
- Why static members live on class object, not prototype
- Why circular references leak (reference counting can't detect cycles)
- Why composition often wins (no language-level polymorphism support)

The mental model shift: **Classes aren't blueprints—they're factory objects with prototype chains for sharing behavior.**

## Cross-References

Related topics:
- docs/manuals/ahk2/explanation/property-vs-field.md - Property mechanics
- docs/manuals/ahk2/how-to/class-patterns.md - Practical class usage
- docs/manuals/ahk2/reference/base-object.md - Prototype chain reference
- docs/manuals/ahk2/explanation/closures-deep-dive.md - Scope and lifetime
