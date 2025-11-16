# AHK2 Research Request: Properties, Getters, Setters & Encapsulation

## What I Need to Learn

I have objects where properties are accessed directly everywhere - no validation, no computed values, no side effects. I need to understand AHK2's property system deeply enough to know when properties vs methods make sense and how to implement validation without boilerplate.

## My Current Understanding (Challenge This!)

I believe:
- Properties with get/set are like C# properties
- They're just syntactic sugar for methods
- There's a performance penalty vs direct access
- You can't have side effects in getters (bad practice)

**Correct me where I'm wrong.**

## Specific Questions

1. **What ARE Properties?**: Are they variables with hooks? Method pairs? Something else?

2. **Infinite Recursion**: I see warnings about this - why does this happen?
```ahk
property {
    get => this.property  // Infinite loop?
}
```
Explain the mechanism that causes this.

3. **When to Cache**: I want computed properties (area from width*height). When should I cache vs recalculate on each access?

4. **Read-Only Properties**: What's the RIGHT way to make properties read-only? Empty setter? Throw error? Something else?

5. **Validation**: Should validation go in setters or methods? Show patterns for both.

## What I'm Building

Transform direct access:
```ahk
global enableWindowSnapping := false
// Later anywhere:
enableWindowSnapping := true  // No validation, no notifications
```

Into validated, observable properties:
```ahk
class SnapConfig {
    // How to add validation (must be boolean)?
    // How to trigger events when changed?
    // How to make read-only from outside but writable internally?
}
```

## How to Write This Guide

### 1. Start with "What vs How"
```ahk
// WHAT I think properties are:
"Variables with special behavior"

// WHAT they actually are:
"Method pairs called transparently"

// Show me the actual implementation:
obj.x     => obj.__Get("x")
obj.x := 5 => obj.__Set("x", [], 5)
```

### 2. The Evolution of Understanding
```markdown
**Level 1**: "Properties are fields with get/set"
**Level 2**: "Properties are methods disguised as fields"
**Level 3**: "Properties are hooks in the property resolution chain"
**Level 4**: "I can intercept ALL property access with __Get/__Set"
```

### 3. Build Complexity Gradually
Start simple:
```ahk
age {
    get => this._age
    set => this._age := value
}
```

Add validation:
```ahk
age {
    get => this._age
    set {
        if value < 0
            throw ValueError("Age cannot be negative")
        this._age := value
    }
}
```

Add caching:
```ahk
area {
    get {
        if !this._areaValid {
            this._cachedArea := this.width * this.height
            this._areaValid := true
        }
        return this._cachedArea
    }
}
```

Add notifications:
```ahk
width {
    get => this._width
    set {
        this._width := value
        this._areaValid := false  // Invalidate cache
        this._NotifyObservers("width", value)
    }
}
```

### 4. Anti-Patterns from Real Code
Show me code that LOOKS good but has problems:
```ahk
// Looks fine, but has a subtle bug:
class BadExample {
    property {
        get => this._property
        set {
            if (value > 100)
                return  // Returning without setting - silent failure!
            this._property := value
        }
    }
}

// Better:
set {
    if (value > 100)
        throw ValueError("Too large")
    this._property := value
}
```

### 5. Decision Matrix
Create a table:
| Scenario | Use Property | Use Method | Why |
|----------|--------------|------------|-----|
| Simple value access | ✓ | | Feels like data |
| Validated value | ✓ | | Property with setter |
| Expensive computation | | ✓ | Method signals cost |
| Action with side effects | | ✓ | Methods are verbs |
| State query | ✓ | | Property for questions |

### 6. Performance Comparison
Benchmark and show results:
```ahk
// Test 1 million accesses:
Direct access: 45ms
Value property: 48ms  (+7%)
Getter property: 125ms (+178%)
Method call: 130ms (+189%)

Conclusion: Properties 2-3x slower than direct, but still microseconds.
Only avoid in hot loops (>100k iterations).
```

### 7. The Transform (My Actual Code)
Take my real scenario:
- 7 features, each with `featureEnabled` global
- Each has identical toggle function
- No validation, no persistence

Show THREE complete solutions:
1. **Minimal class** (just eliminates duplication)
2. **Validated class** (adds validation)
3. **Observable class** (adds event notifications + persistence)

Explain trade-offs of each.

### 8. Gotchas Section
Title: **"I Wasted Hours On This"**

Real stories:
- "I forgot the underscore prefix and got infinite recursion"
- "I put validation in getter instead of setter - too late!"
- "I didn't invalidate cache when dependencies changed - stale data everywhere"
- "I used same name for property and backing field - silent corruption"

## Success Criteria

After this guide, I can:
1. ✓ Explain the difference between property and backing field
2. ✓ Write getter/setter without infinite recursion
3. ✓ Decide when to cache computed properties vs recalculate
4. ✓ Implement validation that fails fast with good errors
5. ✓ Create observable properties that notify on change
6. ✓ Refactor my 7 toggle functions into elegant property-based system

## Most Important

**Teach me the THINKING, not the syntax.**

I want to understand properties so deeply that when I see a problem, I instantly know:
- "This needs a property because..."
- "This needs a method because..."
- "This needs validation in the setter because..."

Not from memorizing rules, but from internalizing the mental model.
