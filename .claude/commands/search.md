# Description: Intelligent codebase search with structured results

Semantic search that understands code patterns, not just text matching.

**Usage:**
- `/search where is window snapping implemented`
- `/search how does color calculation work`
- `/search all files that use GlobalConfig.SnapDistance`
- `/search --quick timer usage` - Fast search
- `/search --thorough preview system` - Deep search

---

## Differences from Manual Search

**Manual approach:**
- You ask me to search
- I use Grep/Glob or Task/Explore agents
- Results vary in format each time

**`/search` command:**
- Consistent structured output every time
- Always uses Explore agent for thorough searching
- Presents results with context and suggestions
- Thoroughness level explicit (quick/medium/thorough)

**Advantage:** Consistency - you get the same quality results without re-explaining what you want

---

## Search Process

1. **Detect search type:**
   - **"where is X"** → Find implementation location
   - **"how does X work"** → Find + explain pattern
   - **"all files using X"** → Find all references
   - **"X pattern"** → Find similar patterns across codebase

2. **Determine thoroughness:**
   - `--quick`: Basic file search, fast results
   - `--medium` (default): Moderate exploration
   - `--thorough`: Deep analysis across multiple locations

3. **Execute search:**
   - Use Explore agent with specified thoroughness
   - Search for semantic matches, not just literal strings
   - Understand code context (functions, classes, imports)

4. **Structure results:**
   ```
   Found in X locations:

   1. file.ext:line-range - Primary implementation
      [Brief explanation]

   2. file.ext:line-range - Related usage
      [Brief explanation]

   Pattern: [Common pattern across findings]

   Related files worth checking:
   - file1.ext (similar implementation)
   - file2.ext (uses this pattern)

   Want details on any of these? [1/2/all/none]
   ```

---

## Search Types

### 1. Location Search ("where is...")
**Query:** `/search where is window snapping implemented`

**Process:**
- Find primary implementation
- Find configuration/settings related to it
- Find usage examples

**Result format:**
```
Window snapping implemented in:

Primary: src/Features/WindowMove.ahk2:45-120
- Main snapping logic in WindowMove feature
- Uses snapDistance threshold from GlobalConfig
- Checks monitor edges and existing windows

Configuration: src/lib/Config/GlobalConfig.ahk2:12
- snapDistance := 50 (default threshold)

Related: src/lib/Config/UserConfig.ahk2:20
- User can override snap distance in settings
```

---

### 2. Pattern Search ("how does...")
**Query:** `/search how does the preview system work`

**Process:**
- Find implementation
- Explain pattern/approach
- Show related code

**Result format:**
```
Preview system pattern:

Core implementation: src/Features/WindowScaleXY.ahk2:150-200
- Creates transparent Gui overlay
- Updates color based on scale ratio
- Hides/shows based on motion detection

Pattern used:
1. Create Gui with transparent background
2. Set WinSetTransparent for overlay effect
3. Update position with WinMove in loop
4. Destroy on operation complete

This pattern is replicated in:
- WindowScaleWidth.ahk2 (width preview)
- WindowScaleHeight.ahk2 (height preview)

Color calculation: src/lib/Core/ColorUtils.ahk2:30-50
- CalculateColorXY() interpolates green/yellow/red
```

---

### 3. Reference Search ("all files using...")
**Query:** `/search all files that use GlobalConfig.SnapDistance`

**Process:**
- Find all references
- Categorize by usage type
- Show context for each

**Result format:**
```
GlobalConfig.SnapDistance used in 3 files:

1. src/lib/Config/GlobalConfig.ahk2:12 (definition)
   global snapDistance := 50

2. src/Features/WindowMove.ahk2:67 (usage)
   if (Abs(currentX - edgeX) < snapDistance)

3. src/UI/SettingsGUI.ahk2:35 (configuration UI)
   Edit for user to change snap distance
```

---

### 4. Pattern Search Across Files
**Query:** `/search timer-based update pattern`

**Process:**
- Find similar patterns across files
- Identify common approach
- Show variations

---

## Thoroughness Levels

**Quick (--quick):**
- Basic file/function search
- Glob patterns + simple grep
- Fast results, less context

**Medium (default):**
- Explore agent with moderate depth
- Checks main locations + related files
- Balanced speed/thoroughness

**Thorough (--thorough):**
- Deep Explore agent analysis
- Multiple naming conventions
- Cross-reference checking
- Slower but comprehensive

---

## Language Adaptation

Search understands language-specific patterns:

**Python:**
- Imports, classes, decorators
- Finds implementations across modules
- Understands inheritance patterns

**JavaScript:**
- Imports/exports, classes, hooks
- Component hierarchies
- Context/state patterns

**AHK (this project):**
- #Include statements
- Global variables and functions
- Hotkey definitions

---

## Example Workflow

```
You: /search where is motion detection implemented

Claude:

Searching codebase for motion detection...

Found in 3 locations:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. src/Features/WindowScaleXY.ahk2:180-195
   Primary implementation - Motion pause detection

   Tracks lastFrameChangeTime to detect when user pauses during resize.
   Uses MOTION_PAUSE_THRESHOLD to determine stationary period.

2. src/Features/WindowScaleWidth.ahk2:210-220
   Same pattern - Width resize motion detection

   Replicates XY approach for width-only resizing.

3. src/lib/Config/GlobalConfig.ahk2:19
   Configuration constant

   global MOTION_PAUSE_THRESHOLD := 55  ; milliseconds

Pattern observed:
All scale features use identical motion detection:
- Track A_TickCount on frame changes
- Calculate time since last change
- Compare against MOTION_PAUSE_THRESHOLD
- Hide preview when stationary + conditions met

Related configuration:
- MOTION_PAUSE_THRESHOLD controls sensitivity

Want to explore any of these in detail? [1/2/3/pattern/config/none]
```

---

## Key Principles

1. **Semantic search** - Understands code concepts, not just text
2. **Structured results** - Same format every time
3. **Context-aware** - Shows related code and patterns
4. **Thoroughness control** - Fast or deep search as needed
5. **Actionable** - Suggests next steps and related files
