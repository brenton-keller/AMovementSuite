# AutoHotkey v2 Manual

> **Optimized for Claude Code** - Use `docs/manuals/ahk2/` syntax to reference these docs in conversations

This manual consolidates verbose research reports into navigable, comprehensive documentation following the Diátaxis framework.

## Quick Navigation by Need

### "I'm new to AutoHotkey"
Start here: docs/manuals/ahk2/tutorials/01-getting-started.md

### "I need to accomplish a specific task"
Browse: docs/manuals/ahk2/how-to/

### "I need to look up exact syntax or parameters"
Search: docs/manuals/ahk2/reference/

### "I want to understand how something works"
Read: docs/manuals/ahk2/explanation/

---

## Tutorials (Learning-Oriented)

Step-by-step lessons for progressive skill building.

### docs/manuals/ahk2/tutorials/01-getting-started.md
Your first steps: installation, hello world, hotkeys, hotstrings, basic GUI, variables, loops, functions
- **Time:** 30-45 minutes
- **Prerequisites:** None
- **You'll learn:** Core AHK2 syntax, create practical scripts

### docs/manuals/ahk2/tutorials/02-intermediate-concepts.md
Objects, classes, closures, error handling
- **Time:** 60-90 minutes
- **Prerequisites:** Completion of getting started tutorial
- **You'll learn:** Object-oriented programming, advanced patterns

---

## How-To Guides (Task-Oriented)

Solve specific problems with proven solutions.

### Closures & Functions

#### docs/manuals/ahk2/how-to/closure-loop-pattern.md
**Problem:** Closures in loops capture the same variable
**Solutions:** Local variable capture, default parameters, .Bind() method
**When:** Creating event handlers or callbacks in loops

### Memory Management

#### docs/manuals/ahk2/how-to/prevent-memory-leaks.md
**Problem:** Circular references and event handlers cause memory leaks
**Solutions:** Breaking cycles in __Delete, unsubscribing from events, SetTimer "Delete"
**When:** Long-running scripts, event-driven code, GUI applications

#### docs/manuals/ahk2/how-to/debug-memory-leaks.md
**Problem:** Need to detect and diagnose memory leaks
**Solution:** GetProcessMemoryInfo monitoring, stress testing, leak detection framework
**When:** Troubleshooting memory growth, performance issues

### Error Handling

#### docs/manuals/ahk2/how-to/custom-error-handler.md
**Problem:** Scattered try/catch blocks, inconsistent logging
**Solution:** Centralized ErrorHandler class with OnError
**When:** Production applications, debugging user issues

#### docs/manuals/ahk2/how-to/retry-network-operations.md
**Problem:** Transient network failures crash script
**Solution:** Exponential backoff with jitter, conditional retry
**When:** File downloads, API calls, database connections

#### docs/manuals/ahk2/how-to/circuit-breaker-pattern.md
**Problem:** Script hammers failing service, cascading failures
**Solution:** Circuit breaker (CLOSED → OPEN → HALF_OPEN states)
**When:** External service calls, preventing cascading failures

### Windows & Messages

#### docs/manuals/ahk2/how-to/window-dragging.md
**Problem:** Need custom drag regions or frameless window dragging
**Solution:** WM_NCHITTEST handler returning HTCAPTION
**When:** Modern UI, custom title bars, drag-anywhere windows

#### docs/manuals/ahk2/how-to/save-restore-positions.md
**Problem:** Users lose window position on restart
**Solution:** WM_CLOSE interception + INI persistence
**When:** GUI apps, multi-monitor setups, user preferences

#### docs/manuals/ahk2/how-to/detect-window-creation.md
**Problem:** Need to act when specific windows open
**Solution:** RegisterShellHookWindow or polling with SetTimer
**When:** Auto-configuration, window monitoring, automation triggers

### Type System

#### docs/manuals/ahk2/how-to/type-validation.md
**Problem:** Invalid data causes crashes, need type safety
**Solution:** IsNumber/IsInteger/IsFloat + ValidateNumber pattern
**When:** GUI input, config files, CSV processing, API data

### GUI & Graphics

#### docs/manuals/ahk2/how-to/responsive-gui-layouts.md
**Problem:** GUI doesn't resize properly or breaks at different sizes
**Solutions:** OnSize + Move(), anchor patterns, percentage layouts, GuiReSizer
**When:** Resizable windows, multi-monitor support, responsive design

#### docs/manuals/ahk2/how-to/reusable-gui-components.md
**Problem:** Duplicate GUI code across multiple windows
**Solutions:** Extend Gui class, fluent API, component wrappers, factory pattern
**When:** Building GUI frameworks, standardizing UI elements

#### docs/manuals/ahk2/how-to/gui-styling-theming.md
**Problem:** Need custom colors, fonts, dark mode
**Solutions:** BackColor/SetFont, dark mode DWM API, LV_Colors, GpGFX
**When:** Custom branding, dark mode support, themed applications

#### docs/manuals/ahk2/how-to/gui-mvc-pattern.md
**Problem:** Business logic mixed with GUI code
**Solution:** Model-View-Controller separation with observer pattern
**When:** Complex applications, testable code, team collaboration

### DllCall & Windows API

#### docs/manuals/ahk2/how-to/work-with-structs.md
**Problem:** Need to pass structures to Windows APIs
**Solutions:** RECT, POINT, SYSTEMTIME, MONITORINFO with NumPut/NumGet
**When:** Window positioning, time operations, monitor info, any struct-based API

#### docs/manuals/ahk2/how-to/dllcall-output-parameters.md
**Problem:** Functions that write to buffers or output parameters
**Solutions:** String buffers, numeric outputs, two-call patterns, VarRef &
**When:** Getting window titles, process IDs, file operations, any output data

#### docs/manuals/ahk2/how-to/debug-dllcall-errors.md
**Problem:** DllCall crashes or returns wrong results
**Solution:** 10-step systematic debugging process with gotcha checklist
**When:** Troubleshooting API calls, crashes, unexpected behavior

#### docs/manuals/ahk2/how-to/translate-msdn-to-dllcall.md
**Problem:** Need to implement Windows API from MSDN documentation
**Solution:** Type translation table, parameter annotation guide, step-by-step process
**When:** Implementing new APIs, reading Windows documentation

#### docs/manuals/ahk2/how-to/common-windows-apis.md
**Problem:** Need to accomplish common Windows API tasks
**Solutions:** Window manipulation, file I/O, monitor info, cursor position, window info
**When:** Extending AHK capabilities, precise control, functionality not in built-ins

---

## Reference (Information-Oriented)

Comprehensive lookup tables for when you know what you need.

### Language Core

#### docs/manuals/ahk2/reference/type-system.md
Complete type reference: primitives, checking, conversion, operators, coercion
- Primitive types (Integer, Float, String, Object)
- Type checking functions (`IsInteger`, `IsNumber`, `IsFloat`, `Type`)
- Type conversion (`Integer()`, `Float()`, `String()`)
- Operator reference (arithmetic, bitwise, comparison, logical)
- Type coercion decision tree
- Map key behavior
- Common pitfalls

### Architecture

#### docs/manuals/ahk2/reference/class-system.md
Class architecture: definition, instantiation, inheritance, built-in classes
- Prototype-based system
- Class definition syntax
- The two-object system
- Static vs instance members
- Inheritance with `extends`
- Built-in base classes (Object, Array, Map, Buffer, Func, Gui)
- Meta-functions (`__Get`, `__Set`, `__Call`, `__Init`, `__New`)
- Memory management
- Common patterns (Singleton, Factory, Manager)

#### docs/manuals/ahk2/reference/property-system.md
Property system: getters, setters, computed properties, validation
- Value properties vs dynamic properties
- Getter/setter syntax
- Backing field pattern
- Property resolution chain
- Validation patterns (type, range, constrained, regex)
- Computed properties with caching
- Observable properties
- Property vs method decision matrix

#### docs/manuals/ahk2/reference/error-types.md
Complete error hierarchy with all properties, when each is thrown
- Full error class hierarchy
- All error object properties (`Message`, `What`, `Extra`, `File`, `Line`, `Stack`, `Number`)
- Detailed reference for each type:
  - MemoryError, OSError, TimeoutError
  - TypeError, ValueError, IndexError, ZeroDivisionError
  - TargetError, UnsetError, UnsetItemError
  - MemberError, PropertyError, MethodError
- Creating custom errors
- Catching specific types
- Common mistakes

### System Integration

#### docs/manuals/ahk2/reference/window-messages.md
Windows messages: MSG structure, parameter decoding, common messages
- MSG structure anatomy
- Parameter decoding patterns
- Essential messages (WM_SIZE, WM_MOVE, WM_SYSCOMMAND, WM_CLOSE, WM_NCHITTEST)
- Mouse and keyboard messages
- SendMessage vs PostMessage
- Common patterns
- Performance considerations
- Debugging with Spy++

### Patterns

#### docs/manuals/ahk2/reference/closure-patterns.md
Catalog of closure patterns with use cases
- Counter (private state)
- Private state object (multiple methods)
- Configuration builder (validated state)
- Event handler with context
- Callback factory (parameterized closures)
- Lazy evaluation, Memoization, Partial application
- Loop closure (correct pattern)
- Resource manager, State machine, Observer pattern
- Performance characteristics
- Pattern selection guide

#### docs/manuals/ahk2/reference/memory-management.md
Complete memory management reference
- Reference counting rules (increment/decrement/no-change)
- Memory management functions (ObjPtr, ObjFromPtr, ObjAddRef, ObjRelease)
- GUI object lifetime behavior
- Closure capture rules
- Leak pattern catalog (10 common patterns)
- SetTimer memory behavior ("Off" vs "Delete" vs 0)
- Common pitfalls (wrong vs right code)
- Decision matrices (when to use each approach)

#### docs/manuals/ahk2/reference/gui-system.md
Complete GUI system reference
- Gui class methods, properties, options, events
- Control types catalog (25+ controls with options)
- Event parameters reference
- Move() method complete specification
- OnSize event reference
- Binding patterns comparison
- Community libraries catalog (GuiReSizer, Easy AutoGUI, WebViewToo, GpGFX, LV_Colors, UIA-v2)
- Color and font options reference
- Resize pattern decision matrix
- Component pattern comparison

#### docs/manuals/ahk2/reference/dllcall-type-system.md
Complete DllCall type reference
- All AHK types (Char, Int, Ptr, Float, Str, etc.) with sizes and ranges
- Windows API type translation (HWND→Ptr, DWORD→UInt complete mapping)
- Type prefix meanings (H, LP, P, C, W, A)
- String type comparison (Str vs WStr vs AStr vs Ptr)
- Output parameter notation (Type*, VarRef &)
- Common Windows API constants (SetWindowPos flags, error codes)
- Struct size quick reference (RECT, POINT, SYSTEMTIME, MONITORINFO)
- ErrorLevel and A_LastError value meanings
- Type decision matrix
- Common gotchas checklist

---

## Explanation (Understanding-Oriented)

Deep dives into concepts - answer "why" and "how it works."

### Language Internals

#### docs/manuals/ahk2/explanation/closures-deep-dive.md
**Understand:** How closures actually work in memory
- Free variable sets (heap-allocated shared state)
- Reference capture proof
- The loop problem explained step-by-step
- Memory management and leaks
- Scope rules (fat arrows vs regular functions)
- **Debunks:** "Closures capture by value", "Fat arrows are special"

#### docs/manuals/ahk2/explanation/type-coercion-logic.md
**Understand:** How the parser decides type conversions
- The core algorithm (math, comparison, concatenation)
- "Looks numeric" rule
- Comparison operators (numeric vs alphabetic)
- Division always returns float
- Logical operators return operands
- Boolean context truthiness
- **Debunks:** "= and == differ in coercion", "Logical operators return true/false"

### Architecture

#### docs/manuals/ahk2/explanation/class-system.md
**Understand:** Why AHK2 classes are prototype-based
- Classes as runtime objects
- The two-object system (Class Object + Prototype Object)
- Static vs instance (memory layout)
- Reference counting without cycle detection
- Inheritance vs composition
- **Debunks:** "Classes are compile-time blueprints", "Static members are separate"

#### docs/manuals/ahk2/explanation/property-vs-field.md
**Understand:** Why properties exist - method dispatch, not variables
- Properties as function pairs
- The infinite recursion trap
- Backing field convention
- Property resolution chain
- Use cases (validation, computed values, caching)
- **Debunks:** "Properties store values", "Underscore is special"

### System Integration

#### docs/manuals/ahk2/explanation/windows-message-architecture.md
**Understand:** Complete message flow from hardware to your code
- Hardware → Driver → System Queue → Thread Queue → GetMessage → WndProc
- Posted vs sent messages (architectural difference)
- Hook interception points
- Message parameter decoding
- Performance: polling vs message-driven (70-90% CPU reduction)
- **Debunks:** "Window message queues exist", "Posted and sent are the same"

#### docs/manuals/ahk2/explanation/error-handling-philosophy.md
**Understand:** Error handling as architecture
- Error flow through call stack
- When to catch vs propagate (decision tree)
- Context-specific strategies
- OnError global safety net
- Timer callback error handling (thread problem)
- Retry patterns (exponential backoff, circuit breaker)
- **Debunks:** "try/catch is expensive", "Catch everything", "Timer errors caught externally"

#### docs/manuals/ahk2/explanation/memory-management-architecture.md
**Understand:** How reference counting works, why cycles leak
- Reference counting vs garbage collection
- Reference count lifecycle (timeline visualization)
- Why circular references leak forever (no cycle detector)
- How closures extend object lifetime
- Why timers/events keep objects alive
- GUI special case (Gui.Show() refcount increment)
- **Debunks:** "AHK2 has GC", "Cycles auto-detected", "SetTimer 'Off' releases memory", "Closures capture by value"

#### docs/manuals/ahk2/explanation/gui-architecture-patterns.md
**Understand:** GUI programming paradigm shift (procedural → OO)
- Why no native anchoring (design philosophy)
- Method chaining architecture (fluent interfaces)
- Event sink pattern (why `this` parameter works)
- Component composition vs inheritance
- MVC in AHK context (separation of concerns)
- State management with observable properties
- Why community libraries exist (intentional gaps)
- **Debunks:** "AHK2 has layout managers", "GUI vars can be local", "Properties store values", "Event handlers don't need params"

#### docs/manuals/ahk2/explanation/dllcall-marshaling-internals.md
**Understand:** What happens at memory level when DllCall executes
- Parameter marshaling (AHK → C binary format)
- Calling conventions (64-bit registers vs 32-bit stack)
- Memory layout and stack frame visualization
- Why incorrect types cause crashes
- Return value extraction from registers
- A_LastError mechanism and timing
- **Debunks:** "Type specs don't matter", "32-bit and 64-bit work the same", "Strings passed directly", "Buffer allocation isn't critical"

#### docs/manuals/ahk2/explanation/windows-api-type-philosophy.md
**Understand:** Why Ptr for handles but Int for coordinates
- What handles really are (kernel object addresses)
- Binary flags and bitwise OR operations
- Why flag combinations use OR not addition
- String encoding choices (UTF-16 vs ANSI)
- Struct alignment rules and padding
- Platform portability considerations
- **Debunks:** "Handles are just numbers", "Int works for handles on 64-bit", "Flags can be added", "Alignment doesn't matter"

---

## Document Map by Topic

### Closures
- **Tutorial:** docs/manuals/ahk2/tutorials/02-intermediate-concepts.md (Section: Understanding Closures)
- **How-To:** docs/manuals/ahk2/how-to/closure-loop-pattern.md
- **Reference:** docs/manuals/ahk2/reference/closure-patterns.md
- **Explanation:** docs/manuals/ahk2/explanation/closures-deep-dive.md

### Error Handling
- **Tutorial:** docs/manuals/ahk2/tutorials/02-intermediate-concepts.md (Section: Error Handling)
- **How-To:**
  - docs/manuals/ahk2/how-to/custom-error-handler.md
  - docs/manuals/ahk2/how-to/retry-network-operations.md
  - docs/manuals/ahk2/how-to/circuit-breaker-pattern.md
- **Reference:** docs/manuals/ahk2/reference/error-types.md
- **Explanation:** docs/manuals/ahk2/explanation/error-handling-philosophy.md

### Classes & Objects
- **Tutorial:** docs/manuals/ahk2/tutorials/02-intermediate-concepts.md (Section: Introduction to Classes)
- **Reference:** docs/manuals/ahk2/reference/class-system.md
- **Explanation:** docs/manuals/ahk2/explanation/class-system.md

### Properties
- **Tutorial:** docs/manuals/ahk2/tutorials/02-intermediate-concepts.md (Section: Working with Properties)
- **Reference:** docs/manuals/ahk2/reference/property-system.md
- **Explanation:** docs/manuals/ahk2/explanation/property-vs-field.md

### Type System
- **Tutorial:** docs/manuals/ahk2/tutorials/01-getting-started.md (Section: Variables)
- **How-To:** docs/manuals/ahk2/how-to/type-validation.md
- **Reference:** docs/manuals/ahk2/reference/type-system.md
- **Explanation:** docs/manuals/ahk2/explanation/type-coercion-logic.md

### Windows & Messages
- **How-To:**
  - docs/manuals/ahk2/how-to/window-dragging.md
  - docs/manuals/ahk2/how-to/save-restore-positions.md
  - docs/manuals/ahk2/how-to/detect-window-creation.md
- **Reference:** docs/manuals/ahk2/reference/window-messages.md
- **Explanation:** docs/manuals/ahk2/explanation/windows-message-architecture.md

### Memory Management
- **How-To:**
  - docs/manuals/ahk2/how-to/prevent-memory-leaks.md
  - docs/manuals/ahk2/how-to/debug-memory-leaks.md
- **Reference:** docs/manuals/ahk2/reference/memory-management.md
- **Explanation:** docs/manuals/ahk2/explanation/memory-management-architecture.md

### GUI & Graphics
- **How-To:**
  - docs/manuals/ahk2/how-to/responsive-gui-layouts.md
  - docs/manuals/ahk2/how-to/reusable-gui-components.md
  - docs/manuals/ahk2/how-to/gui-styling-theming.md
  - docs/manuals/ahk2/how-to/gui-mvc-pattern.md
- **Reference:** docs/manuals/ahk2/reference/gui-system.md
- **Explanation:** docs/manuals/ahk2/explanation/gui-architecture-patterns.md

### DllCall & Windows API
- **How-To:**
  - docs/manuals/ahk2/how-to/work-with-structs.md
  - docs/manuals/ahk2/how-to/dllcall-output-parameters.md
  - docs/manuals/ahk2/how-to/debug-dllcall-errors.md
  - docs/manuals/ahk2/how-to/translate-msdn-to-dllcall.md
  - docs/manuals/ahk2/how-to/common-windows-apis.md
- **Reference:** docs/manuals/ahk2/reference/dllcall-type-system.md
- **Explanation:**
  - docs/manuals/ahk2/explanation/dllcall-marshaling-internals.md
  - docs/manuals/ahk2/explanation/windows-api-type-philosophy.md

---

## Using This Manual with Claude

### In Conversations

Reference docs directly:
```
docs/manuals/ahk2/how-to/closure-loop-pattern.md
```

Claude will:
- Load the exact content
- Understand context from cross-references
- Provide answers grounded in this documentation

### Navigation Pattern

1. **Start with tutorials** if learning
2. **Jump to how-to** for specific tasks
3. **Check reference** for syntax lookups
4. **Read explanations** to understand why/how

### Cross-References

All docs use `docs/manuals/ahk2/` syntax for internal links. Claude can follow these automatically.

---

## Source Attribution

This manual was synthesized from expert research reports:
- `ahk2_learning_guide.md` - Beginner's journey
- `ahk2_mastery.md` - Advanced patterns
- `ahk2_lang_01_types.md` - Type system deep dive
- `ahk2_lang_06_closures.md` - Closure mechanics
- `ahk2_lang_07_memory.md` - Memory management deep dive
- `ahk2_arch_01_classes.md` - Class architecture
- `ahk2_arch_02_properties.md` - Property system
- `ahk2_arch_04_error.md` - Error handling comprehensive guide
- `ahk2_arch_07_file_io.md` - File I/O patterns
- `ahk2_sys_01_window_messages.md` - Windows messaging deep dive
- `ahk2_sys_07_graphics.md` - GUI and graphics patterns
- `ahk2_sys_10_dllcall.md` - DllCall and Windows API mastery

Original reports available in: `docs/source-reports/ahk2/`

---

## Manual Statistics

- **2 Tutorials** (progressive learning)
- **20 How-To Guides** (task solutions)
- **9 Reference Docs** (lookup tables)
- **10 Explanation Docs** (deep understanding)

**Total:** 41 documents optimized for Claude Code integration

---

## Quick Reference Card

```
Need to...                          → Go to...
─────────────────────────────────────────────────────────────
Learn AHK2 from scratch             → tutorials/01-getting-started.md
Fix closure loop problem            → how-to/closure-loop-pattern.md
Fix memory leaks                    → how-to/prevent-memory-leaks.md
Debug memory issues                 → how-to/debug-memory-leaks.md
Build responsive GUI                → how-to/responsive-gui-layouts.md
Create reusable components          → how-to/reusable-gui-components.md
Style/theme GUI                     → how-to/gui-styling-theming.md
Implement MVC pattern               → how-to/gui-mvc-pattern.md
Work with Windows structs           → how-to/work-with-structs.md
Handle DllCall output params        → how-to/dllcall-output-parameters.md
Debug DllCall errors                → how-to/debug-dllcall-errors.md
Translate MSDN to DllCall           → how-to/translate-msdn-to-dllcall.md
Use common Windows APIs             → how-to/common-windows-apis.md
Look up error types                 → reference/error-types.md
Check memory functions              → reference/memory-management.md
Look up GUI methods/events          → reference/gui-system.md
Look up DllCall types               → reference/dllcall-type-system.md
Understand why closures work        → explanation/closures-deep-dive.md
Understand memory model             → explanation/memory-management-architecture.md
Understand GUI architecture         → explanation/gui-architecture-patterns.md
Understand DllCall marshaling       → explanation/dllcall-marshaling-internals.md
Understand Windows API types        → explanation/windows-api-type-philosophy.md
Implement retry logic               → how-to/retry-network-operations.md
Check message parameters            → reference/window-messages.md
Learn about classes                 → tutorials/02-intermediate-concepts.md
Validate user input                 → how-to/type-validation.md
Understand message flow             → explanation/windows-message-architecture.md
Set up error logging                → how-to/custom-error-handler.md
```

---

## Version

**Manual Version:** 3.0
**AutoHotkey Version:** v2.0+
**Last Updated:** 2025-11-01
**Source Reports:** 12 documents consolidating 1000+ pages of research

---

**Happy AutoHotkeying! Reference this manual with `docs/manuals/ahk2/` in any Claude conversation.**
