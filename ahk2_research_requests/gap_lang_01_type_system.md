# AHK2 Research Request: Type System & Dynamic Typing

## What I Need to Learn

I keep hitting type coercion bugs I don't understand. When is "5" treated as a number? When does 5 become "5"? I need to deeply understand AHK2's type system to write code that behaves predictably.

## My Current Understanding (Challenge This!)

I believe:
- AHK2 is "loosely typed" like JavaScript
- Comparison with `=` does type coercion, `==` prevents it
- Empty string equals 0
- "0" (string zero) is falsy like JavaScript
- Division `/` returns integers when both operands are integers

**Tell me what I have wrong.**

## Specific Questions

1. **Type Coercion Rules**: What are the EXACT rules for when strings become numbers and vice versa?

2. **Comparison Operators**: What's the difference between `=`, `==`, `!=`, `<>`? Give me examples where they differ.

3. **Falsy Values**: What values are falsy in AHK2? Is "0" truthy or falsy?

4. **Arithmetic Behavior**:
```ahk
x := "10"
y := "5"
result1 := x + y   // What is this?
result2 := x . y   // What is this?
result3 := x y     // What is this? Error?
```

5. **Integer vs Float**: When does division return Float vs Integer? What's `//` vs `/`?

## The Bugs I've Hit

**Bug 1:**
```ahk
count := "0"  // From config file (string)
if (count)    // I expected false, got true - WHY?
    DoSomething()
```

**Bug 2:**
```ahk
value := "10"
result := value + 5  // 15 (ok)
result := value . 5  // "105" (ok)
result := value 5    // What happens here?
```

**Bug 3:**
```ahk
mapKey := 5
map[mapKey] := "five"

stringKey := "5"
MsgBox map[stringKey]  // Does this work? Are 5 and "5" the same key?
```

## How to Write This Guide

### 1. Shatter Assumptions Immediately

Start with quiz:
```ahk
Quiz 1: What does this print?
if ("0")
    MsgBox "A"
else
    MsgBox "B"

Quiz 2: What does this print?
MsgBox ("10" + "5")

Quiz 3: Are these equal?
"5" == 5

ANSWERS:
[Show answers that contradict common assumptions]
```

### 2. The Type Coercion Flow Diagram

Show EXACTLY how coercion works:
```
Expression: "10" + 5

Step 1: Operator is +
Step 2: + prefers numbers
Step 3: Try to convert "10" to number
  ├─ Parse "10" → Success → 10 (Integer)
  └─ (If failed → TypeError)
Step 4: Both operands now numbers (10, 5)
Step 5: Perform operation: 10 + 5 = 15
Step 6: Result type: Integer
```

Compare to:
```
Expression: "hello" + 5

Step 1: Operator is +
Step 2: + prefers numbers
Step 3: Try to convert "hello" to number
  └─ Parse failed → throw TypeError
```

### 3. Comparison Matrix

Create complete comparison table:

| Expression | = (Case-insensitive) | == (Case-sensitive) | Result |
|------------|---------------------|---------------------|--------|
| "5" vs 5 | true (both→number) | true (both→number) | |
| "hello" vs "HELLO" | true (case-insensitive) | false (case-sensitive) | |
| "" vs 0 | false (different types) | false (different types) | |
| "0" vs 0 | true (both→number) | true (both→number) | |

### 4. Operator Preference Rules

List every operator and its type preference:
```ahk
Arithmetic (+, -, *, /, //, **, %)  → Number
Concatenation (.)                    → String
Bitwise (&, |, ^, <<, >>, ~)        → Integer
Comparison (>, <, >=, <=)           → Number (if both numeric-looking)
Equality (=, ==, !=, <>)            → Complex (see table)
Logical (&&, ||, !)                 → Boolean
```

### 5. Truthiness Table

Complete table of falsy values:
```ahk
Value         | Boolean Context | Why
--------------|-----------------|-----
0             | false          | Integer zero
0.0           | false          | Float zero
""            | false          | Empty string
"0"           | TRUE           | Non-empty string (gotcha!)
unset var     | false          | Uninitialized
false         | false          | Literal false
[]            | true           | Empty array still truthy
{}            | true           | Empty object still truthy
Map()         | true           | Empty map still truthy
```

### 6. The String vs Number Decision Tree

```
Receiving value from user/file/API
  ↓
What operations will you perform?
├─ Math operations → Convert to number immediately
│   ├─ Integer operations → use Integer()
│   └─ Decimal operations → use Float()
├─ String operations → Keep as string
└─ Conditional checks → Be explicit with IsNumber()

Anti-pattern: Letting coercion happen implicitly
```

### 7. Real-World Validation Patterns

Show complete validation function:
```ahk
ValidateInput(value, expectedType) {
    switch expectedType {
        case "Integer":
            try {
                result := Integer(value)
                // Additional range checks?
                return {valid: true, value: result}
            } catch {
                return {valid: false, error: "Not a valid integer"}
            }

        case "PositiveInteger":
            // Similar but with range check

        case "String":
            // What validation makes sense?
    }
}
```

### 8. Performance of Type Operations

Benchmark conversions:
```
Operation               | Time per 100k ops | Notes
------------------------|-------------------|-------
Integer("123")          | 50ms             | Parsing overhead
Integer(123)            | 5ms              | No conversion needed
String(123)             | 30ms             | Formatting
123 . ""                | 40ms             | Implicit conversion
Type(value)             | 15ms             | Type checking
IsNumber(value)         | 10ms             | Faster than Type()
```

### 9. Common Pitfalls (War Stories)

**"The '0' Gotcha"**
> I spent hours debugging why my config validation failed. Turned out `if (configValue)` was true for string "0" when I expected false. In AHK2, only empty string is falsy!

**"Map Key Confusion"**
> I used number 5 as map key, then tried to access with string "5". Surprise: it worked! Map keys are converted to strings. Lost an afternoon to this.

**"Division Type Surprise"**
> I wrote `size := 10 / 2` expecting Integer(5). Got Float(5.0). Use `//` for integer division!

**"The Concatenation Trap"**
> Wrote `total := subtotal + tax` where both came from text inputs. Got "100.0020.00" (concatenation) not 120.00 (addition). Always convert strings from inputs!

### 10. My Actual Code Transformation

Take this real pattern from my project:
```ahk
; Config loaded from file (all strings)
snapDistance := configValue  // String "10"

; Later used in math
if (distance < snapDistance)  // Comparison coerces to number (works)
    snap := true

; But then:
newDistance := snapDistance + adjustment  // Could be concat or add!
```

Show how to make this type-safe and predictable.

## Success Criteria

After this guide:
1. ✓ Can predict type coercion in complex expressions
2. ✓ Know when to use `=` vs `==`
3. ✓ Never get surprised by "0" truthiness
4. ✓ Write validation that handles all edge cases
5. ✓ Choose right conversion function (Integer vs Float vs String)
6. ✓ Debug type-related bugs quickly

## Most Important

**I want the mental model that lets me look at an expression and KNOW what type it will be.**

Not from memorizing tables, but from understanding how the AHK2 parser thinks about types.

Teach me to think like the type system, not just use it.
