# Description: Code cleanup with hybrid deletion/archiving approach

Removes dead code while keeping codebase clean. Trusts git history over keeping old code.

**Usage:**
- `/cleanup` - Scan for cleanup candidates, ask before removing
- `/cleanup [file/directory]` - Focus cleanup on specific location
- `/cleanup aggressive` - More thorough scan including commented code

---

## Cleanup Philosophy

**Keep It Clean:**
- Delete trivial clutter (unused imports, debug statements)
- Ask before removing significant code (functions, features)
- Trust git history - don't keep old code "just in case"
- Project-dependent - adapt to project needs

**When to Delete vs Ask:**

**Auto-delete (trivial):**
- Unused imports/includes
- Console.log / debug print statements
- TODO comments older than 6 months
- Trailing whitespace
- Commented-out single lines

**Ask first (significant):**
- Unused functions/classes
- Commented-out blocks (>5 lines)
- Old feature implementations
- Complete unused files

**Never delete without asking:**
- Anything in production code that might be referenced
- Configuration files (even if seemingly unused)
- Test files (even if outdated)

---

## Process

1. **Safety check:**
   - Verify we're in a git repository
   - Warn if uncommitted changes exist
   - Offer to commit first

2. **Scan for cleanup candidates:**

   **Trivial (auto-remove):**
   - Unused imports/includes
   - Debug logging statements
   - Old TODO comments (>6 months old)
   - Trailing whitespace, multiple blank lines

   **Significant (ask first):**
   - Functions not referenced anywhere
   - Commented-out code blocks
   - Files with no imports/references
   - Duplicate code (offer to consolidate)

3. **Categorize findings:**
   ```
   Found cleanup opportunities:

   Trivial (will auto-remove):
   - 5 unused imports
   - 12 debug statements
   - 3 old TODO comments

   Significant (need approval):
   - CalculateOldColor() function (not referenced)
   - 20-line commented block in WindowMove.ahk2
   - archive/old_hotkeys.ahk2 (no references)
   ```

4. **Get approval:**
   - Show what will be auto-deleted
   - Ask about significant items one by one
   - Offer to show code before deleting

5. **Execute cleanup:**
   - Remove approved items
   - Create cleanup commit with detailed list
   - Show git diff for review

6. **Post-cleanup:**
   - Run tests if they exist
   - Verify project still works
   - Commit cleanup

---

## Cleanup Categories

### 1. Unused Imports/Includes
**Auto-delete if:** Import/include not referenced in file

**Language adaptation:**
- Python: Remove unused imports
- JavaScript: Remove unused imports/requires
- AHK: Remove unused #Include statements

**Safety:** Never remove imports that might have side effects (initialization code)

---

### 2. Debug Statements
**Auto-delete if:** Clear debug code with no production value

**Examples:**
```ahk
; Auto-remove these:
MsgBox "Debug: " value
ToolTip "Test"
OutputDebug "Checkpoint"

; Keep these (not obviously debug):
MsgBox "Error: Operation failed"  ; User-facing error
```

**Language adaptation:**
- Python: Remove print() debugging, pdb statements
- JavaScript: Remove console.log (keep console.error for production)
- AHK: Remove debug ToolTips, MsgBoxes, OutputDebug

---

### 3. Old Comments
**Auto-delete if:**
- TODO older than 6 months with no activity
- Commented-out single lines
- Obvious noise comments ("; Debug", "; Test")

**Ask first:**
- Commented blocks >5 lines
- TODOs with context explaining why deferred

---

### 4. Unused Functions/Classes
**Always ask:**
- Show function/class code
- Show why it appears unused
- Confirm deletion

**Detection:**
- Grep for function name across codebase
- Check for indirect references (reflection, eval, etc.)
- Warn if public API (might be used externally)

---

### 5. Dead Files
**Always ask:**
- Show file purpose
- Show last modified date
- Check git history (if untouched for >6 months, likely dead)

**Safety:** Never auto-delete complete files

---

## Example Workflow

```
You: /cleanup
Claude:

Scanning codebase for cleanup opportunities...

Found in src/Features/WindowScaleXY.ahk2:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

TRIVIAL (auto-remove with your approval):
  - Line 45: OutputDebug "Frame changed"
  - Line 67: OutputDebug "Motion detected"
  - Line 120: TODO: Optimize this (added 8 months ago)

SIGNIFICANT (ask individually):
  - Line 200-215: Commented block (old color calculation)
  - CalculateOldColorXY() function (no references found)

────────────────────────────────────────

Auto-remove trivial items? [Yes/No/Show me first]

> Yes

Removing 3 trivial items...

Now for significant items:

1. Commented block (lines 200-215):
   ; Old calculation method before optimization
   ; newColor := ...
   ; [15 more lines]

This block is 8 months old. Delete? [Yes/No/Show full code]

> Yes

2. CalculateOldColorXY() function:
   No references found in codebase.
   Last modified: 6 months ago

Delete this function? [Yes/No/Show code]

> Show code

[Shows function code...]

Delete? [Yes/No]

> Yes

────────────────────────────────────────

Cleanup complete! Removed:
- 3 debug statements
- 1 old TODO
- 1 commented block (15 lines)
- 1 unused function (CalculateOldColorXY)

Create cleanup commit? [Yes/No]

> Yes

git add src/Features/WindowScaleXY.ahk2
git commit -m "Clean up WindowScaleXY: remove debug statements and unused code"

Done! Run tests to verify nothing broke.
```

---

## Safety Features

1. **Git repository required:**
   - Won't run outside git repo
   - Can recover via `git log` if needed

2. **Uncommitted changes warning:**
   - Warns if working directory dirty
   - Offers to commit first

3. **Dry run available:**
   - Can show what would be removed without doing it
   - Preview changes before applying

4. **Detailed commit:**
   - Cleanup commit lists exactly what was removed
   - Easy to revert if needed

---

## Language Adaptation

**Python:**
- Remove unused imports (via AST analysis)
- Remove print() debugging
- Remove pass statements in empty functions
- Fix PEP8 spacing issues

**JavaScript:**
- Remove unused imports/requires
- Remove console.log debugging
- Remove debugger statements
- Fix ESLint auto-fixable issues

**AHK (this project):**
- Remove unused #Include
- Remove debug MsgBox/ToolTip
- Remove old TODO comments
- Clean up whitespace

---

## Project-Dependent Adaptation

The command adapts to project style:

**Aggressive projects:**
- More auto-deletion
- Less asking
- Trust that git history is enough

**Conservative projects:**
- Ask about everything
- Prefer commenting over deleting
- Keep more "just in case"

**Detection:**
- Checks if project has existing archive/ folder
- Reads contributing docs for cleanup guidance
- Asks user preference on first run

---

## Key Principles

1. **Trust git history** - Don't keep old code "just in case"
2. **Ask before major changes** - Don't silently delete functions
3. **Keep it clean** - Remove clutter proactively
4. **Safety first** - Require git, warn about uncommitted changes
5. **Detailed commits** - Document exactly what was cleaned up
