# UI Components

Technical reference for user interface components and GUI systems in A Movement Suite.

## Overview

The UI layer provides:
- Custom dialog boxes for user interaction
- Settings interfaces for configuration
- Preview overlays for window operations
- Visual feedback (ToolTips, notifications)
- Grid positioning system

## Component Modules

| Component | Purpose | File Location |
|-----------|---------|---------------|
| CustomDialogs | Reusable dialog boxes | src/UI/CustomDialogs.ahk2 |
| SettingsGUI | Configuration interfaces | src/UI/SettingsGUI.ahk2 |
| Preview Overlays | Window operation previews | Embedded in feature modules |
| Grid Positioning | Window positioning menu | src/Features/WindowMove.ahk2 |

---

## CustomDialogs (src/UI/CustomDialogs.ahk2)

### Purpose
Provides reusable, styled dialog boxes for common user interactions.

### Dialog Types

#### 1. Confirmation Dialog
```ahk
ShowConfirmation(message, title := "Confirm") {
    result := MsgBox(message, title, "YesNo Icon?")
    return result = "Yes"
}
```

#### 2. Input Dialog
```ahk
ShowInputDialog(prompt, title := "Input", default := "") {
    ib := InputBox(prompt, title, "", default)
    if (ib.Result = "OK")
        return ib.Value
    return ""
}
```

#### 3. Information Dialog
```ahk
ShowInfo(message, title := "Information") {
    MsgBox message, title, "Iconi"
}
```

#### 4. Error Dialog
```ahk
ShowError(message, title := "Error") {
    MsgBox message, title, "Icon!"
}
```

### Usage Examples

```ahk
; Confirmation
if ShowConfirmation("Are you sure?") {
    ; User clicked Yes
}

; Input
userName := ShowInputDialog("Enter your name:", "User Info", "Default Name")
if (userName != "") {
    ; User provided input
}

; Info
ShowInfo("Operation completed successfully!")

; Error
ShowError("Failed to save settings: " errorMessage)
```

---

## SettingsGUI (src/UI/SettingsGUI.ahk2)

### Purpose
Provides configuration interfaces for user preferences and feature settings.

### Window Move Settings

#### GUI Structure
```ahk
ShowWindowMoveSettings(*) {
    global snapDistance

    ; Create GUI
    settingsGui := Gui("+AlwaysOnTop", "Window Move Settings")
    settingsGui.SetFont("s10")

    ; Add controls
    settingsGui.Add("Text", "x10 y10 w120", "Snap Distance:")
    snapInput := settingsGui.Add("Edit", "x140 y10 w80", snapDistance)
    settingsGui.Add("Text", "x230 y10", "pixels")

    ; Add buttons
    saveBtn := settingsGui.Add("Button", "x10 y40 w100", "Save")
    cancelBtn := settingsGui.Add("Button", "x120 y40 w100", "Cancel")

    ; Event handlers
    saveBtn.OnEvent("Click", (*) => SaveSnapDistance())
    cancelBtn.OnEvent("Click", (*) => settingsGui.Destroy())

    SaveSnapDistance() {
        global snapDistance
        newValue := Integer(snapInput.Value)

        if (newValue < 1 || newValue > 100) {
            MsgBox "Snap distance must be between 1 and 100", "Invalid Input", "Icon!"
            return
        }

        snapDistance := newValue
        UserConfig.Set("SnapDistance", snapDistance)
        UserConfig.Save()

        settingsGui.Destroy()
        ShowInfo("Settings saved successfully!")
    }

    settingsGui.Show("w350 h80")
}
```

### Settings GUI Pattern

```ahk
ShowFeatureSettings(*) {
    ; 1. Create GUI with AlwaysOnTop flag
    gui := Gui("+AlwaysOnTop", "Feature Settings")

    ; 2. Add input controls
    ; - Edit boxes for text/numeric input
    ; - Checkboxes for boolean options
    ; - DropDownLists for enums
    ; - Sliders for ranges

    ; 3. Populate with current values
    ; - Load from UserConfig or GlobalConfig

    ; 4. Add Save/Cancel buttons

    ; 5. Implement validation
    ; - Range checks
    ; - Type validation
    ; - Required fields

    ; 6. Save on confirmation
    ; - Update UserConfig
    ; - Persist to file
    ; - Update runtime state

    ; 7. Show GUI
    gui.Show()
}
```

---

## Preview Overlays

### Purpose
Show semi-transparent overlay windows to preview window operations before applying.

### XY Scaling Preview

Used in `WindowScaleXY.ahk2`:

```ahk
CreatePreviewOverlay(x, y, width, height) {
    ; Create preview GUI
    previewGui := Gui("+AlwaysOnTop +ToolWindow -Caption +E0x20")

    ; Make semi-transparent
    previewGui.BackColor := GlobalConfig.Colors.Preview
    WinSetTransparent(128, previewGui)  ; 50% transparent

    ; Add dimension text
    dimText := previewGui.Add("Text", "Center", width " x " height)
    dimText.SetFont("s12 bold", "Arial")

    ; Position and show
    previewGui.Show("x" x " y" y " w" width " h" height " NoActivate")

    return previewGui
}

; Usage
previewGui := CreatePreviewOverlay(newX, newY, newWidth, newHeight)
; ... wait for user to release mouse ...
previewGui.Destroy()  ; Clean up
```

### Preview Pattern
```ahk
; 1. Create overlay GUI
; 2. Set transparent background
; 3. Position at target location
; 4. Show without activation (NoActivate)
; 5. Update during operation
; 6. Destroy when operation completes or cancels
```

---

## Grid Positioning System

### Purpose
Provides quick window positioning to common layouts via visual grid menu.

### Activation
Press `Z` key during window movement to display grid.

### Grid Layout

```
┌─────────┬─────────┬─────────┐
│  1/3    │  1/3    │  1/3    │
│  Left   │ Center  │  Right  │
├─────────┴─────────┴─────────┤
│         1/2 Left             │
├──────────────────────────────┤
│        1/2 Right             │
├──────────────────────────────┤
│         Full Screen          │
└──────────────────────────────┘
```

### Implementation

```ahk
ShowPositioningGrid(*) {
    global positioningGui

    ; Prevent duplicate GUIs
    if IsObject(positioningGui) {
        try positioningGui.Destroy()
    }

    ; Get monitor work area
    MouseGetPos &mx, &my
    monNum := MonitorGetFromPoint(mx, my)
    MonitorGetWorkArea(monNum, &left, &top, &right, &bottom)

    ; Create grid GUI
    positioningGui := Gui("+AlwaysOnTop +ToolWindow", "Position Window")

    ; Add buttons for each position
    AddPositionButton(positioningGui, "Left 1/3", left, top, (right-left)//3, bottom-top)
    AddPositionButton(positioningGui, "Center 1/3", left+(right-left)//3, top, (right-left)//3, bottom-top)
    AddPositionButton(positioningGui, "Right 1/3", left+2*(right-left)//3, top, (right-left)//3, bottom-top)
    ; ... more positions ...

    positioningGui.Show()
}

AddPositionButton(gui, label, x, y, w, h) {
    btn := gui.Add("Button", "w200", label)
    btn.OnEvent("Click", (*) => PositionWindow(x, y, w, h))
}

PositionWindow(x, y, w, h) {
    global positioningGui

    ; Get window under mouse at start of move operation
    MouseGetPos , , &targetWin

    ; Position window
    WinMove x, y, w, h, "ahk_id " targetWin

    ; Close grid
    positioningGui.Destroy()
}
```

---

## Visual Feedback

### ToolTips

#### Success Messages
```ahk
ShowSuccess(message) {
    ToolTip message, , , 1
    SetTimer () => ToolTip("", , , 1), -2000  ; Auto-hide after 2 seconds
}
```

#### Status Indicators
```ahk
ShowStatus(message, duration := 1000) {
    ToolTip message, , , 1
    SetTimer () => ToolTip("", , , 1), -duration
}
```

#### Snap Indicators (from WindowMove.ahk2)
```ahk
ShowSnapIndicator(message) {
    ToolTip message, , , 1
    SetTimer () => ToolTip("", , , 1), -1000
}

; Usage
ShowSnapIndicator("Snapped to window edge")
ShowSnapIndicator("Snapped to screen edge")
```

### Notification Pattern
```ahk
; DO: Auto-hide with timer
ToolTip "Operation complete"
SetTimer () => ToolTip(), -2000

; DON'T: Leave tooltip showing
ToolTip "Operation complete"  ; Will stay forever
```

---

## GUI Styling

### Font Settings
```ahk
gui := Gui()
gui.SetFont("s10", "Segoe UI")  ; Size 10, Segoe UI font
```

### Colors
```ahk
gui.BackColor := 0xF0F0F0  ; Light gray background
control.SetFont("c0x0000FF")  ; Blue text color
```

### Window Styles
```ahk
; Standard window
Gui("+Resize", "Title")

; Always on top
Gui("+AlwaysOnTop", "Title")

; Tool window (no taskbar button)
Gui("+ToolWindow", "Title")

; Borderless
Gui("-Caption", "Title")

; Click-through (for overlays)
Gui("+E0x20", "Title")
```

---

## Common GUI Patterns

### Modal Dialog Pattern
```ahk
ShowModalDialog() {
    ; Create GUI
    gui := Gui("+AlwaysOnTop", "Modal Dialog")

    ; Add controls
    gui.Add("Text", , "This is a modal dialog")
    okBtn := gui.Add("Button", "Default", "OK")

    ; Block until user clicks OK
    okBtn.OnEvent("Click", (*) => gui.Destroy())

    ; Show and wait
    gui.Show()
    ; (Note: AHK v2 GUIs are not truly modal, but AlwaysOnTop helps)
}
```

### Input Validation Pattern
```ahk
ShowValidatedInput() {
    gui := Gui("+AlwaysOnTop", "Input")

    editCtrl := gui.Add("Edit", "w200", "")
    saveBtn := gui.Add("Button", , "Save")

    saveBtn.OnEvent("Click", Validate)

    Validate(*) {
        value := editCtrl.Value

        ; Validation
        if (value = "") {
            MsgBox "Input cannot be empty", "Error", "Icon!"
            return  ; Don't close GUI
        }

        if (!IsNumber(value)) {
            MsgBox "Input must be a number", "Error", "Icon!"
            return
        }

        ; Valid input - proceed
        ProcessInput(value)
        gui.Destroy()
    }

    gui.Show()
}
```

### Dynamic Control Pattern
```ahk
ShowDynamicGUI() {
    gui := Gui()

    ; Add controls dynamically
    optionCount := 5
    loop optionCount {
        gui.Add("Checkbox", , "Option " A_Index)
    }

    gui.Show()
}
```

---

## Event Handling

### Button Click
```ahk
btn := gui.Add("Button", , "Click Me")
btn.OnEvent("Click", HandleClick)

HandleClick(ctrl, info) {
    MsgBox "Button clicked!"
}
```

### GUI Close
```ahk
gui.OnEvent("Close", HandleClose)

HandleClose(guiObj) {
    ; Clean up before closing
    SaveState()
    guiObj.Destroy()
}
```

### GUI Size (Resize)
```ahk
gui.OnEvent("Size", HandleResize)

HandleResize(guiObj, MinMax, Width, Height) {
    ; Adjust controls when window resizes
    if IsObject(editCtrl) {
        editCtrl.Move(10, 10, Width - 20, Height - 50)
    }
}
```

### Control Value Change
```ahk
edit := gui.Add("Edit")
edit.OnEvent("Change", HandleChange)

HandleChange(ctrl, info) {
    currentValue := ctrl.Value
    ; React to changes
}
```

---

## Monitor Information Display

### Startup Monitor Info (from AWindowMovementSuite.ahk2)

```ahk
ShowMonitorInfo() {
    monitorCount := MonitorGetCount()
    infoText := "Detected " monitorCount " Monitor(s):`r`n`r`n"

    Loop monitorCount {
        MonitorGet(A_Index, &Left, &Top, &Right, &Bottom)
        MonitorGetWorkArea(A_Index, &WorkLeft, &WorkTop, &WorkRight, &WorkBottom)

        infoText .= "--- Monitor " A_Index " ---`r`n"
        infoText .= "  Position: X=" Left ", Y=" Top "`r`n"
        infoText .= "  Size: " (Right-Left) "x" (Bottom-Top) "`r`n"
        infoText .= "  Work Area: " (WorkRight-WorkLeft) "x" (WorkBottom-WorkTop) "`r`n`r`n"
    }

    ; Create GUI to display info
    InfoGui := Gui("+AlwaysOnTop +Resize", "Monitor Information")
    InfoGui.SetFont("s10", "Consolas")

    editCtrl := InfoGui.Add("Edit", "w500 R15 ReadOnly", infoText)
    btnCtrl := InfoGui.Add("Button", "Default", "OK")
    btnCtrl.OnEvent("Click", (*) => InfoGui.Destroy())

    InfoGui.Show()
}
```

**Source:** `AWindowMovementSuite.ahk2:56-134`

---

## Best Practices

### 1. Always Destroy GUIs
```ahk
; DO: Clean up GUIs when done
gui.Destroy()

; DON'T: Leave GUIs in memory
; (GUI just hidden, still consuming resources)
```

### 2. Use AlwaysOnTop for Settings
```ahk
; DO: Settings should stay on top
settingsGui := Gui("+AlwaysOnTop", "Settings")

; DON'T: Settings get lost behind windows
settingsGui := Gui("Settings")
```

### 3. Validate User Input
```ahk
; DO: Validate before processing
if (!IsValidInput(value)) {
    ShowError("Invalid input")
    return
}

; DON'T: Assume input is valid
ProcessInput(value)  ; Might crash or produce errors
```

### 4. Provide Visual Feedback
```ahk
; DO: Confirm actions
ShowSuccess("Settings saved successfully!")

; DON'T: Silent operations
UserConfig.Save()  ; User doesn't know if it worked
```

### 5. Handle Errors Gracefully
```ahk
; DO: Try-catch for risky operations
try {
    RiskyOperation()
}
catch Error as err {
    ShowError("Operation failed: " err.Message)
}

; DON'T: Let errors crash the GUI
RiskyOperation()  ; Unhandled errors = crash
```

---

## Troubleshooting

### GUI Not Showing

**Check:**
1. `gui.Show()` is called
2. GUI is not destroyed before showing
3. GUI is not shown off-screen (check coordinates)
4. No errors during GUI creation

**Debug:**
```ahk
MsgBox "About to show GUI"
gui.Show()
MsgBox "GUI shown"
```

### Controls Not Responding

**Check:**
1. Events are properly registered (`.OnEvent()`)
2. Event handler function exists
3. GUI is not blocked by modal operation
4. Control is enabled (not disabled)

**Debug:**
```ahk
btn.OnEvent("Click", (*) => MsgBox("Button clicked!"))
```

### GUI Positioning Issues

**Check:**
1. Monitor coordinates calculated correctly
2. GUI size fits on screen
3. Multi-monitor setup handled properly

**Solution:**
```ahk
; Center GUI on primary monitor
MonitorGetWorkArea(MonitorGetPrimary(), &Left, &Top, &Right, &Bottom)
centerX := Left + (Right - Left - guiWidth) // 2
centerY := Top + (Bottom - Top - guiHeight) // 2
gui.Show("x" centerX " y" centerY)
```

---

**See Also:**
- `architecture.md` - UI layer in system architecture
- `features.md` - Feature-specific UI components
- `configuration.md` - Settings persistence
