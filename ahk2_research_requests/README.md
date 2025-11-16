# AHK2 Research Requests - README

## What This Is

A collection of 45 focused research requests for mastering AutoHotkey v2.0. Each request is a standalone document asking for deep understanding of a specific topic, designed to enable **transfer learning** rather than memorization.

## Current Status: 15/45 Created (33%)

### ✅ Completed (15 detailed requests)

**Architecture & Patterns (5):**
- gap_arch_01_classes.md - Class-Based Architecture & OOP
- gap_arch_02_properties_encapsulation.md - Properties, Getters, Setters
- gap_arch_04_error_handling.md - Error Handling Architecture
- gap_arch_05_events.md - Event-Driven Architecture
- gap_arch_06_timers.md - Timers & Async Patterns
- gap_arch_07_file_io_persistence.md - File I/O & Persistence
- gap_arch_13_gui.md - GUI Advanced Patterns

**System Integration (4):**
- gap_sys_01_window_messages.md - Window Messages & Hooks
- gap_sys_02_input.md - Keyboard & Mouse Input Internals
- gap_sys_07_graphics.md - Graphics, Drawing & GDI+
- gap_sys_10_dllcall.md - DllCall & Windows APIs
- gap_sys_13_security.md - Security, Permissions & UAC

**Language Mastery (3):**
- gap_lang_01_type_system.md - Type System & Dynamic Typing
- gap_lang_06_closures.md - Closures & Function Scope
- gap_lang_07_memory.md - Memory Management & GC

### ⏳ Remaining (30 to create)

See INDEX.md for complete list with priorities.

---

## What Makes These Different

### Traditional Tutorial:
> "Here's how to create a class: `class MyClass { }`"

### These Requests:
> **"I think classes are blueprints like C#. Challenge this assumption. Show me what they really are, why AHK2 designed them that way, and how understanding the prototype model unlocks new patterns. Include 3 bugs I'll create from not understanding this, and show how experienced devs think about classes differently."**

### Key Differences:

1. **Challenges assumptions** - "You think X. Wrong. Here's why..."
2. **Shows evolution** - Naive → Better → Deep understanding (not just code progression)
3. **Real code transforms** - Takes actual code from project, refactors with new knowledge
4. **War stories** - "I wasted 2 hours because I didn't know X"
5. **Mental models** - Diagrams, analogies, conceptual frameworks
6. **Decision frameworks** - When to use vs when NOT to use
7. **Performance reality** - Actual benchmarks, not assumptions
8. **Complete examples** - Production-ready code, not toys

---

## How to Use These

### For Learning (Individual)

**Phase 1 - Foundation (Start here):**
1. gap_arch_01_classes.md
2. gap_lang_01_type_system.md
3. gap_arch_04_error_handling.md
4. gap_arch_07_file_io_persistence.md

**Impact**: Fix immediate code quality issues, eliminate duplication, add persistence.

**Phase 2 - System Integration:**
5. gap_sys_10_dllcall.md
6. gap_sys_01_window_messages.md
7. gap_sys_02_input.md
8. gap_sys_13_security.md

**Impact**: System-level automation capabilities.

**Phase 3 - Advanced Patterns:**
9. gap_arch_05_events.md
10. gap_lang_06_closures.md
11. gap_arch_06_timers.md
12. gap_lang_07_memory.md

**Impact**: Elegant, maintainable code.

### For Research (Your Friend)

1. Pick a gap file (start with Phase 1)
2. Research the topic deeply (docs, forums, experiments)
3. Write guide following the "How to Write This Guide" section
4. Focus on mental models and "aha moments"
5. Include war stories and gotchas from experience
6. Place completed guide in `ahk2_research_requests/guides/`

---

## Writing Style That Works Best

Based on analysis of partial guides received, the most effective approach is:

### ✅ DO THIS:

**1. Challenge assumptions immediately**
```markdown
## The Reveal

You think classes are compile-time blueprints.

Wrong.

In AHK2, `class MyClass` creates a RUNTIME OBJECT immediately.
Classes ARE objects, not templates for objects.

This changes everything...
```

**2. Show the mental shift**
```markdown
Level 1: "Properties are fields with special behavior"
Level 2: "Properties are methods disguised as fields"
Level 3: "Properties are hooks in property resolution"
Level 4: "I can intercept ALL property access with __Get/__Set"

[Explain what triggers each level of understanding]
```

**3. Include "Why I care" section**
```markdown
## Why This Matters (Not Academic)

- Eliminates 100+ lines of duplicate code in YOUR project
- Fixes the "all instances share staticVar" bug you WILL hit
- Enables patterns you can't currently imagine
```

**4. Real code transformations**
```ahk
// BEFORE (from actual AMovementSuite code):
[paste 30 lines of repetitive code]

// AFTER - Attempt 1 (what beginners try):
[show common mistakes]
// Problems: [list specific issues]

// AFTER - Attempt 2 (better, but still issues):
[show improvement]
// Remaining issues: [list]

// AFTER - Final (production-ready):
[complete, robust solution]
// Why this is correct: [explain design choices]
```

**5. Gotchas as war stories**
```markdown
### "I Wasted 2 Hours On This"

**The Infinite Recursion Horror**
> I wrote `property { get => this.property }` and my script froze.
> Took me forever to realize the getter was calling itself.
> ALWAYS use different names: `property { get => this._property }`
```

**6. Decision frameworks (visual)**
```markdown
Need to manage state across operations?
├─ Yes → Consider classes
│   ├─ >3 related functions? → Classes
│   ├─ 1-2 functions? → Just functions
│   └─ Need multiple instances? → Definitely classes
└─ No → Pure functions
```

### ❌ DON'T DO THIS:

**1. Just list syntax**
```markdown
## Syntax
class MyClass { }
```

**2. Toy examples**
```ahk
class Animal {
    Speak() => "Sound"
}
```

**3. Assume knowledge**
```markdown
"Use the prototype pattern..."  // What IS the prototype pattern?
```

**4. Skip the "why"**
```markdown
"Use properties instead of direct access."  // WHY?
```

**5. Incomplete examples**
```ahk
class Example {
    // ... rest of implementation
}
// Don't show "..." - show COMPLETE working code
```

---

## What Makes a Great Guide

### The "Aha Moment" Test

After reading, the learner should have specific moments where they think:
- "OH! That's why my code didn't work!"
- "Now I understand what I was doing wrong!"
- "This completely changes how I think about X!"
- "I can't believe I didn't know this before!"

### The "Transfer" Test

The learner should be able to:
- Apply knowledge to problems not shown in guide
- Explain concept to someone else
- Debug issues using mental model
- Choose appropriate pattern for new situations

### The "War Story" Test

The guide should include real experiences:
- "I made this mistake and here's what happened..."
- "This pattern looks clever but breaks because..."
- "You'll be tempted to do X. Don't. Here's why..."

---

## Examples of Excellence

### Great Opening (Challenges Assumptions)

```markdown
# Window Messages: How Windows Really Works

You're polling mouse position in a loop.

This is like checking your mailbox every 5 seconds instead of waiting for the mail truck.

Windows is MESSAGE-DRIVEN. Every click, keystroke, and window movement is a message. You're working against the OS instead of with it.

Let me show you the right way...
```

### Great Example Progression

```markdown
## Progressive Implementation

**Attempt 1: What beginners try**
[code that looks reasonable but has subtle bugs]

**Why this fails:**
- [Specific problem 1]
- [Specific problem 2]

**Attempt 2: Better, but still issues**
[improved code addressing problems]

**Remaining issues:**
- [What's still wrong]

**Final: Production-ready**
[Complete, robust solution]

**Why this is correct:**
- [Design choice 1 explained]
- [Design choice 2 explained]
```

### Great Gotcha Format

```markdown
## Common Pitfalls (From Battle Scars)

**"All my closures showed the same value!"**

The code:
[exact code that causes problem]

What I expected:
[what beginners think happens]

What actually happens:
[what really occurs]

Why:
[mechanism explanation]

The fix:
[correct code]

How to never make this mistake again:
[mental model that prevents it]
```

---

## Generating the Remaining 30 Gaps

### Option 1: Batch Generate Now
Create all 30 remaining gaps using template with topic-specific content. Less detail for low-priority gaps, full detail for high-priority.

### Option 2: Generate Phase by Phase
- Create next 10 (Phase 2: System Integration)
- Wait for feedback
- Create final 20 (Phase 3: Advanced Topics)

### Option 3: On-Demand Generation
- Generate when needed for specific research
- Ensures maximum detail and relevance
- Avoids creating gaps never used

## Recommendation

**Start research on the 15 created gaps** before generating all 45.

Why:
- These 15 are the MOST critical (would immediately improve codebase)
- Feedback on guide quality can refine approach for remaining 30
- May discover some gaps aren't needed or should be combined
- May identify NEW gaps not in original list

## Next Steps

1. Review INDEX.md for gap priorities
2. Choose learning phase (1, 2, or 3)
3. Send gap file(s) to researcher
4. When guides complete, apply knowledge to codebase
5. Generate more gap files as needed

---

## File Structure

```
ahk2_research_requests/
├── INDEX.md                          # Complete gap catalog with priorities
├── README.md                         # This file
├── GENERATE_REMAINING.md             # Template and generation strategy
├── gap_arch_01_classes.md            # ✓ Created
├── gap_arch_02_properties.md         # ✓ Created
├── ... (15 created)
├── gap_arch_03_functions.md          # ⏳ To create (30 remaining)
├── ...
└── guides/                           # Completed guides go here
    ├── gap_arch_01_classes_GUIDE.md
    └── ...
```

---

## Success Metrics

**For Individual Gaps:**
- Can apply knowledge to code immediately
- Eliminates specific bugs/duplication
- Unlocks new capabilities

**For Complete Set:**
- Comprehensive AHK2 mastery
- Can solve any automation problem
- Write elegant, maintainable code
- Understand language and OS deeply

---

**The 15 created gaps represent the highest-impact learning** - they would transform the codebase quality immediately. The remaining 30 add breadth and specialized capabilities.
