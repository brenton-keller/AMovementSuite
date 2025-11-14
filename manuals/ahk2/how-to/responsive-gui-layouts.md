# How-To: Build Responsive GUI Layouts

## Problem

Your AutoHotkey v2 GUI breaks when users resize windows. Controls stay fixed in place, overlap, or disappear at different window sizes. Users expect modern applications to adapt to various screen sizes and window states.

## Quick Context

Unlike modern frameworks (WinForms, WPF, Qt), **AHK2 has no native anchoring or dock positioning**. All responsive behavior requires manual implementation through the `OnSize` event and `Move()` method. This is the most significant difference from GUI frameworks you may be familiar with.

Every resizable GUI must handle the Size event, which fires whenever the window resizes. You calculate new positions/sizes and apply them with `Move()`.

## Key Concepts

- **OnSize Event**: Fires on window resize with parameters (GuiObj, MinMax, Width, Height)
- **MinMax States**: -1 (minimized), 0 (restored/resized), 1 (maximized)
- **Move() Method**: `Control.Move([X, Y, Width, Height])` - omitted params remain unchanged
- **Anchor Patterns**: Manually calculate positions to simulate anchoring
- **Percentage Layouts**: Calculate sizes as percentages of available space

For deeper understanding of GUI architecture, see:
- docs/manuals/ahk2/explanation/gui-architecture.md
- docs/manuals/ahk2/reference/gui-methods.md

## Solution 1: Basic OnSize Handler with Fill Pattern

**Use when**: Single control should fill the entire window (editors, viewers).

```ahk
#Requires AutoHotkey v2.0

; Simple resizable edit box
MyGui := Gui("+Resize", "Basic Responsive Window")
EditBox := MyGui.Add("Edit", "w400 h300")
MyGui.Show()
MyGui.OnEvent("Size", Gui_Size)

Gui_Size(GuiObj, MinMax, Width, Height) {
    ; ALWAYS check for minimized state first
    if MinMax = -1
        return

    ; Resize edit box to fill window with 10px margin
    margin := 10
    EditBox.Move(margin, margin, Width - (margin * 2), Height - (margin * 2))
}
```

**Key technique**: Always check `MinMax = -1` to avoid processing when minimized. This prevents unnecessary calculations and potential errors.

## Solution 2: Multi-Panel Layout with Manual Anchoring

**Use when**: Multiple controls with different resize behaviors (sidebars, toolbars, button panels).

```ahk
#Requires AutoHotkey v2.0

; Three-panel layout: fixed sidebar, expanding content, anchored buttons
MyGui := Gui("+Resize +MinSize450x300", "Multi-Panel Layout")

; Left panel - fixed width, expanding height
TreeView := MyGui.Add("TreeView", "x10 y10 w180 h280")
TreeView.Add("Folders")
TreeView.Add("Documents", TreeView.GetChild(0))
TreeView.Add("Pictures", TreeView.GetChild(0))

; Right panel - expanding both dimensions
EditBox := MyGui.Add("Edit", "x200 y10 w290 h240")

; Bottom buttons - anchored to bottom-right
BtnOK := MyGui.Add("Button", "x390 y260 w100 h30", "OK")
BtnCancel := MyGui.Add("Button", "x+10 yp w100 h30", "Cancel")

MyGui.Show("w510 h300")
MyGui.OnEvent("Size", Gui_Size)

Gui_Size(GuiObj, MinMax, Width, Height) {
    if MinMax = -1
        return

    ; TreeView: fixed width (180), expanding height
    ; Keep 10px margins top/bottom, 40px space for buttons
    TreeView.Move(, , , Height - 60)

    ; EditBox: expanding both dimensions
    ; X position fixed (200), width expands with window
    ; Height expands, leaving space for buttons
    EditBox.Move(, , Width - 210, Height - 100)

    ; Buttons: anchored to bottom-right
    ; Calculate from right edge and bottom
    BtnOK.Move(Width - 220, Height - 40)
    BtnCancel.Move(Width - 110, Height - 40)
}

; Add button handlers
BtnOK.OnEvent("Click", (*) => MsgBox("OK clicked!"))
BtnCancel.OnEvent("Click", (*) => ExitApp)
```

**Anchoring patterns**:
- **Fixed size, moving position**: `Control.Move(newX, newY)` - omit width/height
- **Expanding size, fixed position**: `Control.Move(, , newWidth, newHeight)` - omit X/Y
- **Bottom-right anchor**: `Control.Move(Width - controlWidth - margin, Height - controlHeight - margin)`
- **Top-left anchor**: Position stays unchanged, only size expands

## Solution 3: Percentage-Based Layout

**Use when**: Complex multi-section layouts where proportions matter more than fixed sizes.

```ahk
#Requires AutoHotkey v2.0

; Dashboard with three panels: top 45%, two bottom panels 25% each
MyGui := Gui("+Resize +MinSize500x400", "Percentage Layout Dashboard")

; Initial sizes (will be recalculated on resize)
TopPanel := MyGui.Add("Edit", "w580 h180", "Top panel - 45% height")
BottomLeft := MyGui.Add("Edit", "w280 h180", "Bottom left - 25% height, 50% width")
BottomRight := MyGui.Add("Edit", "w280 h180", "Bottom right - 25% height, 50% width")
StatusBar := MyGui.Add("Text", "w580 h30 Center Border", "Status Bar - Fixed height")

MyGui.Show("w600 h450")
MyGui.OnEvent("Size", Gui_Size)

Gui_Size(GuiObj, MinMax, Width, Height) {
    if MinMax = -1
        return

    ; Define layout constants
    margin := 10
    spacing := 10
    statusHeight := 30

    ; Calculate available space
    availWidth := Width - (2 * margin)
    availHeight := Height - (2 * margin) - statusHeight - spacing

    ; Top panel: 100% width, 45% height
    topHeight := Round(availHeight * 0.45)
    TopPanel.Move(margin, margin, availWidth, topHeight)

    ; Bottom panels: 50% width each, 25% height
    bottomY := margin + topHeight + spacing
    bottomHeight := Round(availHeight * 0.25)
    bottomWidth := Round((availWidth - spacing) / 2)

    BottomLeft.Move(margin, bottomY, bottomWidth, bottomHeight)
    BottomRight.Move(margin + bottomWidth + spacing, bottomY, bottomWidth, bottomHeight)

    ; Status bar: 100% width, fixed height at bottom
    statusY := Height - statusHeight - margin
    StatusBar.Move(margin, statusY, availWidth, statusHeight)
}
```

**Why Round()?**: Prevents fractional pixel positioning which can cause alignment issues. Always round percentage calculations.

## Solution 4: Using GuiReSizer Community Library

**Use when**: You want declarative percentage-based layouts without manual calculations.

First, download GuiReSizer from the forum: https://www.autohotkey.com/boards/viewtopic.php?t=113921

```ahk
#Requires AutoHotkey v2.0
#Include <GuiReSizer>  ; Place in Lib folder

MyGui := Gui("+Resize +MinSize400x300", "GuiReSizer Example")
MyGui.OnEvent("Size", GuiReSizer)

; Main content area - 95% width, 85% height
Edit1 := MyGui.Add("Edit", "w400 h250", "Main content")
Edit1.WP := 0.95    ; Width = 95% of GUI width
Edit1.HP := 0.85    ; Height = 85% of GUI height
Edit1.XP := 0.025   ; X = 2.5% from left (centered)
Edit1.YP := 0.02    ; Y = 2% from top

; OK button - anchored to bottom-right
OKBtn := MyGui.Add("Button", "w100 h30", "OK")
OKBtn.XP := 0.80    ; X = 80% from left
OKBtn.YP := 0.92    ; Y = 92% from top
OKBtn.MinW := 80    ; Minimum width constraint

; Cancel button - next to OK
CancelBtn := MyGui.Add("Button", "w100 h30", "Cancel")
CancelBtn.XP := 0.60
CancelBtn.YP := 0.92
CancelBtn.MaxW := 120  ; Maximum width constraint

MyGui.Show("w500 h350")

OKBtn.OnEvent("Click", (*) => MsgBox("Saved!"))
CancelBtn.OnEvent("Click", (*) => ExitApp)
```

**GuiReSizer properties**:
- **XP/YP**: Position as percentage (0.0 to 1.0)
- **WP/HP**: Size as percentage (0.0 to 1.0)
- **MinW/MaxW**: Width constraints in pixels
- **MinH/MaxH**: Height constraints in pixels
- **OffsetX/OffsetY**: Pixel adjustments after percentage calculation

## Solution 5: Advanced - Preventing Resize Flicker

**Use when**: Users report visual artifacts during window resizing (control ghosting, tearing).

```ahk
#Requires AutoHotkey v2.0

MyGui := Gui("+Resize +MinSize400x300", "Anti-Flicker Example")

Edit1 := MyGui.Add("Edit", "w400 h200")
Edit2 := MyGui.Add("Edit", "w400 h80")
StatusBar := MyGui.Add("Text", "w400 h20 Border", "Status")

MyGui.Show("w420 h310")
MyGui.OnEvent("Size", Gui_Size)

; Unfreeze window when user finishes resizing
OnMessage(0x0232, (*) => DllCall("LockWindowUpdate", "UInt", 0))  ; WM_EXITSIZEMOVE

Gui_Size(GuiObj, MinMax, Width, Height) {
    static LastMinMax := -1

    if MinMax = -1
        return

    ; Freeze window redraw during control updates
    DllCall("LockWindowUpdate", "UInt", GuiObj.Hwnd)

    ; Calculate positions
    margin := 10
    availW := Width - (2 * margin)
    statusH := 20

    ; Update all controls
    Edit1.Move(margin, margin, availW, Height * 0.6)
    Edit2.Move(margin, Height * 0.62, availW, Height * 0.25)
    StatusBar.Move(margin, Height - statusH - margin, availW, statusH)

    ; Unfreeze on state change (restored ↔ maximized)
    if MinMax != LastMinMax
        DllCall("LockWindowUpdate", "UInt", 0), LastMinMax := MinMax
}
```

**How it works**: `LockWindowUpdate` freezes screen updates while you move multiple controls. The `WM_EXITSIZEMOVE` message unlocks when the user releases the resize handle. We also unlock on maximize/restore state changes.

**Warning**: Don't lock indefinitely - always have an unlock path or the window stays frozen.

## Complete Working Example: Responsive Settings Dialog

This demonstrates all techniques in a production-ready dialog:

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force

; =============================================================================
; RESPONSIVE SETTINGS DIALOG - Complete Example
; Demonstrates: All anchor patterns, percentage layouts, anti-flicker
; =============================================================================

class SettingsDialog extends Gui {
    __New() {
        super.__New("+Resize +MinSize550x450", "Application Settings", this)

        this.CreateControls()
        this.OnEvent("Size", this.HandleResize.Bind(this))
        this.Show("w600 h500")
    }

    CreateControls() {
        this.SetFont("s10", "Segoe UI")

        ; Header (fixed height, expanding width)
        this.SetFont("s12 Bold")
        this.header := this.Add("Text", "xm ym w580 h40 Center +0x200",
            "Application Settings")
        this.SetFont("s10")

        ; Left panel - Settings tree (fixed width, expanding height)
        this.tree := this.Add("TreeView", "xm y+10 w180 h350")
        general := this.tree.Add("General")
        this.tree.Add("Display", general)
        this.tree.Add("Sound", general)
        this.tree.Add("Network", general)
        advanced := this.tree.Add("Advanced")
        this.tree.Add("Performance", advanced)
        this.tree.Add("Debug", advanced)

        ; Right panel - Settings content (expanding both dimensions)
        this.contentBox := this.Add("Edit", "x+10 yp w380 h350 +Multi")
        this.contentBox.Value := "Settings content appears here.`n`n"
            . "This area expands in both directions as you resize the window.`n`n"
            . "The tree view on the left has fixed width but expanding height.`n`n"
            . "Buttons stay anchored to the bottom-right corner."

        ; Status bar (fixed height, expanding width, anchored to bottom)
        this.status := this.Add("Text", "xm w580 h25 Border Center",
            "Ready")

        ; Button panel (anchored to bottom-right)
        this.btnSave := this.Add("Button", "w100 h30 Default", "Save")
        this.btnApply := this.Add("Button", "w100 h30", "Apply")
        this.btnCancel := this.Add("Button", "w100 h30", "Cancel")

        ; Event handlers
        this.btnSave.OnEvent("Click", (*) => this.HandleSave())
        this.btnApply.OnEvent("Click", (*) => this.HandleApply())
        this.btnCancel.OnEvent("Click", (*) => ExitApp)
        this.tree.OnEvent("Click", (*) => this.HandleTreeClick())
    }

    HandleResize(guiObj, MinMax, Width, Height) {
        if MinMax = -1
            return

        ; Anti-flicker: freeze redraw during updates
        static LastMinMax := -1
        DllCall("LockWindowUpdate", "UInt", guiObj.Hwnd)

        ; Layout constants
        margin := 10
        headerH := 40
        statusH := 25
        buttonH := 30
        buttonW := 100
        spacing := 10
        treeW := 180

        ; Calculate available space
        availW := Width - (2 * margin)
        availH := Height - (2 * margin) - headerH - statusH - buttonH - (spacing * 3)
        contentW := availW - treeW - spacing

        ; Header: expanding width, fixed height
        this.header.Move(margin, margin, availW, headerH)

        ; Tree view: fixed width, expanding height
        treeY := margin + headerH + spacing
        this.tree.Move(margin, treeY, treeW, availH)

        ; Content box: expanding both dimensions
        contentX := margin + treeW + spacing
        this.contentBox.Move(contentX, treeY, contentW, availH)

        ; Status bar: expanding width, fixed height, anchored to bottom
        statusY := Height - margin - statusH - buttonH - spacing
        this.status.Move(margin, statusY, availW, statusH)

        ; Buttons: anchored to bottom-right
        buttonY := Height - margin - buttonH
        this.btnCancel.Move(Width - margin - buttonW, buttonY, buttonW, buttonH)
        this.btnApply.Move(Width - margin - (buttonW * 2) - spacing, buttonY, buttonW, buttonH)
        this.btnSave.Move(Width - margin - (buttonW * 3) - (spacing * 2), buttonY, buttonW, buttonH)

        ; Unfreeze on state change
        if MinMax != LastMinMax
            DllCall("LockWindowUpdate", "UInt", 0), LastMinMax := MinMax
    }

    HandleTreeClick() {
        item := this.tree.GetSelection()
        if !item
            return

        text := this.tree.GetText(item)
        this.status.Text := "Selected: " text
        this.contentBox.Value := "Settings for: " text "`n`nContent would appear here."
    }

    HandleSave() {
        this.status.Text := "Settings saved!"
        SetTimer(() => this.status.Text := "Ready", -2000)
    }

    HandleApply() {
        this.status.Text := "Settings applied!"
        SetTimer(() => this.status.Text := "Ready", -2000)
    }
}

; Unfreeze when user releases resize handle
OnMessage(0x0232, (*) => DllCall("LockWindowUpdate", "UInt", 0))

; Create dialog
dialog := SettingsDialog()

; Development hotkey
^R:: Reload
```

**What this demonstrates**:
- **Header**: Fixed height, expanding width
- **Tree view**: Fixed width, expanding height
- **Content area**: Expanding both dimensions
- **Status bar**: Fixed height, expanding width, anchored to bottom
- **Buttons**: Fixed size, anchored to bottom-right corner
- **Anti-flicker**: LockWindowUpdate prevents visual artifacts
- **Clean code**: Class-based structure with proper binding

## Troubleshooting

### Problem: Controls flicker during resize
**Solution**: Use `LockWindowUpdate` (see Solution 5). Always unlock with `WM_EXITSIZEMOVE` message.

### Problem: Layout breaks at small sizes
**Solution**: Add minimum size constraint: `Gui("+Resize +MinSize400x300")`. Test at your minimum size.

### Problem: Calculations feel complex
**Solution**: Either:
1. Use GuiReSizer library for declarative approach
2. Group calculations at top of handler with descriptive variable names
3. Create helper methods: `CalculateButtonPositions()`, `UpdateContentArea()`

### Problem: Controls overlap after resizing
**Solution**: Account for all spacing in calculations. Draw a diagram with margins, spacing, and control sizes. Common mistake: forgetting spacing between controls.

### Problem: Buttons disappear when maximized
**Solution**: Use dynamic calculations, not fixed positions. Always calculate from Width/Height parameters:
```ahk
; WRONG - fixed position
Button.Move(500, 400)

; CORRECT - calculated from bottom-right
Button.Move(Width - 120, Height - 50)
```

### Problem: Round() causes alignment drift over multiple resizes
**Solution**: This is rare, but if controls drift, calculate positions from window dimensions each time, not incrementally from previous positions.

### Problem: Performance issues with many controls
**Solution**:
1. Use `LockWindowUpdate` to batch updates
2. Only resize visible controls (check tab state)
3. Consider virtualizing lists (don't create all controls upfront)

## Best Practices

1. **Always set minimum size**: `+MinSize{width}x{height}` prevents broken layouts
2. **Check MinMax first**: Skip processing on minimize (`MinMax = -1`)
3. **Use consistent spacing**: Define `margin` and `spacing` variables
4. **Calculate once, use many**: Store commonly used values (availWidth, availHeight)
5. **Test at extremes**: Minimum size, maximized, various aspect ratios
6. **Comment anchor behaviors**: Note which controls are fixed/expanding/anchored
7. **Group related controls**: Calculate panel/section sizes together
8. **Use Round() for percentages**: Prevents fractional pixels
9. **Bind event handlers**: Always `.Bind(this)` in classes
10. **Consider GuiReSizer**: For complex layouts, declarative approach is cleaner

## Testing Checklist

- [ ] Window resizes smoothly without flicker
- [ ] Layout works at minimum size
- [ ] Layout works maximized
- [ ] Layout works on different aspect ratios (wide, tall)
- [ ] Buttons stay accessible (visible, not overlapped)
- [ ] Controls don't overlap or leave gaps
- [ ] Status bar/toolbars anchor correctly
- [ ] Content areas utilize available space
- [ ] No errors when rapidly resizing
- [ ] Minimizing/restoring doesn't cause issues

## When to Use Each Approach

| Approach | Use Case | Complexity | Flexibility |
|----------|----------|------------|-------------|
| **Basic OnSize** | Single control fills window | Low | Low |
| **Manual Anchoring** | Mixed resize behaviors | Medium | High |
| **Percentage Layout** | Proportional sections | Medium | Medium |
| **GuiReSizer** | Complex declarative layouts | Low | Medium |
| **Anti-flicker** | Many controls, visual issues | High | High |

## Further Reading

- docs/manuals/ahk2/reference/gui-events.md - All GUI events reference
- docs/manuals/ahk2/reference/gui-control-methods.md - Move() and other control methods
- docs/manuals/ahk2/how-to/reusable-gui-components.md - Component-based architecture
- docs/manuals/ahk2/explanation/gui-architecture.md - Understanding GUI design patterns

## Related Community Resources

- **GuiReSizer Library**: https://www.autohotkey.com/boards/viewtopic.php?t=113921
- **AHK v2 GUI Megathread**: https://www.autohotkey.com/boards/viewtopic.php?t=123220
- **Easy AutoGUI Designer**: https://github.com/samfisherirl/Easy-Auto-GUI-for-AHK-v2
