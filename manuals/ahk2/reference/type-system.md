# Reference: Type System

Complete reference for AutoHotkey v2 type system including primitives, coercion rules, operators, and type checking.

## Related Documentation
- docs/manuals/ahk2/reference/error-types.md - Error types thrown for type violations

---

## Primitive Types

| Type | Description | Example Values | Truthiness |
|------|-------------|----------------|------------|
| **Integer** | Whole numbers, 64-bit signed | `0`, `-5`, `42`, `0xFF` | 0 is falsy, non-zero truthy |
| **Float** | Floating-point, double precision | `0.0`, `3.14`, `-2.5`, `1e4` | 0.0 is falsy, non-zero truthy |
| **String** | Text, immutable | `"hello"`, `"123"`, `""` | Empty `""` and `"0"` are falsy |
| **Object** | Base object type | `{}`, `[]`, `Map()` | Always truthy (even empty) |

### Integer Format Support

```ahk
decimal     := 42
hexadecimal := 0xFF      ; Prefix: 0x
negative    := -789
```

### Float Format Support

```ahk
standard    := 3.14
scientific  := 1e4       ; 10000.0
negativeExp := 2.1E-4    ; 0.00021
```

### Numeric Detection Rules

A string "looks numeric" if it contains:
- Only digits (0-9)
- Optional sign (+/-)
- Maximum one decimal point
- Leading/trailing whitespace allowed
- Hexadecimal format (0xFF)
- Scientific notation (1e4)

**Looks numeric:** `"123"`, `"  456  "`, `"-789"`, `"12.34"`, `"0xFF"`, `"1e4"`
**Does NOT look numeric:** `"12abc"`, `"12.34.56"`, `"hello"`, `""`

---

## Type Checking Functions

| Function | Returns | Description |
|----------|---------|-------------|
| `IsInteger(value)` | Boolean | True if value is integer or integer string |
| `IsNumber(value)` | Boolean | True if value is numeric (int or float) |
| `IsFloat(value)` | Boolean | True if value is float or float string |
| `IsObject(value)` | Boolean | True if value is an object |
| `Type(value)` | String | Returns type name: "Integer", "Float", "String", "Object", etc. |

### Type Checking Examples

```ahk
IsInteger(42)        ; True
IsInteger("42")      ; True
IsInteger("42.5")    ; False

IsNumber("123")      ; True
IsNumber("0xFF")     ; True
IsNumber("abc")      ; False

Type(42)             ; "Integer"
Type(3.14)           ; "Float"
Type("hello")        ; "String"
Type([])             ; "Array"
Type(Map())          ; "Map"
```

**Performance:** All `Is*()` functions are O(1) operations, optimized at C++ level.

---

## Type Conversion Functions

| Function | Behavior | Error Handling |
|----------|----------|----------------|
| `Integer(value)` | Convert to integer, truncates floats | Throws `TypeError` if non-numeric |
| `Float(value)` | Convert to float | Throws `TypeError` if non-numeric |
| `String(value)` | Convert to string | Never throws |
| `Number(value)` | Convert to int or float (auto-detect) | Throws `TypeError` if non-numeric |

### Conversion Examples

```ahk
Integer("42")        ; 42
Integer(3.14)        ; 3 (truncated)
Integer("abc")       ; Throws TypeError

Float("42")          ; 42.0
Float("3.14")        ; 3.14
Float("")            ; Throws TypeError (v2 behavior)

String(42)           ; "42"
String(3.14)         ; "3.14"
String({})           ; "Object" (class name)
```

**Important v2 Change:** Empty strings (`""`) throw `TypeError` in arithmetic operations. In v1, they were treated as zero.

---

## Boolean Context (Truthiness)

### Falsy Values

| Value | Type | Example |
|-------|------|---------|
| Integer 0 | Integer | `if (0)` → FALSE |
| Float 0.0 | Float | `if (0.0)` → FALSE |
| Empty string | String | `if ("")` → FALSE |
| String "0" | String | `if ("0")` → FALSE |

**Critical:** String `"0"` is falsy in v2 (major gotcha). This is a deliberate v2 design change.

### Truthy Values

| Value | Type | Example |
|-------|------|---------|
| Non-zero numbers | Integer/Float | `if (1)`, `if (-5)`, `if (0.001)` → TRUE |
| Non-empty strings | String | `if ("hello")`, `if ("false")` → TRUE |
| String "0.0" | String | `if ("0.0")` → TRUE (not exactly "0") |
| All objects | Object | `if ([])`, `if (Map())` → TRUE |
| All functions | Func | `if (MyFunc)` → TRUE |

**Empty collections are truthy:** Always check `.Length` or `.Count` explicitly.

```ahk
; Wrong
emptyArray := []
if (emptyArray)  ; TRUE - wrong!
    MsgBox "Has items"

; Right
if (emptyArray.Length > 0)  ; FALSE - correct
    MsgBox "Has items"
```

---

## Operators by Category

### Arithmetic Operators

| Operator | Name | Type Requirements | Return Type | Notes |
|----------|------|-------------------|-------------|-------|
| `+` | Addition | Numeric | Integer if both int, else Float | Throws `TypeError` if non-numeric |
| `-` | Subtraction | Numeric | Integer if both int, else Float | Throws `TypeError` if non-numeric |
| `*` | Multiplication | Numeric | Integer if both int, else Float | Throws `TypeError` if non-numeric |
| `/` | Division | Numeric | **Always Float** | Even `10 / 5` returns `2.0` |
| `//` | Floor Division | Numeric | Integer if both int, else Float | Toward zero (int) or -infinity (float) |
| `**` | Exponentiation | Numeric | Float if fractional exponent | `2 ** 3` = `8` |
| `%` | Modulo | Numeric | Same as dividend | `10 % 3` = `1` |

**Division Gotcha:** Regular division `/` always returns Float, even for integers.

```ahk
result1 := 10 / 5    ; 2.0 (Float!)
result2 := 10 // 5   ; 2 (Integer)
result3 := 10 / 4    ; 2.5 (Float)
```

**Floor Division Quirk:**
```ahk
; Integer operands: truncate toward zero
5 // 3          ; 1
-5 // 3         ; -1

; Float operands: floor toward negative infinity
5.0 // 3        ; 1.0
-5.0 // 3       ; -2.0 (different!)
```

### Bitwise Operators

| Operator | Name | Type Requirements | Return Type | Notes |
|----------|------|-------------------|-------------|-------|
| `&` | Bitwise AND | Integer (floats truncated) | Integer | `5 & 3` = `1` |
| `\|` | Bitwise OR | Integer (floats truncated) | Integer | `5 \| 3` = `7` |
| `^` | Bitwise XOR | Integer (floats truncated) | Integer | `5 ^ 3` = `6` |
| `~` | Bitwise NOT | Integer (floats truncated) | Integer | `~5` = `-6` |
| `<<` | Left shift | Integer (floats truncated) | Integer | `5 << 2` = `20` |
| `>>` | Right shift (arithmetic) | Integer (floats truncated) | Integer | `20 >> 2` = `5` |
| `>>>` | Logical right shift | Integer (floats truncated) | Integer | Fills with zeros |

**Float Handling:** Floats are truncated (not rounded) before bitwise operations.

### Concatenation Operators

| Operator | Name | Type Requirements | Return Type | Notes |
|----------|------|-------------------|-------------|-------|
| `.` | Concatenation | Any | String | Requires spaces: `"a" . "b"` |
| Adjacent strings | Implicit concat | String literals | String | `"Hello" " World"` |

```ahk
"abc" . "def"        ; "abcdef"
5 . 10               ; "510"
"Value: " . 42       ; "Value: 42"

; Implicit (string literals only)
x := "Hello"
y := x " World"      ; "Hello World"
```

### Comparison Operators

| Operator | Name | Case Sensitivity | Coercion Behavior |
|----------|------|------------------|-------------------|
| `=` | Equal | **Insensitive** | Numeric if both look numeric, else alphabetic |
| `==` | Equal | **Sensitive** | Numeric if both look numeric, else alphabetic |
| `!=` | Not Equal | Follows `StringCaseSense` | Same coercion as `=` |
| `!==` | Not Equal | **Sensitive** | Same coercion as `==` |
| `<` | Less Than | N/A | Numeric if both look numeric, else alphabetic |
| `>` | Greater Than | N/A | Numeric if both look numeric, else alphabetic |
| `<=` | Less or Equal | N/A | Numeric if both look numeric, else alphabetic |
| `>=` | Greater or Equal | N/A | Numeric if both look numeric, else alphabetic |

**Critical Distinction:** `=` vs `==` is **case sensitivity**, NOT type coercion behavior. Both coerce to numeric when appropriate.

#### Comparison Examples

```ahk
; Numeric comparisons (both operands numeric)
1 == 01                    ; TRUE
"10" > "5"                 ; TRUE (numeric: 10 > 5)
123 == 123.0               ; TRUE

; String comparisons (at least one non-numeric)
"10" > "5a"                ; FALSE (alphabetic: "1" < "5")
"2" < "10"                 ; FALSE (alphabetic: "2" > "1")
"abc" = "ABC"              ; TRUE (case-insensitive)
"abc" == "ABC"             ; FALSE (case-sensitive)

; Edge cases
0 = "0"                    ; TRUE (numeric)
0 = "00"                   ; FALSE (string mode)
"" . 10 > "" . 5           ; FALSE (forced string)
```

**Force String Comparison:** Concatenate with empty string to prevent numeric coercion.

```ahk
numStr1 := "2"
numStr2 := "10"
numStr1 < numStr2          ; TRUE (2 < 10 numerically)
"" . numStr1 < "" . numStr2  ; FALSE ("2" > "1" alphabetically)
```

### Logical Operators

| Operator | Alt Syntax | Returns | Behavior |
|----------|------------|---------|----------|
| `!` | `not` | 1 or 0 | Returns 1 if falsy, 0 if truthy |
| `&&` | `and` | **First or second operand** | Returns first if falsy, else second |
| `\|\|` | `or` | **First or second operand** | Returns first if truthy, else second |

**Important:** `&&` and `||` do NOT return true/false—they return one of the operands!

```ahk
x := 5 && 10         ; x = 10 (first truthy, returns second)
x := 0 && 10         ; x = 0 (first falsy, returns first)
x := 5 || 10         ; x = 5 (first truthy, returns first)
x := 0 || 10         ; x = 10 (first falsy, returns second)

; Practical use: default values
result := GetValue() || "default"

; Safe division with short-circuit
safe := (x != 0) && (y / x > 2)  ; Prevents division by zero
```

**Precedence Warning:**
```ahk
Test1 := not 0 + 1   ; not(0+1) = not(1) = 0
Test2 := !0 + 1      ; (!0) + 1 = 1 + 1 = 2
```

### Ternary Operator

```ahk
condition ? valueIfTrue : valueIfFalse

age >= 18 ? "Adult" : "Minor"
count > 0 ? "Items: " . count : "Empty"
```

---

## Type Coercion Decision Tree

### Operator Evaluation Process

```
1. IDENTIFY OPERATOR TYPE
   ├─ Math (+, -, *, /) → Require numeric, convert strings
   ├─ Comparison (=, <, >) → Numeric if both look numeric, else alphabetic
   └─ Concatenation (.) → Convert both to string

2. MATH OPERATORS
   ├─ Both pure numbers? → Calculate directly
   ├─ Both numeric strings? → Convert to number, calculate
   └─ At least one non-numeric? → THROW TypeError

3. COMPARISON OPERATORS
   ├─ Both look numeric? → Numeric comparison
   └─ Otherwise → Alphabetic comparison
       ├─ = → Case-insensitive
       └─ == → Case-sensitive

4. CONCATENATION
   ├─ Convert left to string
   ├─ Convert right to string
   └─ Concatenate
```

---

## Comparison Decision Matrix

| Left Operand | Right Operand | Comparison Type | Example | Result |
|--------------|---------------|-----------------|---------|--------|
| Integer | Integer | Numeric | `10 > 5` | TRUE |
| Float | Integer | Numeric | `10.5 > 5` | TRUE |
| Numeric String | Numeric String | Numeric | `"10" > "5"` | TRUE |
| Numeric String | Non-Numeric String | Alphabetic | `"10" > "5a"` | FALSE |
| Non-Numeric | Any | Alphabetic | `"abc" > "xyz"` | FALSE |

---

## Map Key Type Coercion

**Important:** Integers and strings are **distinct key types** in Maps.

```ahk
myMap := Map()
myMap[5] := "integer key"
myMap["5"] := "string key"

MsgBox myMap[5]      ; "integer key"
MsgBox myMap["5"]    ; "string key"

; Float keys become strings
myMap[5.0] := "float key"
MsgBox myMap["5.0"]  ; "float key"
MsgBox myMap[5]      ; Still "integer key" (different key!)
```

**Gotcha:** User input (from InputBox, IniRead, etc.) is always string. Convert explicitly:

```ahk
; Wrong - key type mismatch
userInput := InputBox("Enter ID:").Value  ; Returns string
item := data[userInput]  ; Fails if keys are integers

; Right - convert to integer
userInput := InputBox("Enter ID:").Value
item := data[Integer(userInput)]
```

---

## Common Pitfalls

### 1. String "0" Truthiness

```ahk
; Wrong
count := "0"
if (count)  ; FALSE - not what you expect!
    MsgBox "Has value"

; Right
if (StrLen(count) > 0)  ; TRUE
    MsgBox "Has value"
if (count != "")        ; TRUE
    MsgBox "Has value"
```

### 2. Empty String in Math

```ahk
; v2 behavior - throws TypeError
x := ""
result := x + 5  ; TypeError!

; Must validate first
if IsNumber(x)
    result := x + 5
```

### 3. Division Always Returns Float

```ahk
; Wrong
pages := totalItems / itemsPerPage  ; Returns float!
Loop pages  ; Error - Loop expects integer

; Right
pages := totalItems // itemsPerPage  ; Integer division
pages := Ceil(totalItems / itemsPerPage)  ; Or round up
```

### 4. Empty Collections Are Truthy

```ahk
; Wrong
results := []
if (results)  ; TRUE - wrong!
    MsgBox "Found results"

; Right
if (results.Length > 0)  ; FALSE - correct
    MsgBox "Found results"
```

### 5. Leading Zeros in Comparison

```ahk
0 = "0"    ; TRUE (numeric)
0 = "00"   ; FALSE (string mode - leading zero)
```

---

## Performance Notes

- **Type checking functions** (`IsInteger`, `IsNumber`, etc.): O(1), negligible overhead
- **Type conversion** (`Integer`, `Float`, `String`): Highly optimized, minimal cost
- **Arithmetic operators**: Integer operations slightly faster than float
- **String concatenation**: Use array + join for large batches to avoid repeated allocations

**Best Practice:** Convert once, use many times.

```ahk
; Good - convert once
userInput := "42"
if IsNumber(userInput) {
    num := Integer(userInput)
    result1 := num * 2
    result2 := num + 10
    result3 := num / 3
}

; Less efficient - repeated conversion
if IsNumber(userInput) {
    result1 := Integer(userInput) * 2
    result2 := Integer(userInput) + 10
    result3 := Integer(userInput) / 3
}
```

---

## See Also

- docs/manuals/ahk2/reference/error-types.md - TypeError and ValueError details
- Official: [Type Function](https://www.autohotkey.com/docs/v2/lib/Type.htm)
- Official: [Is Functions](https://www.autohotkey.com/docs/v2/lib/Is.htm)
