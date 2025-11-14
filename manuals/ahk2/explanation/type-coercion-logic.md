# Explanation: How AHK2 Decides Type Conversions

**Understanding the parser's decision-making process for type coercion**

AutoHotkey v2 is often misunderstood as "loosely typed like JavaScript," leading to confusion when `"10" > "5"` is true but `"10" > "5a"` is false. The reality is more nuanced: **AHK2 implements predictable rules with explicit error handling**. Understanding the parser's decision tree—how it determines whether to treat values as numbers or strings—transforms type coercion from mysterious magic into predictable behavior you can reason about.

## The Core Algorithm: How the Parser Thinks

When AHK2 encounters an expression like `value1 + value2`, it follows a precise decision tree:

```
STEP 1: Identify Operator Type
┌───────────┬──────────────┬─────────────┐
│ Math      │ Comparison   │ Concat      │
│ +,-,*,/   │ =,==,<,>     │ .           │
└─────┬─────┴──────┬───────┴──────┬──────┘
      │            │              │
      ▼            ▼              ▼

MATH PATH:             COMPARISON PATH:       CONCAT PATH:
1. Both pure numbers?  1. Both pure nums?     1. Convert left to string
   YES → Calculate        YES → Numeric        2. Convert right to string
   NO → Check next        NO → Check next      3. Concatenate
                                               4. Return String
2. Both numeric strings? 2. Both numeric str?
   YES → Convert, calc      YES → Numeric
   NO → TypeError           NO → Alphabetic

3. At least one             Case sensitivity:
   non-numeric?             = → Insensitive
   → THROW TypeError        == → Sensitive
```

## The "Looks Numeric" Rule

The fundamental principle: **If a value can be interpreted as a number without loss of information, it "looks numeric."**

### Numeric Detection Criteria

**Passes as numeric:**
- Contains only: digits (0-9), optional sign (+/-), one decimal point
- Leading/trailing whitespace allowed
- Hexadecimal: `0xFF` format
- Scientific notation: `1e4`, `-2.1E-4`

**Examples:**
```ahk
; These "look numeric"
"123", "  456  ", "-789", "12.34", "0xFF", "1e4"

; These do NOT
"12abc", "12.34.56", "hello", ""
```

### Empty String Special Case

**v1 behavior:** Empty string treated as 0 (implicit conversion)
**v2 behavior:** Empty string throws TypeError in math operations

```ahk
; v1: Works silently
result := "" + 5  ; 5

; v2: Throws TypeError
result := "" + 5  ; ERROR!
```

**Why the change:** Explicit errors catch bugs. Silent coercion hides problems.

## Comparison Operators: Numeric vs. Alphabetic

### The Decision Matrix

| Left Operand | Right Operand | Comparison Type | Example | Result |
|--------------|---------------|-----------------|---------|--------|
| Integer | Integer | Numeric | `10 > 5` | TRUE |
| Float | Integer | Numeric | `10.5 > 5` | TRUE |
| Numeric String | Numeric String | Numeric | `"10" > "5"` | TRUE |
| Numeric String | Non-Numeric String | Alphabetic | `"10" > "5a"` | FALSE |
| Non-Numeric | Any | Alphabetic | `"abc" > "xyz"` | FALSE |

### Case Sensitivity: `=` vs. `==`

**Common misconception:** `=` does type coercion, `==` doesn't.

**Reality:** The difference is **case sensitivity, not type coercion**. Both do numeric comparison when appropriate.

| Operator | Case Sensitivity | Type Coercion |
|----------|------------------|---------------|
| `=` | Insensitive | Numeric if both look numeric |
| `==` | Sensitive | Numeric if both look numeric |
| `!=` | Follows StringCaseSense | Same as `=` |
| `!==` | Sensitive | Same as `==` |

**Examples:**
```ahk
"abc" = "ABC"      ; TRUE - case-insensitive string
"abc" == "ABC"     ; FALSE - case-sensitive string
0 = "0"            ; TRUE - numeric comparison
0 = "00"           ; FALSE - "00" triggers string mode
"10" > "5"         ; TRUE - numeric comparison
"10" > "5a"        ; FALSE - alphabetic comparison
```

### The "00" Gotcha

```ahk
0 = "0"    ; TRUE - both look numeric, numeric comparison
0 = "00"   ; FALSE - "00" doesn't "look numeric" (ambiguous)
           ; Falls back to string comparison
```

**Why "00" isn't numeric:** The extra leading zero makes it ambiguous. Is this the number 0 or the string "00"? AHK chooses string comparison to preserve information.

## Arithmetic Operators: Type Requirements and Returns

### Division: Always Returns Float

```ahk
result1 := 10 / 5    ; 2.0 (Float, not Integer!)
result2 := 10 // 5   ; 2 (Integer - floor division)
result3 := 10 / 4    ; 2.5 (Float)
```

**Critical:** Regular division `/` **always** returns Float, even with integer operands.

Only floor division `//` returns Integer (when both operands are integers).

### Type Preservation Rules

**Integer + Integer → Integer**
```ahk
x := 5 + 3  ; 8 (Integer)
```

**Any Float operand → Float result**
```ahk
x := 5 + 3.0  ; 8.0 (Float)
x := 5.0 * 2  ; 10.0 (Float)
```

**Division exception → Always Float**
```ahk
x := 10 / 5  ; 2.0 (Float, not Integer)
```

### Floor Division Quirk

```ahk
; Integer operands: truncates toward zero
5 // 3          ; 1
-5 // 3         ; -1 (toward zero)

; Float operand: floors toward negative infinity
5.0 // 3        ; 1.0
-5.0 // 3       ; -2.0 (toward negative infinity!)
```

**Behavior change:** Integer operands truncate toward zero. Float operands floor toward negative infinity.

### Empty String in Arithmetic

**v2 change:** Empty strings no longer treated as zero.

```ahk
; v1: Silent conversion
x := "" + 5  ; 5

; v2: Explicit error
x := "" + 5  ; TypeError: Cannot convert "" to number
```

## Logical Operators: Short-Circuit and Return Values

### Critical Misconception

Logical operators do NOT return true/false—they return **one of the operands**!

```ahk
x := 5 && 10         ; x = 10 (second operand)
x := 0 && 10         ; x = 0 (first operand)
x := 5 || 10         ; x = 5 (first operand)
x := 0 || 10         ; x = 10 (second operand)
```

### Operator Behavior

**`!` / `not`** - Returns 1 if operand is falsy, 0 otherwise

**`&&` / `and`** - Returns first operand if falsy, otherwise second operand

**`||` / `or`** - Returns first operand if truthy, otherwise second operand

### Practical Use: Default Values

```ahk
; Use default if GetValue returns 0/""
result := GetValue() || "default"

; Safe division (prevents division by zero)
safe := (x != 0) && (y / x > 2)
```

### Precedence Matters

```ahk
; High precedence ! vs low precedence not
Test1 := not 0 + 1   ; not(0+1) = not(1) = 0
Test2 := !0 + 1      ; (!0) + 1 = 1 + 1 = 2
```

## Boolean Context: Truthiness Rules

### Falsy Values (Evaluate to False)

| Value | Example | Notes |
|-------|---------|-------|
| Integer 0 | `if (0)` | Standard falsy |
| Float 0.0 | `if (0.0)` | Standard falsy |
| Empty String | `if ("")` | Standard falsy |
| **String "0"** | `if ("0")` | **GOTCHA: Falsy in v2!** |
| Uninitialized Var | `if (undeclared)` | **Throws error in v2** |

### Truthy Values (Evaluate to True)

| Value | Example | Notes |
|-------|---------|-------|
| Non-zero numbers | `if (1)`, `if (-5)` | All truthy |
| Non-empty strings | `if ("hello")`, `if ("false")` | Even "false" is truthy! |
| String "0.0" | `if ("0.0")` | Truthy (not exactly "0") |
| All objects | `if ([])`, `if (Map())` | **Even empty objects!** |
| All functions | `if (MyFunc)` | Function references truthy |

### The String "0" Gotcha Explained

**Why is "0" falsy?** In v2, quoted strings are evaluated the same as variables containing strings. Since "0" is a numeric string, it's evaluated as numeric 0 in boolean contexts.

**Official documentation:**

> "Quoted literal strings and strings produced by concatenating with quoted literal strings are no longer unconditionally considered non-numeric. This has the following implications: Quoted literal '0' is considered false."

**Workarounds:**
```ahk
userInput := "0"

; Problem
if (userInput)  ; FALSE - not what you want!

; Solution 1: Check length
if (StrLen(userInput) > 0)  ; TRUE

; Solution 2: Explicit comparison
if (userInput != "")        ; TRUE

; Solution 3: Type check
if (userInput !== "")       ; TRUE
```

### Empty Collections Are Truthy

**Critical:** Empty arrays/maps/objects are truthy. Always check properties.

```ahk
emptyArray := []
if (emptyArray)                  ; TRUE - wrong!
if (emptyArray.Length > 0)       ; FALSE - correct

emptyMap := Map()
if (emptyMap)                    ; TRUE - wrong!
if (emptyMap.Count > 0)          ; FALSE - correct
```

## String vs. Number Decision Process

### When to Treat as String

**Semantic meaning:**
- Identifiers/Labels/Names → STRING
  - Examples: IDs, usernames, product codes
  - Action: Keep as string, never do math

**Even if numeric format:**
```ahk
productID := "12345"  ; Don't convert to Integer

; Correct comparison
if (productID = "12345")  ; String comparison

; Wrong approach
if (Integer(productID) = 12345)  ; Unnecessary
```

### When to Treat as Number

**Semantic meaning:**
- Quantifiable/Countable → NUMBER
  - Examples: age, price, quantity
  - Action: Validate with IsNumber(), convert

```ahk
ValidateAge(input) {
    if (input = "")
        return {valid: false, error: "Age is required"}

    if !IsNumber(input)
        return {valid: false, error: "Age must be numeric"}

    age := Integer(input)
    if (age < 0 || age > 150)
        return {valid: false, error: "Age must be 0-150"}

    return {valid: true, value: age}
}
```

### Depends on Use Case

**Example: Zip Codes**

```ahk
zipCode := "90210"

; For display: keep as string
MsgBox "Shipping to: " zipCode

; For range validation: can compare numerically
if IsNumber(zipCode) {
    numZip := Integer(zipCode)
    if (numZip >= 90000 && numZip <= 96699)
        MsgBox "California zip code"
}
```

## Map Key Type Coercion

**Critical:** Integer and string keys are **distinct** in Maps.

```ahk
myMap := Map()
myMap[5] := "integer key"
myMap["5"] := "string key"

MsgBox myMap[5]    ; "integer key"
MsgBox myMap["5"]  ; "string key"

; These are DIFFERENT keys!
```

**However:** `map[5.0]` stores as string "5.0", creating potential lookup failures.

```ahk
myMap[5] := "int"
myMap[5.0] := "float"  ; Different key!

MsgBox myMap[5]    ; "int"
MsgBox myMap[5.0]  ; "float"
```

**Best practice:** Be consistent with key types. If using user input, always convert:

```ahk
; Solution 1: Convert key
userChoice := InputBox("Enter item:").Value
if IsInteger(userChoice)
    MsgBox myMap[Integer(userChoice)]

; Solution 2: Use string keys consistently
myMap := Map()
myMap["1"] := "First"
myMap["2"] := "Second"
userChoice := InputBox("Enter item:").Value
MsgBox myMap[userChoice]  ; Works - both strings
```

## Real-World Pitfalls

### Pitfall 1: IRC Bot Scoreboard Disaster

**The Bug:** User GeekDude built an IRC bot with Object storing user scores. When user `_NewEnum` received kudos, it shadowed the built-in `_NewEnum` method. JSON library couldn't enumerate, returned `{}`, and the entire scoreboard was wiped.

```ahk
; This destroyed the scoreboard
scoreboard["_NewEnum"] := 5  ; Shadows built-in method
JSON.Dump(scoreboard)        ; Returns "{}"
FileDelete("scoreboard.json")
```

**The Lesson:** Never use dynamic keys in basic Objects. Use Maps:

```ahk
; ✅ Correct - Maps don't have method shadowing
scoreboard := Map()
scoreboard["_NewEnum"] := 5   ; Safe
scoreboard["Clone"] := 10     ; Also safe
```

### Pitfall 2: String "0" Truthiness

Config validation checked `if (maxConnections)` but treated `maxConnections=0` as missing.

```ahk
; ❌ Wrong
maxConnections := IniRead("config.ini", "Server", "MaxConnections", "")
if (maxConnections) {  ; FALSE when "0"!
    ApplySettings(maxConnections)
}

; ✅ Right
if (maxConnections != "") {  ; Explicit check
    ApplySettings(Integer(maxConnections))
}
```

### Pitfall 3: Division Type Assumptions

```ahk
; ❌ Wrong
totalItems := 10
itemsPerPage := 3
pages := totalItems / itemsPerPage  ; 3.333... (Float!)
Loop pages {  ; Error - Loop expects integer
    ; Process page
}

; ✅ Right
pages := totalItems // itemsPerPage + (totalItems mod itemsPerPage ? 1 : 0)
; Or explicitly
pages := Ceil(totalItems / itemsPerPage)
```

### Pitfall 4: Empty String Math (v1→v2 Migration)

```ahk
; v1: Works silently
userInput := ""
total := userInput * price  ; 0

; v2: Throws TypeError
userInput := ""
total := userInput * price  ; ERROR!

; ✅ Solution: Validate first
if (userInput = "") {
    MsgBox "Quantity is required"
    return
}
if !IsNumber(userInput) {
    MsgBox "Quantity must be numeric"
    return
}
total := Integer(userInput) * price
```

## Type Conversion Functions

### Integer(value)

Converts to integer, truncating decimals:

```ahk
Integer("42")     ; 42
Integer("42.7")   ; 42 (truncates)
Integer("  42 ")  ; 42 (trims whitespace)
Integer("abc")    ; Throws ValueError
```

### Float(value)

Converts to floating-point:

```ahk
Float("42.7")     ; 42.7
Float("42")       ; 42.0 (becomes float)
Float("1e3")      ; 1000.0
Float("abc")      ; Throws ValueError
```

### String(value)

Converts to string:

```ahk
String(42)        ; "42"
String(42.5)      ; "42.5"
String([1,2,3])   ; "Array"
```

### Type Checking Functions

```ahk
IsNumber("42")     ; True
IsNumber("42.5")   ; True
IsNumber("abc")    ; False

IsInteger("42")    ; True
IsInteger("42.5")  ; False

IsFloat("42.5")    ; True
IsFloat("42")      ; False (integer, not float)
```

## Performance Considerations

### Type Checking Is Fast

All `Is*()` functions are O(1) operations optimized at the C++ level. Even in tight loops, type checking overhead is negligible.

### Type Conversion Is Fast

`Integer()`, `Float()`, `String()` are highly optimized. The overhead is minimal compared to I/O or complex algorithms.

### Best Practices

**1. Convert Once, Use Many Times**
```ahk
; ✅ Good
userInput := "42"
if IsNumber(userInput) {
    num := Integer(userInput)  ; Convert once
    result1 := num * 2
    result2 := num + 10
    result3 := num / 3
}

; ❌ Wasteful
if IsNumber(userInput) {
    result1 := Integer(userInput) * 2
    result2 := Integer(userInput) + 10
    result3 := Integer(userInput) / 3
}
```

**2. Validate at Boundaries**
```ahk
; ✅ Good - Validate at entry points
ProcessUserData(data) {
    if !IsNumber(data)
        throw ValueError("Data must be numeric")
    return InternalCalc(data)
}

InternalCalc(num) {
    ; No validation - caller validates
    return num * 2
}
```

## Key Insights

**Insight 1: "Looks numeric" drives most decisions**
The parser asks: "Can this be interpreted as a number without ambiguity?" This single principle governs type behavior.

**Insight 2: Explicit over implicit**
v2 favors throwing errors over silent coercion. When types are ambiguous, the parser refuses to guess.

**Insight 3: Context determines conversion**
Math operators demand numbers. Concatenation accepts anything. Comparisons choose numeric or alphabetic based on operand types.

**Insight 4: Truthiness is numeric**
Boolean evaluation treats values numerically when possible. This makes `"0"` falsy (controversial but consistent).

**Insight 5: Types are distinct in collections**
Maps don't blur type boundaries. Integer `5` and string `"5"` are different keys.

**Insight 6: Empty string behavior changed**
v1 treated `""` as 0. v2 throws TypeError. This catches bugs but breaks legacy code.

**Insight 7: String "0" is falsy**
Quoted strings are evaluated like variables. `"0"` is numeric string, evaluates as 0, which is falsy.

## Cross-References

Related topics:
- docs/manuals/ahk2/reference/type-functions.md - Type checking/conversion reference
- docs/manuals/ahk2/how-to/input-validation.md - Validation patterns
- docs/manuals/ahk2/reference/operators.md - Operator precedence and behavior
- docs/manuals/ahk2/explanation/error-handling-philosophy.md - Handling type errors
