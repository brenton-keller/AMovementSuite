# AHK2 Research Request: Security, Permissions & UAC

## What I Need

My script can't interact with admin windows. I don't understand UAC, integrity levels, or security boundaries. I need to know how to work within Windows security model.

## Current Understanding (Challenge!)

- UAC windows block all input
- Running script as admin solves everything
- Non-admin process can never interact with admin process
- File/registry operations just work or fail

**What's more nuanced than I think?**

## The Problems

**Problem 1: Can't move admin windows**
```ahk
WinMove x, y, , , "ahk_id " hwnd  // Fails silently if hwnd is elevated
// How do I detect this? How do I handle it?
```

**Problem 2: Registry access denied**
```ahk
RegWrite value, "HKLM\Software\..."  // Needs admin
// How to request elevation for just this operation?
```

**Problem 3: Cross-privilege communication**
```ahk
// Script A (non-admin) needs to tell Script B (admin) to do something
// How do they communicate across security boundary?
```

## Questions

1. **Integrity Levels**: What are low/medium/high/system levels? How to check my level?

2. **Elevation**: How to elevate script at runtime? How to run single operation as admin?

3. **UIPI**: What is User Interface Privilege Isolation? Why does it block input?

4. **Detection**: How to detect if window/process is elevated? If operation needs admin?

5. **Workarounds**: How to interact with admin processes from non-admin script (if possible)?

## Guide Should Cover

- UAC and integrity levels explained
- How to detect elevation needs
- Requesting elevation properly
- Security boundaries and limitations
- Cross-privilege communication patterns
- File/registry permissions
- Complete example: Smart elevation system

## Success

1. ✓ Detect which operations need admin
2. ✓ Request elevation only when needed
3. ✓ Handle elevation denial gracefully
4. ✓ Never fail silently due to permissions
5. ✓ Communicate across security boundaries safely

## Most Important

Understand Windows security model deeply enough to **design around it** rather than fight it.

I want to build software that works correctly at any privilege level and handles elevation elegantly.
