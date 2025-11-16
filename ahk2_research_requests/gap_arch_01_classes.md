# AHK2 Research Request: Class-Based Architecture & OOP

## What I Need to Learn

I have a codebase with 50+ scattered global variables and 7 identical toggle functions. I suspect classes could eliminate this duplication, but I don't deeply understand AHK2's object model. I need to know **how AHK2 implements OOP** (prototype-based? class-based?) and when using classes actually makes sense vs when it's over-engineering.

## My Current Understanding (Challenge This!)

I believe:
- AHK2 has classes similar to Java/C#
- Classes are compile-time blueprints for objects
- Static members are like C# statics (class-scoped, not inherited)
- Classes provide "true" encapsulation

**I suspect some of this is wrong.** Tell me where.

## Specific Questions

1. **Fundamental Model**: Is AHK2 class-based or prototype-based? What's actually happening when I write `obj := MyClass()`?

2. **Static vs Instance**: I have code like this:
```ahk
class Counter {
    static total := 0
    count := 0
    __New() {
        this.total++  // Does this work? Or do I need Counter.total++?
    }
}
```
What's the RIGHT mental model here?

3. **Garbage Collection**: How does AHK2 decide when to destroy objects? I see reference counting mentioned - what about circular references?

4. **Inheritance Patterns**: When should I use `extends` vs composition? Show me examples where inheritance is WRONG.

5. **Performance Reality**: What's the actual overhead of classes vs global functions? Should I care?

## What I'm Building

Transform this code pattern (repeated 7 times):
```ahk
global featureEnabled := true

ToggleFeature(*) {
    global featureEnabled
    featureEnabled := !featureEnabled
    if featureEnabled
        A_TrayMenu.Check "Feature Name"
    else
        A_TrayMenu.Uncheck "Feature Name"
}
```

Into something like this (single implementation):
```ahk
class FeatureManager {
    // How do I design this elegantly?
    // How do I avoid 7 repeated functions?
}
```

Show me the evolution: Naive → Better → Best

## How to Write This Guide

### 1. Start with "The Reveal"
Begin with something that shatters my assumptions. Example:
> "You're thinking classes are blueprints. Wrong. In AHK2, `class MyClass` creates a RUNTIME OBJECT immediately. Classes are objects."

### 2. Progressive Understanding (Not Just Examples)
Don't just show beginner → advanced CODE. Show beginner → advanced UNDERSTANDING:
- **Naive understanding**: "Classes are like C# classes"
- **Better understanding**: "Actually, they're prototype-based like JavaScript"
- **Deep understanding**: "Classes are syntactic sugar over `{base: Class.Prototype}` objects"

### 3. Side-by-Side Misconception Corrections
```ahk
// ❌ What I THINK this does:
Counter.total++  // Accesses class variable

// ✓ What it ACTUALLY does:
Counter.total++  // Walks prototype chain, creates own property if assigned

// 🧠 The insight:
// Writes create own properties, reads walk the chain
```

### 4. Real-World Patterns, Not Toy Code
Instead of:
```ahk
class Animal {
    Speak() => "Sound"
}
```

Show me:
```ahk
// Managing 7 toggle features with single class
// Eliminating 100+ lines of duplicated code
// Making it extensible for future features
```

### 5. "Aha Moments" Section
Include a section called **"The Moment It Clicks"** with the key insights that make everything fall into place:
- "When I realized `MyClass()` is just calling `MyClass.Call()`, everything made sense"
- "The breakthrough was understanding that static members are on the class object, instance members on the prototype"

### 6. When NOT to Use This
Be brutally honest:
- "Classes add overhead for simple scripts"
- "If you have <3 related functions, just use globals"
- "Don't class-ify everything just because you can"

### 7. Performance Reality Check
Give me actual numbers:
- "10,000 method calls: 10.2ms (class) vs 9.8ms (function) = 0.4ms difference"
- "Conclusion: Optimize algorithm, not object model"

### 8. Common Pitfalls from Experience
Not "common mistakes" but **"I wasted 2 hours on this"**:
- "I tried to use `this.staticVar` and wondered why all instances shared it"
- "Circular references aren't auto-detected - I leaked memory for weeks"

### 9. Decision Tree
End with a flowchart:
```
Need to manage state across operations?
├─ Yes → Consider classes
    ├─ >3 related functions? → Probably yes
    └─ <3 functions? → Probably just use functions
└─ No → Use pure functions
```

### 10. The Complete Transform
Show the FULL evolution of my real code:
```ahk
// Before (7 identical functions, 100+ lines)
[actual verbose code]

// After - Attempt 1 (why this fails)
[class with problems]

// After - Attempt 2 (better, but still issues)
[improved class]

// After - Final (production-ready)
[complete, working solution with error handling]
```

## What Would Make This Guide EXCEPTIONAL

1. **Challenge my assumptions directly**: "You mentioned you think X. That's wrong because..."

2. **Show the debugging process**: How to figure out what's happening when classes don't work as expected

3. **Memory visualization**: Diagrams showing object memory layout, prototype chains

4. **Comparison to other languages**: "If you know JavaScript, this is similar to... BUT different because..."

5. **Future-proofing**: "You can't do X today, but you can structure code to make X possible later"

6. **Anti-patterns from real code**: "I see people do X all the time. It works until it doesn't because..."

## Success Criteria

After reading this guide, I should be able to:
1. ✓ Refactor my 7 toggle functions into a single elegant class
2. ✓ Explain why `this.staticVar++` vs `ClassName.staticVar++` behave differently
3. ✓ Decide immediately if a new feature needs a class or not
4. ✓ Avoid circular reference memory leaks
5. ✓ Debug prototype chain issues
6. ✓ Teach someone else how AHK2's object model differs from Java/C#

## Output Format

```markdown
# Class-Based Architecture in AHK2: The Complete Guide

## The Reveal: What Classes Actually Are
[Mind-bending truth that corrects assumptions]

## Part 1: Fundamental Understanding
### The Prototype Model (Not Class Model)
### How Object Creation Really Works
### Static vs Instance - The Mental Model

## Part 2: Progressive Refactoring
### Your Code: The Problem
### Attempt 1: Naive Class (Why It Fails)
### Attempt 2: Better Design (Still Has Issues)
### Final: Production-Ready Pattern

## Part 3: Deep Dive
### Lifecycle & Garbage Collection
### Inheritance vs Composition (When Each)
### Memory Layout & Performance

## Part 4: Mastery
### Design Patterns in AHK2
### Common Pitfalls (With War Stories)
### Decision Framework

## The Aha Moments
[The insights that make it all click]

## Quick Reference
[Cheat sheet for daily use]
```

## Final Note

**Don't just teach me syntax - teach me thinking.** I want to internalize how AHK2's object system works so deeply that I can solve problems I've never seen before.

Show me the **mental shift** from "classes as blueprints" to "classes as prototype-based runtime objects."
