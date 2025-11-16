# AHK2 Research Request: File I/O & Data Persistence

## What I Need to Learn

My application has ZERO file I/O - settings don't persist, window groups are lost on restart, no configuration files exist. I need to understand file operations and data persistence patterns in AHK2.

## My Current Understanding (Challenge This!)

I believe:
- FileRead loads entire file into memory
- FileAppend is how you write files
- INI files are the standard config format for AHK2
- JSON support requires external library
- File encoding is automatic (UTF-8 default)

**What's wrong here?**

## The Core Problem

Current state:
```ahk
global windowGroup := {window1: 0, window2: 0, offsetX: 0, offsetY: 0}
// When app closes, this is lost forever
// User has to re-group windows every time
```

What I need:
```ahk
// On shutdown:
SaveWindowGroups()  // Persist to file

// On startup:
LoadWindowGroups()  // Restore from file

// Must handle:
// - First run (no file exists)
// - Corrupted file (invalid JSON/INI)
// - Partial writes (crash during save)
```

## Specific Questions

1. **FileRead vs FileOpen**: When do I use each? What's the difference?

2. **Encodings**: UTF-8 vs UTF-16 vs ANSI - which should I use? How do I specify?

3. **Atomic Writes**: How do I prevent corrupted files if app crashes during save? Write to temp + rename?

4. **INI vs JSON**: Pros/cons of each? Does AHK2 have built-in JSON support?

5. **File Locking**: What if another process is writing the same file? How do I handle?

6. **Large Files**: FileRead loads all into memory - what if file is huge? Streaming?

## What I'm Building

Complete configuration system:
```ahk
class Config {
    // Load from file (create default if missing)
    // Validate structure (handle corrupted data)
    // Merge defaults with user settings
    // Save atomically (prevent corruption)
    // Watch for external changes (reload if edited)
}
```

## How to Write This Guide

### 1. File Operations Comparison

Side-by-side all methods:
```ahk
// Method 1: FileRead (simplest)
content := FileRead("file.txt")  // Entire file in memory

// Method 2: FileOpen + Read (control)
file := FileOpen("file.txt", "r")
content := file.Read()
file.Close()

// Method 3: Loop Read (line-by-line)
Loop Read "file.txt" {
    ProcessLine(A_LoopReadLine)
}

// When to use each? Memory implications? Performance?
```

### 2. Encoding Deep Dive

Test different encodings:
```ahk
; Write with different encodings
FileAppend("Test", "utf8.txt", "UTF-8")
FileAppend("Test", "utf16.txt", "UTF-16")
FileAppend("Test", "ansi.txt", "CP0")

; What happens if you read with wrong encoding?
// Show the actual garbled output
// How to detect encoding?
// When does it matter?
```

### 3. The Corruption Problem

Show the problem:
```ahk
; User data file
SaveSettings() {
    data := SerializeSettings()  // 10KB of data
    FileDelete("settings.json")
    FileAppend(data, "settings.json")
    // ⚠️ If crash happens here, file is empty!
}
```

Then solve it progressively:

**Attempt 1: Write to temp**
```ahk
// Write to temp, then rename (atomic)
// But what if rename fails?
```

**Attempt 2: Backup before write**
```ahk
// Keep backup, restore on corruption
// But how to detect corruption?
```

**Final: Production-ready**
```ahk
class AtomicFileWriter {
    // Complete implementation with:
    // - Temp file write
    // - Verification before commit
    // - Automatic backup
    // - Rollback on failure
    // - Corruption detection
}
```

### 4. INI vs JSON Deep Comparison

| Feature | INI | JSON |
|---------|-----|------|
| Built-in support | ✓ IniRead/Write | ? (how to parse) |
| Nested data | ✗ Flat only | ✓ Arbitrary nesting |
| Arrays | ✗ | ✓ |
| Human-editable | ✓✓ Very easy | ✓ Moderate |
| Comments | ✓ | ✗ |
| Type preservation | ✗ All strings | ✓ Numbers, bools |

Then show complete example of each for same data.

### 5. Configuration Patterns

**Pattern 1: Simple INI**
```ahk
; Write
IniWrite(value, "config.ini", "Section", "Key")

; Read with default
value := IniRead("config.ini", "Section", "Key", "default")

; Problem: What about validation?
```

**Pattern 2: JSON with validation**
```ahk
; How to serialize AHK2 object to JSON?
; How to parse JSON back to object?
; How to validate schema?
```

**Pattern 3: Complete config system**
```ahk
class Configuration {
    static Load() {
        // 1. Check if file exists
        // 2. Read and parse
        // 3. Validate structure
        // 4. Merge with defaults
        // 5. Return config object
        // Handle each failure case
    }

    static Save(config) {
        // 1. Validate before writing
        // 2. Atomic write to prevent corruption
        // 3. Verify written data
        // 4. Backup old version
    }
}
```

### 6. Real-World File Operations

Show complete examples:

**Append to log file (thread-safe)**
```ahk
// Problem: Multiple instances writing same log
// Solution: File locking? Separate files? Queue?
```

**Read large file (streaming)**
```ahk
// File is 500MB, can't load all into memory
// Solution: Loop Read line-by-line
// What about binary files?
```

**Configuration with watchers**
```ahk
// Watch config file for external changes
// Reload when file modified
// How to detect changes without polling?
```

### 7. Performance Benchmarks

Test operations:
```
Write 1000 lines:
- FileAppend in loop: ???ms
- Build string, single FileAppend: ???ms
- FileOpen, Write, Close: ???ms

Read 1000 lines:
- FileRead (all at once): ???ms
- Loop Read (line by line): ???ms
- FileOpen + Read: ???ms

Conclusion: Best practice for each scenario?
```

### 8. The Gotchas

**"File disappeared!"**
> Used FileDelete before FileAppend - crash between calls = data loss

**"Encoding nightmare"**
> Saved as UTF-16, read as UTF-8 - got gibberish with special characters

**"File locked error"**
> Another process had file open - FileRead threw OSError, didn't handle it

**"Settings corrupted"**
> Power loss during write - file half-written, JSON invalid on next load

## Success Criteria

After this guide:
1. ✓ Implement persistent settings (survive restarts)
2. ✓ Handle corrupted files gracefully
3. ✓ Choose right encoding for data
4. ✓ Write files atomically (corruption-proof)
5. ✓ Parse and generate JSON in AHK2
6. ✓ Handle file errors without losing data

## Most Important

I want to understand file I/O at a **systems level** - how do I ensure data integrity, handle errors gracefully, and make configuration that "just works" for users?

Not just "how to call FileRead" but "how to build robust persistence that survives crashes, power loss, and user mistakes."
