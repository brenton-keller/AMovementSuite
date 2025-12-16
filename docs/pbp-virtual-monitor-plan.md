# PBP (Picture-by-Picture) Virtual Monitor Implementation Plan

## Goal
Enable a single physical monitor (5120x2160) to be treated as two virtual monitors for window management purposes.

**Initial Implementation:** Hardcoded 75/25 split (3840x2160 + 1280x2160)
**Future:** Configurable split position via GUI

## Current State Analysis

### How Monitor Detection Works Now
- `MonitorGetCount()` - Returns physical monitor count (currently 1)
- `MonitorGet(index, &Left, &Top, &Right, &Bottom)` - Gets physical monitor bounds
- `MonitorGetWorkArea(index, ...)` - Gets work area (minus taskbar)
- `GetWindowMonitor(x, y, w, h)` - Determines which monitor contains a window (by center point)
- `MonitorFromPoint(x, y)` - Determines which monitor contains a point

### Current Window Management Features
All features rely on monitor detection:
- **WindowMove.ahk2** - Window positioning, snapping, grid placement
- **WindowScaleWidth.ahk2** - Horizontal resizing with monitor bounds
- **WindowScaleHeight.ahk2** - Vertical resizing with monitor bounds
- **WindowScaleXY.ahk2** - Proportional scaling
- **WindowCascade.ahk2** - Window cascading within monitor bounds
- **WindowGrouping.ahk2** - Group movement across monitors

### Key Functions Using Monitor APIs
- `GetWindowMonitor()` in CoreUtils.ahk2:142 - Used by all move/resize features
- `MonitorFromPoint()` in MoveUtils.ahk2:64 - Used for point-to-monitor detection
- `ProcessScreenSnapping()` in MoveUtils.ahk2:458 - Snaps to screen edges
- `ProcessWindowSnapping()` in MoveUtils.ahk2:272 - Snaps to other windows
- Window positioning GUI in MoveUtils.ahk2:813 - Shows positioning options

## Proposed Solution

### Virtual Monitor Abstraction Layer

Create a **virtual monitor management system** that sits between the physical monitor APIs and the rest of the codebase.

#### Core Concept
When PBP mode is **disabled**:
- Virtual monitor functions pass through to physical monitor APIs
- Behavior is identical to current implementation

When PBP mode is **enabled**:
- Single physical monitor is treated as N virtual monitors
- Virtual monitor bounds are calculated based on configuration
- All window management operations use virtual monitor coordinates

### Architecture Changes

#### 1. New Module: `VirtualMonitorManager.ahk2`
Location: `src/lib/Core/VirtualMonitorManager.ahk2`

**Responsibilities:**
- Store PBP configuration (enabled/disabled, split positions)
- Provide virtual monitor API that mimics physical monitor API
- Translate between virtual and physical coordinate spaces

**Core Functions:**
```ahk
; Configuration
EnablePBPMode(enabled := true)
SetPBPSplit(splitX)  ; e.g., 3120 for left monitor width
GetPBPSplitX()

; Virtual Monitor API (replaces physical APIs)
VirtualMonitorGetCount()  ; Returns 2 when PBP enabled, else physical count
VirtualMonitorGet(index, &Left, &Top, &Right, &Bottom)
VirtualMonitorGetWorkArea(index, &Left, &Top, &Right, &Bottom)
GetVirtualMonitorFromPoint(x, y)  ; Returns virtual monitor containing point
GetVirtualMonitorForWindow(x, y, w, h)  ; Returns virtual monitor for window

; Utility
IsPointInVirtualMonitor(x, y, monitorIndex)
GetVirtualMonitorBounds(monitorIndex)  ; Returns map with left/top/right/bottom
GetPhysicalMonitorForVirtual(virtualIndex)  ; Which physical monitor contains virtual
```

#### 2. Configuration Updates
Location: `src/lib/Config/GlobalConfig.ahk2`

**New Settings:**
```ahk
; PBP Virtual Monitor Settings
global pbpEnabled := false
global pbpSplitX := 3840  ; 75% split (3840x2160 + 1280x2160)
global pbpSplitPercent := 75.0  ; Alternative: percentage-based split
global pbpMonitorIndex := 1  ; Which physical monitor to split (future: multi-monitor support)
```

**Hotkey:**
- `ScrollLock` - Toggle PBP mode on/off

#### 3. Refactor Existing Code

**Replace Direct Monitor API Calls:**
All files currently using:
- `MonitorGetCount()` → `VirtualMonitorGetCount()`
- `MonitorGet()` → `VirtualMonitorGet()`
- `MonitorGetWorkArea()` → `VirtualMonitorGetWorkArea()`
- `GetWindowMonitor()` → `GetVirtualMonitorForWindow()`
- `MonitorFromPoint()` → `GetVirtualMonitorFromPoint()`

**Files to Update:**
- ✓ `src/lib/Core/CoreUtils.ahk2` - Update `GetWindowMonitor()`
- ✓ `src/lib/Core/MoveUtils.ahk2` - Update `MonitorFromPoint()`, snapping functions
- ✓ `src/Features/WindowMove.ahk2` - Update monitor detection
- ✓ `src/Features/WindowScaleWidth.ahk2` - Update bounds calculations
- ✓ `src/Features/WindowScaleHeight.ahk2` - Update bounds calculations
- ✓ `src/Features/WindowScaleXY.ahk2` - Update monitor detection
- ✓ `src/Features/WindowGrouping.ahk2` - Update monitor boundaries
- ✓ `src/Features/WindowCascade.ahk2` - Update cascade bounds
- ✓ `AWindowMovementSuite.ahk2` - Update monitor info display, add PBP toggle

#### 4. UI for PBP Configuration

**Tray Menu Addition:**
```
Window Move Settings
Program Paths Configuration
--- NEW SEPARATOR ---
PBP Virtual Monitor (Toggle)
```

**Initial Implementation:**
- Tray menu toggle only (no settings GUI yet)
- Hardcoded 75/25 split
- Scroll Lock hotkey to toggle

**Future GUI (Phase 4 - deferred):**
- Slider: Split Position (X coordinate or percentage)
- Display: Shows resulting virtual monitor dimensions
- Buttons: Apply, Cancel, Reset to Default

## Implementation Steps

### Phase 1: Core Infrastructure
1. **Create VirtualMonitorManager.ahk2**
   - Implement configuration storage
   - Implement virtual monitor API
   - Add coordinate translation logic
   - Write unit tests (manual testing)

2. **Update GlobalConfig.ahk2**
   - Add PBP configuration variables
   - Document new settings

3. **Include new module in main script**
   - Add `#Include` to AWindowMovementSuite.ahk2
   - Initialize PBP system on startup

### Phase 2: Refactor Core Utilities
4. **Update CoreUtils.ahk2**
   - Replace `GetWindowMonitor()` to use virtual monitors
   - Test with PBP enabled/disabled

5. **Update MoveUtils.ahk2**
   - Replace `MonitorFromPoint()` to use virtual monitors
   - Update snapping functions for virtual monitor edges
   - Update `ShowWindowPositioningGUI()` for virtual monitors

### Phase 3: Update Features
6. **Update WindowMove.ahk2**
   - Use virtual monitor APIs
   - Test window dragging between virtual monitors
   - Test snapping to virtual monitor edges

7. **Update WindowScale*.ahk2**
   - WindowScaleWidth: Use virtual monitor bounds
   - WindowScaleHeight: Use virtual monitor bounds
   - WindowScaleXY: Use virtual monitor detection

8. **Update remaining features**
   - WindowGrouping.ahk2
   - WindowCascade.ahk2

### Phase 4: UI & Configuration
9. **Update main script tray menu**
   - Add PBP toggle menu item
   - Add ScrollLock hotkey handler
   - Update monitor info display to show virtual monitors

10. **Update startup monitor display**
    - Show physical monitor info
    - Show virtual monitor info when PBP enabled

11. **Add persistence (UserConfig.ahk2)**
    - Save pbpEnabled state
    - Load on startup

### Phase 5: Testing & Documentation
12. **Comprehensive testing**
    - Test each feature with PBP enabled/disabled
    - Test split position changes
    - Test edge cases (windows spanning virtual boundary)

13. **Update documentation**
    - `docs/features.md` - Add PBP Virtual Monitor section
    - `docs/configuration.md` - Document PBP settings
    - `docs/claude/architecture-overview.md` - Add VirtualMonitorManager
    - `CLAUDE.md` - Update if needed
    - Create `docs/pbp-virtual-monitors.md` - Detailed technical doc

## Technical Challenges & Solutions

### Challenge 1: Window Spanning Virtual Boundary
**Problem:** A window could span across the virtual monitor split point.

**Solution:**
- `GetVirtualMonitorForWindow()` determines monitor by window center point
- If center is at split point, default to left/first monitor
- Future: Add "sticky" behavior to prevent windows from being half on each side

### Challenge 2: Smooth Transition When Toggling PBP
**Problem:** Windows positioned on "monitor 2" when PBP is disabled might be off-screen.

**Solution:**
- When enabling PBP: Windows stay in place (no repositioning)
- When disabling PBP: Optional - reposition windows that would be off-screen
- Store original physical monitor in window metadata (future enhancement)

### Challenge 3: Multi-Physical Monitor Support
**Problem:** User might have 2+ physical monitors and want PBP on one.

**Solution (Future):**
- `pbpMonitorIndex` in config specifies which physical monitor to split
- Virtual monitor indices: Physical monitors + virtual splits
- Example: 2 physical, PBP on monitor 1 → 3 virtual monitors (1a, 1b, 2)

### Challenge 4: Snapping Between Virtual Monitors
**Problem:** Windows should snap to the virtual split edge.

**Solution:**
- `ProcessScreenSnapping()` uses virtual monitor bounds
- Add virtual split edge to edge detection
- Treat virtual boundary same as physical monitor edge

### Challenge 5: Work Area vs Full Monitor
**Problem:** Taskbar might affect one virtual monitor differently than the other.

**Solution:**
- **Simplified approach:** Use full monitor bounds for both virtual monitors
- Ignore taskbar work area complications for now
- Both virtual monitors span full height (0 to 2160)
- Future: Can add work area calculation if needed

## Configuration Examples

### Initial Implementation: 75/25 Split
```ahk
pbpEnabled := true
pbpSplitX := 3840  ; 75% of 5120
pbpSplitPercent := 75.0
; Virtual Monitor 1: 0-3840 (3840x2160)
; Virtual Monitor 2: 3840-5120 (1280x2160)
```

### Future: Other Split Ratios
```ahk
; 50/50 Split
pbpSplitX := 2560  ; 50% of 5120
; Virtual Monitor 1: 0-2560 (2560x2160)
; Virtual Monitor 2: 2560-5120 (2560x2160)

; 60/40 Split
pbpSplitX := 3072  ; 60% of 5120
; Virtual Monitor 1: 0-3072 (3072x2160)
; Virtual Monitor 2: 3072-5120 (2048x2160)
```

## Testing Checklist

### Unit Testing (Manual)
- [ ] VirtualMonitorGetCount() returns correct values
- [ ] VirtualMonitorGet() returns correct bounds
- [ ] GetVirtualMonitorFromPoint() correctly identifies monitors
- [ ] Coordinate translation works bidirectionally
- [ ] PBP toggle updates all functions correctly

### Integration Testing
- [ ] Window dragging respects virtual monitor bounds
- [ ] Window resizing respects virtual monitor bounds
- [ ] Snapping works at virtual monitor edges
- [ ] Positioning GUI shows correct split options
- [ ] Window grouping works across virtual monitors
- [ ] Cascade works within each virtual monitor
- [ ] Virtual desktop switching maintains virtual monitor awareness

### Edge Cases
- [ ] Window at exact split point
- [ ] Toggle PBP while dragging window
- [ ] Change split position while windows are open
- [ ] Maximized windows in PBP mode
- [ ] Minimized windows in PBP mode
- [ ] Multi-window operations (group move, cascade)

## Future Enhancements

1. **Multiple Splits** - Divide monitor into 3+ virtual monitors
2. **Vertical Splits** - Split horizontally (top/bottom)
3. **Preset Layouts** - Common configurations (50/50, 60/40, 2:1, etc.)
4. **Per-Application PBP** - Different splits for different workflows
5. **Visual Split Indicator** - Overlay showing virtual monitor boundaries
6. **Window Pinning** - Keep specific windows on specific virtual monitors
7. **Smart Window Placement** - Automatically position new windows

## Complexity & Risk Assessment

**Overall Difficulty:** 3/5 (Medium)

**Phase Breakdown:**
- Phase 1 (Core Infrastructure): **3/5** - New module, abstraction layer design
- Phase 2 (Refactor Utilities): **2/5** - Straightforward function replacements
- Phase 3 (Update Features): **3/5** - Multiple files, testing each feature
- Phase 4 (UI & Config): **2/5** - Simple toggle and hotkey (no GUI initially)
- Phase 5 (Testing/Docs): **2/5** - Thorough but straightforward

**Risk Assessment:**
- **Risk Level:** Medium
- **Factors:**
  - Touches many files across codebase
  - Core architectural change (monitor detection)
  - Requires thorough testing with various features
  - **Mitigation:** Clean abstraction layer, passthrough when disabled

**Critical Success Factors:**
- VirtualMonitorManager must handle edge cases correctly
- All existing features must work identically when PBP is disabled
- Split boundary must be treated as monitor edge for snapping

## Implementation Decisions

### Confirmed Requirements
1. **Split Position:** Support both pixels and percentage (either/or)
2. **Taskbar Handling:** Ignore work area complications - use full monitor bounds
3. **Hotkey:** ScrollLock to toggle PBP mode on/off
4. **Visual Feedback:** No visual indicator
5. **Presets:** No preset GUI - start with hardcoded 75/25 split
6. **Persistence:** Yes - save pbpEnabled state to UserConfig

### Initial Scope (MVP)
- Hardcoded 75/25 split (3840px + 1280px)
- ScrollLock toggle
- Tray menu toggle
- Persistence of enabled state
- Full monitor bounds (ignore taskbar work area)

### Deferred Features
- Configurable split position GUI
- Visual split indicator
- Preset layouts
- Per-monitor PBP (multi-monitor support)

---

**Status:** Plan approved. Ready to implement Phase 1.
