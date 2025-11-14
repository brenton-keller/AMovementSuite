# How-To: Validate and Convert Types Safely

## Problem Statement

User input, configuration files, and external data sources provide values as strings that need to be converted to specific types (integers, floats, booleans). Naive conversion can crash your script, produce wrong results, or silently accept invalid data. You need robust validation that catches errors early, provides clear feedback, and handles edge cases correctly.

## Basic Type Checking Functions

AutoHotkey v2 provides built-in functions for type checking:

```ahk
; Check if value looks numeric (can be converted to number)
IsNumber("123")      ; true
IsNumber("12.34")    ; true
IsNumber("0xFF")     ; true (hexadecimal)
IsNumber("abc")      ; false
IsNumber("")         ; false

; Check if value is or looks like integer
IsInteger("123")     ; true
IsInteger("12.34")   ; false
IsInteger(42)        ; true

; Check if value is or looks like float
IsFloat("12.34")     ; true
IsFloat("12")        ; true (integers are valid floats)
IsFloat(42)          ; true
```

**Key insight:** These functions check if a value **can be interpreted** as that type, not just if it currently is that type.

## Pattern 1: Safe Numeric Conversion

Always validate before converting:

```ahk
SafeToInteger(value, default := 0) {
    ; Empty check
    if (value = "")
        return default

    ; Type check
    if !IsInteger(value)
        throw TypeError("Value must be an integer: " value)

    ; Convert
    return Integer(value)
}

SafeToFloat(value, default := 0.0) {
    if (value = "")
        return default

    if !IsNumber(value)
        throw TypeError("Value must be numeric: " value)

    return Float(value)
}

; Usage
try {
    age := SafeToInteger(userInput)
    price := SafeToFloat(priceInput)
} catch TypeError as e {
    MsgBox "Invalid input: " e.Message
    return
}
```

## Pattern 2: Validation with Range Checking

Combine type checking with value constraints:

```ahk
ValidateInteger(value, min := unset, max := unset) {
    ; Empty check
    if (value = "")
        throw ValueError("Value cannot be empty")

    ; Type check
    if !IsInteger(value)
        throw TypeError("Value must be an integer")

    ; Convert
    num := Integer(value)

    ; Range check
    if IsSet(min) && num < min
        throw ValueError(Format("Value must be >= {1}", min))

    if IsSet(max) && num > max
        throw ValueError(Format("Value must be <= {1}", max))

    return num
}

; Usage
try {
    age := ValidateInteger(input, 0, 150)
    port := ValidateInteger(portInput, 1, 65535)
} catch Error as e {
    MsgBox "Validation error: " e.Message
}
```

## Pattern 3: Universal Validator Function

Reusable validator for any numeric input:

```ahk
ValidateNumber(value, options := unset) {
    ; Default options
    opts := {
        type: "number",           ; "integer", "float", "number"
        min: unset,
        max: unset,
        allowEmpty: false,
        positive: false,          ; Shortcut for min: 0
        fieldName: "Value"
    }

    ; Merge user options
    if IsSet(options) {
        for prop, val in options.OwnProps() {
            opts.%prop% := val
        }
    }

    ; Positive shortcut
    if opts.positive {
        opts.min := 0
    }

    ; Empty check
    if (value = "") {
        if opts.allowEmpty {
            return {valid: true, value: "", isEmpty: true}
        }
        return {valid: false, error: opts.fieldName " cannot be empty"}
    }

    ; Numeric check
    if !IsNumber(value) {
        return {valid: false, error: opts.fieldName " must be numeric"}
    }

    ; Type-specific conversion
    try {
        switch opts.type {
            case "integer":
                if !IsInteger(value) {
                    return {
                        valid: false,
                        error: opts.fieldName " must be a whole number"
                    }
                }
                converted := Integer(value)

            case "float":
                converted := Float(value)

            default:  ; "number" - auto-detect
                converted := IsInteger(value) ? Integer(value) : Float(value)
        }
    } catch {
        return {valid: false, error: "Failed to convert " opts.fieldName}
    }

    ; Range validation
    if IsSet(opts.min) && converted < opts.min {
        return {
            valid: false,
            error: opts.fieldName " must be at least " opts.min
        }
    }
    if IsSet(opts.max) && converted > opts.max {
        return {
            valid: false,
            error: opts.fieldName " cannot exceed " opts.max
        }
    }

    return {valid: true, value: converted, isEmpty: false}
}

; Usage examples
result := ValidateNumber("42", {
    type: "integer",
    min: 1,
    max: 100,
    fieldName: "Age"
})

if result.valid {
    age := result.value
    MsgBox "Valid age: " age
} else {
    MsgBox result.error
}
```

## Pattern 4: Boolean Conversion

Safely convert string values to boolean:

```ahk
ToBool(value) {
    if (value = "" || value = 0 || value = "0")
        return false

    lower := StrLower(Trim(value))

    if (lower = "true" || lower = "yes" || lower = "on" || lower = "1")
        return true

    if (lower = "false" || lower = "no" || lower = "off" || lower = "0")
        return false

    throw ValueError("Cannot convert to boolean: " value)
}

; Usage
try {
    enabled := ToBool(configValue)
    autoSave := ToBool(settingValue)
} catch ValueError as e {
    MsgBox "Invalid boolean value: " e.Message
}
```

## Pattern 5: GUI Input Validation with Live Feedback

Real-time validation in GUI applications:

```ahk
#Requires AutoHotkey v2.0

CreateValidatedInput() {
    gui := Gui()
    gui.Add("Text", , "Enter positive integer (1-1000):")

    ; Input field
    edit := gui.Add("Edit", "vUserInput w200")
    edit.OnEvent("Change", (*) => ValidateInput())

    ; Status indicator
    status := gui.Add("Text", "w200 h40")

    ; Submit button
    btn := gui.Add("Button", "Default", "Submit")
    btn.OnEvent("Click", (*) => Submit())

    gui.Show()

    ValidateInput() {
        value := edit.Value

        ; Empty check
        if (value = "") {
            status.Text := "⚠️ Required field"
            status.Opt("cOrange")
            btn.Enabled := false
            return
        }

        ; Numeric check
        if !IsNumber(value) {
            status.Text := "❌ Must be a number"
            status.Opt("cRed")
            btn.Enabled := false
            return
        }

        ; Range check
        try {
            num := Integer(value)
            if (num < 1) {
                status.Text := "❌ Must be at least 1"
                status.Opt("cRed")
                btn.Enabled := false
            } else if (num > 1000) {
                status.Text := "❌ Must be at most 1000"
                status.Opt("cRed")
                btn.Enabled := false
            } else {
                status.Text := "✅ Valid"
                status.Opt("cGreen")
                btn.Enabled := true
            }
        } catch {
            status.Text := "❌ Invalid integer"
            status.Opt("cRed")
            btn.Enabled := false
        }
    }

    Submit() {
        if (status.Text = "✅ Valid") {
            result := Integer(edit.Value)
            MsgBox "Submitted: " result
            gui.Destroy()
        }
    }
}

CreateValidatedInput()
```

## Pattern 6: Configuration File Validation

Type-safe config loader with schema validation:

```ahk
class Config {
    static Load(filePath) {
        config := Map()
        errors := []

        ; Define schema with types and constraints
        schema := Map(
            "Server.Port", {type: "integer", min: 1, max: 65535},
            "Server.Timeout", {type: "float", min: 0.1, max: 300.0},
            "Server.MaxConnections", {type: "integer", min: 1, max: 10000},
            "Server.Enabled", {type: "boolean"},
            "Database.Host", {type: "string", required: true},
            "Database.Port", {type: "integer", min: 1, max: 65535}
        )

        ; Read and validate each config value
        for key, rules in schema {
            parts := StrSplit(key, ".")
            section := parts[1]
            name := parts[2]

            ; Read raw value
            try {
                rawValue := IniRead(filePath, section, name, "")
            } catch {
                if rules.HasProp("required") && rules.required {
                    errors.Push("Missing required: " key)
                }
                continue
            }

            ; Validate based on type
            result := this.ValidateValue(key, rawValue, rules)
            if !result.valid {
                errors.Push(key ": " result.error)
                continue
            }

            config[key] := result.value
        }

        if (errors.Length > 0) {
            errorMsg := "Config validation failed:`n" errors.Join("`n")
            throw ValueError(errorMsg)
        }

        return config
    }

    static ValidateValue(key, rawValue, rules) {
        ; Check required
        if (rawValue = "") {
            if rules.HasProp("required") && rules.required {
                return {valid: false, error: "Required field is empty"}
            }
            return {valid: true, value: ""}
        }

        ; Validate by type
        switch rules.type {
            case "integer":
                if !IsInteger(rawValue) {
                    return {valid: false, error: "Must be integer"}
                }
                value := Integer(rawValue)
                if rules.HasProp("min") && value < rules.min {
                    return {valid: false, error: "Must be >= " rules.min}
                }
                if rules.HasProp("max") && value > rules.max {
                    return {valid: false, error: "Must be <= " rules.max}
                }
                return {valid: true, value: value}

            case "float":
                if !IsNumber(rawValue) {
                    return {valid: false, error: "Must be numeric"}
                }
                value := Float(rawValue)
                if rules.HasProp("min") && value < rules.min {
                    return {valid: false, error: "Must be >= " rules.min}
                }
                if rules.HasProp("max") && value > rules.max {
                    return {valid: false, error: "Must be <= " rules.max}
                }
                return {valid: true, value: value}

            case "boolean":
                try {
                    value := ToBool(rawValue)
                    return {valid: true, value: value}
                } catch {
                    return {valid: false, error: "Must be true/false or 1/0"}
                }

            case "string":
                return {valid: true, value: rawValue}
        }

        return {valid: false, error: "Unknown type: " rules.type}
    }
}

; Usage
try {
    config := Config.Load("app.ini")
    port := config["Server.Port"]
    MsgBox "Loaded config successfully. Port: " port
} catch Error as e {
    MsgBox "Config error: " e.Message
    ExitApp
}
```

## Pattern 7: CSV Data Validation

Process CSV with comprehensive error reporting:

```ahk
ProcessCSV(filePath) {
    rows := []
    errors := []

    Loop Read filePath {
        lineNum := A_Index

        ; Skip header
        if (lineNum = 1) {
            continue
        }

        parts := StrSplit(A_LoopReadLine, ",")

        ; Validate structure
        if (parts.Length < 3) {
            errors.Push({
                line: lineNum,
                type: "STRUCTURE",
                message: "Expected 3 columns, got " parts.Length
            })
            continue
        }

        ; Parse fields
        id := Trim(parts[1])
        amount := Trim(parts[2])
        name := Trim(parts[3])

        ; Validate ID (integer)
        if !IsInteger(id) {
            errors.Push({
                line: lineNum,
                type: "ID",
                message: "ID must be integer: '" id "'"
            })
            continue
        }

        ; Validate amount (positive number)
        if !IsNumber(amount) {
            errors.Push({
                line: lineNum,
                type: "AMOUNT",
                message: "Amount must be numeric: '" amount "'"
            })
            continue
        }
        amountNum := Float(amount)
        if (amountNum <= 0) {
            errors.Push({
                line: lineNum,
                type: "AMOUNT",
                message: "Amount must be positive: " amountNum
            })
            continue
        }

        ; Validate name (non-empty)
        if (name = "") {
            errors.Push({
                line: lineNum,
                type: "NAME",
                message: "Name cannot be empty"
            })
            continue
        }

        ; All valid - add row
        rows.Push({
            id: Integer(id),
            amount: amountNum,
            name: name
        })
    }

    return {
        success: true,
        validRows: rows,
        errorCount: errors.Length,
        errors: errors
    }
}

; Usage
result := ProcessCSV("data.csv")
if (result.errorCount > 0) {
    errorMsg := "Found " result.errorCount " errors:`n`n"
    for error in result.errors {
        errorMsg .= "Line " error.line ": " error.message "`n"
    }
    MsgBox errorMsg
}
MsgBox "Successfully processed " result.validRows.Length " rows"
```

## Common Edge Cases

### The String "0" Problem

```ahk
; String "0" is FALSY in v2!
count := "0"
if (count) {
    MsgBox "Never shown"  ; "0" is false!
}

; ✅ Solution 1: Check for empty string explicitly
if (count != "") {
    MsgBox "Shown correctly"
}

; ✅ Solution 2: Check length
if (StrLen(count) > 0) {
    MsgBox "Shown correctly"
}

; ✅ Solution 3: Convert first, then check
if IsNumber(count) {
    num := Integer(count)
    ; Now you can safely check if num = 0
}
```

### Leading Zeros

```ahk
; Leading zeros affect comparison mode
0 = "0"    ; true (numeric comparison)
0 = "00"   ; false! (string comparison because of extra zero)

; ✅ Solution: Explicit conversion
Integer("00") = 0  ; true
```

### Empty String Math

```ahk
; Empty strings throw TypeError in v2 (unlike v1)
result := "" + 5  ; TypeError!

; ✅ Solution: Validate before math
if IsNumber(input) && input != "" {
    result := Float(input) + 5
}
```

## Troubleshooting

### IsNumber returns true for invalid values

**Problem:** IsNumber accepts hex (0xFF) and scientific notation (1e4).

**Solution:** For strict decimal validation:

```ahk
IsStrictDecimal(value) {
    if (value = "")
        return false

    ; Only allow digits, optional sign, one decimal point
    if !RegExMatch(value, "^-?\d+(\.\d+)?$")
        return false

    return IsNumber(value)
}
```

### Validation accepts negative values

**Problem:** Forgot to check for positive constraint.

**Solution:** Use ValidateNumber with `positive: true` or explicit `min: 0`.

### Float precision issues

**Problem:** Float comparisons fail due to precision.

```ahk
result := 0.1 + 0.2  ; 0.30000000000000004 (not exactly 0.3!)

; ✅ Solution: Compare with tolerance
FloatEquals(a, b, tolerance := 0.0001) {
    return Abs(a - b) < tolerance
}

if FloatEquals(result, 0.3) {
    MsgBox "Equal within tolerance"
}
```

## Performance Considerations

**Type checking overhead:**
- IsNumber, IsInteger, IsFloat: ~1 microsecond each
- Negligible for typical validation scenarios
- Even in loops of 10,000 iterations: ~10ms total

**Best practices:**
- Validate at input boundaries (user input, file read, API response)
- Don't re-validate internal calculations
- Convert once, use many times
- Cache validation results for repeated use

## Related Concepts

- docs/source-reports/ahk2/ahk2_lang_01_types.md - Complete type system guide
- docs/manuals/ahk2/reference/type-system.md - Type coercion rules
- docs/manuals/ahk2/how-to/custom-error-handler.md - Error handling patterns

## Key Takeaway

Type validation transforms brittle scripts into robust applications. Always validate before converting, provide clear error messages, handle edge cases (empty strings, "0", leading zeros), and use the ValidateNumber pattern for reusable validation. The Config class demonstrates production-ready schema-based validation that catches errors early and provides actionable feedback.
