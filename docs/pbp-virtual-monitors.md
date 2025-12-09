# PBP Virtual Monitors

## Overview

The PBP (Picture-by-Picture) Virtual Monitor system allows a single physical monitor to be treated as multiple virtual monitors for window management purposes. This is particularly useful for ultra-wide monitors where you want to manage windows as if you had multiple physical displays.

## Features

- **Virtual Monitor Splitting**: Divides a physical monitor into multiple virtual monitors
- **Seamless Integration**: All existing window management features work with virtual monitors
- **Hardcoded 75/25 Split**: Initial implementation uses a fixed 3840px / 1280px split (75% / 25%)
- **Toggle On/Off**: Enable/disable PBP mode via ScrollLock hotkey or tray menu
- **Persistent Settings**: PBP state is saved and restored between script sessions
- **Zero Impact When Disabled**: When PBP is off, everything works exactly as before

## Usage

### Enabling PBP Mode

**Method 1: Keyboard Shortcut**
- Press `ScrollLock` to toggle PBP mode on/off
- A tooltip will show the current state

**Method 2: Tray Menu**
- Right-click the system tray icon
- Click "PBP Virtual Monitor" to toggle
- Checkmark indicates PBP is enabled

### What Changes with PBP Enabled

When PBP is enabled with a 75/25 split on a 5120x2160 monitor:

**Virtual Monitor 1 (Left)**
- Bounds: X=0 to X=3840
- Resolution: 3840x2160
- Primary monitor

**Virtual Monitor 2 (Right)**
- Bounds: X=3840 to X=5120
- Resolution: 1280x2160

**All Window Management Features Now Work Across Virtual Monitors:**
- Window dragging respects virtual monitor boundaries
- Window resizing uses virtual monitor bounds
- Snapping works at the virtual monitor split edge (X=3840)
- Window positioning (Left 50%, Right 50%, etc.) applies to virtual monitors
- Window grouping can span virtual monitors
- Cascade operates within each virtual monitor

## Configuration

### Current Split Configuration

**Hardcoded 75/25 Split:**
- `pbpSplitX = 3840` (75% of 5120px)
- `pbpSplitPercent = 75.0`

**Location:** `src/lib/Config/GlobalConfig.ahk2`

### Settings Persistence

Settings are automatically saved to: `settings/pbp_settings.ini`

**Saved Settings:**
- `Enabled`: Whether PBP mode is active
- `SplitX`: Pixel position of the split
- `SplitPercent`: Percentage-based split value

## Technical Architecture

### Virtual Monitor Abstraction Layer

The system uses a virtual monitor abstraction layer (`VirtualMonitorManager.ahk2`) that sits between the application code and Windows monitor APIs.

**Key Functions:**
```ahk
VirtualMonitorGetCount()  ; Returns 2 when PBP enabled, else physical count
VirtualMonitorGet(index, &Left, &Top, &Right, &Bottom)
VirtualMonitorGetWorkArea(index, &Left, &Top, &Right, &Bottom)
GetVirtualMonitorFromPoint(x, y)  ; Returns virtual monitor containing point
GetVirtualMonitorForWindow(x, y, w, h)  ; Returns virtual monitor for window
GetVirtualMonitorPrimary()  ; Returns primary virtual monitor
```

### Integration Points

**Modified Functions:**
- `GetWindowMonitor()` in CoreUtils.ahk2 → Uses virtual monitors
- `MonitorFromPoint()` in MoveUtils.ahk2 → Uses virtual monitors
- All `MonitorGet()` calls → `VirtualMonitorGet()`
- All `MonitorGetWorkArea()` calls → `VirtualMonitorGetWorkArea()`

**Files Updated:**
- `src/lib/Core/VirtualMonitorManager.ahk2` (new)
- `src/lib/Core/CoreUtils.ahk2`
- `src/lib/Core/MoveUtils.ahk2`
- `src/lib/Config/GlobalConfig.ahk2`
- `src/Features/WindowMove.ahk2`
- `src/Features/WindowScaleWidth.ahk2`
- `src/Features/WindowScaleHeight.ahk2`
- `src/Features/WindowScaleXY.ahk2`
- `src/Features/WindowGrouping.ahk2`
- `src/Features/WindowCascade.ahk2`
- `AWindowMovementSuite.ahk2`

## Startup Behavior

**On Script Launch:**
1. `LoadPBPSettings()` is called automatically
2. Loads saved PBP state from `settings/pbp_settings.ini`
3. If file doesn't exist, uses defaults (PBP disabled)
4. Monitor info display shows both physical and virtual monitors

**Monitor Info Display:**
- Shows physical monitor information
- When PBP is enabled: Shows virtual monitor configuration
- When PBP is disabled: Shows toggle instructions

## Edge Cases & Behavior

### Window Center Point Detection
Windows are assigned to virtual monitors based on their center point. If a window spans the virtual boundary (X=3840), the virtual monitor containing the window's center determines which monitor it belongs to.

### Snapping to Virtual Boundary
The virtual monitor split edge (X=3840) acts as a physical monitor edge:
- Windows snap to X=3840 when dragged nearby
- Window resizing respects the virtual boundary
- Window positioning treats it as a monitor edge

### Work Area Simplification
Initial implementation uses full monitor bounds for both virtual monitors, ignoring taskbar work area complexities. This means:
- Both virtual monitors span the full height (0 to 2160)
- No special handling for vertical taskbars
- Taskbars on bottom affect both virtual monitors equally

## Future Enhancements

**Planned Features:**
1. **Configurable Split Position** - GUI to adjust the split point
2. **Multiple Split Presets** - 50/50, 60/40, 2:1 presets
3. **Percentage or Pixel Entry** - Choose input method
4. **Visual Split Indicator** - Optional overlay showing the split line
5. **Multi-Monitor PBP** - Apply PBP to specific monitors in multi-monitor setups
6. **Vertical Splits** - Split monitors top/bottom instead of left/right
7. **Triple Split** - Divide into 3+ virtual monitors

## Troubleshooting

### PBP Not Working

**Check:**
1. Is PBP enabled? (ScrollLock or tray menu)
2. Does the tooltip appear when toggling?
3. Check monitor info display for virtual monitor details

### Windows Not Snapping to Split Edge

**Verify:**
1. Window snapping is enabled globally
2. You're not holding Ctrl (disables snapping)
3. Window center must be near the split edge

### Settings Not Persisting

**Check:**
1. Settings folder exists: `settings/`
2. File created: `settings/pbp_settings.ini`
3. File permissions allow writing

### Reset to Defaults

**Delete settings file:**
```
settings/pbp_settings.ini
```
Script will recreate with defaults on next toggle.

## API Reference

See `src/lib/Core/VirtualMonitorManager.ahk2` for complete API documentation.

**Core Functions:**
- `VirtualMonitorGetCount()` - Get number of virtual monitors
- `VirtualMonitorGet()` - Get virtual monitor bounds
- `VirtualMonitorGetWorkArea()` - Get virtual monitor work area
- `GetVirtualMonitorFromPoint()` - Point-to-monitor mapping
- `GetVirtualMonitorForWindow()` - Window-to-monitor mapping
- `TogglePBPMode()` - Toggle PBP on/off
- `LoadPBPSettings()` - Load settings from file
- `SavePBPSettings()` - Save settings to file
- `GetPBPInfo()` - Get current PBP configuration info
