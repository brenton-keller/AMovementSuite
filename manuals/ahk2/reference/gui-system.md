# GUI System Reference

Complete technical reference for AutoHotkey v2 GUI classes, methods, properties, events, and community libraries.

---

## Gui Class Reference

### Core Methods

| Method | Syntax | Returns | Description |
|--------|--------|---------|-------------|
| `__New()` | `Gui([Options, Title, EventSink])` | Gui object | Creates new GUI. EventSink (typically `this`) enables method-name event handlers |
| `Add()` | `Add(ControlType, Options, Text)` | Control object | Adds control, returns control object for chaining |
| `Show()` | `Show([Options])` | - | Displays window. Options: `w{Width} h{Height} Center` |
| `Hide()` | `Hide()` | - | Hides window without destroying |
| `Destroy()` | `Destroy()` | - | Destroys GUI and releases resources |
| `OnEvent()` | `OnEvent(EventName, Callback)` | - | Registers event handler function/method |
| `SetFont()` | `SetFont([Options, FontName])` | - | Sets font for subsequently added controls |

### Core Properties

| Property | Type | Access | Description |
|----------|------|--------|-------------|
| `BackColor` | String/Integer | Read/Write | Background color: `"Red"`, `"0xFF0000"`, `0xFF0000` |
| `MarginX` | Integer | Read/Write | Horizontal margin for controls (default: 10) |
| `MarginY` | Integer | Read/Write | Vertical margin for controls (default: 10) |
| `Hwnd` | Integer | Read | Window handle for API calls |
| `Title` | String | Read/Write | Window title text |
| `MenuBar` | MenuBar | Write | Attach menu bar to window |

### GUI Options (Constructor)

| Option | Effect | Example |
|--------|--------|---------|
| `+Resize` | Enable window resizing | `Gui("+Resize")` |
| `-Resize` | Disable window resizing | `Gui("-Resize")` |
| `+MinSize{W}x{H}` | Set minimum window size | `Gui("+MinSize400x300")` |
| `+MaxSize{W}x{H}` | Set maximum window size | `Gui("+MaxSize1920x1080")` |
| `+AlwaysOnTop` | Keep window above others | `Gui("+AlwaysOnTop")` |
| `-Caption` | Remove title bar | `Gui("-Caption")` |
| `-SysMenu` | Remove system menu | `Gui("-SysMenu")` |
| `+ToolWindow` | Small title bar, no taskbar | `Gui("+ToolWindow")` |
| `+Border` | Add window border | `Gui("+Border")` |
| `-Border` | Remove window border | `Gui("-Border")` |
| `+MinimizeBox` | Enable minimize button | `Gui("+MinimizeBox")` |
| `-MinimizeBox` | Disable minimize button | `Gui("-MinimizeBox")` |
| `+MaximizeBox` | Enable maximize button | `Gui("+MaximizeBox")` |
| `-MaximizeBox` | Disable maximize button | `Gui("-MaximizeBox")` |
| `+Owner{Hwnd}` | Set owner window | `Gui("+Owner" ParentHwnd)` |

### GUI Events

| Event | Handler Signature | Description | When Fired |
|-------|-------------------|-------------|------------|
| `Close` | `Func(GuiObj)` | Window close button clicked | User clicks X, Alt+F4 |
| `Escape` | `Func(GuiObj)` | Escape key pressed | ESC pressed while GUI focused |
| `Size` | `Func(GuiObj, MinMax, Width, Height)` | Window resized | Resize, minimize, maximize |
| `DropFiles` | `Func(GuiObj, Ctrl, FileArray, X, Y)` | Files dropped on GUI | Drag-drop operation |
| `ContextMenu` | `Func(GuiObj, Ctrl, Item, IsRightClick, X, Y)` | Right-click context menu | Right-click on control |

---

## Control Types Reference

### Control Type Catalog

| Type | Purpose | Common Options | Key Methods | Key Properties |
|------|---------|----------------|-------------|----------------|
| `Text` | Static label | `w{W} h{H} c{Color} Center Border` | - | `Text`, `Value` |
| `Edit` | Text input box | `w{W} h{H} Password Multi ReadOnly Number` | `SetFont()` | `Text`, `Value` |
| `Button` | Push button | `w{W} h{H} Default` | `SetFont()` | `Text`, `Enabled` |
| `Checkbox` | Checkbox | `Checked` | - | `Value` (0/1) |
| `Radio` | Radio button | `Checked Group` | - | `Value` (0/1) |
| `DropDownList` | Dropdown selector | `w{W} Choose{N}` | `Add()`, `Delete()` | `Text`, `Value` |
| `ComboBox` | Editable dropdown | `w{W}` | `Add()`, `Delete()` | `Text`, `Value` |
| `ListBox` | List selector | `w{W} h{H} Multi` | `Add()`, `Delete()` | `Text`, `Value` |
| `ListView` | Multi-column list | `w{W} h{H} Grid` | `Add()`, `Delete()`, `Modify()` | `FocusedRowNumber` |
| `TreeView` | Tree hierarchy | `w{W} h{H} Checked` | `Add()`, `Delete()`, `Modify()` | - |
| `Tab` | Tab control | `w{W}` | `UseTab()` | `Value` |
| `Slider` | Value slider | `Range{Min}-{Max} TickInterval{N} ToolTipTop` | - | `Value` |
| `Progress` | Progress bar | `w{W} h{H} Range{Min}-{Max}` | - | `Value` |
| `UpDown` | Spin control | `Range{Min}-{Max}` | - | `Value` |
| `Pic` | Picture/image | `w{W} h{H}` | - | `Value` (file path) |
| `GroupBox` | Visual grouping | `w{W} h{H}` | - | `Text` |
| `StatusBar` | Status bar | - | `SetText()`, `SetIcon()` | - |
| `MonthCal` | Calendar picker | - | - | `Value` (date string) |
| `DateTime` | Date/time picker | `LongDate ShortDate Time` | - | `Value` (datetime) |
| `Hotkey` | Hotkey input | `w{W}` | - | `Value` (hotkey string) |
| `Link` | Hyperlink | - | - | `Text` |
| `Custom` | Custom control | - | - | - |

### Common Control Options

| Option | Applies To | Effect | Example |
|--------|-----------|--------|---------|
| `x{N}` | All | X position in pixels | `"x50"` |
| `y{N}` | All | Y position in pixels | `"y100"` |
| `x+{N}` | All | Offset from previous control | `"x+10"` (10px right) |
| `xm` | All | Reset to left margin | `"xm"` |
| `w{N}` | All | Width in pixels | `"w300"` |
| `h{N}` | All | Height in pixels | `"h200"` |
| `v{Name}` | All | Variable name for Submit | `"vUserName"` |
| `c{Color}` | Text, Button | Text color | `"cRed"`, `"c0xFF0000"` |
| `+Background{Color}` | Most | Background color | `"+Background0xFFFF00"` |
| `Disabled` | All | Initially disabled | `"Disabled"` |
| `Hidden` | All | Initially hidden | `"Hidden"` |
| `Default` | Button | Default button (Enter key) | `"Default"` |
| `Multi` | Edit, ListBox | Multi-line / multi-select | `"Multi"` |
| `ReadOnly` | Edit | Read-only edit box | `"ReadOnly"` |
| `Password` | Edit | Password masking | `"Password"` |
| `Number` | Edit | Numeric input only | `"Number"` |
| `Checked` | Checkbox, Radio | Initially checked | `"Checked"` |
| `Group` | Radio | Start radio button group | `"Group"` |
| `Grid` | ListView | Show grid lines | `"Grid"` |
| `Border` | Text, Edit | Add border | `"Border"` |
| `Center` | Text | Center-align text | `"Center"` |

### Control Events

| Control Type | Available Events | Event Parameters |
|--------------|------------------|------------------|
| Button | `Click` | `Func(CtrlObj, Info)` |
| Edit | `Change`, `Focus`, `LoseFocus` | `Func(CtrlObj, Info)` |
| Checkbox | `Click`, `Change` | `Func(CtrlObj, Info)` |
| Radio | `Click`, `Change` | `Func(CtrlObj, Info)` |
| DropDownList | `Change`, `Focus` | `Func(CtrlObj, Info)` |
| ComboBox | `Change`, `Focus` | `Func(CtrlObj, Info)` |
| ListBox | `Change`, `DoubleClick` | `Func(CtrlObj, Info)` |
| ListView | `Click`, `DoubleClick`, `ItemChanged`, `ItemFocus`, `ItemSelect` | `Func(CtrlObj, Item, Selected)` |
| TreeView | `Click`, `DoubleClick`, `ItemExpand`, `ItemSelect` | `Func(CtrlObj, Item)` |
| Tab | `Change` | `Func(CtrlObj, Info)` |
| Slider | `Change` | `Func(CtrlObj, Info)` |
| Link | `Click` | `Func(CtrlObj, Info, Href)` |

---

## Event Parameters Reference

### Size Event Parameters

```ahk
Gui_Size(GuiObj, MinMax, Width, Height)
```

| Parameter | Type | Description | Values |
|-----------|------|-------------|--------|
| `GuiObj` | Object | The GUI object that was resized | GUI instance |
| `MinMax` | Integer | Window state | `-1` = Minimized<br>`0` = Restored/Resized<br>`1` = Maximized |
| `Width` | Integer | New client area width | Pixels |
| `Height` | Integer | New client area height | Pixels |

**Critical**: Always check `if MinMax = -1` and return early to skip processing when window is minimized.

### Control Event Parameters

```ahk
HandleClick(CtrlObj, Info)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `CtrlObj` | Object | The control object that fired the event |
| `Info` | Integer/String | Event-specific information (often unused) |

### ListView Event Parameters

```ahk
LV_ItemSelect(CtrlObj, Item, Selected)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `CtrlObj` | Object | ListView control object |
| `Item` | Integer | Row number (1-based) |
| `Selected` | Integer | 1 if selected, 0 if deselected |

### Close Event Parameters

```ahk
HandleClose(GuiObj)
```

| Parameter | Type | Description | Return Value |
|-----------|------|-------------|--------------|
| `GuiObj` | Object | GUI object being closed | Return `true` to prevent close<br>Return `false` or nothing to allow |

---

## Move() Method Complete Reference

Controls the position and size of GUI controls.

### Syntax

```ahk
ControlObj.Move([X, Y, Width, Height])
```

### Parameters

| Parameter | Type | Optional | Description | Behavior if Omitted |
|-----------|------|----------|-------------|---------------------|
| `X` | Integer | Yes | Horizontal position (pixels from left) | Position unchanged |
| `Y` | Integer | Yes | Vertical position (pixels from top) | Position unchanged |
| `Width` | Integer | Yes | Control width in pixels | Width unchanged |
| `Height` | Integer | Yes | Control height in pixels | Height unchanged |

### Usage Patterns

| Pattern | Code | Use Case |
|---------|------|----------|
| Move position only | `Ctrl.Move(100, 50)` | Relocate without resizing |
| Resize only | `Ctrl.Move(, , 300, 200)` | Keep position, change size |
| Move and resize | `Ctrl.Move(10, 10, 400, 300)` | Full repositioning |
| Expand width | `Ctrl.Move(, , Width - 20)` | Responsive width in Size handler |
| Anchor to bottom-right | `Ctrl.Move(Width - 120, Height - 40)` | Button positioning |
| Preserve width, expand height | `Ctrl.Move(, , , Height - 60)` | Vertical expansion only |

### Common Calculations

```ahk
; Full window fill with margins
Ctrl.Move(margin, margin, Width - (2 * margin), Height - (2 * margin))

; Anchor to bottom
Ctrl.Move(10, Height - ctrlHeight - 10)

; Anchor to right
Ctrl.Move(Width - ctrlWidth - 10, 10)

; Percentage-based width
Ctrl.Move(, , Round(Width * 0.75))

; Split panels (50/50)
LeftPanel.Move(margin, margin, Round((Width - spacing) / 2), Height - (2 * margin))
RightPanel.Move(margin + panelWidth + spacing, margin, Round((Width - spacing) / 2), Height - (2 * margin))
```

---

## OnSize Event Complete Reference

Fired when GUI window is resized, maximized, minimized, or restored.

### Event Handler Pattern

```ahk
MyGui.OnEvent("Size", Gui_Size)

Gui_Size(GuiObj, MinMax, Width, Height) {
    if MinMax = -1    ; CRITICAL: Skip when minimized
        return

    ; Perform resize calculations here
}
```

### MinMax Values

| Value | State | When to Process | Common Action |
|-------|-------|-----------------|---------------|
| `-1` | Minimized | **NEVER** | Return immediately (skip processing) |
| `0` | Restored/Resized | Always | Apply resize logic |
| `1` | Maximized | Always | Apply resize logic |

### Best Practices

1. **Always check MinMax first**: `if MinMax = -1 return` prevents wasted calculations
2. **Calculate common values once**: Store margins, spacing at top of handler
3. **Use LockWindowUpdate for complex layouts**: Prevents flicker during rapid resizing
4. **Set minimum size**: `Gui("+Resize +MinSize400x300")` prevents layout breakage
5. **Test at extremes**: Minimum size, maximum size, various aspect ratios

### Anti-Flicker Pattern

```ahk
Gui_Size(GuiObj, MinMax, Width, Height) {
    static LastMinMax := -1

    if MinMax = -1
        return

    ; Freeze redraw during updates
    DllCall("LockWindowUpdate", "UInt", GuiObj.Hwnd)

    ; Move all controls
    EditBox.Move(10, 10, Width - 20, Height - 60)
    Button.Move(Width - 120, Height - 40, 100, 30)

    ; Unfreeze on state change (minimize/maximize)
    if MinMax != LastMinMax
        DllCall("LockWindowUpdate", "UInt", 0), LastMinMax := MinMax
}

; Unfreeze when user stops dragging (WM_EXITSIZEMOVE)
OnMessage(0x0232, (*) => DllCall("LockWindowUpdate", "UInt", 0))
```

---

## Binding Patterns Reference

### When to Use Bind(this)

| Scenario | Pattern | Reason |
|----------|---------|--------|
| Class method as event handler | `Btn.OnEvent("Click", this.Handler.Bind(this))` | Preserve class context |
| Loop-generated callbacks | `Btn.OnEvent("Click", Handler.Bind(A_Index))` | Capture loop variable |
| Passing methods to timers | `SetTimer(this.Update.Bind(this), 1000)` | Context preservation |
| Callback with extra data | `Btn.OnEvent("Click", Process.Bind(userData))` | Pass additional parameters |

### Event Sink Pattern (Alternative to Bind)

Pass `this` as third parameter to `Gui.__New()` to make class instance the event sink:

```ahk
class MyDialog extends Gui {
    __New() {
        super.__New("+Resize", "Title", this)  ; 'this' as event sink

        this.btn := this.Add("Button", "w100", "Click")

        ; Can use method name as string (no Bind needed)
        this.btn.OnEvent("Click", "HandleClick")
        this.OnEvent("Size", "HandleResize")
    }

    ; Methods called automatically with proper context
    HandleClick(ctrl, info) {
        MsgBox("Clicked: " this.btn.Text)
    }

    HandleResize(guiObj, minMax, w, h) {
        ; 'this' correctly refers to class instance
    }
}
```

### Comparison Table

| Approach | Syntax | Pros | Cons |
|----------|--------|------|------|
| **Bind(this)** | `OnEvent("Click", this.Method.Bind(this))` | Explicit, works everywhere | Verbose, manual binding |
| **Event Sink** | `__New(opts, title, this)` + `OnEvent("Click", "Method")` | Cleaner syntax, automatic | Only for class-based GUIs |
| **Lambda** | `OnEvent("Click", (*) => this.Method())` | Inline, captures context | Creates closure, slight overhead |
| **Function** | `OnEvent("Click", HandlerFunc)` | Simple, no binding | Can't access class members |

---

## Community Libraries Catalog

### Library Feature Comparison

| Library | Purpose | Primary Features | Difficulty | Repository/Forum |
|---------|---------|------------------|------------|------------------|
| **GuiReSizer** | Automatic responsive layouts | Percentage-based positioning, Min/Max constraints | Easy | [Forum](https://www.autohotkey.com/boards/viewtopic.php?t=113921) |
| **Easy AutoGUI** | Visual GUI designer | Drag-drop designer, window cloning, v1→v2 conversion | Easy | [GitHub](https://github.com/samfisherirl/Easy-Auto-GUI-for-AHK-v2) |
| **WebViewToo** | Modern web-based GUIs | HTML/CSS/JS interfaces, AHK↔JS communication | Medium | [GitHub](https://github.com/The-CoDingman/WebViewToo) |
| **GpGFX** | Custom graphics/controls | GDI+ wrapper, shapes, gradients, text rendering | Medium | [GitHub](https://github.com/bceenaeiklmr/GpGFX) |
| **LV_Colors** | ListView styling | Row/column/cell colors, alternating rows | Easy | [Forum](https://www.autohotkey.com/boards/viewtopic.php?f=83&t=93922) |
| **UIA-v2** | UI Automation | Modern app automation, element inspection | Advanced | [GitHub](https://github.com/Descolada/UIA-v2) |

### GuiReSizer - Percentage Positioning

**Use When**: You want automatic responsive layouts without manual calculations.

**Installation**: Download from forum, `#Include <GuiReSizer>`

**Properties**:

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `XP` | Float | X position as percentage (0.0 - 1.0) | `Ctrl.XP := 0.5` (50% from left) |
| `YP` | Float | Y position as percentage (0.0 - 1.0) | `Ctrl.YP := 0.25` (25% from top) |
| `WP` | Float | Width as percentage (0.0 - 1.0) | `Ctrl.WP := 0.8` (80% of window width) |
| `HP` | Float | Height as percentage (0.0 - 1.0) | `Ctrl.HP := 0.6` (60% of window height) |
| `MinW` | Integer | Minimum width in pixels | `Ctrl.MinW := 200` |
| `MaxW` | Integer | Maximum width in pixels | `Ctrl.MaxW := 800` |
| `MinH` | Integer | Minimum height in pixels | `Ctrl.MinH := 100` |
| `MaxH` | Integer | Maximum height in pixels | `Ctrl.MaxH := 600` |
| `OffsetX` | Integer | Pixel adjustment to X | `Ctrl.OffsetX := 10` |
| `OffsetY` | Integer | Pixel adjustment to Y | `Ctrl.OffsetY := -5` |

**Basic Pattern**:
```ahk
MyGui := Gui("+Resize")
MyGui.OnEvent("Size", GuiReSizer)

Edit := MyGui.Add("Edit", "w400 h300")
Edit.WP := 0.95    ; 95% width
Edit.HP := 0.80    ; 80% height

Btn := MyGui.Add("Button", "w100 h30", "OK")
Btn.XP := 0.85     ; Anchor to 85% from left
Btn.YP := 0.90     ; Anchor to 90% from top

MyGui.Show("w500 h400")
```

### Easy AutoGUI - Visual Designer

**Use When**: Rapid prototyping, converting existing Windows applications, learning GUI layout.

**Key Features**:
- Drag-drop visual designer
- Clone any Windows application to AHK2 code (under 1 second)
- Real-time v1→v2 code conversion
- Menu builder
- Auto-update system
- Generates clean, error-free code

**Output Code Style**:
```ahk
#Requires Autohotkey v2
myGui := Gui()
ButtonOK := myGui.Add("Button", "x16 y48 w80 h23", "&OK")
ButtonCancel := myGui.Add("Button", "x104 y48 w80 h23", "&Cancel")
ButtonOK.OnEvent("Click", OKHandler)
myGui.OnEvent('Close', (*) => ExitApp())
myGui.Show("w480 h300")
```

### WebViewToo - HTML/CSS/JS GUIs

**Use When**: Need modern web technologies, responsive Bootstrap/Tailwind designs, rich dashboards.

**Requirements**: Edge WebView2 Runtime (included in Windows 11, downloadable for Windows 10)

**Core Methods**:

| Method | Syntax | Description |
|--------|--------|-------------|
| `Load()` | `Load(URL/HTML)` | Load URL or raw HTML |
| `Show()` | `Show([Options])` | Display window |
| `ExecuteScript()` | `ExecuteScript(JSCode)` | Run JavaScript in page |
| `QueryPage()` | `QueryPage(JSExpression)` | Execute JS and return value |
| `PostWebMessageAsString()` | `PostWebMessageAsString(Data)` | Send data to JavaScript |

**Communication Pattern**:

AHK → JavaScript:
```ahk
MyWindow.PostWebMessageAsString("Hello from AHK")
MyWindow.ExecuteScript("console.log('AHK says hi')")
```

JavaScript → AHK:
```javascript
// Receive from AHK
window.chrome.webview.addEventListener('message', (event) => {
    console.log('From AHK:', event.data);
});

// Send to AHK (requires message handler setup)
window.chrome.webview.postMessage('Hello from JS');
```

### GpGFX - Graphics Library

**Use When**: Custom controls, animations, 2D graphics, visual effects.

**Core Components**:

| Component | Purpose | Key Parameters |
|-----------|---------|----------------|
| `Layer()` | Transparent window layer | Base for all graphics |
| `Rectangle()` | Draw rectangle | `X, Y, Width, Height, Color/Gradient` |
| `Square()` | Draw square | `X, Y, Size, Color/Gradient` |
| `Ellipse()` | Draw ellipse | `X, Y, Width, Height, Color` |
| `Text()` | Render text | `X, Y, String, Font, Size, Color` |
| `Draw()` | Display graphics | `Layer` |
| `End()` | Cleanup resources | - |

**Gradient Syntax**:
```ahk
; Two-color gradient
rect := Rectangle(50, 50, 200, 100, ["Red", "Orange"])

; Hex colors
square := Square(100, 100, 150, ["#FF0000", "#880000"])
```

**Text Overlay**:
```ahk
rect := Rectangle(50, 50, 200, 100, "Blue")
rect.str := "Button Text"
rect.Font := "Arial"
rect.FontSize := 16
```

### LV_Colors - ListView Styling

**Use When**: Need colored ListView rows/columns/cells.

**Methods**:

| Method | Syntax | Description |
|--------|--------|-------------|
| `AlternateRows()` | `AlternateRows(Color1, Color2)` | Zebra striping |
| `Row()` | `Row(RowNum, BgColor, TextColor)` | Color specific row |
| `Col()` | `Col(ColNum, BgColor, TextColor)` | Color entire column |
| `Cell()` | `Cell(Row, Col, BgColor, TextColor)` | Color single cell |

**Color Format**: RGB hex integer `0xRRGGBB`

**Example**:
```ahk
LV := MyGui.Add("ListView", "w400 h300", ["Name", "Status", "Value"])
LV.Add("", "Item 1", "Active", "100")
LV.Add("", "Item 2", "Inactive", "50")

LVC := LV_Colors(LV)
LVC.AlternateRows(0xF0F0F0, 0xFFFFFF)  ; Gray/white alternating
LVC.Row(2, 0xFFCCCC)                    ; Red background for row 2
LVC.Col(2, , 0x008800)                  ; Green text for status column
```

### UIA-v2 - UI Automation

**Use When**: Automating modern apps (UWP, browsers, Office), element inspection, accessibility.

**Core Methods**:

| Method | Syntax | Description |
|--------|--------|-------------|
| `ElementFromHandle()` | `ElementFromHandle(WinTitle)` | Get root element from window |
| `FindElement()` | `FindElement({Conditions})` | Find first matching element |
| `FindElements()` | `FindElements({Conditions})` | Find all matching elements |
| `Invoke()` | `Element.Invoke()` | Click/activate element |
| `SetValue()` | `Element.Value := "text"` | Set input value |

**Condition Properties**: `Name`, `Type`, `AutomationId`, `ClassName`, `Value`

**Example**:
```ahk
; Find and click element
explorerEl := UIA.ElementFromHandle("ahk_exe explorer.exe")
documentsItem := explorerEl.FindElement({Name: "Documents", Type: "ListItem"})
if documentsItem
    documentsItem.Invoke()

; Browser automation
chromeEl := UIA.ElementFromHandle("ahk_exe chrome.exe")
searchBox := chromeEl.FindElement({Type: "Edit", Name: "Address and search bar"})
searchBox.Value := "autohotkey v2"
searchBox.SendKeys("{Enter}")
```

---

## Color and Font Options Reference

### Color Syntax

| Format | Example | Description |
|--------|---------|-------------|
| Named color | `"Red"`, `"Blue"`, `"White"` | Standard color names |
| Hex string | `"0xFF0000"` | Hex string format |
| Hex integer | `0xFF0000` | Direct hex value |
| RGB function | `RGB(255, 0, 0)` | Custom RGB function |

### Common Colors

| Name | Hex | RGB |
|------|-----|-----|
| Black | `0x000000` | `0, 0, 0` |
| White | `0xFFFFFF` | `255, 255, 255` |
| Red | `0xFF0000` | `255, 0, 0` |
| Green | `0x00FF00` | `0, 255, 0` |
| Blue | `0x0000FF` | `0, 0, 255` |
| Yellow | `0xFFFF00` | `255, 255, 0` |
| Gray | `0x808080` | `128, 128, 128` |
| Light Gray | `0xF0F0F0` | `240, 240, 240` |
| Dark Gray | `0x202020` | `32, 32, 32` |

### Font Options

| Option | Description | Example |
|--------|-------------|---------|
| `s{Size}` | Font size in points | `"s12"` |
| `Bold` | Bold weight | `"Bold"` |
| `Italic` | Italic style | `"Italic"` |
| `Strike` | Strikethrough | `"Strike"` |
| `Underline` | Underline | `"Underline"` |
| `c{Color}` | Text color | `"cRed"`, `"c0xFF0000"` |

**SetFont() Pattern**:
```ahk
; Set font for next controls
MyGui.SetFont("s12 Bold", "Segoe UI")
MyGui.Add("Text", "w300", "Large bold text")

; Reset to default
MyGui.SetFont()
MyGui.Add("Text", "w300", "Normal text")

; Inline color
MyGui.Add("Text", "w300 cRed", "Red text")
```

### Dark Mode Colors

Recommended dark mode palette:

| Element | Color | Hex |
|---------|-------|-----|
| Background | Dark gray | `0x202020` |
| Text | White | `0xFFFFFF` |
| Controls | Medium gray | `0x303030` |
| Borders | Light gray | `0x404040` |
| Accent | Blue | `0x0078D7` |

---

## Resize Pattern Decision Matrix

Choose the right approach based on your needs:

| Pattern | Complexity | Flexibility | Performance | Best For |
|---------|------------|-------------|-------------|----------|
| **Manual Move() calls** | High | Maximum | Best | Simple layouts, custom logic, learning |
| **GuiReSizer library** | Low | Medium | Good | Percentage-based layouts, rapid development |
| **Custom resize class** | Medium | High | Good | Reusable patterns, complex apps |
| **Fixed size (no resize)** | None | None | N/A | Dialogs, simple forms |
| **WebViewToo (CSS)** | Low | Maximum | Moderate | Modern responsive designs, web tech familiarity |

### Layout Pattern Quick Reference

| Layout Type | Best Approach | Code Pattern |
|-------------|---------------|--------------|
| Single expanding control | Manual Move | `Edit.Move(margin, margin, Width - 2*margin, Height - 2*margin)` |
| Anchored buttons | Manual Move | `Btn.Move(Width - btnW - margin, Height - btnH - margin)` |
| Multi-panel split | Manual Move + calculations | Store panel ratios, calculate in Size handler |
| Percentage-based grid | GuiReSizer | Set WP/HP properties on controls |
| Complex responsive | Custom class + Move | Extend Gui class with custom resize logic |
| Modern web layout | WebViewToo | Use CSS flexbox/grid |

---

## Component Pattern Comparison

### Extend vs Compose vs Factory

| Pattern | When to Use | Pros | Cons | Example |
|---------|-------------|------|------|---------|
| **Extend Gui** | Custom dialog types | Full GUI customization, event sink | Tight coupling, inheritance | `class MyDialog extends Gui` |
| **Compose components** | Reusable UI elements | Modular, flexible, testable | More code | `class LabeledInput` wraps label+edit |
| **Factory methods** | Standardized controls | Consistency, centralized | Less flexible | `ComponentFactory.CreateButton()` |

### Extension Pattern

```ahk
class CustomDialog extends Gui {
    __New(title := "Dialog") {
        super.__New("+AlwaysOnTop", title, this)  ; Event sink
        this.CreateControls()
    }

    CreateControls() {
        this.Add("Text", "w300", "Message here")
        this.Add("Button", "w100", "OK").OnEvent("Click", "HandleOK")
    }

    HandleOK(ctrl, info) {
        this.Hide()  ; Can access GUI methods directly
    }
}
```

**Best for**: Modal dialogs, specialized window types, full customization.

### Composition Pattern

```ahk
class LabeledInput {
    __New(parentGui, label, options := "w200") {
        this.label := parentGui.Add("Text", "xm", label . ":")
        this.edit := parentGui.Add("Edit", "x+10 " options)
    }

    Value {
        get => this.edit.Value
        set => this.edit.Value := value
    }
}

; Usage
myGui := Gui()
nameField := LabeledInput(myGui, "Name", "w250")
emailField := LabeledInput(myGui, "Email", "w250")
myGui.Show()
```

**Best for**: Reusable UI components, complex controls, testable units.

### Factory Pattern

```ahk
class ComponentFactory {
    static CreateButton(gui, text, callback := "", isDefault := false) {
        btn := gui.Add("Button", "w100 h32", text)
        if callback
            btn.OnEvent("Click", callback)
        if isDefault
            btn.Opt("+Default")
        return btn
    }

    static CreateSection(gui, title) {
        gui.SetFont("s11 Bold")
        gui.Add("Text", "xm w400 +0x200", title)
        gui.SetFont()
        gui.Add("Text", "xm w400 h2 +0x10")  ; Separator
    }
}

; Usage
myGui := Gui()
ComponentFactory.CreateSection(myGui, "User Details")
ComponentFactory.CreateButton(myGui, "Submit", SubmitHandler, true)
```

**Best for**: Consistent styling, standardized components, centralized changes.

### Decision Guide

```
Need custom window behavior? → Extend Gui
Need reusable UI element? → Compose component class
Need consistent styling? → Factory methods
Need all of the above? → Combine patterns (extend + compose + factory)
```

---

## Quick Reference Cheat Sheet

### Essential Patterns

```ahk
; Class-based GUI with resize
class MyApp extends Gui {
    __New() {
        super.__New("+Resize +MinSize400x300", "Title", this)
        this.edit := this.Add("Edit", "w400 h300")
        this.btn := this.Add("Button", "w100 h30", "OK")
        this.btn.OnEvent("Click", "HandleClick")
        this.OnEvent("Size", "HandleResize")
        this.Show()
    }

    HandleClick(ctrl, info) {
        MsgBox(this.edit.Text)
    }

    HandleResize(guiObj, minMax, w, h) {
        if minMax = -1
            return
        this.edit.Move(10, 10, w - 20, h - 60)
        this.btn.Move(w - 120, h - 40, 100, 30)
    }
}

; Event handler with Bind
this.btn.OnEvent("Click", this.Handler.Bind(this))

; Lambda event handler
this.btn.OnEvent("Click", (*) => MsgBox("Clicked"))

; Loop with captured variable
Loop 5
    MyGui.Add("Button", "xm", "Button " A_Index)
        .OnEvent("Click", HandleBtn.Bind(A_Index))

; Percentage resize (GuiReSizer)
edit := MyGui.Add("Edit", "w400 h300")
edit.WP := 0.95
edit.HP := 0.85
MyGui.OnEvent("Size", GuiReSizer)
```

### Common Mistakes to Avoid

| Mistake | Problem | Solution |
|---------|---------|----------|
| Handler with no parameters | `HandleClick() { }` crashes | `HandleClick(*) { }` or `HandleClick(ctrl, info) { }` |
| No MinMax check in Size | Processes when minimized | `if MinMax = -1 return` |
| Missing Bind(this) | `this` refers to wrong object | `.OnEvent("Click", this.Method.Bind(this))` |
| Local GUI variable | GUI destroyed when function exits | Use `global` or class property |
| Not calling Destroy() | Memory leak | `MyGui.Destroy()` when done |
| Circular reference | Prevents cleanup | Use Bind or weak references |

---

## Additional Resources

- **AHK v2 GUI Megathread**: [Forum Link](https://www.autohotkey.com/boards/viewtopic.php?style=19&t=123220) - Comprehensive library listing
- **Official Documentation**: [AutoHotkey v2 Docs](https://www.autohotkey.com/docs/v2/)
- **Diátaxis Framework**: For understanding this manual's structure

---

*This reference is part of the AHK2 manual system. See also: tutorials/gui-basics.md, how-to/responsive-layouts.md, explanation/gui-architecture.md*
